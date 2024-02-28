---
title: "Automatic module versioning with golang"
date: 2024-02-28T13:17:46+01:00
---

One of the basic features most tools have is a command-line `--version` parameter. However, I realized this was not something trivial to do in a golang project maintained in a git repository. I'll describe the various approaches I took to implement automatic versioning in go, which involves some not so well-known go and git features.

### Using a go constant

The easiest way to implement this is to use a `version` constant in our go code:
```go
package main

const version = "v0.0.1"

func main() {
	fmt.Println("hello", version)
}
```
However, when the code is maintained in git, this comes with some limitations:
- when making a release, we need to remember to update the `version`variable before creating the release tag in git
- the information is very static, `version v0.0.1` does not indicate if the current binary corresponds to the released `v0.0.1` version, or some intermediate release built from git at some point between the `v0.0.1` and `v0.0.2`releases, or even to a build with unofficial patches.

### Using go linker flags

`git describe` can be used to get a more accurate version. If we run `git describe` on an commit corresponding to an annotated tag, we get `v0.0.1`.
If we run it on a commit made between `v0.0.1` and `v0.0.2` it prints something like `v0.0.1-3-gfa2d305`. `3` is the number of commits since the last release, and `fa2d305` is the abbreviated git commit hash for the current git HEAD.

In order to use `git describe`, we need a way of invoking it each time our program is built to get an up-to-date version. It's typically done using [go linker flags](https://www.digitalocean.com/community/tutorials/using-ldflags-to-set-version-information-for-go-applications)

`go build -ldflags="-X 'main.version=v0.0.2'"` will set the `version` variable in our test program to `v0.0.2` regardless of what its value is in the go program. `version`must be defined as a variable and not a constant:

```bash
$ cat main.go
package main

import "fmt"

var version = "v0.0.1"

func main() {
	fmt.Println("hello", version)
}

$ go build -ldflags="-X 'main.version=v0.0.2'" ./

$ ./hello-version
hello v0.0.2
```

We can combine this `-ldflags` with `git describe`: `go build -ldflags="-X 'main.version=$(git describe)'" ./`. If you are using a `Makefile` to build your project, this can be done using Makefile variables.

This approach avoids the 2 drawbacks we had with the use of a go constant.
However, this adds new problems we did not have with the simple solution:
- if the binary is installed with `go install gitlab.com/teuf/hello@latest`, we have no way of specifying the correct version using `-ldflags`
- if the build is done from a release tarball, there will be no associated git repository, and `git describe` won't generate a useful version

### Using golang build information

One way of getting a good version number when using `go install gitlab.com/teuf/hello@latest` is to make use of the [`debug.BuildInfo`  infrastructure.](https://pkg.go.dev/runtime/debug#BuildInfo)

`debug.ReadBuildInfo()

https://shibumi.dev/posts/go-18-feature/
https://pkg.go.dev/runtime/debug#BuildInfo


`make` from git is fine, `go install` is fine, release is made. Now I get a nice tarball on github, but what if I build from this tarball? `git describe` does not return something useful, `Main.Version` from `debug.ReadBuildInfo()` is `(devel)`.
### Using `.gitattributes`

There is still one use-case which is not covered in the previous paragraphs : building from a GitLab/GitHub tarball. The tarball does not contain any git metadata, so `git describe` cannot be used. As seen previously, `debug.BuildInfo`is also not usable for local builds, and even more so when there is no git information.

> If the attribute `export-subst` is set for a file then Git will expand several placeholders when adding this file to an archive. The expansion depends on the availability of a commit ID, i.e., if `git-archive(1)` has been given a tree instead of a commit or a tag then no replacement will be done. The placeholders are the same as those for the option `--pretty=format` of `git-log(1)`, except that they need to be wrapped like this: `$Format:PLACEHOLDERS` in the file. E.g. the string `$Format:%H$` will be replaced by the commit hash. However, only one `%(describe)` placeholder is expanded per archive to avoid denial-of-service attacks.


`/pkg/cmdline/version.go export-subst`
This is a git-level solution, and instructs git to replace ... with ...

### A few more caveats

If you build in github actions, it does a shallow build, so git describe won't return anything useful. Need to include the full history for a good --version
And there's a bug in actions/checkout causing 

### Conclusion

Automatically generating our program version in go is more complex than one would think, and requires different approaches to cover all the different ways your program can be built.

`--version` in [vfkit](https://github.com/crc-org/vfkit)is implemented as described in this post, support was added with these commits:
[version: Generate version using git describe](https://github.com/crc-org/vfkit/commit/da0b9b03519914b95405d93a69b8c0ce15514b2b)
[version: Add moduleVersionFromBuildInfo](https://github.com/crc-org/vfkit/commit/90e37b12212192cb05c6059c92d4922172cd67b3)
[version: Add versioning for github tarballs](https://github.com/crc-org/vfkit/commit/206bfd40ca48eb7603cdc6e36325f5cf4a2c889f)

