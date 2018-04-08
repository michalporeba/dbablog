---
layout: post
title:  "DBA blogging with jekyll on GitHub Pages"
date:   2018-04-07 07:00:00 +0100
categories: jekyll
permalink: "about/jekyll-on-GitHub-Pages"
---

### I decided to start blogging again

I had few prerequisits: 
* the code examples in the posts should look well with syntax highlighting
* the writing, including examples should be easy, preferably in markdown 
* the content shoulb be mine to take and share easily
* it should be possible to easily work on content over a period of time, from multiple devices
* it should be possible to see some trafic data as after all we all have our egos that need to be fed

After reviewing a number of more obvious (at the time) options I have decided to use [jekyll](https://jekyllrb.com), manage the content in a public [GitHub](https://github.com) repo and host the blog on [GitHub Pages](https://pages.github.com/). This post not meant to explain what those tools are, or why I made this decision. It is simply a set of instructions, mostly for myself, how to use jekyll to write and deploy blog posts hosted on GitHub Pages. If you find it helpful too, good for you.

GitHub Pages support jekyll, in many cases all one needs to do is follow [the instructions](https://jekyllrb.com/docs/github-pages/) and enjoy the results. I created the respotiory, gh-pages branch, generated an empty jekyll site, committed, pushed and there it was, my new website satisfying most of my requirements. Then trying to get a custom plugin ([jekyll-analytics](https://github.com/hendrikschneider/jekyll-analytics) for the ego requirement) to work took me much longer than I expected, almost to the point when I started to question my choice. Locally everything would work just as expected, but after pushing to github, the page would update, but the plugin was not available. 

My problem was caused by a little advertised feature: *GitHub Pages supports jekyll*, and the plugins, *but only* plugins which have been *whitelisted* in the [pages-gem](https://github.com/github/pages-gem). The same applies to themes it appears. The full list can be found in the source code [here](https://github.com/github/pages-gem/blob/master/lib/github-pages/plugins.rb)

Here are the steps to work around this limitation:
* create `/docs` folder in your repository
* add `destination: docs/` to the `_config.yml`
* change GitHub Pages configuration of your repository to serve pages from the `/docs` folder 
* when ready to publish new version of the blog, build it locally
* commit and push to master branch

## working with the existing blog

Since we are working with github I assume Git Bash is installed, and that is what I use in the next steps

### setting up a new workstation

A good place to start is [here](https://jekyllrb.com/docs/installation/#requirements). Below are the steps I take. 

1. install [Ruby](https://rubyinstaller.org/downloads/).
    * MSYS2 base
    * MSYS2 system update 
    * MSYS and MINGW development toolchain
2. check versions - (versions at the time of writing)
    * `gcc --version` (5.2.0)
    * `ruby -v` (2.5.1)
    * `gem -v` (2.7.6)
3. install the Bundler gem `gem install bundler`
3. install jekyll `gem install jekyll`
4. restart the git bash
    
### working with an existing blog 

5. go to the folder containing the blog
6. pull the latest version `git pull`
7. install the plugins `bundle install`
8. start local version `bundle exec jekyll serve --watch`

Edit the files as needed, check the changes through a web browser on `http://localhost:4000`. jekyll observes the changes in the files and regenerates the site automatically

### when finished 

9. make sure all old files are removed `bundle exec jekyll clean`
10. build the production version of the blog `JEKYLL_ENV=production bundle exec jekyll build`
11. stage, commit and push