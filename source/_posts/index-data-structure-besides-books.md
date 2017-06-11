title: 课本之外的索引结构——SkipList
date: 2017-04-25 13:25:08
tags: [算法]
---
上一篇中（教你手写红黑树），我们介绍了算法书上常见的自平衡树——红黑树。今天我们再介绍一种书本上不常见，但是在业内广泛使用的索引数据结构——跳跃表（SkipList）。   
跳跃表，或叫跳表，是一种随机化的数据结构，基于并联的链表，实现简单，插入、删除、查找的复杂度均为O(logN)。   
SkipList在学校教材中的算法书和课程中出现的频率不高，猜测有一下几个原因：
* 算法发明的时间较晚。SkipList的Paper首次发表于1990年，而红黑树1978年，AVL树1962年。。。可能还没来得及编进教材。
* 算法设计和实现比较简单。因为太简单了，可能老师们觉得不值得花一节课的时间去讲，学生应该去自学。。。
不同于红黑树各类旋转和增删操作的复杂，SkipList在设计思想和代码实现上都简单的超乎你的想象。但是，大道至简，是任何事物追求的终极目标。

## 从单链表到跳表
来看一个有序链表(这里H表示链表头部，T表示链表尾部，不是有效节点)：

![单链表](http://7xpwqp.com1.z0.glb.clouddn.com/%E5%8D%95%E9%93%BE%E8%A1%A8.png)

假设我们要查找7，只能老老实实地按照1->2->3->…的顺序走，忍受O(n)的效率；但如果是数组的话，可以使用二分查找达到O(lgn)。
可以在链表中使用二分查找吗？
不可以，因为二分查找需要用到中间位置的节点，而链表不能随机访问。
那么就把中间位置的节点单独保存吧。
<!--more-->
![单链表带中间指针](http://7xpwqp.com1.z0.glb.clouddn.com/%E5%8D%95%E9%93%BE%E8%A1%A8%E5%B8%A6%E4%B8%AD%E9%97%B4%E6%8C%87%E9%92%88.png)

原来的链表写成了三个链表，记从下到上的编号为0、1、2，可以发现0号链表就是原始链表，1号链表是原始链表四等分点，2号链表是原始链表的二等分点。
我们再来查找7，初始搜索范围为(H, T)：
* 在2号链表中与4比较，7>4，更新搜索范围为(4, T)
* 在1号链表中与6比较，7>6，更新搜索范围为(6, T)
* 在0号链表中与7比较，7=7，查找成功。

形象化地说，SkipList就是额外保存了二分查找的中间信息。不过SkipList中含有随机化，生成的结构不会像上面那样完美。

![SkipList](http://7xpwqp.com1.z0.glb.clouddn.com/SkipList.png)

之后会详细讨论随机化的问题，现在先承上启下地梳理下信息：
* SkipList结合了链表和二分查找的思想
* 将原始链表和一些通过“跳跃”生成的链表组成层
* 第0层是原始链表，越上层“跳跃”的步距越大，链表元素越少
* 上层链表是下层链表的子序列
* 查找时从顶层向下，不断缩小搜索范围

![skip-search](http://7xpwqp.com1.z0.glb.clouddn.com/skip-search.png)

## SkipList与红黑树、哈希表比较
* skiplist和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的。因此，在哈希表上只能做单个key的查找，不适宜做范围查找。所谓范围查找，指的是查找那些大小在指定的两个值之间的所有节点。
* 在做范围查找的时候，平衡树比skiplist操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。
* 平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。
* 从内存占用上来说，skiplist比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而skiplist每个节点包含的指针数目平均为1/(1-p)，具体取决于参数p的大小。如果像Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势。
* 查找单个key，skiplist和平衡树的时间复杂度都为O(log n)，大体相当；而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的。
* 从算法实现难度上来比较，skiplist比平衡树要简单得多。
* 因为算法设计简单，便于在其基础上设计出一些无锁的数据结构。如ConcurrentSkipListMap。

## SkipList的性质
一个跳表，应该具有以下特征：
1. 一个跳表应该有几个层（level）组成；
2. 跳表的第一层包含所有的元素；
3. 每一层都是一个有序的链表；
4. 如果元素x出现在第i层，则所有比i小的层都包含x；
5. 每个节点包含key及其对应的value和一个指向同一层链表的下个节点的指针数组

## SkipList的实现
### Node节点定义
```JAVA
public Class SkipListNodeEntry {
  private SkipListNodeEntry[] next;
  private int val;

  public SkipListNodeEntry(int val) {
    this.next = new SkipListNodeEntry[MAX_LEVEL];
    this.val = val;
    for(int i=MAX_LEVEL; i>=0; i--) {
      this.next[i] = Null;
    }
  }

  public SkipListNodeEntry() {
    this(MIN_VALUE)；
  }

  public SkipListNodeEntry getNext(int level) {
    return this.next[level];
  }

  public int getVal() {
    return this.val;
  }

  public void setNext(int level, SkipListNodeEntry next) {
    this.next[level] = next;
  }

}
```

### SkipList定义
```JAVA
public Class SkipList {
  private SkipListNodeEntry header;
  private int level;

  public SkipList() {
    this.level = 0;
    this.header = new SkipListNodeEntry();

  }
  public SkipListNodeEntry find(int val) {
    SkipListNodeEntry p = this.header;
    for(int i=this.level; i>=0; i--) {
      while(p.getNext(i)!=Null && p.getNext(i).getVal()<val) {
        p = p.getNext(i);
      }
      if(p.getNext(i).getVal() == val) {
        return p.getNext(i);
      }
    }
    return Null;
  }

  public int genRandomLevel() {
    int level = 0;
    while(rand() % 2 && level < MAX_LEVEL - 1)
      ++level;
    return level;
  }

  public void insert(int val) {
    SkipListNodeEntry update[] = new SkipListNodeEntry[this.level];
    SkipListNodeEntry p = this.header;

    for(int i=this.level; i>=0; i--) {
      while(p.getNext(i)!=Null && p.getNext(i).getVal()<val) {
        p = p.getNext(i);
      }
      if(p.getNext(i)!=Null && p.getNext(i).getVal()==val) {
        return; //不能插入重复的节点
      }

      update[i] = p;
    }

    SkipListNodeEntry newNode = new SkipListNodeEntry(val);
    int newLevel = genRandomLevel();
    //注意，如果level比之前变高了，需要调整header
    if(newLevel>this.level) {
      for(int i=newLevel; i>this.level; i--) {
        header.setNext(i, newNode);
      }

      this.level = newLevel;
    }
    for(int i=newLevel; i>=0; i--) {
      newNode.setNext(i, update[i].getNext(i));
      update[i].setNext(i, newNode);
    }
  }

  public void delete(int val) {
    SkipListNodeEntry update[] = new SkipListNodeEntry[MAX_LEVEL];
    SkipListNodeEntry p = header;

    for(int i=this.level; i>=0; i--) {
      while(p.getNext(i)!=Null && p.getNext(i).getVal()<val) {
        p = p.getNext(i);
      }
      if(p.getNext(i).getVal() == val) {
        update[i] = p;
      }
    }

    if(update[0] == Null) {
      return; //没有找到节点
    }

    //判断SkipList的Level是不是需要降级
    int level = this.level;
    while(level>=0) {
      if(update[level]==Null) {
        continue;
      }
      else if(update[level]==this.header && update[level].getNext(level).getNext(level)==Null) {
        update[level].setNext(level, Null);
        this.level--;
      } else {
        update[level].setNext(level, update[level].getNext(level).getNext(level));
      }

      level--;
    }

  }
}
```
