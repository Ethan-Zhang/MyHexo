title: OpenStack虚拟机的用户客制化方法（User Data）
date: 2012-10-30 11:54:19
tags: [OpenStack, Nova]
---
![盛大云的用户信息定制界面](http://7xpwqp.com1.z0.glb.clouddn.com/2012-10-30-01)
　　很多成熟的公有云产品在申请虚拟机资源的时候，都可以定制客制化的用户信息，如盛大云、阿里云等可以定制虚拟机的服务器名称，用户名及密码口令等。特别是用户口令，虽然OpenStack有非常成熟的公私钥KeyPairs登陆体系，但是对于国内大多开发者还是习惯于用户名口令的登陆方式。某些场景下，服务器管理员在客户现场需要对服务器进行某些简单操作，也许没有SSH环境等，这时如果通过VNC的方式就比较简单，然而KeyPair的登陆方式还不支持VNC模式。
<!--more-->
　　在服务器管理员需要使用用户名口令的方式进行登陆的模式下，如果能让用户自己定义用户名口令可以提高一定的安全等级，增加用户对产品的认知和信任度。
　　在OpenStack中，我们通过user-data功能实现客户信息的定制，可以对虚拟机进行许多初始化的操作如设定语言区域，设定主机名称，生成SSH密钥，设定挂载节点等。
　　通过研究ubuntucloud-init和AWS的相关文档，user-data的设置可以支持有以下几种脚本语言：
* Gzip Compressed Content
content found to be gzip compressed will be uncompressed.
Theuncompressed data will then be used as if it were not
compressed.Compression of data is useful because user-data is
limited to 16384bytes1
* Mime Multi Part archive
This list of rules is applied to each part ofthis multi-part
file. Using a mime-multi part file, the user canspecify more than
one type of data. For example, both a user datascript and a
cloud-config type could be specified.
* User-Data Script
begins with: "#!" or"Content-Type: text/x-shellscript"
script will be executed at "rc.local-like" level during first boot.rc.local-like means "very late in the boot sequence"
* Include File
begins with "#include" or"Content-Type: text/x-include-url"
This content is a "include" file. The file contains a list of urls,one per line. Each of the URLs will be read, and their content willbe passed through this same set of rules. Ie, the content read fromthe URL can be gzipped, mime-multi-part, or plain text
* Cloud Config Data
begins with "#cloud-config" or"Content-Type: text/cloud-config"
This content is "cloud-config" data. See the examples for acommented example of supported config formats.
* Upstart Job
begins with "#upstart-job" or"Content-Type: text/upstart-job"
Content is placed into a file in /etc/init, and will be consumed byupstart as any other upstart job.
* Cloud Boothook
begins with "#cloud-boothook" or"Content-Type: text/cloud-boothook"
This content is "boothook" data. It is stored in a file under/var/lib/cloud and then executed immediately.
This is the earliest "hook" available. Note, that there is nomechanism provided for running only once. The boothook must takecare of this itself. It is provided with the instance id in theenvironment variable "INSTANCE_ID". This could be made use of toprovide a 'once-per-instance'
Only available in 10.10 or later (cloud-init 0.5.12 andlater)
* Part Handler
begins with "#part-handler" or"Content-Type: text/part-handler"
This is a 'part-handler'. It will be written to a file in/var/lib/cloud/data based on its filename. This must be python codethat contains a list_types method and a handle_type method.
Oncethe section is read the 'list_types' method will be called. It mustreturn a list of mime-types that this part-handler handlers.
The 'handle_type' method must be like:
```Python
def handle_part(data,ctype,filename,payload):
# data = the cloudinit object
# ctype = "__begin__", "__end__", or the mime-type of the part that is
# being handled.
# filename = the filename of the part (or a generated filename if none is
# present in mime data)# payload = the parts' content
```

　　这里主要关注User-Data Script，其使用的就是常用的shell脚本，我们只要在dashboard创建虚拟机的时候讲脚本写入user data输入框中即可。
![user-data sample](http://7xpwqp.com1.z0.glb.clouddn.com/2012-10-30-02)
　　目前还仅仅测试了ubuntu的cloudimage，非UEC镜像即使按照installturtion安装了cloud-init包也没有测试成功，还在查找原因，后面弄好了会接着给大家介绍。

***

　　非UEC镜像的问题实际上是cloud-init这个包的安装需要进行配置，详见OpenStack解决非UEC镜像的虚拟机cloud-init不工作不能自动修改主机名称不能注入userdata。
