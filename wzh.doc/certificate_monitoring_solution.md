# OpenShift证书更新监控方案

本文档详细说明如何在OpenShift Container Platform (OCP)中实现一个第三方监控方案，用于监督证书的更新过程。该方案将监控证书的生命周期，在证书即将更新以及更新发生时创建事件通知。

## 1. 背景理解

### 1.1 证书轮转控制器概述

在OpenShift中，`CertRotationController`负责管理和自动更新各种证书。从提供的代码中可以看到，该控制器管理多个证书轮转器(certRotators)，每个轮转器负责特定类型的证书。

关键概念：
- **Validity**: 证书的有效期
- **Refresh**: 证书的刷新期，通常设置为有效期的一定比例（如80%）
- **RefreshOnlyWhenExpired**: 是否仅在证书过期时才刷新

例如，对于`KubeAPIServerToKubeletClientCert`：
```go
Validity: 1 * 365 * defaultRotationDay, // 1年有效期
Refresh: 292 * defaultRotationDay,      // 约80%的有效期后刷新
```

### 1.2 证书轮转过程

证书轮转过程大致如下：
1. 控制器定期检查证书的状态
2. 当证书接近刷新期时，系统会生成新的证书
3. 新证书生成后，系统会更新相关的Secret或ConfigMap
4. 相关组件会加载新的证书

## 2. 监控方案设计

### 2.1 方案概述

我们的监控方案将：
1. 监控证书相关的Secret和ConfigMap资源
2. 计算证书的剩余有效期
3. 在证书即将更新时创建警告事件
4. 在证书更新后创建信息事件
5. 提供可视化界面展示证书状态

### 2.2 技术架构

![证书监控架构](https://placeholder-for-architecture-diagram.com)

主要组件：
- **证书监控控制器**: 核心组件，监控证书状态并生成事件
- **事件处理器**: 处理和转发证书相关事件
- **存储后端**: 存储证书状态历史数据
- **API服务**: 提供RESTful API查询证书状态
- **Web界面**: 可视化展示证书状态和历史记录

## 3. 实现步骤

### 3.1 创建证书监控控制器

首先，我们需要创建一个控制器来监控证书相关的Secret资源：

```go
package certmonitor

import (
    "context"
    "crypto/x509"
    "encoding/pem"
    "fmt"
    "time"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/util/runtime"
    "k8s.io/apimachinery/pkg/util/wait"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    corelisters "k8s.io/client-go/listers/core/v1"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/util/workqueue"
    "k8s.io/klog/v2"

    "github.com/openshift/library-go/pkg/operator/events"
)

const (
    // 证书即将过期的警告阈值（30天）
    warningThreshold = 30 * 24 * time.Hour
    
    // 控制器名称
    controllerName = "CertificateMonitorController"
)

// CertMonitorController 监控证书相关的Secret资源
type CertMonitorController struct {
    kubeClient    kubernetes.Interface
    secretLister  corelisters.SecretLister
    secretsSynced cache.InformerSynced
    workqueue     workqueue.RateLimitingInterface
    eventRecorder events.Recorder
    
    // 记录已处理的证书更新事件，避免重复事件
    processedCerts map[string]time.Time
}

// NewCertMonitorController 创建一个新的证书监控控制器
func NewCertMonitorController(
    kubeClient kubernetes.Interface,
    secretInformer informers.SharedInformerFactory,
    eventRecorder events.Recorder,
) *CertMonitorController {
    secretInformer := secretInformer.Core().V1().Secrets()
    
    controller := &CertMonitorController{
        kubeClient:     kubeClient,
        secretLister:   secretInformer.Lister(),
        secretsSynced:  secretInformer.Informer().HasSynced,
        workqueue:      workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Certificates"),
        eventRecorder:  eventRecorder,
        processedCerts: make(map[string]time.Time),
    }
    
    // 添加事件处理器
    secretInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueueSecret,
        UpdateFunc: func(old, new interface{}) {
            controller.enqueueSecret(new)
        },
    })
    
    return controller
}

// enqueueSecret 将Secret添加到工作队列
func (c *CertMonitorController) enqueueSecret(obj interface{}) {
    var key string
    var err error
    if key, err = cache.MetaNamespaceKeyFunc(obj); err != nil {
        runtime.HandleError(err)
        return
    }
    
    // 只关注特定命名空间和特定类型的Secret
    secret := obj.(*corev1.Secret)
    if isTargetCertSecret(secret) {
        c.workqueue.Add(key)
    }
}

// isTargetCertSecret 判断是否是目标证书Secret
func isTargetCertSecret(secret *corev1.Secret) bool {
    // 这里可以根据实际需求过滤Secret
    // 例如，只关注特定命名空间或特定标签的Secret
    
    // 检查是否包含证书数据
    if _, ok := secret.Data["tls.crt"]; ok {
        return true
    }
    if _, ok := secret.Data["ca.crt"]; ok {
        return true
    }
    
    return false
}

// Run 启动控制器
func (c *CertMonitorController) Run(workers int, stopCh <-chan struct{}) {
    defer runtime.HandleCrash()
    defer c.workqueue.ShutDown()
    
    klog.Infof("Starting %s", controllerName)
    defer klog.Infof("Shutting down %s", controllerName)
    
    // 等待缓存同步
    if !cache.WaitForCacheSync(stopCh, c.secretsSynced) {
        runtime.HandleError(fmt.Errorf("failed to wait for caches to sync"))
        return
    }
    
    // 启动工作线程
    for i := 0; i < workers; i++ {
        go wait.Until(c.runWorker, time.Second, stopCh)
    }
    
    <-stopCh
}

// runWorker 处理工作队列中的项目
func (c *CertMonitorController) runWorker() {
    for c.processNextWorkItem() {
    }
}

// processNextWorkItem 处理下一个工作项
func (c *CertMonitorController) processNextWorkItem() bool {
    obj, shutdown := c.workqueue.Get()
    if shutdown {
        return false
    }
    
    defer c.workqueue.Done(obj)
    
    var key string
    var ok bool
    if key, ok = obj.(string); !ok {
        c.workqueue.Forget(obj)
        runtime.HandleError(fmt.Errorf("expected string in workqueue but got %#v", obj))
        return true
    }
    
    if err := c.syncHandler(key); err != nil {
        c.workqueue.AddRateLimited(key)
        runtime.HandleError(fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error()))
        return true
    }
    
    c.workqueue.Forget(obj)
    return true
}

// syncHandler 处理Secret并检查证书状态
func (c *CertMonitorController) syncHandler(key string) error {
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }
    
    secret, err := c.secretLister.Secrets(namespace).Get(name)
    if err != nil {
        return err
    }
    
    // 处理证书数据
    return c.processCertificateData(secret)
}

// processCertificateData 处理证书数据并生成事件
func (c *CertMonitorController) processCertificateData(secret *corev1.Secret) error {
    // 查找证书数据
    var certData []byte
    for _, key := range []string{"tls.crt", "ca.crt"} {
        if data, ok := secret.Data[key]; ok {
            certData = data
            break
        }
    }
    
    if len(certData) == 0 {
        return nil
    }
    
    // 解析证书
    block, _ := pem.Decode(certData)
    if block == nil {
        return fmt.Errorf("failed to decode PEM block")
    }
    
    cert, err := x509.ParseCertificate(block.Bytes)
    if err != nil {
        return err
    }
    
    // 计算剩余有效期
    remainingTime := time.Until(cert.NotAfter)
    
    // 生成唯一标识符
    certID := fmt.Sprintf("%s/%s/%s", secret.Namespace, secret.Name, cert.SerialNumber.String())
    
    // 检查是否已处理过该证书
    lastProcessed, exists := c.processedCerts[certID]
    
    // 如果证书即将过期且未发送过警告
    if remainingTime < warningThreshold && (!exists || time.Since(lastProcessed) > 24*time.Hour) {
        c.eventRecorder.Warningf(
            "CertificateExpirationWarning",
            "Certificate in Secret %s/%s will expire in %.1f days",
            secret.Namespace, secret.Name, remainingTime.Hours()/24,
        )
        c.processedCerts[certID] = time.Now()
    }
    
    // 检测证书是否刚刚更新（通过比较上次处理时的过期时间）
    if exists {
        // 如果证书序列号变化，说明证书已更新
        if cert.SerialNumber.String() != certID[strings.LastIndex(certID, "/")+1:] {
            c.eventRecorder.Eventf(
                "CertificateRotated",
                "Certificate in Secret %s/%s has been rotated, new expiration: %s",
                secret.Namespace, secret.Name, cert.NotAfter.Format(time.RFC3339),
            )
            // 更新处理记录
            delete(c.processedCerts, certID)
            newCertID := fmt.Sprintf("%s/%s/%s", secret.Namespace, secret.Name, cert.SerialNumber.String())
            c.processedCerts[newCertID] = time.Now()
        }
    } else {
        // 首次处理该证书
        c.processedCerts[certID] = time.Now()
    }
    
    return nil
}
```

### 3.2 创建自定义资源定义(CRD)

为了更好地管理证书监控配置，我们可以创建一个自定义资源定义：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: certificatemonitors.monitoring.openshift.io
spec:
  group: monitoring.openshift.io
  names:
    kind: CertificateMonitor
    listKind: CertificateMonitorList
    plural: certificatemonitors
    singular: certificatemonitor
    shortNames:
    - certmon
  scope: Namespaced
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              targetNamespaces:
                type: array
                items:
                  type: string
              warningThresholdDays:
                type: integer
                minimum: 1
                maximum: 365
              secretSelectors:
                type: array
                items:
                  type: object
                  properties:
                    key:
                      type: string
                    operator:
                      type: string
                      enum: [In, NotIn, Exists, DoesNotExist]
                    values:
                      type: array
                      items:
                        type: string
          status:
            type: object
            properties:
              monitoredCertificates:
                type: array
                items:
                  type: object
                  properties:
                    namespace:
                      type: string
                    name:
                      type: string
                    notBefore:
                      type: string
                      format: date-time
                    notAfter:
                      type: string
                      format: date-time
                    serialNumber:
                      type: string
                    issuer:
                      type: string
                    subject:
                      type: string
                    remainingDays:
                      type: integer
```

### 3.3 部署监控控制器

创建一个Deployment来部署监控控制器：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: certificate-monitor
  namespace: openshift-monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: certificate-monitor
  template:
    metadata:
      labels:
        app: certificate-monitor
    spec:
      serviceAccountName: certificate-monitor
      containers:
      - name: controller
        image: quay.io/example/certificate-monitor:latest
        args:
        - "--v=2"
        - "--logtostderr=true"
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi
```

### 3.4 创建RBAC权限

为监控控制器创建必要的RBAC权限：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: certificate-monitor
  namespace: openshift-monitoring

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: certificate-monitor
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
- apiGroups: ["monitoring.openshift.io"]
  resources: ["certificatemonitors"]
  verbs: ["get", "list", "watch", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: certificate-monitor
subjects:
- kind: ServiceAccount
  name: certificate-monitor
  namespace: openshift-monitoring
roleRef:
  kind: ClusterRole
  name: certificate-monitor
  apiGroup: rbac.authorization.k8s.io
```

### 3.5 配置Prometheus监控

创建ServiceMonitor来监控控制器指标：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: certificate-monitor
  namespace: openshift-monitoring
spec:
  selector:
    matchLabels:
      app: certificate-monitor
  endpoints:
  - port: metrics
    interval: 30s
```

### 3.6 创建Grafana仪表板

为证书监控创建Grafana仪表板：

```json
{
  "annotations": {
    "list": []
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": 1,
  "links": [],
  "panels": [
    {
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "yellow",
                "value": 30
              },
              {
                "color": "red",
                "value": 7
              }
            ]
          },
          "unit": "d"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 0
      },
      "id": 2,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true
      },
      "pluginVersion": "7.5.5",
      "targets": [
        {
          "expr": "certificate_days_until_expiry",
          "interval": "",
          "legendFormat": "{{namespace}}/{{name}}",
          "refId": "A"
        }
      ],
      "title": "Certificate Days Until Expiry",
      "type": "gauge"
    }
  ],
  "refresh": "10s",
  "schemaVersion": 27,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "Certificate Monitoring",
  "uid": "certificates",
  "version": 1
}
```

## 4. 使用方法

### 4.1 部署监控方案

1. 应用CRD定义：
   ```bash
   oc apply -f certificate-monitor-crd.yaml
   ```

2. 创建RBAC资源：
   ```bash
   oc apply -f certificate-monitor-rbac.yaml
   ```

3. 部署监控控制器：
   ```bash
   oc apply -f certificate-monitor-deployment.yaml
   ```

4. 配置监控目标：
   ```bash
   oc apply -f - <<EOF
   apiVersion: monitoring.openshift.io/v1alpha1
   kind: CertificateMonitor
   metadata:
     name: cluster-certificates
     namespace: openshift-monitoring
   spec:
     targetNamespaces:
     - openshift-kube-apiserver
     - openshift-kube-apiserver-operator
     warningThresholdDays: 30
   EOF
   ```

### 4.2 查看证书状态

1. 通过CLI查看证书状态：
   ```bash
   oc get certificatemonitor cluster-certificates -n openshift-monitoring -o yaml
   ```

2. 通过Web控制台查看Grafana仪表板：
   - 导航到OpenShift控制台
   - 选择"Monitoring" -> "Dashboards"
   - 选择"Certificate Monitoring"仪表板

### 4.3 查看证书相关事件

查看系统中的证书相关事件：
```bash
oc get events --field-selector reason=CertificateExpirationWarning,CertificateRotated
```

## 5. 故障排除

### 5.1 常见问题

1. **控制器无法启动**
   - 检查RBAC权限是否正确配置
   - 检查ServiceAccount是否存在
   - 查看控制器日志：`oc logs deployment/certificate-monitor -n openshift-monitoring`

2. **未收到证书过期警告**
   - 检查warningThresholdDays配置是否合理
   - 确认目标证书Secret是否在监控范围内
   - 检查控制器是否正常运行

3. **Grafana仪表板无数据**
   - 检查ServiceMonitor配置
   - 确认Prometheus能够抓取指标
   - 检查控制器是否正确暴露指标

### 5.2 日志分析

控制器日志中的关键信息：
- `Starting CertificateMonitorController`: 控制器启动
- `Processing certificate in Secret namespace/name`: 处理证书
- `Certificate will expire in X days`: 证书即将过期
- `Certificate has been rotated`: 证书已更新

## 6. 总结

本文档详细介绍了如何在OpenShift中实现一个第三方证书监控方案。该方案可以：

1. 监控集群中的证书状态
2. 在证书即将过期时发出警告
3. 在证书更新后记录事件
4. 提供可视化界面展示证书状态

通过部署这个监控方案，集群管理员可以更好地了解证书的生命周期，及时发现潜在的证书过期问题，确保集群的安全和稳定运行。
