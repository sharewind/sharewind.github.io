---
layout: post
title: "Linux 常用命令大全"
date: 2013-03-17 18:25
comments: true
categories: Linux Shell 
---


【帮助命令】  
command --help  
man command  
man 2 command 查看第2个帮助文件  
man -k keyword 查找含有关键字的帮助  
info command 查看指令的帮助信息  
whatis command 获取指令索引的简短说明  
whatis apropos keyword = man -k keyword  

type command  显示是内建指令/别名/path 中的路径  
type -a command 显示path 中的命令路径  
which [command]  查找命令的所在位置  
whereis [command]  显示系统命令及其文档所在目录  
sh  shell_script_file 以shell 的方式执行命令  


【登录】  
login  
logout 或 exit (Ctrl + D)  

【系统管理】  
reboot    重新开机  
shutdown -r now 立即重启  
shutdown -h now 立即关机  
shutdown -c 取消关机  

date    显示或设置系统时间与日期。  
cal    显示系统日历。  
exit    退出目前的shell。  

su 变更用户身份  
sudo 以其他身份来执行指令。  
sudo !! 以管理员身份执行上次指令。  

【系统日志查看】  
uptime 查看系统负载与 运行时间  
last reboot 查看上次重启的时间  
lastlog  查看上次登录时间  
less /var/log/message  可以滚动浏览日志信息  
tail -1000f  /var/log/message 查看日志文件最后1000行，并继续监控文件并输出新内容。  
head /var/log/messages 查看日志文件的头10行  
dmesg |more  查看最后一次系统引导的引导日志。  
more 分页查看日志信息
 

【查看系统限制】  
ulimit -n   
ulimit -a  查看系统的连接数限制  
或者写入 ulimit -n 65536  >>  /etc/profile    
cat /etc/security/limits.conf  
cat /proc/sys/fs/file-max  


【文件查看】  
cat [文件名]     输出文件   
tail -10 -f filename 显示文件最后10行(参数-f 不停地读取文件最新的内容)  
head -10 filename  查看文件的头10行  
more filename  分页显示文件内容  
less filename  可翻页滚屏的文件查看

diff [文件或目录1] [文件或目录2]     比较文件的差异  

【文档编辑】  
vi 编辑文档命令  
awk 文本编辑指令  
grep 查找文件里符合条件的字符串  
sed    利用script来处理文本文件  
wc    计算字数  
wc -l  统计行数  

【目录】  
~ 用户主目录  
/ 根目录  

【文件管理】 
touch  [filename] 生成空文件或改变文件时间截  
pwd    显示当前目录。  
realpath [文件名] 显示当前文件的真实路径 (需要安装)  
cd    切换目录。  
cd - 切换到上一次访问的目录。  
mkdir  -p [目录结构]  建立目录。  
mkdir -m 755 newdir  建立目录并指定权限  
rmdir    删除目录。  
rmdir  -p 逐层删除目录。  

du    显示目录或文件的大小。  
du -sh dir 显示目录的汇总大小。  
du -h --max-depth=0 显示当前文件的文件大小，指定了深度  
df  -ahT  显示磁盘的相关信息。  

ls    列出目录内容。  
ll --time-style full-iso   完全格式时间  
ll -t   按时间排序   
ls -lrt  最新的在最后面。  
tree    以树状图列出目录的内容。  

cp -r  [源文件] [目标文件]  复制  
cp -p   保留原文件的日期  
ln -s  [源文件] [目标文件]   创建符号链接  
mv  [源文件] [目标文件|新名称]   移动或重命名现有的文件或目录  
rm -rf *  删除文件或目录  
rm -ri  删除文件并确认  
split -n     切割文件  

【权限管理】  
chown -R [user.group]     变更文件或目录的拥有者或所属群组  
chmod -R [ugo|a] [rwx-]  变更文件或目录的权限  
chgrp -R     变更文件或目录的所属群组  
（不常用）umask 设置文件的默认权限 掩码  

【文件查找】  
grep 命令 
 
	grep -r  keyword /home/cjf  在指定目录/home/cjf 查找 包含关键字 文件  
	grep -r --include=*.java  keyword  /home/cjf/   查找指定目录下某一类型文件，包含keyword的文件  
	grep -v "keyword"  忽略掉含有关键词  

find 查找文件或目录   

	查找文件名中含有activity的java文件  
	find  path  -name *.java  -name *Activity*  
	find  /home/cjf/  -name *.java  -name *Activity*  

    查找文件中含有 SwitchyPac 的文件 （建议用grep 实现 ）  
    find /etc -name '*' -type f -exec cat {} \;|grep 'SwitchyPac'  

locate    通过索引查找文件  
     cd / && locate *.desktop  
updatedb 建立或更新locate 使用的索引数据库  


【文件传输】  
scp local_file user@host:remote_file 本地上传文件到远程    
scp user@host:remote_file local_file  下载远程文件到本地   

	scp ./cloudatlas-topic-service-dist.tar.gz  root@192.1.1.202:/opt/webapps/cloudatlas-topic-service-dist.tar.gz  


wget [url]  -P  [local_dir] 利用wget下载文件  

lftp,sftp      

	     lftp sftp://ip
	     user root
	     password
	     mget file
	     exit
快速启用http服务  python -m SimpleHTTPServer  


【磁盘管理】   
df -ah   显示磁盘的相关信息。  
mount    挂载设备   
 
     mount / mount -l 列出当前已挂载的文件系统  
     mount -a 从/etc/fstab 挂载所有文件，可用来测试当前配置是否正确   
     mount -t vfstype  -o options  dev dir    挂载文件系统类型为vfstype 的 dev 设备到 目录  dir.  
          写入 /etc/fstab 实现开机自动挂载       

     sudo mkdir /media/Work      
     sudo mount -t ntfs -o rw /dev/sda3 /media/Work
     sudo umount /dev/sda3
     sudo rmdir /media/Work

     sudo mount -t ntfs -o rw,nosuid,nodev,allow_other /dev/sda3 /media/Work
     
     挂载光盘
     mkdir /media/iso
     mount -o loop  linux.iso /media/iso
     

umount    卸除文件系统。  
umount -a  卸载/etc/mtab 所有的文件系统  
quota    显示磁盘已使用的空间与限制。  


【磁盘维护】  
dd      dd可从标准输入或文件读取数据，依指定的格式来转换数据，再输出到文件，设备或标准输出。  
fdisk -l   列出所有磁盘分区。  
mkswap     设置交换区(swap area)。  

【网络通讯】  
hostname 显示或修改主机名(临时有效)  
vi /etc/hostname 修改主机名  
dnsconfig    设置DNS服务器组态。  
ifconfig    显示或设置网络设备。  

netstat   -tulnp|grep  [port|processname]    显示网络状态。  
ss -l  显示正处于监听状态的socket  
ss -s 显示socket 统计信息  
lsof 显示打开的文件  
lsof -p pid  
lsof -i  显示打开的IPv4网络连接  
lsof -i|grep pid|wc -l 显示某个进程打开的网络连接  

tcpdump   

ping -c 3 www.google.com   检测主机。  
traceroute    显示数据包到主机间的路径。  
nslookup [域名domain] 显示域名的dns 服务器  
nslookup www.baidu.com  
mtr google.com     traceroute + ping  google.com  

nc    设置路由器。  
samba    Samba服务器控制。 

【网络代理】  
http代理 http_proxy  
https安全代理 https_proxy  
ftp理  ftp_proxy  
不使用代理  no_proxy  

export https_proxy=localhost:8087  

	[inbi@debian ~]#export http_proxy=itwhy:123456@proxy.itwhy.org:8080
	#http_proxy：表示使用http代理方式
	#itwhy：是代理使用的用户名
	#123456：密码啊！
	#proxy.itwhy.org：代理地址，可以是IP，也可以是域名
	#8080：使用的端口
	#如果需要永久有效，需要将以上命令写入文件哦！例如：
	[inbi@debian ~]#echo "export http_proxy=proxy.itwhy.org:8888" > ~/.profile


【进程或性能】  
top 管理执行中的程序。  
top -p pid -H  查看进程中线程的运行状态  

free  -m  显示内存状态。  
vmstat 报告系统内存状态.  
vmstat -S m 1  每1秒打印系统状态   
pmap pid 查看某个进程的内存占用状态  
strace -p pid 跟踪linux 系统调用   

sar   
sar -d 查看磁盘IO统计   
sar -n SOCK 查看socket 连接  
sar -n DEV 查看网络情况  
sysctl -a 查看系统内核参数  
vi /etc/sysctl.conf  
sysctl -p 永久修改内核参数  

iostat 显示当前IO状态  
time 查看命令执行的时间  

last    列出目前与过去登入系统的用户相关信息。  
lastlog 上次登录日志  
last reboot 上次重启记录  
uptime 显示当前系统的负载情况  

ps aux|grep [processname]  查找进程   
ps -ef|grep [processname]  查找进程 (可以看到父进程id)  
ps axu|grep qemu|awk ‘{print $2}’|xargs kill -9  杀死进程名称中包含qemu的所有进程  
ps -efww|grep LOCAL=NO|grep -v grep|cut -c 9-15|xargs kill -9 　杀死进程命令行中包含LOCAL=NO的所有进程  
pkill 通过进程名杀死进程  
ps e 查看进程所用的环境变量  
pstree 以树状图显示程序。  
kill 删除执行中的程序或工作。 

【后台执行】  
ctrl + c 终止并退出前台命令的执行，回到shell  
crrl + z 暂停前台命令的执行，将该进程放入后台（暂停状态），回到shell  
jobs 查看后台运行的任务,可查看命令进程作业号job id  
     + 代表当前的默认作业。  
     -  代表下一个默认作业。  
command  & (加在命令末尾)让程序在后台运行，如果终端关闭，那么程序也会被关闭。  
bg N 让作业号为N的进程在后台运行  
fg N  让作业号为N的进程恢复到前台运行  
%% 或 %+ 表示默认作业号  
%N   让作业号为N的进程恢复到前台运行  
kill %N 可以杀死对应的作业进程   
nohup command  [args] [&] 让程序永远在linux后台运行。  
setsid command 在新的会话中运行命令，父进程id 为1.  

【定时任务】  
crontab [-u user] -l  列出定时任务  
crontab [-u user]  -e 编辑定时任务  
crontab [-u user]  -r  删除定时任务  


【用户管理】 
adduser    新增用户帐号。  
useradd    建立用户帐号。  
userconf    用户帐号设置程序。  
userdel 删除用户帐号。  
usermod    修改用户帐号。  
w    who    显示目前登入系统的用户信息。  
password    设置密码。  

groupdel [群组名称]    删除群组。  
groupmod [-g <群组识别码> <-o>][-n <新群组名称>][群组名称]   更改群组识别码或名称。  

【系统设置】  
hostname 显示或修改主机名  
cat /etc/profile 显示系统配置  

export 查看所有环境变更，同windows中的set  
export [-fnp][变量名称]=[变量设置值]    设置或显示环境变量。  

alias[别名]=[指令名称]     设置指令的别名。  
unalias    删除别名。  

chroot    改变根目录。  
clear    清除屏幕。  
depmod    分析可载入模块的相依性。  

【服务管理】  
chkconfig --list    检查，设置系统的各种服务。  
     chkconfig --list|grep on  
     chkconfig servicename on  
ntsysv 设置系统的各种服务。  
服务启动配置路径 /etc/init.d  


【软件安装】
1.readhat 系统  

yum search   [软件名]  
yum install  [软件名]  

rpm -i [软件名]  安装软件  
rpm -e [软件名]  删除软件  
rpm -V 验证软件安装  
rpm -U 升级  
rpm -q [软件名]  查询软件情况  
rpm -qa|grep [关键字]  查询软件是否已安装  

2. ubuntu 系统

apt-get update 更新软件列表  
apt-cache search [软件名]  
apt-get install [软件名]  
apt-get remove [软件名]    

dpkg -L [软件名]  显示软件安装目录   


【文件磁盘大小】
du -ah --max-depth =1 查看文件夹大小  
ll -ah  查看文件本身大小  
df -ah 查看当前磁盘分区占用情况。   
fdisk -l 查看硬盘分区的情况 。  
lsblk  查看物理硬盘列表。  

【压缩】  
gzip 压缩文件  
tar  压缩指令  

	1.将当前目录下所有.txt文件打包并压缩归档到文件this.tar.gz，我们可以使用
	tar -zcvf  target.tar.gz  ./*.txt \
	tar -zcv  srcfolder -f target.tar.gz

	2.将当前目录下的this.tar.gz中的文件解压到当前目录我们可以使用
	tar -zxvf this.tar.gz ./
	tar -zxvf apache-tomcat.gz -C  /opt

tar -xvf file.tar 

zip [参数] [文件列表]

     zip -r  test.zip test/*
unzip test.zip
bzip2  压缩产生bz2后缀的文件
bunzip2

jar 指令  

	jar -cvfM0  game.war  ./   将当前目录或指定目录打包成war  
	jar -xvf game.war  解压war到当前目录  


【备份】  
dump   
restore 

【SSH】  
.ssh 文件夹下  
ssh-keygen -t rsa -f id_rsa  生成RSA密钥对  
cp id_rsa  ~/.ssh/authorized_key/   

---

【系统信息查看】

查看系统与内核信息
uname -r 查看系统kernal 版本  
uname -a  显示全部信息  
lsb_release -a  查看当前系统的发行版本信息  
cat /etc/issue 查看查看系统的发行版  
cat /proc/version 查看当前系统的发行版本  
getconf LONG_BIT 查看当前的Linux计算机是32位或64位  
cat /etc/profile 查看环境变量  

**查看硬件信息**  
lsblk 逻辑块设备，可以查看挂载的硬盘信息  
lscpu 查看cpu  
cat /proc/cpuinfo 查看cpu详细信息  
lsusb 查看usb 接口   
lsmod    program to show the status of modules in the Linux Kernel  
hostname 查看当前系统的主机名  

**查看内存**  
free   
free -m 以MB的单位查看  
free -g  以GB为单位查看   
vmstat  
cat /proc/meminfo 查看内存信息  

