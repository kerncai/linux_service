ubuntu和windows7共享文件
=====================


#ubuntu
设置共享目录文件夹给Windows（win7系统）用户，这样在windows下可以打开Ubuntu里面共享的目录文件夹，这样的话同时使用ubuntu和win7，就不会因为传送文件到烦恼了。

----------
先准备需要共享的目录

> cd /home/ 
> mkdir public
> chmod -R 777 public

要在Ubuntu下给windows系统共享文件夹、目录，首先要在Ubuntu系统中安装一个软件：samba，在Ubuntu系统中打开终端命令行窗口，然后输入下面的命令就可以安装samba了：

> sudo apt-get install samba
> sudo apt-get install smbfs

配置smb.conf：

vim /etc/samba/smb.conf #默认路径

> [global] 这里只需要更改以下，使用用户登录 安全性考虑 
> security = user

[printers] 这里配置的是打印机方面的共享，无视掉～

继续配置smb.conf

vim /etc/samba/smb.conf

> [public]
>        comment = Bridge Ubuntu and windows communication
>        path = /home/kerncai/public
>        public = yes
>        writeable = yes
>        valid users = kerncai
>        create mask = 0777
>        directory mask = 0777
>        force user = kerncai
>        force group = kerncai
>        available = yes
>        browseable = yes


配置文件很简单，指定了读写权限，还有共享文件的用户与组；建议最好是ubuntu当前的登录用户，方便操作～

配置访问共享目录的用户和密码

> sudo smbpasswd -a kerncai

重启下smbd

> sudo /etc/init.d/smbd restart

至此，ubuntu上面的配置完成

#win7
双击打开我的电脑
在地址栏输入之前ubuntu上面的ip地址

> \\\192.168.100.100

可以看到之前在ubuntu下创建的public目录了 7