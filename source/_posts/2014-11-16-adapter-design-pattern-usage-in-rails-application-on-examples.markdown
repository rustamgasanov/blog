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

Design pattern called `Adapter` is used, when we have two objects, which need to communicate to each other, but can't do so, because their interfaces do not match. And the adapter is kind of a bridge between these objects. In Rails framework, you can find this design pattern used a lot. Particularly in `ActiveRecord` to communicate between application and different kinds of databases, in `ActiveJob` to communicate between application and different kinds of background job providers.
