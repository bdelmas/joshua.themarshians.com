<!-- Title: Dynamic Server Management With Doozer and Groupcache -->
<!-- Author: Joshua Marsh -->
<!-- Description: Groupcache provides a great way to cache data. Instances can be dynamically managed using software like Doozer. -->
<!-- Tags: go,golang,groupcache,doozer -->
<!-- Languages: go,shell -->

[Groupcache](https://github.com/golang/groupcache), while not a
complete replacement for [memcached](http://memcached.org/), is an
amazing caching library. In just a few lines of code, you can greatly
improve the access time to your immutable data. One of the problems
many people quickly run into when using groupcache is maintaining a
list of peers where the cached data is distributed.

Using a configuration file is a fairly brittle method for maintaining
the list. Fortunately, this isn't a new problem. There are several
programs which store configuration information in a highly available
and consistent system. [ZooKeeper](http://zookeeper.apache.org/) is
fairly popular with some of the people I know. There is even a library
for connecting to ZooKeepr in Go:
[gozk](https://wiki.ubuntu.com/gozk). Let's be Go purists for a few
minutes though and give [Doozer](https://github.com/ha/doozerd) a try.

Applications like Doozer solve the problem of maintaining a list of
peers in two ways. First, we can fetch a current list of peers from
the Doozer server. Second, we can create a watch on the list of
peers. Whenever the list changes, Doozer will push us an updated
list. We can use both of these to keep our list of peers up to date.

I've put together a contrived example
[in a gist](https://gist.github.com/icub3d/7505798). In this example,
we are caching definition responses from a
[dict](http://www.dict.org/) service. Why would you want to do that? I
don't know. Maybe you host your own dict service and there is a
spelling bee craze in your town. The actual cached data isn't what's
interesting in this discussion.

Here is how we'll setup our groupcache:

<pre><code data-language="go">// Setup the cache.
pool = groupcache.NewHTTPPool("http://" + addr)
dict = groupcache.NewGroup("dict", 64<<20, groupcache.GetterFunc(
	func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
		def, err := query(key)
		if err != nil {
			err = fmt.Errorf("querying remote dictionary: %v", err)
			log.Println(err)
			return err
		}

		log.Println("retrieved remote definition for", key)
		dest.SetString(def)
		return nil
	}))
</code></pre>

The _query_ function can be found farther in the file, but it
basically [Dial](http://golang.org/pkg/net/#Dial)s into the dict
service and asks for the definition, returning the first. With this
setup, we'll only have to query the dict service if it's not already
in the cache.

If we left it at that, each instance of groupcache would only store
results locally. We want to distribute the cache among a group of
servers though, so we need to tell groupcache where its peers are with
[HTTPPool.Set](http://godoc.org/github.com/golang/groupcache#HTTPPool.Set). In
our application we do that by first fetching the list from Doozer,
then we add ourselves to the list, and finally we call the _Set_ method:

<pre><code data-language="go">// Run the initial get.
data, rev, err := d.Get(peerFile, nil)
if err != nil {
	log.Println("initial peer list get:", err)
	log.Println("using empty set to start")
	peers = []string{}
} else {
	peers = strings.Split(string(data), " ")
}

// Add myself to the list.
peers = append(peers, "http://"+addr)
rev, err = d.Set(peerFile, rev,
	[]byte(strings.Join(peers, " ")))
if err != nil {
	log.Println("unable to add myself to the peer list (no longer watching).")
	return
}
pool.Set(peers...)
log.Println("added myself to the peer list.")
</code></pre>

The calls to Doozer are relatively simple. In fact most of this code
is logging and error checking. We take this a step further by actively
listening to signals and changes from Doozer and updating the peer
list accordingly:

<pre><code data-language="go">// Setup signal handling to deal with ^C and others.
sigs := make(chan os.Signal, 1)
signal.Notify(sigs, os.Interrupt, os.Kill)

// Get the channel that's listening for changes.
updates := wait(d, peerFile, &rev)

for {
	select {
	case <-sigs:
		// Remove ourselves from the peer list and exit.
		for i, peer := range peers {
			if peer == "http://"+addr {
				peers = append(peers[:i], peers[i+1:]...)
				d.Set(peerFile, rev, []byte(strings.Join(peers, " ")))
				log.Println("removed myself from peer list before exiting.")
			}
		}
		os.Exit(1)

	case update, ok := <-updates:
		// If the channel was closed, we should stop selecting on it.
		if !ok {
			updates = nil
			continue
		}

		// Otherwise, update the peer list.
		peers = update
		log.Println("got new peer list:", strings.Join(peers, " "))
		pool.Set(peers...)
	}
}
</code></pre>

We start by setting up two channels. The first is used for receiving
signals from the operating system. When we kill the application, we
remove ourselves from the peer list before we exit. The second is used
for receiving changes from Doozer. Doozer doesn't give us a channel to
receive from, but wrapping it around one is fairly simple. If you
aren't familiar with how to do that, you can check out the _wait_
function.

Grab the code and try it out yourself! You'll first want to download
[Doozer](https://github.com/ha/doozerd/downloads) and start up an
instance if you don't have one:

<pre><code data-language="shell">
./doozerd -w false
</code></pre>

You may also need to get the libraries if you don't have them
already:

<pre><code data-language="shell">
go get github.com/golang/groupcache
go get github.com/ha/doozer
</code></pre>

In one terminal run: 

<pre><code data-language="shell">
go run main.go -addr="127.0.0.1:8000"
</code></pre>

Then in another terminal, run a similar command using a different
port:

<pre><code data-language="shell">
go run main.go -addr="127.0.0.1:8001"
</code></pre>

Notice in the first terminal, that Doozer notifies us of the change:

	got new peer list:  http://127.0.0.1:8000 http://127.0.0.1:8001
	
You can keep adding new instances and you should see all the other
systems receive the updates. Try killing one of the instances (Ctrl+C)
and notice that it removes itself from the list.

You can verify that the service works using curl. For example:

<pre><code data-language="shell">
$ time curl http://localhost:8002/define/piston
Piston \Pis"ton\, n. [F. piston; cf. It. pistone piston, also
   pestone a large pestle; all fr. L. pinsere, pistum, to pound,
   to stamp. See {Pestle}, {Pistil}.] (Mach.)
   A sliding piece which either is moved by, or moves against,
   fluid pressure. It usually consists of a short cylinder
   fitting within a cylindrical vessel along which it moves,
   back and forth. It is used in steam engines to receive motion
   from the steam, and in pumps to transmit motion to a fluid;
   also for other purposes.
   [1913 Webster]
curl http://localhost:8002/define/piston  0.01s user 0.01s system 7% cpu 0.264 total

$ time curl http://localhost:8002/define/piston
Piston \Pis"ton\, n. [F. piston; cf. It. pistone piston, also
   pestone a large pestle; all fr. L. pinsere, pistum, to pound,
   to stamp. See {Pestle}, {Pistil}.] (Mach.)
   A sliding piece which either is moved by, or moves against,
   fluid pressure. It usually consists of a short cylinder
   fitting within a cylindrical vessel along which it moves,
   back and forth. It is used in steam engines to receive motion
   from the steam, and in pumps to transmit motion to a fluid;
   also for other purposes.
   [1913 Webster]
curl http://localhost:8002/define/piston  0.01s user 0.01s system 76% cpu 0.020 total
</code></pre>

Because we've cached the results of the definition of piston, the
second request is substantially faster. Overall, the code involved is
fairly simple and the wins are huge. You no longer need to worry about
maintaining a list of peers in some configuration file because your
code will take care of it.

There are a couple of things you can do to improve upon the code we
have here. Some applications may have multiple caches they want to
maintain. In our contrived example, we may want a cache for different
dictionaries. Making this change isn't too hard. You'd want to make a
channel yourself and then modify the _wait_ function to use that
channel when responding to changes. Then for each cache, you'd want to
run _wait_ for that cache in a goroutine.

Another way to improve upon this code might be to maintain the list of
peers more rigorously. In a catastrophic or network failure, the peer
might not be able to remove itself from Doozer. To solve this problem,
you could have a cron job periodically poll the servers on the list
and remove the servers that are not accessible.
