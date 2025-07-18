# 为什么要选择适合数据库的硬件

在部署数据库前选择合适的硬件会对后续的集群稳定、集群性能更优都有收益，避免盲目的堆硬件为未来带来不可预估的风险。

# CPU 选择

1、物理机的生产 `CPU` 建议最低选择 18 Core 以上、并且选择支持超线程。

2、在`CPU`的选择上建议选择主频在2.5GHz以上的CPU，并且CPU的指令支持sse2、sse4、avx及avx2的指令集。

3、如果是虚拟机需要把`CPU`工作模式设置为直通模式，与宿主机的指令保持一致。

4、`CPU` 建议使用直通模式（Passthrough Mode）,避免使用性能模式（Performance Mode）和仿真模式（Emulation Mode）。

## CPU 查看常用命令
```shell
# 查看物理CPU个数
cat /proc/cpuinfo|grep "physical id"|sort -u|wc -l

# 查看每个物理CPU中core的个数(即核数)
cat /proc/cpuinfo|grep "cpu cores"|uniq

# 查看逻辑CPU的个数
cat /proc/cpuinfo|grep "processor"|wc -l

# 查看CPU的名称型号
cat /proc/cpuinfo|grep "name"|cut -f2 -d:|uniq

# 查看CPU的睿频
grep -E '^model name|^cpu MHz' /proc/cpuinfo

# 查看物理CPU个数
grep "physical id" /proc/cpuinfo | sort | uniq| wc -l

# 查看每个物理CPU中core的个数(即核数)
grep "cpu cores" /proc/cpuinfo | uniq

# 查看逻辑CPU的个数
grep "processor" /proc/cpuinfo | wc -l
```


# 储存


# RAID 卡


# 网络



# 参考资料

[1]. (https://mp.weixin.qq.com/s/cUCcyfs_xfl2WFqM-LICWQ)

[2]. (https://mp.weixin.qq.com/s/XZLpAr7Oau0rwtAm7B_3TQ)

[3]. (https://mp.weixin.qq.com/s/XZLpAr7Oau0rwtAm7B_3TQ)

[4]. https://cloudberry.apache.org/zh/docs/cbdb-op-software-hardware/


