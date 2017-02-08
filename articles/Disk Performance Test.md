# 测试目标

* IO 读写的延迟
* IO 读写的吞吐量（带宽）

以上的两类信息基本上可以反应出某个系统在IO上的性能，但是要注意：在不同的应用场景下，
上述信息的具体值会有较大差别，因此在进行测试之前，需要首先确定具体的应用场景。
比如该场景的读写是连续的还是跳跃的，是读多还是写多等等，然后在此场景的基础上设计测试例程。

另外，以上两类信息并不停准确反应该系统的IO性能，想要完成测试某个系统的IO性能，至少还需要测试以下参数：

* 抖动： 反应IO子系统工作的稳定性。
* IOPS： 每秒可以进行多少次IO传输。

每次进行IO性能测试之前最好先清空系统的IO缓存以保证测试结果准确。
关于`Cache`和`Buffers`的区别，这儿[^3]有详细的解释。

### 清空系统中缓存的方法
* 清空 Cache `sync;sync;sync`
* 清空 交换分区 `sudo swapoff -a && swapon -a`
* 清空 Buffers `sudo sysctl -w vm.drop_caches=3` 这条命令会清空所有的slab对象和页面缓存。如果只想清空页面缓存，可以传参数`1`。

# 可用工具

## fio[^1]

fio是一个功能强大的io性能测试工具，用它可以模拟多种场景下的io传输过程。

## Iozone[^2]

# 参考链接
[^1]: [https://github.com/axboe/fio](https://github.com/axboe/fio)
[^2]: [http://www.iozone.org/](http://www.iozone.org/)
[^3]: [http://stackoverflow.com/questions/6345020/linux-memory-buffer-vs-cache](http://stackoverflow.com/questions/6345020/linux-memory-buffer-vs-cache)