---
layout: post
title: "Dynamic session time(timeout_in) management with Devise"
date: 2015-11-29 18:40:16 -0500
comments: true
categories: 
  - rails
  - devise
  - sessions
  - cookies
  - rememberme
---

From the box Devise has 2 options to manage sessions: `rememberable` and `timeoutable`. `rememberable` utilizes database to store the time when user was signed in [and compares it with current time](https://github.com/plataformatec/devise/blob/v3.2/lib/devise/models/rememberable.rb#L67) - if the difference is more than config's `remember_for` setting the user will be signed out. `timeoutable` first checks [if `rememberable` is being used and not expired](https://github.com/plataformatec/devise/blob/v3.2/lib/devise/models/timeoutable.rb#L29), then [compares current time with last user's request time](https://github.com/plataformatec/devise/blob/v3.2/lib/devise/models/timeoutable.rb#L30) - the difference is then being compared with config's `timeout_in` setting. What if you want to have multiple `timeout_in` settings for different cases?
<!-- more -->

Well, there is an existing solution in devise wiki: [How To: Add timeout_in value dynamically](https://github.com/plataformatec/devise/wiki/How-To:-Add-timeout_in-value-dynamically). The idea is to override `timeout_in` method in the `User` model.
