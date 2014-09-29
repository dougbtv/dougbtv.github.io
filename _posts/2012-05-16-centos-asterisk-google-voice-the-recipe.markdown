---
author: doug
comments: true
date: 2012-05-16 23:12:33+00:00
layout: post
slug: centos-asterisk-google-voice-the-recipe
title: CentOS + Asterisk + Google Voice -- The recipe.
wordpress_id: 153
categories:
- Asterisk
---

So I've gone through this a couple times, and this time... enough is enough: I'm documenting it.

It's not all that hard, but, I had to wade through to get it to work, again. Let's fire up the pre-reqs first. I'm not going to chance my luck with a binary package from a yum repo, and I don't want to search for a supplemental one. Especially when it's so easy to compile Asterisk from source. However, I don't have that philosophy with many other things, here I've compiled only what I needed -- and the rest from my default yum repo(s).

Mainly to get it to work we need two modules for Asterisk: chan_gtalk & res_jabber. They're not going to go without a few things, [so download iksemel]( http://code.google.com/p/iksemel/downloads/list). But before you install it, go ahead with install these few modules:

    
     
    [user@localhost asterisk]$ yum install gnutls-devel gnutls-utils


I got that idea from [this blog post](http://michigantelephone.wordpress.com/2010/09/08/how-to-fix-the-problem-of-missing-modules-res_jabber-so-andor-chan_gtalk-so-in-your-asterisk-installation/). I think it worked, but, I did it at a weird spot between failed installs. I didn't get errors compiling iksemel without it, honestly. But, for the purposes here, I'd say: install 'em. Next up, we'll compile iksemel:

    
     
    [user@localhost iksemel]$ ./configure --prefix=/usr --with-libgnutls-prefix=/usr
    [user@localhost iksemel]$ make
    [user@localhost iksemel]$ make check
    [user@localhost iksemel]$ make install


I had a warning with an extra parameter on the ./configure, it was "--with-gnutls", add this if you have troubles.

But after all this, I'd compile Asterisk and make a CLI call -- only to get this error.

    
    localhost*CLI> module load res_jabber.so
    Error loading module 'chan_gtalk.so':	libiksemel.so.3: cannot open 
    shared object file: No such file or directory


So... The so linkage is screwy. I found [this post from the asterisk-users list](http://lists.digium.com/pipermail/asterisk-users/2010-November/256619.html), which details what I did. Condensed here for you as:

    
     
    [user@localhost asterisk]$ echo "/usr/local/lib" > /etc/ld.so.conf.d/iksemel.conf 
    [user@localhost asterisk]$ ldconfig


And finally! ...Let's compile Asterisk. Here's a refresher if you need it [followed by more fun with jabber/gtalk afterwords]. (Note that you're going to want to install libpri & dahdi first [in that order] if you want to support them in your build)

<!-- more -->

    
     
    [user@localhost asterisk]$ make uninstall-all      # In case your previous build failed
    [user@localhost asterisk]$ ./configure             # I needed libxml2-devel, too.
    [user@localhost asterisk]$ make menuselect         # Look for chan_gtalk & res_jabber.
    [user@localhost asterisk]$ make
    [user@localhost asterisk]$ make config             # If you please.
    [user@localhost asterisk]$ make samples            # They're handy.


Next we'll actually configure it to talk to google! Make sure that the jabber & gtalk modules show up when you do a:

    
    localhost*CLI> module show
    localhost*CLI> module show like gtalk
    localhost*CLI> module show like jabber


So to configure Google Voice... It turns out that _finally _there's a comprehensive document from the canonical wiki.asterisk.org with the [sample configs to use with Google Voice. I'll refer you to them](https://wiki.asterisk.org/wiki/display/AST/Calling+using+Google) -- and just make notes as I continue with a build.

Well, it's not always so simple. I followed their instructions, and while I like them... They're not perfect. I used their skeletons, but I had to google around quite a bit, still. Again, why I'm documenting it this time!

    
    ; ---------------------------------
    ; --------------------- gtalk.conf
    
    [general]
    context=local        ;# Set a default context for this type of channel.
    allowguest=yes       ;# Allow a guest call (those will be unknown calling parties)
    bindaddr=0.0.0.0     ;# Bind to any interface
    externip=11.235.8.13 ;# Your external IP
    
    [guest]
    disallow=all         ;# Ok, setup a media negotiation...
    allow=ulaw           ;# For only ulaw.
    context=gtalk_in     ;# Here is the context we actually arrive into.
    connection=asterisk  ;# And this is the name of the header in jabber.conf


Note that the ";#" is parse-able but _not_ official syntax. I don't have a plugin for wordpress that supports asterisk config syntax higlighting. This is using bash highlighting, so I have the hash in there to make it look pretty on the site. Feel free to mangle this in your own code if it bugs you.

    
    ; -----------------------------
    ; ----------------- jabber.conf
    [general]
    autoregister=yes
    
    [asterisk]
    type=client
    serverhost=talk.google.com
    username=thine_email_here@gmail.com/Talk
    secret=andyourpasswordhereofcourse
    port=5222
    usetls=yes
    usesasl=yes
    statusmessage="Word"
    timeout=100


Let's review some command line hints and talk about some contingencies that I encountered.

    
    localhost*CLI> jabber reload           ;# This did not work in all so cases.
    localhost*CLI> core restart now        ;# So I had to do this.
    localhost*CLI> jabber set debug on     ;# If you run into trouble (I did, naturally)
    localhost*CLI> jabber set debug off    ;# It is noisy so turn it off when done.
    localhost*CLI> jabber show connections ;# Make sure it says its connected
    localhost*CLI> gtalk show channels     ;# When you have a call up, check this out.



Which is all great, however... This is a community supported module, and it doesn't always want to play nice with google voice. So, buyer beware. As the wiki.asterisk.org link states "don't run a business on this thing". I wrestled with it for a while before it would answer my calls, but once I did get it to do it... I noticed that I could have multiple calling parties, ssssssssh. This feature is awesome, thanks Google. 


    
    
    ;# ------------------------------------------------
    ;# --------------------------- Extensions.conf
    
    [gtalk_in]
    exten => s,1,Noop(google in context!) ;# Just a little output for your eyes.
    same =>    n,Answer()                 ;# Answer the incoming call.
    same =>    n,Wait(2)                  ;# Important! Wait 2 to 8 seconds.  
    same =>    n,SendDTMF(11)             ;# Important! I had to send this 2x
    same =>    n,Wait(1)                  
    same =>    n,Playback(hello-world)    ;# Get some audio to make you feel good.
    same =>    n,MeetMe(1,do)             ;# Start a conference call.
    same =>    n,Hangup()                 ;# Always hang up your extensions.
    



And that, is, that.
