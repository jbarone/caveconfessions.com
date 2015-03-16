+++
date = "2015-03-15T10:35:29-05:00"
draft = false
title = "Ghost to Hugo"
tags = ["Ghost", "Hugo", "Blog"]
+++

I recently converted my blog from using [Ghost](https://ghost.org/ "Ghost") to
[Hugo](http://gohugo.io/ "Hugo"). I made the decision to do this for a few
reasons, which I'll explain below. As well as the process I used to do the
conversion.

Why Leave Ghost?
---------------

Don't get me wrong. I really liked Ghost. I think it was constructed really
well with bloggers in mind. It comes with extremely pretty, functional,
responsive designs that just work right out of the box. I found Ghost really
easy to manage and using markdown for the posts just made writing fun. So why
am I leaving? Well the biggest reason for me was security. I don't really have
time to keep up with my server the way that I wanted and needed to make sure
that I have all the latest and greatest patches for node.js and Ghost. So I
made the decision to go with a static site generator instead.

With a static site generator, I'm only serving up static html and resources.
This eliminates a lot of possible vulnerabilities and need for maintenance out
of the gate. So I found this very appealing. And hence the move.

Why Hugo?
---------

I'll be honest, Hugo was not the first static site generator that I tried. My
first attempt was really with using [Nikola](http://getnikola.com/). Being a
huge fan and daily user of Python this just felt like the natural choice. I
quickly decided though that this was not the platform for me. I found it a bit
hard to get my head around, and even harder to port my current blog design
over. Themes really seem to be where Nikola is lacking. I think if there was a
better repository for them it might be a better choice. Though while trying to
find a port of the Casper theme, the theme used by ghost, I discovered a
[ported theme](https://github.com/vjeantet/hugo-theme-casper) for Hugo. So this
led me to looking into Hugo. First thing I found is that it was written in Go.
This was pretty awesome since I have started learning and working with Go and
really just falling in love with it. I installed it and tried it out. It just
worked and felt right.

The Move
--------

I had been using Ghost for a while and so had a few posts that needed to move
with me to Hugo. To make that happen I had to get creative. I found a post on
the blog of the author of the Hugo Casper theme that talked about moving from
Ghost to Hugo so I read it. I found that he had used php script to take the
data from the Ghost export and create the markdown files used by Ghost. This
seemed like an easy enough proposal. However, I'm not really a big fan of php
(a long story for another time). So I figured I would do a quick port of the
script to Python so I could get up and running quickly. The php script on the
blog didn't work without some help, the format of the json that Ghost exported
seemed to have changed since the script was created. After a few minutes and
few false starts I created the script:

    #!/usr/bin/env python

    import json
    import datetime

    with open('GhostData.json', 'rb') as data_file:
        ghost_data = json.load(data_file)

    posts = ghost_data['db'][0]['data']['posts']

    for post in posts:
        created_at = datetime.datetime.fromtimestamp(
            post['created_at']/1000).isoformat()
        title = post['title']
        slug = post['slug']
        markdown = post['markdown']
        draft = 'false' if post['published_at'] else 'true'
        published_at = (
            datetime.datetime.fromtimestamp(
                post['published_at']/1000).isoformat()
            if post['published_at']
            else datetime.datetime.fromtimestamp(
                post['created_at']/1000).isoformat()
        )

        with open('output/%s.md' % slug, 'w') as post_file:
            post_file.write('+++\n')
            post_file.write('date = "%s"\n' %
                            published_at.encode('utf8'))
            post_file.write('draft = "%s"\n' % draft.encode('utf8'))
            post_file.write('title = "%s"\n' % title.encode('utf8'))
            post_file.write('slug = "%s"\n\n' % slug.encode('utf8'))
            post_file.write('+++\n\n')
            post_file.write(markdown.encode('utf8'))

After that, simply moved the files from `output` to `content/post` and run
`hugo`, and BAM I had a blog.

Publishing
----------

With the move to the static site, I also decided to make publishing my blog
posts easier. To accomplish this I setup a git repository on my server and
configured a post-receive hook that would deploy my site for me.

    #!/bin/bash

    DEPLOY_DIR=<deploy directory>

    GIT_WORK_TREE="$DEPLOY_DIR" git checkout -f

    cd "$DEPLOY_DIR" && hugo -t casper --uglyUrls=true

This means when I'm ready to publish I simply do a `git push deploy`. It's
awesome!

What's Next?
------------

I decided that even though my one off python script worked, it didn't bring
everything over. It missed the tags, for example. Also, it was Python and not
Go, this felt out of spirit with the whole change. So with that in mind, I'm
working on a more comprehensive conversion tool in Go. I should have a write up
all about it soon.

I also think that I should look into setting up a cron job on my server that
can run every night and rebuild my site, in case there was post that was setup
with a scheduled post date, future date.
