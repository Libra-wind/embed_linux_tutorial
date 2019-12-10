.. vim: syntax=rst


挂载NFS网络文件系统
-----------

本章将介绍如何挂载NFS网络文件系统，为后面的主机编译生成的ARM Linux应用传输到开发板做准备。

本章的示例exports文件目录为：base_code/section2/nfs_config/exports。

网络文件系统简介
~~~~~~~~

网络文件系统，常被称为NFS（Network File System），它是一种非常便捷的在服务器与客户端通过网络共享文件的方式，具体见图 15‑1。

|mountn002|

图 15‑1 NFS服务器与客户端连接

开启了NFS服务后，客户端访问服务器共享的文件时如同访问本地存储器（磁盘/SD卡/NAND FLASH等）上的文件一样，对于上层应用来说没有任何差别，在嵌入式开发时，我们常常利用这个特性在主机上共享文件，主要应用场景如下：

-  在NFS服务器上编译应用软件，客户端（开发板）通过NFS访问并运行应用程序进行测试。

-  把NFS作为根文件系统来启动

使用NFS的实验环境架构
~~~~~~~~~~~~

在后面章节的实验中，我们常常通过NFS给开发板共享开发主机编写的应用程序，在本节内容将介绍如何在开发板和开发主机之间共享目录。我们要构建的使用NFS文件系统的实验环境架构如图 15‑2。

|mountn003|

图 15‑2 使用NFS的实验环境架构

在这样的环境中，开发板与开发主机接入到同一个局域网中，然后开发主机提供NFS服务，开发板通过NFS与开发主机连接共享文件。开发主机生成的目标板应用程序放在NFS的共享文件夹内，开发板访问该文件夹执行应用程序进行测试。

在另一方面，开发主机与开发板还有串口连接，使用串口终端控制开发板。

搭建NFS环境
~~~~~~~

接下来介绍如何利用NFS搭建上述实验环境。主要包含连接网络、主机开启NFS服务以及开发板挂载文件系统三个步骤。

我们的目标是使开发主机与开发板的以下目录映射起来：

开发主机共享目录：/home/embedfire/workdir

开发板的共享目录：/home/root/workdir

连接到局域网络
^^^^^^^

连接到局域网
''''''

在本应用场景中开发主机和开发板需要通过网络互相访问，另外由于NFS文件系统暴露到公网需要处理很多安全问题，为简化操作，我们把开发主机和开发板连接到局域网络，即都使用网线把它们连接到同一个交换机（路由器）上，如前面的图 15‑2所示。

如果开发主机是安装在虚拟机上，注意要在VirtualBox把虚拟机的网络配置改成“桥接网卡”模式，见 图 15‑3，若修改了该配置，需要重启虚拟机才生效。

|mountn004|

图 15‑3虚拟机的网络配置为“桥接网卡”模式

互ping测试
'''''''

连接好网络后在开发主机和开发板之间进行互ping测试，以保证网络互通。

ping时可以直接使用主机名，各自的主机名可以在终端上查看，见图 15‑4。

|mountn005|

图 15‑4 查看主机名

在终端输入提示符的“@”与“:”之间的就是主机名。例如本示例中的开发主机名为“dev”，开发板主机名为“imx6ull14x14evk”，互ping测试成功的结果如图 15‑5，ping命令格式为：

ping [目标主机名或目标ip地址]

若测试成功，会返回ttl域和时间，使用Ctrl+c可退出命令。

|mountn006|

图 15‑5 互ping测试

在图 15‑5中我们还可以了解到本示例中开发主机的IP为192.168.0.219，开发板的IP为192.168.0.215，它们处于192.168.0.*的网段，即在同一个局域网内。

如果使用主机名无法ping通，请使用ifconfig命令查看各自的IP并直接用IP地址进行ping测试，见图 15‑6。

|mountn007|

图 15‑6 查看开发板IP并进行ping测试

注：若在开发主机上使用ifconfig提示找不到命令，请使用如下命令进行安装：

#以下命令在主机上运行

sudo apt-get install net-tools

在图 15‑6中开发板使用ifconfig命令可查看到开发板有两个网卡，分别是其两个网络接口的信息，其中eth0没有接网线，所以不能正常使用，而eth1可看到其IP地址为192.168.0.215，在开发主机的终端上可使用该IP进行ping测试。

若使用ifconfig命令查看不到IP地址，或IP地址与开发主机的IP不在同一个网段，请检查网络连接。

在开发主机开启NFS服务
^^^^^^^^^^^^

确认网络并了解局域网IP地址信息后，可配置开发主机的NFS服务，以下步骤说明均在开发主机上进行操作。

安装NFS服务
'''''''

Ubuntu系统默认没有安装NFS服务，需要使用如下命令安装NFS服务端软件：

#以下命令在主机上运行

sudo apt-get install nfs-kernel-server

查看用户id
''''''

在配置NFS时需要使用到用户uid和组gid，可使用id命令查看，在开发主机上的终端输入如下命令：

#以下命令在主机上运行

id

具体见图 15‑7。

|mountn008|

图 15‑7 使用id命令用户id和组id

从图 15‑7可知本开发主机的用户uid和组gid均为998，请使用该命令查看自己的开发主机相关id，在下面的配置文件中将会使用到。

配置NFS
'''''

安装NFS服务后，会新增一个/etc/exports文件（即/etc目录下名字为exports的文件），NFS服务根据它的配置来运行，其默认内容可通过命令cat /etc/exports查看，它默认包含了一些配置的范例，内容如图 15‑8。

图 15‑8 文件/etc/exports的默认内容（/etc/exports文件）

1 # /etc/exports: the access control list for filesystems which may be exported

2 # to NFS clients.
See exports(5).

3 #

4 # Example for NFSv2 and NFSv3:

5 # /srv/homes hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)

6 #

7 # Example for NFSv4:

8 # /srv/nfs4 gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)

9 # /srv/nfs4/homes gss/krb5i(rw,sync,no_subtree_check)

详细的帮助说明可以使用 命令man nfs查看，此处直接以我们配置的范例进行讲解。

修改配置文件常常需要系统用户权限，所以通常使用sudo vim或sudo gedit来打开修改，要使用gedit编辑器的话，把后面的vim命令换成gedit即可。

使用vim打开/etc/exports文件命令如下：

#以下命令在主机上运行，可用gedit替换vim

sudo vim /etc/exports

在/etc/exports文件末尾添加如下语句并保存，注意如下语句写到/etc/exports文件是在同一行，此处由于排版问题分行了。

#把以下内容添加至/etc/exports文件末尾，注意以下内容处于同一行

#以下内容的IP地址和uid，gid需要根据自己的环境进行修改

/home/embedfire/workdir 192.168.0.0/24(rw,sync,all_squash,anonuid=998,anongid=998,no_subtree_check)

注意具体的配置需要根据自己的实验环境进行配置，请理解如下说明根据自己的实验环境进行修改：

-  /home/embedfire/workdir：要共享的开发主机目录，注意使用空格与后面的配置隔开。

-  192.168.0.0/24：配置谁可以访问，其中的/24是掩码，此处表示24个1，即11111111.11111111.11111111.00000000，即掩码是255.255.255.0。结合前面192.168.0.0表示此处配置IP为 192.168.0.\*
  的主机均可以访问该目录，即局域网上的所有主机。

-  若局域网是其它网段，请参考此处的配置，不能直接用星号表示，如欲配置192.168.1.\* 的局域网下所有机器都可以访问，则配置为 192.168.1.0/24。

-  这个配置域也可以直接写可访问的主机名，如把“192.168.0.0/24”替换为开发板主机名“imx6ull14x14evk”，则仅该开发板能访问共享的目录。

-  rw: 表示客户机的权限，rw表示可读写，具体的权限还受文件系统的rwx及用户身份影响。

-  sync：资料同步写入到内存与硬盘中。

-  anonuid=998：将客户机上的用户映射成指定的本地用户ID的用户，此处998是开发主机embedfire用户的uid，此处请根据具体的主机用户uid进行配置。

-  anongid=998： 将客户机上的用户映射成属于指定的本地用户组ID，此处998是开发主机embedfire用户组gid，此处请根据具体的主机用户组gid进行配置。

-  no_subtree_check：不检查子目录权限，默认配置。

本配置中的anonuid和anongid把客户机的用户映射成本地uid/gid为998的用户，即开发主机的embedfire，那么当在开发板上使用与开发主机不同的用户访问NFS共享目录时，都会具有embedfire的权限，方便互相访问。如开发板上的root用户创建文件，在开发主机上会被认为是embe
dfire创建的；在开发主机上仅embedfire能读写的文件，在开发板上的root或其它用户也能对该文件进行读写（被当成了embedfire）。当然这并不是一种安全的访问设置方式，不过这用在我们的开发中是非常方便的配置。

创建共享目录
''''''

为了确保共享的配置有效，我们还需要创建共享的目录，在本例子中共享的目录为/home/embedfire/workdir。

本例子中创建的目录命令如下，请根据自己的实验环境设置要共享的目录：

#以下命令在主机上运行

mkdir ~/workdir

更新exports配置
'''''''''''

修改完/etc/exports文件并保存后，可使用exportfs命令更新配置：

#以下命令在主机上运行

sudo exportfs -arv

该命令的参数说明如下：

-  -a：全部mount或umount文件/etc/exports中的内容。

-  -r：重新mount文件/etc/exports中的共享内容。

-  -u：umount目录。

-  -v：在exportfs的时候，将详细的信息输出到屏幕上。

..

   若配置正常，该命令执行后会列出共享的目录项，本示例的执行结果见图 15‑9。

   |mountn009|

图 15‑9 exportfs 更新配置

查看NFS共享情况
'''''''''

使用showmount –e 可查看当前NFS服务器的加载情况，具体见图 15‑10。

#以下命令在主机上运行

showmount -e

|mountn010|

图 15‑10 显示当前的NFS加载情况

在开发板上挂载NFS目录
^^^^^^^^^^^^

开发主机共享目录后，我们还需要在开发板端挂载该共享目录。

所谓挂载，就是把Linux系统本地的某个目录与存储设备建立连接，以使系统能通过目录访问存储设备上的资源，这个存储设备可以是磁盘、光盘、SD卡以及此处使用的网络共享的目录。在Linux系统下，可使用mount命令挂载目录。

下面以实例进行说明，以下操作均在开发板的终端上进行，默认用户为root。

创建挂载点
'''''

挂载时需要本地存在该目录，本例子把开发板挂载在/home/root/mountnfs目录下，首先需要创建该目录：

#以下命令在开发板上运行

mkdir /home/root/mountnfs

临时挂载NFS文件系统
'''''''''''

接下来使用mount命令进行挂载：

#以下命令在开发板上运行

#需要把下面的dev设置为前面自己的主机名

mount -o vers=4 dev:/home/embedfire/workdir /home/root/mountnfs

以上命令使用的各个参数如下：

-  -o vers=4：表示使用NFS文件系统第4版本，若不注明版本可能会提示参数错误。

-  dev：目标主机的名字，此处也可以直接使用目标主机的IP地址，如本例子的目标机器IP为192.168.0.219。

-  /home/embedfire/workdir：远端的主机共享目录。

-  /home/root/mountnfs：本地挂载点，即要把远端共享的目录映射到本地的哪个目录。

若挂载成功，终端不会有输出，Linux下执行命令后若没有输出通常就是表示执行成功。

使用这种方式挂载目录只是临时的，开发板在重启后需要重新挂载该NFS目录才能访问。

使用脚本进行测试
''''''''

挂载成功后，可以在开发机上把前面编写的hello world脚本复制至共享目录，然后在开发板上的终端直接访问该脚本文件并执行。在后面我们都会采用这样的形式进行开发，即在开发主机编写代码，复制至共享目录，然后在开发板上测试。

取消挂载
''''

当客户机在网络上无法找到NFS共享的目录时，如开发主机关机时，在NFS的客户机的终端常常会输出一些提示，或在使用ls命令查看共享目录会导致长时间等待，这时可以对目录使用umount命令取消挂载，示例如下：

#以下命令在开发板上运行

umount /home/root/mountnfs

使用该命令时以要取消挂载的目录作为参数即可，没有输出表示执行正常。如果在当前挂载的目录进行umount操作，会提示“device is busy”。建议取消挂载时，先切换到家目录”~”，在进行umount操作。

.. |mountn002| image:: media/mountn002.png
   :width: 3.00117in
   :height: 3.32639in
.. |mountn003| image:: media/mountn003.png
   :width: 5.76806in
   :height: 4.09973in
.. |mountn004| image:: media/mountn004.png
   :width: 4.88794in
   :height: 2.65972in
.. |mountn005| image:: media/mountn005.jpg
   :width: 5.76806in
   :height: 1.51042in
.. |mountn006| image:: media/mountn006.jpg
   :width: 5.76806in
   :height: 2.80417in
.. |mountn007| image:: media/mountn007.jpg
   :width: 5.76806in
   :height: 4.22361in
.. |mountn008| image:: media/mountn008.jpg
   :width: 4.675in
   :height: 0.31667in
.. |mountn009| image:: media/mountn009.jpg
   :width: 4.425in
   :height: 0.30833in
.. |mountn010| image:: media/mountn010.jpg
   :width: 3.15833in
   :height: 0.475in
