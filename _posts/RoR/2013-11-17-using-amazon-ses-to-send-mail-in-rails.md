---
layout: post
title: "Using Amazon SES to send mail in Rails"
description: "Some tips on how to send email in Rails application with Amazon SES(simple email service)"
category: "Ruby on Rails"
tags: [Ruby on Rails, Email Service]
---
{% include JB/setup %}


It took me half an hour to find the working configuration for Amazon SES configuration. I recorded here for reference.

<pre><code>
ActionMailer::Base.smtp_settings = {
  :address              => 'email-smtp.us-east-1.amazonaws.com',
  :port                 => '587', # This is the secret, I had to set the port to 587, otherwise, it won't work!
  :user_name            => 'you user name , assigned by amazon',
  :password             => "you password",
  :domain               => "your.mail.domain",
  :authentication       => :login,
  :enable_starttls_auto => true,
  :openssl_verify_mode  => 'none'
}
</code></pre>
