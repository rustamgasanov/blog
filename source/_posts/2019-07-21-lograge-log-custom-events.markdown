---
layout: post
title: "Lograge log custom events"
date: 2019-07-21 09:08:11 +0000
comments: true
categories:
  - ruby
  - rails
  - lograge
  - log
  - logging
  - tutorial
  - example
---

The initializer below sets up the lograge but doesn't allow you to use `Rails.logger.*severity*`.

<!-- more -->

```ruby
# config/initializers/lograge.rb

# frozen_string_literal: true

Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.keep_original_rails_log = true
  config.lograge.formatter = Lograge::Formatters::Json.new
  config.lograge.logger = ActiveSupport::Logger.new(STDOUT)
  config.lograge.ignore_actions = [
    'ActiveStorage::DiskController#show',
    'ActiveStorage::BlobsController#show',
    'ActiveStorage::RepresentationsController#show'
  ]
end
```

Let's create a separate class for this purpose:

```ruby
# lib/log.rb

# frozen_string_literal: true

# Adds logger Log.*level* methods
module Log
  extend self

  [:debug, :info, :warn, :error, :fatal, :unknown].each do |severity|
    define_method severity do |message, params = {}|
      raise ArgumentError, "Hash is expected as 'params'" unless params.is_a?(Hash)
      logger.public_send(severity, {
        m: message
      }.merge(params).to_json)
    end
  end

  private

  def logger
    Lograge.logger
  end
end
```

Now it is possible to write logs with:

```ruby
Log.info 'SMS received', from: from, body: body, message_id: message_id
```

[See how to stream your logs to Google Cloud in Part 2](http://rustamagasanov.com/blog/2019/07/19/stream-lograge-and-rails-logs-to-google-cloud-stackdriver/)
