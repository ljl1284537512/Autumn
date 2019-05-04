### 一致性Hash算法
为什么要有一致性Hash算法？
一致性Hash算法为什么是对2^32取模？
如何解决Hash非均匀分配(数据倾斜)的问题？
为什么称为"一致性"hash?
> 增加节点扩容的时候，它对集群中原有节点的影响是一致的，就是对大部分节点没有什么影响。
#### 为什么需要一致性hash算法？

以缓存为例，当我们需要提高系统性能时，常常会引入缓存。
分布式高并发系统的特点：1. 高并发；2. 海量数据；而高并发的解决方案，一般都是集群(服务器)+缓存，而当并发量巨大时，服务器集群可以解决，但是如果只有单台缓存机器的话，扛不住高并发，并且在海量数据的情况下，达到单台缓存服务器的极限，这个时候就需要引入集群缓存，而为了解决海量数据的问题，需要将数据切分到个各个缓存服务器上。

如何保证数据均衡分布到缓存集群的节点上？
- 均衡分布方式一：hash(key) % 集群节点数。假如3个缓存节点形成集群，那么根据数据hash(key) % 3 就能获得所要存储的节点机器。这个时候如果需要临时增加一台缓存服务器来提高集群性能，按照hash(key)%4 的算法，将查不到对应的key,则从数据库中查找，影响范围是3/4（3和4的最小公倍数是12，而只有当key=0，1，2时，对3或4的取模值是一样的，也就是说只有4分之1的数据不受影响，其他数据都受到了影响）, 如果是从99台扩展到100台，则影响范围是99%，这些范围的数据都将从数据库中查找，很可能会导致数据库被打死，数据库被打死了，整个服务器都没用了（缓存雪崩）。
- 一致性Hash算法：
	1. hash值一个非负整数，把非负整数的值范围做成一个圆环；
	2. 对集群的节点的某个属性求Hash值（如节点名称），根据hash值把节点放到环上；
	3. 对数据的Key求hash，一样的把数据页放到环上，按顺时针方向，找离它最近的节点，就存储到这个节点上。

#### 新增节点后能均衡缓解原有节点的压力吗？
#### 集群中的节点一定会均衡分布在环中吗？
不会，所以引入一致性Hash算法+虚拟节点，虚拟节点分布越多，越均匀。
#### 什么是一致性Hash算法?
#### 什么地方使用到了一致性hash算法？
缓存，ES，Hadoop，分布式数据库

#### 分析一致性Hash算法的实现

##### 核心点

- 物理节点
- 虚拟节点
- Hash算法
	- hashCode: 不够散列，会有负值；
	- 其他hash算法：Memcache推荐的KETAMA_HASH算法；
- 虚拟节点放到环上
	- 根据hash值排序存储；
	- 排序存储要被快速查找；
	- 这个排序存储还要能方便变更(因为有可能要加节点)；
- 数据找到对应的虚拟节点
- 数据结构-treeMap
	- headMap()
	- tailMap()
	- subMap()

#### 源码实现
```java
package com.risesun.hash;

import java.util.*;

public class ConsistenceHash {

    // 物理节点集合
    private List<String> phyNodes = new ArrayList<>();
    // 虚拟节点数，用户可指定
    private int virtualNums = 100;
    // 物理节点与虚拟节点的对应关系
    // 根据key找缓存节点，首先需要找到对应的虚拟节点，再去找物理节点
    private Map<String, List<Integer>> real2VirtualMap = new HashMap<>();
    // 排序存储结构红黑树-放置虚拟节点，key是虚拟节点的hash值，value是物理节点
    private SortedMap<Integer,String> sortedMap = new TreeMap<>();

    public ConsistenceHash(){
    }
    public ConsistenceHash(int virtualNums) {
        this.virtualNums = virtualNums;
    }

    /**
     * 增加服务器-物理节点
     * 3台服务器->4台
     * @param node
     */
    public void addServer(String node){
        // 放入到集合中
        this.phyNodes.add(node);

        String vnode = null;
        int i = 0,count = 0;

        List<Integer> virtualNodes = new ArrayList<>();
        real2VirtualMap.put(node,virtualNodes);

        while(count < this.virtualNums){
            vnode = node + "$$" + i++;
            int hash = vnode.hashCode();
            // 排除hash碰撞，需要反复计算hash值
            if(!this.sortedMap.containsKey(hash)){
                virtualNodes.add(hash);
                this.sortedMap.put(hash, node);
                count ++;
            }
        }

    }

    /**
     * 移除物理节点
     * @param node
     */
    public void removeServer(String node){
        phyNodes.remove(node);
        real2VirtualMap.get(node).forEach(item-> {sortedMap.remove(item);});
        List<Integer> virtualNodes = real2VirtualMap.get(node);
        virtualNodes= null; // help gc
        real2VirtualMap.remove(node);
    }

    /**
     * 找到数据所在的物理节点
     * @param key
     * @return
     */
    public String getServer(String key){
        int hash = key.hashCode();
        // 顺时针方向找最近的物理节点，得到大于该Hash值的所有虚拟节点
        SortedMap<Integer, String> subMap = sortedMap.tailMap(hash);
        if(subMap.isEmpty()) // 已经是尾节点了
            // 取hash值最小的节点
            return sortedMap.get(sortedMap.firstKey());
        return subMap.get(subMap.firstKey());
    }

    public int getVirtualNums() {
        return virtualNums;
    }
    public void setVirtualNums(int virtualNums) {
        this.virtualNums = virtualNums;
    }

    public static void main(String[] args) {
        ConsistenceHash consistenceHash = new ConsistenceHash(10);
        consistenceHash.addServer("192.168.0.111");
        consistenceHash.addServer("192.168.0.112");
        consistenceHash.addServer("192.168.0.113");
        consistenceHash.addServer("192.168.0.114");
        consistenceHash.addServer("192.168.1.114");
        consistenceHash.addServer("192.168.1.115");
        consistenceHash.addServer("192.168.1.116");

        System.out.println(consistenceHash.getServer("hello"));
        System.out.println(consistenceHash.getServer("this"));
        System.out.println(consistenceHash.getServer("world"));
        System.out.println(consistenceHash.getServer("!"));
    }
}

```
