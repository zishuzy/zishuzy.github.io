---
title: docker代理
date: 2022-10-02
tags: 
- docker
- 代理
---

## 1. dockerd 代理  

在执行`docker pull`或者`docker push`等操作时，其实是由守护进程`dockerd`来执行，
而`dockerd`是由`systemd`管控的，所以我们需要针对`systemd`进行配置

``` bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo touch /etc/systemd/system/docker.service.d/proxy.conf
```

在这个 `proxy.conf` 文件（可以是任意 `*.conf` 的形式）中，添加以下内容：

``` conf
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:8080/"
Environment="HTTPS_PROXY=http://proxy.example.com:8080/"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
```

其中的`proxy.example.com`换成自己的代理地址即可。

## 2. Container 代理

在容器运行时，如果想要在容器内部使用代理，则需要配置`~/.docker/config.json`，内容如下：

``` json
{
  "proxies":
  {
    "default":
    {
      "httpProxy": "http://proxy.example.com:8080",
      "httpsProxy": "http://proxy.example.com:8080",
      "noProxy": "localhost,127.0.0.1,.example.com"
    }
  }
}
```

此外，容器的网络代理，也可以直接在其运行时通过 `-e` 注入 `http_proxy` 等环境变量。

## 3. docker build 代理

虽然 `docker build` 的本质，也是启动一个容器，但是机制略有不同，配置文件无效。只能在构建时，注入 `http_proxy` 等参数。

``` bash
docker build . \
    --build-arg "HTTP_PROXY=http://proxy.example.com:8080/" \
    --build-arg "HTTPS_PROXY=http://proxy.example.com:8080/" \
    --build-arg "NO_PROXY=localhost,127.0.0.1,.example.com" \
    -t your/image:tag
```

重启`systemd`服务

``` bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```
