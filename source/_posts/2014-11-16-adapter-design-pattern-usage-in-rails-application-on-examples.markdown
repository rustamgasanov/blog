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

Design pattern called `Adapter` is used, when we have two objects, which need to communicate to each other, unable to do so, because their interfaces do not match. And the adapter is kind of a bridge between these objects.
In Rails framework, you can find this design pattern is being used a lot. Particularly in `ActiveRecord` adapters are being implemented to communicate with different databases and supply developer with a common interface to use, so you don't bother wheter it is a PostgreSQL or MySQL. In `ActiveJob` adapters are being used to communicate with background job providers, and so on.

#### Examples.

Ok, lets move on to practical cases and code examples.

So, as you understand, it is a good idea to write adapters if we don't want to depend on particular implementation of some functionality(no matter in which entity this functionality is being wrapped: gem, standalone script, etc). 



With this implementation we obtain following benefits:

