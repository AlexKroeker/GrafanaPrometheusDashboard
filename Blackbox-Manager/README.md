# 如何优雅的实现Consul管理站点监控

## 实现功能

- 基于Prometheus + Blackbox_Exporter实现站点与接口监控。
- 基于Consul实现Prometheus监控目标的自动发现。
- Blackbox Manager：基于Flask + Vue实现的Web管理平台维护监控目标。
- 实现了一个脚本可批量导入监控目标到Consul。
- 更新了一个Blackbox Exporter的Grafana展示看板。

## 截图

### Blackbox Manager Web管理界面
![0](https://raw.githubusercontent.com/starsliao/Prometheus/master/Blackbox-Manager/screenshot/0.png)

### Blackbox Exporter Dashboard 截图
![1](https://raw.githubusercontent.com/starsliao/Prometheus/master/Blackbox-Manager/screenshot/1.png)![2](https://raw.githubusercontent.com/starsliao/Prometheus/master/Blackbox-Manager/screenshot/2.png)

## 部署需求

### 部署Consul

##### 安装

```bash
# 使用yum部署consul
yum install -y yum-utils
yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
yum -y install consul
# 或者直接下RPM包安装
wget https://rpm.releases.hashicorp.com/RHEL/7/x86_64/stable/consul-1.11.1-1.x86_64.rpm
rpm -ivh ./consul-1.11.1-1.x86_64.rpm
```

##### 配置

```bash
vi /etc/consul.d/consul.hcl
data_dir = "/opt/consul"
client_addr = "0.0.0.0"
ui_config{
  enabled = true
}
server = true
bootstrap = true
acl = {
  enabled = true
  default_policy = "deny"
  enable_token_persistence = true
}
```

##### 启动与鉴权

```bash
systemctl enable consul.service
systemctl start consul.service
# 获取登录密码
consul acl bootstrap
# 记录 SecretID
```

### 部署Blackbox Manager

##### Consul字段设计说明

- 所有数据存在一个Services项中，每个监控目标为一个Service。
- 每个Service存一个Tag：目标属于Prometheus的JOB名称。
- 每个Service使用Meta的kv存监控目标的明细：
- `module`，`company`，`project`，`env`，`name`，`instance`
- 分别表示：JOB名称，公司/部门，项目，环境，名称，实例url

##### Web使用说明

- 通过Web界面来对Consul数据增删改查，从而实现对监控目标的管理。
- Web界面可以方便对监控目标分组、分类，方便查询维护。

##### 使用docker-compose来部署

编辑yaml文件，传入3个环境变量：

- consul的token，consul的URL，登录Blackbox Manager的密码
- 启动：`docker-compose up -d`
- 登录：`http://{IP}:1026`

```yaml
cat docker-compose.yml
version: "3.8"
services:
  flask-consul:
    image: registry.cn-shenzhen.aliyuncs.com/starsl/flask-consul:latest
    container_name: flask-consul
    hostname: flask-consul
    restart: always
    volumes:
      - /usr/share/zoneinfo/PRC:/etc/localtime
    environment:
      consul_token: xxxxx-xxxxx-xxxxx
      consul_url: http://x.x.x.x:8500/v1
      admin_passwd: xxxxxxxx
  nginx-consul:
    image: registry.cn-shenzhen.aliyuncs.com/starsl/nginx-consul:latest
    container_name: nginx-consul
    hostname: nginx-consul
    restart: always
    ports:
      - "1026:1026"
    volumes:
      - /usr/share/zoneinfo/PRC:/etc/localtime
```

### 配置Prometheus

##### 基于Consul实现Prometheus的自动发现功能配置

- 根据Consul每个service的tag来把监控目标关联到Prometheus的JOB。
- 把Consul每个service的Meta的KV关联到Prometheus每个指标的标签。
- 根据标签来对监控目标分类，分组，方便管理维护。

```yaml
vi prometheus.yml
#####blackbox_exporter#####
  - job_name: 'http_2xx'
    metrics_path: /probe
    params:
      module: [http_2xx]
    consul_sd_configs:
      - server: 'x.x.x.x:8500'
        token: 'xxx-xxx-xxx-xxx'
        services: ['blackbox_exporter']
        tags: ['http_2xx']
    relabel_configs:
      - source_labels: ["__meta_consul_service_metadata_instance"]
        target_label: __param_target
      - source_labels: ["__meta_consul_service_metadata_company"]
        target_label: company
      - source_labels: ["__meta_consul_service_metadata_env"]
        target_label: env
      - source_labels: ["__meta_consul_service_metadata_name"]
        target_label: name
      - source_labels: ["__meta_consul_service_metadata_project"]
        target_label: project
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115

  - job_name: 'tcp_connect'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    consul_sd_configs:
      - server: 'x.x.x.x:8500'
        token: 'xxx-xxx-xxx-xxx'
        services: ['blackbox_exporter']
        tags: ['tcp_connect']
    relabel_configs:
      - source_labels: ["__meta_consul_service_metadata_instance"]
        target_label: __param_target
      - source_labels: ["__meta_consul_service_metadata_company"]
        target_label: company
      - source_labels: ["__meta_consul_service_metadata_env"]
        target_label: env
      - source_labels: ["__meta_consul_service_metadata_name"]
        target_label: name
      - source_labels: ["__meta_consul_service_metadata_project"]
        target_label: project
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115

  - job_name: 'http200igssl'
    metrics_path: /probe
    params:
      module: [http200igssl]
    consul_sd_configs:
      - server: 'x.x.x.x:8500'
        token: 'xxx-xxx-xxx-xxx'
        services: ['blackbox_exporter']
        tags: ['http200igssl']
    relabel_configs:
      - source_labels: ["__meta_consul_service_metadata_instance"]
        target_label: __param_target
      - source_labels: ["__meta_consul_service_metadata_company"]
        target_label: company
      - source_labels: ["__meta_consul_service_metadata_env"]
        target_label: env
      - source_labels: ["__meta_consul_service_metadata_name"]
        target_label: name
      - source_labels: ["__meta_consul_service_metadata_project"]
        target_label: project
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115

  - job_name: 'http4xx'
    metrics_path: /probe
    params:
      module: [http4xx]
    consul_sd_configs:
      - server: 'x.x.x.x:8500'
        token: 'xxx-xxx-xxx-xxx'
        services: ['blackbox_exporter']
        tags: ['http4xx']
    relabel_configs:
      - source_labels: ["__meta_consul_service_metadata_instance"]
        target_label: __param_target
      - source_labels: ["__meta_consul_service_metadata_company"]
        target_label: company
      - source_labels: ["__meta_consul_service_metadata_env"]
        target_label: env
      - source_labels: ["__meta_consul_service_metadata_name"]
        target_label: name
      - source_labels: ["__meta_consul_service_metadata_project"]
        target_label: project
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
```

### 配置Blackbox_Exporter

主要是默认的配置，增加了4xx和忽略ssl的模块。

```
cat blackbox.yml
modules:
  http_2xx:
    prober: http
    http:
      valid_status_codes:
      - 200
      - 301
      - 302
      - 303
      no_follow_redirects: true

  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
      - send: "SSH-2.0-blackbox-ssh-check"
  icmp:
    prober: icmp

  http4xx:
    prober: http
    http:
      valid_status_codes:
      - 401
      - 403
      - 404
  http200igssl:
    prober: http
    http:
      valid_status_codes:
      - 200
      tls_config:
        insecure_skip_verify: true
```

### 批量导入脚本

在文本文件中写入监控目标的信息：JOB名称，公司/部门，项目，环境，名称，实例url，每行一个，空格分隔。

执行导入脚本，即可导入所有监控目标到Consul，并符合Prometheus的自动发现配置。

修改脚本中的consul_token和consul_url，即可导入。

```python
#!/usr/bin/python3
import requests,json
consul_token = 'xxxxxxxxxx'
consul_url = 'http://x.x.x.x:8500/v1'
with open('instance.list', 'r') as file:
  lines = file.readlines()
  for line in lines:
    module,company,project,env,name,instance = line.split()
    headers = {'X-Consul-Token': consul_token}
    data = {"id": f"{module}/{company}/{project}/{env}@{name}",
            "name": 'blackbox_exporter',
            "tags": [module],
            "Meta": {'module':module,'company':company,'project':project,'env':env,'name':name,'instance':instance}
           }
    reg = requests.put(f"{consul_url}/agent/service/register", headers=headers, data=json.dumps(data))
    if reg.status_code == 200:
        print({"code": 20000,"data": "增加成功！"})
    else:
        print({"code": 50000,"data": f'{reg.status_code}:{reg.text}'})
```

### 导入Blackbox Exporter Dashboard

- 支持Grafana 8，基于blackbox_exporter 0.19.0设计
- 采用图表+曲线图方式展示TCP，ICMP，HTTPS的服务状态，各阶段请求延时，HTTPS证书信息等
- 优化展示效果，支持监控目标的分组、分类级联展示，多服务同时对比展示。

```
导入ID：9965
详细URL：https://grafana.com/grafana/dashboards/9965
```

### GitHub

所有代码都在里面，抛砖引玉。

```
https://github.com/starsliao/Prometheus/tree/master/Blackbox-Manager
```

