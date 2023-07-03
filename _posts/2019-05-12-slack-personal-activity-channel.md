---
layout: post
excerpt_separator: <!--more-->
title: "Slack tip: personal activity channel"
permalink: slack-personal-activity-channel/
date: 2019-05-12
---

I'm a big fan of using the [Slack Github integration](https://slack.github.com/) to get Github notifications for projects I'm involved with

Typically in the past, each project has set up its own project-specific "activity channel" in Slack eg `#project-abc-activity` and then someone sets up all the automated integrations for the project there (Github, Zeplin, ...)

This works ok, but there have been differing opinions about eg the desired level of verbosity/frequency of notifications on that channel

### Personalize it
Since there's typically no human interaction on these activity channels (they're "read-only"), why not just have your own personal version of that activity channel where you can tweak the configuration to your heart's content?

That's the pattern I've landed on: a `#julian-activity` channel where I configure notifications for projects I'm currently involved in

Then as I move between projects, that `#julian-activity` channel remains the single home for such notifications - I just add/remove integrations to that channel. No creating new channels every time a new project starts (and then eventually gathering dust once a project is completed since noone usually makes the call to go ahead and delete the shared channel)

### Github integration tips :octocat:

Once the Slack Github integration is installed, turning on and configuring notifications for a new project is super easy. To start notifications for a new Github repo just type in your Slack activity channel:
```
/github subscribe repo-owner/repo-name
```

This will turn on notifications for issues, pull requests, statuses, commits, deployments, and public by default (per the [docs](https://get.slack.help/hc/en-us/articles/232289568-GitHub-for-Slack))

I like to tweak this a little bit and also get notifications for reviews, comments, and branches. So I'd typically then type a second command:
```
/github subscribe repo-owner/repo-name reviews comments branches
```
(You can probably do that all as a single initial command, but this gets the job done)

Then once I'm no longer involved in a project, I'll go to my activity channel and type:
```
/github unsubscribe repo-owner/repo-name
```
Boom! unsubscribed
