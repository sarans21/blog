---
title: Hello, Friend
date: '2022-08-16'
---
> *"Hello, friend. Hello, friend? That's lame."*  
>   
> &mdash; Elliot Alderson, Mr. Robot

Welcome to my shiny new blog!

## How I Create This Blog?

This blog is built using my own static blog engine, called [blogger](), written from scratch in Ruby.

In development, it is like this:

```sh
$ blogger init blog
```

This creates a new directory named `blog` with this structure:

```
blog/
  _layouts/
    stylesheets/
      blog.css
      sanitize.css
      syntax.css
    index.haml
  _assets/
    logo.png
  _contents/
    articles/
      hello.md
```

Then,

```sh
$ blogger serve .
```

The engine will listen on `http://localhost:8080` with `LiveReload` enabled. Edited Markdown contents in `_contents` get automatically re-rendered with layouts and reloaded in the browser.

### Deployment

```sh
$ blogger build --scheme https --host sarans.co --port 443 .
```

This creates a static site in `_public` directory, ready to be deployed anywhere.

## Blog Engine Implementation

Layouts ([Haml](https://haml.info/)) and contents ([Markdown](https://daringfireball.net/projects/markdown/)) rendering uses [tilt](https://github.com/rtomayko/tilt) and [redcarpet](https://github.com/vmg/redcarpet) gem respectively. Watching for changes (in `serve` command) for re-rendering is implemented by using the [listen](https://github.com/guard/listen) gem for listening to file system notifications.

The CSS is hand-written without a framework in `_layouts/stylesheets/blog.css` using pure CSS code because I'm too lazy to make live SASS compiling and reloading work. The theme is based on my own unpublished Sublime theme, which in turn was based on [ayu](https://github.com/dempfi/ayu).

Code highlighter uses [rouge](https://github.com/rouge-ruby/rouge), which is compatible with Pygments.

Browser live reloading uses [LiveReload-JS](https://github.com/livereload/livereload-js) in client-side. Server-side is written based on [EventMachine WebSocket](https://github.com/igrigorik/em-websocket) following [LiveReload protocol version 7](http://livereload.com/api/protocol/), which I borrowed the code from Jekyll's `LiveReloadReactor`.

The CLI is built using [Thor](http://whatisthor.com/).

Front Matter parsing is hand-written using standard Ruby's `String#split` and [yaml](https://github.com/ruby/yaml) module.

The whole engine is [<300 LOC](https://github.com/sarans21/blogger) in Ruby.

## What's Next?

There are so many more things to do with Blog (SEO, custom pages, analytics, and comments to name a few). I will only add things as I needed.

Until next time!