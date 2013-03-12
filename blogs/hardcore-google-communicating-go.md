<!-- Title: Hardcore Google: RESTful Web Services In Go -->
<!-- Author: Joshua Marsh -->
<!-- Description: I recently began a project using Go, AngularJS, and Google App Engine. In this installment of a series of blog posts, I describe how to get Go to be a robust RESTful web service. -->
<!-- Tags: go,appengine,golang,angularjs,google -->
<!-- Languages: go -->

This post is part of a series _Hardcore Google_. You can find other
posts here:

* [Unit Testing App Engine](hardcore-google-unit-testing.html)
* [RESTful Web Services In Go](hardcore-google-communicating-go.html)
* [RESTful Communication With AngularJS](hardcore-google-communicating-angularjs.html)

When you first start developing a web application, one of the first
big decision you make after choosing your tool set is how these tools
will interact with each other. For my project, I had already chosen
[Go](http://golang.org/) for my back-end,
[AngularJS](http://angularjs.org/) for by front-end, and
[Google App Engine](https://developers.google.com/appengine/) to host
it all. It was just a matter of figuring out how to get Go and
AngularJS to communicate with each other. Fortunately doing that was
stupid-simple.

I chose a
[REST](http://en.wikipedia.org/wiki/Representational_state_transfer)ful
API for communication because it seemed to be the most organized
method of communication and both sides of the web application support
it quite well. I've found in my development career that trying to fit
a square peg into a round hole only makes you lose your hair
faster, so REST it was.

Using a RESTful web service means that you'll be managing your data
based on the HTTP method (i.e. GET, POST, DELETE, etc.) and URL. On
the Go side of things, you'll be using the
[net/http](http://golang.org/pkg/net/http/) package to handle the
requests from AngularJS. At a high level, you tell Go to handle a
request like this:

<pre><code data-language="go">http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Thanks for the %s!", r.Method)
})
</code></pre>

Here we register a function to handle all requests to */bar*. Every
function that handles a request has the same function definition: it
has as its parameters an http.ResponseWriter which is used to write a
response and an http.Request which contains the details of the request
being made. In this case, if we started up this server, and made the
request "DELETE /bar", we'd receive back "Thanks for the DELETE!"

Hopefully, you can see where this is going. To implement a RESTful API
in Go, you would register handlers for each URL endpoint and then
based on the given method you'd perform an operation. We can make this
process even better though by using
[gorilla/mux](http://www.gorillatoolkit.org/pkg/mux). It is a drop in
replacement for Go's http handlers, but it can take care of the details
of routing based on the HTTP method. Here is an example:

<pre><code data-language="go">m := mux.NewRouter()

// Get all lists.
m.HandleFunc("/", GetAllLists).Methods("GET")

// Make a new list.
m.HandleFunc("/", PostList).Methods("POST")

// Singe list operations.
m.HandleFunc("/{key}/", GetList).Methods("GET")
m.HandleFunc("/{key}/", PutList).Methods("PUT")
m.HandleFunc("/{key}/", DeleteList).Methods("DELETE")

// Everything else fails.
m.HandleFunc("/{path:.*}", gorca.NotFoundFunc)
</code></pre>

As you can see, I register handlers for each of the REST methods I'm
interested in. In this example, I'm registering some functionality for
the list part of my web application. I can either *GET* all of the
lists or *POST* a new list at the root (*/*). If I'm given a key
(*/{key}/*) in the URL, then I'm dealing with a specific list. In that
case, I can get the list, update the list, or delete with list with
*GET*, *PUT*, and *DELETE*.

The last *HandleFunc* acts as a catch all. If the client makes a
request that isn't one explicitly listed, we return a 404 status and a
JSON response with the details. The *net/http* package already returns
a simple 404, but I wanted to return a JSON response as well. Using
the catch all allows me to send that JSON response. The client can
then present a useful message to the user should something go wrong
instead of a somewhat useless "Not Found".

As an example of how the handlers look, the *GetAllLists* handler
looks like this:

<pre><code data-language="go">// GetAllLists fetches all of the lists.
func GetAllLists(w http.ResponseWriter, r *http.Request) {
	// Create the query.
	c := appengine.NewContext(r)
	q := datastore.NewQuery("List").Order("-LastModified")

	// Fetch the lists. 
	lists := []List{}
	if _, err := q.GetAll(c, &lists); err != nil {
		gorca.LogAndUnexpected(c, w, r, err)
		return
	}

	// Write the lists as JSON.
	gorca.WriteJSON(c, w, r, lists)
}
</code></pre>

Some of the details may be fuzzy if you aren't familiar with App
Engine, but I basically fetch all of the lists from App Engine's
[datastore](https://developers.google.com/appengine/docs/go/datastore/)
and then convert it to [JSON](http://www.json.org/) and send that as
the response.

The *GetAllLists* function is a great example of how Go and App Engine
interact to make development simple. In just a couple of dozen lines
of code, I can create a robust REST web service. I don't have to deal
with [MySQL](http://www.mysql.com/) connections, user authentication,
or parsing incoming HTTP requests. App Engine and Go do all that for
me. The net result is that I have more readable, testable, and
maintainable code.

You can see how I hooked this all up in more detail in
my application:

* [Home App](https://github.com/icub3d/home/)
* [The Global Handler](https://github.com/icub3d/home/blob/master/rest/rest.go)
* [The List Handler](https://github.com/icub3d/home/blob/master/rest/list/muxer.go)
* [The List Handler Functions](https://github.com/icub3d/home/tree/master/rest/list)

I was truly surprised at how easy this process was. Rigging up a
back-end can be a nightmare. At work, I deal with a
[SOAP](http://en.wikipedia.org/wiki/SOAP) web service and I am willing
to testify in court that the 'S' (simple) is a lie. With Go, this
isn't the case. It already provides great functionality and the open
source nature of the language means that great packages like
*gorilla/mux* are there when you want them. Stay tuned, for next
time. I'll talk about getting AngularJS to consume and present the
JSON we are sending back.
