<!-- Title: Honing Your Inner Command Line -->
<!-- Author: Joshua Marsh -->
<!-- Tags: bash,shell,linux -->
<!-- Languages: shell -->

As a [Linux](http://en.wikipedia.org/wiki/Linux) enthusiast, I often
find that there is just so much to learn that it can get
overwhelming. The keyboard is by far still the fastest way to interact
with a computer, but memorizing all of the shortcuts can be daunting.

When I was getting the hang of
[emacs](http://www.gnu.org/software/emacs/), I thought it would be
good to read a book to try and improve my skills. I quickly realized
though that I was getting too much information at once to make it
useful. My cap seems to be two or three new shortcuts or commands per
day that I will be able to make use of on a regular basis.

I think that is generally applicable to all people. Learning one or
two new commands is certainly feasible, as long as it's a command you
will be able to use regularly. In that same vien, I was recently
directed to the website
[commandlinefu.com](http://www.commandlinefu.com/commands/browse)
during a discussion at my local
[Linux users group](http://www.plug.org/).

This is a great place to get all kinds of amazing new commands. I've
found that if you sort them by
[popularity](http://www.commandlinefu.com/commands/browse/sort-by-votes),
you either know them or they are so awesome you want to try them out
immediately. There is still that pesky matter of recalling them when
you actually would benefit from having them.

To that end, I decided to make aliases for them. That way, all of my
favorite commands would be in my <em>.bash\_aliases</em> file and I
could have an easy place to look them up. Many of them have command
line parameters though which aliases don't normally accept. There is a
way around this though. You can alias a bash function. So, for
example, if you'd like to alias the
[remote diff](http://www.commandlinefu.com/commands/view/47/compare-a-remote-file-with-a-local-file),
you'd probably want the server and the two files to be defined at the
command line. In your .bash_aliases file, you can do something like
this:

<pre><code data-language="shell">remote_diff()  
{  
     ssh $1 cat $2 | diff $3 -  
}  

alias rdiff=remote_diff  
</code></pre>

Then the rdiff command could be executed like:

<pre><code data-language="shell">rdiff user@host /some/remote/file /some/local/file
</code></pre>
