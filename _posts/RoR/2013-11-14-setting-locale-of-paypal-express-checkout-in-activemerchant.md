---
layout: post
title: "Setting locale of Paypal express checkout in activemerchant"
description: "How do you setting locale of Paypal express checkout in activemerchant "
category: "Ruby on Rails"
tags: [Ruby on Rails]
---
{% include JB/setup %}

Recently I had a trouble during debug a website which use Paypal express checkout. The checkout page always redirected to China Paypal, not Paypal.

I finally got a solution for Paypal support engineer, so I recorded here if
someone else encounter the same issue.
Based on Paypal express checkout integration guide, the LOCALECODE should be
set to 'US', but in activemerchant, you can't set it to this directly.

Apparently, ActiveMerchant did some translation for us, setting this in your
express_checkout_options:

<!--more-->

<pre><code>
 :locale => 'US'
</code></pre>

This would work!
Enjoy!,  :-)


