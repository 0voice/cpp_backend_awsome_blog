# 【NO.370】SRS配置升级，云原生友好的配置能力

什么才是更好的配置方法？NGINX的conf，还是MySQL的ini，还是新潮的yaml，或者JS友好的json？它们都有各自的问题，最好的方式是conf+环境变量，也就是Grafana的配置方法。

## 1.Why Important?

为何配置这么重要？因为它是最基本的API，也就是程序和人的接口，也决定了使用体验是否良好。想象下xml的配置文件，想起来都觉得烦躁，这是因为xml并不是对人友好的接口。

SRS的配置一直都对人挺友好的，因为是使用的NGINX的配置方法，熟悉NGINX的同学可能会觉得很熟悉，比如：

```
listen 1935;
max_connections 1000;
vhost __defaultVhost__ {
}
```

相当直观和好理解，这些年也就这么过来了，但它并非没有问题，比如：

- • 程序读写不友好，要用代码修改配置，就需要自己实现这个解析。NGINX是有生态支撑，有工具支持读写，但它只支持NGINX的配置，并不是所有conf格式都能支持。
- • 在文档或Wiki中，或者在给出例子时，总是要给出一个配置文件，而一般还需要修改现有的配置文件，很不方便，也有可能会出错。
- • 在K8s部署时，或者Docker启动时，需要创建文件，并映射到Docker中，哪怕只需要修改某个配置项，也需要这么做，这套机制很麻烦。

也正是因为有这些问题，陆陆续续的有同学提出来支持其他的配置方式，详细可以参考#2277[1]中的详细讨论，大概有几种解决方案：

- • 支持JSON或YAML，这是为了解决程序读写的问题，JSON对JS程序员友好，YAML对DevOps友好。无论哪种方式，都不能对所有人友好，而且还解决不了后面两个问题，还是依赖文件。
- • 支持Redis那种命令行参数的方式，这解决了配置文件的问题，对人也相对比较友好，但若有参数变更，则还是需要依赖文件。
- • 环境变量，方便设置和命令行启动，是基本的传递配置的办法，但多了后不太友好，另外和目前的文件配置方法有差异，导致Reload等机制需要修改。

> Note: 如果直接换成新的配置方式，都会对目前支持的NGINX的conf文件的方式造成不兼容，影响使用习惯。因此最好的办法不是替代，而是结合现有配置方法，实现配置能力的增强。

直到看到了Grafana的配置方式，发现了在目前基础上，可以更好的配置方法。

## 2.Solution

目前NGINX的conf的配置方法，是对人比较友好的，需要支持的是对程序接口友好的方式，而且是和这种方式是可以配合而不是替代的方案。这就是环境变量的方式，先看Grafana的启动方式：

```
docker run --rm -it --name grafana \
    --env GF_SECURITY_ADMIN_USER=admin \
    --env GF_SECURITY_ADMIN_PASSWORD=admin \
    --env GF_DATABASE_TYPE=mysql \
    --env GF_DATABASE_HOST=host.docker.internal:3306 \
    --env GF_DATABASE_NAME=grafana \
    --env GF_DATABASE_USER=root \
    --env GF_DATABASE_PASSWORD=****** \
    --env GF_INSTALL_PLUGINS=tencentcloud-monitor-app \
    -p 3000:3000 \
    grafana/grafana
```

这种方式是最方便用Docker启动的方式，在K8s中也是一样的。Grafana所有的配置既可以通过配置文件配置，也可以通过环境变量配置。

借鉴这种办法，SRS的配置也可以支持环境变量，比如WebRTC over TCP[2]：

```
docker run --rm -it -p 8080:8080 -p 1985:1985 -p 8000:8000 \
  -e CANDIDATE="192.168.3.82" \
  -e SRS_RTC_SERVER_TCP_ENABLED=on \
  -e SRS_RTC_SERVER_PROTOCOL=tcp \
  registry.cn-hangzhou.aliyuncs.com/ossrs/srs:v5.0.60
```

无论是命令行启动，还是Docker启动，还是分享命令给其他人，还是K8s启动，这都是最简单直接的办法。

> Note: 一般我们并不会需要修改特别多的配置，往往只需要修改几个配置，因此这种方式是最便捷的。

当然也不是所有SRS的配置都支持环境变量，因为有些配置比如Transcode是数组，就很难支持，具体以full.conf[3]中标记为`Overwrite by env SRS_XXX`为准，比如：

```
# Overwrite by env SRS_LISTEN
listen 1935;
# Overwrite by env SRS_MAX_CONNECTIONS
max_connections 1000;

rtc_server {
    # Overwrite by env SRS_RTC_SERVER_ENABLED
    enabled on;
}

vhost __defaultVhost__ {
    rtc {
        # Overwrite by env SRS_VHOST_RTC_ENABLED for all vhosts.
        enabled on;
    }
}
```

基本上SRS的配置都支持了环境变量的覆盖，非常感谢mapengfei53[4]的贡献。

## 3.Next

目前SRS支持了环境变量的配置，还需要改进支持Reload。

由于Reload依赖配置文件，在收到Reload信号后，重新加载配置文件，对比发现变更后，实现定向的快速Reload。而环境变量的配置，则需要实现对应的变更检测机制，我们会在后续改进和完善。

此外，之前Reload的机制过度设计，有些其实没有必要支持Reload，比如侦听的端口，是不会在运行中变化，而且变化会导致很多异常问题。因此，会在未来把很多不常用的Reload功能去掉，只支持必要的Reload。

还有，K8s中的配置是通过ConfigMap加载到容器，而通过inotify机制，可以在文件内容修改后，主动加载配置Reload，而不需要发送Reload信号，这样在K8s集群中只需要修改ConfigMap就会自动生效到容器。这个机制同样也需要支持环境变量，如何在环境变量变更后，在K8s集群中生效。

最后补充一句：SRS会永远支持配置文件，以及对应的配置变更Reload机制，环境变量的配置方法，是对目前配置能力的完善，并不是替代。

**引用链接**

`[1]` #2277: *https://github.com/ossrs/srs/issues/2277*
`[2]` WebRTC over TCP: *https://ossrs.net/lts/zh-cn/docs/v5/doc/webrtc#webrtc-over-tcp*
`[3]` full.conf: *https://github.com/ossrs/srs/blob/develop/trunk/conf/full.conf*
`[4]` mapengfei53: *https://github.com/mapengfei53*

原文作者：Written by 马鹏飞, Winlin

原文链接：https://mp.weixin.qq.com/s/ABMrvGEer1FYrdR9PrearA