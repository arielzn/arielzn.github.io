---
layout: post
title:  "Useful git related commands"
date:   2016-10-05 02:06:07 +0200
categories: git
---

**How to keep a fork synced with upstream**


Assuming the repo 
```
https://github.com/someuser/repo.git
```
was forked in your github account to 
```
https://github.com/youruser/repo.git
```

{% highlight bash %}
$ git clone  https://github.com/youruser/repo.git
$ git remote add upstream https://github.com/otheruser/repo.git
{% endhighlight %}

Check that upstream remote was added correctly

{% highlight bash %}
$ git remote -v

origin    https://github.com/youruser/repo.git (fetch)
origin    https://github.com/youruser/repo.git (push)
upstream  https://github.com/someuser/repo.git (fetch)
upstream  https://github.com/someuser/repo.git (push)
{% endhighlight %}


When commits are made to the upstream repo you'll see on your fork "this branch
is XXX commits behind someuser:master"

To sync, fetch the upstream commits

{% highlight bash %}
$ git fetch upstream
{% endhighlight %}

and merge with our local master branch, make sure you are in master before

{% highlight bash %}
$ git checkout master
$ git merge upstream/master
{% endhighlight %}

then push to your fork

{% highlight bash %}
$ git push master
{% endhighlight %}



That's the setup.


When developing in your fork it's a good idea to do it in an independent 'dev' branch
so the master can be always kept in sync with upstream and compared with our current work one.

{% highlight bash %}
$ git checkout -b dev
$ git push origin dev
{% endhighlight %}

Then after working on the code and commiting you can compare with master

{% highlight bash %}
$ git diff master dev
{% endhighlight %}


