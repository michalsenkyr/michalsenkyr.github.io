---
title: Blogging with Jekyll/GitHub Pages
date: 2017-02-05 15:46
layout: post
category: blog
tags: blog github gh-pages jekyll
---

* content
{:toc}

In this post, I explain how I ended up with my present blogging solution and provide a simple guide to anyone who is interested in using the Jekyll/GitHub Pages combination to publish a site of their own.

## Motivation

As a Scala engineer, I always prized simplicity in all the projects I worked on. I do not like frontend development as I find it tedious and needlessly time-consuming, so I wanted to avoid that. However I also see myself as a "hacker" so I wanted to be able to freely tweak and customize my solution. That is why I didn't choose the insanely popular [WordPress](https://wordpress.com) platform (even though I was tempted by it more than once while setting things up). All the tools had to, of course, be free and open-source, so that I would be able to migrate should the need arise. And lastly, I wanted to be able to integrate my solution with GitHub and share it there.


## GitHub Pages

Some time ago, I was exploring the possibilities of publishing Scaladoc using [sbt](http://www.scala-sbt.org) (I like to use it to build my personal projects) and I stumbled upon the [sbt-site](https://github.com/sbt/sbt-site) plugin. I did not get a chance to use it yet (we are still using Maven at work for Scala projects) but I remembered reading that it supported publishing to [GitHub Pages](https://pages.github.com). When looking into it, I found out that GitHub Pages was a free hosting for GitHub projects' static web pages. GitHub Pages work seamlessly with GitHub repositories by watching changes to the master branch and automatically deploy the changed files.

There are two types of sites GitHub Pages can host:

- Project site
  - Every repository can be configured to be served from the root directory of the _master_ or _gh-pages_ branch or the _docs/_ directory of the _master_ branch
  - Served as **_username_.github.io/_projectname_**
- User/Organization site
  - A special repository called **_username_.github.io** can be served from the root directory of the _master_ branch
  - Served as **_username_.github.io**

GitHub Pages are fully integrated into GitHub and can be enabled/disabled and configured on the project's _Settings_ page.

## Jekyll

Having the means to publish the site is all well and good but we also need to have some means of generating it from the provided content if we do not want to do it by hand all the time. And GitHub Pages has us covered with the integrated support of [Jekyll](https://jekyllrb.com), a very customizable Ruby-based blog-aware static page generator. In fact, the _Theme Chooser_ in the _GitHub Pages_ section of your project's _Settings_ page even generates a simple Jekyll page skeleton (which displays the contents of your _README.md_). You do not even have to install Jekyll on your local machine as GitHub Pages uses it automatically. So the only thing you need is its configuration (which resides in the *_config.yml* file). Sadly, that's where the convenient integration of Jekyll with GitHub Pages ends.

Jekyll itself provides a bunch of useful tools to help you generate pages from content (like posts). Among them are [themes](https://jekyllrb.com/docs/themes) and [templates](https://jekyllrb.com/docs/templates). The idea is that you just select a theme, add your content and the theme uses its templates to generate all your pages. None of the [GitHub Pages supported themes](https://pages.github.com/themes), however, defines templates for posts and the included page templates do not provide any means for navigating through the generated site.

Fortunately, the Jekyll community provides lots of [themes](http://jekyllthemes.org) that do (you can recognize them by the existence of the *_layouts/post.html* file). The downside is that they cannot be simply selected using the _Theme Chooser_ or referenced in the *_config.yml* and thus have to be included in your GitHub project. This immediately excludes all commercial themes. I eventually settled for [HyG's theme](https://github.com/Gaohaoyang/gaohaoyang.github.io) which provided everything I needed (even though it was probably intended mainly for Chinese audiences so I had to modify it quite a bit). Further setup depends on the theme chosen (and every theme uses a different structure) but usually involves editing the *_config.yml* file to provide additional information about the site as well as editing static content (like the _About_ page).

Content pages then have to be properly tagged (Jekyll uses the term [Front Matter](https://jekyllrb.com/docs/frontmatter)) by adding a header to each file in the style of:

```
---
title: The blog post's title
layout: post
categories: category1 category2
tags: tag1 tag2
---
```

Individual posts are then added into the *_posts/* directory with filenames containing the date (for sorting purposes) in the format of `yyyy-MM-dd-name` with the appropriate format extension (usually `md` for [Markdown](https://daringfireball.net/projects/markdown)).

When all that is set, the solution should automatically assemble and publish the entire blog when you push changes into your repository.

## Running it locally

In order to see changes before they are published as well as experiment with new features and customization options, you can fire up Jekyll locally.

To do that, you need to install [Ruby](https://www.ruby-lang.org) and Jekyll. This is how the whole procedure looked like for me on [Arch Linux](https://www.archlinux.org) using [Bundler](https://bundler.io/):

1. Install Ruby using your distro's package manager:

   ```bash
   sudo pacman -Syu ruby
   ```

2. Set your `GEM_HOME` to your Gems' user directory (to keep Bundler from installing your Gems globally) and add its _bin/_ directory to your `PATH`:

   _~/.bashrc_

   ```bash
   export GEM_HOME=$(ruby -e 'print Gem.user_dir')
   PATH=$PATH:$GEM_HOME/bin
   ```

3. Install Bundler:

   ```bash
   gem install bundler
   ```

4. Add a _Gemfile_ to your project and add the _github-pages_ gem to it:

   _Gemfile_

   ```ruby
   source 'https://rubygems.org'
   gem 'github-pages', group: :jekyll_plugins
   ```

5. Install the Gems required by your project:

   ```bash
   bundle install
   ```

6. Run Jekyll in _serve_ mode to generate and serve your site (on localhost:4000 by default):

   ```bash
   bundle exec jekyll serve
   ```

You can make Jekyll run in [drafts mode](https://jekyllrb.com/docs/drafts) by providing the `--drafts` parameter. That way you can create draft posts in your *_drafts/* directory (these do not need to have the date in their filename as their date is determined by their modification time so you always see them first) which are hidden until moved to *_posts/*.

## Conclusion

I must say that I was quite sceptical when first encountering Jekyll. I do not have good experience in setting up Ruby projects (had to set up [Redmine](https://www.redmine.org/) a few times and, even though its a good piece of technology, I found the installation procedure just awful), but Jekyll's setup is actually quite easy. Usage is pretty straight-forward when you understand the basics and the automatic publishing through GitHub Pages is great.

The only bad thing is, sadly, the themes. And that's a major one and I spent quite a bit of time figuring it out. It is not easy to find a good blog-aware theme (or even a usable one) and I find the absence of a default one in GitHub Pages very unfortunate. I would be happy to pay for a commercial one but then I wouldn't be able to use it with GitHub Pages. So here's hoping they fix that in the future.
