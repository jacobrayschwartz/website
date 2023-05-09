---
title: Ruby clamdscan - New Gem Available
taxonomy:
	tag: [Programming, Ruby, ]
media_order: joshua-fuller-XzuJuyYLjmE-unsplash.jpg
date: 5/8/2023
---

In an attempt to learn how Ruby and Ruby Gems development works, I've published a new gem!

[Check it out on rubygems.org](https://rubygems.org/gems/ruby_clamdscan)

This removes the neccessity for Ruby and Ruby on Rails services to have ClamAV or any of its programs installed on the same host. You can now communicate with a separate service. 

Why? Separation of concerns! The fewer items you have installed on your host the better. The only other available gem I was able to find is `clamby`, which isn't the best for what we're looking for in services at my company. 

Please take a look and check it out! Let me know if there are any issues via the Issue Tracker on [Github.](https://github.com/jacobrayschwartz/ruby_clamdscan)

Happy scanning!

