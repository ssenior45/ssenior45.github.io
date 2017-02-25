GIT

Make changes

git status
git diff
git add -A
git status
git commit -m "Test"
git status
git push

If changes don't appear check settings in GitHub -> Webhooks. See if it was delivered.

Check jeykll-webhook is running on server:

forever list

if not

su - stevensenior
cd /home/stevensenior/jekyll-hook/
forever start jekyll-hook.js

------

# Basics of post writing

[Markdown Cheatsheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)<br/>
[Github Flavored Markdown](https://help.github.com/articles/writing-on-github/) explains nuances about GitHub.

Always use ``` for code blocks

If you want the TOC then include this at the start of the post:

`<div class="toc"></div>`

Then headers (h1, h2 etc) become the TOC entries.

**Appears this happens even without the div at the start**

Always start a post with an introductory paragraph. This will appear on the home page of the blog as the excerpt and also will look better when you are social sharing (see below).


# Paragraphs and line spacing

To force a line-break you must enter:

`<br/>`

manually in your post.

Write the post first then proof read and visually inspect it and add any where you see fit is generally the recommended approach.


# Meta-Tags

Referenced [here](http://jekyllrb.com/docs/frontmatter/)

##Essential

```
---
published: true
layout: post
title: "Build The COSMAC ELF A Low-Cost Experimenter’s Microcomputer"
---
```

##Optional

```
---
tags: computing old
categories: generic
date: "YYYY-MM-DD HH:MM:SS"
---
```

##Social Sharing

```
---
share: twitter --twitter-hashtags linkedin
---
```

###LinkedIn

Include the linkedin word in the share tag.

This will add an entry to the [linkedin rss feed](http://stevensenior.co.uk/social-feeds/linkedin.xml)

This is periodically polled by this [IFTTT recipe](https://ifttt.com/myrecipes/personal/32495441) which will post on LinkedIn.


###Twitter

Include the twitter word in the share tag.

Optionally include the --twitter-hastags option after it. This will convert all tag entries to hashtags.

This will add an entry to the [twitter rss feed](http://stevensenior.co.uk/social-feeds/twitter.xml)

This is periodically polled by this [IFTTT recipe](https://ifttt.com/myrecipes/personal/32351145) which will post on Twitter.


# Search

The on-site search function is provided by a free third party service from [Swiftype](https://swiftype.com/)

The service should periodically sweep the site and update the search entries.

You can force a refresh by clicking on the [recrawl](https://swiftype.com/engines/stevensenior-dot-co-dot-uk/domains) button here. You are limited to the amount of manual recrawls you can perform, so use this judicially.
