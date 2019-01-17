title: Rhinowiki
date: 2018-08-03

It's been a long time coming, but I've finally replaced
[blosxom](http://blosxom.sourceforge.net/) with a *custom CMS* I've
been writing called
[Rhinowiki](https://github.com/mschaef/rhinowiki). More than a serious
attempt at a CMS, this is mainly a fun little side project to write
some [Clojure](https://clojure.org/), experiment a bit with
[JGit](https://www.eclipse.org/jgit/), and hopefully make it easier to
implement a few of my longer term plans that might have been tricky to
do in straight Perl.

Full source in the link above, a high level summary here:

* Everything is in [Clojure](https://clojure.org/).
* Backend format is [Markdown](https://daringfireball.net/projects/markdown/) as interpreted by [`markdown-clj`](https://github.com/yogthos/markdown-clj).
* Source code is highlighted using [`highlight.js`](https://highlightjs.org/).
* Markdown rendering is done entirely on the server, with syntax highlighting on the client.  (I'm looking into [Nashorn](http://openjdk.java.net/projects/nashorn/) to run `highlight.js` server side too, but don't know if that's possible within my time constraints.)
* Back end storage is managed using  and retrieved via [JGit](https://www.eclipse.org/jgit/).
* All requests are served out of memory.
* There's a [hand rolled](https://github.com/mschaef/rhinowiki/blob/master/src/rhinowiki/atom.clj) (and conformant) [Atom](https://tools.ietf.org/html/rfc4287) feed.
* [Also](https://github.com/mschaef/rhinowiki/blob/master/src/rhinowiki/rss.clj) [RSS 2.0](http://cyber.harvard.edu/rss/rss.html).


