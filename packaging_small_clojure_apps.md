title: Packaging Small Clojure Apps
date: 2020-08-12
tags: tech clojure

Like a lot of engineers, I have a handful of personal projects I keep
around for various reasons. Some are useful and some are just for fun,
but none of them get the same sort of investment as a funded
commercial effort. The consequence of this is that it's all the more
important to keep things as simple as possible, to focus the
investment where it counts. Part of the way I achieve that is that
I've spent some initial time putting together a standard packaging
approach. I know, I know - *"standard packaging approach"* doesn't
sound like *"fun personal project"* - but getting the packaging out of
the way makes it easier to focus on the actual fun part - building out
functionality. It's for that reason that I've also successfully used
variants of this approach on smaller commercial projects too. Hopefully,
this will be useful to you too.

Setting the state, the top level view is this:

* Uberjar packaging of single binaries using [Leiningen](https://leiningen.org/) and a few plugins.
* Standard scripts and tools for packaging and install.
* Use of existing Linux mechanisms for service control.
* A heavy tendancy toward [12 Factor](https://12factor.net/) principles.

What this gets you is a good local interactive development story and
easy deployment to a server. I've also gotten it to work with Client
side code too, using [Figwheel](https://figwheel.org/)

What it doesn't get you is direct support for large numbers of
processes or servers.  Modern hardware is fast and capable, so you may
not have those requirements, but if you do, you'll need something
heavier weight, to reduce both management overhead and costs. (In my
[day job](https://projectdabl.com/), we've done some amazing things
with [Kubernetes](https://kubernetes.io/).)

The example project I'm going to use is the engine for this blog,
[Rhinowiki](https://github.com/mschaef/rhinowiki). It's useful, but
simple enough to be used as a model for a packaging approach. If
you're also interested in strategies for managing apps with read/write
persistance (SQL) and rich client code, I have a couple other programs
packaged this way with those features. Even with these, the essentials
of the packaging strategy are exactly the same as what I describe
here:

* [Toto](https://github.com/mschaef/toto) - Web app with persistance
  to an embedded SQL database managed with
  [sql-file](https://github.com/mschaef/sql-file).
* [Metlog](https://github.com/mschaef/metlog) - Single Page App done
  with a Clojure back end, [Reagent](https://reagent-project.github.io/),
  [ClojureScript](https://clojurescript.org/), and [Figwheel](https://figwheel.org/).

Everything begins with a traditional
[`project.clj`](https://github.com/mschaef/rhinowiki/blob/master/project.clj),
and the project can be started locally with the usual `lein run`.

Once running, `main` immediately writes a herald log message at `info`
level:

```clojure
(defn -main [& args]
  (log/info "Starting Rhinowiki" (get-version))
  (let [config (load-config)]
    (log/debug "config" config)
    (webserver/start (:http-port config)
                     (blog/blog-routes (blog/blog-init config)))
    (log/info "end run.")))
```

This immediately lets you know the process has started, logs are
working, and which version of the code is running. These are usually
the first things verified after an install, so it's good to ensure
they happen early on. This is particularly useful for software that's
not interactive or running on slow hardware. (I've run some of this
code on Raspberry Pi hardware that takes ten or so seconds to get to
the startup herald.)

The way the version is acquired is interesting too. The call to
`get-version` is really a macro invocation and not a function call.

```clojure
(defmacro get-version []
  ;; Capture compile-time property definition from Lein
  (System/getProperty "rhinowiki.version"))

```

Because macros are evaluated at compile time, the macroexpansion of
`get-version` has access to JVM properties defined at build time by
Leiningen.

The next step is to pull in configuration settings using Anatoly
Polinsky's [https://github.com/tolitius/cprop](cprop) library. `cprop`
can do more than what I use it for here, but here, I use it to load a
single EDN config file. `cprop` lets the name of that file be
identified at startup via a system proprety, making it possible to
define a file for each server, as well as a local config file
specified in: `project.clj`.

```clojure
:jvm-opts ["-Dconf=local-config.edn"]
```

I've also found it useful to minimize the number and power of
configuration settings. Every setting that changes is a risk that
something will break when you promote code. Every setting that doesn't
change is a risk of introducing a bug in the settings code.

I also dump the configugration to a log very early in the startup
process.

```clojure
(log/debug "config" config)
```

Given the importance of configuration settings, it's occasionally
important to be able to inspect the settings in use at
run-time. However, this log is written at `debug` level, so it doesn't
normally print. This reduces the risk of accidentally revealing secret
keys in the log stream. Depending on the importance of those keys,
there is also much more you can do to protect them, if preventing
the risk is worth the effort.

After all that's done, `main` transfers control over to the actual
application:

```clojure
(webserver/start (:http-port config)
                 (blog/blog-routes (blog/blog-init config)))
```

With a configurable application running, the next step is to get it
packaged in a way that lets us predictably install it elsewhere. The
strategy here is a two step approach: build the code as an uberjar and
include the uberjar in a self-contained `.tar.gz` as an installation
pacakge.

* The installer package contains everything needed to install the
  software (the one exception being the JVM itself).
* The package name includes the version number of the software:
  `rhinowiki-0.3.3.tar.gz`.
* Files in the installation package all have a prefix
  (`rhinowiki-install`, in this case) to confine the installation
  files to a single directory when installing. This is to make it easy
  to avoid crosstalk between multiple installers and delete installation
  directories after you're done with an installation.
* There is an [idempotent](/?tag=idempotence) installation script
  ([`install.sh`](https://github.com/mschaef/rhinowiki/blob/master/pkg/install.sh))
  at the root of the package. Running this script either creates or
  updates an installation.
* The software is installed as a
  [Linux service](https://github.com/mschaef/rhinowiki/blob/master/pkg/rhinowiki).

The net result of this packaging in an installation/upgrade process
that works like this:

```bash
tar xzvf rhinowiki-0.3.3.tar.gz
cd rhinowiki-install
sudo service rhinowiki stop
sudo ./install.sh
sudo service rhinowiki start
```

To get to this point, I use the [Leiningen](https://leiningen.org/)
`release` task and the [`lein-tar`](https://github.com/technomancy/lein-tar)
plugin, both originally by Phil Hagelberg. There's a
[wrapper script](https://github.com/mschaef/rhinowiki/blob/master/package.sh),
but the essential command is `lein clean && lein release $RELEASE_LEVEL`.
This instructs Leiningen to execute a series of tasks listed in the
[`release-tasks` key in `project.clj`](https://github.com/mschaef/rhinowiki/blob/master/project.clj#L42).

I've had to modify Leiningen's default list of release tasks, in two
ways: I skip signing of tagged releases in git, and I invoke
`lein-tar` rather than `deploy`. However, the full task list needs to
be [completely restated in
`project.clj`](https://github.com/mschaef/rhinowiki/blob/master/project.clj#L42),
so it's a lengthy setting.

```clojure
:release-tasks [["vcs" "assert-committed"]
                ["change" "version" "leiningen.release/bump-version" "release"]
                ["vcs" "commit"]
                ["vcs" "tag" "--no-sign" ]
                ["tar"]
                ["change" "version" "leiningen.release/bump-version"]
                ["vcs" "commit"]
                ["vcs" "push"]]
```

The configuration for `lein-tar` is more straightforward - include the
plugin, and specify a few options. The options request that the
packaged output be written in the project root, include an uberjar,
and extract into an install directory rather than just CWD.

```clojure
:plugins [[lein-ring "0.9.7"]
          [lein-tar "3.3.0"]]

;; ...

:tar {:uberjar true
      :format :tar-gz
      :output-dir "."
      :leading-path "rhinowiki-install"}
```


Give the uberjar a specific fixed name:

```clojure
:uberjar-name "rhinowiki-standalone.jar"
```

And populate it with a few files additional to the uberjar itself -
`lein-tar` accepts these files in
[`pkg/`](https://github.com/mschaef/rhinowiki/tree/master/pkg) at the
root of the project directory hierarchy. These files include
everything else needed to install the application - a configuration
map for `cprop`, an install script, a service script, and log
configuration.

The install script is the last part of the process. It's an
[idempotent](/?tag=idempotence) script that, when run on a server as
`sudo`, guarantees that the application is installed. It sets up users
and groups, copies files from the package to wherever they belong, and
uses
[`update-rc.d`](http://manpages.ubuntu.com/manpages/xenial/man8/update-rc.d.8.html)
to ensure that the service scripts are correctly installed.

This breaks down the packaging and installation process to the
following:

* `./package.sh`
* `scp` package tarball to server and `ssh` in
* Extract the package - `tar xzvf rhinowiki-0.3.3.tar.gz`
* Change into the expanded package directory - `cd rhinowiki-install`
* Stop any existing instances of the service - `sudo service rhinowiki stop`
* Run the install script - `sudo ./install.sh`
* (Re)Start the service - `sudo service rhinowiki start`

At this point, I've sketched out the approach end to end, and I hope
it's evident that this can be used in fairly simple scenarios.  Before
I close, let me also talk about a few sharp edges to be aware of.
Like every other engineering approach, this packaging strategy has
tradeoffs, and some of these tradeoffs require specific compromises.

The first is that this approach requires dependencies (notably the
JVM) to be manually installed on target servers. For smaller
environments, this can be acceptable, for larger numbers of target
VM's, almost definately not.

The second is that there's nothing about persistance in this
approach. It either needs to be managed externally, or the entire
persistance story needs to be internal to the deployed uberjar. This
is why I wrote [`sql-file`](https://github.com/mschaef/sql-file),
which provides a built in SQL database with schema migration
support. Another approach is just to handle it altogether externally,
which is what I do for Rhinowiki. The Rhinowiki store is a git
repository, and it's managed out of band with respect to the
deployment of Rhinowiki itself.

But these are both specific problems that can be managed for smaller
applications. Often times, it's worth the costs associated with these
problems, to gain the benefits of reducing the number of software
components and moving pieces. If you're in a situation like that, I
hope you give this approach a try and find it useful. Please let me
know if you do.
