#### 安装Istio

https://istio.io/latest/docs/setup/getting-started/

###### 下载 Istio

下载内容将包含：安装文件、示例和 [istioctl](https://istio.io/latest/zh/docs/reference/commands/istioctl/) 命令行工具。

1. 访问 [Istio release](https://github.com/istio/istio/releases/tag/1.13.2) 页面下载与您操作系统对应的安装文件。在 macOS 或 Linux 系统中，也可以通过以下命令下载最新版本的 Istio：

   ```bash
   $ wget https://github.com/istio/istio/releases/download/1.13.2/istio-1.13.2-linux-amd64.tar.gz
   ```

2. 解压并切换到 Istio 包所在目录下。例如：Istio 包名为 `istio-1.13.2`，则：

   ```bash
   $ tar zxf istio-1.13.2-linux-amd64.tar.gz
   $ ll istio-1.13.2
   drwxr-x---  2 root root    22 Jul 15 13:32 bin
   -rw-r--r--  1 root root 11348 Jul 15 13:32 LICENSE
   drwxr-xr-x  5 root root    52 Jul 15 13:32 manifests
   -rw-r-----  1 root root   854 Jul 15 13:32 manifest.yaml
   -rw-r--r--  1 root root  5866 Jul 15 13:32 README.md
   drwxr-xr-x 20 root root   332 Jul 15 13:32 samples
   drwxr-xr-x  3 root root    57 Jul 15 13:32 tools
   ```

3. 将 `istioctl` 客户端拷贝到 path 环境变量中

   ``` bash
   $ cp bin/istioctl /bin/
   ```

4. 配置命令自动补全

   `istioctl` 自动补全的文件位于 `tools` 目录。通过复制 `istioctl.bash` 文件到您的 home 目录，然后添加下行内容到您的 `.bashrc` 文件执行 `istioctl` tab 补全文件：

   ```bash
   $ cp tools/istioctl.bash ~
   $ source ~/istioctl.bash
   ```



###### 安装istio组件

https://istio.io/latest/zh/docs/setup/install/istioctl/#display-the-configuration-of-a-profile

使用istioctl直接安装：

```bash
$ istioctl install --set profile=demo
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete

$ kubectl -n istio-system get po
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-7bf76dd59-n9t5l     1/1     Running   0          77s
istio-ingressgateway-586dbbc45d-xphjb   1/1     Running   0          77s
istiod-6cc5758d8c-pz28m                 1/1     Running   0          84s
```

istio针对不同的环境，提供了几种不同的初始化部署的[profile](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)

```bash
# 查看提供的profile类型
$ istioctl profile list

# 获取kubernetes的yaml：
$ istioctl manifest generate --set profile=demo > istio-kubernetes-manifest.yaml
```



###### 卸载

```bash
$ istioctl manifest generate --set profile=demo | kubectl delete -f -
```

