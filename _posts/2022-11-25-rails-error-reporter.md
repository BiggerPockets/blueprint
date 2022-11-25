---
author: John Gallagher
title: Handling errors with Rails 7 new error handlers
description: Rails 7 adds an error handling API. How can we use it for fun and profit?
excerpt:
tags: product engineering rails
image: /assets/images/blank-branding-identity-business-cards-6372.jpg
---

## Context

* Writing code to request a quote from an external JSON API
* When the API request fails, we want to:
  1. Log the error
  2. Report error to Sentry

Example code:

```ruby
class RequestQuote
  def call(id)
    http_client.get("/api/v1/quotes/#{id}.json")
  rescue HTTPFailure => error
    logger.error("Something went wrong: #{error.message}")
    Sentry.capture_exception(error)
  end

  def http_client
    Faraday.new("https://pricing.com") { |f| f.response :raise_error }
  end
end
```

Next - create a quote. Write another service object:

```ruby
class CreateQuote
  def call(quote)
    http_client.post("/api/v1/quotes.json", body: quote.to_json)
  rescue HTTPFailure => error
    # This looks familiar...
    logger.error("Something went wrong: #{error.message}")
    Sentry.capture_exception(error)
  end

  def http_client
    Faraday.new("https://pricing.com") { |f| f.response :raise_error }
  end
end
```

## The problems

### 1. Reinventing the wheel

The code forces us to make the same decisions over and over again:

* Do we log errors? Or warnings? Or log nothing?
* What context do we pass along with the logs? Just the error message? Or more details?
* What context do we pass along to Sentry?
* What level of error do we raise in Sentry?

As a result, in my experience, error handling in Rails codebases tends to be wildly inconsistent.

It either causes too much noise in the logs or gives too few details when debugging.

### 2. Difficult to read

This code is noisy. It contains a lot of details we don't care about.

The code doesn't tell us concisely what it's doing.

It has abstractions that are low level.

### 3. Tightly coupled

This code is tightly coupled to Sentry and the logger.

Any changes to our error handling cause changes to ripple across the whole codebase.

Examples of changes:

* Swapping out Sentry for New Relic
* Adding Rollbar error tracking
* Adding structured logging with Semantic Logger (which has a different interface to Ruby Logger)
* Adding extra context to all error tracking (e.g. user IDs)

## The solution

Rails 7 introduces a new API - [`Rails.error`](https://edgeguides.rubyonrails.org/error_reporting.html).

### Usage

### 1. Create handler class

```ruby
class LoggingErrorHandler
  def initialize(logger: Rails.logger)
    @logger = logger
  end

  def report(error, handled:, severity:, context:, source: "application")
    @logger.info("Something went wrong: #{error.message}")
  end
end
```

### 2. Subscribe to errors

```ruby
# config/initializers/semantic_logger.rb
Rails.error.subscribe(LoggingErrorHandler.new)

# config/initializers/sentry.rb
# Sentry::Rails::ErrorSubscriber comes with Sentry
Rails.error.subscribe(Sentry::Rails::ErrorSubscriber.new)
```

### 3. Handle the error

```ruby
class RequestQuotes
  def call(id)
    Rails.error.handle(HTTPFailure) do
      http_client.get("/api/v1/quotes/#{id}.json")
    end
  end

  def http_client
    Faraday.new("https://pricing.com") { |f| f.response :raise_error }
  end
end
```

## Features

### Three levels of handling

All examples taken from [the edge Rails guides](https://edgeguides.rubyonrails.org/error_reporting.html)

**1. Reporting and swallowing errors with `#handle`**

```ruby
result = Rails.error.handle do
  1 + '1' # raises TypeError
end
result # => nil
1 + 1 # This will be executed
```

Passes `severity: :warning` to the error handler.

**2. Reporting and reraising errors with `#record`**

```ruby
Rails.error.record do
  1 + '1' # raises TypeError
end
1 + 1 # This won't be executed
```

Passes `severity: :error` to the error handler.

**3. Manually reporting errors with `#report`**

```ruby
begin
  # code
rescue StandardError => e
  Rails.error.report(e)
end
```

### Allows a fallback

Perfect for when you want to support the null object pattern if something goes wrong:

```ruby
result = Rails.error.handle(fallback: -> { 0 }) do
  1 + '1' # raises TypeError
end
result # => 0
```

It will accept any object that responds to `#call`.

### Allows context for better debugging

Pass in `context:` for the error to help with debugging.

Each error handler gets to decide what to do with this context.

```ruby
Rails.error.handle(HTTPFailure, context: { user_id: "123456" }) do
  http_client.get("/api/v1/quotes/#{id}.json")
end
```

Or maybe [set global context](https://edgeguides.rubyonrails.org/error_reporting.html) in some middleware:

```ruby
class SetUserIdMiddleware
  def call(env)
    Rails.error.set_context(user_id: env.cookies["user_id"])
  end
end
```

Every `Rails.error.handle` after this will include the context.

## Advantages

### 1. Standardised

Make the decision about how to handle errors once, write the handler once, then move on.

### 2. Decoupled

Want to add more error reporters? No worries, create a new class, subscribe and now all your errors go to the new reporter. Very powerful!

Want to change how logging works? Change it in one place. And it reflects everywhere you use `Rails.error`.

### 3. Testable

The subscribers are very small Ruby classes, so they're easy to test.

It'd be very easy to write an in memory test only subscriber, meaning your tests just got easier to write and faster too.

### 4. Severity levels

The severity is passed to each subscriber and they decide how to report it.

For warnings, we can now log them all as `#warn` very easily.

### 5. Structured logging

A perfect fit for structured logging. We can pass around relevant `context` and the error handlers can log as necessary.

At `BiggerPockets` we've gone a stage further and created a `StructuredError` object that can hold context.

### 6. Cleaner

`Rails.error.handle` - it doesn't get much cleaner or obvious than that.

## Disadvantages

### 1. More code

As usual, when you find the right abstraction, it's more code. There are more concepts to understand.

### 2. Learning curve

As with every new way of doing things, you'll need to invest some time to learn and remember this new way of error handling.

### 3. Less obvious

The details are hidden. This is a good thing - we're encapsulating behaviour.

But as with all abstractions, when you do want to know the details, it can be a bit harder to find.

### 4. Missing features

We can't rescue multiple exceptions yet. But it's just a matter of time before that becomes available.

# Summary

In summary, the Rails team have done an wonderful job of creating a clean, easy to use API that solves a real problem.

You can [read more in the Rails Edge guides](https://edgeguides.rubyonrails.org/error_reporting.html).

Happy error handling!

By [John Gallagher](https://github.com/johngallagher)
