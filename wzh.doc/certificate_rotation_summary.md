# OpenShift 证书轮换过程总结

本文档总结了 OpenShift 中证书轮换的工作原理，特别关注事件发出与实际证书更新之间的时间关系。

## 证书轮换概述

OpenShift 的证书轮换控制器负责：

1. 创建和维护自签名的 CA 证书
2. 维护包含所有未过期 CA 证书的 CA 证书包
3. 创建和轮换由最新签名 CA 签发的目标证书

控制器每分钟运行一次，检查是否有证书需要轮换。

## 证书轮换触发条件

当满足以下任一条件时，证书会被轮换：

- 证书已过期
- 证书已达到有效期的 80%（如果 `RefreshOnlyWhenExpired` 为 false）
- 证书已超过配置的刷新时间（如果设置了 `Refresh` 参数）
- 签名 CA 发生变化

## 事件与证书更新的时间关系

**关键发现**：证书轮换事件的发出与实际证书更新是在同一个控制器周期内立即发生的：

1. 控制器每分钟运行一次检查
2. 当检测到证书需要轮换时，立即发出相应事件（如 `SignerUpdateRequired`）
3. 在同一函数调用中，立即创建新证书并应用更新
4. 事件发出和证书更新之间没有延迟，它们是原子性操作

## 证书轮换流程

```mermaid
flowchart TD
    Start[控制器同步周期\n每分钟运行一次] --> CheckSigner[检查签名 CA\n是否需要轮换]
    
    CheckSigner -- 否 --> CheckCABundle[检查 CA 证书包\n是否需要更新]
    CheckSigner -- 是 --> EmitSignerEvent[发出 SignerUpdateRequired 事件]
    
    EmitSignerEvent --> CreateNewSigner[立即创建新的签名 CA]
    CreateNewSigner --> ApplySigner[应用新的签名 CA Secret]
    ApplySigner --> CheckCABundle
    
    CheckCABundle -- 否 --> CheckTarget[检查目标证书\n是否需要轮换]
    CheckCABundle -- 是 --> EmitCABundleEvent[发出 CABundleUpdateRequired 事件]
    
    EmitCABundleEvent --> UpdateCABundle[立即更新 CA 证书包 ConfigMap]
    UpdateCABundle --> CheckTarget
    
    CheckTarget -- 否 --> End[结束同步周期]
    CheckTarget -- 是 --> EmitTargetEvent[发出 TargetUpdateRequired 事件]
    
    EmitTargetEvent --> CreateNewTarget[立即创建新的目标证书]
    CreateNewTarget --> ApplyTarget[应用新的目标证书 Secret]
    ApplyTarget --> End
    
    End --> Wait[等待下一次同步\n(1分钟)]
    Wait --> Start
```

## 证书轮换决策逻辑

```mermaid
flowchart TD
    Start[检查证书是否需要轮换] --> HasAnnotations{是否有必要的注解?}
    
    HasAnnotations -- 否 --> NeedsRotation[需要轮换:\n"缺少 notAfter/notBefore"]
    HasAnnotations -- 是 --> IsExpired{证书是否已过期?}
    
    IsExpired -- 是 --> NeedsRotation[需要轮换:\n"已过期"]
    IsExpired -- 否 --> RefreshOnlyWhenExpired{RefreshOnlyWhenExpired\n是否启用?}
    
    RefreshOnlyWhenExpired -- 是 --> NoRotation[不需要轮换]
    RefreshOnlyWhenExpired -- 否 --> At80Percent{是否超过有效期的80%?}
    
    At80Percent -- 是 --> NeedsRotation[需要轮换:\n"超过最晚可能时间"]
    At80Percent -- 否 --> PastRefresh{是否超过刷新时间?}
    
    PastRefresh -- 是 --> NeedsRotation[需要轮换:\n"超过刷新时间"]
    PastRefresh -- 否 --> SignerChanged{签名 CA 是否变化?}
    
    SignerChanged -- 是 --> NeedsRotation[需要轮换:\n"签发者不在 CA 证书包中"]
    SignerChanged -- 否 --> NoRotation
    
    NeedsRotation --> EmitEvent[发出事件并轮换证书]
    NoRotation --> End[结束检查]
```

## 系统输出

在证书轮换过程中，系统会输出：

### 事件

- **SignerUpdateRequired**: 当需要新的签名证书/密钥对时发出
- **TargetUpdateRequired**: 当需要新的目标证书/密钥对时发出
- **CABundleUpdateRequired**: 当需要更新 CA 证书包时发出
- **RotationError**: 当轮换过程中出现错误时发出

### 日志

- 关于证书更新的详细日志，包括 CA 证书包更新时的日志

### 状态条件

- **CertRotation_[NAME]_Degraded**: 当轮换失败时设置为 `True`，原因为 `RotationError`

## 结论

OpenShift 中的证书轮换过程设计为主动且即时的。当系统检测到证书需要轮换时，它会发出事件并在同一个控制器周期内立即执行轮换操作。这确保了证书在过期前得到及时轮换，维护集群的安全性和可用性。
