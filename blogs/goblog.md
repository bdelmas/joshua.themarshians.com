<!-- Title: Colds, Static Blogs, and Go -->
<!-- Author: Joshua Marsh -->
<!-- Tags: go,blog,html -->
<!-- Languages: shell -->

Last week in my [local Linux user group](http://plug.org/) mailing
list the topic of static blogging came up. I had been thinking of
converting my blog over, but wasn't sure which one I'd prefer to
use. One of the responses to my inquiry was:

> If you've been looking for a project with which to learn a new
> language, writing your own blog engine is a reasonable choice.  The
> 'market' is already hopelessly polluted with toy blog engines, which
> is evidence for its suitability as a learning project.  Most of what
> makes a blog memorable has absolutely nothing to do with the
> back-end software, anyway.  If you are feeling ambitious, you could
> write a blog engine *and* your own Markdown parser with custom
> extensions
> ([Pearson, 2013](http://plug.org/pipermail/plug/2013-January/029154.html)).

I don't normally find myself with much time on my hands. I have two
young children, school, a full-time job, and now a sister-in-law with
crutches in the home. Misfortune (or fortune I suppose) befell me this
weekend though and my son gave me a terrible cold. I was out of
commission for the entire weekend.

You can only watch so much
[Psych](http://www.usanetwork.com/series/psych/) and
[Downton Abbey](http://www.pbs.org/wgbh/masterpiece/downtonabbey/), so
I took upon myself the challenge of writing my own static blog
application as suggested. Without further guilding the lilies, here
it is: [goblog](https://github.com/icub3d/goblog).

I'm so daring, that I've even used it already to convert my existing
blog at blogger into a static blog and I've hosted it using
appengine. If you are reading this, I can say that the software
worked! What's nice about this all is that it's still
free. [Github](https://github.com) can host the code and I don't
imagine myself exceeding the free limits of
[appengine](https://developers.google.com/appengine/).

It is certainly rough around the edges, but I was surprised what I
could accomplish over a weekend while heavily medicated. I attribute a
lot of the success to the tools already available to me. Go provides a
[template package](http://golang.org/pkg/text/template/) and
[Russ Ross](https://github.com/russross/blackfriday) already had a
[markdown](http://daringfireball.net/projects/markdown/) parser. The
front-end is simply [bootstrap](http://twitter.github.com/bootstrap/),
[jquery](http://jquery.com/), and
[rainbow](http://craig.is/making/rainbows). It was merely a matter
putting the pieces together.

You can see a completed working example with my own
[blog](http://joshua.themarshians.com/) which originates from a
[project on github](https://github.com/icub3d/joshua.themarshians.com). It
works great for me, but I'd be happy to entertain requests for
changes. I tried to keep it simple, so it expects you to design the
templates yourself but I imagine most DIY bloggers would want that
anyway. All I do now to publish is:

<pre><code data-language="shell">$ goblog
$ appcfg.py update joshua.themarshians.com
</code></pre>

Overall, it was a fun weekend for me and not just because of the
codeine. I can't say that I was a novice at Go, but I did learn a lot
and hope to use the project as a springboard for more learning. I
still need to add unit tests, improve the documentation, etc. All of
those are sure to increase my knowledge of Go.


