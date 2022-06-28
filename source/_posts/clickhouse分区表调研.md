---
title: clickhouse分区表调研
date: 2022/06/28 17:24:27
---

# clickhouse分区表调研


<!-- toc -->


## 1.搭建本地环境
本次clickhouse的环境搭建在docker中进行

windows或者mac系统均需要先安装好docker
### 1.1.搜索镜像是否存在

*window系统打开cmd，mac系统在命令行里直接敲入*

```shell
docker search clickhouse
```

**搜索结果**

```shell
C:\Users\Pengyu.Ren>docker search clickhouse
NAME                                  DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
yandex/clickhouse-server              ClickHouse is an open-source column-oriented…   365                  [OK]
clickhouse/clickhouse-server          ClickHouse is an open-source column-oriented…   57
clickhouse/kerberos-kdc                                                               1
antrea/clickhouse-server                                                              1
antrea/clickhouse-operator                                                            0
clickhouse/integration-helper                                                         0
clickhouse/python-bottle                                                              0
clickhouse/dotnet-client              ClickHouse.NET client for ClickHouse integra…   0
clickhouse/s3-proxy                                                                   0
clickhouse/stateless-test                                                             0
clickhouse/stateful-test                                                              0
clickhouse/unit-test                                                                  0
clickhouse/postgresql-java-client                                                     0
clickhouse/binary-builder                                                             0
clickhouse/integration-test                                                           0
clickhouse/mysql-java-client                                                          0
clickhouse/mysql-php-client                                                           0
clickhouse/deb-builder                                                                0
clickhouse/mysql-golang-client                                                        0
clickhouse/mysql-js-client                                                            0
clickhouse/stress-test                                                                0
clickhouse/integration-tests-runner                                                   0
clickhouse/fuzzer                                                                     0
clickhouse/performance-comparison                                                     0
clickhouse/kerberized-hadoop                                                          0

C:\Users\Pengyu.Ren>
```

### 1.2.下载镜像

*window系统打开cmd，mac系统在命令行里直接敲入*

```shell
docker pull yandex/clickhouse-server
# 如果需要下载指定版本的clickhouse镜像，可在后面加上版本号，如：
docker pull yandex/clickhouse-server:21.3.20
```

### 1.3.运行容器

*window系统打开cmd，mac系统在命令行里直接敲入*

```shell
docker run -d --name clickhouse-server --ulimit nofile=262144:262144 -p 8123:8123 -p 9000:9000 -p 9009:9009 yandex/clickhouse-server
```

启动成功后，在docker desktop中可以看到新启动的clickhouse-server

![](https://s3.bmp.ovh/imgs/2022/06/08/9a72046d38ff737d.png)

### 1.4.修改密码

#### 1.4.1.进入容器

要修改clickhouse-server的默认密码，需要先进入容器，使用如下命令：

```shell
docker exec -it clickhouse-server /bin/bash

# sample
C:\Users\Pengyu.Ren>docker exec -it clickhouse-server /bin/bash
root@77720e63254e:/#
```

进入容器后，进入/etc/clickhouse-server/users.xml

```shell
vim /etc/clickhouse-server/users.xml
```

**如果提示容器内没有编辑器vim，则需要我们手动安装**

```shell
# 先后执行如下两条命令
apt-get update
apt-get install vim -y

apt-get update，这个命令的作用是：同步 /etc/apt/sources.list 和 /etc/apt/sources.list.d 中列出的源的索引，这样才能获取到最新的软件包
```

#### 1.4.2.生成自定义密码密文

指令及执行结果如下：

```shell
PASSWORD=$(base64 < /dev/urandom | head -c8); echo "123456"; echo -n "123456" | sha256sum | tr -d '-'
```

```shell
root@77720e63254e:/# PASSWORD=$(base64 < /dev/urandom | head -c8); echo "123456"; echo -n "123456" | sha256sum | tr -d '-'
123456
8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
```

123456是明文

8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92是密文

由于官方不建议直接写明文密码，因此我们在这里讲密码修改为密文密码；

继续刚才打开的vim /etc/clickhouse-server/users.xml

默认新安装的，里面有一条<password>,这个标签是设置明文密码的，**我们将这个标签改为<password_sha256_hex>**

```xml
<!-- 明文123456加密后的密文如下 -->
<password_sha256_hex>8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92</password_sha256_hex>
```

#### 1.4.3.重启clickhouse-server

mac系统可使用如下指令

```shell
systemctl restart clickhouse-server
```

如使用docker desktop可点击容器后面的restart按钮进行重启即可

### 1.5.连接clickhouse

本次连接使用了DBeaver工具

在DBeaver中新建连接，类型选择clickhouse

![](https://s3.bmp.ovh/imgs/2022/06/08/a28a748b11c2d742.png)

点击“下一步”，在地址栏中填入localhost，如果docker假设在本地的话；

用户名default

密码就是刚刚设置的密文密码对应的明文，我这里设置的是123456；

检查驱动可以自动下载所需的驱动，需要之前配好过maven仓库和环境变量；

![](https://s3.bmp.ovh/imgs/2022/06/08/99846864b83458d6.png)

填写完成，并且下载好驱动之后，点击“测试连接”，测试成功会弹出连接成功和clickhouse-server的版本信息

![](https://s3.bmp.ovh/imgs/2022/06/08/fad34bf958cc889f.png)

最后点击“完成”，客户端连接的配置工作就完成了。

### 1.6.测试本地环境

新建sql窗口，执行如下：

```sql
SELECT version();
```

![](https://s3.bmp.ovh/imgs/2022/06/08/4294966bcfc3ca73.png)

得到版本信息，说明测试通过。

## 2.分区表

### 2.1.概念

分区是**逻辑分区**的概念。

> 分区是在一个表中通过指定的规则划分而成的逻辑数据集。可以按任意标准进行分区，如按月，按日或按事件类型。为了减少需要操作的数据，每个分区都是分开存储的。访问数据时，ClickHouse 尽量使用这些分区的最小子集。
>
> 分区是在 建表 时通过 **PARTITION BY expr** 子句指定的。

### 2.2.作用

> 是降低扫描的范围，优化查询速度；

### 2.3.分区键的表现形式

1. 分区键可以是表中列的任意表达式。sample：指定按月分区，表达式为 toYYYYMM(date_column)：

   ```sql
   CREATE TABLE visits
   (
       VisitDate Date,
       Hour UInt8,
       ClientID UUID
   )
   ENGINE = MergeTree()
   PARTITION BY toYYYYMM(VisitDate)
   ORDER BY Hour;
   ```

2. 分区键也可以是表达式元组（类似 主键 ）。sample：

   ```sql
   ENGINE = ReplicatedCollapsingMergeTree('/clickhouse/tables/name', 'replica1', Sign)
   PARTITION BY (toMonday(StartDate), EventType)
   ORDER BY (CounterID, StartDate, intHash32(UserID));
   ```

### 2.4.分区键的使用注意事项

官方文档建议，不要用太精细的分区方案（超过一千个分区）。否则，会因为文件系统中的文件数量过多和需要打开的文件描述符过多，导致 SELECT 查询效率不佳。

#### 2.4.1.如何查询一张表的分区情况

```sql
SELECT
    partition,
    name,
    active
FROM system.parts
WHERE table = 'visits'
```

这里我们用一个现存的例子来看，这张表是dev环境中的一张m241的表，表字段较多，因此做了省略，只保留了几个关键字段；

```sql
-- 建表
CREATE TABLE dwd_tsp_m241_newenergy_hi
(
    `dt` String COMMENT 'odps表分区字段',
    `time` String COMMENT '毫秒时间戳',
    `second_time` String COMMENT '秒时间戳',
    `date_time_str` DateTime COMMENT '基于time算出的yyyy-mm-dd hh:mm:ss',
    `date_hour` String COMMENT '基于time算出的分区：yyyymmddhh',
    `vin` String,
    `configid` String,
    `sequenceid` String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(date_time_str)
ORDER BY (date_time_str,vin);

-- 初始化数据
INSERT INTO vdp.dwd_tsp_m241_newenergy_hi (dt,`time`,second_time,date_time_str,date_hour,vin,configid,sequenceid) 
VALUES ('12345678','1654672961748','1654672962','2022-06-08 15:23:00','2022060815','L6T79P2N5MP009118','newenergy','178');
INSERT INTO vdp.dwd_tsp_m241_newenergy_hi (dt,`time`,second_time,date_time_str,date_hour,vin,configid,sequenceid) 
VALUES ('12345678','1654672961748','1654672962','2022-06-08 15:23:00','2022060815','L6T79P2N5MP009118','newenergy','178');
INSERT INTO vdp.dwd_tsp_m241_newenergy_hi (dt,`time`,second_time,date_time_str,date_hour,vin,configid,sequenceid) 
VALUES ('12345678','1654672961750','1654672962','2022-06-08 15:23:00','2022060815','L6T79P2N5MP009119','newenergy','178');
INSERT INTO vdp.dwd_tsp_m241_newenergy_hi (dt,`time`,second_time,date_time_str,date_hour,vin,configid,sequenceid) 
VALUES ('12345678','1654672961750','1654672962','2022-06-08 15:23:00','2022060815','L6T79P2N5MP009119','newenergy','178');
INSERT INTO vdp.dwd_tsp_m241_newenergy_hi (dt,`time`,second_time,date_time_str,date_hour,vin,configid,sequenceid) 
VALUES ('12345678','1654672961750','1654672962','2022-06-09 15:23:00','2022060815','L6T79P2N5MP009119','newenergy','178');
INSERT INTO vdp.dwd_tsp_m241_newenergy_hi (dt,`time`,second_time,date_time_str,date_hour,vin,configid,sequenceid) 
VALUES ('12345678','1654672961750','1654672962','2022-06-09 15:23:00','2022060815','L6T79P2N5MP009119','newenergy','178');

-- 查询分片情况
SELECT partition, name, active FROM system.parts WHERE table = 'dwd_tsp_m241_newenergy_hi';
```

可以查询得到如下的分片结果：

```sql
partition|name          |active|
---------|--------------|------|
20220608 |20220608_1_1_0|     1|
20220608 |20220608_2_2_0|     1|
20220608 |20220608_3_3_0|     1|
20220608 |20220608_4_4_0|     1|
20220609 |20220609_5_5_0|     1|
20220609 |20220609_6_6_0|     1|
```

因为建表时使用的分片键是PARTITION BY toYYYYMMDD(date_time_str)，精确到年月日，初始化数据中存在20220608和20220609的数据，因此存在2个partition；

20220608_1_1_0这个命名的规则如下：

- 20220608是分区的名称；
- 第一个1是数据块的最小编号；
- 第二个1是数据块的最大编号；
- 第三个0是块级别(即在由块组成的合并树中，该块在树中的深度);

**active** 列为片段状态。1 代表激活状态；0 代表非激活状态。非激活片段是那些在合并到较大片段之后剩余的源数据片段。损坏的数据片段也表示为非活动状态。

再次执行如下语句

```sql
INSERT INTO vdp.dwd_tsp_m241_newenergy_hi (dt,`time`,second_time,date_time_str,date_hour,vin,configid,sequenceid) 
VALUES ('12345678','1654672966666','1654372962','2022-06-08 15:23:00','2022060815','L6T79P2N5MP009120','newenergy','178');

OPTIMIZE TABLE dwd_tsp_m241_newenergy_hi PARTITION 20220608;

SELECT partition, name, active FROM system.parts WHERE table = 'dwd_tsp_m241_newenergy_hi';
```

可以得到新的分片在合并之后的结果

```sql
partition|name          |active|
---------|--------------|------|
20220608 |20220608_1_1_0|     0|
20220608 |20220608_1_7_1|     1|
20220608 |20220608_2_2_0|     0|
20220608 |20220608_3_3_0|     0|
20220608 |20220608_4_4_0|     0|
20220608 |20220608_7_7_0|     0|
20220609 |20220609_5_5_0|     1|
20220609 |20220609_6_6_0|     1|
```

这里可以看到，相同计算分片键结果的数据块进行了合并；

还可以通过进入表目录的方式查询分片的结果:

```shell
root@77720e63254e:/var/lib/clickhouse/data/vdp/dwd_tsp_m241_newenergy_hi# ll
total 48
drwxr-x--- 11 clickhouse clickhouse 4096 Jun  8 07:34 ./
drwxr-x---  3 clickhouse clickhouse 4096 Jun  8 07:07 ../
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:24 20220608_1_1_0/
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:34 20220608_1_7_1/
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:28 20220608_2_2_0/
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:28 20220608_3_3_0/
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:28 20220608_4_4_0/
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:34 20220608_7_7_0/
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:30 20220609_5_5_0/
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:30 20220609_6_6_0/
drwxr-x---  2 clickhouse clickhouse 4096 Jun  8 07:07 detached/
-rw-r-----  1 clickhouse clickhouse    1 Jun  8 07:07 format_version.txt
root@77720e63254e:/var/lib/clickhouse/data/vdp/dwd_tsp_m241_newenergy_hi# pwd
/var/lib/clickhouse/data/vdp/dwd_tsp_m241_newenergy_hi
```

#### 2.4.2.数据写入与分区合并

任何一个批次的数据写入都会产生一个**临时分区**，不会纳入任何一个已有的分区；
写入后的某个时刻（大概 10-15 分钟后），ClickHouse 会自动执行合并操作（等不及也可以手动通过 optimize 执行），把临时分区的数据，合并到已有分区中；

**非激活片段(active为0)**会在合并后的10分钟左右被**删除**。

上文中的例子，过一段时间再次查询分区情况可得：

```sql
partition|name          |active|
---------|--------------|------|
20220608 |20220608_1_7_1|     1|
20220609 |20220609_5_5_0|     1|
20220609 |20220609_6_6_0|     1|
```



## 3.分布式表

### 3.1.概念

物理分片，数据经过路由，存储到不同的节点上；每个节点上存储部分的数据，分而治之的思想；

### 3.2.作用

根本目的是解决单机数据量太大的问题；

### 3.3.实现原理

clickhouse中，是通过水平切分的方式，将完整的数据切分成不同的分片，这些分片只保存一部分数据，分布在不同的节点上，通过Distributed表引擎来将数据拼成一个完整的表来使用的。Distributed表本身不存储数据，但是能作为查询的表来使用。

在这里，将distributed表称为**分布式表**，将实际存储数据的节点上的表叫做**数据表**，每一个片区称为一个**shard**，每个片区的权重叫**weight**，每个片区的节点（副本）叫**replica**，以便于后续的阐述区分。

负载的策略具体通过**users.xml**配置进行实现

```xml
    <!-- Profiles of settings. -->
    <profiles>
        <!-- Default settings. -->
        <default>
            <!-- Maximum memory usage for processing single query, in bytes. -->
            <!-- 单次查询的最大内存，默认10000000000，约9.3gb -->
            <max_memory_usage>10000000000</max_memory_usage>

            <!-- How to choose between replicas during distributed query processing.
                这里有几种负载均衡的策略
                 random - choose random replica from set of replicas with minimum number of errors
                随机策略，但是随机的基础是错误最少的几个节点之中随机
                 nearest_hostname - from set of replicas with minimum number of errors, choose replica
                  with minimum number of different symbols between replica's hostname and local hostname
                  (Hamming distance).
                hostname最相近的，但是最小的基础是错误最少的几个节点之中
                 in_order - first live replica is chosen in specified order.
                顺序
                 first_or_random - if first replica one has higher number of errors, pick a random one from replicas with minimum number of errors.
                基本上可以理解为还是找最低错误次数的节点
            -->
            <load_balancing>random</load_balancing>
        </default>

        <!-- Profile that allows only read queries. -->
        <readonly>
            <readonly>1</readonly>
        </readonly>
    </profiles>
```



集群的配置通过**config.xml**实现

以下是clickhouse-server默认自带的配置

```xml
<remote_servers>
    <!-- Test only shard config for testing distributed storage -->
    <test_shard_localhost>
        <!-- Inter-server per-cluster secret for Distributed queries
             default: no secret (no authentication will be performed)

             If set, then Distributed queries will be validated on shards, so at least:
             - such cluster should exist on the shard,
             - such cluster should have the same secret.

             And also (and which is more important), the initial_user will
             be used as current user for the query.

             Right now the protocol is pretty simple and it only takes into account:
             - cluster name
             - query

             Also it will be nice if the following will be implemented:
             - source hostname (see interserver_http_host), but then it will depends from DNS,
               it can use IP address instead, but then the you need to get correct on the initiator node.
             - target hostname / ip address (same notes as for source hostname)
             - time-based security tokens
        -->
        <!-- <secret></secret> -->

        <shard>
            <!-- Optional. Whether to write data to just one of the replicas. Default: false (write data to all replicas). -->
            <!-- <internal_replication>false</internal_replication> -->
            <!-- Optional. Shard weight when writing data. Default: 1. -->
            <!-- <weight>1</weight> -->
            <replica>
                <host>localhost</host>
                <port>9000</port>
                <!-- Optional. Priority of the replica for load_balancing. Default: 1 (less value has more priority). -->
                <!-- <priority>1</priority> -->
            </replica>
        </shard>
    </test_shard_localhost>
    <test_cluster_one_shard_three_replicas_localhost>
        <shard>
            <internal_replication>false</internal_replication>
            <replica>
                <host>127.0.0.1</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>127.0.0.2</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>127.0.0.3</host>
                <port>9000</port>
            </replica>
        </shard>
        <!--shard>
            <internal_replication>false</internal_replication>
            <replica>
                <host>127.0.0.1</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>127.0.0.2</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>127.0.0.3</host>
                <port>9000</port>
            </replica>
        </shard-->
    </test_cluster_one_shard_three_replicas_localhost>
    <test_cluster_two_shards_localhost>
         <shard>
             <replica>
                 <host>localhost</host>
                 <port>9000</port>
             </replica>
         </shard>
         <shard>
             <replica>
                 <host>localhost</host>
                 <port>9000</port>
             </replica>
         </shard>
    </test_cluster_two_shards_localhost>
    <test_cluster_two_shards>
        <shard>
            <replica>
                <host>127.0.0.1</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>127.0.0.2</host>
                <port>9000</port>
            </replica>
        </shard>
    </test_cluster_two_shards>
    <test_cluster_two_shards_internal_replication>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>127.0.0.1</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>127.0.0.2</host>
                <port>9000</port>
            </replica>
        </shard>
    </test_cluster_two_shards_internal_replication>
    <test_shard_localhost_secure>
        <shard>
            <replica>
                <host>localhost</host>
                <port>9440</port>
                <secure>1</secure>
            </replica>
        </shard>
    </test_shard_localhost_secure>
    <test_unavailable_shard>
        <shard>
            <replica>
                <host>localhost</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>localhost</host>
                <port>1</port>
            </replica>
        </shard>
    </test_unavailable_shard>
</remote_servers>
```

![](https://s3.bmp.ovh/imgs/2022/06/08/ea237d9501dc4d01.png)

#### 3.3.1 写入数据

官方提供了向集群写入数据的两种方式：

1. 由应用程序决定数据写入什么**数据表**，并直接在指定的分片上写入；查询的时候，使用**分布式表**进行查询；这样是最灵活的方案，

> 你可以使用任何分片方案，对于复杂业务特性的需求，这可能是非常重要的。 这也是最佳解决方案，因为数据可以完全独立地写入不同的分片。

2. 在**分布式表**上进行insert，这种情况下，由分布式表来进行跨服务分发数据。

> 二，在分布式表上执行 INSERT。在这种情况下，分布式表会跨服务器分发插入数据。 为了写入分布式表，必须要配置分片键（最后一个参数）。当然，如果只有一个分片，则写操作在没有分片键的情况下也能工作，因为这种情况下分片键没有意义。
>
> 每个分片都可以在配置文件中定义权重。默认情况下，权重等于1。数据依据分片权重按比例分发到分片上。例如，如果有两个分片，第一个分片的权重是9，而第二个分片的权重是10，则发送 9 / 19 的行到第一个分片， 10 / 19 的行到第二个分片。
>
> 分片可在配置文件中定义 ‘internal_replication’ 参数。**默认是false，推荐使用true。**
>
> 此参数设置为«true»时，写操作只选一个正常的副本写入数据。如果分布式表的子表是复制表(*ReplicaMergeTree)，请使用此方案。换句话说，这其实是把数据的复制工作交给实际需要写入数据的表本身而不是分布式表。
>
> 若此参数设置为«false»（默认值），写操作会将数据写入所有副本。实质上，这意味着要分布式表本身来复制数据。这种方式不如使用复制表的好，因为不会检查副本的一致性，并且随着时间的推移，副本数据可能会有些不一样。

#### 3.3.2 读取数据

通过分布式表读取数据，分布式表会自动整合各个分片的数据；

> 当查询一个`Distributed`表时，`SELECT`查询被发送到所有的分片，不管数据是如何分布在分片上的(它们可以完全随机分布)。当您添加一个新分片时，您不必将旧数据传输到它。相反，您可以使用更重的权重向其写入新数据——数据的分布会稍微不均匀，但查询将正确有效地工作。

### 3.4.语句

```sql
Distributed(cluster_name, database, table, [sharding_key]);
```

参数解析：
      cluster_name：服务器配置文件中的集群名，在/etc/metrika.xml中配置的。具体配置见前文。
      database：数据库名。
      table：表名。
      sharding_key：数据分片键。

```sql
create table dis_table(id UInt16, name String) engine=Distributed(clickhouse_cluster, default, t(数据表), id(分片键));
```

在配置好集群的remote_server配置以及负载的策略后，重启clickhouse-server服务即可生效，再执行上述语句可以实现节点同步新增表dis_table，新增数据时，只会在其中一个**shard**新增。

```shell
clickhouse-server restart
```



## 4.参考资料

https://clickhouse.com/docs/zh/getting-started/

https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/custom-partitioning-key#alter_attach-partition

https://juejin.cn/post/7073021690875215885

https://juejin.cn/post/6897418615075635214

https://blog.csdn.net/congge_study/article/details/123822299

https://www.bilibili.com/video/BV1xg411w7AP?p=11

