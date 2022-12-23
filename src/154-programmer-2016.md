## 打造金融行业私有云数据库——宁波银行的分布式数据库探索

文/磊，王智臣，费振旭，谢渊

### 企业简述

宁波银行股份有限公司（以下简称“宁波银行”）成立于1997年4月10日，是一家具有独立法人资格的股份制商业银行。2007年7月19日，宁波银行在深圳证券交易所挂牌上市，成为国内首批上市的城市商业银行之一。宁波银行积极推进管理和金融技术创新，努力打造公司银行、零售公司、个人银行、信用卡、金融市场五大利润中心，实现利润来源多元化。也是中国银行业资产质量好、盈利能力强、资本充足率高、不良贷款率低的银行之一，是全球1000强银行，位居第279位，也是中国财富500强企业。

### 架构探索的行业背景

金融行业的数据服务主要是国外商业软硬件产品构成的集中式数据架构，简称 IOE 架构。IOE 是指以 IBM 为代表的小型机、以 Oracle 为代表的数据库和以 EMC 为代表的中高端存储设备组合从软件到硬件的企业数据系统，且 IOE 架构占据了全球商用数据服务的大多数市场份额。

中国银行业信息科技十三五发展规划提出深化科技创新，推进互联网、大数据、云计算新技术应用及加强信息安全管理，提升信息科技风险管理水平，推进科技开发协作等方面。

鼓励金融企业打造私有云平台，面向互联网场景的手机银行、网上银行、电话银行、短信银行、在线支付、扫码支付等主要信息系统全部迁移至云计算架构平台。

### 业务场景简述

宁波银行科技部积极顺应行业发展趋势，借助科技创新，打造互联网+普惠金融，研发运营移动端互联网产品 App，使用互联网思维的抢红包营销和秒杀类促销活动吸引用户，打造粉丝经济并为创新业务引流，实现银企和个人的互动，增强银行客户的粘性、信息通达率和效率。

移动互联网和 PC 互联网化的业务渠道，相比传统行业、门店和 ATM 机等渠道，新渠道存在用户基数大、数据容量大、并发访问井喷、高可靠和高性能等技术门槛，为传统模式的 IOE 集中式架构带来极大的挑战，IOE 集中式数据架构的初衷以欧美人口基数和增长率为预判基础设计研制，从未考虑过中国庞大的用户基数和企业经营模式转型的巨大挑战。新型互联网技术已在创新型的传统企业和互联网企业应用成熟，且借助技术创新和业务模式创新优势，正在迫使金融行业变更。为此，作为科技创新和模式创新排头兵的宁波银行，选择拥抱变化，积极探索业务创新和创新型业务，率先试点探索给予分布式数据库的私有数据库云技术。

### 性能测试

私有云数据库性能测试结果见表1，IOE 集中式数据库性能测试结果见表2。私有云分布式数据库与 IOE 集中式数据库性能对比如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63b7fee86a.jpg" alt="表1  私有云数据库性能测试结果" title="表1  私有云数据库性能测试结果" />

表1  私有云数据库性能测试结果

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63b8bc8d20.jpg" alt="表2  集中式数据库性能测试结果" title="表2  集中式数据库性能测试结果" />

表2  集中式数据库性能测试结果

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63bc6b4eee.jpg" alt="图1  私有云分布式数据库 Vs. 集中式数据库性能" title="图1  私有云分布式数据库 Vs. 集中式数据库性能" />

图1  私有云分布式数据库 Vs. 集中式数据库性能

#### 私有云数据库性能测试

从测试的数据可以看出，在这类批量的数据加载业务场景，私有云数据库的吞吐量与拆分的数据节点以及并发的连接数紧密关联。在并发连接数固定的情况下，增加拆分数据节点数量，可极大提高私有云数据库的吞吐量，如图2、图3所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63bf838df7.png" alt="图2  INSERT查询QPS" title="图2  INSERT查询QPS" />

图2  INSERT 查询 QPS

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63c196a657.png" alt="图3  SUID查询QPS" title="图3  SUID查询QPS" />

图3  SUID 查询 QPS

在拆分数据节点数量固定的情况下，成倍增加并发连接数，HotDB 的吞吐量略有波动，如图4、图5所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63c3edce10.png" alt="图4  INSERT查询QPS" title="图4  INSERT查询QPS" />

图4  INSERT 查询 QPS

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63c686c972.png" alt="图5  SUID查询QPS" title="图5  SUID查询QPS" />

图5  SUID 查询 QPS

若同时增加拆分数据节点的数量与并发连接数，HotDB 的吞吐量大大提高，如图6、图7所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63c9ac5c49.png" alt="图6  INSERT操作QPS" title="图6  INSERT操作QPS" />

图6  INSERT 操作 QPS

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63cde238cb.png" alt="图7  SUID操作QPS" title="图7  SUID操作QPS" />

图7  SUID 操作 QPS

### 基础环境

硬件环境如表3所示，软件环境如表4所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63d520ab33.png" alt="表3  硬件环境" title="表3  硬件环境" />

表3  硬件环境

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63d5c19fae.png" alt="表4  软件环境" title="表4  软件环境" />

表4  软件环境

### 私有云数据库的组件构成

#### 私有云数据库的组件图

私有云数据库由分布式数据库中间件服务、图形化的分布式数据库管理平台、数据存储的 MySQL 数据库服务和多块本地盘的 X86 服务器四大组件构成：

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63e9e309ca.jpg" alt="图8  私有云数据库组件图" title="图8  私有云数据库组件图" />

图8  私有云数据库组件图

- 分布式数据库中间件

分布式数据库中间件实现 OLTP 业务场景信息系统的海量数据存储分布式化、海量访问请求负载均衡化、数据访问透明化，应用程序的全部操作通过分布式数据库中间件完成，有相应的应用连接池和数据库连接池，还有数据服务高可用、死锁检测、主备同步数据一致性、主备切换等功能。

- 图形化的分布式数据库管理平台

提供业务数据分片规则设置、数据源管理、数据节点管理、数据服务监控拓扑大屏等图形化功能，帮助数据库运维团队直观地呈现和管理私有云数据库集群。

- MySQL 数据库软件

私有云数据库底层的数据存储和数据服务，由 MySQL 数据库服务实例集群组成和提供，MySQL 数据库软件采用开源社区版本。

- 多块本地硬盘的 X86 服务器

数据物理存储由行业通用型 X86 服务器的本地硬盘提供，取代传统数据系统挂载存储设备的硬件架构，也是业界主流模式和趋势。

### 数据服务的可靠性测试

#### 分布式数据库中间件服务的可靠性

分布式数据库中间件服务是采用 HA 高可用架构的方式部署，借助开源软件 KeepAlive 实现虚拟 IP 服务并控制 VIP 的漂移，为此我们手工模拟操作：Kill 掉当下主服务的分布式数据库中间件服务，观察正处在并发操作的应用程序端报错信息和分布式数据库中间件服务恢复情况；拔掉当下主服务的分布式数据库中间件服务的物理服务器网线和电源线情况。反复操作后，总体观察到应用程序服务有重连机制情况下，能顺利切换到备用服务，时间长度在3-5秒之间。

#### MySQL 数据库服务的可靠性

MySQL 数据库的数据服务采用 HA 高可用双主同步架构的方式部署，MySQL 数据库的数据冗余高可用借助半同步复制实现，MySQL 数据库的数据服务高可用由分布式数据库中间件服务实现，内含 MySQL 数据库服务的检测机制、数据同步一致性检测算法和切换算法。为此我们手工模拟操作：Kill 掉当下主服务的 MySQL 数据库实例进程，观察正处于并发操作的应用程序端报错信息和 MySQL 数据库的数据服务恢复情况；拔掉当下主服务的 MySQL 数据库服务器的网线和电源线，观察正处于并发操作的应用程序端报错信息和 MySQL 数据库的数据服务恢复情况；模拟 MySQL 数据库服务实例 Hang 住情况。

我们能观察到应用程序端正在主 MySQL 数据库服务端运行的操作报错，还未发到 MySQL 数据库服务端的操作请求被分布式数据库中间件短暂 Hold 住，待切换至备用 MySQL 数据库服务实例上后，再下发运行。无主备延迟的情况下，数据库服务的切换时间为1-3秒；有主备延迟的情况下，要视数据延迟多少和达到同步一致的时长决定。

### 基础功能测试

**DDL 命令**

- 创建表结构及数据类型的测试验证

```
root@localhost:test 5.6.32-log 12:44:58> CREATE TABLE table_define_type(
    -> tinyint_type TINYINT,
    -> smallint_type SMALLINT,
mediumint_type MEDIUMINT,
    -> mediumint_type MEDIUMINT,
    -> int_type INT AUTO_INCREMENT NOT NULL ,
    -> bigint_type BIGINT,
    -> char_type CHAR(20),
    -> varchar_type VARCHAR(100),
    -> text_type TEXT,
    -> enum_type ENUM('hotdb','cobar','mycat'),
    -> set_type SET('hotdb','cobar','mycat'),
    -> decimal_type DECIMAL(10,2),
    -> date_type DATE,
    -> datetime_type DATETIME,
    -> timestamp_type TIMESTAMP,
    -> PRIMARY KEY(int_type),
    -> UNIQUE KEY(bigint_type))ENGINE=InnoDB  COMMENT "testing data type";
Query OK, 0 rows affected (0.18 sec)
```

总结：新创建的表对象，测试验证了私有云数据库对常用的整型、精确浮点型、字符串型、日期型、枚举型、集合型等数据类型的兼容性。

- 测试数据库表对象删除操作

```
root@localhost:test 5.6.32-log 12:52:21> DROP TABLE table_define_type;
Query OK, 0 rows affected (0.06 sec)
```

- 测试数据库表对象变更操作

```
root@localhost:test 5.6.29-log 12:58:58> ALTER TABLE  table_define_type ADD COLUMN add_col VARCHAR(40);
Query OK, 0 rows affected (0.19 sec)
Records: 0  Duplicates: 0  Warnings: 0
root@localhost:test 5.6.29-log 12:59:05> ALTER TABLE table_define_type MODIFY COLUMN add_col VARCHAR(40) NOT NULL DEFAULT '';
Query OK, 0 rows affected (0.18 sec)
Records: 0  Duplicates: 0  Warnings: 0
root@localhost:test 5.6.29-log 12:59:13> ALTER TABLE table_define_type ADD INDEX idx_addcol(add_col);
Query OK, 0 rows affected (0.07 sec)
Records: 0  Duplicates: 0  Warnings: 0
root@localhost:test 5.6.29-log 12:59:19> ALTER TABLE table_define_type DROP INDEX idx_addcol;
Query OK, 0 rows affected (0.07 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

总结：测试了表对象增加字段、修改字段属性、创建索引和删除索引四种常见变更操作。

**全局序列**

私有云数据库实现了全透明的全局序列，达到无须做任何配置或设置，一张表定义为分片表后，表对象创建命令中含自增序列则自动由私有云数据库的分布式数据库中间件服务提供全局序列服务。

**判断子句**

测试验证 SELECT 子句中常见的 IF()、IFNULL()和 CASE…WHEN…END 等判断函数，在不涉及多张表多个字段的情况下，该功能完全兼容，满足业务正常需求。

**跨库JOIN**

测试验证 SELECT 子句中常见的 LEFT JOIN（左连接）、RIGHT JOIN（右连接）、INNER JOIN（内连接）三种，非混合使用的情况下，私有云数据库兼容三种连接且能做到多表连接，但若混合表连接类型和 ON 子句中出现 OR，则暂不支持。同时，从性能测试角度，不同连接情况下，有无数据交叉的性能优化。

**分组/分页/排序**

测试验证了 ORDER BY 排序、GROUP BY 分组、ORDER BY ... LIMIT 分页、HAVING 结果集过滤，私有云数据库产品完全兼容 MySQL 数据库。在数据未变更的前提下，能确保每次执行返回的结果集都相同，分页语句同集中式数据库的结果集稍有区别，其它类型同集中式数据库结果集相同。

**DML 语句**

测试验证了单库情况和跨库情况下，INSERT … SELECT …、INSERT BATCH、INSERT、DELETE、UPDATE 五种类型的 DML 操作，同时测试 DELETE 和 UPDATE 子句中的 JOIN 内连接，也完全兼容 MySQL 数据库的 DML 操作且数据更新正确。

**分布式事务**

抽选测试分布式事务简述见表5。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c63feb2d0c7.jpg" alt="表5  抽选测试分布式事务" title="表5  抽选测试分布式事务" />

表5  抽选测试分布式事务

抽选测试分布式事务的通过率见表6。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c640318f620.jpg" alt="表6  抽选测试分布式事务的通过率" title="表6  抽选测试分布式事务的通过率" />

表6  抽选测试分布式事务的通过率

**分布式事务验证总结**

分布式事务的测试用例选型和逻辑设计至关重要，分布式事务的测试类型分为正常逻辑、非正常逻辑、极端故障和事务并发冲突环境，需借助自动化压测程序完成极端和复杂情况的测试。

### 去 IOE 实战总结

移动 App 业务系统完全运行于 X86 服务器和 MySQL 数据库软件构成的数据系统平台，完全能达到金融行业的可靠性、安全性和可维护性的要求。金融行业的互联网化业务系统将率先基于 IaaS 层 Openstack 私有云平台和基于 MySQL 数据库软件的私有云数据库平台，完全可取代小型机、国外商业数据库软件、中高端存储设备，但需要在数据库服务器硬件配置、规范标准与自动化、架构设计等方面进行适当变更。

#### 数据库硬件配置指导方针

开源数据库 MySQL 更加适合运行在 2U X86 服务器上，CPU 核数不宜过多，主频高低视是否有大量数据计算或批量数据计算而定；本地盘优先选择配置至少12块 600G SAS 15000转的机械盘，其次可选择 Intel SSD 固态硬盘，无须挂载存储设备；物理服务器内存优先推荐 256GB，其次推荐配置 512GB。

分布式数据库中间件的物理服务器优先选择 1U X86 服务器，需要购置 CPU 核数越多越佳，至少配置2颗 CPU 共计16核，主频要求不高；内存需求不大推荐 32GB 或 64GB；若无大量数据导出的功能，对硬盘无任何要求。

#### 环境规范标准化与自动化
私有云数据库的运行环境，指分布式数据库中间件服务器和 MySQL 数据库服务器的环境要做到标准统一，尤其是 MySQL 数据库服务器的硬件配置及设置、磁盘空间划分、操作系统、文件系统、访问权限、安装路径、配置参数、数据库账号密码、数据库命名等要做到规范标准，再借助私有云数据库的分布式数据库管理平台实现自动化，达到实现数据存储和数据服务的分布式化后，不会导致运维维护成本增加，且不提高运维技术门槛。

#### 私有云数据库替代集中式数据库的关注点

互联网化业务的信息系统，存在海量数据、海量用户和高并发的技术特征，易导致后台数据库端的资源成为瓶颈，为此我们需要放弃集中式数据库中常用的存储过程、自定义函数、触发器，这样也更有利于发挥私有云数据库的水平扩展性。同时，分布式计算环境对视图和子查询的优化技术还待完善，故也建议研发工程师和数据架构设计师进行规避。

<img src="http://ipad-cms.csdn.net/cms/attachment/201609/57c640b84062b.jpg" alt="" title="" />