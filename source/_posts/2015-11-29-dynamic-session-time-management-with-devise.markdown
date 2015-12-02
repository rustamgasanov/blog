---
layout: post
title: "Dynamic session time management with Devise"
date: 2015-11-29 18:40:16 -0500
comments: true
categories: 
  - rails
  - devise
  - sessions
  - cookies
  - rememberme
---

From the box Devise has 2 options to manage sessions: `rememberable` and `timeoutable`. `rememberable` utilizes database to store the time when user was signed in [and compares it with current time](https://github.com/plataformatec/devise/blob/v3.2/lib/devise/models/rememberable.rb#L67) - if the difference is more than config's `remember_for` setting the user will be signed out. `timeoutable` first checks [if rememberable is being used and not expired](https://github.com/plataformatec/devise/blob/v3.2/lib/devise/models/timeoutable.rb#L29), then [compares current time with last user's request time](https://github.com/plataformatec/devise/blob/v3.2/lib/devise/models/timeoutable.rb#L30) - the difference is then being compared with config's `timeout_in` setting. What if you want to have multiple `timeout_in` settings for different cases?
<!-- more -->

Well, there is an existing solution in `devise` wiki: [How To: Add timeout_in value dynamically](https://github.com/plataformatec/devise/wiki/How-To:-Add-timeout_in-value-dynamically). The idea is to override `timeout_in` method in the `User` model, if it's enough for your purposes you can stop reading here. In my case I wanted to have a different session time based on the url which user uses(long session for regular application sign in, short session for embedded app version), so the implementation should be independed from `User` model. There is no built in solution for this case in `devise`, so I decided to build my own.

### Solution

First of all, I checked [warden manager in devise for timeoutable](https://github.com/plataformatec/devise/blob/v3.2/lib/devise/hooks/timeoutable.rb) and removed this feature from the `User` model to implement my own. Here is how it looks like:

``` ruby config/initializers/warden.rb
Warden::Manager.after_set_user do |record, warden, options|
  scope = options[:scope]
  env   = warden.request.env

  if record && warden.authenticated?(scope)
    last_request_at = warden.session(scope)['last_request_at']

    if last_request_at.is_a? Integer
      last_request_at = Time.at(last_request_at).utc
    elsif last_request_at.is_a? String
      last_request_at = Time.parse(last_request_at)
    end

    proxy = Devise::Hooks::Proxy.new(warden)

    session_valid_for = warden.env['rack.session'][:session_valid_for]

    if session_valid_for.present? && Time.now.utc.to_i - last_request_at.to_i > session_valid_for.to_i
      Devise.sign_out_all_scopes ? proxy.sign_out : proxy.sign_out(scope)
      throw :warden, scope: scope, message: :timeout
    end

    unless env['devise.skip_trackable']
      warden.session(scope)['last_request_at'] = Time.now.utc.to_i
    end
  end
end
```

All that's needed to manipulate the session time is to use 2 session parameters: `last_request_at`, which is being updated on each request and `session_valid_for` which I set up during login in `SessionsController#create`. I used a `super` call with a block to make it work:

``` ruby app/controllers/sessions_controller.rb
class SessionsController < Devise::SessionsController
  ...
  def create
    super do
      if params[:layout] == 'embedded'
        session[:session_valid_for] = SessionTimeManagerService.short
      else
        session[:session_valid_for] = SessionTimeManagerService.standard
      end
    end
  end
  ...
end
```

In order to respect SRP, I moved session time related logic into separate service, but of course the easiest way is to set it explicitly like `session[:session_valid_for] = 5.minutes/1.hour` or using environment variables: `session[:session_valid_for] = ENV['session_time_short]/ENV['session_time_standard']`.

One remaining issue is that existing signed in users don't have `session_valid_for` variable in their sessions, so they would never be signed out. The easy fix is to set it in `ApplicationController` `before_filter` which is being executed before any call to any controller action:

``` ruby app/controllers/application_controller.rb

class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  before_action :ensure_session_valid_for_set
  ...
  def ensure_session_valid_for_set
    if session[:session_valid_for].blank? && user_signed_in?
      session[:session_valid_for] = SessionTimeManagerService.standard
    end
  end
  ...
end
```
