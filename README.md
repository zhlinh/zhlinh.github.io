# zhlinh.github.io

[![Build Status](https://travis-ci.org/zhlinh/zhlinh.github.io.svg?branch=master)](https://travis-ci.org/zhlinh/zhlinh.github.io)

ZHLINH'S TECH BLOG based on the template [Skinny Bones](https://mmistakes.github.io/skinny-bones-jekyll/).

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


# If got stuck or 'bundler' (2.3.11) required by ... ERROR
# Maybe should change another gem source
$ gem sources
$ gem sources --remove https://rubygems.org/
$ gem sources -a https://mirrors.tencent.com/rubygems/

# Also set config on bundle based project
bundle config mirror.https://rubygems.org https://mirrors.tencent.com/rubygems/
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

> Navigation

`_includes/navigation.html` and `_includes/navigation-sliding.html` retrieve the
links from the definition in `_data/navigation.yml` and `_data/navigation-sliding.yml`.

## LICENSE

The content of this project itself is licensed under the
[Creative Commons Attribution 4.0 license - Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA)](https://creativecommons.org/licenses/by-nc-sa/4.0/legalcode),
and the underlying source code used to format and display that content is
licensed under the [MIT license](https://opensource.org/licenses/mit-license.php).
