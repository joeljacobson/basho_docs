---
title: Bitcask
project: riak
version: 1.4.2+
document: tutorials
toc: true
audience: intermediate
keywords: [backends, planning, bitcask]
prev: "[[Choosing a Backend]]"
up:   "[[Choosing a Backend]]"
next: "[[LevelDB]]"
interest: false
---

## 概览

[Bitcask](https://github.com/basho/bitcask) 是一个 Erlang 程序，提供了 API 把键值对
存储在日志结构的哈希表中，速度非常快。其设计借鉴了很多日志结构文件系统的原则，也受到了日志
文件合并的启发。

### 优点

  * 各条目读写时迟延很低

    因为 Bitcask 数据库文件“只写入一次，而后附加”的特性

  * 高吞吐量，特别是把随机的条目写入输入流中时

    因为要写入的数据不需要在硬盘上排序，还因为日志结构的设计在写入时只需要最少得磁头移动，
    磁头移动往往会用尽硬盘的 IO 和带宽

  * 无需降级即可处理比 RAM 更大的数据集

    因为在 Bitcask 中获取数据时是直接查看内存中的哈希表来查找硬盘上的数据的，这种方式效率
    很高，即使数据集很大也不怕

  * 读取任何数据只需单次寻道

    Bitcask 在内存中保存的键哈希表直接指明了数据在硬盘上的位置，因此读取数据时磁盘寻道
    从来不会超过一次，某些情况下，因为操作系统的文件系统有缓存，甚至一次都不用

  * 可预测的查询和插入性能

    从上面的说明可以看出，读取操作的方式是固定可预测的。你可能没发现，这种特性对写入操作
    同样有效。写入操作只要找到当前打开文件的末尾，把数据附加其后即可

  * 快速、有限制的恢复

    因为“只写入一次，而后附加”的特性，恢复的过程异常简单而且快速。唯一会丢失的数据是上次
    打开文件写入末尾的记录。恢复时只要检查最后一到两个记录仪，确认 CRC 数据保证数据的一致性

  * 备份简单

    在大多数系统中备份都很复杂，得益于“只写入一次，而后附加”，Bitcask 简化了这个过程。
    任何能按照硬盘上的顺序存储或复制文件的工具都可以备份或复制 Bitcask 数据库

### 缺点

  * 键必须能装入内存

    Bitcask 总是把所有的键都保存在内存中，因此系统必须有足够大的内存才能放得下全部的键，
    而且还要有余量处理其他操作和常驻内存的文件系统缓冲。

## 安装 Bitcask

Riak 中就包含了 Bitcask。其实 Bitcask 是默认的存储引擎，无需再安装。

`app.config` 中针对 Bitcask 的默认设置如下：

```erlang
 %% Bitcask Config
 {bitcask, [
             {data_root, "/var/lib/riak/bitcask"}
           ]},
```

## 设置 Bitcask

要想修改 Bitcask 的默认行为，请把下面的设置加入 [[app.config|Configuration Files]] 文件
的 `bitcask` 区。

### 打开超时

`open_timeout` 设置指定在尝试创建或打开数据文件夹时 Bitcask 允许使用的最长时间，单位
为秒，默认值是 `4`。一般来说无需修改这个值。如果由于某些原因导致超时了，会在日志中看到
消息：`"Failed to start bitcask backend: ...`。这时才需要设置一个较长的超时值。

```erlang
{bitcask, [
        ...,
            {open_timeout, 4} %% Wait time to open a keydir (in seconds)
        ...
]}
```

### 同步策略

`sync_strategy` 设置何时把数据同步到硬盘来控制写入的持久性。默认的设置可以避免因应用程序
出错导致数据丢失，但却有可能因为系统出错（例如 硬件问题，OS 问题，或者停电）导致其中的数据丢失。

默认的设置是 `none`，即在操作系统刷新缓存时，会把其中的数据写入硬盘。如果在刷新之前系统出
问题了（停电，损毁等），数据就会丢失。

设置为 `o_sync` 就可以强制每次写入缓冲后都摆数据存入硬盘。这样可以得到更好地持久性，不过写
操作的吞吐量会增加，因为每次写入都要等待完全写入硬盘。

___可用的同步策略___

* `none` - （默认值）交由操作系统管理同步写操作
* `o_sync` - 使用 O_SYNC 旗标，强制每次写操作都要同步
* `{seconds, N}` - Riak 会强制 Bitcask 每 `N` 秒同步一次

```erlang
{bitcask, [
        ...,
            {sync_strategy, none}, %% Let the O/S decide when to flush to disk
        ...
]}
```

<div class="note">
<div class="title">在 Linux 中 Bitcask 并不会真的设置 O_SYNC</div>
<p>在写这篇文档时，因为 Linux 有一个<a
href="http://permalink.gmane.org/gmane.linux.kernel/1123952">内核问题</a>牵涉到 <a
href="https://github.com/torvalds/linux/blob/master/fs/fcntl.c#L146..L198"><code>fcntl</code> 的实现</a>，所以 Bitcask 并不会在写入的文件上设置 <code>O_SYNC</code> 旗标，但调用 <code>fcntl</code> 也不会失败，因为这个旗标直接被 Linux 内核忽略了。如果在日志文件中看到一个<a
href="https://github.com/basho/riak_kv/commit/6a29591ecd9da73e27223a1a55acd80c21d4d17f#src/riak_kv_bitcask_backend.erl">警告消息</a>，形式如下：<br />
<code>{sync_strategy,o_sync} not implemented on Linux</code><br />
就说明你所用的系统存在这个问题。如果没设置 <code>O_SYNC</code> 旗标，一旦系统损坏，没有写入硬盘的缓冲数据就有可能丢失。
</div>

### 硬盘使用和合并设置

Riak K/V 会把环中每个虚拟节点分别存储在 Bitcask 数据文件夹中的独立子文件夹中。
每个子文件夹都有很多文件用来存储键值对数据；一个或多个“提示”文件，记录每个键
保存在哪个数据文件中了还有一个写锁定文件。Bitcask 的设计方式允许即便没有完全把
数据同步到硬盘，仍能进行恢复操作。这种特性是得益于数据文件只能附加，无法重新
打开修改。（只能读取）

这种数据管理方式为了操作性能会丧失一些硬盘空间，有很大一部分空间不会用来存储
工作数据，但可以执行其他的操作减少使用量。简单来说，在达到每个极限值之前，硬盘
空间可以一直正常使用，但超过这个值之后，就可以通过合并释放一些空间。合并时，
遍历所有数据文件，删除过时的键值对，然后把当前版本的键值对写入一系列新的文件中。

合并过程会收到下面介绍的设置影响。在下面的讨论中，“死亡”的意思是键对应的值
不是最新版本，要删除掉；“新鲜”表示键对应的值是最新的，不会被删除。

#### 文件大小最大值

`max_file_size` 设定在 Bitcask 文件夹中可以保存的单个数据文件大小最大值。如果
写入数据时超过了这个值，该文件会被关闭，打开一个新文件继续写入。

增加 `max_file_size` 的值，Bitcask 会创建更少更大的文件，合并次数会少一些；
而减小这个值 Bitcask 会创建更多更小的文件，合并次数就会多一些。如果环的大小
为 16，服务器就能处理 32GB 的数据，如果超过这个大小就会进行第一次合并，而不管
数据集的大小。请相应的规划存储，不要担心会超过硬盘中数据集的大小。

默认值是 `16#80000000`，即 2GB

```erlang
{bitcask, [
        ...,
        {max_file_size, 16#80000000}, %% 2GB default
        ...
]}
```

#### 合并时段

`merge_window` 设定一天中的哪个时段允许进行合并操作。可选的值有：

* `always`：（默认值）不限制
* `never`：不要进行合并
* `{Start, End}`：进行合并的时间区间，`Start` 和 `End` 都可以取 0 到 23 之间的整数

如果合并会严重影响集群的整体性能，或者有某个时间段存储活动量很少，那么就可以
修改默认值。

默认值是 `always`

```erlang
{bitcask, [
        ...,
            {merge_window, always}, %% Span of hours during which merge is acceptable.
        ...
]}
```

<div class="note">
<div class="title">`merge_window` 和 Multi 后台</div>
在 [[Multi 后台|Multi]]中使用 Bitcask 时，如果要设置合并时段，必须在全局 <code>app.config</code> 文件的 <code>bitcask</code> 区中设置，针对各后台的设置区中的 `merge_window` 会被忽略。
</div>

#### 合并触发条件

合并触发条件设置遇到什么样的情况时会执行合并操作。

*   _碎片_：`frag_merge_trigger` 设置文件中死亡键和总键数的比例达到多少
    时进行合并。这个设置的值是个百分比（0-100）。例如，如果数据文件中有 6 个
    死亡键，4 个新鲜键，那么在默认设置时就会触发合并操作。增加这个值合并次数就
    少一点，减少这个值合并次数就多一些。

    默认值：`60`

*   _死亡字节_：`dead_bytes_merge_trigger` 设置单个文件中存储了多少死亡数据
    时触发合并，单位为字节。如果文件到达或超过这个值，就会触发合并。增加这个值
    合并次数就少一点，减少这个值合并次数就多一些。

    只要数据文件夹中的任何一个文件达到了这个要求，Bitcask 就会尝试合并文件。

    默认值：`536870912`，即 512MB

```erlang
{bitcask, [
        ...,
        %% Trigger a merge if any of the following are true:
            {frag_merge_trigger, 60}, %% fragmentation >= 60%
        {dead_bytes_merge_trigger, 536870912}, %% dead bytes > 512 MB
        ...
]}
```

#### 合并范围

合并范围设定哪些文件会包含在合并操作中。

-   _碎片_：`frag_threshold` 设置文件中死亡键和总键数的比例达到多少
    时，该文件会包含在此次合并操作中。这个设置的值是百分比（0-100）。例如，
    一个文件中有 4 个死亡键和 6 个新鲜键，那么在默认设置时，这个文件就会包含
    在合并操作中。增加这个值合并的文件数会少一点，减少这个值合并的文件就会
    多一些。

    默认值：`40`

-   _死亡字节_：`dead_bytes_threshold` 设置死亡键至少占用了多少数据量时，该文件
    会包含在此次合并中。增加这个值合并的文件数会少一点，减少这个值合并的文件就会
    多一些。

    默认值：`134217728`，即 128MB

-   _小型文件_：`small_file_threshold` 设置文件的大小大于多少时才不会包含在
    合并操作中。小于设定值的文件会包含在合并操作中。增加这个值合并的文件数会
    多一些，减少这个值合并的文件就会少一点。

    默认值：`10485760`，即 10MB

当单个文件满足上述任何一个限制时，就会包含在合并操作中：

```erlang
{bitcask, [
        ...,
        %% Conditions that determine if a file will be examined during a merge:
            {frag_threshold, 40}, %% fragmentation >= 40%
        {dead_bytes_threshold, 134217728}, %% dead bytes > 128 MB
        {small_file_threshold, 10485760}, %% file is < 10MB
        ...
]}
```

<div class="note">
<div class="title">选择阈值</div>
<p><code>frag_threshold</code> 和 <code>dead_bytes_threshold</code> 的值必须
小于或等于相应的触发值。如果设高了，就没有文件能满足这些限制条件，也就不会触发
合并操作。</p>
</div>

#### 折叠键阈值

如果另一个折叠在 `max_fold_age` 之前开始，而且有不超过 `max_fold_puts` 个更新，那么
折叠键会重用键目录。否则就会等到所有当前的折叠键完成后再开始。把前面这两个设置设为 -1 则
完全禁止折叠键。

```erlang
{bitcask, [
        ...,
            {max_fold_age, -1}, %% Age in micro seconds (-1 means "unlimited")
        {max_fold_puts, 0}, %% Maximum number of updates
        ...
]}
```

#### 自动过期

默认情况下，Bitcask 会保存所有数据。如果数据具有时效性，或者空间有限需要清除数据，那么就
可以设定 `expiry_secs` 选项。如果要在一天后自动清除数据，请将其设为 `86400`。

默认值：`-1`，即禁用自动过期

```erlang
{bitcask, [
        ...,
        {expiry_secs, -1}, %% Don't expire items based on time
        ...
]}
```

<div class="note">
<p>被过期数据占用的空间可能不会立马就能使用，但这些数据立即就无法请求。向键写入新的数据会
修改时间戳，因此不会过期。</p>
</div>

{{#1.2.1+}}

默认情况下，只要数据文件包含过期的键就会触发合并操作。某些情况下，会导致过多的合并。
为了避免发生这样的事，可以设置 `expiry_grace_time` 选项。如果只是由于键过期，Bitcask 会
延迟触发合并所设置的秒数。设置为 `3600` 可以有效限制每个桶每小时只进行一次合并。

默认值：`0`

```erlang
{bitcask, [
        ...,
        {expiry_grace_time, 3600}, %% Limit rate of expiry merging
        ...
]}
```

{{/1.2.1+}}

## 调整 Bitcask

Bitcask 有很多令人满意的功能，在生产环境中已经证实其很稳定、很可靠，迟延小，吞吐量大。

### 提示和技巧

  * __Bitcask 依赖于文件系统的缓存__

    某些数据存储层把页缓冲和块缓冲放在内存中，但 Bitcask 不是这样。Bitcask 依赖于文件系统
    的缓存。调整文件系统的缓存可以影响 Bitcask 的性能。

  * __注意文件句柄限制__

    请阅读“[[打开文件限制|Open-Files-Limit]]”一文了解详情。

  * __避免每次读写操作更新文件元数据的开销（例如上次访问时间）__

    如果把 `noatime` 挂载选项加入 `/etc/fstab` 可以提速不少。设置后，所有文件都不会记录
    “上次访问时间”，因此减少了磁头寻道时间。如果需要记录上次访问时间，但又想从这种优化措施
    中受益，可以试一下 `relatime`。

    ```
    /dev/sda5    /data           ext3    noatime  1 1
    /dev/sdb1    /data/inno-log  ext3    noatime  1 2
    ```

  * __不要频繁修改键__

    如果频繁修改键，分段数量会显著增多。为了避免这种情况，应该把片段触发和阈值设的小一点。

  * __限制空间用量__

    如果限制了空间用量，控制死亡键占用的空间就显得尤为重要了。为了避免浪费空间，可以把死亡
    键字节阈值和触发阈值设的小一点。

  * __定期清除过期数据__

    如果要自动清除过期数据，请设置一个所需的 `expiry_secs` 值。
    等于或超过 `expiry_secs` 时间没有修改的键就无法访问了。

  * __各节点的分区数多一些__

    集群中分区越多，Bitcask 能[[打开的文件|Open-Files-Limit]]就越多。如果要减少打开文件
    数，可以增加 `max_file_size`，写入更大的文件。还可以减少分段和死亡键设置，并
    增加 `small_file_threshold`，这样合并操作会把打开文件数保持在一个很低的水平上。

  * __白天流量多，晚上流量少__

    为了复制大量写入卷而不降低性能，应该避免在高峰期进行合并操作。尽量把 `merge_window` 的
    值设在一天中流量很低的时段。

  * __多集群副本（Riak Enterprise）__

    如果 Riak Enterprise 激活了副本功能，集群可能会由于“重演”（replay）产生大量分段和
    死亡字节。而且，因为完整同步会操作全部分区，尽量连续的访问数据（在较少的文件之间）可以
    提高效率。减少分段和死亡字节设置可以提高性能。

## FAQ

  * [[为什么好像只有 Riak 节点重启时 Bitcask 才会进行合并？|Developing on Riak FAQs#why-does-it-seem-that-bitc]]
  * [[如果键索引超出了内存大小，Bitcask 会怎么办？|Operating Riak FAQs#if-the-size-of-key-index-e]]
  * [[Bitcask 容量规划|Bitcask Capacity Planning]]

## Bitcask 的实现细节

Riak 会为使用 Bitcask 数据库的虚拟节点创建一个文件夹。在每个文件夹中，一次最多只有一个
数据库文件打开接受数据写入。写入的这个文件会一直保持打开只到其大小超过了某个值，然后会被
关闭，系统再创建一个新文件接受写入操作。只要文件被关闭了，不管是有意的还是因为服务器出问题了，
这个文件就无法再被修改，无法再次打开接受写入操作了。

当前打开接受写入操作的文件只能把数据附加到文件末尾，也就是说，连续的写入不会明显增加
硬盘的 IO 使用量。注意，如果文件系统启用了 `atime`，可能会破坏写入操作，因为硬盘磁头要在
来回移动更新数据块和文件及文件夹的元数据块。基于日志的数据块其主要优势是可以最小化硬盘磁头的
寻道时间。从 Bitcask 中删除数据分两步。首先，我们在打开的文件末尾加入一个“墓碑”记录，把值
标记为删除。同时，我们从内存中的 `keydir` 中删除这个键的引用。

合并时，只会扫描不活动的数据文件，而且只有那些不是“墓碑”的值才会合并到活动的数据文件中。
这样可以很高效的删除过期数据，并将其占用的空间释放出来。这种数据管理方式久而久之便会用掉
很多空间，因为只写入新数据，而不动旧数据。一个我们称之为“合并”的压缩过程可以避免这个问题。
合并操作会遍历 Bitcask 中所有不活动的文件，生成的文件中只包含新鲜数据，即每个键对应最新
版本的值。

### Bitcask 数据库文件

下面列出了两个文件夹，其内容就是使用 Bitcask 时可以在硬盘上看到的。在这个例子中，我们使用
的是有 64 个分区的环，所以有 64 个文件夹，各自对应到自己的 Bitcask 数据库。

```
bitcask/
|-- 0-131678707860265
|-- 1004782375664995756265033322492444576013453623296-1316787078215038
|-- 1027618338748291114361965898003636498195577569280-1316787078218073

... etc ...

`-- 981946412581700398168100746981252653831329677312-1316787078206121
```

注意，节点刚启动时只会为每个虚拟节点分区创建保存数据的文件夹，不会有 Bitcask 相关的文件。

在使用 Bitcask 的 Riak 集群中执行一次PUT 请求（写操作）

```
curl http://localhost:8098/riak/test/test -XPUT -d 'hello' -H 'content-type: text/plain'
```

这个集群的 N 值是 3，所以你会看到有 3 个虚拟节点的分区响应了这个请求，这是就创建
了 Bitcask 数据库文件。

```
bitcask/

... etc ...

|-- 1118962191081472546749696200048404186924073353216-1316787078245894
|   |-- 1316787252.bitcask.data
|   |-- 1316787252.bitcask.hint
|   `-- bitcask.write.lock

... etc ...


|-- 1141798154164767904846628775559596109106197299200-1316787078249065
|   |-- 1316787252.bitcask.data
|   |-- 1316787252.bitcask.hint
|   `-- bitcask.write.lock

... etc ...


|-- 1164634117248063262943561351070788031288321245184-1316787078254833
|   |-- 1316787252.bitcask.data
|   |-- 1316787252.bitcask.hint
|   `-- bitcask.write.lock

... etc ...

```

随着数据不断的写入集群，Bitcask 文件的数量会不断增多，只到触发合并操作。

```
bitcask/
|-- 0-1317147619996589
|   |-- 1317147974.bitcask.data
|   |-- 1317147974.bitcask.hint
|   |-- 1317221578.bitcask.data
|   |-- 1317221578.bitcask.hint
|   |-- 1317221869.bitcask.data
|   |-- 1317221869.bitcask.hint
|   |-- 1317222847.bitcask.data
|   |-- 1317222847.bitcask.hint
|   |-- 1317222868.bitcask.data
|   |-- 1317222868.bitcask.hint
|   |-- 1317223014.bitcask.data
|   `-- 1317223014.bitcask.hint
|-- 1004782375664995756265033322492444576013453623296-1317147628760580
|   |-- 1317147693.bitcask.data
|   |-- 1317147693.bitcask.hint
|   |-- 1317222035.bitcask.data
|   |-- 1317222035.bitcask.hint
|   |-- 1317222514.bitcask.data
|   |-- 1317222514.bitcask.hint
|   |-- 1317223035.bitcask.data
|   |-- 1317223035.bitcask.hint
|   |-- 1317223411.bitcask.data
|   `-- 1317223411.bitcask.hint
|-- 1027618338748291114361965898003636498195577569280-1317223690337865
|-- 1050454301831586472458898473514828420377701515264-1317223690151365

... etc ...

```

以上是 Bitcask 的常见表现。