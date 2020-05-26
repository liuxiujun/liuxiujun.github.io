---
title: Python实现微服务从听过到略懂
date: 2019-12-30 08:45:26
tags:
	- Python
	- 后端
---

## 准备

python3, flask, python-consul, docker

**使用docker 拉取 consul**

``` shell
docker pull consul
```

**使用docker-compose 编排 consul 容器， 为了快速搞定注册发现这块， 启一个节点测试用， 生产环境别这么搞**

``` shell
version: '3.1'
services:
  consul:
    image: consul
    hostname: consul
    container_name: badger
    ports:
      - 8500:8500
      - 8600:8600/udp
    command: agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
```

**切换到 docker-compose.yml 所在目录运行**

``` shell
docker-compose up -d
```

**访问8500**

...

**测试consul DNS 解析**

``` shell
dig @127.0.0.1 -p 8600 consul.service.dc1.consul. ANY
```

**~~本机如果没有dig命令， 替代方法~~**

``` shell
docker run --rm  azukiapp/dig:latest dig @0.0.0.0 -p 8600 consul.service.dc1.consul. ANY
```

上面的命令有问题...换成宿主机ip也不成？ 先不研究这个，安装dig命令吧

**返回结果应该包含节点IP地址**

``` shell
; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 8600 consul.service.dc1.consul. ANY
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16647
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;consul.service.dc1.consul.	IN	ANY

;; ANSWER SECTION:
consul.service.dc1.consul. 0	IN	A	172.26.0.2
consul.service.dc1.consul. 0	IN	TXT	"consul-network-segment="

;; Query time: 15 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Mon Dec 30 09:05:44 CST 2019
;; MSG SIZE  rcvd: 106
```

### 搭建 flask 开发环境

**create venv**

``` shell
mkdir myproject && cd myproject && python3 -m venv venv
```

**active venv**

``` shell
. venv/bin/activate
```

**install flask**

``` shell
pip install flask
pip install python-consul
```



## 现在正式开始

**编写第一个服务, firstapp.py**

``` python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    ver="1.0"
    res='{"Service":"first app", "Version":' + ver + '}\n'
    return res
```

**启动第一个服务**

``` shell
export FLASK_ENV=development && export FLASK_APP=firstapp.py && flask run --host=0.0.0.0 --port=5000
```

**编写第二个服务， secondapp.py**

``` python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    ver="1.0"
    res='{"Service":"second app", "Version":' + ver + '}\n'
    return res
```

**启动第二个服务**

``` shell
export FLASK_ENV=development && export FLASK_APP=secondapp.py && flask run --host=0.0.0.0 --port=5001
```

**将两个服务注册到consul的脚本**

``` shell
function register() {
	# 查询当前服务IP地址, 默认情况下docker容器内访问宿主机不能用127.0.0.1或localhost
	addr=`/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"`

	# 注销服务
	curl -X PUT http://127.0.0.1:8500/v1/agent/service/deregister/$1

	# 注册服务
	curl -X PUT -d '{
		"id": "'$1'", 
		"name": "'$1'", 
		"address": "'${addr}'", 
		"port": '$2', 
		"tags": [""], 
		"checks": [
			{
				"http": "http://'${addr}':'$2'",
				"interval": "5s"
			}
		]
	}' http://127.0.0.1:8500/v1/agent/service/register

	# 查询指定节点以及指定的服务信息
	curl http://127.0.0.1:8500/v1/catalog/service/$1
}

servs=('firstapp 5000' 'secondapp 5001')

for i in "${servs[@]}"; do
	serv=($i)
	name="${serv[0]}"
	port="${serv[1]}"
	register ${name} ${port}
done
```

**或者用另一种方式， 在代码里使用 python-consul注册服务， 修改 firstapp.py**

``` python
from flask import Flask
import consul
import subprocess

app = Flask(__name__)

cmd = "/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d \"addr:\""
host = subprocess.check_output([cmd], shell=True).decode("utf-8").strip()
port = 5000

c = consul.Consul(host='127.0.0.1',
                  port=8500,
                  scheme='http',
                  token=None,
                  verify=True,
                  cert=None)

c.agent.service.register(name='firstapp',
                         service_id='firstapp',
                         address=host,
                         port=port,
                         tags=None,
                         check=consul.Check().tcp(host=host,
                                                  port=port,
                                                  interval="5s"))

@app.route('/')
def index():
    ver = "1.0"
    res = '{"Service":"first app", "Version":' + ver + '}\n'
    return res
```

**修改secondapp.py**

```python
from flask import Flask
import consul
import subprocess
import requests

app = Flask(__name__)
cmd = "/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d \"addr:\""
host = subprocess.check_output([cmd], shell=True).decode("utf-8").strip()
port = 5001

c = consul.Consul(host='127.0.0.1',
                  port=8500,
                  scheme='http',
                  token=None,
                  verify=True,
                  cert=None)

c.agent.service.register(name='secondapp',
                         service_id='secondapp',
                         address=host,
                         port=port,
                         tags=None,
                         check=consul.Check().tcp(host=host,
                                                  port=port,
                                                  interval="5s"))

def getServiceAddress(name):
    services = c.agent.services()
    service = services.get(name)
    if not service:
        return None, None
    addr = "{0}:{1}".format(service['Address'], service['Port'])
    return service, addr


@app.route('/')
def index():
    ver = "1.0"
    res = '{"Service":"second app", "Version":' + ver + '}\n'
    s = c.Agent.services(c.agent)['firstapp']
    serivce, addr = getServiceAddress("firstapp")
    res += requests.get("http://{0}".format(addr), ).content.decode('utf-8')
    return res
```

到这就算成功略懂了


