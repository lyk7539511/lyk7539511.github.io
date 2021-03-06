---
title: "Linux 禁用内核更新"
categories:
  - Blog
  - Tech
tags:
  - Linux
  - Fedora
  - Kernel

---

[![Screenshot-from-2019-01-21-22-27-11.png](https://i.postimg.cc/T322LNPX/Screenshot-from-2019-01-21-22-27-11.png)](https://postimg.cc/rKb27JWh)

Fedora,一个RHEL的激进发行版，永远追求最新的技术，以极高的频率推送软件更新，甚至包含内核，可以说是一个激进分子。但是依托于Redhat公司强大的技术力量，即使更新速度非常快，仍然保持了一定的稳定性，至少在我看来要比Ubuntu的LTS版本更加稳定，所以我现在正在慢慢从Ubuntu转向Fedora。当然，转换的过程中一定会遇到问题，今天要说的就是其中一个。

Fedora的更新实在是太快了，几天就会推送一个新版本的内核，虽然一般情况下不会出现什么大的问题，但是这毕竟是内核，更新太快没有安全感，所以我就在想能否可以保持其他软件的更新而只忽略内核的更新呢，答案是可以的，我也是找了好久才找到解决方案。

首先：

```bash
cd /etc/dnf
sudo vim dnf.conf
```

打开dnf的配置文件，一定要用超级用户权限，因为该文件的所有者是root。
然后在里面添加下面一行

```bash
exclude=kernel*
```

这句话就是告诉dnf在执行任何命令(包含update\check-update\remove等)的时候忽略以kernel开头的包。这样以后在使用check-update命令的时候就会自动跳过kernel，只更新其他包。要是还想禁止其它的软件更新，可以使用同样的方法。
如果想要更新一下内核，就在exclude前边加上一个#，注释掉就好了，更新完以后可以再把#删掉。
