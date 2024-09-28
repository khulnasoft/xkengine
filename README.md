`xkengine` - Custom Kengine Builder
===============================

This command line tool and associated Go package makes it easy to make custom builds of the [Kengine Web Server](https://github.com/khulnasoft/kengine).

It is used heavily by Kengine plugin developers as well as anyone who wishes to make custom `kengine` binaries (with or without plugins).

Stay updated, be aware of changes, and please submit feedback! Thanks!

## Requirements

- [Go installed](https://golang.org/doc/install)

## Install

You can [download binaries](https://github.com/khulnasoft/xkengine/releases) that are already compiled for your platform from the Release tab. 

You may also build `xkengine` from source:

```bash
go install github.com/khulnasoft/xkengine/cmd/xkengine@latest
```

For Debian, Ubuntu, and Raspbian, an `xkengine` package is available from our [Cloudsmith repo](https://cloudsmith.io/~kengine/repos/xkengine/packages/):

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/kengine/xkengine/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/kengine-xkengine-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/kengine/xkengine/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/kengine-xkengine.list
sudo apt update
sudo apt install xkengine
```

## :warning: Pro tip

If you find yourself fighting xkengine in relation to your custom or proprietary build or development process, **it might be easier to just build Kengine manually!**

Kengine's [main.go file](https://github.com/khulnasoft/kengine/blob/master/cmd/kengine/main.go), the main entry point to the application, has instructions in the comments explaining how to build Kengine essentially the same way xkengine does it. But when you use the `go` command directly, you have more control over the whole thing and it may save you a lot of trouble.

The manual build procedure is very easy: just copy the main.go into a new folder, initialize a Go module, plug in your plugins (add an `import` for each one) and then run `go build`. Of course, you may wish to customize the go.mod file to your liking (specific dependency versions, replacements, etc).


## Command usage

The `xkengine` command has two primary uses:

1. Compile custom `kengine` binaries
2. A replacement for `go run` while developing Kengine plugins

The `xkengine` command will use the latest version of Kengine by default. You can customize this for all invocations by setting the `KENGINE_VERSION` environment variable.

As usual with `go` command, the `xkengine` command will pass the `GOOS`, `GOARCH`, and `GOARM` environment variables through for cross-compilation.

Note that `xkengine` will ignore the `vendor/` folder with `-mod=readonly`.


### Custom builds

Syntax:

```
$ xkengine build [<kengine_version>]
    [--output <file>]
    [--with <module[@version][=replacement]>...]
    [--replace <module[@version]=replacement>...]
    [--embed <[alias]:path/to/dir>...]
```

- `<kengine_version>` is the core Kengine version to build; defaults to `KENGINE_VERSION` env variable or latest.<br>
  This can be the keyword `latest`, which will use the latest stable tag, or any git ref such as:
  - A tag like `v2.0.1`
  - A branch like `master`
  - A commit like `a58f240d3ecbb59285303746406cab50217f8d24`

- `--output` changes the output file.

- `--with` can be used multiple times to add plugins by specifying the Go module name and optionally its version, similar to `go get`. Module name is required, but specific version and/or local replacement are optional.

- `--replace` is like `--with`, but does not add a blank import to the code; it only writes a replace directive to `go.mod`, which is useful when developing on Kengine's dependencies (ones that are not Kengine modules). Try this if you got an error when using `--with`, like `cannot find module providing package`.

- `--embed` can be used to embed the contents of a directory into the Kengine executable. `--embed` can be passed multiple times with separate source directories. The source directory can be prefixed with a custom alias and a colon `:` to write the embedded files into an aliased subdirectory, which is useful when combined with the `root` directive and sub-directive.

#### Examples

```bash
$ xkengine build \
    --with github.com/khulnasoft/ntlm-transport

$ xkengine build v2.0.1 \
    --with github.com/khulnasoft/ntlm-transport@v0.1.1

$ xkengine build master \
    --with github.com/khulnasoft/ntlm-transport

$ xkengine build a58f240d3ecbb59285303746406cab50217f8d24 \
    --with github.com/khulnasoft/ntlm-transport

$ xkengine build \
    --with github.com/khulnasoft/ntlm-transport=../../my-fork

$ xkengine build \
    --with github.com/khulnasoft/ntlm-transport@v0.1.1=../../my-fork
```

You can even replace Kengine core using the `--with` flag:

```
$ xkengine build \
    --with github.com/khulnasoft/kengine/v2=../../my-kengine-fork
    
$ xkengine build \
    --with github.com/khulnasoft/kengine/v2=github.com/my-user/kengine/v2@some-branch
```

This allows you to hack on Kengine core (and optionally plug in extra modules at the same time!) with relative ease.

---

If `--embed` is used without an alias prefix, the contents of the source directory are written directly into the root directory of the embedded filesystem within the Kengine executable. The contents of multiple unaliased source directories will be merged together:

```
$ xkengine build --embed ./my-files --embed ./my-other-files
$ cat Kenginefile
{
	# You must declare a custom filesystem using the `embedded` module.
	# The first argument to `filesystem` is an arbitrary identifier
	# that will also be passed to `fs` directives.
	filesystem my_embeds embedded
}

localhost {
	# This serves the files or directories that were
	# contained inside of ./my-files and ./my-other-files
	file_server {
		fs my_embeds
	}
}
```

You may also prefix the source directory with a custom alias and colon separator to write the source directory's contents to a separate subdirectory within the `embedded` filesystem:

```
$ xkengine build --embed foo:./sites/foo --embed bar:./sites/bar
$ cat Kenginefile
{
	filesystem my_embeds embedded
}

foo.localhost {
	# This serves the files or directories that were
	# contained inside of ./sites/foo
	root * /foo
	file_server {
		fs my_embeds
	}
}

bar.localhost {
	# This serves the files or directories that were
	# contained inside of ./sites/bar
	root * /bar
	file_server {
		fs my_embeds
	}
}
```

This allows you to serve 2 sites from 2 different embedded directories, which are referenced by aliases, from a single Kengine executable.

---

If you need to work on Kengine's dependencies, you can use the `--replace` flag to replace it with a local copy of that dependency (or your fork on github etc if you need):

```
$ xkengine build some-branch-on-kengine \
    --replace golang.org/x/net=../net
```

### For plugin development

If you run `xkengine` from within the folder of the Kengine plugin you're working on _without the `build` subcommand_, it will build Kengine with your current module and run it, as if you manually plugged it in and invoked `go run`.

The binary will be built and run from the current directory, then cleaned up.

The current working directory must be inside an initialized Go module.

Syntax:

```
$ xkengine <args...>
```
- `<args...>` are passed through to the `kengine` command.

For example:

```bash
$ xkengine list-modules
$ xkengine run
$ xkengine run --config kengine.json
```

The race detector can be enabled by setting `XKENGINE_RACE_DETECTOR=1`. The DWARF debug info can be enabled by setting `XKENGINE_DEBUG=1`.


### Getting `xkengine`'s version

```
$ xkengine version
```


## Library usage

```go
builder := xkengine.Builder{
	KengineVersion: "v2.0.0",
	Plugins: []xkengine.Dependency{
		{
			ModulePath: "github.com/khulnasoft/ntlm-transport",
			Version:    "v0.1.1",
		},
	},
}
err := builder.Build(context.Background(), "./kengine")
```

Versions can be anything compatible with `go get`.



## Environment variables

Because the subcommands and flags are constrained to benefit rapid plugin prototyping, xkengine does read some environment variables to take cues for its behavior and/or configuration when there is no room for flags.

- `KENGINE_VERSION` sets the version of Kengine to build.
- `XKENGINE_RACE_DETECTOR=1` enables the Go race detector in the build.
- `XKENGINE_DEBUG=1` enables the DWARF debug information in the build.
- `XKENGINE_SETCAP=1` will run `sudo setcap cap_net_bind_service=+ep` on the resulting binary. By default, the `sudo` command will be used if it is found; set `XKENGINE_SUDO=0` to avoid using `sudo` if necessary.
- `XKENGINE_SKIP_BUILD=1` causes xkengine to not compile the program, it is used in conjunction with build tools such as [GoReleaser](https://goreleaser.com). Implies `XKENGINE_SKIP_CLEANUP=1`.
- `XKENGINE_SKIP_CLEANUP=1` causes xkengine to leave build artifacts on disk after exiting.
- `XKENGINE_WHICH_GO` sets the go command to use when for example more then 1 version of go is installed.
- `XKENGINE_GO_BUILD_FLAGS` overrides default build arguments. Supports Unix-style shell quoting, for example: XKENGINE_GO_BUILD_FLAGS="-ldflags '-w -s'". The provided flags are applied to `go` commands: build, clean, get, install, list, run, and test
- `XKENGINE_GO_MOD_FLAGS` overrides default `go mod` arguments. Supports Unix-style shell quoting.

---

&copy; 2020 KhulnaSoft Ltd
