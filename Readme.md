# 从UCloud迁移ES到AWS

本文介绍如何将索引数据从Ucloud Elasticsearch 迁移到AWS Elasticsearch中。
本次ES索引迁移方案的参考架构图如下所示。本次通过对象存储的方式进行迁移，避免占用目前的ipsec vpn的线路资源。
 
总体步骤：
UCloud导出镜像
## 1.	相关概念
•	Elasticsearch：一个分布式的RESTful风格的搜索与分析引擎，适用于各种应用场景。作为Elastic Stack的核心，Elasticsearch可以集中存储您的数据，并对数据进行搜索分析。
•	Kibana：您可以使用Kibana对Elasticsearch数据进行可视化搜索分析。
•	亚马逊Elasticsearch服务（简称AES）： 一项完全托管的服务，提供了各种易于使用的Elasticsearch API和实时分析功能，还可以实现生产工作负载需要的可用性、可扩展性和安全性。您可以使用Amazon Elasticsearch Service轻松部署、保护、操作和扩展Elasticsearch，以便进行日志分析、全文搜索和应用程序监控等工作。
•	Ucloud Elasticsearch服务（简称UES）： UES是基于Elasticsearch和Kibana的打造的日志管理分析服务。通过创建集群的方式来创建服务，能够快速实现集群的部署，集群自动初始化合适的配置和丰富的插件，通过安全插件提供账户角色权限管理功能，为用户提供快速创建、便于管理、并可线性扩容。此外，产品还提供丰富的性能指标监控和可视化管理平台。高性能SSD磁盘的使用，对海量日志数据存储、检索、分析有效提升处理效率。
•	快照和恢复（Snapshot and Restore）：您可以使用快照和恢复功能将各个索引或整个集群的快照创建到远程存储库（如共享文件系统，S3或HDFS）中，并且这些快照可以快速恢复。但这些快照只能恢复到对应版本的Elasticsearc中。
o	UES版本：5.5.1 （5.5的版本已经过了生命周期了，建议在适当的时候进行版本升级，目前AWS ES已经支持到了7.4，且升级非常方便。）
o	UES目前使用情况：10个数据节点，没有专门的master节点,没有ingest节点,活跃主分片数量 12091,活跃分片数量 24182, 每天数据增量20G左右,每天按照日期生成索引导入es。
o	AES版本：5.5（小版本为5.5.2，）
 
o	5.5.1与5.5.2的版本差别，本次版本全部都是bug的修复，不涉及到新功能：https://www.elastic.co/guide/en/elasticsearch/reference/5.6/release-notes-5.5.2.html
Bug fixes
Aggregations
•	Fixes array out of bounds for value count agg #26038 (issue: #17379)
Core
•	Release operation permit on thread-pool rejection #25930 (issue: #25863)
Inner Hits
•	Fix inner hits to work with queries wrapped in an indices query #26138 (issue: #26133)
•	When fetching nested inner hits only access stored fields when needed #25864 (issue: #6)
Logging
•	Declare XContent deprecation logger as static #25881 (issue: #25879)
Query DSL
•	Parse "*" in query_string_query as MatchAllDocsQuery #25872 (issues: #25556, #25726)
## 2.	UCloud导出镜像
操作步骤，可以参考文档https://docs.ucloud.cn/ues/operate/backup
数据备份功能入口：登录UES控制台，点击集群列表中目标集群的“详情”按钮，进入集群详情页面，切换至“数据备份”标签页。
1.注册仓库
在“仓库管理”子页面中，点击“注册仓库”按钮，在弹出的对话框中，按照页面提示完成相关信息的填写，点击确认，系统将为该集群创建一个存放快照的仓库。（前期需要相应安装ufile插件，配置ufile存储桶及ufile令牌）
 
创建备份规则
 

如果是备份特定的索引，可以在备份全部索引那边取消，然后输入需要备份索引。
 

创建备份数据，由于索引都是以日期作为单位的，所以备份的时候按照日期进行备份即可，最后的时候合并的时候可以最大程度的减少恢复的时间。

创造一些测试数据。我这里创建一个index为20200427，有5个分片。
 

到备份管理，点击立刻执行。
 
然后在快照管理可以看到相应的快照。
 

在对象存储上面可以看到相应的备份快照。

 

在aws创建一台ec2云主机，需要给ec2给一个s3的管理的权限（用于免密钥上传文件到s3）。同时开通s3的privatelink（这样s3的传输就会变成内网传输，速度更快）
 

下载ucloud的文件管理工具
wget http://tools.ufile.ucloud.com.cn/filemgr-win64.zip
unzip filemgr-win64.zip
cd filemgr-win64
vim config.cfg ( 修改相应的aksk及终端节点信息)
将ucloud的文件夹下载到本地（需要本地留够充足的空间，）
./filemgr-linux64 --action batch-download --bucket essnapshot --pattern / --prefix ues_backup/ --dir ~/uessnapshot
配置aws的区域信息，只需要在region那边进行配置
aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [cn-northwest-1]: 
Default output format [None]:
aws s3 cp --recursive ~/uessnapshot s3://mssqldemo
(需要保证文件在存储桶的根目录下面)
这样就把所有的快照文件上传到了AWS，UCloud的下载速度大约为10mb/s(1.16G/88秒下载完成)，AWS的传输速度为150mb/s。可以相应计算大约需要的时间.。

## 3.	AWS配置
具体可以参考如下的链接进行配置。https://docs.amazonaws.cn/elasticsearch-service/latest/developerguide/es-managedomains-snapshots.html。
需要配置权限，仓库等。然后就可以查询到之前上传的快照信息了。

 

在aws上执行还原的操作。

 

查看还原的数据。
 

增量index数据还原，
Snapshot的操作是增量的，但是ucloud并没有增量上传的脚本（AWS s3的sync是可以实现增量操作的），所以，如果需要对某个index进行增量操作，需要增加一些编程的工作，如将上传的文件在一个文件里面保存，然后下次如果在该文件存在的文件则不进行传输。（当然，这里也可以在前一步的还原的基础上，配置数据摄取的工具同时往aws及ucloud的es写入）



