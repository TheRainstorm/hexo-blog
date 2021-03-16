---
title: 'Vacation routine: ASC2020 tips (up to 02/09)'
date: 2020-02-09 22:39:14
tags:
---

记录了这么多天弄ASC2020学到的一些知识，原本记录在txt里:-)，因此有点丑。

<!--more-->

```plain
1/28
1. apt unmet dependency (depend: xxx but not going to install) 错误的可能原因。
2. virtualbox 的安装增强功能可以：1）提高分辨率；2）剪切板共享；3）共享文件夹
         以及如何命令行手动安装：1）点击菜单栏里的安装增强功能会将含有addition...iso镜像的cdrom加载。
			2）mount /dev/cdrom 到一个挂载点，推荐/mnt/cdrom（即mount /dev/cdrom /mnt/cdrom），其中cdrom可能需要自己创建。
			3）在挂载点 ./V**.run即可。（需要有gcc, make, perl才能安装成功)
3. ubuntu-server版没有X窗口因此无法复制粘贴，可以通过ssh连接。(1. 网络选择桥接模式 2. ifconfig查看ip地址 3. 运行sshd(server版安装时可勾选openssh，然后就会默认在22端口开启ssh服务) 4.ssh连接

1/29
1. 	chown [-r] user:grp file
	chmod [-r] xyz file 或 chmod [-r] ugo+-=rwx file
2.  软/硬链接
	ln f1 f2创建硬链接。f1, f2完全对等，删除f1，f2文件依然存在。为操作系统课中的无环图结构。当删除掉所有硬链接后，文件才会被真正删除。
	ln -s f1 f2创建软链接。f1和f2有主从关系。f2相当于一个很小的存储了f1的位置的特殊文件。删除掉f1后，f2失效。
3. 
	useradd [opt] username
		-g 用户所属的组
		-G 指定多个组
		-d 指定用户主目录
		-s 指定登录的shell
	userdel [-r] username 	-r会同时删除用户主目录
	usermod [opt] username
4.
	passwd修改当前用户密码（需要验证）
	(root)passwd username修改指定用户密码（无需验证）
	
	/etc/passwd文件一行记录对应着一个用户
	example line：
		sam:x:200:50:Sam san:/home/sam:/bin/sh
		用户名:口令(加密后的密码，或x而真实的密码位于/etc/shadow):用户标识号:组标识号:注释性描述:主目录:登录Shell
	
	/etc/shadow和/etc/group文件格式

02/05
1. 安装gcc
	1) linux下安装程序可分为:
		(1) 源码安装: 需要有编译环境如gcc环境，相对于离线安装，不用操心依赖，就是编译时间有点长。
		(2) 包安装(已经编译为二进制文件)：通过包管理器
		        	在线: 
				yum install package
		 	离线: 
				rpm -ivh rpm文件
			离线安装需要手动处理依赖。
				rpm -Uvh *.rpm --force --nodeps: 忽略依赖覆盖安装
		Ubuntu下为apt[-get]和dpkg
	3) 在centos的镜像的Packages下可找到各种程序的rpm文件
	4) GNU/gcc 中c, c++, fortran对应编译器需要分开安装
2. 编译安装mpich2
	1) 若没有g++, gfortran ./configure需要添加--disable-c++, --disable-fortran
	2) 还需要perl才能安装成功

02/06
1. Run an mpi cluster within a LAN
	1) 在每个节点创建mpiuser(添加到sudoers:使用visudo命令或vi /etc/passwd)(ubuntu为/etc/sudoers, 更简单的方式为sudo usermod -aG sudo 用户名)
	    修改/etc/hosts文件
	2) 	ssh-keygen -t dsa(rsa is ok, too)
		ssh-copy-id cilent
		ssh cilent		(默认使用当前用户mpiuser登入)
	3) nfs (该centos上已经安装nfs-utils-xxx, 既包含server又包含client)
		(1) 在server上vi /etc/exports:
			/home/mpiuser/cloud *(rw,sync,no_root_squash,no_subtree_check)
			exportfs -arv(全部, 重新,显示信息 挂载)
		     检查
			a. showmount [-ae] [hostname|IP]
			b. rpcinfo [-p] [IP|hostname]
			c. netstat -autp
		(2) 在client上
			mkdir cloud
			sudo mount -t nfs centos1:/home/mpiuser/cloud ~/cloud
			
			sudo vi /etc/fstab:
				#MPI CLUSTER SETUP
				centos1:/home/mpiuser/cloud /home/mpiuser/cloud nfs
			
			one line:
				(root)mkdir cloud;mount -t nfs centos1:/home/mpiuser/cloud /home/mpiuser/cloud;echo 'centos1:/home/mpiuser/cloud /home/mpiuser/cloud nfs'>>/etc/fstab
		     check
			df -h
	4) mpiexec -n N --hostfile hostfile executablefile
		N: 进程数
	    hostfile格式:
		MPICH2:	ip/host:n		n为该节点core数
		OpenMPI: ip/host slots=n
2. run HPL
	1) root 安装导致没有MPdir, MPinc, MPlib。决定重新安装。
	2) mpich2和openmpi都是
		mkdir build; cd build
		../configure --prefix=/path/to/install
		make; make install
	    >>>from mpich2 README:169
		IMPORTANT NOTE: The install directory has to be visible at exactly
    the same path on all machines you want to run your applications
    on. This is typically achieved by installing MPICH on a shared
    NFS file-system. If you do not have a shared NFS directory, you
    will need to manually copy the install directory to all machines
    at exactly the same location.
	   <<<
	全局安装会将可执行文件放在/bin, /usr/bin /usr/local/bin等里面, 库放在/lib, /usr/lib等
	而局部安装程序的所有依赖会存在于一个文件夹下, 可以复制移动到其它位置执行?(需要添加环境变量)
	3) BLAS库安装
	    ATLAS:
	    GotoBLAS2(Goto2):

	4) HPL&HPCG
	    HPL:
		cp Make.<arch> in top-level directory
		make arch=xxx
		cd /bin/<arch>
		run xhpl
	    HPCG:
		cp Make.<arch> to top/setup directory
		mkdir build
		cd build
   		 ../configure <arch>   #error1
		make
		cd /bin/
		run xhpcg
		
		#error1:/bin/sh^M: bad interpreter: No such file or directory
		#solve:
			vi ./configure
			:set ff/fileformat=unix
02/07
1. tips
	处理器型号, 主频, cache等信息: cat /proc/cpuinfo
	内存信息: cat /proc/meminfo
	磁盘信息: fdisk -l
	查看linux版本: lsb_release -a 

	top: 任务管理器
	free -m 查看内存使用状况
02/08
1.	OpenMP也是一个库，需要安装，目的是使得一个进程有多个线程。
2. rpm用法:
    (已安装)
	rpm -qa		all
	rpm -q pkg	返回带版本的包名
	rpm -qi pkg	info
	rpm -qf file	file: 查看文件属于哪个包
	rpm -ql pkg	list: 查看包安装的文件的路径
	rpm -qR pkg	依赖
	rpm -qc pkg	conf: 查看包的配置文件
    (未安装加个p选项, 且需要有包的rpm文件)
	rpm -qpl xxx.rpm
3. gcc -l选项与LD_LIBRARY_PATH
	编译时，如果不链接库(静态/动态)会出现undefined reference:
		[mpiuser@izwz9fx70wphqjnuv22919z dynamic_lib]$ gcc -I./include -std=c99 -o Test Test.c
		/tmp/cc4vyXu4.o: In function `main':
		Test.c:(.text+0x302): undefined reference to `mult_s'
		collect2: error: ld returned 1 exit status
	且如果库不在标准位置如/usr/lib/处的话(可通过gcc -lc --verbose查看, lc代表libc.a, 可换成其他库)，需要添加-L选项,
	否则会报/usr/bin/ld: cannot find: 
		[mpiuser@izwz9fx70wphqjnuv22919z dynamic_lib]$ gcc -I./include -lfpu -std=c99 -o Test Test.c
		/usr/bin/ld: cannot find -lfpu
		collect2: error: ld returned 1 exit status
	运行时, 如果链接库不在/etc/ld.so.conf里的话, 需要定义环境变量LD_LIBTARY_PATH才能运行
	否则会报cannot open shared object file:
		[mpiuser@izwz9fx70wphqjnuv22919z dynamic_lib]$ bin/Test_shared
		bin/Test_shared: error while loading shared libraries: libfpu.so: cannot open shared object file: No such file or directory
```