# OpenShift Certificate Rotation Process Summary

This document summarizes how certificate rotation works in OpenShift, with special focus on the timing between event emission and actual certificate updates.

## Certificate Rotation Overview

The certificate rotation controller in OpenShift is responsible for:

1. Creating and maintaining self-signed CA certificates
2. Maintaining CA bundle ConfigMaps with all non-expired CA certificates
3. Creating and rotating target certificates signed by the latest signing CA

The controller runs every minute to check if any certificates need rotation.

## Certificate Rotation Triggers

Certificates are rotated when any of these conditions are met:

- The certificate has already expired
- The certificate has reached 80% of its validity period (if `RefreshOnlyWhenExpired` is false)
- The certificate has passed its configured refresh time (if set)
- The signing CA has changed

## Timing Between Events and Certificate Updates

**Key Finding**: Certificate rotation events and actual certificate updates occur immediately within the same controller cycle:

1. The controller runs checks every minute
2. When a certificate needs rotation, an event (e.g., `SignerUpdateRequired`) is emitted
3. In the same function call, a new certificate is immediately created and applied
4. There is no delay between event emission and certificate update - they are atomic operations

## Certificate Rotation Process Flow

```mermaid
flowchart TD
    Start["Controller Sync Cycle<br/>(Runs Every Minute)"] --> CheckSigner["Check if Signing CA<br/>Needs Rotation"]
    
    CheckSigner -- "No" --> CheckCABundle["Check if CA Bundle<br/>Needs Update"]
    CheckSigner -- "Yes" --> EmitSignerEvent["Emit SignerUpdateRequired Event"]
    
    EmitSignerEvent --> CreateNewSigner["Create New Signing CA<br/>Immediately"]
    CreateNewSigner --> ApplySigner["Apply New Signing CA Secret"]
    ApplySigner --> CheckCABundle
    
    CheckCABundle -- "No" --> CheckTarget["Check if Target Certificate<br/>Needs Rotation"]
    CheckCABundle -- "Yes" --> EmitCABundleEvent["Emit CABundleUpdateRequired Event"]
    
    EmitCABundleEvent --> UpdateCABundle["Update CA Bundle ConfigMap<br/>Immediately"]
    UpdateCABundle --> CheckTarget
    
    CheckTarget -- "No" --> End["End Sync Cycle"]
    CheckTarget -- "Yes" --> EmitTargetEvent["Emit TargetUpdateRequired Event"]
    
    EmitTargetEvent --> CreateNewTarget["Create New Target Certificate<br/>Immediately"]
    CreateNewTarget --> ApplyTarget["Apply New Target Certificate Secret"]
    ApplyTarget --> End
    
    End --> Wait["Wait for Next Sync<br/>(1 minute)"]
    Wait --> Start
```

## Certificate Rotation Decision Logic

```mermaid
flowchart TD
    Start["Check if Certificate<br/>Needs Rotation"] --> HasAnnotations{"Has Required<br/>Annotations?"}
    
    HasAnnotations -- "No" --> NeedsRotation["Needs Rotation:<br/>missing notAfter/notBefore"]
    HasAnnotations -- "Yes" --> IsExpired{"Is Certificate<br/>Expired?"}
    
    IsExpired -- "Yes" --> NeedsRotation["Needs Rotation:<br/>already expired"]
    IsExpired -- "No" --> RefreshOnlyWhenExpired{"RefreshOnlyWhenExpired<br/>Enabled?"}
    
    RefreshOnlyWhenExpired -- "Yes" --> NoRotation["No Rotation Needed"]
    RefreshOnlyWhenExpired -- "No" --> At80Percent{"Past 80% of<br/>Validity Period?"}
    
    At80Percent -- "Yes" --> NeedsRotation["Needs Rotation:<br/>past its latest possible time"]
    At80Percent -- "No" --> PastRefresh{"Past Refresh<br/>Time?"}
    
    PastRefresh -- "Yes" --> NeedsRotation["Needs Rotation:<br/>past its refresh time"]
    PastRefresh -- "No" --> SignerChanged{"Signer CA<br/>Changed?"}
    
    SignerChanged -- "Yes" --> NeedsRotation["Needs Rotation:<br/>issuer not in ca bundle"]
    SignerChanged -- "No" --> NoRotation
    
    NeedsRotation --> EmitEvent["Emit Event and<br/>Rotate Certificate"]
    NoRotation --> End["End Check"]
```

## System Outputs

During certificate rotation, the system outputs:

### Events

- **SignerUpdateRequired**: Emitted when a new signing certificate/key pair is needed
- **TargetUpdateRequired**: Emitted when a new target certificate/key pair is needed
- **CABundleUpdateRequired**: Emitted when the CA bundle needs to be updated
- **RotationError**: Emitted when an error occurs during rotation

### Logs

系统会通过klog记录证书更新的详细信息，例如在更新CA证书包时会记录"Updated ca-bundle.crt configmap %s/%s with:\n%s"。

### Status Conditions

如果证书轮换过程中出现错误，系统会更新CertRotationDegraded状态条件，并设置原因为"RotationError"。
当状态更新成功且有错误时，系统会记录一个警告事件："RotationError"，包含错误信息。

## 证书轮换的触发条件包括：

证书已过期
证书已达到有效期的80%（如果RefreshOnlyWhenExpired为false）
证书已超过刷新时间（如果设置了Refresh参数）
签名CA发生变化

## Conclusion

The certificate rotation process in OpenShift is designed to be proactive and immediate. When the system detects that a certificate needs rotation, it emits an event and immediately performs the rotation in the same controller cycle. This ensures that certificates are rotated well before they expire, maintaining the security and availability of the cluster.
