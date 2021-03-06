goxc
====

[goxc](https://github.com/laher/goxc) is a build tool for Go, with a focus on cross-compiling and packaging.

By default, goxc [g]zips (& .debs for Linux) the programs, and generates a 'downloads page' in markdown (with a Jekyll header). Goxc also provides integration with [bintray.com](https://bintray.com) for simple uploads.

goxc is written in Go but uses *os.exec* to call 'go build' with the appropriate flags & env variables for each supported platform.

goxc was inspired by Dave Cheney's Bash script [golang-crosscompile](https://github.com/davecheney/golang-crosscompile).
BUT, goxc crosscompiles to all platforms at once. The artifacts are saved into a directory structure along with a markdown file of relative links.

Thanks to [dchest](https://github.com/dchest) for the tidy-up and adding the zip feature, and [matrixik](https://bitbucket.org/matrixik) for his improvements and input. Thanks also to all the helpful issue reporters.

**NOTE: Version 0.8.5 no longer supports go1.0. If you want go1.0 support please use goxc version 0.8.4 (see the git tag 'v0.8.4')**
*the reason why I dropped go1.0 support is mentioned in issue #16*

Notable Features
----------------
 * Cross-compilation, to all supported platforms, or a specified subset.
 	* Validation of toolchain & verification of cross-compiled artifacts
 	* Specify target platform, via 'Build Constraint'-like syntax (via commandline flag e.g. `-bc="windows linux,!arm"`, or via config)
 * *Automatic* (re-)building toolchain to all or specified platforms.
 * 'task' based invocation, similar to 'make' or 'ant'. e.g. `goxc xc` or `goxc clean go-test` 
	* The 'default' task alias will, test, cross-compile, verify, package up your artifacts for each platform, and generate a 'downloads page' with links to each platform. 
 	* Various go tools available via tasks: `go-test`, `go-vet`, `go-fmt`, `go-install`, `go-clean`.
	* You can modify task behaviour via configuration or commandline flags.
 * JSON-based configuration files for repeatable builds. 
 	* Most config data can be written via flags (using -wc option) - less JSON fiddliness.
	* Includes support for multiple configurations per-project.
 	* Per-task configuration options.
	* 'Override' config files for 'local environment' - working-copy-specific, or branch-specific, configurations.
 * Packaging & distribution
 	* Zip (or tar.gz) archiving of cross-compiled artifacts & accompanying resources (READMEs etc)
 	* Packaging into .debs (for Debian/Ubuntu Linux)
 	* bintray.com integration (deploys binaries to bintray.com). *bintray.com registration required*
 	* 'downloads page' generation (markdown/html format; templatable).
 * Versioning:
	* track your version number via configuration data.
	* version number interpolation at compile-time (uses go's `-ldflags` compiler option to populate given constants or global variables with build version or build date)
	* version number interpolation of source code. `goxc interpolate-source` (new task available in 0.10.x).
	* the `bump` task facilitates increasing the app version number.
 * support for multiple binaries per project (goxc now searches subdirectories for 'main' packages)
 * support for multiple Go installations - choose at runtime with `-goroot=` flag.

Installation
--------------
goxc requires the go source and the go toolchain.

 1. [Install go from source](http://golang.org/doc/install/source). (Requires gcc (or MinGW) and 'hg')

	* OSX Users Note: If you are using XCode 5 (OSX 10.9), it is best to go straight to Go 1.2rc5 (or greater). This is necessary because Apple have replaced the standard gcc with llvm-gcc, and Go 1.1 compilation tools depend on the usual gcc.

		* There is another workaround incase Go 1.2rc5 is not an option:

				brew tap homebrew/versions
				brew install apple-gcc42
				go get github.com/laher/goxc
				CC=`brew list apple-gcc42 | grep bin/gcc-4.2` goxc -t

 2. Install goxc:

		go get github.com/laher/goxc
			
 3. a. *Optional*: to pre-build the toolchains for all platforms:

		goxc -t


	* Note that this step is now optional, because goxc now builds/rebuilds the cross-compiler toolchains automatically, as necessary). 
	* Also note that building the toolchain takes a while. Cross-compilation itself is quite fast.

Basic Usage
-----------

	cd path/to/app/dir
	goxc


More options
------------

Use `goxc -h` to list all options.

 * e.g. To restrict by OS and Architecture (using the same syntax as Go Build Constraints):

		goxc -bc="linux,!arm windows,386 darwin"
	
	* Note that build constraints are described in Go's ['build' package documentation](http://golang.org/pkg/go/build/) (in the overview section).

 * e.g. To set a destination root directory and artifact version number:

		goxc -d=my/jekyll/site/downloads -pv=0.1.1

 * 'Package version' can be compiled into your app if you define a VERSION variable in your main package.

"Tasks"
-------

goxc performs a number of operations, defined as 'tasks'. You can specify tasks as commandline arguments

 * `goxc -t` performs one task called 'toolchain'. It's the equivalent of `goxc -d=~ toolchain`
 * The *default* task is actually several tasks, which can be summarised as follows:
    * validate (tests the code) -> compile (cross-compiles code) -> package ([g]zips up the executables and builds a 'downloads' page)
 * You can specify one or more tasks, such as `goxc go-fmt xc`
 * You can skip tasks with '-tasks-='. Skip the 'package' stage with `goxc -tasks-=package`
 * For a list of tasks and 'aliases', run `goxc -h tasks`
 * Several tasks have options available for overriding. You can specify them in config or via flags. Just use `goxc <taskname> -task-setting=value <othertask>`
 * For more info on a particular task, run `goxc -h <taskname>`. This will also show you the options available for that task.
 * The easiest way to see how to configure tasks in config is to write some task config via `-wc`, e.g. `goxc -wc xc -GOARM=5`

Outcome
-------

By default, artifacts are generated and then immediately archived into (outputdir).

Examples:

 * /my/outputdir/0.1.1/myapp\_0.1.1\_linux\_arm.tar.gz
 * /my/outputdir/0.1.1/myapp\_0.1.1\_windows\_386.zip
 * /my/outputdir/0.1.1/myapp\_0.1.1\_linux\_386.deb

The version number is specified with -pv=0.1.1 .

By default, the output directory is ($GOBIN)/(appname)-xc, and the version is 'unknown', but you can specify these.

e.g.

      goxc -pv=0.1.1 -d=/home/me/myuser-github-pages/myapp/downloads/

*NOTE: it's **bad idea** to use project-level github-pages - your repo will become huge. User-level gh-pages are an option, but it's better to use the 'bintray' tasks.*:

If non-archived, artifacts generated into a directory structure as follows:

 (outputdir)/(version)/(OS)\_(ARCH)/(appname)(.exe?)

Configuration file
-----------------

For repeatable builds (and some extra options), it is recomended to use goxc with one or more configuration file(s) to save and re-run compilations.

To create a config file, just use the -wc (write config) option.

	goxc -wc -d=../site/downloads -bc="linux windows" xc -GOARM=7

You can also use multiple config files to support different paremeters for each platform.

The configuration file(s) feature is documented in much more detail in [the wiki](https://github.com/laher/goxc/wiki/config)

Limitations
-----------

 * Tested on Linux, Windows (and Mac during an early version). Please test on Mac and \*BSD
 * Currently goxc is only designed to build standalone Go apps without linked libraries. You can try but YMMV
 * The *API* is not considered stable yet, so please don't start embedding goxc method calls in your code yet - unless you 'Contact us' first! Then I can freeze some API details as required.
 * Bug: issue with config overriding. Empty strings do not currently override non-empty strings. e.g. `-pi=""` doesnt override the associated config setting PackageInfo

License
-------

   Copyright 2013 Am Laher

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.

See also
--------
 * [Changelog](https://github.com/laher/goxc/wiki/changelog)
 * [Package Versioning](https://github.com/laher/goxc/wiki/versioning)
 * [Wiki home](https://github.com/laher/goxc/wiki)
 * [Contributions](https://github.com/laher/goxc/wiki/contributions)
 * [TODOs](https://github.com/laher/goxc/wiki/todo)
