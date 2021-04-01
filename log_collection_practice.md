# 日志收集企业实战作业练习
参与者: 卢金树，曾茂泉，黎杰铭，赵国鹏
场景提供：黎杰铭

结合本课所学实战内容，根据公司业务发展阶段，构建公司大数据日志采集中台（平台）。

要求:
* 第一：给出架构总体设计以及详细落地方案设计
* 第二：给出落地可运行代码


场景:
- 目的：
	- 前端：抓取用户交互行为，为平台设计提供支持。
	- 系统业务响应等指标诊断，故障追溯。
- 问题
	- 前端埋点可行，但治理复杂，如何统一 做 Central logging, 有误操作时，快速排查。
	- 服务端 黑盒如何不侵入状态下 获取 log,轻量无dependency直接插拔。
	- 由于人力限制，需要考虑统一 前端，服务端，业务数据库日志采集。
	- 其他: biolog日志 延迟重。数据完整性需要支持双路校验。

分析复杂度来源: 
- 高性能(高扩展)：目前前端性能已经满足要求，机器扩展也比较好做。
- 高可用: 目前前端设计 在物理机范围，做好采集高可用，其他由kafka集群自身高可用。
- 高可靠(数据零丢失完整性): 要求高。
- 成本(开发维护监控排查成本): 要求高。


## 架构设计

![[log_collection_v1.jpg]]

### 落地方案解析

数据平台部门已有方案:
- 依赖托管服务 datadog 提供infrasturcture metrics 监控。
- 使用轻量无JVM依赖的 filebeat 读取 Nginx 用户行为日志。
- 使用Kafka 作为数据统一入口 隔离数据部门与业务部门。
- 全量数据 使用 Kafka Connect 将原始数据保存在 S3。
- 离线分析部分 由 Spark 和 Airflow 批量处理。
以上方案经反馈，目前可行并且已经运行一段时间。

基于已有方案
- 数据采集部分
	- 添加 supervisor 重启filebeat, 可选添加主备filebeat，支持狭义高可用。
	- 添加同上的filebeat 在业务服务器上，收集业务日志。
	- 添加Canal 集群收集 binlog 增量。

- 数据平台部分
	- 替换 flink, 改为flume 集群作为Kafka集群的消费者进行 主题分拣。
		- 原因：flink 需要维护代码，但对自定义友好。 flume 只需配置文件，能满足一般需求。由于主要目的是分拣，flume可以达到目的。另外，flume 更改sink 配置也可以支持 Elasticsearch.
	- 添加 flink 消费 分拣过后的Kafka，支持实时分析业务。

- 平台治理部分
	- Option 1: 利用第三方托管的日志服务，收集业务日志，用于排查目的。加上之前一份，一共两份业务日志。
		- 分析: 基于已经使用第三方服务，datadog 或者 AWS CloudWatch, 可以将业务日志一起托管。
	
	- Option 2: 第三方日志服务 只收集infrastructure metrics, 业务日志由数据部门统一由 flume -> Elasticsearch -> Kibana 使用。
		- 分析: 
			- 使用 Flume 不使用 flink 主要出于简化代码开发维护成本。
			- 不使用 filebeat-> logstash-> Elasticsearch, 主要是因为 logstash 虽然针对 Elasticsearch 友好，但没有kafka 作为统一数据入口的扩展性好。另外，Flume 支持sink 配置 Elasticsearch。这里的方案采取 kafka -> flume -> Elasticsearch，而不多维护一套 logstash。
	- Option 1 与 Option 2相比：
		- pros: 上线快; 统一查询界面，不用切换; 减少维护Elasticsearch, 降低人力成本; 
		- cons: 敏感业务日志不允许托管; 增加服务成本; 发两份日志 增加服务器负担。可以通过部分日志托管，有需求的时候通过S3的全量日志查找保护敏感数据。
	- Option 3: 为了减轻Option 1 的业务服务器负担，Monitoring agent 不收集业务数据，统一由 Kafka 和 flume分拣。 不同于 Option 2, 利用Kafka Connect 将部分日志发送到托管服务。


## 代码设计


### Nginx and Filebeat
by 黎杰铭
refer to `config/1.setup_nginx_filebeat.txt`

### Canal for MySQL binlog
by 曾茂泉

![[2.setup_canal_binlog]]

### Flume and Kafka
by 卢金树

refer to `config/3.setup_flume_kafka.txt`

### Kafka Connect
by 黎杰铭

refer to directory `config/4.setup_kafkaconnect`
