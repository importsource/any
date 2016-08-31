docker作为一个linux平台上一款轻量级虚拟化容器的管理引擎。在短短的两年内火得不得了。人人都在说docker。大大小小的容器summit。

docker的前景被普遍看好。最近恰好在研究docker。看了孙宏亮大牛的《docker源码解析》更是很有感触。于是我就在想，可不可以写一个学习

体会来做一个阶段性总结。我的想法就是如何从起名字的角度来阐释docker架构的一些内部原理。曾经有一位大牛导师给我说过这样几句话：

你给每个类起的名字代表了你对这个实现逻辑对最高理解。原话不记得了。但大体意思就是这样，这句话也成了我后来判断一个人代码质量的

重要依据。


docker架构图  此处。。。。。



Docker的总架构图就是这样。架构中主要有DockerClient、DockerDaemon、Docker Registry、Graph 、 Driver、libcontainer 以及Docker Container。


首先我们来看看DockerClient。这个名字很显然。是一个客户端。这时候你是不是联想到了命令、浏览器等等。不说了。


DockerDaemon。Daemon是守护的意思。读“滴萌”。前面说了client。那么就可能会联想到这应该是一个server。没错这个就是一个server。但docker的大牛们为什么不叫DockerServer呢？是有原因的。因为Daemon中不仅仅有server，还有其他的。还有Engine。这个Engine中有很多的job。DockerDaemon内部所有的任务都是由Engine中的一个个Job来完成。



