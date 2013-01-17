<!-- Title: Go App, Do You Read Me? -->
<!-- Author: Joshua Marsh -->
<!-- Tags: rpc,go,linux -->
<!-- Languages: shell,go -->

I am now running a few personal [Go](http://golang.org/) applications
and am working on a Go web application that will eventually be ready
for public consumption. Since Go gives us the ability to act as a
[FastCGI](http://www.fastcgi.com/drupal/) backend or a
[stand alone web application](http://golang.org/pkg/net/http/), it
only seems reasonable that in a production system you'd want the
ability to use it like any other web server. This includes the ability
to reload the configuration on the fly and start and stop the
application.

I thought I'd put together an example of how I'm doing it right
now. From what I've been able to gather,
[RPC](http://en.wikipedia.org/wiki/Remote_procedure_call) seems to be
the best way to interact with your application. If you are wanting to
run the application yourself, you can try it by doing:

<pre><code data-language="shell">git clone https://github.com/icub3d/go-ipc-with-rpc.git
cd go-ipc-with-rpc/servicer
go install  
cd ../greeter  
go install  
cd ..  
emacs greeter.service
# Modify greeter.service to use the path to your GOBIN where the   
# above apps were installed.  
sudo cp greeter.services /etc/systemd/system  
cp config.json /tmp  
</code></pre>

You can then run it and test if like so:

<pre><code data-language="shell">$ sudo systemctl start greeter  
$ curl http://localhost:9000  
Howdy Doodie!  
$ emacs /tmp/config.json  
$ sudo systemctl reload greeter  
$ curl http://localhost:9000  
Hey there, partner!  
$ sudo systemctl stop greeter  
$ curl http://localhost:9000  
curl: (7) couldn't connect to host at localhost:9000  
</code></pre>

At a high level there are three parts to the application. The first
part is the web application itself which can be found in the greeter
directory. The second is a helper application that makes RPC calls to
the greeter. It can be found in the servicer directory. Finally, we
have a default configuration for the greeter application and a systemd
service configuration file.

Within the greeter application, I've divided the code into four
parts. The file
[config.go](https://github.com/icub3d/go-ipc-with-rpc/blob/master/greeter/config.go)
manages reading the config file and translating it into a map that the
rest of the application can use. Using a [JSON](http://www.json.org/)
config file makes reading and loading the configuration unbelievably
simple. I also use a [mutex](http://golang.org/pkg/sync/#Mutex) to
protect access to the configuration while it is being reloaded.

The file
[flag.go](https://github.com/icub3d/go-ipc-with-rpc/blob/master/greeter/main.go)
just initializes the command line flags for parsing. It isn't really
related to this post, but it was helpful during testing because I
could point to various configuration files. Under normal
circumstances, you'd likely have more complex flags or maybe none at
all.

The file
[main.go](https://github.com/icub3d/go-ipc-with-rpc/blob/master/greeter/main.go)
contains the main function which starts up the application. It first
parses the command line flags and then reads the configuration
file. It then starts the listener for our web application. Next, we
start the RPC services. We pass the web application's listener to this
function so that the RPC services know which listener to interact
with. Finally, we setup our web server.

The last file in the greeter application is
[rpc.go](https://github.com/icub3d/go-ipc-with-rpc/blob/master/greeter/rpc.go),
which is the meat of what I wanted to talk about. If this looks
foreign to you, don't fret. Go has some great [documentation](http://golang.org/pkg/net/rpc/) on using
RPC. We create a ServiceHandler struct which will manage all of the
RPC calls. RPC requires a very specific method definition, so even
though we don't use the incoming arguments, they need to be present. I
used an integer for simplicity.

The Stop function checks to see if the listener is available and if so
closes it. This has the net effect of stopping the web application
because the [http.Serve](http://golang.org/pkg/net/http/#Serve)
function exits when the listener is closed. The Reload function is
equally simple because we need only call the ReadConfig function
again.

The StartRPC function is what sets up the RPC service. You register
objects with the [rpc](http://golang.org/pkg/net/rpc/) package and
then it takes care of determining which functions meet the criteria of
being exposed. In our case we register a ServiceHandler. We then set
up RPC to handle HTTP requests and open up a port to listen for those
request. Finally, we use the http's Serve function to start accepting
connections to the RPC service.

With that done, we now just need some way to talk to the RPC
service. I wrote a little helper application that does this in the
servicer directory. It is all contained in
[servicer.go](https://github.com/icub3d/go-ipc-with-rpc/blob/master/servicer/servicer.go). First,
it gets our operation from the command line and translates that into
the RPC method we want to call. Next, we create a client RPC
connection with the
[DialHTTP](http://golang.org/pkg/net/rpc/#DialHTTP) function. We then
use the client to [Call](http://golang.org/pkg/net/rpc/#Client.Call) the method we wanted.

Now all that remains is to hook it up into your service manager. I run
[Arch Linux](https://www.archlinux.org/) and use [systemd](http://www.freedesktop.org/wiki/Software/systemd). If you run
another service manager, you'll have to set up your own configuration
for it. I've provided a [greeter.service](https://github.com/icub3d/go-ipc-with-rpc/blob/master/greeter.service) which starts up the greeter
application and uses the servicer to reload and stop the application.

Overall, I'm extremely happy with the outcome. I can see this being a
common pattern and plan to eventually make a package out of it. If for
some reason all my links above weren't enough, here is where I got
most of my information from and my example:

* [Go's RPC Documentation](http://golang.org/pkg/net/rpc/)
* [Systemd's Service Configuration Options](http://www.freedesktop.org/software/systemd/man/systemd.service.html)
* [My Completed Example](https://github.com/icub3d/go-ipc-with-rpc)

Updates
-------

There was some discussion about the security of using RPC. RPC itself
doesn't care about security so that can leave you open to
attack. Obviously a firewall would help solve this. Alternatively, you
can use a named pipe instead of a socket and go's rpc package will use
it just the same.
