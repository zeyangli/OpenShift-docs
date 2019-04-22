# 安装OpenShift (okd3.11)

本次试验采用3台虚拟机，每台机器即作为node节点又作为master节点使用。

## 1.安装前准备

### 1.1 主机划分

| 主机名称 | IP地址 | 系统版本 | 资源配置 |
| ------ | ------ | ------ | ------ |
| node01.example.com | 192.168.0.21 | Centos7.6 | 1C4G |
| node02.example.com | 192.168.0.32 | Centos7.6 | 1C4G |
| node03.example.com | 192.168.0.43 | Centos7.6 | 1C4G |

将上述表格配置添加到/etc/hosts文件中

```
192.168.0.21 node01.example.com
192.168.0.32 node02.example.com 
192.168.0.43 node03.example.com
```

### 1.2 SSH免密交互
此处配置node01能够免密登录其他机器即可，如需所有机器免密则在所有节点执行以下操作。

```
ssh-keygen  #生成秘钥 一路回车
ssh-copy-id node01.example.com   # 输入yes，输入密码。
ssh-copy-id node02.example.com   # 输入yes，输入密码。
ssh-copy-id node03.example.com   # 输入yes，输入密码。
```

### 1.3 更新操作系统组件
我使用的是centos7.3的镜像安装的虚拟机，在这里可以通过以下命令升级操作系统为最新(目前centos7.6)。

```
yum update -y
yum install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct -y
reboot
```

### 1.4 获取安装脚本
使用下面的源安装ansible2.7，2.4版本的不能使用。
```
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo
yum -y --enablerepo=epel install ansible pyOpenSSL
```

下载ansible安装的playbook脚本。

```
git clone https://github.com/openshift/openshift-ansible
cd openshift-ansible
git checkout release-3.11     #切换到对应版本的分支
```

### 1.5 导入需要的Image镜像
需要的镜像列表

```
[root@node01 ~]# docker images
REPOSITORY                                           TAG                 IMAGE ID            CREATED             SIZE
docker.io/openshift/origin-node                      v3.11.0             556a4e6d52cb        44 hours ago        1.17 GB
docker.io/openshift/origin-control-plane             v3.11               24459e9dfa74        44 hours ago        829 MB
docker.io/openshift/origin-control-plane             v3.11.0             24459e9dfa74        44 hours ago        829 MB
docker.io/openshift/origin-haproxy-router            v3.11.0             ef1088298e6a        44 hours ago        410 MB
docker.io/openshift/origin-deployer                  v3.11.0             5d3c9c639024        44 hours ago        384 MB
docker.io/openshift/origin-template-service-broker   v3.11.0             0f87900796d2        44 hours ago        335 MB
docker.io/openshift/origin-pod                       v3.11.0             2f0a5aac5c39        44 hours ago        262 MB
docker.io/openshift/origin-docker-registry           v3.11.0             9dffb2abf1dd        8 weeks ago         310 MB
docker.io/openshift/origin-console                   v3.11.0             ab1db955ef9e        2 months ago        264 MB
docker.io/openshift/origin-service-catalog           v3.11.0             eda829aae0a0        2 months ago        330 MB
docker.io/openshift/origin-web-console               v3.11.0             be30b6cce5fa        5 months ago        339 MB
docker.io/tripleorocky/coreos-prometheus-operator    v0.23.2             835a7e260b35        7 months ago        47 MB
quay.io/coreos/prometheus-operator                   v0.23.2             835a7e260b35        7 months ago        47 MB
docker.io/grafana/grafana                            5.2.1               1bfead9ff707        9 months ago        245 MB
quay.io/coreos/etcd                                  v3.2.22             ff5dd2137a4f        9 months ago        37.3 MB
docker.io/openshift/oauth-proxy                      v1.1.0              90c45954eb03        13 months ago       235 MB

```
下载镜像包导入镜像 [镜像压缩包](../others/images)

```
docker load -i *.tar.gz 
```


## 2. 开始安装集群

### 2.1 准备部署的hosts文件

```
[root@master ~]# cat /etc/ansible/hosts
[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
ansible_ssh_user=root
openshift_deployment_type=origin
#因采用虚拟机部署学习 配置此选项跳过主机硬件信息检查
openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability
openshift_master_identity_providers=[{'name':'htpasswd_auth','login':'true','challenge':'true','kind':'HTPasswdPasswordIdentityProvider',}]

openshift_deployment_type=origin
os_firewall_use_firewalld=true

[masters]
node01.example.com
node02.example.com
node03.example.com

[etcd]
node01.example.com
node02.example.com
node03.example.com

[nodes]
node01.example.com openshift_node_group_name='node-config-master'
node01.example.com openshift_node_group_name='node-config-compute'
node02.example.com openshift_node_group_name='node-config-master'
node02.example.com openshift_node_group_name='node-config-compute'
node03.example.com openshift_node_group_name='node-config-master'
node03.example.com openshift_node_group_name='node-config-compute'

```

### 2.2 开始部署

```
ansible-playbook openshift-ansible/playbooks/prerequisites.yml   #执行安装前检查

ansible-playbook openshift-ansible/playbooks/deploy_cluster.yml   #真正的安装集群

``` 


### 2.3 部署测试
创建管理员账号

```
htpasswd -b /etc/origin/master/htpasswd admin admin
oc login -u system:admin
oc adm policy add-cluster-role-to-user cluster-admin admin
```

登录页面
![iamges](../images/01.png)

首页
![iamges](../images/02.png)


## FAQ

### 1.执行安装monitoring的时候出现错误
通过oc get pod -n openshift-monitoring 查看到pod的状态是镜像错误，需要下载prometheus-operator:v0.23.2失败。

```
docker pull tripleorocky/coreos-prometheus-operator:v0.23.2

docker tag tripleorocky/coreos-prometheus-operator:v0.23.2 quay.io/coreos/prometheus-operator:v0.23.2
```

如果安装的时候没有给node打上对应的label标记，也会出现调度错误。需要的label,所有节点需要执行。（具体的label可以通过oc edit pod获取）

```
oc label node node01.example.com node-role.kubernetes.io/infra=true
```


### 2. 执行安装web-console出现错误
跟上面的问题类似，这次是label问题（调度失败），需要给节点打上对应的label标记。

```
 oc label node node01.example.com node-role.kubernetes.io/master=true
```

### 3. service-catalog失败

```
  Normal   Pulled     18m   kubelet, node01.example.com  Container image "docker.io/openshift/origin-service-catalog:v3.11.0" al
ready present on machine  Normal   Created    18m   kubelet, node01.example.com  Created container
  Normal   Started    18m   kubelet, node01.example.com  Started container
  Warning  Unhealthy  15m   kubelet, node01.example.com  Liveness probe failed: Get https://10.129.0.11:6443/healthz: net/http: 
request canceled (Client.Timeout exceeded while awaiting headers)
```

删除pod重新创建解决此问题。

### 4.导出镜像
将镜像名称存放到images.txt文件中

```
[root@node01 ~]# docker images | awk '{print $1":"$2}'
REPOSITORY:TAG
quay.io/coreos/cluster-monitoring-operator:v0.1.1
docker.io/openshift/origin-node:v3.11.0
docker.io/openshift/origin-control-plane:v3.11
docker.io/openshift/origin-control-plane:v3.11.0
docker.io/openshift/origin-haproxy-router:v3.11.0
docker.io/openshift/origin-deployer:v3.11.0
docker.io/openshift/origin-template-service-broker:v3.11.0
docker.io/openshift/origin-pod:v3.11.0
docker.io/openshift/origin-docker-registry:v3.11.0
docker.io/openshift/origin-console:v3.11.0
docker.io/openshift/origin-service-catalog:v3.11.0
docker.io/openshift/origin-web-console:v3.11.0
docker.io/tripleorocky/coreos-prometheus-operator:v0.23.2
quay.io/coreos/prometheus-operator:v0.23.2
docker.io/grafana/grafana:5.2.1
quay.io/coreos/etcd:v3.2.22
docker.io/openshift/oauth-proxy:v1.1.0
```

导出

```
for image in `cat images.txt`
do 
    zipname=`echo ${image} | awk -F / '{print $3}'`
    docker save ${image} > images/${zipname}.tar.gz
done

```
