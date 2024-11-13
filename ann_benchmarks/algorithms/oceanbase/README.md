# 在 OceanBase 上运行 ann-benchmark

0. 安装所需 python 依赖，并下载数据集
```
pip install -r requirements.txt
wget -O data/sift-128-euclidean.hdf5 http://ann-benchmarks.com/sift-128-euclidean.hdf5
```
1. 编译/部署 OceanBase 数据库，并创建用于测试的租户
```bash
# 编译
bash build.sh release -DOB_USE_CCACHE=ON --init --make
# 部署
# 在 oceanbase 目录下运行
./tools/deploy/obd.sh prepare -p /tmp/obtest
# 配置文件请参考下面 obcluster.yaml 示例
./tools/deploy/obd.sh deploy -c obcluster.yaml

# 创建用于测试的租户
create resource unit unit_1 max_cpu 6, memory_size "10G", log_disk_size "10G";
create resource pool pool_2 unit = 'unit_1', unit_num = 1, zone_list = ('zone1');
create tenant perf replica_num = 1,primary_zone='zone1', resource_pool_list=('pool_2') set ob_tcp_invited_nodes='%';

```

obcluster.yaml:

```yaml
oceanbase-ce:
  servers:
    - name: server1
      ip: 127.0.0.1
  server1:
    mysql_port: 2881
    rpc_port: 2882
    home_path: /data/obcluster
    zone: zone1
    # The directory for data storage. The default value is home_path/store.
    # data_dir: /data
    # The directory for clog, ilog, and slog. The default value is the same as the data_dir value.
    # redo_dir: /redo
  tag: latest
  global:
    # for default system config used by farm, please see tools/deploy/obd/observer.include.yaml
    # You can also specify the configuration directly below (stored locally, switching the working directory and redeploying will still take effect)
    devname: lo
    root_password:
    cpu_count: '24'
    memory_limit: 14G
    system_memory: 1G
    datafile_size: 60G
    log_disk_size: 40G
    cluster_id: 1730447006
    enable_syslog_recycle: true
    enable_syslog_wf: false
    max_syslog_file_count: 4
    production_mode: false
```
2. 运行 ann-benchmark.
```bash
# 测试时导入数据并构建索引
python run.py --algorithm oceanbase --local --force --dataset sift-128-euclidean --runs 1
# 测试时跳过导入数据及构建索引
python run.py --algorithm oceanbase --local --force --dataset sift-128-euclidean --runs 1 --skip_fit
# 计算召回率及 QPS
python plot.py --dataset sift-128-euclidean --recompute
```
3. 生成运行结果.
```bash
# 示例命令
python plot.py --dataset sift-128-euclidean --recompute
# 示例输出如下，其中每行结果倒数第一个值为该算法对应的QPS，每行结果倒数第二个值为该算法对应的召回率。
Computing knn metrics
  0:                               OBVector(m=16, ef_construction=200, ef_search=400)        0.999      416.990
Computing knn metrics
  1:                                                                 BruteForceBLAS()        1.000      355.359
```

## 运行向量标量混合场景测试
运行混合场景测试需要先导入数据并构建索引。
运行当前目录下的 hybrid_ann.py 即可。