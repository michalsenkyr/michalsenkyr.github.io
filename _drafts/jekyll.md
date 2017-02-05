---
title: Blogging with Jekyll/GitHub Pages
layout: post
category: blog
tags: blog github
---

In this post, I will explain how I ended up with my present blogging solution and provide a simple guide to anyone who is interested in using it.

## Motivation

As a Scala engineer, I always prized simplicity in all the projects I worked on. I do not like frontend development as I find it tedious and needlessly time-consuming, so I wanted to avoid that. However I also see myself as a "hacker" so I wanted to be able to freely tweak and customize my solution. That is why I didn't choose the insanely popular [WordPress](https://wordpress.com) platform (even though I was tempted by it more than once later on). All the tools had to, of course, be free and open-source, so that I would be able to migrate should the need arise. And lastly, I wanted to be able to integrate my solution with GitHub and share it there.

## GitHub Pages

Some time ago, I was exploring the possibilities of publishing Scaladoc using [sbt](http://www.scala-sbt.org) (I like to use it to build my personal projects) and I found the [sbt-site](https://github.com/sbt/sbt-site) plugin. I did not get a chance to use it yet (we are still using Maven at work for Scala proejcts) but I remembered reading that it supported publishing into [GitHub Pages](https://pages.github.com). Turns out it was a free hosting for GitHub projects' static web pages.

GitHub Pages work seamlessly with GitHub repositories by watching changes to the master branch and automatically deploying the changed files.
