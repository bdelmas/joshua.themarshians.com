<!-- Title: Go, Go, Go, Go On An Adventure -->
<!-- Tags: go -->
<!-- Author: Joshua Marsh -->

I've been having a lot of fun playing with
[Google's](http://www.google.com) new language
[Go](http://golang.org/. I just finished
[A Tour of Go](http://tour.golang.org/) as well as their references:
[How to Write Go Code](http://golang.org/doc/code.html) and
[Effective Go](http://golang.org/doc/effective_go.html). Some of the
exercises in the tour were quite fun to implement. You can find my
answers in [gist](https://gist.github.com/2978920). Feedback is
welcomed!

The built in concurrency was the most fascinating for me. I was
surprised at how simple the last problem (WebCrawler.go) ended up
being even though it took a few different designs for me to get it
right. I've been using [Twisted](http://twistedmatrix.com/trac/) as a
back-end for a web app I'm making and the asynchronous behavior is
awesome, but its much more complex than Go. The reactor pattern isn't
especially complex, It's just not baked into the language like it is
with Go.

I'm really tempted to make the switch to Go. I don't have much invested into Twisted because 90% of my app is the front-end. The back-end is really just a [REST](http://en.wikipedia.org/wiki/Representational_state_transfer) interface with authentication and authorization.

If you are looking for something new and interesting to do for a few
days, try taking [A Tour of Go](http://tour.golang.org/)! If nothing
else, they have one of the coolest mascots I've seen in a while...

![Gopher](http://golang.org/doc/gopher/frontpage.png)
