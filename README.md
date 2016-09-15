# zhlinh.github.io

[![Build Status](https://travis-ci.org/zhlinh/zhlinh.github.io.svg?branch=master)](https://travis-ci.org/zhlinh/zhlinh.github.io)

ZHLINH'S TECH BLOG based on the template [Skinny Bones](http://mmistakes.github.io/skinny-bones-jekyll/).

![screenshot of Skinny Bones](skinny-bones-template.jpg)

---

## NOTABLE FEATURES

* Stylesheet built using Sass. *Requires Jekyll 2.x*
* Data files for easier customization of the site navigation/footer and for supporting multiple authors.
* Optional Disqus comments, table of contents, social sharing links, and Google AdSense ads.
* And more.

## USAGE

Install:
```
$ bundle install
```

Script to run Jekyll:
```
$ bundle exec jekyll build
$ bundle exec jekyll serve
```

## DEVELOPMENT

### SCAFFOLDING

```
project root
├── _site                               # compiled site ready to deploy
├── _images                             # unoptimized images
├── _includes                           # reusable blocks for _layouts
├── _layouts
|    ├── archive.html                   # archive listing of a group of posts or collection
|    ├── article.html                   # articles, blog posts, text heavy material layout
|    ├── default.html                   # base
|    ├── home.html                      # home page
|    └── media.html                     # portfolio, work, media layout
├── _posts                              # posts grouped by category for sanity
├── _sass
|   ├── vendor
|   |   ├── bourbon                     # Bourbon mixin library
|   |   └── neat                        # Neat grid library
|   ├── _animations.scss                # CSS3 animations
|   ├── _badges.scss                    # small badges
|   ├── _bullets.scss                   # visual bullets
|   ├── _buttons.scss                   # buttons
|   ├── _grid-settings.scss             # Neat settings
|   ├── _helpers.scss                   # site wide helper classes
|   ├── _layout.scss                    # structure and placement, the bulk of the design
|   ├── _mixins.scss                    # custom mixins
|   ├── _notices.scss                   # notice blocks
|   ├── _syntax.scss                    # Pygments.rb syntax highlighting
|   ├── _reset.scss                     # normalize and reset elements
|   ├── _sliding-menu.scss              # sliding menu overlay
|   ├── _variables.scss                 # global colors and fonts
|   ├── css
|   └── main.scss                       # loads all Sass partials
├── fonts                               # webfonts
├── images                              # images
├── js
|   ├── plugins                         # jQuery plugins
|   ├── vendor                          # vendor scripts that don't get combined with the rest
|   ├── _main.js                        # site scripts and plugin settings go here
|   └── main.min.js                     # concatenated and minified site scripts
├── apple-touch-icon-precomposed.png    # 152x152 px for iOS
├── favicon.ico                         # 32x32 px for browsers
└── index.md                            # homepage content
└── _config.yml                         # Jekyll settings
```

### POST

Create new MarkDown (.md) files in `_posts`. You can organize it like
`_posts/category-name/2014-06-01_aticle-name.md`.

The only YAML Front Matter required for posts and pages are title, layout and
permalink, everything else is optional. The following is some examples.

> Title & Layout

```
tilte: About
layout: article
permalink: /about/

title: Latest Posts
layout: home
permalink: /
```

> Modified Date

```
modified: 2014-08-27
modified: 2014-08-27T11:57:41-04:00 # more verbose, also acceptable
```

> Image

```
image:
  feature: feature-image-filename.jpg
  credit: Michael Rose #name of the person or site you want to credit
  creditlink: https://mademistakes.com #url to their site or licensing
```

> Others

```
# Table of contents
toc: true
# Advertisement
ads: true
# Disqus comments
comments: true
```

### PAGE

> Layout

Modify layouts (.html) files in `_layouts`.

> Organization

Maintaining pretty URLs for your site can be handled in two ways when creating
new pages.

Place a `.md` file at the root level and add the appropriate permalink to the
YAML Front Matter.  For example if you want your **About** page to live at
`domain.com/about/` create a file named `/about.md` and add `permalink: /about/`
to its YAML Front Matter.

Or you can create `/about/index.md` and omit the YAML permalink.
Up to you how you’d like to organize your pages.

You can also group pages in a `_pages` folder similiar to `_posts`.

## LICENSE

[The MIT License (MIT)](http://opensource.org/licenses/MIT)
