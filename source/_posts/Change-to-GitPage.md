title: 博客搬新家啦
date: 2016-01-07 19:12:32
tags: [感悟]
---
&emsp;&emsp;博客搬了新家之后终于又和大家见面了。时逝如隙，从写第一篇博文算起来已经有9年了，说来也是惭愧，直至现在才有了自己的独立博客。其名为林中小灯，同时申请了域名——lightthewoods，意为照亮幽幽森林中的那盏独自闪烁着熠熠光亮的小灯。其实呢，说来说去，还是和我笔名的意义有些关联。虽然我写的东西，分享的内容，在这个时代浩瀚的信息量之中的作用微乎其微，但是我一直会在那里，默默的坚持着。

![lightthewoods](http://7xpwqp.com1.z0.glb.clouddn.com/2016-01-07-01)
&emsp;&emsp;从MySpace、新浪博客、51、一直到CSDN，用过的博客平台很多很多。看着这些博客平台起步、繁荣、火爆再到一点儿一点儿没落，也一直忍受着各类博客平台上写博的诸多不爽。这么多年来一直没放到独立博客上其实也有很多这样那样的原因。大学的那个年代，搞独立博客，感觉真的像是黑客或者技术大牛才能搞定的东西。虽然自己也是CS专业，充其量也就会用C写个if、else，倒腾倒腾死记硬背下算法书上的各种题目，以求个门专业课不要挂掉。当时世界上最好的语言PHP也才刚刚展露头角，Nginx在那时看来还是黑客大神们搞得替代apache的玩具，用MySQL的人还不如SQL Server的人数的一个零头。更别说那个年代昂贵的VPS的租用价格了。
<!--more-->
&emsp;&emsp;上面列举的这些也是造成各类博客平台能流行一段时间的原因，不管它有这样那样的问题，免费的博客平台终于能满足人们在网络生活中的一项基本需求了。后期随着web技术的日益成熟，搭建web环境变得原来越简单，网络基础设施的价格也越来越便宜，慢慢的博客平台就开始衰落了，逐步的被个人独立博客所替代。
&emsp;&emsp;说句题外话，最近新浪借助新浪博客10周年的机会配合移动端进行了一次大幅度的改版。虽然出发点不错，但是不管怎么改，技术发展所带来的趋势是没法改变的，只能顺应。所以结果而言也就不言而喻了。
&emsp;&emsp;后来身边的好多同事、朋友都慢慢的搞了自己的独立博客，尤其是技术博客，如果你放在博客平台上，哪怕是技术门类的博客平台上，也会被人先入为主的认为你的博文技术水平肯定不高。当时迟迟没动手有两个原因吧，首先确实还是自己懒，买空间、买虚拟机、配环境、wordpress、MySQL等等，在工作内容之外把这些东西统统弄一遍想想就不愿意动手，还不用说后期的其实维护上的工作。其次，我当时心里始终觉得在CSDN上有许多志同道合一直支持我的朋友，而且经过首页的推荐，或者论坛里的转发，我的文章能被更多需要的人看到，让他们看到后哪怕是得到一点儿微小的启发。因为在CSDN上的排名曾经一度进入到800名之内，还有编辑看过我写的文章后私信联系我找我出书呢（笑）。当时一直觉得可能搬到独立博客上，也许没了博客平台这个入口大家可能就都看不到了。
&emsp;&emsp;直到遇到了github page，遇到了hexo静态页面生成框架，已经再也没有不动手的理由了。因对开源的兴趣，github本身就是用的比较多，github page作为一个免费的静态页面容器提供了一切我们所需要的功能。用hexo这样的静态页面生成工具，让我们的博文先生成静态的HTML文件，再进行部署，省去了许多麻烦。再也不用担心以后老眼昏花的时候，不知道那时候php还是不是最好的语言，到时候忽然起意，想看看以前自己写的博文，还要去处理下为什么会报出db connect error的错误。
&emsp;&emsp;Hexo，因为嫌弃jallar生成页面的速度慢，由一个台湾大学生开发的框架。支持自定义的皮肤，自定义的布局，并且可以多套皮肤随时切换。再也不用受制于博客平台上对页面布局的种种限制。如果懂一些前端技术的话，真的是想怎么改就怎么改。如果不懂也没关系，网上有许多前端大神分享的漂亮的布局皮肤，真的是做到了个性化定制。
&emsp;&emsp;关于博客流量入口的问题，在全民大数据这个环境下也并不需要过多的担心。类似于今日头条、技术头条、知乎专栏这样的内容聚合平台也如雨后春笋般涌现，主要博客的内容好，完全不用担心入口流量的问题。
&emsp;&emsp;看到这里，你还有什么理由不去试一下呢~

__PS:__
&emsp;&emsp;基于github page+Hexo搭建博客的方法可以在网上找到许多优质的相关教程和帖子，这里就不赘述了，我当时参考了[hexo你的博客](http://ibruce.info/2013/11/22/hexo-your-blog/)。
&emsp;&emsp;在安装node以及hexo等相关node包的时候，最好使用国内淘宝的源，国外的源反正我是连不上的。部署的方式推荐直接用hexo的deploy脚本，直接输入hexo deploy。同时在站点配置文件里配置:
```Bash
deploy:
    type: git
    repository: https://github.com/Ethan-Zhang/Ethan-Zhang.github.io.git
    branch: master
```
&emsp;&emsp;主题用到了github上fork和start数最多的issnan大神的[next主题](https://github.com/iissnan/hexo-theme-next)，主要看重了设计的简洁和清爽。
&emsp;&emsp;图床服务使用的是七牛的免费服务，图片外联都在上面。后期看情况可能也会转成付费用户吧，毕竟好用不贵，当个高级网盘也是不错的。
&emsp;&emsp;在过程中遇到困难的，欢迎一起交流。
