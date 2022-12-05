---
author: John Gallagher
title: Gracefully handle errors in Rails 7
description: Rails 7 adds an error handling API. How can we use it to remove duplication?
excerpt: Rails 7 includes a new error reporting pattern. Handle your errors in one simple clean line of code. Let's see how.
tags: product engineering rails
image: /assets/images/error-report.jpg
---

Rails 7 includes a new error reporting pattern.

The new pattern introduces decoupled error handlers whilst reducing repetitious boilerplate code.

Handle your errors in one simple clean line of code. Let's see how.

## Example

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

As a result, error handling in Rails codebases tends to be wildly inconsistent.

We either get too much noise in the logs or get too few details when debugging.

### 2. Difficult to read

This code is noisy. It contains a lot of details we don't care about.

There's duplication of `logger.error`, `Sentry.capture_exception`.

The code doesn't tell us concisely what it's doing.

It has abstractions that are too low level.

### 3. Tightly coupled

This code is tightly coupled to Sentry and the logger.

Any changes to our error handling cause changes to ripple across the whole codebase.

Examples of changes:

* Swapping out Sentry for New Relic
* Adding Rollbar error tracking
* Adding structured logging with [Semantic Logger](https://logger.rocketjob.io/index.html) (has a different interface to Ruby Logger)
* Adding extra context to all error tracking (e.g. user IDs)

## The solution

Rails 7 introduces a new API - [`Rails.error`](https://edgeguides.rubyonrails.org/error_reporting.html).

### 1. Write an error subscriber

```ruby
class LoggingErrorSubscriber
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
Rails.error.subscribe(LoggingErrorSubscriber.new)

# config/initializers/sentry.rb
# Sentry::Rails::ErrorSubscriber comes with Sentry
Rails.error.subscribe(Sentry::Rails::ErrorSubscriber.new)
```

### 3. Handle the error

```ruby
class RequestQuotes
  def call(id)
    # Errors handled in one line - concise and intention revealing
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

Passes `severity: :warning` to the error subscriber.

**2. Reporting and reraising errors with `#record`**

```ruby
Rails.error.record do
  1 + '1' # raises TypeError
end
1 + 1 # This won't be executed
```

Passes `severity: :error` to the error subscriber.

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

This is passed along to the error reporter.

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

Every `Rails.error.handle` called afterwards will include the context.

## Advantages

### 1. Standardised

Make the decision about how to handle errors once, write the subscriber once, then move on.

### 2. Decoupled

Want to send errors to a different error reporting platform?

Write a new class that responds to `#report`, subscribe and now every call to `Rails.error` will send the error along.

Want to change how logging works?

Change the logging in your error reporter. And it reflects everywhere you use `Rails.error`.

### 3. Testable

The error reporter instances are very small Ruby classes, so they're easy to test.

It'd be very easy to write [testing helpers for error reporting](https://github.com/rails/rails/pull/46029), meaning your tests just got easier to write and faster too.

### 4. Severity levels

The severity is passed to each subscriber and they decide how to report it.

For warnings, we can now log them all as `#warn` very easily.

### 5. Great for structured logging

We pass around `context` and the error subscribers can log this extra detail.

Ideal for tagging log entries with user IDs, transaction IDs or other cross cutting data that provides better debugging.

At `BiggerPockets` we've gone a stage further and created a `StructuredError` object which stores this context.

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

We can't rescue multiple exceptions yet. But [this PR, when released](https://github.com/rails/rails/pull/46299) will add that.

## Summary

The Rails team have done an wonderful job of creating a clean, easy to use API that solves a big problem.

You can [read more in the Rails Edge guides](https://edgeguides.rubyonrails.org/error_reporting.html).

Happy error handling!

By [John Gallagher](https://github.com/johngallagher)

Photo by [David Pupaza](https://unsplash.com/@dav420?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/error?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
