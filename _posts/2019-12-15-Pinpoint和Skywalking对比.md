## Pinpoint和Skywalking对比

现在市场主流的采用 `agent` 监控的开源项目主要有2个，一个是韩国团队开源的[pinpoint](https://github.com/naver/pinpoint) 以及 [wu-sheng](https://github.com/wu-sheng) 开源的[skywalking](https://github.com/apache/skywalking)，下面主要比较一下在团队运维，二次开发，用户使用等几个方面进行比较。

### 对比

#### 团队运维

* pinpoint采用的Hbase进行存储
* skywalking采用Elasticsearch进行存储

以中小公司的体量维护一套Hbase是较为困难的，出现问题也比较难解决，可能团队中都没有人能够hold住这个东西，对比Elasticsearch，搭建更加简单，数据的操作也更加简单，资料也随处可找，相对Hbase来说运维更为简单。

#### 二次开发

> 相同点: Java agent 的开发都是大同小异的，都是采取 -javaagent:/path/agent.jar 引入jar包，然后新建classLoader加载core包和plugin包，再使用 `webapp-classloader` 进行加载，能够在plugin中使用Spring/SpringBoot的classloader，相比之下，pinpoint和skywalking采用的方式都是一样的


* pinpoint:使用 `ASM#tree` 进行字节码注入，自己阅读它的代码，去二次开发相对来说难度较大，如果不熟悉 `ASM` 的话，更是一只无头苍蝇，难以下手。
* skywalking:使用 `ByteBuddy` 进行处理，相比 `ASM` 难度瞬间降低`99%` ， 在看过 `pinpoint` 的基础上，我只用了半个小时便基本上理顺了 `skywalking` 的整个结构，比起 `pinpoint` 来说，更加简单，即使是没有了解过 `Agent` 的人，也能在这个基础上做一些二次开发和处理。

#### Agent接入和前端UI

接入的方式2个差不多，但是pinpoint的接入方式更加的繁琐. 

在前端UI上

* pinpoint: service map ， call stack，jvm 这些信息都是我们所需要的，没有一个是累赘，看到这些数据基本上就可以找到应用出问题的所在，其中异常被明显的标注出来，应用的耗时也是，在跟踪慢请求时很有用，当然它的处理方式是在opentracing的基础上，引入了自己的设计。
* skywalking: service map , call stack 这些都比较模糊，初次使用根本就不能直接的找出应用问题所在，前端界面难以使用。当然这也是接入opentracing的规范有关系。

### 其他

上述的功能在阿里云的 ARMS 中也有体现，不过ARMS是修改了开源项目 `pinpoint` 整合而来，更是加上了 `arthas` ， 不需要使用 `-javaagent` 在项目启动的时候进行导入. 这一点可以学习，但是有一个致命的缺点的，就是太贵了.