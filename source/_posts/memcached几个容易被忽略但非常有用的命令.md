title: memcached几个容易被忽略但非常有用的命令
date: 2014-07-14 11:25:29
tags: memcached
---
# 一、CAS和GETS
　　Memcached从1.2.4版本新增CAS(Check and Set)协议，用于处理同一个ITEM（key-value）被多个session更新修改时的数据一致性问题。
　　假设有两个session（A、B），要同时修改某个key的值x，并且修改的数据是基于原来数据的一个计算的结果。session A和B同时得到了key的值x，session A经过计算后应该更新为y，session B经过计算后也更新为y，但是session B其实期望的是拿到y值，并将其计算为Z后更新。造成这个问题的原因就是缺少一个类似于MySQL的事务，用于数据并发修改的一致性问题。
<!--more-->
　　CAS命令着眼于解决一定的并发修改问题，引入了乐观锁的概念。在试图修改某个KEY的值之前，先由GETS命令得到某个KEY的值及其版本号，完成数据操作更新数据时，使用CAS谨慎更新，比较版本号是否与本地的版本号一致，否则放弃此次的修改。
　　Memcached在默认开启CAS协议后，每个key关联有一个64-bit长度的long型惟一数值，表示该key对应value的版本号。这个数值由Memcached server产生，从1开始，且同一Memcached server不会重复。在两种情况下这个版本数值会加1：
1. 新增一个key-value对。
2. 对某已有的key对应的value值更新成功。删除item版本值不会减小。

　　首先为了获得KEY值的版本号，引入了GETS命令，可以发现GETS命令比GET命令多返回了一个数字，这个数字就是上面我们提到的KEY值的版本号。        然后是CAS命令，与SET命令类似，只是在最后面多了一个参数，也就是key值得版本号，只有版本号与存储的数据版本号一致时，更新操作才会生效。
```bash
set a 0 0 1
x
STORED

gets a
VALUE a 0 1 1  //最后一位就是a的版本号
x
END

set a 0 0 1
y
STORED

gets a
VALUE a 0 1 2    //新增或修改之后，版本号都会增加
y

cas a 0 0 1 1
//cas更新的时候，不同于set操作，最后一位要指定版本号，当本地版本号和服务器版本号相同的时候，更新才会有效
x
EXISTS

cas a 0 0 1 2
y
STORED
```
# 二、stats items和stats cachedump
　　你曾经是否也有想知道memcached里面都存了哪些数据的需求，你是否也曾经在寻找一个方法能像redis一样可以遍历memcached所有的key
　　其实就是应用我们平时经常用到的stats方法。stats方法不仅能获得memcached的一个概况信息，如果加上子命令还可以获得更多的更加详细的信息。如slabs，items等。
　　stats items命令，可以获得memcached内item组的相关信息，如分组内item的数量，踢掉次数等。后面运行cachedump命令的时候会用到这个命令的返回信息（item组序号）。
　　stats cachedump命令，可以将某个slab中的items全部dump出来，第一个参数就是上面stats items返回的items组号，也就是slab的编号，第二个参数为一次显示多少个item信息，如果为0就显示这个item组的全部items，第二列就是key。
```Bash
stats items
STAT items:1:number 3
STAT items:1:age 943
STAT items:1:evicted 0
STAT items:1:evicted_nonzero 0
STAT items:1:evicted_time 0
STAT items:1:outofmemory 0
STAT items:1:tailrepairs 0
STAT items:1:reclaimed 0
STAT items:1:expired_unfetched 0
STAT items:1:evicted_unfetched 0
END

stats cachedump 1 0
ITEM c [5 b; 1405246917 s]
ITEM b [1 b; 1405246917 s]
ITEM a [1 b; 1405246917 s]
END
```
　　虽然应用这两个命令并不能一次显示全部的key，但是如果我们自己根据stats items的返回值自己做一次迭代，或者仅仅是为了手动做几个item的抽样，那么就能很好的帮助我们了解memcached中数据的情况。
