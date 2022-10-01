## 背景
由于之前使用的是 ArchLinux 系统进行开发，而系统是运行于 Alienware 上的，笔记本比较沉重，出行携带不方便。   
因此需要考虑在便携笔记本上进行开发。  

## 方案选择
1. 购置一台专门的笔记本用于 Linux 开发，考虑到成本问题，犹豫了一段时间后，果断放弃。    
2. 购买云主机，再使用远程桌面连接，在云主机上进行开发，在初期使用此方案，确实可行，    
但是后续发现在旅途中，受限于网络环境，体验很差，另一方面由于成本原因，购置的云主机配置不高。   
3. 使用虚拟机, 方案比较消耗资源 
4. 由于我使用的是 MacBook, 考虑如何在 MacBook 上搭建 ArchLinux 开发环境， 以记录此方案为主。 

## 使用到的工具 
MacBook  Docker-Desktop  Vscode  docker-mac-net-connect  Clash

## 踩坑过程    
1. Container 无法和 Host宿主主机 通信    
之所以需要通信, 是由于希望 Container 能够使用 宿主主机 的网络代理访问一些资源,   
首先想起 Container 和 Host 之间存在网络隔离, 进而联想到需要在创建 Container 时, 指定 --network=host.   
但是配置完成后, 发现 即使 Container 内配置了 http_proxy 等, 依然无法使用宿主主机代理, 尝试几次后, 想起来之前看过 Docker Documentation    
提到 Mac 和 Windows 无法使用 host 网络, 赶紧翻看 Docker 文档, 应证了自己的猜想.   
``` doc 
# https://docs.docker.com/network/host  
The host networking driver only works on Linux hosts, and is not supported on Docker Desktop for Mac,     
Docker Desktop for Windows, or Docker EE for Windows Server. 
``` 
只能另辟蹊径, 在 stack overflow 看到一个项目, 发现可以实现自己的需求  docker-mac-net-connect, 具体原理就不再赘述了.   
直接上链接 [docker-mac-net-connect](https://github.com/chipmk/docker-mac-net-connect)      

2. 无法使用 yay 安装某些应用   
安装会出现 
``` shell 
> System has not been booted with systemd as init system (PID 1). Can't operate. 
```
很显然, Container 的一号进程 是 /bin/bash, 这是由于之前在创建 Container 时, 指定了 docker run --it xxxx /bin/bash 时指定的,    
既然知道原因就很好办了, 直接指定 1 号进程为 /sbin/init.  

3. 使用 yay 安装报 Failed to connect to bus: No such file or directory. 
解决完以上问题, 发现 yay 还是没办法使用, 但是根据报错, 立马想到可能是当前用户为 root 的关系, 从而导致 bus 没有启动,  
立马着手测试, 创建用户, su到用户, 再执行 yay -S cpr, 果然成功了.   

## 配置步骤   
前面说了这么多, 开始正式配置吧      
1. 安装 Docker-Desktop Vscode, 推荐使用 brew 安装    
2. 按照 docker-mac-net-connect 项目步骤安装    
``` shell 
> brew install chipmk/tap/docker-mac-net-connect    
> sudo brew services start chipmk/tap/docker-mac-net-connect
```
3. 从 Hub 上 pull ArchLinux 
``` shell
> docker pull archlinux
```
4. 创建 Container, 记得配置1号进程为 /sbin/init, 同时要记得做好目录映射, 比如我指定了两个目录  
``` 
> docker run -itd --privileged --name ArchDevEnv -v ~/Desktop/OpenSource:/workspace/OpenSource -v ~/Desktop/Projects:/workspace/Projects archlinux:latest /sbin/init
```
5. 修改宿主主机的 Clash 配置, 或者可以直接在 Clash-for-windows 上设置. 
``` conf
allow-lan: true
bind-address: '*'
```

6. 设置代理环境变量为宿主主机的 ip:port, 使用 curl -v 测试是否正常  
``` shell
> export https_proxy=http://192.168.1.11:7890 http_proxy=http://192.168.1.11:7890
> curl -v https://www.google.com/
* Uses proxy env variable https_proxy == 'http://192.168.1.8:7890'
*   Trying 192.168.1.8:7890...
* Connected to 192.168.1.8 (192.168.1.8) port 7890 (#0)
```

7. 使用 docker exec 或者 之前通过 Docker-Desktop 进入 CLI, 安装和配置好相应的源  
我使用的是 reflector 更新为 cn 源  
6. 安装 sudo, base-devel. 
``` shell 
> yay -S sudo base-devel 
```
8. 添加和配置 sudo 组权限  
``` shell 
# 添加 sudo 组
> groupadd sudo 
# 修改 /etc/sudoers 
将 "# %sudo   ALL=(ALL:ALL) ALL" 前面的 “#” 取消 
```
9. 创建个人用户, 属组为 sudo, 设置密码    
10. 测试一下 yay 是否正常使用  
``` shell
> yay -S cpr 
```
11. vscode 安装 “Dev Containers” 和 "Docker" 插件, 在 Docker 插件栏可以看到 自己的 Container,    
右键选择 "attach Visual Studio Code", 再选择打开项目, 发现当前是从 ArchLinux 上打开的项目

12. 配置好 oh-my-zsh 和 powerlevel10k, 发现在 vscode 终端, 发现这两个项目也能在 root 用户下正常使用.  
``` shell 
  /workspace/P/DeathNode on   master *2 !2 ?13
❯ 
```
