## 我的WSL一键布置标准k8s控制平面脚本

---
目前测试Ubuntu和Fedora中可以由脚本完成绝大多数下载工作与安装，启动工作不稳定


#### 启动之前
确保你的wsl是在mirrored网络模式启动，并开启了`已启用自动代理`选项
1. 首先更新WSL版本，在这之前请务必保存wsl内的所有工作，并使用`wsl --shutdown`退出WSL
2. 而后按win，在WSL settings将WSL的网络改为mirrored，打开`已启用自动代理`选项
    ```
    wsl --update
    ```
3. 启动一个新的wsl发行版，`wsl --install ubuntu --name ku --location D:\WSL\ku`，任意替换示例中的几个字段以使用你想要的linux版本和wsl名称，以及存储位置，但应当确保该发行版含有glibc，这是启动一个使用默认设置的k8s集群所必须的，参见[关于glibc的官方文档](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#%E5%87%86%E5%A4%87%E5%BC%80%E5%A7%8B)，而后应当确保该linux中有python3及其标准库，以运行脚本，但不需要来自其他任何第三方 \( 如`Pypi` \) 的扩展模块

4. 可以直接复制，克隆，或其他方式，将脚本内容放到该wsl中，譬如直接在github复制脚本内容，而后在wsl中`vi ~/pyinit`，shift + insert粘贴，`:wq`保存退出，如果不是`git clone`的，应当`chomod +x ~/pyinit`使脚本可执行
5. 启动前将脚本顶部的代理变量换成你的本机代理端口，如果你有自己的网关代理或者TUN模式，请删除一切与代理相关的代码，包括代理环境变量配置文件的创建，containerd代理环境配置文件的创建，urllib（python3标准库模块）全局（指脚本执行过程）代理设置，代理环境变量设置，脚本中大小写不限搜索关键词`proxy`即可

### 食用说明
1. 脚本需要 sudo 或 root 用户启动，`sudo ~/pyinit`
2. 各组件下载完成后脚本会尝试`kubeadm init`直接启动控制平面，如果启动失败可以尝试手动`kubeadm init`，不论是脚本或手动，留意kubeadm形如如下示例的输出，这代表启动成功
    ```
    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    Alternatively, if you are the root user, you can run:

    export KUBECONFIG=/etc/kubernetes/admin.conf

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 127.0.0.1:12345 --token mytoken.mymytotokenken \
            --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
    ```
3. 而后参照输出信息的提示，将/etc/kubernetes/admin.conf复制到你想要使用kubectl的用户根目录下的~/.kube/config文件

4. 然后就可以用kubectl 和 crictl 愉快地玩耍了，enjoy it!
5. 此外，k8s默认未实现网络部分，集群还需要网络实现插件，如calico或flannel 简单部署为：
    ```
    kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
    ```
    ```
    k8s@Desktop:~/stdk$ kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
    namespace/kube-flannel created
    serviceaccount/flannel created
    clusterrole.rbac.authorization.k8s.io/flannel created
    clusterrolebinding.rbac.authorization.k8s.io/flannel created
    configmap/kube-flannel-cfg created
    daemonset.apps/kube-flannel-ds created
    k8s@Desktop:~/stdk$ kubectl get nodes
    NAME      STATUS     ROLES           AGE   VERSION
    desktop   NotReady   control-plane   87m   v1.34.2
    k8s@Desktop:~/stdk$ kubectl get nodes
    NAME      STATUS   ROLES           AGE   VERSION
    desktop   Ready    control-plane   91m   v1.34.2
    ```
    可以看见部署网络插件后不久集群node就转为了Ready状态

6. 当然，集群除了控制平面还需要工作节点，kubernetes最佳实践通常用于生产级集群环境，出于安全与稳定性考量默认不允许控制平面部署工作负载，可以参考[来自D指导的回答](./taint.md)
7. 此外，要处理整个集群，参考[启动、停止、重置集群](./administrate)


### 管理集群简明教程
1. `kubectl create deployment --image=nginx:latest`快速部署容器
2. `kubectl get all/deploymnet/replicaset/pod`查看集群运行状态
3. `crictl image/container/network/volume list/rm`查看、管理本地容器状态
