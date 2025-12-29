---
layout: single
title: "Hello, Jekyll!"
date: 2025-09-12 10:00:00 +0200
categories: [blogging]
---

My mind is constantly running, and it's a form of cognitive offloading to put these ideas down, almost like placing them in a mental drawer that I can come back
to later on.

### The Problem: Too Many Ideas, Not Enough Focus

My original thought was to build a blog. I figured I would use a service like S3 for hosting. I would write my articles in Markdown and keep everything in Git,
have the process to transform in html and dedicated css.

### Finding a Simpler Way

I found a better solution for hosting and publishing. Instead of S3, I chose GitHub Pages. It's free and designed to automatically build and host static
websites. It's a "push and publish" kind of workflow, which means I can focus on writing instead of on deployment pipelines and server configurations.

For the writing itself, I needed a tool to convert my Markdown files into a website. I looked at a couple of options:

- Jekyll: Built with Ruby, it's a very straightforward tool for generating blogs.

- Hugo: A much faster tool built with Go, designed for massive websites.

The idea of using Hugo and find an excuse to learn Go and distract myself again was appealing. I remember I learnt Ruby when Ruby on Rails was really trendy.

I chose Jekyll because of its simple design and, most importantly, its tight integration with GitHub Pages. It's a perfect match for what I need: a no-fuss way
to get my thoughts out of my head and into a website.

### The Final Setup

The setup was simple. I created a GitHub repository, named it specifically so GitHub Pages would recognize it, and then selected the
no-style-please theme. I chose this theme because it's as minimalist as it gets—it has almost no CSS, which means there are no visual distractions. The focus is
entirely on the words.

I also set up a local development environment with rbenv and Jekyll so I can write offline and see my changes instantly. Now, when an idea pops into my head, I
just write it down, and it's published with a simple git push.

This first post is a perfect example of what this blog is about: taking a complex problem—getting too many ideas out of my head—and finding a simple, effective
solution.

`cat Gemfile`

```ruby
source "https://rubygems.org"
gem "github-pages"
gem "no-style-please"
```

and in `_config.yml` at the root:

```yaml
title: Cognitive offloading
description: Emptying the mind, one article at a time
author: j3r3myfoobar
email: jeremy... (@) gmail.com
theme: no-style-please
```

Posts are written in Markdown files and must be placed in the \_posts directory. or example, \_posts/2025-09-10-hello-world.md.

For example:

```markdown
---
layout: post
title: "Hello, Jekyll!"
date: 2025-09-12 10:00:00 +0200
categories: [blogging]
---

some very important content here ..
```

create an index.md for the root content:

```markdown
---
layout: default
---

# I'm J3r3my.

something important ...

## Posts

{% raw %}
{% for post in site.posts %}

- [{{ post.title }}]({{ post.url }})
  <small>{{ post.date | date: "%Y-%m-%d" }}</small>

{% endfor %}
{% endraw %}
```

Assuming you have ruby install, if you want to see how it looks like on your local machine before publishing run `bundle install` and after run `bundle exec jekyll serve` or simply push and forget when everything has been setup.

PS1: You need to enable Jekyll in the Settings of your repository. It will create a GitHub action for deployment.
