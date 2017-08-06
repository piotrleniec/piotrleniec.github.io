---
layout: post
title: Handling complex business logic with dry-rb
date: 2017-08-06 15:52:57 +0200
categories: ruby
description: |
  When I write Rails code, I tend to write as much business logic as possible
  in CRUD, because it's predictable, easy to maintain and it's "the Rails way".
  The fun begins when business needs aren't simple and cannot be expressed in
  CRUD. In cases like that, I extract code into service objects, but what if
  the logic has multiple steps and each step can fail independently? Here comes
  dry-rb library with schema and transactions.
last_modified_at: 2017-08-06 15:52:57 +0200
header-img: assets/images/dry-rb-background.png
image: assets/images/dry-rb-background-with-logo.jpg
---

When I write Rails code, I tend to write as much business logic as possible in
CRUD, because it's predictable, easy to maintain and it's "the Rails way". The
fun begins when business needs aren't simple and cannot be expressed in CRUD.
In cases like that, I extract code into service objects, but what if the logic
has multiple steps and each step can fail independently? Here comes [dry-rb](http://dry-rb.org/)
library with schema and transactions.

## Our task

Let's suppose that we're working on an e-commerce system, and our job is to
create a backend for one step checkout. In this part of the application, we
need to validate user's data (which is quite complicated) and conduct a series
of operations like capturing payments, removing stocks and sending emails.

It's surely not a simple CRUD like action so "the Rails way" might not be the
best choice to build this solution. Putting everything in "fat models" is a
violation of [the single responsibility principle](https://www.youtube.com/watch?v=Gt0M_OHKhQE).

![uncle bob approves](/assets/images/uncle-bob-approves.jpg)

Are we doomed to an ugly code? Fortunately, no. We can extract validation logic
to validators and move business logic to operations and compose them with
transactions. Thanks to `dry-rb` library, it's straightforward to achieve it.

In this post, I'm not going to show you how `dry-validation` and
`dry-transaction` works. It's not a tutorial. My goal is to show you that
"the Rails way" isn't always effective and that `dry-rb` is a great library
that can help you decouple business logic from the rest of your application.

## Validation

Our first task is to validate the following JSON request:

```json
{
  "user_token": "abc123",
  "line_items": [
    {
      "product_id": 1,
      "quantity": 1
    },
    {
      "product_id": 2,
      "quantity": 2
    },
    {
      "product_id": 3,
      "quantity": 3
    }
  ],
  "shipping_method_id": 4,
  "payment_token": "def456"
}
```

Let's break the request into parts:

* `user_token` - it's a regular session token. We need to validate if it matches
  any user
* `line_items` - an array of products which user wants to buy. We need to verify
  that every product is eligible and the quantities don't exceed stocks
* `shipping_method_id` - an id of the selected shipping method. Simple validation
  which checks if the shipping method exists and if it applies to the user's address
* `payment_token` - Stripe's or Braintree's token used to capture payments. We
  need to request an endpoint to validate if the amount of money we want to
  capture equals the amount of order's total.

How would "the Rails way" validation look like?

```ruby
# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  before_action :authenticate!

  def create
    order = Order.new(create_params)

    authorize! order

    if order.save
      render json: order, status: 201
    else
      render json: order.errors, status: 422
    end
  end

  private

  def create_params
    params.require(:order).permit(...)
  end
end

# app/models/order.rb

class Order < ApplicationRecord
  belongs_to :user
  has_many :line_items
  belongs_to :shipping_method

  accepts_nested_attributes_for :line_items
  validates :shipping_method, presence: true
  validate :shipping_method_applicable
  validates :payment_token, presence: true
  validate :amount_of_money_for_payment_token

  private

  def shipping_method_applicable
    ...
  end

  def payment_token_applicable
    ...
  end
end

# app/models/line_item.rb

class LineItem < ApplicationRecord
  belongs_to :order
  belongs_to :product

  validates :order, presence: true
  validates :product, presence: true
  validates :quantity, presence: true, numericality: { only_integer: true, greater_than: 0 }
end
```

To validate only one request we modified three files that aren't checkout
specific. What's worse we tied `Order` and `LineItem` models to the validation
and logic is split between controller and model. For me, it's a total disaster.
"the Rails way" doesn't work in this case.

To make it a bit cleaner, we could use `ActiveModel` to extract validation to a
single class, but still, there would be problems with line items validation
because they're nested models.

Let's use [dry-validation](http://dry-rb.org/gems/dry-validation/).
Rewritten code could look like this:

```ruby
CompleteOneStepCheckoutSchema = Dry::Validation.Schema do
  required(:user_token).filled(:str?)
  required(:line_items).each do
    schema do
      required(:product_id).filled(:int?)
      required(:quantity).filled(:int?, gt?: 0)
    end
  end
  required(:shipping_method_id).filled(:int?)
  required(:payment_token).filled(:str?)
end
```

Ok, cool but we don't validate products presence, stock availability and
payment token. That's right. We don't validate them in schema because that
wouldn't be optimal. We'd do the same operations like product or user loading
all over again. To tackle this issue optimally, we can use transactions.

## Operations

Let's split the checkout completion to small operations that can be executed
and tested independently. In the first operation, we could validate the
parameters against the schema, and in the following operations we could verify
`user_token`, load user and check if products are available. Here's the code
for the first operation

```ruby
class ValidateParameters
  include Dry::Transaction::Operation

  def call(input)
    validation_result = CompleteOneStepCheckoutSchema.call(input)

    if validation_result.errors.any?
      Left(validation_result.errors)
    else
      Right(params: input)
    end
  end
end
```

`Right` means success and `Left` means failure. Notice that we return
`{ params: input }` instead of `input`. It'll be clear later. In the next
operation, we check if `user_token` is valid and load a user.

```ruby
class LoadUser
  include Dry::Transaction::Operation

  def call(input)
    user_token = input[:params][:user_token]
    user = User.find_by_token(user_token)

    user ? Right(input.merge(user: user)) : Left(:invalid_user_token)
  end
end
```

Now you should realize why did we return a hash with params under `params` key.
This pattern allows us to avoid multiple database calls and to accumulate state
in a very clean way. We don't modify the internal state, so testing is trivial.
Here's an example of the RSpec test:

```ruby
describe LoadUser do
  describe '#call' do
    subject { described_class.new }

    context 'when user_token matches a user' do
      let(:user) { create(:user) }
      let(:input) { { params: { user_token: user.token } } }

      it 'loads a user and puts it in the output' do
        result = subject.call(input)

        expect(result).to be_success
        expect(result[:user]).to eq user
      end
    end

    context "when user_token doesn't match any user" do
      let(:input) { { params: { user_token: 'invalidtoken' } } }

      it "doesn't load a user and returns Left" do
        result = subject.call(input)

        expect(result).to be_failure
        expect(result.value).to eq :invalid_user_token
      end
    end
  end
end
```

Great, isn't it? Isolating the functionality in "the Rails way" would be very
difficult if not impossible.

## Transaction

Now when we have implemented and tested the operations, we need to compose
them. We'll use a [transaction](http://dry-rb.org/gems/dry-transaction/).
Transaction is a wrapper for operations which runs them in a specified order
and specified behavior. `dry-transaction` defines the following step adapters:

* `step` - the most commonly used adapter. It only passes `Right` value to the
  next operation or stops the transaction on `Left`
* `map` - gives an output to the next transaction (it assumes the operation
  cannot fail)
* `try` - an extension of `map` step. Additionally to the `map` functionality
  it catches exceptions (specified in step definition) and stops the transaction
* `tee` - great adapter for side effects. It passes the input to the next
  operation without changing it.

Enough theory, let's code it.

```ruby
class CompleteOneStepCheckout
  include Dry::Transaction(container: Container)

  step :validate_parameters
  step :load_user
end
```

You might've seen, that I have skipped `Container` definition. In a nutshell,
it's a class that contains dependencies for our transaction. You can read about
it [here](http://dry-rb.org/gems/dry-container/).

The code seems to be clean and brief, but the real magic is about to happen.
Look at this:

```ruby
transaction = CompleteOneStepCheckout.new

transaction.call(params) do |m|
  m.success do
    puts 'transaction succeeded'
  end

  m.failure :validate_parameters do |errors|
    puts 'params are invalid'
    puts errors
  end

  m.failure :invalid_user_token do
    puts 'invalid user token'
  end
end
```

This is what makes transactions so great. We can call a transaction and
describe what are we going to do with each result. We can define blocks that
run after success, after an individual step fails (see `validate_parameters`
block) or after a particular error (see `invalid_user_token` block). Such
flexibility is impossible to achieve when writing "the Rails way" code.

`dry-validation` and `dry-transaction` are great libraries, but they aren't
flawless. My biggest complaints to dry-validation are that custom predicates
are cumbersome and custom validation blocks don't work well for nested data.
When it comes to transactions, there aren't any viable ways to rollback a
transaction. There are listeners, but it should be built in transactions.

Schemas and transactions are only a tip of the iceberg. There are many other
interesting libraries from `dry-rb` team. `dry-types` brings runtime type
checking for objects, `dry-auto_inject` offers inversion of control pattern or
`dry-monads` which is a powerful implementation of monads in Ruby.

As you can see, `dry-rb` is a great gem. It provides a lot of utilities without
enforcing any architecture. It's a must have for every Ruby programmer who
wants to make its code awesome.
