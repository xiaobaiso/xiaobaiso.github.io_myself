---
layout:     post
title:      关于树莓派的基本配置
category: project
description: 关于树莓派的基本配置
---
##有关于系统和分区
用的是mac下命令烧写系统ubuntu，官方推荐pi2用这个比较好，我就下了官方最新版本的ubuntu mate版本
，关于分区是这样的，安装好是剩下了11G的空间未分配，好多网上介绍的时候都只是对自己做个记录，看的云里雾里，我这么操作：把sb卡拿出来到读卡器里面去放到linux下用fdisk看，比如说在linux下是sdc 那么一看就有两个分区，一个是swap，还有一个是主分区，把主分区删除了，记住start块地址，然后加一个分区，命令：按n（加分区），选择P (主要)，於分区2选择2，第一空格输入原来分区2的开始位置，最后的空格输入默认值，然后w 保存，sudo resize2fs /dev/sdc2 这样就ok了，吧卡插到树莓派中，分区已经扩展了。


##制作samba
首先先apt-get install samba下来，然后这个配置文件已经备份了，原版的也有，目前是不需要确认身份的。这个配置一下就好了，服务命令：service smbd restart，具体配置文件稍后放出网盘，顶替原有的就可以。

##使用vnc
这个也简单，很好配置的。windows下使用VNCViewer，ubuntu这一边的话，安装上apt-get install vcn4server 就好了。 然后运行vncserver :1 这样端口号就是5901，然后启动就可以了。如果要连接桌面，vim /root/.vnc/xstartup    注意：vnc也是可以结合花生壳的
将其内容设置为：


 #!/bin/sh 
 # Uncomment the following two lines fornormal desktop: (去掉以下两行的#就可以允许使用桌面了)
unset SESSION_MANAGER

exec /etc/X11/xinit/xinitrc

[ -x /etc/vnc/xstartup ] && exec/etc/vnc/xstartup

[ -r $HOME/.Xresources ] && xrdb$HOME/.Xresources

xsetroot -solid grey

vncconfig -iconic &

 #xterm -geometry 80x24+10+10 -ls -title"$VNCDESKTOP Desktop" &

 #twm & ---把这两行注释掉，加上

gnome-session &


这样就好了。
这里也有配置文件，也都好办，拿我的配置文件一顶替就好了。
这是本地的VCN显示：

![vnc](http://7xr0og.com1.z0.glb.clouddn.com/Screenshot-4.png =300x230 "这是vnc的图标")![vnc](http://7xr0og.com1.z0.glb.clouddn.com/Screenshot-3.png =300x230 "这是vnc的图标")
这是远程的VCN显示，走了外网：

![vnc](http://7xr0og.com1.z0.glb.clouddn.com/Screenshot.png =300x230 "这是vnc的图标")![vnc](http://7xr0og.com1.z0.glb.clouddn.com/Screenshotdesktop.png =300x230 "这是vnc的图标")

看到vnc环境下显示效果，果断放弃掉用来构建无线家庭媒体中心的想法，这玩意还真不行。

走外网的vnc虽然显示效果不佳，但是用来处理下邮件，写个word还是没有什么问题，带宽占用不是很大（vnc有显示参数，可以调节）。
##制作花生壳的远程ssh
话说这个搞了一下，容我想想：
因为22端口经常被运营商封掉，所以要改成新端口共享出来。

首先8块钱买一个花生壳的内网版，然后在官网上下载树莓派版本的压缩包，解压就可以用。然后就是这样的http://service.oray.com/question/2680.html，这里面说的很详细，但是他妈的花生壳有个bug，就是你要在之前先是内网版的账号，然后设备再与这个账号绑定也变成了内网版，如果像我这样先申请了免费的公网版，然后再绑定设备，再升级为内网版，设备还他妈的是公网版。要都是内网版才能用。

其次要去绑定花生壳的端口，这个简单，按照教程来，公开端口就是了，注意server上的防火墙要打开规则，这个网上已经将这些东西讲的很清楚了，配置好后可以远程连接ssh了。

最后发现这个东西真亏，急急忙忙搞掉了8块钱，__结果才只能绑定两个端口__，我还以为无限的呢，我又受不了这种的，找了下发现有更好用的反向代理，就可以公开端口了，这真是简直了，还不要钱，我把我之前翻。墙的服务器拿出来代理了一下，就好用了，就是慢了点。反向代理的教程网上更是一把一把的，__注意，把5901端口共享出来就可以开启远程桌面了__。就是网不怎么样。

另外要绑定域名的可以考虑namecheap上买，价钱还行，这里是绑定A地址的办法：https://www.namecheap.com/support/knowledgebase/article.aspx/319/78/

##制作家庭下载服务器
想法很简单，就是让他实现类似于小米路由器内置的下载功能，而这个功耗不大，完成可以承载24小时不间断的下载任务，找了下，linux下载的工具也是有不少，但是一用就发现完全没资源，这就很纠结了，最后还是决定用迅雷下载。
linux下有一个迅雷的固件，封装安装后就好了，参考办法：http://www.chinagtd.com/archives/xunleipi.html
__注意，这个东西启动居然要root权限，还不开源。。。。我忍你很久了。。。__



{% if site.uyan %}
    <!-- UY BEGIN -->
    <div id="uyan_frame"></div>
    <script type="text/javascript" src="http://v2.uyan.cc/code/uyan.js?uid=2066802"></script>
    <!-- UY END -->
{% endif %}




