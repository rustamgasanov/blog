---
layout: post
title: "How to use ERB with ECO"
date: 2015-11-04 19:39:31 -0500
comments: true
categories: 
  - erb
  - eco
  - rails
  - backbone
  - marionette
  - templates
---

In my current project we use Rails with Backbone(Marionette). For Rails, it's essential to use `.erb`. For Backbone, we use `.jst.eco`. Everything works fine unless you need Ruby helpers(`asset_url`/`asset_path` for example) in JavaScript templates. Here I'll show how to combine the power of `erb` and `eco` template engines.

<!-- more -->

Say, you need to put a link to compiled asset in your JavaScript template.

#### Solution 1

Add `.erb` to your `.js(.coffee)` Marionette View object and generate `asset_url` in `templateHelpers`, then use generated url in the template:

``` coffeescript new_view.js.coffee.erb
class New.Api extends Marionette.ItemView
  ...
  templateHelpers: ->
    apiUrl: "<%= asset_url('API_Documentation_v1.0.pdf') %>"
  ...
```

``` html api.jst.eco
...
<div>
  <p><a href="<%= @apiUrl %>">Link to API.</a></p>
</div>
...
```

