## MySQL 从库扩展探索

文/Jean-François Gagné 

>本文主要介绍 Booking 网站在业务发展过程中碰到 MySQL 主库挂载几十甚至上百个从库时探索的解决方案：使用 Binlog Server。Binlog Server 可以解决五十个以上从库时主库网络带宽限制问题，并规避传统的级联复制方案的缺点；同时介绍了使用 Binlog Server 还可以用于优化异地机房复制和拓扑重组后的主库故障重组。作者探索问题循序渐进的方式以及处理思路值得我们学习。

Booking 网站后台有着非常复杂的 MySQL 主从架构，一台主库带五十个甚至有时带上百个从库并不少见。当从库到达这个数量级之后，一个必须重点关注的问题是主库的网络带宽不能被打满。业界有一个现成的但是有缺陷的的解决方案。我们探索了另外一种能更好适应自身需求的方案：Binlog Server。我们认为 Binlog Server 可以简化灾难恢复过程，也能使故障后从库迅速升级为新主库变得容易。下面会详细描述。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a887444de41.jpg" alt="图1  多从库的MySQL主从架构" title="图1  多从库的MySQL主从架构" />

图1  多从库的 MySQL 主从架构

一个 MySQL 主库带多个复制的从库时，每次对主库的修改都会被每个从库请求复制，提供大量二进制日志服务会导致主库的网络带宽饱和。产生大量二进制日志的修改是很常见的，下面是两个例子：

- 在使用行模式 Binlog 日志复制方式的实例中执行大事务删除操作；

- 对一个大表执行在线结构修改操作（Online Schema Change）。

在图1的拓扑图中，假设我们在一个 MySQL 主库上部署100个从库，主库每产生 1MB 的修改每秒都会产生 100MB 复制流量。这和千兆网卡的流量上限很接近了，而这在我们的主从复制结构中很常见。

这个问题的传统解决方案是在主库和从库之间部署中继主库。在图2的拓扑部署中，与很多从库直接连到主库不同的是，我们有几个从主库复制的中继主库，同时每个中继主库有几个下级从库。假设有100个从库和10个中继主库，这种情况下，允许在打满网卡流量之前产生10倍于图1架构的二进制日志。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a8873423d82.jpg" alt="图2  包含中继主库的MySQL主从架构" title="图2  包含中继主库的MySQL主从架构" />

图2  包含中继主库的 MySQL 主从架构

然而，使用中继主库是有风险的：

- 中继主库上的主从复制延迟将影响它的所有从库；

- 如果一个中继主库出现异常，所有该中继主库的从库将停止复制并必须重新初始化，参考1 （这会带来很高的维护成本并有可能产生在线故障，译者注）；

- 针对图2第二个问题我们做深入研究。一个思路是，如果 M1 出现故障，可以把它的从库的主库配置指向到其他中继主库，但事实上并没有那么简单；

- S1 从 M1 复制的二进制日志依赖于 M1；

- M1 和 M2 有不同的二进制日志位置（译者注：这两个库是不同的数据库，在同一时间二进制日志状态、位置可能不同）；

- 手工推进 S1 的二进制日志位置到 M2 非常难而且可能导致数据不一致。

GTID 可以协助我们指向从库，但是它不能解决第一个关于延迟的问题。

实际上我们不需要中继主库的数据，而只是需要提供 Binlog 二进制日志服务。同时，如果 M1 和 M2 可以提供二进制日志服务，并且日志位置是相同的，我们可以很容易地交换各自的从库。根据这两点观察，我们构思了 Binlog Server 二进制日志服务。

Binlog Server 替代图2中的中继主库，每个 Binlog Server 做如下事情：

- 从主库下载二进制日志；

- 与主库使用相同结构（文件名和内容）保存二进制日志到磁盘；

- 提供二进制日志给从库，就像它们是这些从库的二级主库；

当然，如果一个 Binlog Server 异常了，我们可以很容易地把它的从库指向到其他 Binlog Server。更惊喜的是，由于这些 Binlog Server 没有本地数据的变化，只是给下游提供日志流，相对于有数据的中继主库来说，可以很好地解决延迟的问题。

### 另一个案例1：避免远程站点上的深度嵌套复制

Binlog Server 还能用于规避远程站点上的深度嵌套复制的问题。

假设有两个不同地域机房，每个机房需要四个数据库服务器，当网络带宽需要特别关注的时候（E、F、G和 H 在远程站点），图3的拓扑图是一个典型的部署方式。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a8872311bbd.jpg" alt="图3  使用中继主库部署的MySQL异地主从架构" title="图3  使用中继主库部署的MySQL异地主从架构" />

图3  使用中继主库部署的 MySQL 异地主从架构

但是这个拓扑结构会受到上述讨论问题的影响（复制延迟将从 E 传递至 F、G 和 H，同时 E 异常之后，F、G、H 就会失败）。如果我们用图4的架构就好很多，但是这种架构需要更多的网络带宽，而且一旦主复制节点发生问题，异地机房从库需要重建一套新的主从架构。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a8871519309.jpg" alt="图4  不包含中继主库的异地机房MySQL主从部署" title="图4  不包含中继主库的异地机房MySQL主从部署" />

图4  不包含中继主库的异地机房 MySQL 主从部署

在异地机房主从架构中使用 Binlog Server，我们可以综合上面两种方案的优势（低带宽使用和中继数据库不产生延迟）。拓扑图如图5所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a88709e340a.jpg" alt="图5  包含Binlog Server的异地机房MySQL主从架构" title="图5  包含Binlog Server的异地机房MySQL主从架构" />

图5  包含 Binlog Server 的异地机房 MySQL 主从架构

在图5的 MySQL 主从架构中，Binlog Server (X)看起来是一个单点，但是如果它异常了, 重新启动另外一个 Binlog Server 是很容易的。而且也可以像图6示例的那样在异地机房运行两个 Binlog Server。在这个部署中，如果 Y 异常了，G 和 H 可以指向到 X，如果 X 异常了，E 和 F 可以指向到 Y，Y 可以指向到 A。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a886fee7d8f.jpg" alt="图6  包含两个Binlog Server的异地机房MySQL主从架构" title="图6  包含两个Binlog Server的异地机房MySQL主从架构" />

图6  包含两个 Binlog Server 的异地机房 MySQL 主从架构

运行 Binlog Servers 其实不需要更多更好的硬件，在图6中，X、E、Y、G 可以安装在同一台硬件服务器上。

最后，这种架构（有1或2个 Binlog Server）有一个很有意思的属性：如果主站点的主库发生故障，异地机房从库可以收敛到完全一致的状态（只要 X 服务器的二进制正常）。这使得重组 MySQL 主从架构变得很容易：

- 任何一个从库可以成为新主库；

- 新主库的二进制日志位置在发送写之前会标注出来；

- 其他的节点成为新主库的从库，在之前提到的二进制日志位置。

### 另一个案例2：简单的高可用实现

Binlog Server 可以用于高可用架构的实现。假如图7主库故障了，我们希望尽快选出新的主库，就可以部署 GTIDS 或使用 MHA，但它们都有缺点。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a886eff0f6c.jpg" alt="图7  6个从库直连主库的MySQL主从架构" title="图7  6个从库直连主库的MySQL主从架构" />

图7  6个从库直连主库的 MySQL 主从架构

假设我们像图8一样在主库和从库之间部署一个 Binlog Server：

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a886e10710f.jpg" alt="图8   包含Binlog Server的MySQL主从架构" title="图8   包含Binlog Server的MySQL主从架构" />

图8   包含 Binlog Server 的 MySQL 主从架构

- 如果 X 异常，我们能把所有从库指向 A；

- 如果 A 异常，所有从库会达到一个一致的状态，使得将从库重组成一个复制树变得很容易（像上面提到的）。

- 如果我们希望实现高扩展性和高可用性，就可以部署成图9的主从架构。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a886d1d6b18.jpg" alt="图9  包含多个Binlog Servers的MySQL主从架构" title="图9  包含多个Binlog Servers的MySQL主从架构" />

图9  包含多个 Binlog Servers 的 MySQL 主从架构

如果一个 Binlog Server 异常，它的从库将指向到其他 Binlog Servers。如果 I1 失败了：

- 我们找到有更多二进制日志的 Binlog Server （假定在这个例子中是 I2）；

- 将其他 Binlog Server 指向到 I2（像图10一样）；

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a886c8ab822.jpg" alt="图10  主库异常后调整Binlog Server指向的MySQL主从架构" title="图10  主库异常后调整Binlog Server指向的MySQL主从架构" />

图10  主库异常后调整 Binlog Server 指向的 MySQL 主从架构

当所有的从库都达到一个共同的状态，我们重新组 MySQL 主从架构。 

### 结论

我们在 MySQL 主从架构中引入了一个新的组件：Binlog Server。它使从库水平扩展不会超越网络带宽的的限制，同时也没有传统的级联复制解决方案的缺点。

我们觉得 Binlog Server 还可以用于解决其他两个问题：远程站点复制和拓扑重组后的主库故障重组。后续我们将带来的 Binlog Server 的其他使用案例，敬请期待更多细节。

### 参考

从库重新初始化的大量工作可以通过 GTIDS 或通过采用高可用的中继主库（DRBD 或 Netapp 的 Snapshot）来避免，但是这两个解决方案分别会带来新的问题。