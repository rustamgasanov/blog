---
layout: post
title: "Stream Lograge and Rails logs to Google Cloud Stackdriver"
date: 2019-07-22 10:52:20 -0300
comments: true
categories:
  - google_cloud
  - stackdriver
  - ruby
  - rails
  - lograge
  - logging
  - tutorial
  - example
---

[Part 1 about logging custom events with lograge](http://rustamagasanov.com/blog/2019/07/21/lograge-log-custom-events/)

`stackdriver` gem contains the bundle of monitoring tools

- google-cloud-debugger
- google-cloud-error_reporting
- google-cloud-logging
- google-cloud-trace

If you don't plan to use all of them, but only work with logging - install `google-cloud-logging` alone.

Before you begin - you need to navigate to the Google Cloud Console and create a `IAM & admin -> Service Account` with a Logging Admin rights. Then navigate to `IAM & admin -> IAM` and make sure the service account has a role Logging Admin, if not - click `Add`.

<!-- more -->

## Stream `rails` logs to Google Cloud

You can bind a default Rails logger to Google Cloud quite easily, add those lines to `config/application.rb`:

```ruby
require 'google/cloud/logging'
...
credentials = Google::Cloud::Logging::Credentials.new(JSON.parse(*service_account_key*))

config.google_cloud.project_id = *project_id*
config.google_cloud.keyfile = credentials # or *path_to_a_keyfile*
config.google_cloud.use_logging = true # if you want to send non-production logs
...
```

The `google-cloud-logging` gem if well-documented, there are examples for how to use [Credentials](https://googleapis.github.io/google-cloud-ruby/docs/google-cloud-logging/latest/Google/Cloud/Logging/Credentials.html) or [Logger](https://googleapis.github.io/google-cloud-ruby/docs/google-cloud-logging/latest/Google/Cloud/Logging/Logger.html).

If logs do not appear in the Google Cloud, you might have an incorrect rights, to check what went wrong debug the logger writer:

```ruby
$ bundle exec rails c
> Rails.logger # Must be Google Cloud instance
> Rails.logger.info 'test'
...
> Rails.logger.writer.last_exception
```

## Stream `lograge` logs to Google Cloud

### Configure the `lograge` formatter

The set of existing formatters for this gem is located here: [lib/lograge/formatters](https://github.com/roidrage/lograge/tree/master/lib/lograge/formatters). What we'd like to do is to stream `jsonPayload` to the cloud, however if you select `Lograge::Formatters::Json.new` as a formatter - lograge will send a `textPayload`. That's because this formatter does `JSON.dump(message)` which transforms a `Hash` to a json `String`(which is cool if you want to send it to `STDOUT`) and that string is being sent to the cloud. To have a `jsonPayload` in the cloud - you need to send a `Hash` without any transformations. So let's write our own formatter:

```ruby
# lib/lograge/formatters/json_custom.rb

# frozen_string_literal: true

module Lograge
  module Formatters
    class JsonCustom
      # @param data [Hash] Contains the log message as key-values.
      #   In development-like environments we’d like to convert
      #   this hash to JSON string to display in STDOUT/file/wherever.
      #   In production-like environments we’d like to send it raw
      #   to the Google Cloud, which interprets it as JSON payload.
      def call(data)
        if Log::GCLOUD_ENVIRONMENTS.include?(Rails.env.to_sym)
          data
        else
          JSON.dump(data)
        end
      end
    end
  end
end
```

### Configure lograge custom events class to send JSON to STDOUT and Hash to GCloud

```ruby
# lib/log.rb

# frozen_string_literal: true

# Adds logger Log.*level* methods
module Log
  extend self

  GCLOUD_ENVIRONMENTS = [:production, :staging].freeze

  [:debug, :info, :warn, :error, :fatal, :unknown].each do |severity|
    define_method severity do |message, params = {}|
      raise ArgumentError, "Hash is expected as 'params'" unless params.is_a?(Hash)
      payload = {
        m: message
      }.merge(params)
      unless GCLOUD_ENVIRONMENTS.include?(Rails.env.to_sym)
        payload = payload.to_json
      end
      logger.public_send(severity, payload)
    end
  end

  private

  def logger
    Rails.application.config.lograge.logger
  end
end
```

### Setting up Google Cloud auth and lograge

The following initializer sets up GCloud streaming for `Log::GCLOUD_ENVIRONMENTS`(which typically is a `[:production, :staging]`) and `STDOUT` streaming for other envs. `:credentials` is a GCloud JSON key. I prefer to ignore any `ActiveStorage` logs as they aren't useful.

```ruby
# ./config/initializers/lograge.rb

# frozen_string_literal: true

if Log::GCLOUD_ENVIRONMENTS.include?(Rails.env.to_sym)
  require 'google/cloud/logging'

  google_cloud_config = Rails.application.credentials.dig(:google_cloud, Rails.env.to_sym)

  credentials = Google::Cloud::Logging::Credentials.new(
    JSON.parse(google_cloud_config[:credentials])
  )

  logging = Google::Cloud::Logging.new(
    project_id: google_cloud_config[:project_id],
    credentials: credentials
  )

  resource = logging.resource('gae_app', module_id: '1')
  logger = logging.logger(Rails.env, resource, env: Rails.env.to_sym)
else
  logger = ActiveSupport::Logger.new(STDOUT)
end

Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.keep_original_rails_log = true
  config.lograge.formatter = Lograge::Formatters::JsonCustom.new
  config.lograge.logger = logger
  config.lograge.ignore_actions = [
    'ActiveStorage::DiskController#show',
    'ActiveStorage::BlobsController#show',
    'ActiveStorage::RepresentationsController#show'
  ]
  config.lograge.custom_options = lambda do |event|
    {
      user_id: event.payload[:user_id]
    }
  end
end
```

### Adding user_id to payload

```ruby
# controllers/application_controller.rb

def append_info_to_payload(payload)
  super
  payload[:user_id] = current_user&.id
end
```
