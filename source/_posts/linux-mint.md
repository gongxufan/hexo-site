---
layout: post
title: "Linux mint作为开发机的日常使用记录，持续更新"
date: 2017-10-26 09:42
tags: linux
category: 随笔
description: 记录linux-mint系统使用笔记。

---
## 配置软件源
更新系统：
```sql
sudo apt update  
sudo apt-get upgrade  
```
![img](/upload/images/linux/source.png) 
这里选择网络连接最快的源
## 中文输入法问题
作为开发机器装好系统第一件事就是中文输入法的配置，在安装的时候选择中文语言会自动下载相关的语言包。
1) 使用系统默认的fcitx的拼音输入法：
发现输入中文的时候根本看不到候选框。解决办法：
```sql
sudo apt purge fcitx-ui-qimpanel  
sudo apt-get  install fcitx-config-gtk  
```
在附加组件把qimpanel勾选去掉  
原因是fcitx跟QT冲突
2) 使用sogou输入法：
下载地址：http://cdn2.ime.sogou.com/dl/index/1491565850/sogoupinyin_2.1.0.0086_amd64.deb?st=91JBmvFURKntLXuHf1VhkA&e=1508982588&fn=sogoupinyin_2.1.0.0086_amd64.deb
安装：
```sql
sudo apt install libopencc2 libopencc1 fcitx-libs fcitx-libs-qt  
sudo dpkg -i sogoupinyin_2.1.0.0086_amd64.deb  
```
安装之前需要安装依赖包，然后我们开始输入中文，发现候选框全是一堆乱七八糟的字母数字，原因是我们在第一步把qimpanel选项去掉了，在fcitx-config中重新启用然后注销登录之后发现可以正常使用了。
3)全新安装
如果系统装好后没有中文输入法，手动安装：
```sql
sudo apt install fcitx fcitx-googlepinyin fcitx-config-gtk fcitx-frontend-all fcitx-module-cloudpinyin fcitx-ui-classic    

```
完成后注销重新登录即可。
## 安装java
Oracle Java 8
1. sudo add-apt-repository ppa:webupd8team/java  
2. sudo apt-get update  
3. sudo apt-get install oracle-java8-installer 

## 安装网易云音乐
下载地址：http://s1.music.126.net/download/pc/netease-cloud-music_1.0.0-2_amd64_ubuntu16.04.deb
```sql
sudo apt install libqt5multimediawidgets5  
sudo dpkg -i netease-cloud-music_1.0.0-2_amd64_ubuntu16.04.deb  
```
或者直接`sudo apt install netease-cloud-music`
## 安装解码和flash插件
`sudo apt install flashplugin-installer gstreamer1.0-fluendo-mp3 ttf-mscorefonts-installer
`
## 安装MySQL
`sudo apt-get install mysql-server mysql-common`
安装过程会提示输入root密码，使用sudo mysql -uroot -p测试安装结果
## 安装chrome
下载地址：http://www.google.cn/chrome/browser/desktop/index.html?standalone=1
访问不了可以下载这个：http://download.csdn.net/download/qq381332153/10039953
`sudo dpkg -i google-chrome-stable_current_amd64.deb`
## 安装Android Studio
开启32位包支持
`sudo dpkg --add-architecture i386  `
下载地址：[https://developer.android.google.cn/studio/install.html](https://developer.android.google.cn/studio/install.html)
启动过程中提示设置代理下载SDK，地址设置为：mirrors.neusoft.edu.cn，端口80
## 开机挂在文件系统
在/etc/fstab可以配置系统启动的时候挂在指定文件系统。在双系统中我们一般会去挂载WIN下的磁盘：
```html
# /etc/fstab: static file system information.  
#  
# Use 'blkid' to print the universally unique identifier for a  
# device; this may be used with UUID= as a more robust way to name devices  
# that works even if disks are added and removed. See fstab(5).  
#  
# <file system> <mount point>   <type>  <options>       <dump>  <pass>  
# / was on /dev/sdb7 during installation  
UUID=d281aec2-a293-45e0-868b-546c763fbdd2 /               ext4    errors=remount-ro 0       1  
# /boot/efi was on /dev/sdb2 during installation  
UUID=861D-7C13  /boot/efi       vfat    umask=0077      0       1  
# /home was on /dev/sdb8 during installation  
UUID=e1bfa6c7-6a21-45c4-81c2-48ac209fc2c7 /home           ext4    defaults        0       2  
#C  
UUID=66CC2017CC1FDFDB    /media/C   ntfs  defaults,user,rw,iocharset=utf8,umask=000,nls=utf8    0    3  
#D  
UUID=503EC4343EC414BE    /media/D    ntfs  defaults,user,rw,iocharset=utf8,umask=000,nls=utf8    0    4 
``` 
上边我配置了C和D盘的启动自动挂载，文件系统格式是NTFS，UUID可以通过命令查看。其中每项的说明如下：

gongxufan@gongxufan-ThinkPad-E460 ~ $ blkid  
```sql
/dev/sda1: LABEL="M-fM-^AM-\"M-eM-$M-^M" UUID="92501C01501BEB2D" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="b50dc9c9-eb35-499d-b02c-ee8378a63282"  
/dev/sda2: UUID="861D-7C13" TYPE="vfat" PARTLABEL="EFI system partition" PARTUUID="6ecc9e85-c8a0-4623-b6f6-94bef54c975d"  
/dev/sda4: LABEL="C" UUID="66CC2017CC1FDFDB" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="cffc126d-484f-4b08-8462-c99f103a8175"  
/dev/sda5: UUID="0E201336201323ED" TYPE="ntfs" PARTUUID="2a95117a-f37c-4034-b298-da7d57ed0985"  
/dev/sda6: LABEL="D" UUID="503EC4343EC414BE" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="8fcb1827-9fed-4c71-9a82-f524d973c8ad"  
/dev/sda7: UUID="d281aec2-a293-45e0-868b-546c763fbdd2" TYPE="ext4" PARTUUID="ecd02f18-c001-4b32-888b-f2e28515116c"  
/dev/sda8: UUID="e1bfa6c7-6a21-45c4-81c2-48ac209fc2c7" TYPE="ext4" PARTUUID="6c2cbde7-ac4e-4beb-a863-958f5cb2bbf7"  
```
接着是挂载点，需要自己在系统预先创建好目录
第三项是目标文件系统的格式
接着是参数，一般中文的要设置UTF8，还有读写权限等
再接着是否备份，一般设置为0就可以
最后是fsck磁盘自检顺序，设置0表示不进行自检
最后可以通过sudo mount -a测试配置正确。
## home目录下模板目录的管理
默认情况下会有Downloads,Docments等模板目录，这些目录是根据语言来设置。具体如下：
```sql
gongxufan@gongxufan-ThinkPad-E460 ~/.config $ ls  
autostart  cinnamon-session  enchant         filezilla      gtk-3.0      menus          netease-cloud-music  SogouPY         tomboy          user-dirs.locale  xplayer  
caja       cobinja           fcitx           google-chrome  hexchat      mimeapps.list  pix                  SogouPY.users   Trolltech.conf  VirtualBox        xviewer  
chromium   dconf             fcitx-qimpanel  gtk-2.0        libreoffice  nemo           pulse                sogou-qimpanel  user-dirs.dirs  xed  
```
其定义：
```sql
gongxufan@gongxufan-ThinkPad-E460 ~/.config $ cat user-dirs.dirs   
# This file is written by xdg-user-dirs-update  
# If you want to change or add directories, just edit the line you're  
# interested in. All local changes will be retained on the next run  
# Format is XDG_xxx_DIR="$HOME/yyy", where yyy is a shell-escaped  
# homedir-relative path, or XDG_xxx_DIR="/yyy", where /yyy is an  
# absolute path. No other format is supported.  
#   
XDG_DESKTOP_DIR="$HOME/Desktop"  
XDG_DOWNLOAD_DIR="$HOME/下载"  
XDG_TEMPLATES_DIR="$HOME/模板"  
XDG_PUBLICSHARE_DIR="$HOME/公共的"  
XDG_DOCUMENTS_DIR="$HOME/文档"  
XDG_MUSIC_DIR="$HOME/音乐"  
XDG_PICTURES_DIR="$HOME/图片"  
XDG_VIDEOS_DIR="$HOME/视频"  
```
该模板是根据l语言自动设置的：
user-dirs.locale文件定义了当前的语言
## 查看硬件信息并保存到html文件
`sudo lshw -html > ~/hw.html  `
## 安装numpy

for python3+
`sudo apt-get install python3-numpy python3-scipy python3-matplotlib ipython3 ipython3-notebook python3-pandas python-sympy python3-nose  `

for python2+
`sudo apt-get install python-numpy python-scipy python-matplotlib ipython ipython-notebook python-pandas python-sympy python-nose `
## 禁用笔记本触摸板
禁用 sudo rmmod psmouse 
开启 sudo modprobe psmouse
注意这里执行禁用会把触点功能也关闭，如果只禁用触摸板的可以安装pointing-device
## 安装redis
`sudo apt-get install redis-server`
### 查看进程
```
gongxufan@gongxufan-ThinkPad-E460 ~ $ ps -aux|grep redis
redis     5938  0.1  0.0  41880  6644 ?        Ssl  15:44   0:00 /usr/bin/redis-server 127.0.0.1:6379
gongxuf+  6040  0.0  0.0  15988  1020 pts/0    S+   15:45   0:00 grep --color=auto redis

```
### 查看服务状态
```
gongxufan@gongxufan-ThinkPad-E460 ~ $ sudo /etc/init.d/redis-server status
● redis-server.service - Advanced key-value store
   Loaded: loaded (/lib/systemd/system/redis-server.service; enabled; vendor preset: enabled)
   Active: active (running) since 四 2017-11-16 15:44:11 CST; 1min 55s ago
     Docs: http://redis.io/documentation,
           man:redis-server(1)
 Main PID: 5938 (redis-server)
   CGroup: /system.slice/redis-server.service
           └─5938 /usr/bin/redis-server 127.0.0.1:6379

11月 16 15:44:11 gongxufan-ThinkPad-E460 systemd[1]: Starting Advanced key-value store...
11月 16 15:44:11 gongxufan-ThinkPad-E460 run-parts[5920]: run-parts: executing /etc/redis/redis-server....ple
11月 16 15:44:11 gongxufan-ThinkPad-E460 run-parts[5939]: run-parts: executing /etc/redis/redis-server....ple
11月 16 15:44:11 gongxufan-ThinkPad-E460 systemd[1]: Started Advanced key-value store.
Hint: Some lines were ellipsized, use -l to show in full.
```
### 查看服务端口
```
gongxufan@gongxufan-ThinkPad-E460 ~ $ netstat -nlt|grep 6379
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN     
```
### 客户端访问
```
gongxufan@gongxufan-ThinkPad-E460 ~ $ redis-cli
127.0.0.1:6379> help
redis-cli 3.0.6
Type: "help @<group>" to get a list of commands in <group>
      "help <command>" for help on <command>
      "help <tab>" to get a list of possible help topics
      "quit" to exit
127.0.0.1:6379> keys
(error) ERR wrong number of arguments for 'keys' command
127.0.0.1:6379> keys *
(empty list or set)
127.0.0.1:6379> 

```
## 安装nginx
`sudo apt-get install nginx`
配置文件位于：/etc/nginx
服务启动控制：
```
#重新加载配置
sudo service nginx reload
#启动停止
sudo service nginx restart
sudo service nginx stop
```

