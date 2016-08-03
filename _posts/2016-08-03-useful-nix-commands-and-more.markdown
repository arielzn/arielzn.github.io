---
layout: post
title:  "Useful linux commands and config files"
date:   2016-01-03 02:06:07 +0200
categories: test
---

**Useful commands**


rsync through SSH. 
Ideally, server will be defined in .ssh/config file in case you have custom ports keypair to use.

{% highlight bash %}
rsync  -avz --progress -u /origin/path  user@server:/dest/path  
{% endhighlight %}

Mount remote share with SSH access.
to be added to fstab

{% highlight bash %}
sshfs#user@remoteserver: /local/mountpoint fuse user,idmap=user,workaround=rename,noauto,fsname=sshfs#user@remoteserver: 0 0
{% endhighlight %}
