<!-- Title: Hardcore Google: RESTful Communication With AngularJS -->
<!-- Author: Joshua Marsh -->
<!-- Description: I recently began a project using Go, AngularJS, and Google App Engine. In this installment of a series of blog posts, I describe how to use AngularJS and get it communicating with Go. -->
<!-- Tags: go,appengine,golang,angularjs,google -->
<!-- Languages: go,javascript -->

This post is part of a series _Hardcore Google_. You can find other
posts here:

* [Unit Testing App Engine](hardcore-google-unit-testing.html)
* [RESTful Web Services In Go](hardcore-google-communicating-go.html)
* [RESTful Communication With AngularJS](hardcore-google-communicating-angularjs.html)

My history with [JavaScript](http://en.wikipedia.org/wiki/JavaScript)
goes back almost 15 years. Back then, working in the browser was was
difficult. Eventually tools like [jQuery](http://jquery.com/) came
around and made the work somewhat simpler. It wasn't until I
discovered [AngularJS](http://angularjs.org/) though that I felt like
JavaScript development had come out of adolescence. AngularJS does a
great job at solving problems that professional engineers often find
clunky with JavaScript &emdash; namely separating the
[HTML](http://en.wikipedia.org/wiki/HTML) and
[CSS](http://en.wikipedia.org/wiki/CSS) from the code that drives it,
properly testing code, and developing rich, dynamic web
applications. What I love about AngularJS as well is that it doesn't
doesn't try to get away from using JavaScript but merely enhance it.

Enough gushing though. If you are new to AngularJS, you can find great
tutorials on their website or you can contact me directly. What I'm
interested in talking about today is how AngularJS interacts with web
services, particularly
[REST](http://en.wikipedia.org/wiki/Representational_state_transfer)ful
web services. In my
[last post](hardcore-google-communicating-go.html), we got
[Go](http://golang.org/) to serve our web services. Now we just need
our front end web app to communicate with that service.

AngularJS provides two ways to wire up a back end. The simplest method
is to use the
[$resource](http://docs.angularjs.org/api/ngResource.$resource)
service. It provides a simple interface to a web service. If you need
finer grained control though, you can use the
[$http](http://docs.angularjs.org/api/ng.$http) service. I'll show you
below how to use the *$http* service because that's what I ended up
going with. If you decide to use the *$resource* service though, just
keep in mind that it currently always strips the trailing slash wether
you ask it to or not. You'll need to have your
[http.Handler](http://golang.org/pkg/net/http/)s account for that. For
example:

<pre><code data-language="go">func fooHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Thanks for the %s!", r.Method)
}

// Handle /foo
http.HandleFunc("/foo", fooHandler)

// Or Handle /foo/
http.HandleFunc("/foo/", fooHandler)
</code></pre>

Before, we go about using the *$http* service though, we need to talk
about services in general. Like the *$http* and *$resource* services
that AngularJS provides, you can create your own. Services are a great
way to decouple code such as connecting to a web service. You can
[mock](http://en.wikipedia.org/wiki/Mock_object) them when you are
testing the parts of your application that use them and AngularJS will
manage the instantiation and destruction of the services for you. You
can find a complete example of a user defined service in the file
[list.js](https://github.com/icub3d/home/blob/master/dev/js/rest/lists.js).

Let's walk through the create function. It handles the creation of new
lists or making copies of existing lists. We start by making a *POST*
using the *$http* service.

<pre><code data-language="javascript">var promise = $http.post("/rest/list/", data);
</code></pre>

This performs an HTTP POST to the URL *"/rest/list/"* with the JSON
data as the content of the POST. There are similar functions for
*put*, *get*, and *delete* that correspond to the HTTP *PUT*, *GET*
and *DELETE* methods. The data could technically be anything. In this
case though, it's a JavaScript object that contains the information
necessary to create the new list. AngularJS will take care of
converting the object into JSON that our Go web service can consume.

The [promise](http://wiki.commonjs.org/wiki/Promises) returned allows
you to hook a success and failure function to the asynchronous
response to the request. Here, we call the *scall* or *ecall*
functions based on the success or failure of the *POST*. A similar
thing happens in all the other functions for the service. In the *get*
function, for example, we do the same thing. The only thing that
differs is that we expect a key for the list being retrieved.

<pre><code data-language="javascript">this.get = function(key, scall, ecall) {
    var promise = $http.get("/rest/list/" + key + "/");
	...
</code></pre>

You can see an example of using both of these functions and the
service in the
[all.js](https://github.com/icub3d/home/blob/master/dev/js/lists/all.js)
file in the *$scope.copy* function:

<pre><code data-language="javascript">$scope.copy = function() {
	Lists.get($scope.data.key, function(l) {
		l.Name = $scope.data.name;
		Lists.create(l, function(nl) {
				$location.path('/lists/view/' + nl.Key + '/');
		});
	});
};
</code></pre>

The goal of the *copy* function is to make a copy of a list with a new
name. We do that by *get*ting the list to be copied by its key. If
that succeeds, we change the list name and call the *create* function
for that list. If it succeeds, then we redirect to the new list.

I like this function because it shows the simplicity of using
AngularJS's frameworks. I can make two service calls in just a few
lines of code to do something meaningful with the application. It also
exemplifies the notion of "going with the flow." Most libraries or
programming languages have a right and wrong way of doing things. If
you do it the right way, you can get a lot of functionality with just
a few lines of code. This makes your code both easier to read and
maintain.
