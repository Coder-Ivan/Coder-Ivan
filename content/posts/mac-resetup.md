+++
title = "Mac重装+初始化配置"
date = "2016-10-14"
tags = [
    "Mac",
    "小技巧",
]
categories = [
    "IT杂文",
]
+++

![MacBook Pro](http://upload-images.jianshu.io/upload_images/1587104-57cc6195b19136a2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> MacBook使用时间长了之后难免会有很多杂七杂八的文件，存储空间严重不足，我一狠心，就趁着升级到10.12(macOS Sierra)系统的机会，把MacBook直接全部抹掉重装了。重装还不算麻烦，但是麻烦的就是在一清二白的系统上重新搭建起各种环境……

## 一、重装系统
据我所知的重装Mac系统的方法有三种：
- 通过Time Machine恢复
- 在线重装
- 自制U盘重装

具体的操作方式可以在网上查，本人懒一点，再加上网速还可以，就用了最省事的在线重装。
1. 关闭MacBook，然后按住Commend + R不放，打开电源，直到出现苹果的标志，然后放开Commend + R。
2. 这时候进入了OS X实用工具的界面，分别有“从Time Machine备份进行恢复“、”更新安装OS X“、”获得在线帮助“、”磁盘工具“这四类。
3. 先进入”磁盘工具“，这时候左边能看到两个盘。上面的一个盘是自己的主存储盘，下属一个Macintosh HD子盘；下面的一个大概有2G左右的盘，就是类似于Windows PE系统的一个空间，不要去动它。选择Macintosh HD子盘，抹掉。
4. 返回进入”更新安装OS X“选项，选择Macintosh HD子盘，进行在线安装。我安装的时候显示还有5分钟，可是我足足用了半个多小时才把这5分钟给跑完(吐槽一下苹果弱智的时间算法)……然后就自动进行系统安装了。
5. 剩下的就是配置Apple ID，以便于以后还原数据。

## 二、配置系统
首先我就安装了Xcode，吃饭的家伙，必须要先保证有。然后安装了搜狗输入法、Clean My Mac、 有道词典、腾讯QQ、 为知笔记、 网易云音乐、 Dash、 Source Tree、 iStat Menus、 Snip、 The Unarchiver、 XMind这几个软件，其他的以后想起来再安装。
> 值得注意的一点是，有些软件从App Store下载和从官方网站下载是完全不同的，App Store会对某些功能进行限制。比如；有道词典的划词和屏幕取词功能、网易云音乐的支持键盘顶部音乐控制功能，Snip的滚动截屏功能等。

#### 升级ruby
1. 从网上看到有因为ruby版本低而安装Cocoapods失败的，所以本着谨慎的原则，要先更新ruby。
2. 更新ruby需要安装RVM，所以要先安装RVM。
``` ruby
curl -L https://get.rvm.io | bash -s stable
```
指定位置
``` ruby
source ~/.rvm/scripts/rvm
```
然后通过命令
``` ruby
rvm -v
```
查到我的RVM版本是1.27.0。
3. 通过命令
``` ruby
ruby -v
```
查到我的ruby版本是2.0.0p648。
然后通过命令
``` ruby
rvm list known
```
查到最新的版本是2.3.0。
通过命令
``` ruby
rvm install 2.3.0
```
来安装在最新版本的ruby，但是它提示你没有安装Homebrew,输入路径进行安装，按回车键选择默认位置，然后一路回车，安装brew完毕，然后终端自动继续安装ruby 2.3.0，安装成功。

#### 安装Cocoapods
1. 通过命令
``` ruby
gem sources -l
```
查到系统默认的gem的源是https://rubygems.org/ ，这个在国内有被墙的危险，~~所以要改为国内的淘宝源。~~`因为近期淘宝的ruby镜像网站已经放弃维护，RubyGems 镜像的管理工作以后将交由 [Ruby China](https://gems.ruby-china.com/) 负责，所以之前的淘宝镜像已经不能使用了。同时，Ruby China的镜像网址也由https://gems.ruby-china.org/替换为https://gems.ruby-china.com/，请大家注意及时更改！`
``` ruby
gem sources --remove https://rubygems.org/
gem sources -a https://gems.ruby-china.com/
```
然后对gem进行更新
``` ruby
sudo gem update --system
```
更新后的版本为2.6.6。
2. 终于到了要安装Cocoapods的时候了，由于Cocoapods的库文件太大，下载时间太长，可以从一台已经下载好了的电脑的~/目录里面把.cocoapods/文件夹拷贝到对应的位置，然后再下载库文件就省很多时间。
但是系统默认是看不到隐藏文件夹的，可以通过命令
``` ruby
defaults write com.apple.finder AppleShowAllFiles -bool true
```
来使隐藏文件失效，必须重启finer才能生效。
6. 通过命令
``` ruby
sudo gem install cocoapods
```
安装Cocoapods。

7. 把从其它电脑复制得来的.cocoapods文件放到~/路径里面，然后终端运行
``` ruby
pod setup
```
在很短的时间内就安装好了。

#### 安装LLDB命令插件chisel
1. 在控制台输入以下命令
``` ruby
brew install chisel
```
2. 根据提示，在~/路径查找.lldbinit文件，发现没有，就新建一个该文件，并插入提示的文字。
``` ruby
cd ~
touch .lldbinit
vi .lldbinit
```
在文件中粘贴
command script import /usr/local/opt/chisel/libexec/fblldb.py
这段话，并进行保存。

#### Safari插件配置
1. 点击左上角Safari按钮 -> Safari扩展... ，进入了一个插件库。先安装了Evernote Web Clipper，它可以很方便地把网页中的内容保存到自己的印象笔记账户里面去。
2. 接着安装了Adblock Plus，拦截广告还是这个管用！
3. 因为苹果系统对Flash的不兼容，导致网页上的大部分视频都不能正常观看，看一小会儿就会发热很严重。在百度上搜索“妈妈再也不用担心我的macbook发烫了计划”，它能自动把网页上的Flash格式视频转换成HTML5格式的视频。把这个插件安装上，在进入优酷等对应网页的时候，就可以放心看视频了。

#### 安全性和隐私里面添加“任何来源”
在更新到10.12的系统后，发现在系统偏好设置->安全性和隐私->通用里面去掉了“任何来源“选项，导致没有签名的应用没办法安装，只允许AppStore和被认可的开发者的应用可以安装。
在终端输入
``` ruby
sudo spctl --master-disable
```
命令，就可以重新看到”任何来源“了。

就先配置到这里，差不多可以用了，有啥添加的再看具体情况就可以了。