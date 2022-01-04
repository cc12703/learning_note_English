
# 我在一台服务器上的最初5分钟--Linux服务器的必要安全配置

* [文档链接](https://sollove.com/2013/03/03/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers/)



## 概述  

&emsp;&emsp;服务器安全不需要很复杂。我的安全原则很简单：采取哪些可以防护最频繁攻击的措施，在保持管理足够高效的同时不去开发“安全杂项”。如果你可以明智的使用服务器上最初的5分钟，我相信你可以做到这些。   

&emsp;&emsp;任何老道的系统管理员都会告诉你，随着加入更多的服务器和开发者，对这些的管理会不可避免地成为负担。在一个快速增长的环境中，维持便捷地准许接入是一场高地作战，你最终都会进入陈旧的密码，被抛弃的内部账号，一大堆类似于“我在服务器A上可以用sudo，但是在服务器B上不行”的问题。虽然通过使用账号同步工具可以缓解这些问题，但是在我看来，这些增加的收益不值得花时间(the incremental benefit isn’t worth the time nor the security downsides)。好的安全性的核心就是简洁性。  

&emsp;&emsp;我们的服务器都会配置两种账号：根用户、部署用户。部署用户使用随机的长密码可以进行sudo操作，是开发者可以登录账号。开发者使用自己的公钥而不是密码来登录服务器，这样的话，管理就会简单到只要保证**authorized_keys**文件在各个服务器之间同步更新即可。根用户禁止通过ssh登录，部署用户只能从我们的办公IP区登录  

&emsp;&emsp;这个方案的缺点就是如果**authorized_keys**文件被破坏或丢失权限信息了，就必须要登录远程终端来修复它。但是如果你适当的谨慎就不需要进行这个操作了。  


## 开始行动

&emsp;&emsp;我们的盒子是刚孵化的，提示处的是原始像素(Our box is freshly hatched, virgin pixels at the prompt)。我喜欢Ubuntu，如果你使用其他版本的Linux，你的命令行会不一样。

```
passwd
```
将根密码修改成长的、复杂的。你不需要记住它，只需要保存在一个安全的地方。只有当你无法通过ssh登录系统或者丢失掉sudo密码的时候，才会去使用这个密码

```
apt-get update
apt-get upgrade
```
上面的操作将会是一个很好的起点


### 安装Fail2ban

```
apt-get install fail2ban
```

Fail2ban是一个守护进程，监控着所有登录操作，会阻止可疑的行为。需要在命令行外配置好  

现在可以设置登录用户了。用户名可以随意命名，下面的示例是为了方便起见
```
useradd deploy
mkdir /home/deploy
mkdir /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
```

### 要求公钥鉴权
使用密码登录的日子依旧结束了。通过使用公钥鉴权来代替密码，可以一下子加强安全性和易用性

```
vim /home/deploy/.ssh/authorized_keys
```
将你本地机器中id_rsa.pub文件的内容和所有想访问该机器的公钥都加入这个文件中

```
chmod 400 /home/deploy/.ssh/authorized_keys
chown deploy:deploy /home/deploy -R
```

### 测试新账号 & 启用sudo
现在使用deploy用户登录新服务器来测试一下新账号（保持根用户登录的终端打开）。如果成功了，你可以切换到终端并给该账号设置一个sudo秘密

```
passwd deploy
```
设置一个复杂的密码 -- 你可以将密码保持在安全的地方、或者让密码对于团队是难忘的

```
visudo
```

注释点所有存在的 用户/组 授权行，加入以下内容
```
root    ALL=(ALL) ALL
deploy  ALL=(ALL) ALL
```
上面的授权行会在用户输入恰当密码的时候，赋予sudo访问权限


### 锁定SSH
将ssh配置成无法用密码登录、无法根用户登录、只允许特定IP访问

```
vim /etc/ssh/sshd_config
```

向文件中插入以下内容
```
PermitRootLogin no
PasswordAuthentication no
AllowUsers deploy@(your-ip) deploy@(another-ip-if-any)
```

现在重启ssh
```
service ssh restart
```


### 设置防火墙
无防火墙的服务器是防护不完整的。Ubuntu提供了ufw来管理防火墙
```
ufw allow from {your-ip} to any port 22
ufw allow 80
ufw allow 443
ufw enable
```

以上设置了一个最基本的防火墙，配置了服务器只能访问80端口和443端口，你可以根据你要运行的服务来加入更多的端口号


### 开启自动安全更新
最近几年我养成了运行`apt-get update/upgrade`的习惯，但是面对一打的服务器，我发现哪些登录频率少的服务器很难保证程序是最新的。特别是负载均衡服务器，将这些服务器都保持最新是很重要的。自动安全更新或许有时候会出现一些问题，但是不会比不打安全补丁更糟糕

```
apt-get install unattended-upgrades

vim /etc/apt/apt.conf.d/10periodic
```

在文件中加入以下内容
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

还需要配置以下文件
```
vim /etc/apt/apt.conf.d/50unattended-upgrades
```

按以下内容修改文件，这样就可以关闭正常的更新，只进行安全更新
```
Unattended-Upgrade::Allowed-Origins {
        "Ubuntu lucid-security";
//      "Ubuntu lucid-updates";
};
```


### 安装Logwatch
Logwatch是一个守护进程，用于监控你的日志，并将其通过email发送给你。这对于跟踪和检测入侵行为非常有用。如果某人想访问你的服务器，发送给你的日志对于确定何时发生及行为就会非常有帮助 -- 因为服务器上的日志可能会被盗用

```
apt-get install logwatch

vim /etc/cron.daily/00logwatch
```

在文件中加入以下内容
```
/usr/sbin/logwatch --output mail --mailto test@gmail.com --detail high
```

## 完成
我想现在我们已经有了一个坚实的基础。在刚才的几分钟内，我们锁定了服务器，设置了一个可以击退大部分攻击的安全等级。在一天结束的时候，我们会发现导致入侵的几乎都是用户错误
