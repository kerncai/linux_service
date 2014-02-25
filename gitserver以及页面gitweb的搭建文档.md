gitserver以及页面gitweb的搭建文档
=====================


#git
Git是一个开源的分布式版本控制系统，用以有效、高速的处理从很小到非常大的项目版本管理。目前比较火的就是github了，但是，一般公司内部使用的文档保存在github上面是不安全的，这个时候，就需要自己搭建git server实现内部的版本控制

----------
**安装git**
搭建过程我使用了两台机器，一台ubuntu、一台centos，系统不会影响到后续的搭建。centos的机器作为gitserver，ubuntu作为client使用。至于两天机器之间的通讯，使用的ssh的key登录。

ubuntu端：

> sudo apt-get install git

centos端：

> sudo yum install git

检查版本可以使用一下命令:

> git --version
> git version 1.7.1

**2、server端的配置**
创建git的专有用户

> sudo useradd git -d /data/git
> 将git的家目录放在了data下，这个根据个人考虑，我这里data的目录比较大

切换到git账户

> su - git

安装gitolite
这里需要用到client端的key，主要是用来管理以后的git库和用户

先要在ubuntu上生成密钥对admin和admin.pub

> ssh-keygen -f ~/.ssh/admin

将生成的密钥对拷贝到gitserver，路径为/data/git/，一定要保证.ssh/authorized_keys不存在或者为空，因为后续要生成。

> git clone git://github.com/sitaramc/gitolite  
> mkdir -p ~/bin   
> gitolite/install -to ~/bin  
> gitolite setup -pk admin.pub

如果出现error：Can't locate Time/HiRes.pm in @INC (@INC contains: /home/git/gitolite/src/lib /usr/local/lib/perl5 /usr/local/share/perl5 /usr/lib/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib/perl5 /usr/share/perl5 .) at /home/git/gitolite/src/lib/Gitolite/Common.pm line 76.
BEGIN failed--compilation aborted at /home/git/gitolite/src/lib/Gitolite/Common.pm line 76.
Compilation failed in require at gitolite/install line 15.
BEGIN failed--compilation aborted at gitolite/install line 15.

说明缺少perl需要的软件Time::HiRes，安装该软件包后，重新执行上面的命令：

> sudo yum install perl-Time-HiRes  

**3、添加新库以及访问权限控制** 
在ubuntu上面克隆admin的库

> git clone git@serverip:gitolite-admin

克隆完成后，在./gitolite-admin目录下需要关注两个子目录：conf和keydir。conf是gitolite的权限配置文件夹，keydir用于放置所有用户的公钥。所以，现在可以将kerncai的公钥放入文件夹keydir。然后编辑conf/gitolite.conf文件，在文件末尾添加新的repo：

> repo kerncai
> > RW+     =   kerncai  
> >   R     =   @all

repo kerncai 表示创建一个新库，名为kerncai
RW+     =   kerncai  表示，kerncai的用户有读写的权限
R       =   @all     表示，除kerncai以外的用户只有可读权限

更改完成之后，提交到server端,完成了用户以及其库的添加

> cd gitolite-admin/
> git add '*'
> git commit -m 'add user kerncai'
> git push -u origin master

**4、gitweb的配置 (gitserver端的配置)**
git在终端访问还是有那么一点不直观的，我们可以在页面进行访问git库以及简单的操作；这就要使用到gitweb了。

> yum install gitweb

安装完成之后打开/etc/gitweb.conf的配置文件，按照注释的格式更改，并指向git库

> our $projectroot = "/data/git/repositories";
> our @git_base_url_list = qw(git://serverip
>>ssh://git.corp.howbuy.com/var/lib/git);

编辑apache的/etc/httpd/conf.d/git.conf,使用的8090端口

>< VirtualHost *:8090>
    ServerName git.corp.howbuy.com
    DocumentRoot /var/www/git
    < Directory /var/www/git>
        Allow from all
        AllowOverride all
        Order allow,deny
        Options +ExecCGI
        AddHandler cgi-script .cgi
        DirectoryIndex gitweb.cgi
        SetEnv GITWEB_CONFIG /etc/gitweb.conf
        Dav On
        RewriteEngine Off
    < /Directory>
< /VirtualHost>

修改git库的权限，否则有可能出现404 no projects found的错误

> sudo chmod 775 /home/git/repositories  
> sudo chmod 775 /home/git  
> sudo /etc/init.d/httpd restart 

apache重启的时候要注意下当前apache有没有其他业务，最好还是用重置的命令
这里遇到过一个问题，就是用户git add 数据到库的时候，会造成其库下的文件权限不对，导致gitweb渲染不出来其库的相关信息，比较笨的方法，起个crontab 每份中去chmod 755 所有库的权限。
这样就搭建完成了，可以访问http:serverip:8090在页面查看git的相关信息了

注：如果在完成上述操作后，仍然显示404 no project found，那很可能又是SELinux惹的麻烦，尝试更改selinux的状态为permissive后再刷新页面试试。

**5、用户使用git**

> git clone git@serverip:user

serverip是gitserver的ip或者apache配置的域名，user为之前为用户创建的库

git基本命令：

**git add** 是将当前更改或者新增的文件加入到Git的索引中，加入到Git的索引中就表示记入了版本历史中，这也是提交之前所需要执行的一步，例如'git add app/model/user.rb'就会增加app/model/user.rb文件到Git的索引中，该功能类似于SVN的add

**git commit**提交当前工作空间的修改内容，类似于SVN的commit命令，例如'git commit -m story #3, add usermodel'，提交的时候必须用-m来输入一条提交信息，该功能类似于SVN的commit

**git push -u origin master** 将本地commit的代码更新到远程版本库中，例如'git push origin'就会将本地的代码更新到名为orgin的远程版本master库中

**git rm -rf** 根据克隆下来的目录进行删除，会直接删除掉远程库当中的东西，慎用！

**git pull** 拉取远程库中的更新内容，一般在执行git add之前，最好先执行下拉取的动作 。

**git log**  查看历史日志，该功能类似于SVN的log

基本命令就这么多，特性的比如控制分支、回滚等请自行百度或者谷歌什么的