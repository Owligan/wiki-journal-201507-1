# 理解 Cassandra 压缩储存的作用  
## 介绍  
在 Librato，我们对时间序列的储存主要是应用了我们一直在研发的自定义架构所建立的 Apache Cassandra。关于它我们之前已经写到并呈现过几次。在 Cassandra 上我们既存储真实的时间序列也存储历史汇总时间序列。Cassandra 存储节点在我们的基础设施中占有最大的足迹因而这些节点驱动着我们的成本开支，所以我们一直在寻找方式来改进我们数据的效率。  
  
作为我们正在进行的效率改进和后台功能研发的部分，我们最近花时间重新评估了我们的存储架构。自 Cassandra 0.8.x 版本的早期以来，我们的架构一直被建立在 Thrift APIs 的内容之上，并且无论何时当我们竖起一个新环路，我们使用‘节点工具’命令来移动它。我们一直都在紧密的跟随 CQL 的发展并且已经将我们的读取路径的部分在 2.0.x 版本中移动到了新的本地接口。甚至，我们想要更进一步关注使用本地 CQL 接口完全构造我们的架构转移（创建 CQL 表，或者他们所称的“列族”）。  
  
## CQL 表存储选项  
  
先前当我们关注 CQL 表结构时困住我们的事情之一是涉及到一个 COMPACT STORAGE 选项--我将停止吐槽并且从这之后把它称作为压缩存储。其定义文档如下：  
  
“…主要将对 CQL3 之前创建的定义的反向兼容性为目标…在硬盘上提供一个更压缩紧凑一些的数据分布，但这样是以减小适应新和拓展性为代价的…由于以上原因是不推荐向后兼容性之外使用的。”  
  
在 CQL 之前当 Thrift 接口是唯一的 API 时，所有的表都是以压缩存储的方式建立的。和文档状态一样，它是一个除了向后兼容性外别无它用的遗留选项。  
  
在我们的调研大约进行到这个时候我们偶然发现了某团队研究解析的一篇博客文章。主要是关于他们对 CQL 表的经验认识。这是一篇很棒的文章，我十分推荐它。文章有一段的标题是“我们没有使用 COMPACT STORAGE…你敢相信下面发生了什么吗”。它记录了使用压缩存储的他们自己的调研。在这篇文章中，他们提及了当没有对表使用压缩存储时他们看到一个大 30 倍的存储空间容量。如果知道我们所处理的数据库的容量大小，这种类型的增大是不可忽视的。  
  
## CQL 数据分布  
  
那么让我们研究一下一个 CQL 表的数据分布是什么样的。我们将以一个简化过的例子开始，它来自用 Cassandra 建立一项音乐服务的教程指南。我们已经用一个<id，song_id>的指针把表减少到只有一个 id，音乐 id 和歌曲名称。  
  
```
CREATE TABLE playlists_1 (   id uuid,   song_id uuid, title text,
PRIMARY KEY  (id, song_id )
);

INSERT INTO playlists_1 (id, song_id, title)

  VALUES (62c36092-82a1-3a00-93d1-46196ee77204,
  7db1a490-5878-11e2-bcfd-0800200c9a66,
  'Ojo Rojo');

INSERT INTO playlists_1 (id, song_id, title)
  VALUES (444c3a8a-25fd-431c-b73e-14ef8a9e22fc,
  aadb822c-142e-4b01-8baa-d5d5bdb8e8c5,
  'Guardrail');
```
  
现在让我们来看一下它的数据分布是什么样的。我们将使用 sstable2json 来把原始 sstable 转储成一个我们可以分析的格式：  
  
```
$ sstable2json Metrics/playlists_1/*Data*

[
    {
        "columns": [
            [
                "7db1a490-5878-11e2-bcfd-0800200c9a66:",
                "",
                1436971955597000
            ],
            [
                "7db1a490-5878-11e2-bcfd-0800200c9a66:title",
                "Ojo Rojo",
                1436971955597000
            ]
        ],
        "key": "62c3609282a13a0093d146196ee77204"
    },
    {
        "columns": [
            [
                "aadb822c-142e-4b01-8baa-d5d5bdb8e8c5:",
                "",
                1436971955602000
            ],
            [
                "aadb822c-142e-4b01-8baa-d5d5bdb8e8c5:title",
                "Guardrail",
                1436971955602000
            ]
        ],
        "key": "444c3a8a25fd431cb73e14ef8a9e22fc"
    }
]
```
  
你可以看到，每一排都存在有两个列插入，一个列给主键标的元素，第二个列给表中剩下的“title”列。同样注意到名称列的名字被加入到每一排的列关键字中。这一切说明，列关键字赋予的重复副本的数量是合适的，和一个 64 位的列时间标记。  
  
## 使用压缩存储的同一个表分布  
  
让我们来比较一下使用压缩存储选项的同一个表，再看看同一排的存储格式看起来是什么样。  
  
```
CREATE TABLE playlists_2 (   id uuid,   song_id uuid, title text,
PRIMARY KEY  (id, song_id )
) WITH COMPACT STORAGE;

INSERT INTO playlists_2 (id, song_id, title)
  VALUES (62c36092-82a1-3a00-93d1-46196ee77204,
  7db1a490-5878-11e2-bcfd-0800200c9a66,
  'Ojo Rojo');

INSERT INTO playlists_2 (id, song_id, title)
  VALUES (444c3a8a-25fd-431c-b73e-14ef8a9e22fc,
  aadb822c-142e-4b01-8baa-d5d5bdb8e8c5,
  'Guardrail');

[
    {
        "columns": [
            [
                "7db1a490-5878-11e2-bcfd-0800200c9a66",
                "Ojo Rojo",
                1436972070334000
            ]
        ],
        "key": "62c3609282a13a0093d146196ee77204"
    },
    {
        "columns": [
            [
                "aadb822c-142e-4b01-8baa-d5d5bdb8e8c5",
                "Guardrail",
                1436972071215000
            ]
        ],
        "key": "444c3a8a25fd431cb73e14ef8a9e22fc"
    }
]
```
  
和名字所代表的一样，存储格式确实更加紧凑了。对同一排现在只有单独一个列同时这一列并不包括列的字符串名。如果你只看原始表数据这样会描述不充分，但是你可以结合架构一起来定义余下列的名字。  
  
## 从长远看这会是什么样的？  
  
正如你从所看到的一样
