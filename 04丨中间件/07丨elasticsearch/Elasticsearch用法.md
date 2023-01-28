# 1.ES安装

## 1.1.window

### 1.1.1.单机单服务

在window环境下，直接将下载好的ES压缩包解压，找到解压目录下的..\elasticsearch-5.6.9\bin下面的elasticsearch.bat，双击即可运行，打开浏览器，输入：[http://localhost:9200](http://localhost:9200)，能返回JSON信息说明启动成功，把ES安装成一个服务：到ES安装目录的bin目录，执行命令：

```bash
elasticsearch-service.bat install
```

启动ES也可以执行命令：

```bash
elasticsearch-service.bat start
```

### 1.1.2.单机多服务

ES在安装成服务之前，可以通过设置window环境变量，配置服务的属性：( 注意：可能不同版本之间这些环境变量的变量名会不一样 )，正常情况，仅仅使用到SERVICE_DISPLAY_NAME改变ES服务名称而已：

- **SERVICE_ID** - 服务的唯一标识符。如果在同一台机器上安装多个实例，这很有用。 默认为elasticsearch-service-x64.

- **SERVICE_USERNAME** - 运行服务的用户，默认为本地系统帐户

- **SERVICE_PASSWORD** - 在%SERVICE_USERNAME%中指定的用户的密码

- **SERVICE_DISPLAY_NAME** - 服务的名称。

- **SERVICE_DESCRIPTION** - 服务的描述。

- **JAVA_HOME** - 运行服务所需的JVM的安装目录。

- **SERVICE_LOG_DIR** - 服务日志目录，默认为%ES_HOME%\logs。注意，这并不控制Elasticsearch日志的路径;这些路径是通过在elasticsearch.yml配置文件中的path.logs设置或命令行上设置的。

- **ES_PATH_CONF** - 配置文件目录（）
   Configuration file directory (需要包含 elasticsearch.yml, jvm.options, and log4j2.properties文件), 默认是 %ES_HOME%\config。

- **ES_JAVA_OPTS** - 您可能想要应用的任何其他JVM系统属性。

- **ES_START_TYPE** - 服务的启动模式。可以是自动的，也可以是手动的(默认)。

- **ES_STOP_TIMEOUT** - 等待服务优雅退出的超时时间(秒)。默认值为0。

1. 先设置ES服务名称，在window环境变量创建一个`SERVICE_DISPLAY_NAME`，值就是ES服务名称；

2. 修改ES安装服务的脚本文件elasticsearch-service.bat的SERVICE_ID，在默认SERVICE_ID值后面加上ES的版本号即可( 当然也可以加上其他，只要保证唯一性就行，避免本机服务名冲突 )

   ![](./images/修改SERVICE_ID.png)

3. 全部设置好，执行elasticsearch-service.bat install就可以安装了。一个版本的ES安装好后，重新执行①②步，安装另一个版本的ES，结果：

   ![](./images/es多版本安装结果.png)

## 1.2.Linux

### 1.2.1.安装步骤

1. 准备好ES的安装包，使用rz上传到Linux服务器上，或者直接联网下载，网址为：[ES-6.4.3.tar.gz](curl -L -O https:/artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.3.tar.gz)。
2. 解压刚才上传或下载的ES压缩包，切换到解压后的目录下的bin目录，执行命令：./elasticsearch（前台启动），./ elasticsearch -d（后台启动）

### 1.2.2安装报错

#### 1.2.2.1.拒绝root用户启动

es不允许以root用户启动，在2.x版本还可以在启动时加上参数，如：./elasticsearch -Des.insecure.allow.root=true来以root用户启动；但是在5.x以后，除非修改源代码，否则不能以root用户启动，因此需要创建一个单独的用户用来运行ES。

**步骤：**

1. 创建用户组，执行命令：groupadd esUsers（esUsers为自定义组名）

2. 创建用户，执行命令：useradd es -g esUsers -p elasticsearch ( es为用户名，esUsers为用户组名，-p后面是密码 )

3. 更改elasticsearch文件夹及内部文件的所属用户及组，执行命令：chown -R es:esUsers /usr/local/... (最后一个参数是ES安装目录)

#### 1.2.2.2.启动报错

①**报错原因**：“max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]”。

**解决方法：**切换到root用户，在vim /etc/security/limits.conf中添加：

 ```tex
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
 ```

②**报错原因**：“max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]”

**解决方法：**切换到root用户，修改配置vim /etc/sysctl.conf，添加如下的配置：vm.max_map_count=655360，并执行命令：sysctl -p

### 1.2.3.启动关闭ES

**启动ES（先切换到ES安装目录下的bin下**）**

①前台启动ES，执行：./elasticsearch

②后台启动ES，执行：./elasticsearch -d

**关闭ES**

①查找ES的进程号，执行命令：ps -ef | grep elastic

②杀掉ES进程，执行命令：kill -9 2971 （2971是上条命令查的ES进程号）

## 1.3.导入数据

### 1.3.1.JSON导入

准备好JSON文件，注意格式要符合_bulk的要求，然后执行下面的语句：

```bash
curl -H "Content-Type: application/x-ndjson" -XPOST "127.0.0.1:9200/bank/account/_bulk?pretty" --data-binary @accounts.json
```

# 2.搜索实战

## 2.1.搜索结果调整

```json
POST kibana_xxx/_search
{
    "_source":["order_date"], //只返回需要的字段
    "sort":[{"order_date":"desc"}] //按照指定字段排序
}
```

## 2.2.脚本字段script

```json
POST kibana_xxx/_search
{
    "script_fields":{
        "##自定义field_name":{
            "script":{
                "lang":"painless",
                "source":"doc['order_quantity'].value * params['multiplier']",
                "params":{
                    "multiplier":2
                }
            }
        }
    }
    "query":{"match_all":{}}
}
```

## 2.3.match相关

- match查询

```json
POST kibana_xxx/_search
{
    "query":{
        "match":{
            "order_title":"Last Christmas" // 词语之间是或的关系
        },
        "match":{
            "order_title":{
                "query":"Last Christmas",
                "operator":"and"          // 词语之间是与的关系
            }
        }
    }
}
```

- macth_phrase查询

```json
POST kibana_xxx/_search
{
    "query":{
        "match_phrase":{
            "order_title":"one love", 
            "slop":1  // slop表示短语之间允许插入多少个词语,为0说明只能匹配"one love"文档
                      // 为1说明可以匹配"one xxx love"的文档,以此类推
        }
    }
}
```

## 2.4.聚合

聚合的分类：

1. Bucket Aggregation：一些列满足特定条件的文档的集合，类似SQL的group by
2. Metrics Aggregation：一些数学运算，可以对文档字段进行统计分析，类似SQL的count(1)
3. Pipeline Aggregation：对其它的聚合结果进行二次聚合
4. Matrix Aggregation：支持对多个字段的操作并提供一个结果矩阵

## 2.5.基于Term的查询

Term是表达语意的最小单位，在ES中，Term查询对输入不做分词，而是将整个输入作为一个整体，在倒排索引中查找准确的词项，并用相关度算分公式为匹配文档进行相关度算分。与Term查询相关的Api有：

- Term Query
- Range Query
- Exists Query
- Prefix Query
- Wildcard Query

```json
POST /products/_search
{
    "query":{
        "term":{
            "productId.keyword":"XHDK-A-1293"
        }
    }
}
```

使用 Constant Score 可以将Term查询转成Filter，忽略TF-IDF计算，避免相关性算分的开销

```json
POST /products/_search
{
    "query":{
        "constant_score":{
            "filter":{
                "term":{
                    "productId.keyword":"XHDK-A-1293"
                }
            }
        }
    }
}
```

## 2.6.基于全文的查询

基于全文本的查询，索引和搜索时都会进行分词。索引时候，会将数据分词后存入倒排索引；查询的时候，会对输入的查询进行分词，然后每个词项逐个进行底层的查询，最终将结果进行合并。并为每个文档生成一个算分。基于全文的查询API有：

- Match Query
- Match Phrase Query
- Query String Query

## 2.7.复合查询

bool查询是一个或者多个查询子句的组合，总共包括4种子句，其中2种会影响算分，2种不影响算分。每个查询子句计算得出的朴分会被合并到总的相关性评分中。

| 子句     | 作用                                       |
| -------- | ------------------------------------------ |
| must     | 必须匹配，贡献算分                         |
| should   | 选择性匹配，贡献算分                       |
| must_not | Filter Context，必须不能不匹配，不贡献算分 |
| filter   | Filter Context，必须匹配，不贡献算分       |

在ES中，有Query和Filter两种Context：

- Query Context：相关性算分
- Filter Context：不需要算分，可以利用Cache，获得更好的性能

```json
POST /product/_search
{
    "query":{
        "bool":{
            "must":{
                "term":{"price":30}
            },
            "filter":{
                "term":{"avaliable":"true"}
            },
            "must_not":{
                "range":{
                    "price":{"le":10}
                }
            },
            "should":[
                {"term":{"productID.keyword":"LOL"}},
                {"term":{"productID.keyword":"mhxy"}},
            ]
        }
    }
}
```

# 3.集群运维

## 3.1.集群健康

查看集群的健康状况：GET _cluster/health。其中有一个status变量值，有3个取值：

1. Green：主分片与副本都正常分配
2. Yellow：主分片全部正常分配，有副本分片未能正常分配
3. Red：有主分片未能分配

常见错误返回

| 问题         | 原因               |
| ------------ | ------------------ |
| 无法连接     | 网络故障或集群挂了 |
| 连接无法关闭 | 网络故障或节点出错 |
| 429          | 集群过于繁忙       |
| 4xx          | 请求体格式有错     |
| 500          | 集群内部错误       |

# 4.插件体系

## 4.1.分词

分词器（Analyzer）是专门处理分词的组件，由三部分组成：

- 【Character Filters】：针对原始文本处理，比如去除html语法等。一些自带的Character Filters
  - html_strip：去除html标签
  - Mapping：字符串替换
  - Pattern replace：正则匹配替换

- 【Tokenizer】：将被character filters处理过的文本按照一定规则，切分为词（term or token）。内置：
  - whitespace：按照空格切分词语
  - standard：
  - uax_url_email：按照email或者url的格式切分词语
  - pattern：自定义正则表达式切词
  - keyword：不做任何处理，输入即输出
  - path hierarchy：按照文件路径分词

- 【Token Filter】：将Tokenizer切分后的词语进行加工、小写、删除，增加。内置的：
  - lowercase：转为小写
  - stop：去除英文停用词，比如the、on、in、a等
  - synonym：添加近义词


当ES自带的分词器无法满足时，可以自定义分词器，通过自己组合上述的三种组件来实现想要的分词效果

```json
POST _analyze
{
    "tokenizer":"keyword",
    "char_filter":["html_strip"],
    "text":"<b>hello world</b>"
}
```

```json
POST _analyze
{
    "tokenizer":"standard",
    "char_filter":[
        {
            "type":"mapping",
            "mappings":["- => _"] //把"-"替换成下划线"_"
        }
    ]
}
```

```json
GET _analyze
{
    "tokenizer":"whitespace",
    "filter":["stop"],
    "text":["the rain in Spain falls mainly on the plain."]
}
```

**内置分词器**

1. Standard Analyzer：默认分词器，按词切分，小写处理
2. Simple Analyzer：按照非字母切分（符号被过滤），小写处理
3. Stop Analyzer：小写处理，停用词过滤（the,a,is）
4. Whitespace Analyzer：按照空格切分，不转小写
5. Keyword Analyzer：不分词，直接将输入当做输出
6. Patter Analyzer：正则表达式，默认 \w+（非字符分隔）
7. Language：提供了30多种常见语言的分词器
8. Customer Analyzer：自定义分词器

**语法验证**

```xquery
GET _analyze
{
	"analyzer":"simple",
	"text":"1 running Quick brown-foxes leap over lazy dogs in the summer evening"
}
```

**中文分词**

- icu analyzer，提供Unicode的支持，更好的支持亚洲语言，需要安装，执行命令：Elasticsearch-plugin install analysis-ico。语法为：

  ```xquery
  GET _analyze
  {
  	"analyzer":"icu_analyzer",
  	"text":"他说的确实在理"
  }
  ```

- IK。第三方插件，支持自定义词库，支持热更新分词词典。地址： [elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)
- THULAC。第三方插件，清华大学自然语言处理和社会人文计算实验室的一套中文分词器。地址： [elasticsearch-thulac-plugin](https://github.com/microbun/elasticsearch-thulac-plugin)

