# k8s-rke2-rancher
k8s-rke2-rancher

# Ansible + RKE2 + Rancher 三节点 Kubernetes 集群部署



### 一、概述

在 **Kylin** 11 上通过 Ansible 自动化部署 1 节点 master + 2 节点 worker 的 K8s 1.36.3 测试集群并集成 **Rancher**，**技术栈是 Ansible 2.15+ + RKE2 v1.36.3+rke2r1（tar 包方式）+ Canal CNI + Traefik（v1.36 起为默认 Ingress）+ Rancher v2.15.2（通过 kubernetes.core.helm 部署）**。选择 **RKE2** 而非 kubeadm/K3s 的原因：它由 **Rancher** 官方维护、与 **Rancher** 管理面天然兼容，内置 **containerd** 与 **SELinux** 支持，在 openEuler 内核的麒麟 V11 上兼容性最好；且 v1.36 起官方已将 ingress-nginx 弃用，Traefik 成为新集群默认 Ingressrke2。



### 升级通知

在从早期版本进行升级之前，请务必阅读《Kubernetes 紧急升级说明》，如下是官网链接：

https://docs.rke2.io/release-notes/v1.36.X


<img width="2535" height="1466" alt="image" src="https://github.com/user-attachments/assets/ad11a7d1-3f95-48e1-8409-453d6f41c13e" />





## 二、架构与节点规划



### 2.1 **核心组件版本说明**

| 组件           | 系统版本                 | 说明                                                         |
| -------------- | ------------------------ | ------------------------------------------------------------ |
| 操作系统       | 麒麟 V11 (Swan25) x86_64 | 内核 ≥ 6.6                                                   |
| Kubernetes     | v1.36.3+rke2r1           | v1.36.3 已有多个安全补丁，生产应使用最新 patch 版本          |
| RKE2           | rke2r1                   | 2026-07 发布周期，containerd v2.3.3-k3s1，etcd 3.6.12        |
| Rancher        | v2.15.2                  | Rancher 2.15 已支持 RKE2 v1.36 作为下游集群，全生命周期集群管理 |
| Containerd     | containerd 1.7.23        | 容器运行时 RKE2 内置，符合 k8s 1.36 运行时标准               |
| CNI 网络插件   | Canal                    | 生成推荐 Canal；若用 Calico 需额外装 Tigera SELinux 策略     |
| Ingress 控制器 | Nginx Ingress 1.10.x     | RKE2 内置，七层路由转发                                      |





### 2.2 **集群节点规划E**

| 角色    | 主机名    | IP              | 规格       | 组件                                      |
| ------- | --------- | --------------- | ---------- | ----------------------------------------- |
| master  | k8smaster | 192.168.230.140 | 4C/8G/100G | rke2-server、etcd + controlplane、Rancher |
| worker1 | k8snode1  | 192.168.230.141 | 4C/8G/100G | rke2-agent                                |
| worker2 | k8snode2  | 192.168.230.142 | 4C/8G/100G | rke2-agent                                |



### 2.3 **端口开通清单**（三台节点互通）

| 端口      | 协议 | 用途                                |
| --------- | ---- | ----------------------------------- |
| 6443      | TCP  | K8s API                             |
| 9345      | TCP  | RKE2 节点注册                       |
| 10250     | TCP  | kubelet                             |
| 8472/UDP  | UDP  | Flannel VXLAN                       |
| 80/443    | TCP  | Traefik Ingress（Rancher 访问入口） |
| 2379-2380 | TCP  | etcd（单 master 仅本机使用）        |



### 2.4  前置条件检查（三台节点）

- 磁盘根分区 ≥ 60G，余量 ≥ 30%（RKE2 镜像 + Rancher 镜像约 4~5G）
- 三台节点之间 SSH root 免密互通（ansible 控制机 → 三节点）
- 防火墙放行：6443、9345、10250、8472/udp（Flannel VXLAN）、80/443
- 主机名唯一且写入系统文件 `/etc/hosts`
- swap 已关闭（RKE2 安装程序会自动处理，提前处理更稳）



**Ansible 控制节点要求**：任意一台 Linux 工作机（可在 master 上），安装 Ansible ≥2.14、`kubernetes.core` collection（用于 helm 模块）、SSH 免密登录到三台节点、root 权限（或 sudo）。





## 三、项目代码目录结构

![图片](https://mmbiz.qpic.cn/mmbiz_png/2HHcAN85a8PEWbPicFtdicuxBQCVmeJzicKjmyBlnxbubyvdaGI9PtQe2AVicjmmpBm9NcVEnXahkdhyLOkdTPc10v1DX5ySAuHn0FMicDZylPVg/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=12)

```
.
├── ansible.cfg
├── images
│   ├── cert-manager-v1.18.2.tgz
│   ├── helm-v3.16.4-linux-amd64.tar.gz
│   ├── rancher-2.15.2.tgz
│   ├── rke2-images.linux-amd64.tar.gz
│   ├── rke2-images.linux-amd64.tar.zst
│   ├── rke2-images.linux-amd64.txt
│   └── sha256sum-amd64.txt
├── inventory
│   ├── group_vars
│   │   ├── all.yml                  # 全局变量：版本、镜像源、密码等
│   │   ├── master.yml
│   │   └── workers.yml
│   └── hosts.yml
├── roles
│   ├── common                       # 所有节点：系统初始化
│   │   ├── handlers
│   │   │   └── main.yml
│   │   └── tasks
│   │       └── main.yml             # 系统初始化
│   ├── rancher                      # master：部署 Rancher + cert-manager
│   │   ├── defaults
│   │   │   └── main.yml
│   │   └── tasks
│   │       └── main.yml
│   ├── rke2-agent                   # worker：安装 agent
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── config.yaml.j2
│   └── rke2-server                  # master：安装 server
│       ├── handlers
│       │   └── main.yml
│       ├── tasks
│       │   └── main.yml
│       └── templates
│           └── config.yaml.j2
└── site.yml                         # 主 Playbook
```



## 四、Ansible 配置文件

### 4.1 ansible.cfg

```
[defaults]
inventory = /root/rke2-rancher/inventory/hosts.yml
remote_user = root
forks = 10
deprecation_warnings = False
stdout_callback = yaml
host_key_checking = False
#roles_path = /root/ansible/roles:/usr/share/ansible/roles/
#collections_paths = /root/ansible/collections

#[ssh_connection]
#ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=/dev/null
#pipelining = True

[privilege_escalation]
become=True
become_method=sudo
become_user=root
bacome_ask_pass=False

```





### 4.2 inventory/hosts.yml

```
all:
  children:
    master:
      hosts:
        k8s-master:
          ansible_host: 192.168.230.140
          node_name: k8smaster
    workers:
      hosts:
        k8s-node1:
          ansible_host: 192.168.230.141
          node_name: k8snode1
        k8s-node2:
          ansible_host: 192.168.230.142
          node_name: k8snode2
  vars:
    ansible_user: root
    ansible_ssh_private_key_file: ~/.ssh/id_rsa


```



### 4.3 inventory/group_vars/all.yml

这个文件是 Ansible 项目的 **全局变量文件**（通常放在 `group_vars/all.yml`），作用是**把所有“可能需要调整的参数”集中在一个地方**，其他剧本/角色通过变量名引用。改配置只需要改这一个文件，不用到处翻剧本——这是自动化运维的标准做法。

```
---
# ====== 集群全局变量（按需修改） ======
rke2_version: "v1.36.3+rke2r1"  # 要安装的 RKE2 版本号。格式含义：v1.36.3 是 Kubernetes 版本，+rke2r1 是 Rancher 的第 1 次打包发布（RKE2 会把 K8s 和配套组件打包在一起）。
rke2_install_method: "tar"          # Kylin 必须 tar，避免 rpm 仓库兼容问题
cluster_token: "kylin-test-cluster-token-2026"     # 集群加入令牌
master_ip: "192.168.230.140"
master_hostname: "k8smaster"

# ====== 网段规划 ======
cluster_cidr: "10.244.0.0/16"       # Pod 网段
service_cidr: "10.96.0.0/12"        # Service 网段（ClusterIP）
cluster_dns: "10.244.0.10"          # CoreDNS 的固定 IP

# ====== CNI 选择 ======
cni: "canal"                        # Flannel（网络）+ Calico（策略）

# ====== Kylin 系统参数 ======
enable_firewalld_disable: true
enable_selinux_disable: true
#rke2_selinux: false

# ====== Rancher ======
rancher_hostname: "192.168.230.140.sslip.io"   # 内网无 DNS 时的魔法域名
rancher_bootstrap_password: "Rancher@2026Test"
rancher_version: "2.15.2"
rancher_replicas: 1
cert_manager_version: "v1.18.2"

# ====== Helm ======
helm_version: "v3.16.3"


```



### 4.4 inventory/group_vars/master.yml

是 **Ansible 变量作用域** 的经典用法，**作用**：告诉剧本“这台机器装的是 RKE2 的 **server 端**（控制面）”，而不是 agent 端。这个变量后续会决定：

```
---
rke2_type: "server"      # 给变量 rke2_type 赋值为 "server"

```



### 4.5 inventory/group_vars/workers.yml

```
---
rke2_type: "agent"

```



## 五、角色实现

### 5.1 roles/common/tasks/main.yml（系统初始化）

```
---
- name: 设置主机名
  ansible.builtin.hostname:
    name: "{{ node_name }}"
    use: systemd

- name: 写入 hosts 解析
  ansible.builtin.copy:
    dest: /etc/hosts
    content: |
      127.0.0.1   localhost
      192.168.230.140 k8smaster
      192.168.230.141 k8snode1
      192.168.230.142 k8snode2
    mode: '0644'

- name: 关闭 swap
  ansible.builtin.shell: |
    swapoff -a
    sed -ri 's@^(/.*swap.*)@#\1@' /etc/fstab
  changed_when: false

- name: 加载内核模块
  ansible.builtin.copy:
    dest: /etc/modules-load.d/k8s.conf
    content: |
      overlay
      br_netfilter
      ip_vs
      ip_vs_rr
      nf_conntrack
    mode: '0644'
  register: modules_conf

- name: 立即加载内核模块
  ansible.builtin.shell: |
    modprobe overlay
    modprobe br_netfilter
    modprobe ip_vs || true
    modprobe ip_vs_rr || true
    modprobe nf_conntrack || true
  changed_when: false

- name: 配置内核参数
  ansible.builtin.copy:
    dest: /etc/sysctl.d/k8s.conf
    content: |
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1
      net.core.somaxconn                  = 32768
      net.netfilter.nf_conntrack_max      = 1048576
      fs.inotify.max_user_instances       = 8192
      fs.inotify.max_user_watches         = 1048576
      vm.swappiness                       = 0
      vm.max_map_count                    = 262144
    mode: '0644'
  notify: reload_sysctl

- name: 关闭 firewalld（测试环境从简）
  ansible.builtin.systemd:
    name: firewalld
    state: stopped
    enabled: false
  when: enable_firewalld_disable | bool
  failed_when: false

- name: 设置 SELinux 为 permissive
  ansible.builtin.shell: |
    setenforce 0 2>/dev/null || true
    sed -ri 's@^SELINUX=enforcing@SELINUX=permissive@' /etc/selinux/config
  when: enable_selinux_disable | bool
  changed_when: false

- name: 安装基础依赖
  ansible.builtin.dnf:
    name:
      - curl
      - wget
      - jq
      - socat
      - conntrack
      - ipset
      - iptables
      - nfs-utils
      - iscsi-initiator-utils
      - lsof
      - vim
      - tar
      - bash-completion
      - chrony
    state: present

- name: 启动 iscsid（Longhorn 依赖）
  ansible.builtin.systemd:
    name: iscsid
    state: started
    enabled: true
  failed_when: false

- name: 启动 chronyd 并同步时间
  ansible.builtin.systemd:
    name: chronyd
    state: started
    enabled: true

- name: 创建数据目录
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - /var/lib/rancher
    - /var/lib/kubelet
    - /var/lib/longhorn
    - /etc/rancher/rke2


```



### 5.2 roles/rke2-server/tasks/main.yml（master 节点）

```
---
- name: 检查 rke2 是否已安装
  ansible.builtin.stat:
    path: /usr/local/bin/rke2
  register: rke2_binary

- name: 通过 tar 方式安装 rke2-server
  ansible.builtin.shell: |
    curl -sfL https://get.rke2.io | \
      INSTALL_RKE2_METHOD={{ rke2_install_method }} \
      INSTALL_RKE2_VERSION={{ rke2_version }} \
      INSTALL_RKE2_TYPE=server \
      sh -
  args:
    creates: /usr/local/bin/rke2
  when: not rke2_binary.stat.exists

- name: 渲染 rke2 server 配置
  ansible.builtin.template:
    src: config.yaml.j2
    dest: /etc/rancher/rke2/config.yaml
    mode: '0600'
  notify: restart_rke2_server

- name: 启动并启用 rke2-server
  ansible.builtin.systemd:
    name: rke2-server
    state: started
    enabled: true
  register: rke2_start

- name: 等待 rke2-server 就绪（最多 10 分钟）
  ansible.builtin.shell: |
    /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes 2>/dev/null
  register: kubectl_out
  until: kubectl_out.rc == 0
  retries: 60
  delay: 10
  changed_when: false

- name: 分发 kubectl 到系统路径
  ansible.builtin.file:
    src: /var/lib/rancher/rke2/bin/kubectl
    dest: /usr/local/bin/kubectl
    state: link
    force: true

- name: 配置 root 的 kubeconfig
  ansible.builtin.file:
    path: /root/.kube
    state: directory
    mode: '0755'

- name: 拷贝 kubeconfig 到 ~/.kube/config
  ansible.builtin.copy:
    src: /etc/rancher/rke2/rke2.yaml
    dest: /root/.kube/config
    remote_src: true
    mode: '0600'

- name: 读取 node-token 供 agent 使用
  ansible.builtin.slurp:
    src: /var/lib/rancher/rke2/server/node-token
  register: node_token_b64
  run_once: true

- name: 将 node-token 存为 fact 供 agent play 使用
  ansible.builtin.set_fact:
    cluster_node_token: "{{ node_token_b64.content | b64decode | trim }}"
  delegate_to: "{{ item }}"
  delegate_facts: true
  loop: "{{ groups['workers'] }}"
  run_once: true


```

### 5.3  roles/rke2-server/templates/config.yaml.j2

```
# RKE2 Server 配置（由 Ansible 渲染）
token: {{ cluster_token }}
node-name: {{ node_name }}
write-kubeconfig-mode: "0644"

# TLS SAN 必须包含本机 IP 与主机名，供 Rancher Agent / kubectl 访问
tls-san:
  - {{ master_ip }}
  - {{ master_hostname }}
  - {{ ansible_host }}

# CNI 与网段
cni: {{ cni }}
cluster-cidr: "{{ cluster_cidr }}"
service-cidr: "{{ service_cidr }}"
cluster-dns: {{ cluster_dns }}

# 测试环境允许 master 调度工作负载，不配置 node-taint

# 精简测试环境的可选组件
disable:
  - rke2-snapshot-controller-crd
  - rke2-snapshot-controller
  - rke2-snapshot-validation-webhook

# 运行时
selinux: false


```



### 5.4  roles/rke2-server/handlers/main.yml

```
---
- name: restart_rke2_server
  ansible.builtin.systemd:
    name: rke2-server
    state: restarted

```



### 5.5  roles/rke2-agent/tasks/main.yml（worker 节点）

```
---
- name: 检查 rke2 是否已安装
  ansible.builtin.stat:
    path: /usr/local/bin/rke2
  register: rke2_binary

- name: 通过 tar 方式安装 rke2-agent
  ansible.builtin.shell: |
    curl -sfL https://get.rke2.io | \
      INSTALL_RKE2_METHOD={{ rke2_install_method }} \
      INSTALL_RKE2_VERSION={{ rke2_version }} \
      INSTALL_RKE2_TYPE=agent \
      sh -
  args:
    creates: /usr/local/bin/rke2
  when: not rke2_binary.stat.exists

- name: 渲染 rke2 agent 配置
  ansible.builtin.template:
    src: config.yaml.j2
    dest: /etc/rancher/rke2/config.yaml
    mode: '0600'
  notify: restart_rke2_agent

- name: 启动并启用 rke2-agent
  ansible.builtin.systemd:
    name: rke2-agent
    state: started
    enabled: true

- name: 等待 agent 节点 Ready（在 master 上查询）
  ansible.builtin.shell: |
    /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml \
      get node {{ node_name }} --no-headers 2>/dev/null | awk '{print $2}'
  register: node_status
  until: node_status.stdout == "Ready"
  retries: 60
  delay: 10
  changed_when: false
  delegate_to: "{{ groups['master'][0] }}"
  run_once: false


```



### 5.6  roles/rke2-agent/templates/config.yaml.j2 

```
# RKE2 Agent 配置
server: https://{{ master_ip }}:9345
token: "{{ cluster_node_token }}"
node-name: {{ node_name }}
selinux: false

```

**关键说明**：agent 无需重复配置 `cluster-cidr`/`service-cidr` 等参数，这些必须只由 server 定义一次；agent 通过 `server:` 地址 + `token:` 加入即可，避免 "critical configuration value mismatch" 错误。

https://github.com/ranchergovernment/rke2-ansible

![image-20261012223932747](E:\k8sstudyshare\rke2-rancher\image-20261012223932747.png)





### 5.7 roles/rke2-agent/handlers/main.yml 

```
---
- name: restart_rke2_agent
  ansible.builtin.systemd:
    name: rke2-agent
    state: restarted

```



### 5.8  roles/rancher/tasks/main.yml（部署 Rancher）

```
---
# ========== 1. 安装 helm ==========
- name: 检查 helm 是否已安装
  ansible.builtin.command: helm version --short
  register: helm_check
  failed_when: false
  changed_when: false

- name: 安装 helm
  ansible.builtin.shell: |
    curl -fsSL -o /tmp/get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 /tmp/get_helm.sh && /tmp/get_helm.sh --version {{ helm_version }}
  when: helm_check.rc != 0
  environment:
    KUBECONFIG: /root/.kube/config

# ========== 2. 安装 cert-manager ==========
- name: 添加 jetstack helm 仓库
  kubernetes.core.helm_repository:
    name: jetstack
    repo_url: https://charts.jetstack.io
  environment:
    KUBECONFIG: /root/.kube/config

- name: 部署 cert-manager
  kubernetes.core.helm:
    name: cert-manager
    chart_ref: jetstack/cert-manager
    chart_version: "{{ cert_manager_version }}"
    namespace: cert-manager
    create_namespace: true
    kubeconfig: /root/.kube/config
    values:
      crds:
        enabled: true
    wait: true
    timeout: 10m0s

- name: 等待 cert-manager 就绪
  ansible.builtin.command: >
    kubectl -n cert-manager rollout status deploy/cert-manager --timeout=300s
  environment:
    KUBECONFIG: /root/.kube/config
  changed_when: false

# ========== 3. 安装 Rancher ==========
- name: 添加 rancher helm 仓库
  kubernetes.core.helm_repository:
    name: rancher-stable
    repo_url: https://releases.rancher.com/server-charts/stable
  environment:
    KUBECONFIG: /root/.kube/config

- name: 部署 Rancher
  kubernetes.core.helm:
    name: rancher
    chart_ref: rancher-stable/rancher
    chart_version: "{{ rancher_version }}"
    namespace: cattle-system
    create_namespace: true
    kubeconfig: /root/.kube/config
    values:
      hostname: "{{ rancher_hostname }}"
      replicas: "{{ rancher_replicas }}"
      bootstrapPassword: "{{ rancher_bootstrap_password }}"
      ingress:
        tls:
          source: rancher   # 自签名证书，测试环境足够
    wait: true
    timeout: 20m0s

- name: 等待 Rancher deployment 就绪
  ansible.builtin.command: >
    kubectl -n cattle-system rollout status deploy/rancher --timeout=600s
  environment:
    KUBECONFIG: /root/.kube/config
  changed_when: false

- name: 打印 Rancher 访问信息
  ansible.builtin.debug:
    msg:
      - "Rancher URL : https://{{ rancher_hostname }}"
      - "账号        : admin"
      - "密码        : {{ rancher_bootstrap_password }}"
      - "如内网无法解析 sslip.io，请在浏览器所在机器 /etc/hosts 加："
      - "  192.168.230.140 {{ rancher_hostname }}"

```





### 5.9  roles/rancher/defaults/main.yml

```
---
helm_version: "v3.16.3"
cert_manager_version: "v1.18.2"
rancher_version: "2.15.2"
rancher_replicas: 1
rancher_hostname: "192.168.230.140.sslip.io"
rancher_bootstrap_password: "Rancher@2026Test"

```





## 六、主 Playbook

### site.yml

```
---
# ========= 1. 所有节点：系统初始化 =========
- name: 所有节点系统初始化
  hosts: all
  become: true
  roles:
    - common

# ========= 2. Master 节点：初始化 RKE2 Server =========
- name: Master 节点安装 rke2-server
  hosts: master
  become: true
  roles:
    - rke2-server

# ========= 3. Worker 节点：加入集群（rke2-agent） =========
- name: Worker 节点安装 rke2-agent 并加入集群
  hosts: workers
  become: true
  pre_tasks:                # ← 加在这里，与 roles 同级
    - name: 读取 master 节点 token
      ansible.builtin.slurp:
        src: /var/lib/rancher/rke2/server/node-token
      delegate_to: "{{ groups['k8smaster'][0] }}"
      run_once: true
      register: _node_token_raw
      no_log: true

    - name: 将 token 设置为本机变量
      ansible.builtin.set_fact:
        cluster_node_token: "{{ _node_token_raw.content | b64decode | trim }}"
      no_log: true
  roles:
    - rke2-agent

# ========= 4. Master 节点：部署 Rancher =========
- name: Master 节点部署 Rancher
  hosts: master
  become: true
  roles:
    - rancher

# ========= 5. 全链路验证 =========
- name: 全链路验证
  hosts: master
  become: true
  tasks:
    - name: 显示节点状态
      ansible.builtin.command: kubectl get nodes -o wide
      environment:
        KUBECONFIG: /root/.kube/config
      register: nodes_out
      changed_when: false

    - name: 显示集群系统 Pod
      ansible.builtin.command: kubectl get pods -A
      environment:
        KUBECONFIG: /root/.kube/config
      register: pods_out
      changed_when: false

    - name: 部署测试应用 nginx
      ansible.builtin.shell: |
        kubectl create deployment nginx-test --image=nginx:alpine --replicas=2 2>/dev/null || true
        kubectl expose deployment nginx-test --port=80 --type=NodePort 2>/dev/null || true
        sleep 15
      environment:
        KUBECONFIG: /root/.kube/config
      changed_when: false

    - name: 显示测试 Pod 状态
      ansible.builtin.command: kubectl get pods -o wide -l app=nginx-test
      environment:
        KUBECONFIG: /root/.kube/config
      register: test_pods
      changed_when: false

    - name: 显示测试 Service
      ansible.builtin.command: kubectl get svc nginx-test
      environment:
        KUBECONFIG: /root/.kube/config
      register: test_svc
      changed_when: false

    - name: 打印汇总
      ansible.builtin.debug:
        msg:
          - "========= 节点状态 ========="
          - "{{ nodes_out.stdout_lines }}"
          - "========= 系统 Pod ========="
          - "{{ pods_out.stdout_lines }}"
          - "========= 测试 Pod ========="
          - "{{ test_pods.stdout_lines }}"
          - "========= 测试 Service ========="
          - "{{ test_svc.stdout_lines }}"


```





## 七、执行步骤

### 7.1 在 Ansible 控制机上操作

```
# 1. 安装 Ansible（如未安装）
dnf install -y ansible || pip3 install ansible

# 2. 安装 kubernetes.core collection（helm 模块依赖）
ansible-galaxy collection install kubernetes.core

# 3. 配置 SSH 免密到三台节点
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
for ip in 192.168.230.140 192.168.230.141 192.168.230.142; do
  ssh-copy-id root@${ip}
done

# 4. 建立项目目录并放入上述文件
mkdir -p rke2-rancher && cd rke2-rancher
# （按上文的目录结构放好各文件）

```

### 7.2 连通性检查

```
ansible -i inventory/hosts.yml all -m ping

预期三台节点均返回 pong
```



### 7.3 执行部署

```
# 先进行语法检查，能提前暴露变量名拼错、YAML 语法错这类问题。
ansible-playbook -i inventory/hosts.yml site.yml --syntax-check

# 更进一步，空跑一遍（不实际执行）检查变量渲染
ansible-playbook -i inventory/hosts.yml site.yml --check --diff

# 完整执行
ansible-playbook -i inventory/hosts.yml site.yml

# 若中途失败想从某一步继续，可用 --start-at-task
ansible-playbook -i inventory/hosts.yml site.yml --start-at-task="部署 Rancher"

# 出问题时打开详细日志
ansible-playbook -i inventory/hosts.yml site.yml -vvv

```



### 7.4 登录检查

- **Rancher UI**：浏览器打开 `https://192.168.230.140.sslip.io`，账号 `admin`，密码 `Rancher@2026Test`
- **kubectl（master 上）**：`kubectl get nodes`
- **测试应用**：Rancher 界面应能看到 `nginx-test` Deployment，NodePort 方式即可在浏览器访问





### 7.5  总结说明

- **Kylin V11 必须用 `INSTALL_RKE2_METHOD=tar`**：RKE2 的 rpm 仓库针对 EL8/EL9 包名，麒麟 V11（openEuler 底层）的 dnf 依赖解析存在差异，tar 包方式完全绕开该问题rke2.io。
- **v1.36 起无需单独装 ingress-nginx**：Traefik 已成为 RKE2 默认 Ingress，Rancher 会直接通过 Traefik 暴露，不要再额外安装 ingress-nginxrke2.io+1。
- **tls-san 必须包含所有能访问 master 的 IP**：包括 `master_ip`、`ansible_host`、`master_hostname`，否则 Rancher Agent 或远程 kubectl 会报证书校验失败。
- **agent 与 server 的 cidr 参数不可重复定义**：`cluster-cidr`/`service-cidr` 等只能在 server 端声明一次，agent 只通过 `server:` + `token:` 加入，避免 critical mismatchgithub.com。
- **幂等性**：本方案所有 role 均支持重复执行，已安装的二进制、已存在的 service、已部署的 helm release 会自动跳过，便于后续扩容或重放。
- **从测试转生产的差异**：生产需 3 server HA + kube-vip VIP + `profile: cis` + etcd S3 备份 + Longhorn 3 副本 + Rancher 2 副本 + 真实 CA 证书；本测试方案在这些方面都做了简化。

完整跑完 `site.yml` 预计 15–40 分钟（取决于网络拉取镜像速度）。

任何一步卡住，优先看目标节点上的 `journalctl -u rke2-server` / `rke2-agent` 日志和 master 上的 `kubectl get events -A --sort-by=.metadata.creationTimestamp`，能定位到 90% 的问题。





https://file.cnxiaobai.com/



