---
layout: post
title:  Moving Towards Multidex
author: Dmytro Voronkevych
date:   2015-11-22
categories: android
---
Android development is full of fun. Even more fun we have when trying to overcome
platform limitations.
One of such joyful and funny part is [`64K methods limit`](http://developer.android.com/tools/building/multidex.html).
If you follow the link, there is a solution to 64K limit, which seems simple  
and sounds easy to go. Our practice showed some complications...

Stage1: Stay out from Multidex
------------------------------
Initially, when we faced 64K limit first time, there were no stable solution yet,
so we carefully analyzed all libraries we use and thrown away some of them.
It helped for some time, then we changed our code generator for data model objects
and got rid of 


Conclusions
-----------
