---
title: timescaledb vs influxdb 性能测试
date: 2024-12-17
tags: 数据库
---
# influxdb 性能测试
参考https://github.com/GreptimeTeam/tsbs/blob/master/docs/greptimedb-vs-influxdb-manual-zh.md
## 测试机器配置
 客户端 192.168.1.111

 服务器 192.168.1.112

## 1.下载安装 
```
https://dl.influxdata.com/influxdb/releases/influxdb2-2.7.10-1.x86_64.rpm
rpm -ivh influxdb2-2.7.10-1.x86_64.rpm
```
下载client
```
wget https://download.influxdata.com/influxdb/releases/influxdb2-client-2.7.5-linux-amd64.tar.gz
tar xvfz influxdb2-client-2.7.5-linux-amd64.tar.gz
```
启动服务
```
systemctl start influxdb
```

## 2.生成测试数据
安装go
yum install go
拉取代码生成go测试数据
git clone https://github.com/GreptimeTeam/tsbs.git
编译
```
#设置国内源
export GOPROXY=https://goproxy.cn
cd tsbs && make
```
### 生成数据
```

mkdir bench-data
#运行生成数据的命令，其中 influx-data.lp 就是我们生成的测试数据文件，该数据文件用于单机测试：
./bin/tsbs_generate_data --use-case="cpu-only" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-15T00:00:00Z" --log-interval="10s" --format="influx" > ./bench-data/influx-data.lp
#执行以下命令生成用于 InfluxDB 的查询：
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type cpu-max-all-1 --format="influx" > ./bench-data/influx-queries-cpu-max-all-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type cpu-max-all-8 --format="influx" > ./bench-data/influx-queries-cpu-max-all-8.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-1 --format="influx" > ./bench-data/influx-queries-double-groupby-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-5 --format="influx" > ./bench-data/influx-queries-double-groupby-5.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-all --format="influx" > ./bench-data/influx-queries-double-groupby-all.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type groupby-orderby-limit --format="influx" > ./bench-data/influx-queries-groupby-orderby-limit.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type high-cpu-1 --format="influx" > ./bench-data/influx-queries-high-cpu-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type high-cpu-all --format="influx" > ./bench-data/influx-queries-high-cpu-all.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=10 --query-type lastpoint --format="influx" > ./bench-data/influx-queries-lastpoint.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-1-1 --format="influx" > ./bench-data/influx-queries-single-groupby-1-1-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-1-12 --format="influx" > ./bench-data/influx-queries-single-groupby-1-1-12.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-8-1 --format="influx" > ./bench-data/influx-queries-single-groupby-1-8-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-5-1-1 --format="influx" > ./bench-data/influx-queries-single-groupby-5-1-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-5-1-12 --format="influx" > ./bench-data/influx-queries-single-groupby-5-1-12.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-5-8-1 --format="influx" > ./bench-data/influx-queries-single-groupby-5-8-1.dat

//执行以下命令生成用于生成用于 GreptimeDB 的查询：
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type cpu-max-all-1 --format="greptime" > ./bench-data/greptime-queries-cpu-max-all-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type cpu-max-all-8 --format="greptime" > ./bench-data/greptime-queries-cpu-max-all-8.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-1 --format="greptime" > ./bench-data/greptime-queries-double-groupby-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-5 --format="greptime" > ./bench-data/greptime-queries-double-groupby-5.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-all --format="greptime" > ./bench-data/greptime-queries-double-groupby-all.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type groupby-orderby-limit --format="greptime" > ./bench-data/greptime-queries-groupby-orderby-limit.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type high-cpu-1 --format="greptime" > ./bench-data/greptime-queries-high-cpu-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type high-cpu-all --format="greptime" > ./bench-data/greptime-queries-high-cpu-all.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=10 --query-type lastpoint --format="greptime" > ./bench-data/greptime-queries-lastpoint.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-1-1 --format="greptime" > ./bench-data/greptime-queries-single-groupby-1-1-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-1-12 --format="greptime" > ./bench-data/greptime-queries-single-groupby-1-1-12.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-8-1 --format="greptime" > ./bench-data/greptime-queries-single-groupby-1-8-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-5-1-1 --format="greptime" > ./bench-data/greptime-queries-single-groupby-5-1-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-5-1-12 --format="greptime" > ./bench-data/greptime-queries-single-groupby-5-1-12.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="
T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type 
single-groupby-5-8-1 --format="greptime" > ./bench-data/greptime-queries-single-groupby-5-8-1.dat

idb查询生成
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type cpu-max-all-1 --format="timescaledb" > ./bench-data/timescaledb-queries-cpu-max-all-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type cpu-max-all-8 --format="timescaledb" > ./bench-data/timescaledb-queries-cpu-max-all-8.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-1 --format="timescaledb" > ./bench-data/timescaledb-queries-double-groupby-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-5 --format="timescaledb" > ./bench-data/timescaledb-queries-double-groupby-5.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type double-groupby-all --format="timescaledb" > ./bench-data/timescaledb-queries-double-groupby-all.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type groupby-orderby-limit --format="timescaledb" > ./bench-data/timescaledb-queries-groupby-orderby-limit.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type high-cpu-1 --format="timescaledb" > ./bench-data/timescaledb-queries-high-cpu-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=50 --query-type high-cpu-all --format="timescaledb" > ./bench-data/timescaledb-queries-high-cpu-all.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=10 --query-type lastpoint --format="timescaledb" > ./bench-data/timescaledb-queries-lastpoint.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-1-1 --format="timescaledb" > ./bench-data/timescaledb-queries-single-groupby-1-1-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-1-12 --format="timescaledb" > ./bench-data/timescaledb-queries-single-groupby-1-1-12.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-1-8-1 --format="timescaledb" > ./bench-data/timescaledb-queries-single-groupby-1-8-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-5-1-1 --format="timescaledb" > ./bench-data/timescaledb-queries-single-groupby-5-1-1.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-5-1-12 --format="timescaledb" > ./bench-data/timescaledb-queries-single-groupby-5-1-12.dat
./bin/tsbs_generate_queries --use-case="devops" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-14T00:00:01Z" --queries=100 --query-type single-groupby-5-8-1 --format="timescaledb" > ./bench-data/timescaledb-queries-single-groupby-5-8-1.dat
```
# InfluxDB 
## 初始化
InfluxDB 初次安装后需要初始化 InfluxDB 并拿到请求的 token，如果已经有 token 则可以跳过这部分。

初始化 InfluxDB：
```
tar -zxvf influxdb2-client-2.7.5-linux-amd64.tar.gz

./influx setup \
  --username test \
  --password 12345678 \
  --token test-token \
  --org test-org \
  --bucket test-bucket \
  --force
```
输出：
```
User    Organization    Bucket
test    test-org        test-bucket
```
创建 token：
```
./influx auth create \
  --org test-org \
  --all-access
```
输出

```
ID                      Description     Token                                                                                           User Name      User ID                  Permissions
0e05e4d370c1b000                        cVIR8Lcr9pT-q2awgFkoFYqTTTxUBKFI8rY3R7jvK7kOwI5KHTJRpKrCSjKX8XES-npy-6lR3hDPIB3VibM9AA==        test           0e05e4c75b01b000
```
这里可以拿到 InfluxDB 的 token。为了方便后续请求，我们将该 token export 到环境变量里：
```
export INFLUX2_TOKEN="cVIR8Lcr9pT-q2awgFkoFYqTTTxUBKFI8rY3R7jvK7kOwI5KHTJRpKrCSjKX8XES-npy-6lR3hDPIB3VibM9AA=="
```

## 导入数据
### influxdb导入
```
./bin/tsbs_load_influx2 \
    --urls=http://192.168.1.112:8086 \
    --file=./bench-data/influx-data.lp \
    --do-create-db=false \
    --org-id=test-org \
    --db-name=test-bucket \
    --batch-size=3000 \
    --workers=8 \
    --auth-token=$INFLUX2_TOKEN
```
导入性能
1732613214,1833041.28,4.836090E+09,1804509.59,183304.13,4.836090E+08,180450.96
Summary:
loaded 4838400000 metrics in 2681.264sec with 8 workers (mean rate 1804522.36 metrics/sec)
loaded 483840000 rows in 2681.264sec with 8 workers (mean rate 180452.24 rows/sec)

### greptime导入
```
./bin/tsbs_load_greptime \
    --urls=http://192.168.1.112:4000 \
    --file=./bench-data/influx-data.lp \
    --batch-size=3000 \
    --gzip=false \
    --workers=6
```
tar -zxvf greptime-linux-amd64-centos-v0.11.0-nightly-20241125.tar.gz
./greptime-linux-amd64-centos-v0.11.0-nightly-20241125/greptime standalone start --data-home /root/greptimedb/data --http-addr 192.168.1.112:4000
导入性能
1732625414,2649214.76,4.822290E+09,2724457.44,264921.48,4.822290E+08,272445.74
Summary:
loaded 4838400000 metrics in 1776.165sec with 6 workers (mean rate 2724071.87 metrics/sec)
loaded 483840000 rows in 1776.165sec with 6 workers (mean rate 272407.19 rows/sec)

### IDB导入
```
./bin/tsbs_generate_data --use-case="cpu-only" --seed=123 --scale=4000 --timestamp-start="2023-06-01T00:00:00Z" --timestamp-end="2023-06-15T00:00:00Z" --log-interval="10s" --format="timescaledb" |gzip > ./bench-data/timescaledb-data.gz

cat ./bench-data/timescaledb-data.gz | gunzip| ./tsbs_load_timescaledb --host=192.168.1.112 --port=30000 --postgres="sslmode=disable" --admin-db-name=testdb --db-name=testdb --user=idb --pass=idb --do-create-db=false --workers=6 --batch-size=3000
use
./bin/tsbs_load_timescaledb --host=192.168.1.112 --port=30000 --postgres="sslmode=disable" --admin-db-name=testdb --db-name=testdb --do-create-db=false --workers=6 --batch-size=3000 --file=./bench-data/influx-data.lp 
导入性能

1732701590,4826563.45,4.807590E+09,4713318.83,482656.35,4.807590E+08,471331.88

Summary:
loaded 4838400000 metrics in 1027.162sec with 6 workers (mean rate 4710452.39 metrics/sec)
loaded 483840000 rows in 1027.162sec with 6 workers (mean rate 471045.24 rows/sec)

```


## 查询
在 tsbs 目录下，执行查询
```
//influxdb
./bin/tsbs_run_queries_influx --file=./bench-data/influx-queries-cpu-max-all-1.dat          --db-name=test-bucket   --is-v2=true  --auth-token=$INFLUX2_TOKEN   --urls="http://192.168.1.112:8086"

//greptimedb
./bin/tsbs_run_queries_influx --file=./bench-data/greptime-queries-cpu-max-all-1.dat          --db-name=benchmark   --urls="http://192.168.1.112:4000"
//idb
./bin/tsbs_run_queries_timescaledb --file=./bench-data/timescaledb-queries-cpu-max-all-1.dat --hosts=192.168.1.112 --port=30000 --user=idb --pass=idb --db-name=testdb --postgres='sslmode=disable'
idb config配置：
max_connections = 100
max_prepared_transactions = 100
shared_buffers = 3GB
max_wal_size = 16GB
min_wal_size = 1GB
effective_cache_size = 3GB
temp_buffers = 256MB
work_mem = 2GB
maintenance_work_mem = 2GB
max_worker_processes = 8
max_parallel_maintenance_workers = 4
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
synchronous_commit = off
archive_mode = off
wal_level = minimal
max_wal_senders = 0
shared_preload_libraries = 'timedb'
idb压缩设置
ALTER TABLE cpu SET (
  timedb.compress,
  timedb.compress_segmentby = 'time'
);
SELECT add_compression_policy('cpu', INTERVAL '1 days');
```

|查询类型 |	查询执行次数	 | InfluxDB平均耗时(毫秒)	 | GreptimeDB平均耗时(毫秒)	 |IDB平均耗时(毫秒)|IDB压缩后平均耗时(毫秒)|
| ------ | ---- | -------- | -------- | -------- |-------- |
cpu-max-all-1|	100|	8.06|	25.45|	8.84|967.31|
cpu-max-all-8|	100|	8.92|	36.71|	100.90|1013.84|
double-groupby-1|	50|	1646.83|	818.93|2427.36	|1680.42|
double-groupby-5|	50|	6762.59|	1113.97|	2970.54|2730.16|
double-groupby-all|	50|	12832.77|	1546.60|	3551.83|3908.27|
groupby-orderby-limit|	50|	25790.39|	13676.30|	30.78|7799.27|
high-cpu-1|	100|	9.86|	8.24|	25.23|794.90|
high-cpu-all|	50|	28559.38|	4699.47|	3431.82|3274.46|
lastpoint|	10|	2995.42|	14348.95|	62.76|8180.81|
single-groupby-1-1-1|	100|	1.88|	17.31|	2.00|60.19|
single-groupby-1-1-12|	100|	10.05|	16.83|	26.22|457.63|
single-groupby-1-8-1|	100|	5.74|	19.61|	8.12|60.83|
single-groupby-5-1-1|	100|	7.03|	16.36|	2.09|103.28|
single-groupby-5-1-12|	100|	30.68|	19.45|	27.04|817.98|
single-groupby-5-8-1|	100	|11.32|	21.66|	8.63|104.17|




