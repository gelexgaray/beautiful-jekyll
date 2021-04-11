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

First of all, we will fork [node-red-heroku](https://github.com/joeartsea/node-red-heroku) on Github.
Ok, this is really not necessary... but it will help if we plan to add new nodes to our node-red and redeploy the thing. This repo will contain the backup of our infrastructure, being this infrastructure defined on code.
You could also swallow copy the repo and make it private, but forking it gives you the power to change, while you maintain an easy way to update your code base from the upstream repo.

## Change deployment button

Ok, we have our fork, so let's change readme.md to update the Deploy button with our fork's address.

```
[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy?template=https://github.com/YOUR-GITHUB-HERE/node-red-heroku)
```

## Deploy to Heroku

Now go to your new repo on Github using a browser and press the Deploy button. A new app on Heroku will be created!
As part of the deployment process, a new Postgress database will be created on Heroku and connected to the webapp... here will be stored your data.

### Point your browser to your brand new Heroku app

```
http://YOUR-NODE-RED.herokuapp.com
```

With the first login, you will have to create the admin user. Remember to use a secure password and don't store it on a posit beside your screen 🤭

> Remember
> - Your git repo hosts the infrastructure as code
> - Your Postgress DB on Heroku is where your data and flows will reside

## Install plugins

Node-red has its own plugin installation screen (Menu/Manage palette)... this is good for testing, but remember: if you redeploy your infrastructure, you will lost the changes.

### Backup your nodes in the definition of your infrastructure

I use Twitter and Telegram connectors, so I will edit package.json in the root folder of the github repo and add *node-red-node-twitter* and *node-red-contrib-telegrambot*

```
{
    "name"         : "node-red-heroku",
    "version"      : "0.0.2",
    "dependencies": {
        "when": "~3.x",
        "pg": "^8.3.0",
        "nano": "~5.11.0",
        "feedparser":"~0.19.2",
        "redis":"~0.10.1",
        "node-red": "~1.x",
        "node-red-node-twitter": "~1.x",
        "node-red-contrib-telegrambot": "~9.x"
    }
}
```

## Redeploy the thing

We could press again the deploy button, but there is a smarter way: we could bind permanently our Heroku app to our Github repo to easily redeploy from Heroku.

### Bind Heroku to Github

On Heroku, go to your application's admin page and:

- Go to *Deploy* menu
- Find the *Deployment method* option
- Select *Github*

Authenticate on Github, and select the repo and branch to be deployed.

## Deploy, deploy, deploy!

Now, you will have two options
- Deploy on demand, pressing the Deploy button on Heroku
- Setup an automatic deployment, to deploy on Heroku every time the source branch is changed

# Bye-bye, by now...
And here we arrive to the finish of our recipe. Happy coding on your shiny new node-red!