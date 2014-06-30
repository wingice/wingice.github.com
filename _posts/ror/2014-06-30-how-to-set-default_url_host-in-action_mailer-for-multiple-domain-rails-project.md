---
layout: post
title: "How to set default_url_host in action_mailer for multiple domain rails project"
description: ""
category: "RoR"
tags: [RoR]
---
{% include JB/setup %}


My solution is borrowed from  [https://github.com/rails/rails/issues/4099](https://github.com/rails/rails/issues/4099).
I tested it , it works like a charm.

I create a parent-class to reuse the default_url_options function. 


{% highlight ruby %}
class DynamicBaseMailer < ActionMailer::Base
  default  from: Proc.new { choose_mail_from()}
  
  def default_url_options         
    { :host => choose_host() }
  end

  protected
  def choose_mail_from
    .... # you code here
  end

  def choose_host
    ...... # you code here
  end
end
{% endhighlight %}

