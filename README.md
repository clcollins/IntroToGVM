#This repository is archived and will no longer receive updates.

# Using GVM to manage Go projects

GVM is a an Open Source tool for managing Go environemnts, including installing multiple versions of go, and managing modules per-project using gvm "pkgsets". Developed originally by [Josh Bussdieker](https://github.com/jbussdieker), GVM, like it's similar to it's Ruby counterpart RVM, allows you to create a development environment for each project or group of projects, segregating the different Go versions and package dependencies to allow for more flexibility and prevent versioning issues.

There are several options for managing Go packages, including Go 1.11 Modules, built right into Go.  I find GVM to be simple and intuitive, and even if I didn't use it to manage packages, I'd still use it to manage Go versions.

## Installing GVM

Installing GVM is straightforward.  The GVM repository [https://github.com/moovweb/gvm](https://github.com/moovwm) installation documentation instructs you to download and installer script and pipe it to bash:

```bash
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)
```

Despite the growing adoption of this kind of installation method, it's still good practice to take a look at what the installer is actually doing before you do so.  In the case of GVM, the installer script:

1. Checks some dependencies
2. Clones GVM repo
3. Uses shell scripts to install go, and
4. Manage the GOPATH environment, and,
5. adds a line to your bashrc, zshrc or profile

If you want to double-check what it's doing, you can clone the repo and check out the shell scripts, and then run ./binscripts/gvm-installer to set it up using the local scripts.

_Note:_ Since GVM can be used to download and compile new Go versions, there are some expected dependencies like make, git and curl.  You can find the complete list for your distribution in the [gvm README.md file](https://github.com/moovweb/gvm/blob/master/README.md)

## Installing and managing Go versions with GVM

Once GVM is installed, you can begin using it to install and managed different versions of Go.  The `gvm listall` command shows the available versions of Go that can be downloaded and compiled.

```bash
[chris@marvin ]$ gvm listall
$ gvm listall

gvm gos (available)

   go1
   go1.0.1
   go1.0.2
   go1.0.3

<output truncated>
```

Installing a specific Go version is as easy as `gvm install <version>`, where "version" is one of the available versions from the `gvm listall` command.

Let's say you're working on a project that uses Go version 1.12.8.  You can install it easily with `gvm install go1.12.8`:

```bash
[chris@marvin]$ gvm install go1.12.8
Installing go1.12.8...
 * Compiling...
go1.12.8 successfully installed!
```

Now, using `gvm list`, Go version 1.12.8 is listed as installed, along with the system Go version (the version that comes packed using your OS's package manager).

```bash
[chris@marvin]$ gvm list

gvm gos (installed)

   go1.12.8
=> system
```

You can see GVM is still using the system version of Go, denoted by the "=>" symbol next to it.  You can switch your environment to use the newly installed go1.12.8 with the `gvm use` command:

```bash
[chris@marvin]$ gvm use go1.12.8
Now using version go1.12.8

[chris@marvin]$ go version
go version go1.12.8 linux/amd64
```

GVM makes managing installed versions of Go extremely simple, but it gets even better!

## Using GVM pkgset

Out of the box, Go has a brilliant - and frustrating - way of managing packages or modules.  By default, if you `go get` a package, it is downloaded into the "src" and "pkg" directories in your `$GOPATH`, and they can then be included into your Go program with using `import`.  This makes getting packages easy, especially allowing unprivileged users to do so without requiring `sudo` or root privileges (much like `pip install --user` for Python).  The tradeoff, however, is difficulty managing different versions of the same packages across different projects.

There are a number of attempts at fixing or mitigating the issue, including the experimental Go Modules (preliminary support added in Go v1.11), go dep (an "official experiment" and ongoing alternative to Go Modules), etc.  Prior to discovering GVM, I would build and test Go projects in their own Docker containers to ensure segregation.

GVM accomplishes management and segregation of packaged between projects elegantly by using "pkgsets" - appending a new directory for projects to the default `$GOPATH` for the version of Go installed, much like `$PATH` works on Unix/Linux systems.

It is easiest to visualize how this works in action.  First, we'll install a new version of Go, v1.12.9.

```bash
[chris@marvin]$ echo $GOPATH
/home/chris/.gvm/pkgsets/go1.12.8/global

[chris@marvin]$ gvm install go1.12.9
Installing go1.12.9...
 * Compiling...
go1.12.9 successfully installed

[chris@marvin]$ gvm use go1.12.9
Now using version go1.12.9
```

When GVM is told to use a new version, it changes to a new `$GOPATH`, which corresponds to a default "gloabl" pkgset for that version:

```bash
[chris@marvin]$ echo $GOPATH
/home/chris/.gvm/pkgsets/go1.12.9/global

[chris@marvin]$ gvm pkgset list

gvm go package sets (go1.12.9)

=>  global
```

Packages in the global pkgset are available to any project using this specific version of Go, though by default there are no extra packages installed.

Now, suppose a new project is being started and it needs a specific package.  First, GVM is used to create a new pkgset, called "introToGvm":

```bash
[chris@marvin]$ gvm pkgset create introToGvm

[chris@marvin]$ gvm pkgset use introToGvm
Now using version go1.12.9@introToGvm

[chris@marvin]$ gvm pkgset list

gvm go package sets (go1.12.9)

    global
=>  introToGvm
```

As mentioned above, a new directory for the pkgset is prepended to the `$GOPATH`:

```bash
[chris@marvin]$ echo $GOPATH
/home/chris/.gvm/pkgsets/go1.12.9/introToGvm:/home/chris/.gvm/pkgsets/go1.12.9/global
```

Let's change directories into the "introToGvm" path that was prepended, and examine the directory structure - and take the opportunity to have some fun with `awk` and `bash` to do so!

```bash
[chris@marvin]$ cd $( awk -F':' '{print $1}' <<< $GOPATH )
[chris@marvin]$ pwd
/home/chris/.gvm/pkgsets/go1.12.9/introToGvm

[chris@marvin]$ ls
overlay  pkg  src
```

Notice that the new directory looks a lot like a normal `$GOPATH`.  New Go packages can be downloaded using the same `go get` command you'd use normally with Go, and they are added to the pkgset.

As an example, we'll get the "gorilla/mux" package, and then examine the directory structure of the pkgset:

```bash
[chris@marvin]$ go get github.com/gorilla/mux
[chris@marvin]$ tree
[chris@marvin introToGvm ]$ tree
.
├── overlay
│   ├── bin
│   └── lib
│       └── pkgconfig
├── pkg
│   └── linux_amd64
│       └── github.com
│           └── gorilla
│               └── mux.a
src/
└── github.com
    └── gorilla
        └── mux
            ├── AUTHORS
            ├── bench_test.go
            ├── context.go
            ├── context_test.go
            ├── doc.go
            ├── example_authentication_middleware_test.go
            ├── example_cors_method_middleware_test.go
            ├── example_route_test.go
            ├── go.mod
            ├── LICENSE
            ├── middleware.go
            ├── middleware_test.go
            ├── mux.go
            ├── mux_test.go
            ├── old_test.go
            ├── README.md
            ├── regexp.go
            ├── route.go
            └── test_helpers.go
```

As you can see, "gorilla/mux" was added to the pkgset `$GOPATH` directory as expected, and can now be used with projects that make use of this pkgset.

## Conclusion

GVM is an intuitive and non-intrusive way to manage Go versions and packages. It can be used on it's own, or in combination with other Go module management techniques, making use of GVM's Go version management.  Segregating projects by Go version and package dependencies makes development easier and leads to less complication managing version conflicts, and GVM makes doing so a breeze.
