[TOC]

------

# 安装Ubuntu Server 19.04

## 基本情况

我用的是Supermicro M11SDV-8C+-LN4这块板子，据代理商表示，我这是大陆地区第一块M11SDV系列主板。这个系列超微也是刚出，刚在产品页面出现我就预订了，结果并没有现货，过了两个月才从美国寄到台湾再寄到大陆，真实坎坷。
板子有IPMI，可以在IPMI里面安装，IPMI的使用挺简单的，只要把IPMI网口和操作用的PC放在同一个子网获取IP之后连上就好了。网上有很多其他型号的超微主板的IPMI使用教程，我看了一下，连界面都基本一样，我就不记录了。我在路由器的DHCP配置里面将IPMI网口的MAC绑定了固定的IP，不会变化，这样方便以后连接。

制作一个Ubuntu Server 19.04的启动U盘，教程遍地都是，可以随便谷歌一下，我这里是使用[rufus](https://rufus.ie/)制作。Ubuntu Server的安装也没啥好说的，到处是教程，熟练的Linux用户应该无需教程就能搞定。这块主板有四个网口（除去IPMI专用的之外），分别为eno1~eno4。编号居然不是从0开始，有点不习惯。连接上级路由的网口是eno1，安装过程中的`Primary connection`选择eno1。
语言保持英文以免控制台乱码，地区选中国，这样会从中国镜像下载软件包，比较快。分区看着办，我这里是使用一块台式机淘汰的古代的60G Intel SSD整个作为Ubuntu Server安装盘。
安装过程提示选择装哪些服务的时候，至少把SSH服务选上，以便后面连接操作，Samba也可以选上。其他的暂时没必要。

查一下eno1分配到的IP地址，后面的操作就可以用ssh连接上操作了。

## 软件源

首先，改一下软件源
- 使用阿里云镜像
```bash
sudo sed -i 's/cn.archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
sudo sed -i 's/security.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
sudo sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
sudo apt update
```
- 使用腾讯云镜像
```bash
sudo sed -i 's/cn.archive.ubuntu.com/mirrors.cloud.tencent.com/g' /etc/apt/sources.list
sudo sed -i 's/security.ubuntu.com/mirrors.cloud.tencent.com/g' /etc/apt/sources.list
sudo sed -i 's/archive.ubuntu.com/mirrors.cloud.tencent.com/g' /etc/apt/sources.list
sudo apt update
```
这两个云镜像在全国都有CDN，算是比较快的，当然也可以选择其他镜像源，比如中科大和清华的就不错，公网速度也很快，还支持IPv6。

## 图形界面

### Ubuntu Desktop Minimal

如果想直接在IPMI里面用GUI操作，可以安装最小化桌面获取更好的显示效果

```bash
sudo apt install ubuntu-desktop-minimal
```
重启或者直接启动显示管理器，Ubuntu Desktop的默认显示管理器是gdm3
```bash
sudo systemctl start gdm
```
如果一直远程操作，装了也没啥用，这IPMI最高也就1024×768的分辨率，虽然能改，但改起来挺麻烦得，还不能复制粘贴的，难受，所以我用GUI的时候一般用SSH转发X11或者VNC。

### Xubuntu Core

Ubuntu Desktop是基于Gnome的，最小化安装也只是不装libreoffice之类的应用，不太想搞这么重量级的桌面，装个轻量级 Xubuntu，能用GUI程序就行。也不装完整Xubuntu Desktop，只装一个Core。

```bash
sudo apt install xubuntu-core^
```
重启或者直接启动显示管理器，Xubuntu Desktop的默认显示管理器是lightdm
```bash
sudo systemctl start lightdm
```
### Video Driver

因为板载显卡是ASPEED的BMC控制器AST2500自带的，但是驱动默认不会安装，如果要接VGA显示器用可以安装驱动

```bash
sudo apt install xserver-xorg-video-ast
```
SSH和X11转发推荐使用MobaXterm，贼好用，个人可以免费使用，自带XServer，支持各种远程连接。

### 安装 Xrdp Server
IPMI还是没有直接网络远程操作流畅
装个xrdp就很方便，可以用Windows远程桌面直接连接
```bash
sudo apt install xrdp
sudo systemctl start xrdp
```
如果桌面上有些程序员提示错误`/usr/lib/x86_64-linux-gnu/xfce4/exo-1/exo-helper-1 (No such file or directory)`
```bash
sudo apt install libexo-1-0
```
可以修复这个问题


安装图形界面分区工具

```bash
sudo apt install gparted
```

可能会出现下面这样的提示，虽然一样用但是看着不舒服

```bash
W: Possible missing firmware /lib/firmware/ast_dp501_fw.bin for module ast
```
可以下载[`ast_dp501_fw.bin`](./ast_dp501_fw.bin)，上传到`/lib/firmware/`解决

## 安装VSCode
从微软官网https://code.visualstudio.com/docs/setup/linux 抄来的
```bash
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo install -o root -g root -m 644 microsoft.gpg /etc/apt/trusted.gpg.d/
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main" > /etc/apt/sources.list.d/vscode.list'
sudo apt install apt-transport-https
sudo apt update
sudo apt install code # or code-insiders
```

## 修改IPMIView输出分辨率
（以下命令须在IPMIView的窗口中的terminal emulator里面运行，不能在SSH客户端中执行！）

执行xrandr -q查询显示配置：
```bash
user@Gateway:~$ xrandr -q
Screen 0: minimum 320 x 200, current 1024 x 768, maximum 1920 x 2048
VGA-1 connected primary 1024x768+0+0 (normal left inverted right x axis y axis) 0mm x 0mm
  1024x768   60.00  
  800x600    60.32  56.25  
  640x480    59.94  
```
上面显示的一行“VGA-1 connected...”这里的VGA-1就是output名称，后面要用。从“maximum 1920 x 2048”可以看出最高支持的分辨率

原本这里是没有1920x1080的，这是我加了之后才想起来写教程，懒得重置了。
然后运行
```bash
user@Gateway:~$ cvt 1920 1080
# 1920x1080 59.96 Hz (CVT 2.07M9) hsync: 67.16 kHz; pclk: 173.00 MHz Modeline "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
```
得到必要的参数，将上面参数加入下面命令执行创建新显示模式
```bash
user@Gateway:~$ xrandr --newmode "1920x1080_60.00" 173.00 1920 2048 2248 2576 1080 1083 1088 1120 -hsync +vsync
```
然后加入新显示模式到上面得到的output设备中
```bash
user@Gateway:~$ xrandr --addmode VGA-1 1920x1080_60.00
```
然后切换到新模式
```bash
user@Gateway:~$ xrandr --output VGA-1 --mode 1920x1080_60.00
```
大功告成，其实没什么必要，太卡了，不如ssh转发X11和xrdp……

## 安装Webmin
一个基于BS架构的服务器管理工具，可以通过浏览器调整服务器设置，有点像家用路由器的感觉
```bash
wget http://www.webmin.com/jcameron-key.asc
sudo apt-key --keyring /etc/apt/trusted.gpg.d/webmin.gpg add jcameron-key.asc
sudo sh -c 'echo "deb https://download.webmin.com/download/repository sarge contrib" > /etc/apt/sources.list.d/webmin.list'
sudo apt-get update
sudo apt-get install webmin
```
虽然Ubuntu上还有一个叫Zentyal的玩意儿也是Web管理，更简单更强大，然而，这SB玩意儿会禁用IPv6，什么年代了，还禁用IPv6……不能忍

