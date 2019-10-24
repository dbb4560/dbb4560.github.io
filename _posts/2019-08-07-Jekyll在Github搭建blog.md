---
layout:     post
title:      Jekyll在Github搭建blog
subtitle:   搭建blog 
date:       2019-08-07
header-img: img/post-first-blog-web.jpg
catalog: true
tags:
    - Jekyll 
    - blog
    - picgo
    
---
##背景
1. 最近想试试用Jekyll在Github搭建blog。选取网站模板，修改域名等等这些网上都有很详细的教程了，
 
<https://github.com/Huxpro/huxpro.github.io>

直接folk，安装说明在readme.md。Jekyll 在github 也有一大堆site 案列 ：

< https://github.com/jekyll/jekyll/wiki/sites >

本文主要记录在Windows本地安装jekyll环境的过程，遇到的问题及如何解决的。


##安装环境

1. 安装Ruby
在Windows上使用RubyInstaller安装比较方便，去Ruby官网下载最新版本的RubyInstaller。注意32位和64位版本的区分。

注意：勾选添加到PATH选项，以便在命令行中使用。

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191019220626.png)

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191019220747.png)

这里需要勾选安装msys2，后面安装gem和jekyll时会用到：

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191019220919.png)


本来想把软件包上传到github去的，老是上传失败！网速很差！
只好放在百度云！

rubyinstaller-devkit-2.6.3-1-x64.exe 安装包：

<https://pan.baidu.com/s/1cYILP_pFil_l9VkO23hyuA&shfl=sharepset>

2. 安装RubyGems

rubygems-3.0.4.zip安装包：

<https://pan.baidu.com/s/1WVYr0khAoOYkMqJf6VC5ew&shfl=sharepset>

Windows中下载ZIP格式比较方便，下载后解压到任意路径。打开Windows的cmd界面，输入命令：

$ cd {unzip-path} // unzip-path表示解压的路径，比如我的就是

```
Cd  d\Jekyll\RubyGems\rubygems-3.0.4 
$ ruby setup.rb
```

3. 安装Jekyll

在cmd中输入:

`$ gem install Jekyll`

此时报错

```
ERROR:  Could not find a valid gem 'jekll' (>= 0) in any repository
ERROR:  Possible alternatives: Jekyll
```

从字面意思可以猜测出应该是网络问题，在网上searching了一下还在Jekyll的Github主页上添加了一个issue（结果被大神告知这不是Jekyll的问题是ruby的问题啊，捂脸逃～）。最后找到解决办法，将gem的镜像源改到淘宝镜像。

4. 删除默认的gem源
`gem sources --remove http://rubygems.org/`

增加taobao作为gem源

`gem sources -a http://ruby.taobao.org/`
去 https://ruby.taobao.org/ 网站查看 
发现这句话

中国 ruby 镜像交由社区打理，本站请求已重定向到 

`http://gems.ruby-china.org/`

把镜像改成上面这个链接就可以了。
如何使用？

```
$ gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
$ gem sources -l
*** CURRENT SOURCES ***

```

https://gems.ruby-china.org
 "# 请确保只有 gems.ruby-china.org "
$ gem install rails


gem sources -a http://gems.ruby-china.org/
原因是 ruby-china 更换了域名

`https://gems.ruby-china.com`

`gem sources -a https://gems.ruby-china.com`

5. 查看当前的gem源
`gem sources`

6. 清空源缓存
`gem sources -c`

7. 更新源缓存

`gem sources -u`
这个时间比较长，可能几分钟

于是问题解决。
要注意的是，ruby版本最好在2.0以上，使用gem -v查看版本，gem update --system升级版本。另外不切换ruby镜像到淘宝的话，连升级ruby版本都是不行的。

8. 安装jekyll-paginate
在cmd中输入：
`$ gem install jekyll-paginate`

9. 验证安装完成
在cmd中输入：
`$ jekyll -v`
输出版本说明安装完成（我的版本为3.7.3）

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191019234729.png)


##遇到问题及解决
1. gem install jekyll时报错，而且还是乱码！
C:\User>gem install jekyll
Temporarily enhancing PATH for MSYS/MINGW...
Building native extensions. This could take a while...
ERROR:  Error installing jekyll:
    ERROR: Failed to build gem native extension.

    current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
C:/Ruby25-x64/bin/ruby.exe -r ./siteconf20180308-3672-ueo7ea.rb extconf.rb
creating Makefile

current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
 make "DESTDIR=" clean
 'make' �����ڲ����ⲿ���Ҳ���ǿ����еĳ���
���������ļ���

current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
make "DESTDIR="
'make' �����ڲ����ⲿ���Ҳ���ǿ����еĳ���
 ���������ļ���

 make failed, exit code 1

Gem files will remain installed in C:/Ruby24-x64/bin/ruby_builtin_dlls/Ruby24-x6
4/lib/ruby/gems/2.4.0/gems/http_parser.rb-0.6.0 for inspection.
Results logged to C:/Ruby24-x64/bin/ruby_builtin_dlls/Ruby24-x64/lib/ruby/gems/2
.4.0/extensions/x64-mingw32/2.4.0/http_parser.rb-0.6.0/gem_make.out

参考oneclick/rubyinstaller2的issue #98。
首先cmd中输入：
$ chcp 850

2. 切换编码之后安装：
`$ gem install jekyll`

下面是报错：
Temporarily enhancing PATH for MSYS/MINGW...
Building native extensions. This could take a while...
ERROR:  Error installing jekyll:
        ERROR: Failed to build gem native extension.

    current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
C:/Ruby25-x64/bin/ruby.exe -r ./siteconf20180308-3672-ueo7ea.rb extconf.rb
creating Makefile

current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
make "DESTDIR=" clean
'make' 不是内部或外部命令，也不是可运行的程序或批处理文件。

current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
make "DESTDIR="
'make' 不是内部或外部命令，也不是可运行的程序或批处理文件。

make failed, exit code 1

Gem files will remain installed in C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_p
arser.rb-0.6.0 for inspection.
Results logged to C:/Ruby25-x64/lib/ruby/gems/2.5.0/extensions/x64-mingw32/2.5.0
/http_parser.rb-0.6.0/gem_make.out

原来是没有make指令，上面的步骤其实已经安装了msys2，所以不会出现问题。对于没有勾选的童鞋，可以在cmd中输入下面命令来安装：
$ ridk install

安装完成之后再次安装jekyll和jekyll-paginate就ok了。

3. jekyll serve启动报错
Incremental build: disabled. Enable with --incremental
      Generating...
jekyll 3.7.3 | Error:  Permission denied @ rb_sysopen - C:/Users/username/NTUSER.DAT

这是因为jekyll默认使用4000端口，而4000是FoxitProtect（福昕阅读器的一个服务）的默认端口。网上有教程说kill掉FoxitProtect的进程，但是我觉得首先这个比较麻烦，其次重启计算机时FoxitProtect是默认启动的，除非关闭这个服务，这样又可能带来其他问题。所以最简单的办法还是指定端口：
$ jekyll serve -P 5555

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191019234729.png)

即可在本地localhost:5555进行编译预览!

![](https://raw.githubusercontent.com/dbb4560/StorePicturebed/master/wirtePicture/20191019234819.png)