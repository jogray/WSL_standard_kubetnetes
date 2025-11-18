默认情况下，Kubernetes 控制平面节点不会调度工作负载（Pod），这是通过**污点（Taint）** 机制实现的。要让单控制平面集群运行工作负载，你有几种方法：

## 方法1：移除控制平面节点的污点（推荐用于开发/测试）

这是最简单的方法，允许在控制平面节点上调度任何 Pod。

```bash
# 查看节点的污点
kubectl describe node $(kubectl get nodes -o name | cut -d/ -f2) | grep Taint

# 移除控制平面节点的污点
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-  # 旧版本
```

**验证：**
```bash
# 现在应该可以看到污点已移除
kubectl describe node $(kubectl get nodes -o name | cut -d/ -f2) | grep Taint

# 尝试部署一个测试 Pod
kubectl run test-pod --image=nginx --restart=Never
kubectl get pods -o wide  # 应该能看到 Pod 被调度到控制平面节点
```

## 方法2：在初始化时禁用污点

在 `kubeadm init` 时直接配置不添加控制平面污点：

```bash
# 创建配置文件
cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  taints: []  # 空数组表示不添加污点
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
EOF

# 使用配置文件初始化
sudo kubeadm init --config kubeadm-config.yaml
```

## 方法3：为特定工作负载容忍污点（生产环境推荐）

在生产环境中，更好的做法是保持控制平面节点的污点，只为特定的工作负载添加容忍（Toleration）。

### 示例1：Deployment 中添加容忍
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

### 示例2：使用 nodeSelector 和 tolerations 结合
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: critical-app
  template:
    metadata:
      labels:
        app: critical-app
    spec:
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: app
        image: my-critical-app:latest
```

## 方法4：使用 DaemonSet 在控制平面运行特定服务

对于需要在所有节点（包括控制平面）运行的服务，如监控代理、日志收集器等：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
spec:
  selector:
    matchLabels:
      name: monitoring-agent
  template:
    metadata:
      labels:
        name: monitoring-agent
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: agent
        image: monitoring-agent:latest
```

## 完整的工作流程示例

```bash
# 1. 初始化集群（如果需要）
sudo kubeadm init

# 2. 配置 kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 3. 安装网络插件（必须）
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 4. 移除控制平面污点（用于开发环境）
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

# 5. 验证节点可调度
kubectl get nodes -o wide

# 6. 部署测试工作负载
kubectl create deployment nginx --image=nginx
kubectl scale deployment nginx --replicas=3

# 7. 检查 Pod 调度
kubectl get pods -o wide
```

## 生产环境建议

对于生产环境，建议采用以下策略：

1. **保持控制平面污点**，确保关键组件不受干扰
2. **只为必要的系统工作负载添加容忍**（如监控、日志代理）
3. **考虑资源限制**，控制平面节点资源通常应优先保障控制平面组件
4. **如果需要运行应用工作负载**，建议添加工作节点到集群：

```bash
# 在工作节点上执行（从 kubeadm init 输出中获取命令）
kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## 注意事项

- **资源考虑**：控制平面节点运行工作负载会消耗额外资源，可能影响集群稳定性
- **安全考虑**：在生产环境中，避免在控制平面运行不受信任的工作负载
- **备份重要数据**：定期备份 etcd 数据
- **监控资源使用**：密切监控控制平面节点的 CPU、内存和磁盘使用情况

选择哪种方法取决于你的具体需求：开发测试环境通常使用方法1，生产环境建议使用方法3或添加工作节点。