# cobbler
使用cobbler实现自动化安装系统

1 安装，cobbler的安装非常方便，直接使用yum安装即可

yum install cobbler cobbler-web dhcp tftp-server pykickstart  httpd xinetd -y

2 启动服务并下载cobbler工具

yum安装好httpd 和cobbler后，会自动启动httpd服务和cobbler服务，直接执行cobbler get-loaders即可，##若不执行此操作，则下一步检查时会报错。若下载失败，重启cobbler

 [root@linux-node1 ~]# cobbler get-loaders

task started: 2015-11-29_140933_get_loaders

task started (id=Download Bootloader Content, time=Sun Nov 29 14:09:33 2015)

downloading http://cobbler.github.com/loaders/README to /var/lib/cobbler/loaders/README

downloading http://cobbler.github.com/loaders/COPYING.elilo to /var/lib/cobbler/loaders/COPYING.elilo

downloading http://cobbler.github.com/loaders/COPYING.yaboot to /var/lib/cobbler/loaders/COPYING.yaboot

downloading http://cobbler.github.com/loaders/COPYING.syslinux to /var/lib/cobbler/loaders/COPYING.syslinux

downloading http://cobbler.github.com/loaders/elilo-3.8-ia64.efi to /var/lib/cobbler/loaders/elilo-ia64.efi

downloading http://cobbler.github.com/loaders/yaboot-1.3.17 to /var/lib/cobbler/loaders/yaboot

downloading http://cobbler.github.com/loaders/pxelinux.0-3.86 to /var/lib/cobbler/loaders/pxelinux.0

downloading http://cobbler.github.com/loaders/menu.c32-3.86 to /var/lib/cobbler/loaders/menu.c32

downloading http://cobbler.github.com/loaders/grub-0.97-x86.efi to /var/lib/cobbler/loaders/grub-x86.efi

downloading http://cobbler.github.com/loaders/grub-0.97-x86_64.efi to /var/lib/cobbler/loaders/grub-x86_64.efi

*** TASK COMPLETE ***

3 执行检查：并且按照提示解决相关的错误

[root@linux-node3 ~]# cobbler check

The following are potential configuration items that you may want to fix:

 

1 : The ‘server’ field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.    ###需要更改/etc/cobbler/settings 中的server 384行左右

2 : For PXE to be functional, the ‘next_server’ field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network. #更改/etc/cobbler/settings next_server 272行

3 : SELinux is enabled. Please review the following wiki page for details on ensuring cobbler works correctly in your SELinux environment:

    https://github.com/cobbler/cobbler/wiki/Selinux  #关闭selinux

4 : change ‘disable’ to ‘no’ in /etc/xinetd.d/tftp    #修改该配置文件

5 : change ‘disable’ to ‘no’ in /etc/xinetd.d/rsync   #修改该配置文件

6 : since iptables may be running, ensure 69, 80/443, and 25151 are unblocked

                                            #若防火墙开启，确保如上端口开启

7 : reposync is not installed, need for cobbler reposync, install/upgrade yum-utils? #若使用rsync来同步镜像，请下载安装该服务（本次使用本地镜像。忽略此行）

8 : debmirror package is not installed, it will be required to manage debian deployments and repositories #若安装的debian，需要安装debmirror

9 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to ‘cobbler’ and should be changed, try: “openssl passwd -1 -salt ‘random-phrase-here’ ‘your-password-here'” to generate new one

#更改root登陆的默认密码，将生成的值替换/etc/cobbler/settings中的101行的

default_password_crypted: “$1$sfUMbZwj$ddMdQwMcNwRdHVPqfBhTr.”

Restart cobblerd and then run ‘cobbler sync’ to apply changes.

/etc/cobbler/settings配置文件需更改的地方：

101 default_password_crypted: “$1$sfUMbZwj$ddMdQwMcNwRdHVPqfBhTr.”

242 manage_dhcp: 1

246 manage_dns: 0

258 manage_tftpd: 1

272 next_server: 10.0.0.10

292 pxe_just_once: 1

358 restart_dns: 1

359 restart_dhcp: 1

384 server: 10.0.0.10

注意 cobbler的配置文件遵循YAML语法，注意冒号后面要有空格（salt也是YAML语法）

change ‘disable’ to ‘no’ in /etc/xinetd.d/tftp       yes改为no

[root@linux-node1 ~]# openssl passwd -1 -salt ‘cobbler123456’ ‘123456’

$1$cobbler1$Lbn6lbZ9LrAjJdHWPjJO60

4 修改由cobbler管理的dhcp服务的配置文件

vim /etc/cobbler/dhcp.template

subnet 10.0.0.0 netmask 255.255.255.0 {                #修改子网的IP和子网掩码

option routers             10.0.0.2;              #修改默认网关/路由地址

option domain-name-servers 10.0.0.1;              #修改DNS地址

option subnet-mask         255.255.255.0;         #修改子网掩码

range dynamic-bootp        10.0.0.200 10.0.0.254; #DHCP地址池范围

default-lease-time         21600;

max-lease-time             43200;

next-server                $next_server;

}部分内容省略，其他地方不需要修改。

5 导入iso镜像

# mount /dev/cdrom /mnt                                       # 将光驱挂载到指定的目录

# cobbler import –path=/mnt/ –name=CentOS-7.1-x86_64 –arch=x86_64 #cobbler导入镜像文件

# cobbler distro list                                         # cobbler查看导入的镜像

6 上传自定义的kickstart启动文件并设置为默认。

cobbler profile report命令可以查看到已生成的profile信息和该profile使用默认的kickstart文件。

# cd /var/lib/cobbler/kickstarts/   #cobbler默认的启动文件存放路径

# rz -y                             #上传自定义启动文件CentOS-7.1-x86_64.cfg

# cobbler profile list              #查看profile列表

CentOS-7.1-x86_64

# cobbler profile report            #查看profile详细信息（可显示默认的kickstart文件）

补充：centos7默认情况下网卡的命名与centos6不一样，为了方便和统一标准化，需要修改一下内核参数，对应的profile里面的–kopts选项。(cobbler profile edit -h看帮助)

# cobbler profile edit –name=CentOS-7.1-x86_64 –kickstart=/var/lib/cobbler/kickstarts/CentOS-7.1-x86_64.cfg

### 修改默认的kickstart文件

# cobbler profile edit –name=CentOS-7.1-x86_64 –kopts=’net.ifnames=0 biosdevname=0′

###修改内核参数，使网卡命名规则为eth0、eth1

7 #修改Cobbler节目时的提示（可选，若有操作8，则修改了也看不到，会自动安装，不在询问）

vim /etc/cobbler/pxe/pxedefault.template

8 通过采集到的MAC地址，给主机分配角色和IP（自动化部署的关键）（MAC地址可以从虚拟机的网卡获得或是供应商供货清单提供）

cobbler system 可以理解为：为指定的系统选择一个角色（man cobbler）

[root@linux-node1 ~]# cobbler system add –name=nginx –mac=00:0C:29:21:04:C9 –profile=CentOS-7.1-x86_64 –ip-address=10.0.0.222 –subnet=255.255.255.0 –gateway=10.0.0.2 –interface=eth0 –static=1 –hostname=linux-node222.example.com –name-servers=”114.114.114.114,8.8.8.8″

上面配置的意思是：添加一个角色，名字为nginx，指定MAC地址为00:0C:29:21:04:C9的主机分配该角色，并指定profile文件，IP hostname等，这样，当MAC地址为00:0C:29:21:04:C9的主机在从网络启动后，cobbler就会自动按照system分配的角色来进行安装，不在进行任何界面的提示和等待。

9 重启cobbler服务并进行同步相关配置

/etc/init.d/xinetd restart

/etc/init.d/httpd restart

/etc/init.d/cobbler restart

cobbler sync

10至此，cobbler配置完成。（若简单安装，则7步 8步不需要）

开启一台虚拟机，配置为从网络启动,即可实现无人值守自动安装linux系统（windows系统）

11 cobbler主机 动态观察系统日志，以更好的了解整个过程中，cobbler都做了哪些操作

tail -f /var/log/message

本条目发布于2015年11月28日。属于linux技术交流分类。
