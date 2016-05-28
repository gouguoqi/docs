### cobbler的环境准备
<pre>
[root@linux-node1 ~]# cat  /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
[root@linux-node1 ~]# uname -r
3.10.0-327.18.2.el7.x86_64
[root@linux-node1 ~]# getenforce 
Disabled
[root@linux-node1 ~]# systemctl stop firewalld
[root@linux-node1 ~]# ifconfig eth0|awk -F '[ :]+' 'NR==2 {print $3}'
192.168.56.11
[root@linux-node1 ~]# hostname
linux-node1
[root@linux-node1 ~]# rpm  -ivh http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
[root@linux-node1 ~]# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
</pre>

1.1 **安装cobbler**
<pre>
[root@linux-node1 ~]# yum install cobbler cobbler-web pykickstart httpd tftp dhcp xinetd
[root@cobbler-node1 ~]# systemctl start httpd
[root@cobbler-node1 ~]# systemctl start cobblerd
[root@cobbler-node1 ~]# cobbler check #检查存在的问题,逐一解决
The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : change 'disable' to 'no' in /etc/xinetd.d/tftp
4 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
5 : enable and start rsyncd.service with systemctl
6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.

如上各问题的解决方法如下所示：
1、修改/etc/cobbler/settings文件中的server参数的值为提供cobbler服务的主机相应的IP地址或主机名，如server: 10.0.0.101；
	[root@cobbler-node1 ~]# sed -i 's/server: 127.0.0.1/server: 10.0.0.101/' /etc/cobbler/settings
2、修改/etc/cobbler/settings文件中的next_server参数的值为提供PXE服务的主机相应的IP地址，如next_server: 10.0.0.101；
	[root@cobbler-node1 ~]# sed -i 's/next_server: 127.0.0.1/next_server: 10.0.0.101/' /etc/cobbler/settings
3、修改/etc/xinetd.d/tftp文件中的disable参数修改为 disable = no
4、执行 cobbler get-loaders 命令即可；否则，需要安装syslinux程序包，而后复制/usr/share/syslinux/{pxelinux.0,memu.c32}等文件至/var/lib/cobbler/loaders/目录中；
5、执行 systemctl enable rsyncd命令即可；
6、如果有强迫症可以选择 yum –y install debmirror 然后根据错误进行解决,一般错误如下。
	注释/etc/dedmirror.conf文件中的  @dists=”sid”;  @arches=”i386”; 
7、[root@cobbler-node1 ~]# openssl passwd -1 -salt '$(openssl rand -hex 4)' 'xuliangwei'
$1$$(openss$.wbDUBV/STL0YaNuAcusK/
[root@cobbler-node1 ~]# grep "default_password_crypted" /etc/cobbler/settings  #替换/etc/cobbler/setting内的default_password_crypted
default_password_crypted: "$1$$(openss$.wbDUBV/STL0YaNuAcusK/"
8、yum –y install cman fence-agents
最后重启Cobbler：systemctl restart cobblerd 
</pre>
1.2 **配置dhcp**
<pre>
[root@cobbler-node1 ~]# sed -i 's#manage_dhcp: 0#manage_dhcp: 1#g' /etc/cobbler/settings #使用cobbler管理dhcp
[root@cobbler-node1 ~]# vim /etc/cobbler/dhcp.template #修改cobbler的dhcp模版，因为cobbler会替换。
subnet 10.0.0.0 netmask 255.255.255.0 {
     option routers             10.0.0.2;
     option domain-name-servers 10.0.0.2;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        10.0.0.200 10.0.0.250;
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                $next_server;
</pre>
1.3 **同步cobbler**
<pre>
[root@cobbler-node1 ~]# systemctl restart xinetd #重启xinetd
[root@cobbler-node1 ~]# systemctl restart cobblerd #重启cobbler
[root@cobbler-node1 ~]# cobbler sync #同步最新cobbler配置，可以看具体做了哪些操作
</pre>
1.4 **导入系统镜像**
<pre>
[root@cobbler-node1 ~]# mount /dev/cdrom /mnt/ #挂在ISO光盘至服务器
mount: /dev/sr0 is write-protected, mounting read-only
[root@cobbler-node1 ~]# cobbler import --path=/mnt/ --name=CentOS-7.1-x86_64-distro --arch=x86_64
# --path 镜像路径
# --name 为安装源定义一个名字
# --arch 指定安装源是32位、64位、ia64, 目前支持的选项有: x86│x86_64│ia64
# 安装源的唯一标示就是根据name参数来定义，本例导入成功后，安装源的唯一标示就是：CentOS-7.1-distro-x86_64。
# 镜像存放目录，cobbler会将镜像中的所有安装文件拷贝到本地一份，放在/var/www/cobbler/ks_mirror下的CentOS-7.1-x86_64-distro-x86_64目录下。因此/var/www/cobbler目录必须具有足够容纳安装文件的空间。
[root@cobbler-node1 ~]#  cobbler distro list #列出所有的distro
   CentOS-7.1-distro-x86_64
[root@cobbler-node1 ~]# cobbler profile list #导入distro会自动生成profile
   CentOS-7.1-distro-x86_64
</pre>
1.5 **管理**
<pre>
[root@linux-node1 mnt]# cobbler profile edit --name=CentOS-7.1-x86_64  --kickstart=/var/lib/cobbler/kickstarts/CentOS-7.1-x86_64.cfg
然后修改centos7的内核参数，把网卡名字改为eth0
[root@linux-node1 mnt]# cobbler profile edit --name=CentOS-7.1-x86_64 --kopts='net.ifnames=0 biosdevname=0'
然后重新sync一下 （必须要执行）
[root@linux-node1 mnt]# cobbler sync
task started: 2016-05-23_215439_sync
task started (id=Sync, time=Mon May 23 21:54:39 2016)
running pre-sync triggers
cleaning trees
removing: /var/www/cobbler/images/CentOS-7.1-x86_64
removing: /var/lib/tftpboot/pxelinux.cfg/default
removing: /var/lib/tftpboot/grub/images
removing: /var/lib/tftpboot/grub/grub-x86.efi
removing: /var/lib/tftpboot/grub/grub-x86_64.efi
removing: /var/lib/tftpboot/grub/efidefault
removing: /var/lib/tftpboot/images/CentOS-7.1-x86_64
removing: /var/lib/tftpboot/s390x/profile_list
copying bootloaders
trying hardlink /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
copying: /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
trying hardlink /var/lib/cobbler/loaders/grub-x86.efi -> /var/lib/tftpboot/grub/grub-x86.efi
trying hardlink /var/lib/cobbler/loaders/grub-x86_64.efi -> /var/lib/tftpboot/grub/grub-x86_64.efi
copying distros to tftpboot
copying files for distro: CentOS-7.1-x86_64
trying hardlink /var/www/cobbler/ks_mirror/CentOS-7.1-x86_64/images/pxeboot/vmlinuz -> /var/lib/tftpboot/images/CentOS-7.1-x86_64/vmlinuz
trying hardlink /var/www/cobbler/ks_mirror/CentOS-7.1-x86_64/images/pxeboot/initrd.img -> /var/lib/tftpboot/images/CentOS-7.1-x86_64/initrd.img
copying images
generating PXE configuration files
generating PXE menu structure
copying files for distro: CentOS-7.1-x86_64
trying hardlink /var/www/cobbler/ks_mirror/CentOS-7.1-x86_64/images/pxeboot/vmlinuz -> /var/www/cobbler/images/CentOS-7.1-x86_64/vmlinuz
trying hardlink /var/www/cobbler/ks_mirror/CentOS-7.1-x86_64/images/pxeboot/initrd.img -> /var/www/cobbler/images/CentOS-7.1-x86_64/initrd.img
Writing template files for CentOS-7.1-x86_64
rendering DHCP files
generating /etc/dhcp/dhcpd.conf
rendering TFTPD files
generating /etc/xinetd.d/tftp
processing boot_files for distro: CentOS-7.1-x86_64
cleaning link caches
running post-sync triggers
running python triggers from /var/lib/cobbler/triggers/sync/post/*
running python trigger cobbler.modules.sync_post_restart_services
running: dhcpd -t -q
received on stdout: 
received on stderr: 
running: service dhcpd restart
received on stdout: 
received on stderr: Redirecting to /bin/systemctl restart  dhcpd.service

running shell triggers from /var/lib/cobbler/triggers/sync/post/*
running python triggers from /var/lib/cobbler/triggers/change/*
running python trigger cobbler.modules.scm_track
running shell triggers from /var/lib/cobbler/triggers/change/*
*** TASK COMPLETE ***
</pre>
然后可以开始装机了
