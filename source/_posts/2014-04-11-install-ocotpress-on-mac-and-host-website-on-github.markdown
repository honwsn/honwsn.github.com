---
layout: post
title: "在Mac OSX 10.9搭建Github/Octopress博客"
date: 2014-04-11 19:03:24 +0800
comments: true
categories: github octopress mac
tags: octopress mac ruby git homebrew rvm xcode
---
  *Octopress*需要**ruby 1.9.3**，而我mac系统安装的ruby是2.1的，此外Octopress还需要bundle安装自己的依赖gem包。一开始我想着直接用mac上ruby2.1，结果在执行`gem install bundle`时，发现之下的错误：
  
> ERROR:  While executing gem ... >(Gem::FilePermissionError)
>You don't have write permissions for the /Library/Ruby/>Gems/2.1.0 directory.

这个提示的意思：
>You don't have write permissions into the /Library/Ruby/Gems/2.1.0 directory.

这是OSX自己安装的ruby版本，不允许自行安装其他的安装包。stackOverFlow有相关[说明](http://stackoverflow.com/questions/14607193/installing-gem-or-updating-rubygems-fails-with-permissions-error),所以我又回到老路上－》**安装ruby1.9.3**,下面是过程：
<!--more-->
### 一. 更新安装Xcode command line tools
xcode-tools所在目录安装的目录`/library/developer/CommandLineTools`

安装方法有两种:

- xcode-select -- install
- 直接[下载](https://developer.apple.com/downloads/index.action?name=for%20Xcode%20-#)安装也可以

###二. 安装Homebrew
[Homebrew](http://brew.sh/index_zh-cn.html ) OSX安装软件一种方法，相当于unbuntu上的apt-get,与macports的工作原理差不多。都是通过在命令行上安装软件，比较方便。Homebrew是尽量先在查找本地的依赖库，再进行下载包源码代码进行编译安装，而macports是下载所有的依赖库的源码进行编译安装。RVM在安装过中依赖一些包需要Homebrew去下载安装，

	ruby -e "$(curl -fsSL https://raw.github.com/mxcl/homebrew/go/install)"
	
输入命令下载，按照提示一路回车和输入密码即可。安装过程可以参照这篇文章[How I Install Homebrew on Mavericks Mac OS](http://www.pcdailytips.com/how-i-install-homebrew-on-mavericks-mac-os/)


###三. 安装RVM
[**RVM**](https://rvm.io)（Ruby Version Manager）,安装好了rvm就可以很方便的安装1.9.3以及切换OSX 使用什么版本的ruby。
安装命令：
	curl -L https://get.rvm.io | bash -s stable --ruby
	
rvm安装完成之后，调用

	rvm requirements
然后可能出现各种错误，大致原因是某些依赖库没有被装上，例如

> Error running 'requirements_osx_brew_libs_install autoconf automake libtool pkg-config apple-gcc42 libyaml readline libxml2 libxslt libksba openssl sqlite',

>please read /Users/allegrascrugham/.rvm/log/ruby-1.9.3-p392/1368142352_package_install_autoconf_automake_libtool_pkg-config_apple-gcc42_libyaml_readline_libxml2_libxslt_libksba_openssl_sqlite.log

这里原则是缺啥补啥，brew的用处来了，通过`brew install`安装缺失的安装包，stackOverFlow上也有相关[说明](http://stackoverflow.com/questions/16473115/error-running-requirements-osx-brew-libs-install-on-mac-10-7)。
上面可能需要反复调用`rvm reuirements`以及brew install依赖包直到没有错误。

###四. 安装ruby1.9.3
在rvm安装完成之后，就可以安装和使用1.9.3了

	rvm install 1.9.3
	rvm use 1.9.3 --default
ruby1.9.3是安装在～/.rvm/gems下, `rvm use 1.9.3`需要加上`--default`,意识是说使用ruby1.9.3 作为默认的版本。否则当前terminal退出后，再打开terminal 输入`ruby --version`，会发现还是之前系统的ruby版本2.1.0.

###五. 安装octopress预览博客

下面是本地安装过程

	git clone git://github.com/imathis/octopress.git octopress
	cd Octopress
	gem install bundler //使用gem安装bundler
	bundle install      //使用bundle为Octopress安装依赖的gem包
	rake install        //安装默认主题classic
	rake generate       //生成页面
	rake preview		 //进入预览
	
此时打开浏览器输入localhost:4000,就可以看生成的博客了。

###六. github建立版本仓库
在[github](www.github.com)注册登录之后，建立版本仓库，Reposity name必须是username.github.com，这样的格式(username可以是任何名称),这里是[参考](http://blog.csdn.net/jackystudio/article/details/16334861)

{% img center /images/2014-4/addrepos.png %}

###七. git与github打通

在github创建好版本仓库之后，得要在本地通过git访问。github可以通过ssh验证来实现这点。具体来说我们在本地通过ssh生成一对密钥，我们本地保存有私钥，把公钥给github。[参考](http://blog.csdn.net/jackystudio/article/details/12271877)

1. ssh生成密钥得方法
	
	> ssh-keygen -t rsa -C "YourEmail@example.com"
	
{% img center /images/2014-4/sshkeygen.png 700 300 %}
	
结果在～/.ssh下,id_rsa.pub就是公钥
{% img center /images/2014-4/idrsa_pub.png %}

2. 把公钥给github。
登录github－>右上角accout settings->SSH keys->add SSH key.把id_rsa.pub得内容复制粘贴，Title随意。
{% img center /images/2014-4/addshhkey.png %}

3. 客户端验证与github是否打通

> ssh -T git@github.com

我得得到如下结果:
{% img center /images/2014-4/ssht.png %}



###八. 把Octopress博客发布到github

在我们在打算把博客发布到github上，分为两个部分

1. Octopress源码，这个打算放到git@github.com:username/ username.github.com.git的**source分支**

2. Octopress 在ruby环境中生成的博客本身的html代码，个打算放到git@github.com:username/ username.github.com.git的**master分支**

完成了5.6.三两步之后，在这里要把代码上传到github，进入**Octopress目录**
运行下面得ruby命令，推送页面到github上去。

	rake setup_github_pages
	
这个命令中，需要填入版本仓库得地址，例如git@github.com:username/ username.github.com.git(git协议地址，username替换成你注册得),输入密码等。这里已完成了三件事情：

1. 在当前版本库（注意当前还是clone得Octopress的版本库）添加remote：git@github.com:username/ username.github.com.git作为origin

2. 把当前版本库分支得名称由`master` 改成`source`(这个目录将要提交到git版本库的source分支)

3. 建立_deploy目录，并在该目录下初始化一个git版本库(这里是博客页面，将要提交到git版本库的master分支)

{% img center /images/2014-4/setup_github_pages.png %}

**发布博客：**
	
	rake depoly//提交了_depoly目录下博客代码到github的master分支

**上传Octopress源码:**
	
	git add.
	git commit -m "first commit to github"
	git push origin source
	//推送octopress源码到你自己github版本库source 分支
	//，此后你可以独立修改Octorress了。
	

###九. 克隆博客
换了台机器，想把自己前面上传的源码领取下来，在这台机器上工作继续写博客，先把第7步执行下，保证由访问github版本库的权限。[参考](http://blog.csdn.net/jackystudio/article/details/16800331)
主要过程如下:

1. mkdir Octopress
2. cd Octopress
3. git init
4. git remote add origin yourrepoUrl
5. git pull origin source //拉取octorpess源码
6. git branch -m "master" "source" //把当前分支重命令
7. rake setup_github_papes 生成发布的目录_deploy
8. cd _deploy
9. git pull origin master//拉取博客的网页源码
10. cd ..
11. rake genereate//由Octopress 生成博客页面代码 到_deploy目录
12. rake preview

这里说明下，与第8步`把Octopress博客发布到github`不同的是，这里没有使用**git clone**的方法，主要原因是git clone 会把版本库的**master分支**(这里是博客网页源码)放到当前的目录(比如我们建立的Octopress文件夹)，而我们需要当前目录存放的是*Octorpress的源码*,
所以才采用git remote add 以及git pull origin source等等。这里由[stackOverflow的解释](http://stackoverflow.com/questions/4855561/difference-between-git-remote-add-and-git-clone)。　

下面是我操作过程的截图:
{% img center /images/2014-4/cloneblog1.png %}
还有一点:
{% img center /images/2014-4/cloneblog2.png %}


###十. Octopress目录分析
这个放到下一篇了。。。。先到这里





[参考] 

- [像黑客一样写博客](http://blog.csdn.net/jackystudio/article/details/16367937)
- [Install Octopress on Mac Mavericks Using RVM](http://www.pcdailytips.com/install-octopress-on-mac-mavericks-using-rvm/)
- [How to Install Xcode, Homebrew, Git, RVM, Ruby & Rails on Snow Leopard, Lion, and Mountain Lion](http://www.moncefbelyamani.com/how-to-install-xcode-homebrew-git-rvm-ruby-on-mac/)
- [Octopress Setup](http://octopress.org/docs/setup/)
- [How to Install & Configure Octopress on a Mac, and 
Host Your Static Website on Amazon S3](http://www.moncefbelyamani.com/how-to-install-and-configure-octopress-on-a-mac/)
- [How I Install Homebrew on Mavericks Mac OS](http://www.pcdailytips.com/how-i-install-homebrew-on-mavericks-mac-os/)








	






	
	



	



