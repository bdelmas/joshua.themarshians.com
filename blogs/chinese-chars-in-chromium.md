<!-- Title: Chinese Character Rendering In Chromium -->
<!-- Tags: arch,chromium,chinese -->
<!-- Languages: shell -->
<!-- Author: Joshua Marsh -->

I've been using [Arch Linux](http://www.archlinux.org) for a month now
and have really been enjoying it. It's one of those "Google and
tinker" operating systems. Today I had a problem with
[Chromium](http://www.chromium.org/Home) refusing to render Chinese
characters. I don't speak Chinese, but It bugs me to see boxes where
cool characters should be.

It turns out to be a simple fix. Installing the package
[ttf-arphic-uming](https://www.archlinux.org/packages/extra/any/ttf-arphic-uming/)
did it for me. I thought I'd just throw this out there just in case
anyone else was getting infuriated by ugly rectangles.

<pre><code data-language="shell">$ sudo pacman -S ttf-arphic-uming  
</code></pre>
