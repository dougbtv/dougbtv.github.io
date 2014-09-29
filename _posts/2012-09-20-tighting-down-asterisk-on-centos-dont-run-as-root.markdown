---
author: doug
comments: true
date: 2012-09-20 19:55:08+00:00
layout: post
slug: tighting-down-asterisk-on-centos-dont-run-as-root
title: 'Tightening down Asterisk on CentOS: Don''t run as root.'
wordpress_id: 174
categories:
- Asterisk
---

First order of business, in [Future of Telephony 3rd edition chapter 3](http://www.asteriskdocs.org/en/3rd_Edition/asterisk-book-html-chunk/Installing_id240883.html), their install process goes through adding a specific user for asterisk. It's great. But, one thing it doesn't do for me, is... Allow a user that's "not root" and "not the asterisk" user, to access the Asterisk CLI. So, if you want to run the CLI with group permissions, there's a few more steps. That way I can add my regular users who need access to the Asterisk CLI (including me) to the "asterisk group".

To summarize, without having to sift through the rest of the install (assuming you've already got an install process). Note: You'll probably have to do most of this sudo/root style. I shut my asterisk service down before doing this, you might want to, too.

    
    # You probably want to have something less obvious than "asterisk" as the user name, fwiw.
    [user@host ~]$ /usr/sbin/useradd asterisk
    [user@host ~]$ passwd asterisk
    # assuming you don't want them to login [I don't].
    [user@host ~]$ usermod -s /usr/sbin/nologin asterisk
    # Next, add your user to the asterisk group.
    [user@host ~]$ usermod -a -G asterisk your_user_name
    # Then apply the permissions as suggested by Leif in *:TFoT
    [user@host ~]$ chown -R asterisk:asterisk /usr/lib/asterisk/
    [user@host ~]$ chown -R asterisk:asterisk /var/lib/asterisk/
    [user@host ~]$ chown -R asterisk:asterisk /var/spool/asterisk/
    [user@host ~]$ chown -R asterisk:asterisk /var/log/asterisk/
    [user@host ~]$ chown -R asterisk:asterisk /var/run/asterisk/
    [user@host ~]$ chown asterisk:asterisk /usr/sbin/asterisk


And then you'll have to set that in the asterisk.conf, that you want asterisk to run as that user, uncomment these lines / tailor the user & group:

    
    runuser = asterisk              ; The user to run as.
    rungroup = asterisk             ; The group to run as.



What's left out in Leif's process is making sure your asterisk.ctl file in /var/run/asterisk/ has proper permission so that users of the asterisk group can write to it. These settings can be found in the asterisk.conf file as well.


    
    ; Changing the following lines may compromise your security.
    [files]
    astctlpermissions = 0775
    astctlowner = asterisk
    astctlgroup = asterisk
    astctl = asterisk.ctl
