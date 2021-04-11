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
## Fork node-red-heroku to have your own backup of your infrastructure.
First of all, we will fork [node-red-heroku](https://github.com/joeartsea/node-red-heroku) on github.
Ok, this is really not necessary... but it will help if we plan to add new nodes to our node-red and redeploy the thing. This repo will contain the backup of our infrastructure, being this infrastructure defined on code.
You could also swallow copy the repo and make it private, but forking it gives you the power to update, while you mantain an easy way to update your code base from the upstream repo.

## Change deployment button
Ok, we have our fork, so let's change readme.md to update the Deploy button with our fork's address.

```
[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy?template=https://github.com/YOUR-GITHUB-HERE/node-red-heroku)
```

## Deploy to Heroku
Now go to your new on github using a browser and press the Deploy button. A new app on heroku will be created!
As part of the deployment process, a new postgress database will be created on heroku and connected to the webapp... here will be stored your data.

With the first login, you will have to create the admin user. Remember to use a secure password and don't store it on a posit on your screen 🤭

> Remember
> - Your git repo hosts the infrastructure as code
> - Your postgress DB on heroku is where your data and flows will reside

## Install plugins


TODO: Make a simple flow
TODO: Protect your repo
TODO: Database
