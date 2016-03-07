---
layout: post
title: "Adapter Design Pattern Usage In Rails Application On Examples"
date: 2014-11-16 19:47
comments: true
categories: 
  - rails
  - design patterns
  - adapter
  - example
---

#### Introduction.

Design pattern called `Adapter` is used, when there are two or more objects, which need to communicate to each other, but unable to do so, because their interfaces do not match. And the adapter is kind of a bridge between these objects.
In Rails framework this design pattern is being heavily used. Particularly, in `ActiveRecord`, adapters are implemented to communicate with different databases and supply developer with a common interface to use, so you don't bother whether it is PostgreSQL, MySQL or any other database. In `ActiveJob` adapters are used to communicate with background job providers, and so on.

#### Examples.

Ok, let's move on to practical cases and code examples.
<!-- more -->
It is always a good idea to write adapters if you don't want to depend on particular implementation of some functionality(no matter in which entity this functionality is being wrapped: gem, standalone script, etc).

#### 1

Say, you want to calculate pages count in pdf file, and you've read about simpliest solution: to use `prawn` gem. Let's implement autonomous service that we will use in application:


``` ruby app/services/pdf_generation/document.rb
module PdfGeneration
  class Document
    attr_reader :document
    delegate :page_count, to: :document

    def initialize(options = {}, adapter = self.class.default_adapter)
      @document = adapter::Document.new(options)
    end

    class << self
      def generate(filename, options = {}, adapter = default_adapter, &block)
        adapter::Document.generate(filename, options, &block)
      end

      def default_adapter
        PdfGeneration::PrawnAdapter
      end
    end
  end
end
```

And corresponding `prawn adapter`, which will be default for this service:

``` ruby app/services/pdf_generation/prawn_adapter/document.rb
module PdfGeneration
  module PrawnAdapter
    class Document
      attr_reader :document
      delegate :page_count, to: :document

      def initialize(options = {})
        @document = Prawn::Document.new(options)
      end

      class << self
        def generate(filename, options = {}, &block)
          Prawn::Document.generate(filename, options, &block)
        end
      end
    end
  end
end
```

With this implementation we can now call

```
page_count = PdfGeneration::Document.new(:template => @document.path).page_count
```

anywhere in the app to calculate pdf-document pages count. The service we built is totally autonomous and independent from `prawn`. If the other day you decide to use another gem, you won't need to change any single line of the code, all you will have to do is to add a new adapter. Cool, huh?

### 2

Let's imagine that one of the core features of your application is communication with social networks, consider `twitter`. In different parts of your app, you want to create statuses, update statuses and perform other interactions. Of course you would want to add twitter gem and use it's functionality all over the app. But there are plenty of caveats in this approach, particularly:

  * As your app will grow, Twitter API will also be improved, maybe version will be changed, new functionality added, etc
  * Twitter gem you are using, will, of course, also be upgraded
  
And some day you will decide to upgrade gem version in order to gain new or improved functionality. The gem can be rewritten significantly since the day you started using it. Imagine the amount of work you will have to do, in order to upgrade for new version: check initilization, check all existing calls, rewrite tests related to different parts of your app, that are using this gem, etc. But in the same time, existing code should do the same, you still want to create statuses/update statuses/etc and maybe add some new calls to Twitter API, which weren't available in the previous version.

Now, let's see how adapter design pattern can rescue you from all this pain related to maintaining existing code which should do same things, and how easy it would be to migrate to a new gem version and to add new features.

First of all we want to create autonomous twitter service:


``` ruby app/services/social_networks/twitter_network/twitter_service.rb
module SocialNetworks
  module TwitterNetwork
    class TwitterService
      attr_reader :token, :secret, :adapter
      delegate :tweet_destroy, :tweet_create, to: :adapter

      def initialize(options, adapter = nil)
        @token, @secret = options[:token], options[:secret]

        @adapter = (adapter || default_adapter).new(token, secret)
      end

      private
      def default_adapter
        SocialNetworks::TwitterNetwork::Adapters::TwitterGemV48Adapter
      end
    end
  end
end
```

Next, we need to implement adapter specific to the `twitter gem` version we are currently using, with methods we need in app:

``` ruby app/services/social_networks/twitter_network/adapters/twitter_gem_v48_adapter.rb
module SocialNetworks
  module TwitterNetwork
    module Adapters
      class TwitterGemV48Adapter
        attr_reader :token, :secret

        def initialize(token, secret)
          @token, @secret = token, secret
          init_connection
        end

        def tweet_destroy(tweet_id)
          client.tweet_destroy(tweet_id)
        end

        def tweet_create(tweet_text, options = {})
          client.update(tweet_text, options)
        end

        private
        def init_connection
          client.verify_credentials
        end

        def client
          @client ||= Twitter::Client.new(
            oauth_token: token,
            oauth_token_secret: secret
          )
        end
      end
    end
  end
end
```

Now, having this implemented, in application we can use 
```
@service = SocialNetworks::TwitterNetwork::TwitterService.new({ token: token, secret: secret })
@service.tweet_destroy(tweet_id)
@service.tweet_create(content, options)
```

Now we have all the stuff related to interaction with the gem(and thus with Twitter API) in one place. It is now easy to add new methods or to upgrade a gem version, since all we will have to do is to add new adapter. All the calls in the app to the service will remain the same.

### Conclusion

Adapter design pattern usage in Rails app makes your life, as a developer, easier. Particularly, you obtain following benefits:

  * Maintainable code
  * Encapsulated logic
  * Easy logic extension/enhancement


