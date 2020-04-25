gc调优实战https://plumbr.io/handbook/gc-tuning-in-practice

https://developer.amd.com/4-easy-ways-to-do-java-garbage-collection-tuning/

最佳实战https://backstage.forgerock.com/knowledge/kb/article/a35746010

# 什么时候调优

当应用频繁卡顿/卡顿时间久，预示着该调优了。堆在JVM启动时创建并划分成不同的区域/代，关键是yarn区和Old区：

1. Young区用于存储新对象，当Young区满了时，gc(ParNew)进程移除无用对象并把长期存活对象移到老年代，空出新生代。新生代对象大部分短命。新生代比较小且gc频繁，但是对性能影响有限。
2. 老年代用于长期存活对象，老年代满了时，CMS gc进程启动并清理无用对象。老年代更大，且gc对性能影响比较大。

下图是堆区可用的gc选项(jdk7)

![](https://backstage.forgerock.com/cloud-storage-ws/api/v1/cloudstorage/getfile/61581fd4-6281-46a0-8a15-1ecdf35c79b1?size=large2x)

需要调优Java堆的几个现象：

1. CPU使用率高，当gc线程消耗大量cpu，你会看到cpu使用出现毛刺
2. 应用卡顿，日志文件出现较大间隙

遇到这些问题，首先检查gc日志。

# 读懂gc日志

## ParNew 日志

```shell
24.245: [GC24.245: [ParNew: 545344K->39439K(613440K), 0.0424360 secs] 594671K->88767K(4101632K), 0.0426850 secs] [Times: user=0.17 sys=0.00, real=0.04 secs] 
29.747: [GC29.747: [ParNew: 584783K->27488K(613440K), 0.0889130 secs] 634111K->104723K(4101632K), 0.0892180 secs] [Times: user=0.24 sys=0.01, real=0.09 secs]
```



