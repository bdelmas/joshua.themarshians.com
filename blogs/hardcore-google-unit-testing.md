<!-- Title: Hardcore Google: Unit Testing App Engine -->
<!-- Author: Joshua Marsh -->
<!-- Description: I recently began a project using Go, AngularJS, and Google App Engine. In the first installment of a series of blog posts, I describe how I went about setting up unit testing for Google App Engine. -->
<!-- Tags: go,appengine,golang,angularjs,google -->
<!-- Languages: shell,go -->

This post is part of a series _Hardcore Google_. You can find other
posts here:

* [Unit Testing App Engine](hardcore-google-unit-testing.html)
* [RESTful Web Services In Go](hardcore-google-communicating-go.html)
* [RESTful Communication With AngularJS](hardcore-google-communicating-angularjs.html)

I recently began a project using [Go](http://golang.org/),
[AngularJS](http://angularjs.org/), and
[Google App Engine](https://developers.google.com/appengine/). It is a
personal web site I use at home to help keep my family organized. I'll
talk more about it later in a series of posts. I'm not actually
completely finished with it yet, but I thought I'd start this series.
Some of the aspects of the project were interesting enough that I
wanted to have a place where I could look back and remember how I did
it. There may also be some people out there that get something out of
it as well.

First, let me explain why I picked Go, AngularJS and Google App
Engine. I'm a [DIY](http://en.wikipedia.org/wiki/Do_it_yourself) sort
of guys, and I'm always fascinated with trying different things. The
bulk of my work life revolves around [Python](http://www.python.org/),
[PHP](http://php.net/),
[C](http://en.wikipedia.org/wiki/C_(programming_language), and
[Perl](http://www.perl.org/). These are cool languages and I can do
awesome things with them, but they aren't *new* and *exciting* to me
anymore.

Go and AngularJS seem to be the new kids on the block that are getting
a lot of attention in my circles. I've already posted quite a bit
about Go. I haven't done a major project with it yet though, so I
thought this would be a good opportunity. I also recently came across
AngularJS and was impressed with the way it allows you to abstract the
presentation with the application logic. I've never been more
interested in [JavaScript](http://en.wikipedia.org/wiki/JavaScript)
before.

I wanted to find a place I could host my code for free and not limit
it to a server running in my home. The project would be extremely
useful on the go with my table or phone. App Engine seemed like the
best place for this because I'd likely never go over the free daily
quotas. Since Go is supported, it seemed like a good fit in that
respect as well. I already use
[Google Apps](http://www.google.com/intl/en/enterprise/apps/business/),
so I could manage authentication through that and wouldn't need to
design any of that logic.

And with that, I officially became a Google
[fanboi](http://www.urbandictionary.com/define.php?term=fanboi) for
this project. I'll talk more about the architecture in another post,
but I used App Engine to server my static content and server my
[REST](http://en.wikipedia.org/wiki/Representational_state_transfer)ful
interface built in Go. The website, when minified is just one web
page, one CSS file, and one JavaScript file. It communicates to a
simple REST service to retrieve the data it needs for presentation.

One of the problems I ran into quickly was the ability to
[unit test](http://en.wikipedia.org/wiki/Unit_testing) my code. This
project is a bit more than just a few files, so unit testing was going
to be important. I find that for these large projects if I can write
unit tests I can iteratively develop parts of the system without
needing the others to make sure my code works as expected. Creating
unit tests in go is fairly simple. The
[go tool](http://golang.org/cmd/go/) has a test command and the
[testing package](http://golang.org/pkg/testing/) provides you with
just about everything you need.

Testing code that relies on the *appengine* package is more difficult
though. You need some sort of
[mock object](http://en.wikipedia.org/wiki/Mock_object) for the
appengine to make any sort of testing possible. I found a package by
[Brad Fitzpatrick](http://bradfitz.com/) called
[gae-go-testing](http://code.google.com/p/gae-go-testing/). It was a
bit outdated though and didn't contain all the features I was looking
for. [Open source](http://en.wikipedia.org/wiki/Open_source) to the
rescue! I forked the repository and got it working. You can see and
use the forked package at
[github.com/icub3d/appenginetesting](https://github.com/icub3d/appenginetesting).

The basic premise is to spin up a temporary App Engine instance
locally and then use it with some hooks to simulate typical App Engine
functionality. To use it, you need to already have the
[App Engine SDK installed](https://developers.google.com/appengine/downloads). With
that done, we just need to do some minor setup work before we run our
tests:

<pre><code data-language="shell">export APPENGINE_SDK=/path/to/google_appengine
export PATH=$PATH:$APPENGINE_SDK
cd ~/go/src
ln -s $APPENGINE_SDK/goroot/src/pkg/appengine
ln -s $APPENGINE_SDK/goroot/src/pkg/appengine_internal
go get github.com/icub3d/appenginetesting
</code></pre>

With that done, we should now be able to perform some basic App Engine
functions in our tests. For example:

<pre><code data-language="go">// Create mocked context.
c, err := appenginetesting.NewContext(nil)
if err != nil {
    fmt.Println("initilizing context:", err)
    return
}

// Close the context when we are done.
defer c.Close()

// Get an element from the datastore.
k := datastore.NewKey(c, "Entity", "stringID", 0, nil)
e := ""
if err := datastore.Get(c, k, &e); err != nil {
    fmt.Println("datastore get:", err)
    return
}

// Put an element in the datastore.
if _, err := datastore.Put(c, k, &e); err != nil {
    fmt.Println("datastore put:", err)
    return
}
</code></pre>

We create a new *Context* using the *appenginetesting.NewContext*
function. This call spins up a temporary instance of the App Engine
development server. We can then use it like we would a real *Context*
and perform normal data store queries. 

One feature that was missing was the ability to fetch user information
from the *Context*. Adding it was fairly straight forward. You can use
it's functionality like this:

<pre><code data-language="go">// Create mocked context.
c, err := appenginetesting.NewContext(nil)
if err != nil {
    fmt.Println("initilizing context:", err)
    return
}

// Close the context when we are done.
defer c.Close()

// Log a user in.
c.Login("test@example.com", true)

// Get the user.
u := user.Current(c)
if u == nil {
    fmt.Println("we didn't get a user!")
    return
}

fmt.Println("Email:", u.Email)
fmt.Println("Admin:", u.Admin)
</code></pre>

With all of this setup, it is now extremely simply for me to write
test cases against my App Engine related code. Hopefully you'll find
this helpful in your App Engine projects using Go. You can see the
full documentation on
[godoc](http://godoc.org/github.com/icub3d/appenginetesting). There is
still room for improvement, so if you have feature requests, don't
hesitate to enter an issue or make a
[pull request](https://github.com/icub3d/appenginetesting/pulls).
