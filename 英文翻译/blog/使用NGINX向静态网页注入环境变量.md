

# 使用NGINX向静态网页注入环境变量

## 概述
静态网站生成非常适合于发布文档。在最近的项目汇中，我们使用了NGINX作为web服务器来托管HTML，CSS文件。但是我们想使用SSO来保护我们的网站。这个操作就有点困难了

自然地，当我们使用SSO替换HTTP基本鉴权时，我们就需要显示一个**登出**按键。但是当和静态网站生成(SSG)一起使用时，我就无法知道**登出**的URI地址了，该地址取决于部署系统时的配置。


## 方法
为了能够真正地渲染出按键，我们可以下面的两种选择
1. 在docker容器启动时，运行SSG工具
1. 不使用URI地址来渲染按键，而是在运行时由web服务器注入

我们的项目中，SSG使用的是Jekyll，所以需要加入整个Rudy工具链会破坏使用NGINX的初衷：尽可能的轻量化。此外，巨大的镜像也会导致更多的攻击面，所以无法采用第一个选项

但是第二选项如何实现呢？幸运的是服务端包含(SSI)技术可以帮助到我们。简言之，生成的页面中包含了几乎所有的内容，给web服务器增加一些简单的指令来向页面插入更多的内容。SSI可以用来从其他服务器中获取内容，根据条件在开启、关闭页面中的特定区域，或则打印出某处变量的内容

### 具体做法
NGINX支持SSI，需要进行如下配置
```nginx
location / {
	root /var/www;
	ssi on;
}
```

在Jekyll模板中，需要加入如下片段
```html
<a href="<!--#echo var="ssisignouturl"-->">Sign out</a>
```

当使用`jeklyy serve`在本地浏览页面时，上面的模板会产生无限的HTML标记。所以我推荐在生产模式下使用该片段
```html
{% if jekyll.environment == "production" %}
  <a href="<!--#echo var="ssisignouturl"-->">Sign out</a>
{% endif %}
```

现在唯一的问题就是NGINX无法传递环境变量到SSI引擎的进程中。更糟糕的是，NGINX并不支持在配置文件中直接读取环境变量。

根据官方的NGINX容器镜像，需要使用模板方式才能将变量传入运行的容器中。镜像中包含一段启动脚本，会对标记为模板的配置文件运行`envsubst`命令。当启动该容器时，模板会被处理，环境变量的值会被拷贝进实际的NGINX配置中

为了使用这个机制，需要将NGINX配置中相关的部分移入一个名为`default.conf.template`的模板文件中（可以使用其他名字，只要后缀是`.conf.template`）。

`Dockerfile`需要采样如下格式
```dockerfile
FROM nginx:1.19.9

COPY default.conf.template /etc/nginx/templates/

ENV SIGN_OUT_URL="#"
```
这个是必须的，只有这样启动脚本才会获取该文件，并传给`envsubst`命令
现在NGINX配置文件中，需要使用如下方式获取变量
```nginx
location / {
  root /var/www;
  ssi on;
  set $ssisignouturl "${SIGN_OUT_URL}";
}
```

当容器启动后，脚本会生成如下的配置内容
```nginx
location / {
  root /var/www;
  ssi on;
  set $ssisignouturl "#";
}
```

NGINX模板中有一个重要的点，SSI变量名(`ssisignouturl`)不同于环境变量名(SIGN_OUT_URL)，否则`envsubst`会将所有地方的环境变量都替换一遍。


最后，在以上变动后，SSI引擎就可以替换到URL地址。按以下方式启动镜像
```dockerfile
docker build -t docs .
docker run --rm -it -p 80:80 -e SIGN_OUT_URL="https://sso.bigcorp.com/signout" docs
```

会生成如下的HTML
```html
<a href="https://sso.bigcorp.com/signout">Sign out</a>
```
这里出现的问题是NGINX并没有内建传递环境变量的机制，我们不得不通过Docker镜像提供的启动脚本来实现。在这种情况下无法使用官方的Docker镜像，我推荐将**登出**链接生成为一个文本文件，使用`include`指令将该文件包含进来。