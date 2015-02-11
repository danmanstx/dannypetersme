---
layout: post
title:  "Fixing a Mac's System Time"
date:   2015-02-09 13:00:00
comments: true
categories: mac server
---

> Quick guide to help others who have a mac who can't seem to keep the system time correct.

### The Problem

Recently at work we were deploying a update to our rails app. One of the requirements was that the app appear offline while our payment processor was offline for a server changeover. 

This was all well and good, and not any big deal to implenment in the rails code, but we did notice something fishy. When we checked the system clock for the Mac server it was over a minute off of the current time of everything else we checked...

> Use `date` to check the system clock


### The Solution

Using the follow command we were able to reset the system clock for the mac from the terminal and have our downtime be exactly when we thought it would be. This uses Apple's time server to correctly set the time. Also when changed it shows the the correct offset, in this case 75 seconds. 

`sudo ntpdate -u time.apple.com`

here it is from our actual server

![weird time is weird]({{ site.url }}/assets/time.png)

Congratulations you can set a clock. 🍻

