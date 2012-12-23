<!-- Title: Private Git Repos And Go -->
<!-- Author: Joshua Marsh -->
<!-- Tags: git,go -->
<!-- Languages: shell,go,html,perl -->

I spent a little bit of time this weekend tinkering on a small project
I've been thinking about for quite a while. Google's new(ish) language
Go seemed like a good candidate for the project. One of the great
features of [Go](http://golang.org/) is that your can import external
code sources relatively easily. I have a small private repository at
home and I wanted to simplify the process of importing the code and
maintaining the repositories. In the end, I chose
[Gitolite](https://github.com/sitaramc/gitolite) +
[Gitweb](https://git.wiki.kernel.org/index.php/Gitweb) on an
[Arch](https://www.archlinux.org/) Linux system. Here is how I did it.

Preparations
------------

Since we are using git, you'll want to make sure you have that
installed. We will also be using
[Apache httpd](http://httpd.apache.org/) for gitweb and to facilitate
go's remote import mechanism. On an Arch Linux system, you can do that
in a terminal with:

<pre><code data-language="shell">pacman -S git apache  
</code></pre>

The git package will create a git user that we will use during this
process. The user needs a little help though. As root:

<pre><code data-language="shell">usermod -d /srv/git git
passwd -d git
</code></pre>

We are simply telling the system that the users home directory should
be */srv/git* and it should not have a password (i.e. can't log
in). Make the home directory and give it the proper permissions:

<pre><code data-language="shell">mkdir /srv/git
chown git.git /srv/git
chmod 750 /srv/git
</code></pre>

We also want the http user to be in the git group so that gitweb can
read files from */srv/git* and then restart httpd. You can do this by
running the following commands:

<pre><code data-language="shell">usermod -a -G git http
systemctl restart httpd.service
</code></pre>

From this point on, I'll presume you are using the git user and are in
his *$HOME* directory unless otherwise specified.

Gitolite
--------

Gitolite is a great git repository management tool that is used by
quite a few high profile project (e.g. the
[Linux kernel](http://git.kernel.org/)). The big advantages it has
over using the standard git tools like
[git-daemon](http://www.kernel.org/pub/software/scm/git/docs/git-daemon.html)
or
[git-http-backend](http://www.kernel.org/pub/software/scm/git/docs/git-http-backend.html)
is that it is extremely simple to use and you can manage it remotely
once it is setup.

I simply pull the latest from [GitHub](https://github.com/) and do the
install:

<pre><code data-language="shell">git clone git://github.com/sitaramc/gitolite
mkdir $HOME/bin
gitolite/install -ln
</code></pre>

Gitolite uses SSH for authentication via public keys. As such, you'll
want to have your public key somewhere on the system. I copied mine to
*/tmp/username.pub*. Make sure your file's base name is the name of the
user you plan to log in with when accessing the repositories. Then run
the setup:

<pre><code data-language="shell">bin/gitolite setup /tmp/username.pub
</code></pre>

I made a few changes in *$HOME/.gitolite.rc* to facilitate httpd being
able to access the repositories. Modify the *UMASK* and
*GIT_CONFIG_KEYS* to:

<pre><code data-language="shell">UMASK           => 0027,
GIT_CONFIG_KEYS => 'gitweb\.(owner|description|category)*',
</code></pre>

This will give the git group the ability to read the repositories and allow the gitolite configuration file to store some basic gitweb configuration options. Finally, make sure that the repositories path is accessible by the git group:

<pre><code data-language="shell">chmod g+rx repositories
</code></pre>

Gitolite is ready to use! You configure it and setup repositories by cloning the gitolite-admin repository:

<pre><code data-language="shell">git clone git@git.hostname.com:gitolite-admin
</code></pre>

Here is my *conf/gitolite.conf* file:

<pre><code data-language="shell">@webs = config testing

repo gitolite-admin
    RW+     =   jmarsh

repo config
    RW+     =   jmarsh
  config gitweb.owner = joshua@themarshians.com
  config gitweb.description = A recursive JSON config parser.

repo testing
    RW+     =   @all
  config gitweb.owner = joshua@themarshians.com
  config gitweb.description = A test playground for connection and gitolite.

repo @webs
  R       =   gitweb
</code></pre>

I setup the *@webs* group to allow gitolite to manage the projects
list. The user *gitweb* is a special user for gitolite. Any repository
that givea that user read \(R) permissions will be added to the file
*$HOME/projects.list*. Gitweb can then use this file to display the
repositories you want visible. Any repositories that you want gitweb
to display simply need to be added to the *@webs* group. You can see
Gitolite's documentation for more information on what the rest of the
configurations mean.

Once you are satisfied with your configurations, commit the changes
and push it back to the server. Gitolite will do the rest, including
creating new repositories as necessary. Remember, gitolite will use
the SSH public key you gave it for authentication, so make sure you
are pushing from a system that has the corresponding private key.

Gitweb
------

Gitweb comes with git and is fairly easy to setup. I decided to use
Apache httpd (mostly because I'm more familiar with it) which adds a
few steps, but it is still relatively simple. You'll need to edit the
following configuration files and restart httpd as root. I'm presuming
you have a fresh install of httpd, so let's add a virtual host. Find
the line below in */etc/httpd/conf/httpd.conf* and uncomment it:

<pre><code data-language="shell">Include conf/extra/httpd-vhosts.conf
</code></pre>

Now edit */etc/httpd/conf/extra/httpd-vhosts.conf* and add the virtual
host at the end of the file:

<pre><code data-language="shell"><VirtualHost *:80>
    ServerName    gitserver
    ServerAlias   git.hostname.com
    DocumentRoot  /srv/http/gitweb
    SetEnv        GITWEB_CONFIG   /etc/conf.d/git-web.conf
    <Directory /srv/http/gitweb>
            Options ExecCGI FollowSymLinks
        AddHandler cgi-script cgi
        DirectoryIndex gitweb.cgi
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule ^.* /gitweb.cgi/$0 [L,PT]
    </Directory>
</VirtualHost>
</code></pre>

If you aren't using an Arch Linux package of httpd, you may need to
figure out the right place to add your virtual host, but the virtual
host information should work as is. We are basically using the gitweb
script as a CGI script. Since we are using */srv/http/gitweb* as the
base directory, we'll want to symlink that to gitweb's working
directory:

<pre><code data-language="shell">ln -s /usr/share/gitweb /srv/http/gitweb
</code></pre>

Next, restart httpd:

<pre><code data-language="shell">systemctl restart httpd.service
</code></pre>

With that done, we only need to modify gitweb's configuration file
*/etc/conf.d/git-web.conf*. Here is what mine looks like:

<pre><code data-language="perl">our $git_temp = "/tmp";
our $site_name = "git repos";
our $projectroot = "/srv/git/repositories";
our $projects_list = "/srv/git/projects.list";

$feature{'highlight'}{'default'} = [1];

@git_base_url_list = ( "ssh://git\@git.hostname.com" );


# We want to append the path request from the go get which we can get
# from the REQUEST_URI
$go_repo_path = $ENV{'REQUEST_URI'};
$go_repo_path =~ s/\?.*//g;
$go_prefix = $go_repo_path;
if ($go_repo_path !~ m/\.git$/ && $go_repo_path !~ m/\/$/) { $go_repo_path .= ".git"; }
our $site_html_head_string = "";
</code></pre>

The first half of the file is fairly common gitweb configuring. Note
that we set the *$projects_list* to be the *projects.list* file that
Gitolite is generating. The magic for Go happens in the last half of
the file. Go can pull from external version control systems (VCS), but
the format seemed bulky to me. Originally, I had to use an import
similar to:

<pre><code data-language="go">import "git.hostname.com/git/testing.git/package"
</code></pre>

The addition of the meta tag to Gitweb's pages made it possible to
import with something much more readable and more akin to importing
from a public repository like GitHub.

<pre><code data-language="go">import "git.hostname.com/testing/package"
</code></pre>

The Perl code essentially uses some regular expressions to generate a
meta tag that Go's remote pulling feature will recognize. In order for
it to work seamlessly Go needs the meta tag to include the exact path
being imported to map to an exact repository. We can get that
information from the *REQUEST_URI* that Go sends Gitweb. So, in my
example import above, the meta tag would end up looking something
like:

<pre><code data-language="html"><meta name="go-import" content="git.hostname.com/testing git ssh://git\@git.hostname.com/testing.git\"></meta>
</code></pre>

It Works!
---------

With that done, Gitweb is now setup to work well with Go's remote import feature. I can run something like

<pre><code data-language="go">go get git.hostname.com/config
</code></pre>

to get the package, or simply import that in other code and Go will
fetch it for me.

All of this came together after a lot of searching on the
internet. Here are some links I used to get this all working:

* [Go's get/remote documentation](http://golang.org/cmd/go/#hdr-Remote_import_path_syntax)
* [Gitolite's Manual](http://sitaramc.github.com/gitolite/master-toc.html)
* [Gitweb's gitweb.conf manual page](http://www.kernel.org/pub/software/scm/git/docs/gitweb.conf.html)
