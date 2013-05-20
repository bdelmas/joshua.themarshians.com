<!-- Title: Hosting Your Own Godoc -->
<!-- Author: Joshua Marsh -->
<!-- Description: I sometimes find it useful to host my own version of golang.org. Let's learn how to do that. -->
<!-- Tags: linux,go,arch  -->

One of the biggest reasons to use [Go](http://golang.org/) is its
great tooling. [Godoc](http://golang.org/cmd/godoc/) is one of those
tools and it makes documentation so simple that you'll be wanting to
document your code instead of dreading it. Besides having a command
line interface for displaying documentation, it can also start an HTTP
Server. [Golang.org](http://golang.org/) is a public version of this
for the standard packages. [GoDoc](http://godoc.org/) does a similar
thing for public packages.

I have some private repositories though that I host internally and
I've always wanted to be able to view the package documentation. There
are at least a couple of ways to accomplish this. First, if you simply
want to host it your local machine, godoc can do that for you:

	$ godoc -http=:6060
	
This will create an HTTP web server that listens on port *6060* and
serves up the standard documentation as well as anything in your
*GOPATH*.

What I've always wanted to do though is have it run on the server
that's hosting all my repositories. I finally spent some time getting
this working and it turns out to be quite simple. Since the HTTP
Server is already built, we just need a way to get the system to
recognize it as a service. I use
[Arch Linux](https://www.archlinux.org/) and
[systemd](http://www.freedesktop.org/wiki/Software/systemd/) is the
default service manager. If you use something else, I'm guessing this
should translate fairly easily to the other services.

Since systemd can manage starting and stopping services using signals,
we really just need to tell systemd how to start up the godoc
server. Here is the
[godoc.service](https://github.com/icub3d/godoc-systemd/blob/master/godoc.service)
file in full:

	[Unit]
	Description=Go Documentation
	Wants=network.target

	[Service]
	Environment=GOPATH=/srv/go/ GOROOT=/usr/lib/go/
	ExecStart=/usr/bin/godoc -index -http=:6060
	RestartSec=30
	Restart=always

	[Install]
	WantedBy=multi-user.target

The meat of the configuration is *ExecStart*. It is simply a call to
godoc with the *-http* parameter. I added *-index* so that the
documentation could be searched. I thought this would be useful since
it's intended to be online full-time. 

One thing to note, I set the *GOPATH* to _/srv/go_. Since systemd
doesn't otherwise load environment variables, the *GOPATH* would not
normally be set. You can change that path to your actual *GOPATH*. I
simply used a symbolic link to my actual go repositories.

Copy your service file to a place where system will recognize it:

	$ sudo cp godoc.service /etc/systemd/system

That's all there is to it! I now have a fully functional, locally
hosted Go documentation web site. I use it like golang.org, but it
also searches and displays my private packages. I can use it like any
other systemd service:

	$ sudo systemctl enable godoc.service # Start at boot.
	$ sudo systemctl {start,stop,restart} godoc.service

If you are using Arch Linux, I made an
[AUR](https://aur.archlinux.org/) package named
[godoc-systemd](https://aur.archlinux.org/packages/godoc-systemd/). You
can install it like you would any other AUR package:

	$ wget https://aur.archlinux.org/packages/go/godoc-systemd/PKGBUILD
	$ makepkg
	$ sudo pacman -U godoc-systemd-1.0-1-any.pkg.tar.xz

I can't say enough about how impressed I am with the tooling built for
Go. Godoc is only one of many and it is a good example of how useful,
extendable, and manageable the go tools are.
