---
author: doug
comments: true
date: 2012-09-18 17:00:00+00:00
layout: post
slug: xiki-for-the-vim-and-fedora-guy
title: 'Xiki: For the VIM and Fedora guy'
wordpress_id: 219
categories:
- Geekery
---

So yeah, I love the idea of [Xiki](http://xiki.org/) (definitely [check out this screencast](http://www.youtube.com/watch?v=bUR_eUVcABg) for a snapshot of how it works)... But as they say, "You lost me at Emacs."![photocrati gallery](http://dougbtv.com/wp-content/themes/photocrati-theme/photocrati-gallery/image/gallery-placeholder-1.gif)

_* This is a scambaiting trophy from the infamous "Fritz Weisenheimer" a scam-baiting personality. It's just all too applicable now._

So, ehn. As a long time vi guy (and I don't expect this to fully replace vi), I decided to bite the bullet and try it. Also, I'm not a ruby guy. So this Emacs + Ruby stack was like "Uhhh oh." But as the xiki guy says "You don't have to know Emacs." So along I trudged, and eventually I was able to get it to work in Fedora 17, the beefy miracle.

You can follow their [installation tutorial, at the main page of Xiki's github](https://github.com/trogdoro/xiki/) page. And as usual that kind of thing got me "most of the way there." But, I ran into a few snags, and this is the recipe I came up with.

Note! You may have to run some of these commands as root.

    
    [user@host]$ yum install ruby rubygems
    [user@host]$ yum install emacs
    [user@host]$ gem install bundle
    [user@host]$ gem install trogdoro-el4r
    [user@host]$ wget https://github.com/trogdoro/xiki/zipball/master
    [user@host]$ mv trogdoro-xiki-X.Y.Z-stuff.zip xiki.zip
    [user@host]$ unzip xiki.zip
    [user@host]$ mv xiki/ /usr/src/xiki
    [user@host]$ cd /usr/src
    [user@host]$ cd xiki
    [user@host]$ bundle install --system
    [user@host]$ ruby etc/command/copy_xiki_command_to.rb /usr/local/bin/xiki
    [user@host]$ bash etc/install/el4r_setup.sh
    [user@host]$ xiki
    [user@host]$ xiki status
    [user@host]$ xiki restart
    [user@host]$ xiki directory # copy the output from this for the init.rb
    [user@host]$ nano ~/.el4r/init.rb
    # init.rc file:
    # $LOAD_PATH.unshift "{copied directory from above}/lib"
    # require 'xiki'
    # Xiki.init
    #
    # KeyBindings.keys   # Use default key bindings
    # Themes.use "Default"  # Use xiki theme
    [user@host]$ emacs


Now once you're in emacs, give it a try, create a line that reads: "$ ls -l" and then hit alt-enter. Voila! It should print a list from a ls command and allow you to edit that text, etc.

When you're going through this and run into trouble, list your /tmp dir and check out the newest files there -- you'll see a couple logs from el4r, and from xiki. (This didn't exactly help me through the process, but, figured I would note it)

I also realized that "alt+l" (small l, as in Larry) re-loads the el4r, which if xiki bugs out, is handy to use.

Notably, after this whole process... The "pretty theme" didn't load, all the formatting was plain, and I could only use alt-enter, and not double click. I'm not sure -exactly- what fixed it. But in the meanwhile doing other things, I rebooted my laptop, and then fired up emacs again... And! It all worked. I understand this isn't a solution by any means, but, maybe it can trigger in your own brain reading this, what you may need to reset/restart if you'd like to achieve this without a reboot.

<!-- more -->
