### ElasticSearch全文检索引擎

> docker安装

```shell
docker pull docker.elastic.co/elasticsearch/elasticsearch:版本号（7.6.2）
```

```shell
# 端口绑定 并重命名
$ docker run -d --name es -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.6.2
```

> kibana安装

> Elastic search-head安装

```shell
$ docker pull mobz/elasticsearch-head:5
$ docker run -d --name es-head -p 9100:9100 mobz/elasticsearch-head:5
```

然后访问localhost:9100 即可通过head访问es

Tips: 有个坑，需要进入docker的es，修改config，加入跨域配置，es-head才能访问到es

```yaml
http.cors.enabled: true
http.cors.allow-origin: "*"
```

然后 docker restart es 当然也可以先配置好才启动

> 安装ik 设置分词器

默认分词器会将每个句子拆成单个字

```http
curl --location --request POST 'localhost:9200/_analyze' \
--header 'Content-Type: application/json' \
--data-raw '{
    "analyzer":"standard",
    "text":"上海"
}'
```

​	![截屏2021-05-30 下午6.27.42](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530182753.png)

可以先下载ik的 linux zip包，然后通过docker exec 进入es 插件文件夹下，放入，注意，ik的版本要和es一致，安装完后重启es

```shell
$ docker cp [from] [to]
```

![截屏2021-05-30 下午6.37.23](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530183745.png)

启动后es马上挂掉，分配的内存太小：

```shell
$ docker run -d --name es -p 9200:9200 -p 9300:9300 -e ES_JAVA_OPTS="-Xms512m -Xmx512m" -e "discovery.type=single-node" elasticsearch:(自己版本)
```

这里我一直没成功，版本也是能对应上，最后选择在线安装，方便解决

```shell
# elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.8.0/elasticsearch-analysis-ik-7.8.0.zip
```

提供两种分词，粗力度：`ik_smart`和细粒度：`ik_max_word`

![截屏2021-05-30 下午8.45.53](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530212718.png)

![截屏2021-05-30 下午8.45.21](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530204745.png)

---

> 基本操作和概念

1、Node和Cluster

​	es本质上是一个分布式数据库，每台服务器可以运行多个es实例节点`node`，一组节点构成一个集群`cluster`

2、Index

​	Index`索引`是es的顶层单位，它是单个数据库的同义词，每个index必须为小写

​	查看所有索引

```shell
$ curl -X GET 'http://localhost:9200/_cat/indices?v'
```

3、Document

​	Index里面单条记录称为一个Document`文档` es7.x版本后已经去除了type `类型`，相当于数据库中表的概念，用来区分同一index(数据库)中不同结构的表（type），去除之后document并没有严格要求有相同的结构`schema`，但是也是同一索引下的document都是保持一个结构的，有利于管理和搜索

一个document记录：

```json
{
  "username":"张三",
  "age":20,
  "desc":"dba"
}
```

4、 Field

​	ducument中每一列称为域`Field`，对应数据库中的字段，由域的类型和值

| 类型       |                           具体类型                           |
| ---------- | :----------------------------------------------------------: |
| 字符串类型 |                      `text`、`keyword`                       |
| 数字类型   | `long`、`integer`、`short`、`byte`、`double`、`float`、`half_float`、`scaled_float` |
| 日期类型   |                     `date`、`date_nanos`                     |
| 布尔类型   |                          `boolean`                           |
| 二进制类型 |                           `binary`                           |
| 范围类型   |                           `range`                            |

> 基本curl命令操作

```shell
## 索引
# 新建索引
$ curl -X PUT 'localhost:9200/index-name'
# 删除索引
$ curl -X DELETE 'localhost:9200/index-name’
```

> 索引创建成功

![截屏2021-05-30 下午2.21.17](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530142130.png)

新建索引一般需要指定各个字段

```json
{
  "mappings":{
    "properties":{
      "user"{
      	"type":"text",
      	"analyzer":"ik_max_word",
      	"search_analyzer":"ik_max_word"
    }
    }
  }
}
# type为字段类型 , analyzer是字段文本的分词器 search_analyzer是搜索词的分词器
curl --location --request PUT 'localhost:9200/account' \
--header 'Content-Type: application/json' \
--data-raw '{
    "mappings":{
        "properties":{
            "user":{
                "type":"text",
                "analyzer":"ik_max_word",
                "search_analyzer":"ik_max_word"
            },
            "title":{
                "type":"text",
                "analyzer":"ik_max_word",
                "search_analyzer":"ik_max_word"
            },
            "desc":{
                "type":"text",
                "analyzer":"ik_max_word",
                "search_analyzer":"ik_max_word"
            }
        }
    }
}'
```

> 新增文档

```shell
# 虽然7.x之后除去了type，为了检索速度，但是默认是有type的=_doc，语句中还是要加上
$ PUT /index/_doc/id
{
    "user":"法外狂徒张三",
    "title":"男枪",
    "desc":"爆破"
}
```

多次put会覆盖更新，但是put不传值会为空，_update方式更新只更新有值的字段

```shell
$ POST /index/_doc/id/_update
```



> 查询文档

```shell
# 简单id精确搜索
$ GET /index/_doc/id

# 复杂搜索（排序 分页 高亮 模糊，精确查询）
$ GET /index/_doc/_search
{
   "query":{
       "match":{
           "user":"法外"
       }
   }
}
response:
{
    "took": 57,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.2876821,
        "hits": [
            {
                "_index": "account",
                "_type": "_doc",
                "_id": "1",
                "_score": 0.2876821,
                "_source": {
                    "user": "法外狂徒张三",
                    "title": "男枪",
                    "desc": "爆破"
                }
            }
        ]
    }
}
```

