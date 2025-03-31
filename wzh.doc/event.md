# Kubernetes 证书轮换中的事件触发与日志记录机制

在 Kubernetes 集群中，证书轮换是一个关键的安全操作，确保通信安全和身份验证的有效性。本文档将详细介绍在证书轮换过程中，系统是如何触发 Kubernetes 事件（Events）以及如何记录相关日志的。

## 证书轮换控制器概述

证书轮换控制器（CertRotationController）负责管理 Kubernetes 集群中的证书生命周期，包括：

1. 创建和轮换自签名的签名 CA 证书
2. 维护包含所有未过期 CA 证书的 CA 证书包（CA Bundle）
3. 创建和轮换由最新签名 CA 签名的目标证书

## 事件触发机制

在证书轮换过程中，系统会在关键节点触发 Kubernetes 事件，这些事件可以通过 `kubectl get events` 命令查看。

### 1. 签名 CA 证书轮换事件

当签名 CA 证书需要轮换时，系统会触发事件：

```go
c.EventRecorder.Eventf("SignerUpdateRequired", "%q in %q requires a new signing cert/key pair: %v", c.Name, c.Namespace, reason)
```

触发条件包括：
- 证书已过期
- 证书有效期已达到 80%（如果 RefreshOnlyWhenExpired 为 false）
- 达到预设的刷新时间

### 2. CA 证书包更新事件

当 CA 证书包需要更新时，系统会触发事件：

```go
c.EventRecorder.Eventf("CABundleUpdateRequired", "%q in %q requires a new cert", c.Name, c.Namespace)
```

触发条件包括：
- 添加新的 CA 证书到证书包
- 从证书包中移除过期的 CA 证书

### 3. 目标证书轮换事件

当目标证书（如 kubelet 客户端证书）需要轮换时，系统会触发事件：

```go
c.EventRecorder.Eventf("TargetUpdateRequired", "%q in %q requires a new target cert/key pair: %v", c.Name, c.Namespace, reason)
```

触发条件包括：
- 证书已过期
- 证书有效期已达到 80%（如果 RefreshOnlyWhenExpired 为 false）
- 签名 CA 已更改
- 证书的主机名列表发生变化（对于服务证书）

### 4. 控制器状态事件

当证书轮换控制器状态发生变化时，会触发相应事件：

```go
syncCtx.Recorder().Warningf("RotationError", syncErr.Error())
```

这通常发生在轮换过程中遇到错误时。

## 事件记录实现

事件记录通过 `EventRecorder` 接口实现，主要有两种类型的事件：

1. **普通事件（Normal）**：通过 `Event` 或 `Eventf` 方法记录
2. **警告事件（Warning）**：通过 `Warning` 或 `Warningf` 方法记录

事件记录的核心实现：

```go
func (r *recorder) Event(reason, message string) {
    event := makeEvent(r.involvedObjectRef, r.sourceComponent, corev1.EventTypeNormal, reason, message)
    ctx := context.Background()
    if r.ctx != nil {
        ctx = r.ctx
    }
    if _, err := r.eventClient.Create(ctx, event, metav1.CreateOptions{}); err != nil {
        klog.Warningf("Error creating event %+v: %v", event, err)
    }
}
```

## 日志记录机制

除了 Kubernetes 事件外，证书轮换过程中的关键操作也会记录到系统日志中。

### 1. 控制器日志

证书轮换控制器在启动和运行过程中会记录日志：

```go
klog.Infof("Starting CertRotation")
klog.Infof("Waiting for CertRotation")
klog.Infof("Finished waiting for CertRotation")
klog.Infof("Shutting down CertRotation")
```

### 2. 证书更新日志

当 CA 证书包更新时，会记录详细的证书信息：

```go
klog.V(2).Infof("Updated ca-bundle.crt configmap %s/%s with:\n%s", certs.CertificateBundleToString(updatedCerts), caBundleConfigMap.Namespace, caBundleConfigMap.Name)
```

### 3. 错误日志

在证书轮换过程中遇到错误时，会记录错误日志：

```go
klog.Warningf("Error creating event %+v: %v", event, err)
utilruntime.HandleError(fmt.Errorf("%q controller failed to sync %q, err: %w", c.name, key, err))
```

## 证书轮换的触发条件

证书轮换的触发条件主要由以下几个因素决定：

1. **证书有效期**：证书的 NotBefore 和 NotAfter 时间
2. **刷新时间**：预设的刷新间隔（Refresh）
3. **刷新策略**：是否只在过期时刷新（RefreshOnlyWhenExpired）
4. **签名 CA 变化**：签名 CA 是否已更改

具体的判断逻辑如下：

```go
func needNewTargetCertKeyPairForTime(annotations map[string]string, signer *crypto.CA, refresh time.Duration, refreshOnlyWhenExpired bool) string {
    // 检查证书是否已过期
    if time.Now().After(notAfter) {
        return "already expired"
    }

    // 如果设置了只在过期时刷新，则不进行提前刷新
    if refreshOnlyWhenExpired {
        return ""
    }

    // 检查是否达到有效期的 80%
    validity := notAfter.Sub(notBefore)
    at80Percent := notAfter.Add(-validity / 5)
    if time.Now().After(at80Percent) {
        return fmt.Sprintf("past its latest possible time %v", at80Percent)
    }

    // 检查是否达到预设的刷新时间
    refreshTime := notBefore.Add(refresh)
    if time.Now().After(refreshTime) {
        // 确保签名 CA 已有效超过目标刷新时间的 10%
        timeToWaitForTrustRotation := refresh / 10
        if time.Now().After(signer.Config.Certs[0].NotBefore.Add(time.Duration(timeToWaitForTrustRotation))) {
            return fmt.Sprintf("past its refresh time %v", refreshTime)
        }
    }

    return ""
}
```

## 实际应用示例：kube-apiserver 到 kubelet 的客户端证书

以 kube-apiserver 到 kubelet 的客户端证书为例，其配置如下：

```go
certRotator = certrotation.NewCertRotationController(
    "KubeAPIServerToKubeletClientCert",
    certrotation.RotatedSigningCASecret{
        Namespace: operatorclient.OperatorNamespace,
        Name:      "kube-apiserver-to-kubelet-signer",
        AdditionalAnnotations: certrotation.AdditionalAnnotations{
            JiraComponent: "kube-apiserver",
        },
        Validity: 1 * 365 * defaultRotationDay, // 有效期为 1 年
        // 刷新时间设置为有效期的 80%
        Refresh:                292 * defaultRotationDay,
        RefreshOnlyWhenExpired: refreshOnlyWhenExpired,
        Informer:               kubeInformersForNamespaces.InformersFor(operatorclient.OperatorNamespace).Core().V1().Secrets(),
        Lister:                 kubeInformersForNamespaces.InformersFor(operatorclient.OperatorNamespace).Core().V1().Secrets().Lister(),
        Client:                 kubeClient.CoreV1(),
        EventRecorder:          eventRecorder,
        UseSecretUpdateOnly:    true,
    },
    // ... 其他配置 ...
)
```

在这个例子中：
1. 签名 CA 的有效期为 1 年
2. 刷新时间设置为 292 天（约为有效期的 80%）
3. 是否只在过期时刷新由 refreshOnlyWhenExpired 参数控制

## 查看证书轮换事件和日志

### 查看 Kubernetes 事件

```bash
kubectl get events -n openshift-kube-apiserver | grep -i cert
```

### 查看控制器日志

```bash
kubectl logs -n openshift-kube-apiserver-operator deployment/kube-apiserver-operator | grep -i "cert\|rotation"
```

### 查看证书状态

```bash
kubectl get secrets -n openshift-kube-apiserver kube-apiserver-to-kubelet-signer -o yaml
```

## 总结

证书轮换是 Kubernetes 安全机制的重要组成部分。通过事件触发和日志记录，系统管理员可以监控证书的生命周期，及时发现和解决潜在的安全问题。在证书轮换过程中，系统会在关键节点触发 Kubernetes 事件，并记录详细的日志，以便于问题排查和安全审计。
