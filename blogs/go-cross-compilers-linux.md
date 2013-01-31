<!-- Title: Cross Compiling Go Applications -->
<!-- Author: Joshua Marsh -->
<!-- Tags: go,arch -->
<!-- Languages: go,shell -->

In a recent discussion on
[Go's Google Plus Community](https://plus.google.com/communities/114112804251407510571),
some questions came up about cross compiling. In a default
installation of [Go](http://golang.org/), cross compiling is not
enabled. It only builds to toolchains for your current operating
system and architecture. Since all of my computers are *amd64*, but
some of the applications I build need to run on *i386*, I thought I'd
share how I go about doing it.

I use [Arch](https://www.archlinux.org/) linux, so I have provided a
package in
[AUR](https://aur.archlinux.org/packages/go-cross-compilers-linux/)
for doing this. It makes setup extremely simply. You can install the
package by doing the following:

<pre><code data-language="shell">$ wget https://aur.archlinux.org/packages/go/go-cross-compilers-linux/PKGBUILD
$ makepkg

$ # For amd64
$ sudo pacman -U go-cross-compilers-linux-1.0.3-1-x86_64.pkg.tar.xz

$ # For i386
$ sudo pacman -U go-cross-compilers-linux-1.0.3-1-i686.pkg.tar.xz
</code></pre>

Now that the other toolchain is in place, we can make a sample program
to test it out:

<pre><code data-language="go">package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello world!")
}
</code></pre>

If you were building it for your own architecture, something as simple
as the following would work:

<pre><code data-language="shell">$ go build main.go
$ ./main
Hello world!
</code></pre>

Building for the other architecture isn't much more difficult. Go will
cross compile other operating systems and architectures based on the
environmental variables *GOOS* and *GOOARCH*. In our case, we are
interested in changing *GOARCH*:

<pre><code data-language="shell">$ GOARCH=386 go build main.go
</code></pre>

Now you can copy main over your *i386* machine and run it! If you were
cross compiling to *amd64*, you'd simply replace *386* with *amd64*.

One last interesting tidbit is that if you have a version of
[gcc](http://gcc.gnu.org/) that will cross compile, then Go
applications that link to C libraries can also be used (despite what
many people have suggested). In Arch, you can install a cross
compiling version of *gcc* like this:

<pre><code data-language="shell">$ sudo pacman -S gcc-multilib
</code></pre>

In this example, I'm going to test a [sqlite3](http://www.sqlite.org/)
library, so let's install the 32-bit libraries:

<pre><code data-language="shell">$ sudo pacman -S lib32-sqlite
</code></pre>

Now, we let's build a small app with the example from
[go-sqlite3](https://github.com/mattn/go-sqlite3):

<pre><code data-language="shell">$ go get github.com/mattn/go-sqlite3
$ mkdir testsql
$ cd testsql
$ wget https://raw.github.com/mattn/go-sqlite3/master/example/main.go

$ # Test the program locally to make sure it works.
$ go run main.go

$ # Build for i386
GOARCH=386 go build

$ # Verify it's linked against the right libraries (lib32).
$ ldd testsql
	linux-gate.so.1 (0xf77ba000)
	libsqlite3.so.0 => /usr/lib32/libsqlite3.so.0 (0xf76e2000)
	libpthread.so.0 => /usr/lib32/libpthread.so.0 (0xf76c7000)
	libc.so.6 => /usr/lib32/libc.so.6 (0xf7516000)
	libdl.so.2 => /usr/lib32/libdl.so.2 (0xf7511000)
	/lib/ld-linux.so.2 (0xf77bb000)

$ # Test is on i386.
scp testsql jmarsh@i386-box:
ssh jmarsh@i386-box
./testsql﻿
</code></pre>

Obviously this could all be alleviated by using virtual machines, but
the ability of the go toolchain to do this on a single system is
really powerful. I can't begin to express how useful this is on many
of the projects I've worked on. I'm always looking to improve, so if
you can see ways to improve my PKGBUILD file or if you've found a
simpler way to do something, let me know.
