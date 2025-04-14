# Kube-apiserver Certificate Rotation Analysis in OpenShift

This document analyzes the process triggered when certificates related to the Kubernetes API server (`kube-apiserver`) are rotated in an OpenShift cluster. It details the components involved, storage locations, the role of the Machine Config Operator (MCO), and the restart behavior of `kube-apiserver` and `kubelet`.

## Overview

Certificate rotation for `kube-apiserver` involves several key operators and controllers:

1.  **Cluster Kube API Server Operator (CKAO):** Manages the `kube-apiserver` static pods and their associated certificates (serving certs, client certs like kubelet-client, aggregator-client, etc.). It utilizes the `library-go/operator/certrotation` library for time-based rotation.
2.  **RevisionController (within CKAO/library-go):** Monitors the ConfigMaps and Secrets used by the `kube-apiserver` static pod. When these resources change (due to certificate rotation or configuration updates), it creates a new revision and updates the `KubeAPIServer` Custom Resource (CR) status.
3.  **Static Pod Controllers (within CKAO/library-go):** Detect the revision change in the `KubeAPIServer` CR status and roll out a new static pod manifest to control plane nodes, triggering a `kube-apiserver` restart by the Kubelet.
4.  **Machine Config Operator (MCO):** Monitors certain cluster-wide configurations, including CA bundles like the one Kubelet uses to verify the `kube-apiserver` (`kube-apiserver-to-kubelet-client-ca`). It generates `MachineConfig` objects reflecting the desired node state.
5.  **Machine Config Daemon (MCD):** Runs on each node, applies `MachineConfig` changes, including writing updated CA bundles to the node's filesystem. Applying significant changes often involves node draining and rebooting.

## Certificate Rotation Trigger and Process (CKAO)

CKAO manages the lifecycle of various certificates required by `kube-apiserver`.

*   **Trigger:** Rotation is primarily time-based, configured within CKAO's `certrotationcontroller`. Each certificate type (signer or target) has defined `Validity` and `Refresh` periods. The rotation process begins when the certificate's age exceeds its `Refresh` duration.

    ```go
    // pkg/operator/certrotationcontroller/certrotationcontroller.go
    // Example for KubeAPIServerToKubeletClientCert target certificate:
    certrotation.RotatedSelfSignedCertKeySecret{
        Namespace: operatorclient.TargetNamespace,
        Name:      "kubelet-client",
        // ... other fields ...
        Validity:               30 * rotationDay, // e.g., 30 days
        Refresh:                15 * rotationDay, // e.g., 15 days
        RefreshOnlyWhenExpired: refreshOnlyWhenExpired,
        CertCreator: &certrotation.ClientRotation{
            UserInfo: &user.DefaultInfo{Name: "system:kube-apiserver", Groups: []string{"kube-master"}},
        },
        // ... informer/client config ...
    },
    ```

*   **Process:** The `library-go/pkg/operator/certrotation` logic handles the actual rotation:
    1.  It checks if the current target certificate needs refreshing based on its issuance date and the configured `Refresh` period.
    2.  If refresh is needed, it ensures the corresponding *signer* certificate (defined in `RotatedSigningCASecret`) is valid and loaded.
    3.  It generates a new private key and certificate signing request (CSR) based on the `CertCreator` configuration (e.g., `ClientRotation`, `ServingRotation`).
    4.  It uses the signer certificate and key to sign the CSR, creating the new target certificate.
    5.  The new key and certificate are saved into the target Secret (e.g., `kubelet-client` in `openshift-kube-apiserver` namespace).
    6.  If the *signer* certificate itself is rotated, the corresponding CA bundle ConfigMap (e.g., `kube-apiserver-to-kubelet-client-ca` in `openshift-kube-apiserver-operator`) is updated to include the new CA certificate.

*   **Storage:**
    *   **Signer Certificates/Keys:** Stored in Secrets within the CKAO's namespace (`openshift-kube-apiserver-operator`), e.g., `kube-apiserver-to-kubelet-signer`.
    *   **Target Certificates/Keys:** Stored in Secrets within the operand's namespace (`openshift-kube-apiserver`), e.g., `kubelet-client`, `localhost-serving-cert-certkey`.
    *   **CA Bundles:** Stored in ConfigMaps, often in the CKAO's namespace (`openshift-kube-apiserver-operator`) or `openshift-config-managed`, e.g., `kube-apiserver-to-kubelet-client-ca`, `kube-apiserver-aggregator-client-ca`.

## Kube-apiserver Restart Process (CKAO)

When a certificate used directly by the `kube-apiserver` static pod is rotated (updated in its Secret), the `RevisionController` triggers a restart.

*   **Trigger:** A change detected in any of the ConfigMaps or Secrets monitored by the `RevisionController`. This includes secrets containing rotated certificates like `kubelet-client`, `localhost-serving-cert-certkey`, `aggregator-client`, etc.

*   **Process:**
    1.  The `RevisionController.sync` loop calls `isLatestRevisionCurrent`.
    2.  `isLatestRevisionCurrent` compares the data in the base Secrets/ConfigMaps (e.g., `secrets/kubelet-client`) against the data in the latest revisioned copies (e.g., `secrets/kubelet-client-3`). If the certificate rotation updated the base secret, a difference is detected.
    3.  `createRevisionIfNeeded` is called, which determines a new revision number (`nextRevision = latestAvailableRevision + 1`).
    4.  `createNewRevision` copies the *current* content from the base Secrets/ConfigMaps into new Secrets/ConfigMaps suffixed with `nextRevision` (e.g., `secrets/kubelet-client-4`).
    5.  Crucially, `createRevisionIfNeeded` updates the `status.latestAvailableRevision` field in the `KubeAPIServer` CR via the `operatorClient.UpdateLatestRevisionOperatorStatus` call.
    6.  Other controllers within CKAO (part of the static pod management framework, like `InstallerController`, `NodeController`) monitor the `KubeAPIServer` CR. They detect the change in `status.latestAvailableRevision`.
    7.  The `InstallerController` (likely) generates a new `kube-apiserver` static pod manifest (`pod.yaml`) that references the `nextRevision`. This manifest, along with other revisioned resources, is placed into a revision-specific ConfigMap (e.g., `kube-apiserver-pod-4`).
    8.  The `NodeController` ensures this ConfigMap is mounted into the correct directory (`/etc/kubernetes/static-pod-resources/kube-apiserver-pod-<revision>`) on each control plane node, and updates the static pod manifest file (`/etc/kubernetes/manifests/kube-apiserver-pod.yaml`) to point to the new revision's manifest.
    9.  The Kubelet on the control plane node watches the `/etc/kubernetes/manifests` directory. Upon detecting the change in `kube-apiserver-pod.yaml`, it gracefully stops the old `kube-apiserver` static pod and starts a new one based on the updated manifest, which uses the newly revisioned Secrets/ConfigMaps containing the rotated certificates.

*   **Code Snippet (Revision Trigger):**

    ```go
    // vendor/github.com/openshift/library-go/pkg/operator/revisioncontroller/revision_controller.go

    // isLatestRevisionCurrent compares base resources with the latest revisioned copies.
    func (c RevisionController) isLatestRevisionCurrent(ctx context.Context, revision int32) (bool, bool, string) {
        // ... comparison logic for configmaps and secrets ...
        if !equality.Semantic.DeepEqual(existingData, requiredData) {
            // Difference detected
            return false, false, "resource changed"
        }
        // ...
        return true, false, ""
    }

    // createRevisionIfNeeded creates a new revision if changes are detected.
    func (c RevisionController) createRevisionIfNeeded(ctx context.Context, recorder events.Recorder, latestAvailableRevision int32, resourceVersion string) (bool, error) {
        isLatestRevisionCurrent, requiredIsNotFound, reason := c.isLatestRevisionCurrent(ctx, latestAvailableRevision)
        if isLatestRevisionCurrent {
            return false, nil // No changes
        }

        nextRevision := latestAvailableRevision + 1
        // ... check required resources ...

        // Create new revisioned copies of resources
        createdNewRevision, err := c.createNewRevision(ctx, recorder, nextRevision, reason)
        // ... error handling ...

        if !createdNewRevision { return false, nil }

        // *** KEY STEP: Update operator status with the new revision number ***
        cond := operatorv1.OperatorCondition{ /* ... */ }
        if _, updated, updateError := c.operatorClient.UpdateLatestRevisionOperatorStatus(ctx, nextRevision, v1helpers.UpdateConditionFn(cond)); updateError != nil {
            return true, updateError
        } else if updated {
            recorder.Eventf("RevisionCreate", "Revision %d created because %s", nextRevision, reason)
        }
        return false, nil
    }
    ```

## CA Bundle Distribution (MCO/MCD)

When a CA bundle used by components outside the static pod (like Kubelet) is updated, MCO and MCD handle its distribution to nodes. The primary example is the CA bundle Kubelet uses to verify the `kube-apiserver`'s serving certificate.

*   **Trigger:** CKAO's `CertRotationController` updates a CA bundle ConfigMap (e.g., `kube-apiserver-to-kubelet-client-ca` in `openshift-kube-apiserver-operator`) when the corresponding *signer* certificate rotates.
*   **Components:** MCO Controller, MCD (Machine Config Daemon).
*   **Process:**
    1.  The MCO controller (`pkg/operator/sync.go`) watches relevant ConfigMaps, including `kube-apiserver-to-kubelet-client-ca`.
    2.  Upon detecting a change, it reads the updated CA bundle data (`ca-bundle.crt`).
    3.  This data (`kubeAPIServerServingCABytes`) is stored within the MCO's internal `ControllerConfig` CR spec (`spec.KubeAPIServerServingCAData`).
    4.  MCO renders new `MachineConfig` objects for the relevant pools (e.g., `master`, `worker`). These `MachineConfigs` define the desired state of files on the nodes, including the target path for the Kubelet CA bundle, populated with the data from `spec.KubeAPIServerServingCAData`.
    5.  The MCD running on each node detects that a new `MachineConfig` is available for it.
    6.  MCD's `certificate_writer.go` specifically handles writing the CA data from the applied `ControllerConfig`'s `Spec.KubeAPIServerServingCAData` to the designated path on the node's filesystem.

*   **Storage:** The CA bundle is written to the node filesystem by MCD. The typical path used by Kubelet for its `--client-ca-file` argument (or corresponding KubeletConfiguration field `clientCAFile`) is `/etc/kubernetes/kubelet-ca.crt`.

*   **Code Snippets:**

    ```go
    // pkg/operator/sync.go - MCO reads the CA bundle ConfigMap
    func (optr *Operator) sync(ctx context.Context, syncCtx factory.SyncContext) error {
        // ... other logic ...
        var kubeAPIServerServingCABytes []byte
        // ... logic to determine which CM to read based on auth mode ...
        kubeAPIServerServingCABytes, err = optr.getCAsFromConfigMap("openshift-kube-apiserver-operator", "kube-apiserver-to-kubelet-client-ca", "ca-bundle.crt")
        // ... error handling and merging logic ...

        // Store in ControllerConfig spec
        spec.KubeAPIServerServingCAData = kubeAPIServerServingCABytes
        // ... render MachineConfigs using this data ...
    }

    // pkg/daemon/certificate_writer.go - MCD writes the CA bundle to the node
    func (cw *CertificateWriter) writeCertificatesToDisk() error {
        // ... get controllerConfig ...
        kubeAPIServerServingCABytes := controllerConfig.Spec.KubeAPIServerServingCAData
        // ... other CAs ...

        pathToData := make(map[string][]byte)
        // Assuming caBundleFilePath resolves to /etc/kubernetes/kubelet-ca.crt or similar
        pathToData[caBundleFilePath] = kubeAPIServerServingCABytes
        // ... add other CAs to pathToData ...

        // Write files defined in pathToData
        if err := cw.writeFiles(pathToData); err != nil {
            return fmt.Errorf("error writing certificate files: %w", err)
        }
        // ... potentially restart services ...
        return nil
    }
    ```

## Kubelet Restart Process

Whether Kubelet restarts *directly* due to a CA bundle update is nuanced.

*   **Trigger:** MCD applying a new `MachineConfig` to the node. This `MachineConfig` might contain the updated `/etc/kubernetes/kubelet-ca.crt` file content, or other changes to Kubelet's configuration or related system files.
*   **Process:**
    1.  Kubelet is generally capable of reloading TLS assets like the `clientCAFile` without a full service restart. It periodically checks for file changes or can be triggered to reload.
    2.  However, the standard mechanism for applying `MachineConfig` changes via MCD, especially for core components or critical file paths, involves a coordinated node update process.
    3.  MCD typically **drains** the node (evicting pods) and then **reboots** the node to ensure all changes are applied cleanly and consistently. This reboot inherently restarts the Kubelet service along with the entire node OS.
    4.  While the MCD code (`pkg/daemon/certificate_writer.go`) contains logic mentioning `"Skipping kubelet restart"`, this likely applies to specific, non-disruptive scenarios or might be related to deferring restarts during complex updates. The default and safest mechanism for applying the `MachineConfig` change containing the new CA bundle is a node reboot orchestrated by MCD.

*   **Conclusion on Kubelet Restart:** A Kubelet *service* restart (`systemctl restart kubelet`) is **not directly triggered** by CKAO rotating the `kube-apiserver-to-kubelet-client-ca`. Instead, the update is delivered via a `MachineConfig`, and the application of this `MachineConfig` by MCD **typically results in a node reboot**, which indirectly restarts Kubelet. The Kubelet process itself likely reloads the updated CA file content without requiring a service restart if only the file content changes, but the MCO/MCD update mechanism usually involves a reboot for such changes.

## Sequence Diagram

```mermaid
sequenceDiagram
    participant CKAO CertRotationController
    participant CKAO RevisionController
    participant KubeAPIServer CR
    participant Kubelet (CP Node)
    participant Kube-apiserver Pod
    participant MCO Controller
    participant ControllerConfig CR
    participant MachineConfig CR
    participant MCD (Node)
    participant Kubelet (Worker Node)

    Note over CKAO CertRotationController: Time-based refresh interval reached for cert X (e.g., kubelet-client)
    CKAO CertRotationController->>CKAO CertRotationController: Generate new key/cert for X
    CKAO CertRotationController->>Secret (X): Update with new key/cert

    Note over CKAO RevisionController: Monitors Secret (X)
    CKAO RevisionController->>Secret (X): Detects change
    CKAO RevisionController->>CKAO RevisionController: Calculate nextRevision (N+1)
    CKAO RevisionController->>Secrets/ConfigMaps (Revision N+1): Create copies with updated content
    CKAO RevisionController->>KubeAPIServer CR: Update status.latestAvailableRevision = N+1

    Note over CKAO StaticPod Controllers: Monitor KubeAPIServer CR status
    CKAO StaticPod Controllers->>KubeAPIServer CR: Detect change in latestAvailableRevision
    CKAO StaticPod Controllers->>ConfigMap (Pod Manifest N+1): Generate new manifest referencing revision N+1
    CKAO StaticPod Controllers->>Kubelet (CP Node): Update /etc/kubernetes/manifests/kube-apiserver-pod.yaml

    Kubelet (CP Node)->>Kubelet (CP Node): Detect manifest change
    Kubelet (CP Node)->>Kube-apiserver Pod (Rev N): Stop Pod
    Kubelet (CP Node)->>Kube-apiserver Pod (Rev N+1): Start Pod with new certs

    alt Signer Cert Rotated (e.g., kube-apiserver-to-kubelet-signer)
        CKAO CertRotationController->>ConfigMap (CA Bundle): Update CA Bundle (e.g., kube-apiserver-to-kubelet-client-ca)

        Note over MCO Controller: Monitors CA Bundle ConfigMap
        MCO Controller->>ConfigMap (CA Bundle): Detects change
        MCO Controller->>MCO Controller: Read updated CA data
        MCO Controller->>ControllerConfig CR: Update spec.kubeAPIServerServingCAData
        MCO Controller->>MachineConfig CR: Generate/Update MachineConfig with new file content for /etc/kubernetes/kubelet-ca.crt

        Note over MCD (Node): Monitors assigned MachineConfig
        MCD (Node)->>MachineConfig CR: Detects new desired config
        MCD (Node)->>Node Filesystem: Write updated /etc/kubernetes/kubelet-ca.crt
        MCD (Node)->>MCD (Node): Initiate Node Drain & Reboot (Typical)
        Note right of Kubelet (Worker Node): Node reboots, Kubelet starts with new CA
    end

```

## Summary

*   **Certificate Rotation Trigger:** Time-based refresh intervals configured in CKAO.
*   **Components:** CKAO (`CertRotationController`, `RevisionController`, Static Pod Controllers), MCO, MCD, Kubelet.
*   **Certificate Storage:** Secrets and ConfigMaps in `openshift-kube-apiserver-operator` and `openshift-kube-apiserver` namespaces. CA bundles also written to `/etc/kubernetes/kubelet-ca.crt` on nodes by MCD.
*   **Kube-apiserver Restart:** Triggered by CKAO's `RevisionController` detecting changes in dependent Secrets/ConfigMaps (including rotated certificates). CKAO updates the `KubeAPIServer` CR status (`latestAvailableRevision`), leading to a static pod rollout managed by Kubelet on control plane nodes.
*   **Kubelet Restart:** Not directly triggered by CA bundle rotation. The MCO detects the updated CA bundle ConfigMap, generates a new `MachineConfig`, and the MCD applies this change to the node. Applying the `MachineConfig` typically involves a node drain and **reboot**, which restarts Kubelet indirectly.
