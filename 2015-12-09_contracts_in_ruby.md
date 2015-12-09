# Contracts in Ruby

I'm going to be looking at [Contracts.ruby](https://github.com/egonSchiele/contracts.ruby) in this blog post. It's a library by Aditya Bhargava who has a brilliant blog here: http://adit.io/

Contracts are a form of run-time type checking. You can use contracts to improve tests and documentation. You can also configure the gem to have no performance impact in production.

## Setup

You'll need to install the Contracts gem.

### Bundler

```ruby
gem 'contracts'
```

### Command line

```bash
gem install contracts
```

## Typechecking? Like Java!?

Calm down. It's not bad, I promise.

You only have to use contracts on the methods where you want additional safety. That means you can add a handful of contracts to your app's problem spots, but leave the rest unchanged. Should you decide that you like the typechecks then you can add more. It's up to you.

Here's a simple example. Forgive how utterly contrived this is.

```ruby
require 'contracts'

class Example
  include Contracts

  # This contract ensures the method takes two numbers and returns another number.
  Contract Num, Num => Num
  def multiply_with_contract(a, b)
    a * b
  end

  # This one doesn't have a contract. It won't be type checked, which is fine.
  def multiply_without_contract(a, b)
    a * b
  end
end

example = Example.new

# The method without a contract works just like any other method in Ruby. It's
# completely unchanged.
example.multiply_without_contract 2, 3
# => 6

# This is not what we intended to happen.
example.multiply_without_contract "Oops!", 3
# => "Oops!Oops!Oops!"

# Here's the version with a contract. It does the right thing when the arguments
# are correct.
example.multiply_with_contract 2, 3
# => 6

example.multiply_with_contract "Oops!", 3
# ~> Contract violation for argument 1 of 2: (ParamContractError)
# ~>         Expected: Num,
# ~>         Actual: "Oops!"
# ~>         Value guarded in: Example::multiply_with_contract
# ~>         With Contract: Num, Num => Num
```

## Say Goodbye to `nil can't be coerced to...` Errors

Errors of the type I mentioned above are rare. I can count on one hand the number of times I've passed a string when the method clearly called for an integer. The much more common case is when you  pass a `nil` when you needed something else.

Something like this happens to me fairly frequently:

```ruby
class Example
  def purchase_id
    # Call `find` on an enumerable to get a value
    [1, 2, 3].find { |n| n > 5 }
    # => nil
  end

  def purchase
    # Unfortunately it returned `nil`, and now we're going to try to look up a purchase
    # with an ID of NULL in the database. That's probably not what we meant to do.
    Purchase.find purchase_id
  end
end
```

Depending on how we test the code above, we may never even notice that something is wrong. It's possible that we don't do anything with the result of `purchase`. We've all written a few tests that just assert some method doesn't raise an exception when called. It's a bad test, but some code is hard to test, and we're under immense pressure to ship the code *right now*!

Contracts can make this a lot better.

```ruby
require 'contracts'

class Purchase; end

class Example
  include Contracts

  # This means the method takes no arguments and returns a number.
  Contract None => Num
  def purchase_id
    [1, 2, 3].find { |n| n > 5 }
  end

  def purchase
    Purchase.find purchase_id
  end
end

Example.new.purchase
# ~> Contract violation for return value: (ReturnContractError)
# ~>         Expected: Num,
# ~>         Actual: nil
# ~>         Value guarded in: Example::purchase_id
# ~>         With Contract: None => Num
```

## Contracts as Documentation

Sometimes things get complicated in big applications. I've seen plenty of code where it's not clear what the types of the arguments are. Here's a snippet of real production code:

```ruby
def self.duplicate?(user)
  User.find_by_email(user.email).present?
end
```

We've got a `User` class in our app, so it seems likely that `user` must be a `User`, right? That seems reasonable, but it's incorrect. In practice this method is always passed an instance of `WaitingUser`. It's clearly a poor choice of name, but this stuff happens all the time.

Adding in a contract was one short line of code, but makes the intended use much clearer.

```ruby
Contract WaitingUser => Bool
def self.duplicate?(user)
  User.find_by_email(user.email).present?
end
```

## In Closing

There's a lot that I haven't gone over here. For instance, Contracts has some powerful type system features. They're the kinds of things  you'd find in languages like Haskell and Scala. You can also disable Contracts in production. This lets you expand your test coverage but doesn't slow down production.

I plan to write more about Contracts. In the meantime, their documentation is great and the community is friendly. I encourage you to go check it out.

Github: https://github.com/egonSchiele/contracts.ruby
Project site: http://egonschiele.github.io/contracts.ruby/
