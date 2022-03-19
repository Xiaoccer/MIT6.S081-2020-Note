# Virtual Memory Primitives for User Programs

Authors: Andrew W. Appel
Fields: Linux, vitrual memory
Read: 粗读
Year: 1991

## Paper in 3 Sentences

1. 介绍了用户利用虚拟内存常用的原语：TRAP、PROT1、PROTN、UNPROT、DIRTY和MAP2.
2. 介绍了几种利用虚拟内存的应用。

## Impressions

1. 利用虚拟内存实现内存垃圾回收，详见另一篇关于Baker算法的笔记。
2. 这些应用大体分为两类：
   * 对多个Pages设置保护权限，然后在page fault时，取消某个Page的保护权限。
   * 对某个Pages设置保护权限，然后在page fault时，取消该Page的保护权限。

## Notes

### Concurrent garbage collection

1. 利用虚拟内存实现并发垃圾回收，属于第一类应用，详见另一篇关于Baker算法的笔记。

### Shared vitrual memory

1. 多台节点共享一个大的虚拟内存。
2. 实现上，若Page是"read only"，可以被直接copy到每台计算机上；若某个节点想写一个Page，则必须获取该Page的最新数据，并且告诉其他节点他们的副本已失效。
3. 当节点发现想获取的Page不在其内存中，就产生Page fault从磁盘或者其他节点中加载该Page。
4. 属于第二类应用。

### Concurrent checkpointing

1. 高效实现快照内存。
2. 在开始做快照的时候，设置整个内存地址的权限为"read only"，然后开启复制线程保存每个page的数据，保存完一页恢复一页的权限。
3. 由于整个地址空间是只读，所以用户可以访问只读的page。若用户要写某个Page，会产生page fault，此时让复制线程立即复制那一页的数据，然后恢复权限供用户使用。
4. 还有一个改进的点是，不需要每次checkpoint都对整个内存地址设置权限，只需对上一轮checkpoint到本轮checkpoint中的"dirtied" page 进行权限设置即可。
5. 属于第一类应用。

### Generational garbage collection

1. 这种世代式的垃圾回收机制基于一个观察：老世代的对象通常会存活比较久，新世代的对象（如临时对象）通常更容易被垃圾回收。
2. 在回收新世代的对象时，需要知道是否有老世代的对象指向了新对象。这里可以使用VM来实现：设置老世代区域的权限设置为只读，当老世代对象要指向新对象时，会触发page fault，在page fault中记录该page地址，然会恢复该Page的可写权限。在collector需要回收时，遍历这些在page fault中记录过的地址，找出指向新对象的指针。
3. 属于第一类应用。

### Persistent stores

1. 将内存映射到磁盘，实现数据的持久化。（object-oriented databases）。
2. 到commit点的时候再将"dirty" pages写回磁盘。
3. 属于第一类应用。

### Extending addressability

1. 不太理解，对象在内存中使用32位地址，在disk中使用64位地址。个人理解是有个地方标识了目前内存中的内容是否是匹配的，不匹配的话，通过page fault从disk中重新加载正确的内容。
2. 属于第一类应用

### Data-compression paging

1. 通过压缩算法，将一个Page的内容进行压缩（用更少的bit来表示其内容）。
2. 当内存不足，需要通过LRU驱逐Page时，可将该Page压缩存在内存中，这样恢复该Page时就不需要从disk恢复，加快时间。
3. 属于第二类应用。

### Heap overflow detection

1. 通过guard page，实现heap overflow的检查
2. 属于第二类应用。

### 其他

1. 不同应用利用VM时，需考虑不同Page Size。
2. 需考虑TLB Consistency。
3. 多个虚拟地址的mapping需要考虑Cache的不一致性

## URL

1. https://pdos.csail.mit.edu/6.S081/2020/readings/appel-li.pdf