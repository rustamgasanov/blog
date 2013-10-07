---
layout: post
title: "Custom validation for specific action"
date: 2013-10-07 00:32
comments: true
categories: 
  - rails
  - validation
---
Recently I have faced with a situation when there was a need to build custom validation for a single controller action. Let's say we have a `User` model with a `role` attribute. Administrator should have an ability to create a users(in protected area) with any role. And any guest should have an access to the public registration page, where he can select his role from the limited list of the registerable roles. So how can we implement a validation, which will work correctly for both administrator and public user registration pages? My solution described below.
<!-- more -->
``` ruby User model
class User < ActiveRecord::Base
  validate :ensure_correct_role

  class << self
    def all_roles
      # here goes a full list of available roles  
    end

    def protected_roles
      # and this is protected list of roles which should not be registerable through public page
    end

    def registerable_roles
      all_roles - protected_roles
    end
  end

  def restrict_roles
    @roles_restricted = true
  end

  def roles_restricted?
    @roles_restricted
  end

  private
  def ensure_correct_role
    role_valid = if roles_restricted?
      role.in?(User.registerable_roles)
    else
      role.in?(User.all_roles)
    end
    errors.add(:role, 'is invalid') unless role_valid
  end
end
```

``` ruby Public registration controller
...
def create
  @user = User.new(params[:user])
  @user.restrict_roles
  
  if @user.save
    # valid registration handeling
  else
    render :new
  end
end
...
```
