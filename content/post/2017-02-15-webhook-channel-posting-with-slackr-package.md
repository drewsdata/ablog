---
title: Webhook channel posting with the slackr package
author: Drew Coughlin
date: '2017-02-15'
image: 'webhook.png'
tags:
  - API
  - slackr
  - R
slug: webhook-channel-posting-with-slackr-package
---
A client asked me to develop a method to post a recurring message to a particular Slack channel.  The [slackr package](https://cran.r-project.org/web/packages/slackr/index.html) easily enables inbound webhook Slack channel message posting in just a few lines of R.

```r
library(slackr)

setwd("C:/slackproject/")
slackr_setup(config_file="slackr.dcf")

text_slackr('Any text message you wish to push to the Slack channel goes here.', 
as_user=FALSE)

```


This is the configuration information file referenced in that script:
```r
api_token: <your slack API token>
channel: <name-of-slack-channel>
icon_emoji: :<emoji name>:
username: <you can insert any name here - doesn't have to be a 'real' slack account>
```