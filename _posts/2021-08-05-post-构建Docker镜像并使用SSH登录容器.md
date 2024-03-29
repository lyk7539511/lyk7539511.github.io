---
title: "构建一个可以使用SSH登录的Docker镜像"
categories:
  - Blog
  - Tech
tags:
  - Docker
  - SSH

---

## 环境
  - OS: PopOS 20.04
  - Mem: 8G
  - CPU: i7-6820HQ

## 步骤

### 1、安装Docker
请参考 [Docker官网教程](https://docs.docker.com/engine/install/ubuntu/) ，请不要到处乱找教程，官方的就是最好的，我一路安装下来没有任何问题

### 2、编写Dockerfile
请确保当前用户在docker用户组中，这样可以非特权运行docker

这里顺便安装Anaconda

首先创建一个名为 Dockerfile 的文件，在其中填写以下内容

```bash
FROM ubuntu:20.04

RUN apt update && apt install -y \
curl \
wget \
openssh-server \
vim && \
rm -rf /var/lib/apt/lists/* && \
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config && \
service ssh start

ADD Anaconda3-2021.05-Linux-x86_64.sh /home/anaconda.sh
RUN /bin/bash /home/anaconda.sh -b -p /opt/conda && \
ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh

RUN mkdir /workspace
WORKDIR /workspace

CMD ["/bin/bash"]
```

### 3、下载Anaconda，与Dockerfile同级目录

### 4、构建Docker镜像
```bash
docker build -f Dockerfile -t test:v1.0 .
```
test是镜像的名字，可以自定义，后面跟版本号

可能会有些慢，请耐心等待，构建完成如下图：

[![feRBfP.png](https://z3.ax1x.com/2021/08/05/feRBfP.png)](https://imgtu.com/i/feRBfP)

### 5、运行镜像

[![feW69x.png](https://z3.ax1x.com/2021/08/05/feW69x.png)](https://imgtu.com/i/feW69x)

[![fefUPI.png](https://z3.ax1x.com/2021/08/05/fefUPI.png)](https://imgtu.com/i/fefUPI)

进入镜像之后先使用 passwd 命令修改root密码，方便后期ssh连接

### 6、运行ssh

找一个ssh终端，输入server的IP地址，端口号是映射出来的8888，用户名是root，密码是你上面设置的密码，此时应该可以正常连接了。

如果无法连接，请确认ssh的服务已经开启，没有开启的话请自行start

[![fehVQf.png](https://z3.ax1x.com/2021/08/05/fehVQf.png)](https://imgtu.com/i/fehVQf)

## 总结
使用docker很方便，当有用户想要使用server资源，可以开放容器，甚至开放ssh以及root权限，而不会影响宿主机的安全，做到了资源共享数据隔离。


------------------
2021/08/14更新
上面的脚本有些问题，下面是最新的

整个构建的时间比较长，请耐心等待

sudo bash build.sh
```bash
#!/bin/bash

mkdir Dockerfile
cd Dockerfile

touch run.sh Dockerfile Conda.sh
echo "#!/bin/bash
/usr/sbin/sshd -D
" >> run.sh

echo "#!/bin/bash
wget https://repo.anaconda.com/archive/Anaconda3-2021.05-Linux-x86_64.sh

bash Anaconda3-2021.05-Linux-x86_64.sh -b -p /opt/conda

ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh

" >> Conda.sh

echo "FROM ubuntu:20.04

MAINTAINER \"YuanKun Liu [D20091100124@cityu.mo]\"

RUN echo \"root:xqrlyk133\" | chpasswd

RUN apt update && apt install -y \
openssh-server \
vim \
wget

RUN rm -rf /var/lib/apt/lists/*

RUN echo \"PermitRootLogin yes\" >> /etc/ssh/sshd_config

RUN mkdir /var/run/sshd
RUN mkdir /root/.ssh
ADD run.sh /run.sh
RUN chmod 755 /run.sh
#RUN /etc/init.d/ssh restart
#RUN service ssh restart
EXPOSE 22
#RUN echo \"service ssh restart\" >> /root/.bashrc

RUN mkdir /workspace
WORKDIR /workspace

ADD Conda.sh /workspace/Conda.sh
RUN chmod 755 /workspace/Conda.sh
RUN bash /workspace/Conda.sh

CMD [\"/run.sh\"]
" >> Dockerfile

sudo docker build -f Dockerfile -t testimg:v1.0 .
sudo docker run -p 8890:22 --privileged=true -itd --name testimg1 testimg:v1.0
```

sudo bash clear.sh
```bash
#!/bin/bash

sudo docker stop testimg1
sudo docker rm testimg1
sudo docker rmi testimg:v1.0
sudo rm -rf Dockerfile

```

---------------
2021/08/21更新

本次更新将Anaconda更换为Miniconda，原因是使用Anaconda构建时间过长并且构建完成后的镜像过于庞大，达到了惊人的7G多，之前没有注意这个问题，更换为Miniconda后，镜像大小只有700MB左右，远小于使用Anaconda构建的镜像，并且由于Anaconda的软件包默认全部安装在base环境中，我们一般不推荐使用base环境而是另外重新创建新环境使用，所以由Anaconda切换到Miniconda不会造成任何影响，只是需要自行判断哪些软件包需要安装并手动安装；无需担心依赖问题，conda以及pip可以很轻松的解决。

默认开放8888端口，以保证后期jupyter的安装需求，对应docker对外暴露的端口为10188，可以自行在脚本中修改；或者开放其他端口，这里不再赘述。如果安装jupyter，请注意修改jupyter的配置文件以及配置密码，否则无法远程访问jupyter！！！

```bash
#!/bin/bash

# make a new DIR to save the Dockerfile and other script files
mkdir Dockerfile
cd Dockerfile

# create new script files
touch run.sh Dockerfile_mini Conda.sh motd

# open sshd server
echo "#!/bin/bash

/usr/sbin/sshd -D
" >> run.sh

# install conda pytorch jupyter
echo "#!/bin/bash

wget https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-py39_4.10.3-Linux-x86_64.sh
bash Miniconda3-py39_4.10.3-Linux-x86_64.sh -b -p $HOME/miniconda
~/miniconda/bin/conda init $(echo $SHELL | awk -F '/' '{print $NF}')
source /etc/profile
# Pytorch & Jupyterlab
#conda activate base
#conda create -y -n pytorch_cpu python=3.8
#conda activate pytorch_cpu
#conda install -y pytorch torchvision torchaudio cpuonly -c pytorch-lts
#conda install -y -c conda-forge jupyterlab

" >> Conda.sh

# set ssh banner
echo "
/**
 * ┌───┐   ┌───┬───┬───┬───┐ ┌───┬───┬───┬───┐ ┌───┬───┬───┬───┐ ┌───┬───┬───┐
 * │Esc│   │ F1│ F2│ F3│ F4│ │ F5│ F6│ F7│ F8│ │ F9│F10│F11│F12│ │P/S│S L│P/B│  ┌┐    ┌┐    ┌┐
 * └───┘   └───┴───┴───┴───┘ └───┴───┴───┴───┘ └───┴───┴───┴───┘ └───┴───┴───┘  └┘    └┘    └┘
 * ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───────┐ ┌───┬───┬───┐ ┌───┬───┬───┬───┐
 * │~ \`│! 1│@ 2│# 3│$ 4│% 5│^ 6│& 7│* 8│( 9│) 0│_ -│+ =│ BacSp │ │Ins│Hom│PUp│ │N L│ / │ * │ - │
 * ├───┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─────┤ ├───┼───┼───┤ ├───┼───┼───┼───┤
 * │ Tab │ Q │ W │ E │ R │ T │ Y │ U │ I │ O │ P │{ [│} ]│ | \\ │ │Del│End│PDn│ │ 7 │ 8 │ 9 │   │
 * ├─────┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴─────┤ └───┴───┴───┘ ├───┼───┼───┤ + │
 * │ Caps │ A │ S │ D │ F │ G │ H │ J │ K │ L │: ;│\" \'│ Enter  │               │ 4 │ 5 │ 6 │   │
 * ├──────┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴────────┤     ┌───┐     ├───┼───┼───┼───┤
 * │ Shift  │ Z │ X │ C │ V │ B │ N │ M │< ,│> .│? /│  Shift   │     │ ↑ │     │ 1 │ 2 │ 3 │   │
 * ├─────┬──┴─┬─┴──┬┴───┴───┴───┴───┴───┴──┬┴───┼───┴┬────┬────┤ ┌───┼───┼───┐ ├───┴───┼───┤ E││
 * │ Ctrl│ Win│ Alt│         Space         │ Alt│ Win│ Fn │Ctrl│ │ ← │ ↓ │ → │ │   0   │ . │←─┘│
 * └─────┴────┴────┴───────────────────────┴────┴────┴────┴────┘ └───┴───┴───┘ └───────┴───┴───┘
 */
 --------------------------------------------------------------
 Welcome!
 This is a docker container build by a postgraduate student from City University of Macau.

" >> motd

# build a docker image using Dockerfile
# based on Ubuntu 20.04 LTS
# install some basic software such as openssh-server,vim and wget
# del cache about apt
# run sshd server
# change DIR to save script file on docker
# install conda
# ssh
echo "FROM ubuntu:20.04

MAINTAINER \"YuanKun Liu [D20091100124@cityu.mo]\"

RUN echo \"root:root\" | chpasswd

RUN apt update && apt install -y \
openssh-server \
vim \
wget

RUN rm -rf /var/lib/apt/lists/*

RUN echo \"PermitRootLogin yes\" >> /etc/ssh/sshd_config

RUN mkdir /var/run/sshd
RUN mkdir /root/.ssh
ADD run.sh /run.sh
RUN chmod 755 /run.sh
#RUN /etc/init.d/ssh restart
#RUN service ssh restart
EXPOSE 22
#EXPOSE 8888
#RUN echo \"service ssh restart\" >> /root/.bashrc

RUN mkdir /workspace
WORKDIR /workspace

ADD Conda.sh /workspace/Conda.sh
RUN chmod 755 /workspace/Conda.sh
RUN bash /workspace/Conda.sh

ADD motd /workspace/motd
RUN cp /workspace/motd /etc/motd

CMD [\"/run.sh\"]
" >> Dockerfile_mini

# build docker image
sudo docker build -f Dockerfile_mini -t miniconda:v1.0 .

# run docker container, open ports
sudo docker run -p 10122:22 -p 10188:8888 --privileged=true -itd --name miniconda_10122 miniconda:v1.0
```