---
layout: post
title: "Personal node-red server in Heroku"
date: 2021-05-31 00:00:00 +0200
comments: true
tags: ["Hack", "node-red", "serverless", "heroku"]
publish: true
---
# Why?
Well... why not? I've not been happily hacking since long time ago.
I've been controlled by my agilist alter ego... like if Hulk was asleep by a long dose of Prozac.
Oh, oh, oh!... but the power is back. I feel the geekness traversing my blood torrent.

Heroku gives you 550 hours of free computing per month. Node-red gives you a powerful way to connect things.
There are great rule based systems to automate things out there: Power Automate or IFTTT, for example.
Why them complicating my life? Well, IFTTT and Power Automate both have quite poor free plans, and none of them seem as powerful as node-red.

# Let's start then...
First of all, we will deploy fork [node-red-heroku](https://github.com/joeartsea/node-red-heroku) on github.

Next, we will change readme.md to update the Deploy button with our fork's address.
We could deploy directy from the original repo, but every time the app is deployed, all the existing flows are overriden.
Having our own fork gives a cheap way to backup our flows.

## Deploy to Heroku
Now go to your new repo on github and press the Deploy button. A new app on heroku will be created!
You are ready to make your first "Hello world" flow on your fresh node-red server.

TODO: Make a simple flow
TODO: Protect your repo
