# nanoproxy

This is a tiny HTTP forward proxy written in Go, for me to gain experience in the Go language.

This proxy accepts all requests and forwards them directly to the origin server. It performs no caching.

Despite this not being a full proxy implementation, it is blazing fast. In particular it is significantly faster than Squid and slightly faster than Apache's mod_proxy. This demonstrates that Go's built-in HTTP library is of a very high quality and that the Go runtime is quite performant.

## Prerequisites

- go 1.22.3
- toolchain go1.22.5

## Build & Run

Clone

```shell
git clone https://github.com/allenliu88/nanoproxy.git
cd nanoproxy
```

For Mac Apple M1

```shell
CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 go build -o bin/nanoproxy-darwin-arm64 nanoproxy.go

chmod +x bin/nanoproxy-darwin-arm64
nohup bin/nanoproxy-darwin-arm64 > nanoproxy.log 2>&1 &
```

For Linux

```shell
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/nanoproxy-linux-amd64 nanoproxy.go

chmod +x bin/nanoproxy-linux-amd64
nohup bin/nanoproxy-linux-amd64 > nanoproxy.log 2>&1 &
```

For Windows

```shell
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -o bin/nanoproxy-windows-amd64.exe nanoproxy.go

chmod +x bin/nanoproxy-windows-amd64.exe
nohup bin/nanoproxy-windows-amd64.exe > nanoproxy.log 2>&1 &
```

Or just run

```shell
go run nanoproxy.go
```

Validate port 8080 is open and open the firewall

```shell
## 查看运行详情，默认是监听8080端口
netstat -tunlp | grep 8080
tcp6       0      0 :::8080                 :::*                    LISTEN      19598/./nanoproxy   

## 开放端口
firewall-cmd --zone=public --add-port=8080/tcp --permanent
## 重载配置
firewall-cmd --reload
```

Configure your web browser to route all HTTP traffic through `localhost:8080`.

## Validation

本机验证通过，注意，`-x`参数为设置代理：

```shell
curl -x 1.1.1.1:8080 2.2.2.2:80
```

其中，`1.1.1.1`是`nanoproxy`所在节点，也即`1.1.1.1:8080`为代理服务器地址，`2.2.2.2:80`为目标`HTTP`服务。

## Notes

- Go's HTTP server implementation is really good. I read it all. Only missing feature I desire is the ability to process multiple pipelined HTTP requests in parallel.
- Go's HTTP client implementation is easy to use, based on my limited experience in this proxy. I have not read its implementation.

## 如何在日志中添加`vcs`信息？

实际上借助了`knative.dev/pkg/changeset`。参考[knative/pkg/blob/main/changeset/commit.go](https://github.com/knative/pkg/blob/main/changeset/commit.go)：Package changeset returns version control info that was embedded in the golang binary.

```go
package changeset

import (
	"regexp"
	"runtime/debug"
	"strconv"
	"sync"
)

const Unknown = "unknown"

var (
	shaRegexp = regexp.MustCompile(`^[a-f0-9]{40,64}$`)
	rev       string
	once      sync.Once

	readBuildInfo = debug.ReadBuildInfo
)

// Get returns the 'vcs.revision' property from the embedded build information
// If there is no embedded information 'unknown' will be returned
//
// The result will have a '-dirty' suffix if the workspace was not clean
func Get() string {
	once.Do(func() {
		if rev == "" {
			rev = get()
		}
		// It has been set through ldflags, do nothing
	})

	return rev
}

func get() string {
	info, ok := readBuildInfo()
	if !ok {
		return Unknown
	}

	var revision string
	var modified bool

	for _, s := range info.Settings {
		switch s.Key {
		case "vcs.revision":
			revision = s.Value
		case "vcs.modified":
			modified, _ = strconv.ParseBool(s.Value)
		}
	}

	if revision == "" {
		return Unknown
	}

	if shaRegexp.MatchString(revision) {
		revision = revision[:7]
	}

	if modified {
		revision += "-dirty"
	}

	return revision
}
```

进一步参考[runtime/debug.ReadBuildInfo()](https://pkg.go.dev/runtime/debug#ReadBuildInfo): ReadBuildInfo returns the build information embedded in the running binary. The information is available only in binaries built with module support.

```go
func ReadBuildInfo() (info *BuildInfo, ok bool)

type BuildInfo struct {
	// GoVersion is the version of the Go toolchain that built the binary
	// (for example, "go1.19.2").
	GoVersion string

	// Path is the package path of the main package for the binary
	// (for example, "golang.org/x/tools/cmd/stringer").
	Path string

	// Main describes the module that contains the main package for the binary.
	Main Module

	// Deps describes all the dependency modules, both direct and indirect,
	// that contributed packages to the build of this binary.
	Deps []*Module

	// Settings describes the build settings used to build the binary.
	Settings []BuildSetting
}

type BuildSetting struct {
	// Key and Value describe the build setting.
	// Key must not contain an equals sign, space, tab, or newline.
	// Value must not contain newlines ('\n').
	Key, Value string
}
```

A BuildSetting is a key-value pair describing one setting that influenced a build.

Defined keys include:

- -buildmode: the buildmode flag used (typically "exe")
- -compiler: the compiler toolchain flag used (typically "gc")
- CGO_ENABLED: the effective CGO_ENABLED environment variable
- CGO_CFLAGS: the effective CGO_CFLAGS environment variable
- CGO_CPPFLAGS: the effective CGO_CPPFLAGS environment variable
- CGO_CXXFLAGS: the effective CGO_CXXFLAGS environment variable
- CGO_LDFLAGS: the effective CGO_LDFLAGS environment variable
- GOARCH: the architecture target
- GOAMD64/GOARM/GO386/etc: the architecture feature level for GOARCH
- GOOS: the operating system target
- vcs: the version control system for the source tree where the build ran
- vcs.revision: the revision identifier for the current commit or checkout
- vcs.time: the modification time associated with vcs.revision, in RFC3339 format
- vcs.modified: true or false indicating whether the source tree had local modifications

Karpenter应该是使用了[ko-build/ko](https://github.com/ko-build/ko)工具构建镜像，貌似自动会加上如上信息。参考[Discussion: Can ko generate SLSA provenance? #896](https://github.com/ko-build/ko/issues/896)。

如果我们自己构建可执行时，注意，不能指定`*.go`文件，否则`go build`无法填充如上`build vcs`相关信息，以[allenliu88/nanoproxy](https://github.com/allenliu88/nanoproxy)为例，注意区别两者区别，据说是缺陷[runtime/debug: vcs.modified populated in ReadBuildInfo for go install but not go build#51637](https://github.com/golang/go/issues/51637).

```shell
$ CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 go build -o bin/nanoproxy-darwin-arm64 nanoproxy.go

$ go version -m bin/nanoproxy-darwin-arm64

bin/nanoproxy-darwin-arm64: go1.22.5
        path    command-line-arguments
        dep     github.com/allenliu88/nanoproxy (devel)
        dep     github.com/go-logr/logr v1.4.2  h1:6pFjapn8bFcIbiKo3XT4j/BhANplGihG6tvd+8rYgrY=
        dep     github.com/go-logr/zapr v1.3.0  h1:XGdV8XW8zdwFiwOA2Dryh1gj2KRQyOOoNmBy4EplIcQ=
        dep     github.com/samber/lo    v1.44.0 h1:5il56KxRE+GHsm1IR+sZ/6J42NODigFiqCWpSc2dybA=
        dep     go.uber.org/multierr    v1.11.0 h1:blXXJkSxSSfBVBlC76pxqeO+LN3aDfLQo+309xJstO0=
        dep     go.uber.org/zap v1.27.0 h1:aJMhYGrd5QSmlpLMr2MftRKl7t8J8PTZPA732ud/XR8=
        dep     golang.org/x/text       v0.16.0 h1:a94ExnEXNtEwYLGJSIUxnWoxoRz/ZcCsV63ROupILh4=
        dep     knative.dev/pkg v0.0.0-20240704013837-7ecd5485cbc6      h1:/oGRGm/csTc0sUHo00MQ3NQrJaRP7iMTGC9bXpeEuuU=
        build   -buildmode=exe
        build   -compiler=gc
        build   CGO_ENABLED=0
        build   GOARCH=arm64
        build   GOOS=darwin
```

```shell
$ CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 go build -o bin/nanoproxy-darwin-arm64

$ go version -m bin/nanoproxy-darwin-arm64

bin/nanoproxy-darwin-arm64: go1.22.5
        path    github.com/allenliu88/nanoproxy
        mod     github.com/allenliu88/nanoproxy (devel)
        dep     github.com/go-logr/logr v1.4.2  h1:6pFjapn8bFcIbiKo3XT4j/BhANplGihG6tvd+8rYgrY=
        dep     github.com/go-logr/zapr v1.3.0  h1:XGdV8XW8zdwFiwOA2Dryh1gj2KRQyOOoNmBy4EplIcQ=
        dep     github.com/samber/lo    v1.44.0 h1:5il56KxRE+GHsm1IR+sZ/6J42NODigFiqCWpSc2dybA=
        dep     go.uber.org/multierr    v1.11.0 h1:blXXJkSxSSfBVBlC76pxqeO+LN3aDfLQo+309xJstO0=
        dep     go.uber.org/zap v1.27.0 h1:aJMhYGrd5QSmlpLMr2MftRKl7t8J8PTZPA732ud/XR8=
        dep     golang.org/x/text       v0.16.0 h1:a94ExnEXNtEwYLGJSIUxnWoxoRz/ZcCsV63ROupILh4=
        dep     knative.dev/pkg v0.0.0-20240704013837-7ecd5485cbc6      h1:/oGRGm/csTc0sUHo00MQ3NQrJaRP7iMTGC9bXpeEuuU=
        build   -buildmode=exe
        build   -compiler=gc
        build   CGO_ENABLED=0
        build   GOARCH=arm64
        build   GOOS=darwin
        build   vcs=git
        build   vcs.revision=6b6f61a2164834d59110de9b26291817316a03bb
        build   vcs.time=2024-07-04T12:43:18Z
        build   vcs.modified=true

$ chmod +x bin/nanoproxy-darwin-arm64
$ bin/nanoproxy-darwin-arm64
{"level":"INFO","time":"2024-07-04T21:37:31.355+0800","logger":"nanoproxy","message":"Starting to listen and serve...","commit":"6b6f61a-dirty"}
```

## [runtime/debug: vcs.modified populated in ReadBuildInfo for go install but not go build#51637](https://github.com/golang/go/issues/51637)

### What version of Go are you using (`go version`)?

```shell
$ go version
go version go1.18beta2 darwin/amd64
```

### Does this issue reproduce with the latest release?

Yes

### What operating system and processor architecture are you using (`go env`)?

```shell
$ go env
GO111MODULE=""
GOARCH="amd64"
GOBIN=""
GOCACHE="/Users/james/Library/Caches/go-build"
GOENV="/Users/james/Library/Application Support/go/env"
GOEXE=""
GOEXPERIMENT=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GOINSECURE=""
GOMODCACHE="/Users/james/go/pkg/mod"
GOOS="darwin"
GOPATH="/Users/james/go"
GOPROXY="direct"
GOROOT="/usr/local/go"
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/darwin_amd64"
GOVCS=""
GOVERSION="go1.18beta2"
GCCGO="gccgo"
GOAMD64="v1"
AR="ar"
CC="clang"
CXX="clang++"
CGO_ENABLED="1"
GOMOD="/Users/james/projects/debug-module-version-demo/go.mod"
GOWORK=""
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -arch x86_64 -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0 -fdebug-prefix-map=/var/folders/g1/r_29t6g17l926yzy2610sy3m0000gn/T/go-build2749357119=/tmp/go-build -gno-record-gcc-switches -fno-common"
```

### What did you do?

#### Produce a binary with go build and with go install:

1. Fetch the demo repo from [@mark-rushakoff](https://github.com/mark-rushakoff) : `git clone https://github.com/mark-rushakoff/debug-module-version-demo.git`
2. `go install`
3. `go build -o demo.osx *.go`

#### Compare the attached build info:

##### Output from go build

```shell
$ go version -m demo.osx                                     
demo.osx: go1.18beta2
	path	command-line-arguments
	dep	golang.org/x/text	v0.0.0-20170915032832-14c0d48ead0c	h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
	dep	rsc.io/quote	v1.5.2	h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
	dep	rsc.io/sampler	v1.3.0	h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
	build	-compiler=gc
	build	CGO_ENABLED=1
	build	CGO_CFLAGS=
	build	CGO_CPPFLAGS=
	build	CGO_CXXFLAGS=
	build	CGO_LDFLAGS=
	build	GOARCH=amd64
	build	GOOS=darwin
	build	GOAMD64=v1
```

##### Output from go install

```shell
$ go version -m /Users/james/go/bin/debug-module-version-demo
/Users/james/go/bin/debug-module-version-demo: go1.18beta2
	path	github.com/mark-rushakoff/debug-module-version-demo
	mod	github.com/mark-rushakoff/debug-module-version-demo	(devel)	
	dep	golang.org/x/text	v0.0.0-20170915032832-14c0d48ead0c	h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
	dep	rsc.io/quote	v1.5.2	h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
	dep	rsc.io/sampler	v1.3.0	h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
	build	-compiler=gc
	build	CGO_ENABLED=1
	build	CGO_CFLAGS=
	build	CGO_CPPFLAGS=
	build	CGO_CXXFLAGS=
	build	CGO_LDFLAGS=
	build	GOARCH=amd64
	build	GOOS=darwin
	build	GOAMD64=v1
	build	vcs=git
	build	vcs.revision=f3f34e10b35e88a6f6998d8c1938a5998c33cb64
	build	vcs.time=2018-12-13T20:11:42Z
	build	vcs.modified=true
```

### What did you expect to see?

I expected to see vcs.revision/vcs.modified/etc populated in the `runtime/debug.ReadBuildInfo()` of the binaries produced by `go build -o demo.osx *.go` and `go install`.

### What did you see instead?

I saw vcs.revision/vcs.modified/etc populated only in the `runtime/debug.ReadBuildInfo()` of the binary produced by `go install`, but not the binary produced by `go build -o demo.osx *.go`.

### 解答

With additional testing, non-population of vcs.revision/etc in `runtime/debug.ReadBuildInfo()` seems to only affect `go build` when the specific go files (in this case, `*.go`) are specified. When they are not specified, the tags are populated as expected.

E.g:

```shell
# note *.go not specified
$ go build -o demo.osx
$ go version -m demo.osx                        
demo.osx: go1.18beta2
	path	github.com/mark-rushakoff/debug-module-version-demo
	mod	github.com/mark-rushakoff/debug-module-version-demo	(devel)	
	dep	golang.org/x/text	v0.0.0-20170915032832-14c0d48ead0c	h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
	dep	rsc.io/quote	v1.5.2	h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
	dep	rsc.io/sampler	v1.3.0	h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
	build	-compiler=gc
	build	CGO_ENABLED=1
	build	CGO_CFLAGS=
	build	CGO_CPPFLAGS=
	build	CGO_CXXFLAGS=
	build	CGO_LDFLAGS=
	build	GOARCH=amd64
	build	GOOS=darwin
	build	GOAMD64=v1
	build	vcs=git
	build	vcs.revision=f3f34e10b35e88a6f6998d8c1938a5998c33cb64
	build	vcs.time=2018-12-13T20:11:42Z
	build	vcs.modified=true
```

So the bug I am reporting may be intended behavior when files are specified, whose description I have just missed in the documentation.
