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
