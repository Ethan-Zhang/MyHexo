title: 《Tornado异步框架及协程相关》讲义
date: 2015-10-16 08:04:59
tags: [Tornado, 协程, 异步]
---
>本文是我在公司做的某次培训活动的一个讲义，放到博客里供大家参考

![ppt1](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.001.jpeg)
![ppt4](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.004.jpeg)
![ppt5](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.005.jpeg)
<!--more-->
![ppt6](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.006.jpeg)
![ppt7](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.007.jpeg)
![ppt8](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.008.jpeg)
![ppt9](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.009.jpeg)
![ppt10](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.010.jpeg)
![ppt11](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.011.jpeg)
![ppt12](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.012.jpeg)
![ppt13](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.013.jpeg)
![ppt14](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.014.jpeg)
![ppt15](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.015.jpeg)
![ppt16](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.016.jpeg)
![ppt17](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.017.jpeg)
![ppt18](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.018.jpeg)
![ppt19](http://7xpwqp.com1.z0.glb.clouddn.com/Tornado%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E5%8F%8A%E5%8D%8F%E7%A8%8B%E7%9B%B8%E5%85%B3.019.jpeg)
