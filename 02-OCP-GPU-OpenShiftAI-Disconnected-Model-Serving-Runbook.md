# Document 2 — GPU Enablement, Disconnected Operator Mirroring & OpenShift AI Model Serving
## Cluster `a1procpait01.bank.sbi` — 2× NVIDIA B300 GPU Workers

---

## 0. Document Control

| Field | Value |
|---|---|
| Prerequisite | Document 1 complete — `oc get nodes` shows 3 masters + 8 workers `Ready` |
| GPU hardware | 2× bare-metal worker nodes with NVIDIA **B300** GPUs (`gpu-worker1`, `gpu-worker2`) |
| Sources used for GPU sections | NVIDIA GPU Operator official docs (`docs.nvidia.com/datacenter/cloud-native/gpu-operator` and `.../openshift`) — release notes, air-gapped install guide, and OpenShift-specific disconnected guide, checked live as of this writing |
| Mirroring method | `oc-mirror` **v2** for OLM/operator catalog content (IDMS/ITMS), **direct `podman pull`/`tag`/`push`** for NVIDIA's NGC (`nvcr.io`) operand images — these are two different registries/workflows, see Section 3 |
| Operators covered | Node Feature Discovery (NFD), NVIDIA GPU Operator, LVM Storage (persistent storage), cert-manager, Authorino, OpenShift Serverless, OpenShift Service Mesh, OpenShift Pipelines, Red Hat OpenShift AI (RHOAI) |
| GPU Operator version referenced | **v26.3.3** (current as of this writing) — B300 support was introduced in **v25.10.0**; **do not use anything older than v25.10.0** for these nodes. Always re-confirm the current version against NVIDIA's release notes before you install, since NVIDIA ships new minor versions frequently. |
| AI stack version | Red Hat OpenShift AI Self-Managed **3.3+** (Gen AI Studio / Playground referenced below is available from RHOAI 3.0 onward as a Technology Preview feature) |
| Model chosen for serving | **Meta Llama 3.1 8B Instruct** (per your request) |
| Serving runtime | vLLM ServingRuntime via KServe, single-model serving, packaged as a ModelCar OCI image |
| Training/RAG | Two paths covered — the **Gen AI Studio Playground GUI** (easiest, upload-and-chunk, Technology Preview) and the **Kubeflow Trainer** pipeline (production fine-tuning) |

> **Scope note on "disconnected":** the bastion + mirror registry host has an **open proxy**, so *that one host* can reach the public internet (`registry.redhat.io`, `nvcr.io`, `quay.io`, Hugging Face, etc.) for mirroring. The **cluster nodes** remain fully disconnected and only ever talk to `registry.a1procpvirt01.bank.sbi:8443`.

> **Convention:** every step has **Action / Command / Verification**, with expected output.

---

## 1. Pre-Flight — Confirm the Proxy and What's Already Mirrored

```bash
export HTTP_PROXY=http://<proxy-host>:<proxy-port>
export HTTPS_PROXY=http://<proxy-host>:<proxy-port>
export NO_PROXY=".a1procpvirt01.bank.sbi,.bank.sbi,10.57.33.0/25,localhost,127.0.0.1"
curl -sI https://registry.redhat.io/v2/ | head -1
curl -sI https://nvcr.io/v2/ | head -1
curl -sI https://huggingface.co | head -1
```
**Expected output:** `HTTP/1.1 200`/`401` from all three (401 is fine, proves the TLS path works). `nvcr.io` and `registry.redhat.io` both serve on the standard registry port **443** (HTTPS) — you do not need a special port for either; the only non-standard port in this whole design is your own internal mirror on `:8443`.

```bash
podman login registry.redhat.io
podman login nvcr.io          # NGC API key as the password — generate one at https://ngc.nvidia.com/setup/api-key
```
**Expected output:** `Login Succeeded!` for both.

```bash
curl -sk -u <reg-user>:<reg-pass> https://registry.a1procpvirt01.bank.sbi:8443/v2/_catalog | python3 -m json.tool
```
**Expected output:** existing repos from Document 1 (`openshift/release`, `openshift/release-images`).

---

## 2. Additional Operators Red Hat Recommends for This Build

Beyond NFD + GPU Operator + RHOAI, a production-grade, bank-appropriate deployment needs a few more pieces. Install these **before** RHOAI (Section 6) since RHOAI's KServe/model-serving components depend on some of them.

| Operator | Why you need it here | Required or recommended |
|---|---|---|
| **LVM Storage** (`lvms-operator`) | Bare metal has no default storage class. `image-registry` (flagged Degraded in Document 1) and every PVC used by RHOAI (workbenches, pipelines, model caches, vector DB) need one. LVM Storage is Red Hat's lightweight, in-cluster option for bare metal when you don't already run ODF/Ceph. | **Required** — pick this or ODF (below) |
| **OpenShift Data Foundation (ODF)** | Alternative to LVM Storage if you need replicated/HA storage across nodes (recommended for a bank production cluster, not just a single-node-backed volume) rather than LVM Storage's per-node local volumes. | **Recommended over LVMS if you have the extra disks/nodes for it** — pick one, not both |
| **cert-manager Operator for Red Hat OpenShift** | KServe (RHOAI's model-serving layer) and the Authorino-based token authentication for inference endpoints both expect cert-manager to issue/rotate internal TLS certs. | **Required** for KServe |
| **Red Hat Authorino Operator** | Provides token-based authentication/authorization in front of `InferenceService` endpoints — without it, anyone who can resolve the route can call your model with no auth. Given this is a bank, treat this as non-optional before any real user (even internal) gets a URL. | **Strongly recommended / effectively required for production** |
| **Red Hat OpenShift Pipelines** (Tekton) | RHOAI's data science pipelines (used for RAG ingestion pipelines and fine-tuning pipelines, Section 9) run **on** Tekton under the hood. | **Required** if you use RHOAI Pipelines (recommended for the "production RAG" path in Section 9.2) |
| **Red Hat OpenShift GitOps** (Argo CD) | Not required for serving/training itself, but Red Hat's standard recommendation for managing all of the above declaratively (ClusterPolicy, DataScienceCluster, InferenceServices) instead of one-off `oc apply`. | Optional, recommended for change-managed environments like a bank |
| **Node Health Check + Node Maintenance Operator** | Automatically cordons/reboots a node that fails health checks (e.g., a GPU node whose driver crashes repeatedly) instead of paging someone at 2am. | Optional but recommended given only 2 GPU nodes — losing one silently halves your capacity |

**Verification you'll want to be able to run after Section 3-4 install these:**
```bash
oc get csv -A | grep -E "lvms|odf-operator|cert-manager|authorino|openshift-pipelines|openshift-gitops"
```
*Expected output:* `Succeeded` for whichever subset you chose.

---

## 3. Install `oc-mirror` v2 and Build the Operator ImageSetConfiguration

### 3.1 Install the plugin

```bash
tar xvf oc-mirror.tar.gz -C /usr/local/bin/ oc-mirror
chmod +x /usr/local/bin/oc-mirror
oc mirror version --v2
```
**Expected output:** version string containing `4.22`.

### 3.2 `ImageSetConfiguration` — corrected catalog and package names

> **Correction from earlier draft:** the NVIDIA GPU Operator ships in Red Hat's **`certified-operator-index`** catalog, not `redhat-operator-index` — that was wrong in the previous version of this document. Confirmed against NVIDIA's own OpenShift disconnected-install guide. `registry.connect.redhat.com/nvidia/gpu-operator-catalog` (also used in the earlier draft) is not the correct source either — use the certified index below.

```bash
mkdir -p /home/ocpadmin/mirror
cd /home/ocpadmin/mirror
vi imageset-config.yaml
```

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  platform:
    channels:
      - name: stable-4.22
        type: ocp
    graph: true
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.22
      packages:
        - name: nfd                                  # Node Feature Discovery
        - name: lvms-operator                          # LVM Storage
        - name: openshift-cert-manager-operator        # cert-manager
        - name: authorino-operator                     # Authorino (model endpoint auth)
        - name: openshift-pipelines-operator-rh         # Tekton, needed for RHOAI Pipelines
        - name: serverless-operator                     # KServe dependency
        - name: servicemeshoperator                     # KServe dependency
        - name: rhods-operator                          # Red Hat OpenShift AI
    - catalog: registry.redhat.io/redhat/certified-operator-index:v4.22
      packages:
        - name: gpu-operator-certified                  # NVIDIA GPU Operator — correct catalog
```

> **Confirm exact package/channel names before mirroring** — catalog contents and channel names change release to release. Query the live catalog rather than trusting anything hardcoded, including in this document:
> ```bash
> oc mirror list operators --catalog=registry.redhat.io/redhat/certified-operator-index:v4.22 --package=gpu-operator-certified
> ```
> **Expected output:** a table of channels (e.g. `v26.3`) and the current channel head CSV (e.g. `gpu-operator-certified.v26.3.3`) — use whatever the table actually shows as current, not a value copied from this document.

### 3.3 Mirror-to-mirror (direct, since the bastion has proxy internet access)

```bash
oc mirror -c imageset-config.yaml \
  --workspace file:///home/ocpadmin/mirror/workspace \
  docker://registry.a1procpvirt01.bank.sbi:8443 \
  --v2
```
**Expected output (tail):**
```
info: Mirroring completed in ...
info: Writing IDMS, ITMS, CatalogSource, UpdateService manifests to /home/ocpadmin/mirror/workspace/working-dir/cluster-resources
```

**Verification:**
```bash
ls /home/ocpadmin/mirror/workspace/working-dir/cluster-resources/
skopeo list-tags docker://registry.a1procpvirt01.bank.sbi:8443/redhat/certified-operator-index --cert-dir /etc/containers/certs.d
```
**Expected output:** manifest files present; at least one tag (`v4.22`) for the certified index.

---

## 4. Apply the Mirrored Operator Content to the Cluster

```bash
export KUBECONFIG=/home/ocpadmin/ocpinstall/auth/kubeconfig
oc apply -f /home/ocpadmin/mirror/workspace/working-dir/cluster-resources/
```
**Expected output:**
```
imagedigestmirrorset.config.openshift.io/idms-oc-mirror created
imagetagmirrorset.config.openshift.io/itms-oc-mirror created
catalogsource.operators.coreos.com/cs-redhat-operator-index-v4-22 created
catalogsource.operators.coreos.com/cs-certified-operator-index-v4-22 created
```

**Verification:**
```bash
oc get catalogsource -n openshift-marketplace
oc get pods -n openshift-marketplace
```
**Expected output:** both catalog source pods `Running`.

```bash
oc get packagemanifest -n openshift-marketplace | grep -E "nfd|gpu-operator-certified|lvms-operator|cert-manager|authorino|pipelines-operator|serverless|servicemesh|rhods"
```
**Expected output:** all packages listed and visible to OLM — the definitive proof mirroring worked end-to-end.

---

## 5. Storage, cert-manager, Authorino, Pipelines (in order, each is a quick subscribe-and-verify)

Use the same Subscription/OperatorGroup pattern for each (namespace names below are Red Hat's documented defaults):

```bash
for op in "lvms-operator:openshift-storage" "openshift-cert-manager-operator:cert-manager-operator" \
          "authorino-operator:openshift-operators" "openshift-pipelines-operator-rh:openshift-operators"; do
  name="${op%%:*}"; ns="${op##*:}"
  oc create namespace "$ns" --dry-run=client -o yaml | oc apply -f -
  cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: ${name}-og
  namespace: ${ns}
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ${name}
  namespace: ${ns}
spec:
  channel: stable
  name: ${name}
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
done
```

**Verification:**
```bash
oc get csv -A | grep -E "lvms|cert-manager|authorino|pipelines-operator"
```
**Expected output:** all four `Succeeded`.

### 5.1 Configure LVM Storage as the default storage class

```bash
cat <<EOF | oc apply -f -
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
      - name: vg1
        thinPoolConfig:
          name: thin-pool-1
          sizePercent: 90
          overprovisionRatio: 10
EOF
```
**Verification:**
```bash
oc get lvmcluster -n openshift-storage
oc get sc
```
**Expected output:** `lvmcluster` `Ready`; `oc get sc` shows `lvms-vg1` marked `(default)`.

### 5.2 Fix the image-registry storage gap from Document 1

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p \
  '{"spec":{"managementState":"Managed","storage":{"pvc":{"claim":""}}}}'
```
**Verification:**
```bash
oc get co image-registry
```
**Expected output:** `AVAILABLE=True`, `DEGRADED=False` within a few minutes.

---

## 6. Install Node Feature Discovery and Label the GPU Nodes

*(unchanged from the previous draft — this part was already correct)*

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nfd
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-nfd
  namespace: openshift-nfd
spec:
  targetNamespaces:
  - openshift-nfd
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nfd
  namespace: openshift-nfd
spec:
  channel: stable
  name: nfd
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

cat <<EOF | oc apply -f -
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: openshift-nfd
spec: {}
EOF
```
**Verification:**
```bash
oc get csv -n openshift-nfd
oc get nodes -l feature.node.kubernetes.io/pci-10de.present=true
```
**Expected output:** `nfd` CSV `Succeeded`; both `gpu-worker1`/`gpu-worker2` listed (`10de` = NVIDIA's PCI vendor ID).

---

## 7. Mirror NVIDIA's NGC Operand Images

**This is the step the earlier draft got wrong.** The GPU Operator's *catalog/bundle* (from Section 3-4) comes from Red Hat's certified index — but the actual operand container images it deploys (driver, container-toolkit, device-plugin, dcgm-exporter, gpu-feature-discovery, mig-manager) are published by NVIDIA on **`nvcr.io`**, a completely separate registry from `registry.redhat.io`/`quay.io`. `oc-mirror`/IDMS does not touch `nvcr.io` — you mirror these with plain `podman pull` / `tag` / `push` through the bastion's proxy.

### 7.1 Confirm current versions before pulling anything

```bash
curl -sk https://nvcr.io/v2/nvidia/gpu-operator/tags/list | python3 -m json.tool | tail -20
```
Cross-check against the NVIDIA GPU Operator **Component Matrix** (`docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html#gpu-operator-component-matrix`) for the exact operand versions that ship together for your chosen GPU Operator release. As of this writing, the **v26.3.x** line pairs with:

| Component | Image | Version (confirm current before pulling) |
|---|---|---|
| GPU Operator | `nvcr.io/nvidia/gpu-operator` | `v26.3.3` |
| Driver | `nvcr.io/nvidia/driver` | `580.126.20` (default, RHCOS-tagged variant — see 7.2 note) |
| Container Toolkit | `nvcr.io/nvidia/k8s/container-toolkit` | `v1.19.1` |
| Kubernetes Device Plugin | `nvcr.io/nvidia/k8s-device-plugin` | `v0.19.3` |
| GPU Feature Discovery | `nvcr.io/nvidia/gpu-feature-discovery` | `v0.19.3` |
| DCGM Exporter | `nvcr.io/nvidia/k8s/dcgm-exporter` | `4.5.3-4.8.2` |
| MIG Manager | `nvcr.io/nvidia/cloud-native/k8s-mig-manager` | `v0.14.2` |

> **RHCOS driver tag note:** on Ubuntu you'd pull `driver:580.126.20-ubuntu22.04`. RHCOS is different — NVIDIA publishes precompiled driver containers for specific RHCOS/kernel combinations, or the Operator can build the driver in-cluster using the OpenShift **Driver Toolkit (DTK)** image (matched automatically to your RHCOS version via the `openshift/driver-toolkit` imagestream already present on every OCP cluster — no separate mirroring needed for DTK itself). **Use precompiled driver containers where available for your exact RHCOS 4.22.0 build** to avoid needing a local RPM/kernel-headers mirror at all (see Section 7.4) — check the **Tags** tab at `catalog.ngc.nvidia.com/orgs/nvidia/containers/driver` for a tag matching your RHCOS kernel version before falling back to the DTK build-from-source path.

### 7.2 Pull, tag, push each operand image (proxy-connected bastion)

```bash
export HTTP_PROXY=http://<proxy-host>:<proxy-port>
export HTTPS_PROXY=http://<proxy-host>:<proxy-port>

for img in \
  "nvidia/gpu-operator:v26.3.3" \
  "nvidia/driver:580.126.20-rhcos4.22" \
  "nvidia/k8s/container-toolkit:v1.19.1-ubi9" \
  "nvidia/k8s-device-plugin:v0.19.3-ubi9" \
  "nvidia/gpu-feature-discovery:v0.19.3-ubi9" \
  "nvidia/k8s/dcgm-exporter:4.5.3-4.8.2-ubi9" \
  "nvidia/cloud-native/k8s-mig-manager:v0.14.2-ubi9" ; do
    podman pull nvcr.io/${img}
    podman tag nvcr.io/${img} registry.a1procpvirt01.bank.sbi:8443/${img}
    podman push registry.a1procpvirt01.bank.sbi:8443/${img} --cert-dir /etc/containers/certs.d
done
```
> The exact tag suffixes (`-ubi9`, `-rhcos4.22`, etc.) vary by component and release — **confirm each one on the NGC catalog page for that image** (`catalog.ngc.nvidia.com/orgs/nvidia/containers/<image>` → Tags tab) rather than assuming the pattern above is universal; it differs per component.

**Verification:**
```bash
for img in gpu-operator driver k8s/container-toolkit k8s-device-plugin gpu-feature-discovery k8s/dcgm-exporter cloud-native/k8s-mig-manager; do
  echo -n "$img -> "; skopeo list-tags docker://registry.a1procpvirt01.bank.sbi:8443/nvidia/$img --cert-dir /etc/containers/certs.d | python3 -c "import json,sys; print(json.load(sys.stdin)['Tags'])"
done
```
**Expected output:** each line shows at least the one tag you just pushed — confirms every operand image actually landed on your internal mirror before the `ClusterPolicy` tries to pull it from there.

### 7.3 GPU Operator OperatorHub subscription

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nvidia-gpu-operator-group
  namespace: nvidia-gpu-operator
spec:
  targetNamespaces:
  - nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: gpu-operator-certified
  namespace: nvidia-gpu-operator
spec:
  channel: v26.3          # confirm this is still the current channel — Section 3.2 verification step shows you the live answer
  name: gpu-operator-certified
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```
**Verification:**
```bash
oc get csv -n nvidia-gpu-operator
```
**Expected output:** `gpu-operator-certified.v26.3.x` — `Phase: Succeeded`.

### 7.4 `ClusterPolicy` — pointed at your mirror, CDI enabled by default

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  operator:
    defaultRuntime: crio
  cdi:
    enabled: true              # default since GPU Operator v25.10.0 — leave enabled, don't disable
  driver:
    enabled: true
    repository: registry.a1procpvirt01.bank.sbi:8443/nvidia
    image: driver
    version: "580.126.20-rhcos4.22"   # confirm exact tag per Section 7.2 note
  toolkit:
    enabled: true
    repository: registry.a1procpvirt01.bank.sbi:8443/nvidia/k8s
    image: container-toolkit
    version: "v1.19.1-ubi9"
  devicePlugin:
    enabled: true
    repository: registry.a1procpvirt01.bank.sbi:8443/nvidia
    image: k8s-device-plugin
    version: "v0.19.3-ubi9"
  dcgmExporter:
    enabled: true
    repository: registry.a1procpvirt01.bank.sbi:8443/nvidia/k8s
    image: dcgm-exporter
    version: "4.5.3-4.8.2-ubi9"
  gfd:
    enabled: true
    repository: registry.a1procpvirt01.bank.sbi:8443/nvidia
    image: gpu-feature-discovery
    version: "v0.19.3-ubi9"
  migManager:
    enabled: true                     # B300 supports MIG — see profile table below
    repository: registry.a1procpvirt01.bank.sbi:8443/nvidia/cloud-native
    image: k8s-mig-manager
    version: "v0.14.2-ubi9"
  nodeStatusExporter:
    enabled: true
  validator:
    plugin:
      env:
      - name: WITH_WORKLOAD
        value: "true"
```

```bash
oc apply -f clusterpolicy.yaml
```

**B300 MIG profiles available if you enable partitioning later** (from NVIDIA's release notes, v25.10.0): `1g.34gb`, `1g.34gb+me`, `1g.67gb`, `2g.67gb`, `3g.135gb`, `4g.135gb`, `7g.269gb`, plus an `all-balanced` preset (`1g.34gb`×2 + `2g.67gb`×1 + `3g.135gb`×1). You don't need to configure any of this to serve one model per GPU (Section 10) — only relevant if you later want to split a single B300 across multiple smaller workloads.

**Verification:**
```bash
oc get pods -n nvidia-gpu-operator -o wide
```
**Expected output:** driver, toolkit, device-plugin, dcgm-exporter, gfd pods `Running`, scheduled only on `gpu-worker1`/`gpu-worker2`.

```bash
oc describe node gpu-worker1.a1procpait01.bank.sbi | grep -A5 "Allocatable:"
```
**Expected output:** `nvidia.com/gpu: <N>` listed.

```bash
oc exec -n nvidia-gpu-operator -it $(oc get pod -n nvidia-gpu-operator -l app=nvidia-driver-daemonset -o jsonpath='{.items[0].metadata.name}') -- nvidia-smi
```
**Expected output:** the standard `nvidia-smi` table listing the B300 GPU(s) and driver version — definitive proof the driver loaded.

### 7.5 Best practice — dedicate the GPU nodes to AI workloads

```bash
oc adm taint node gpu-worker1.a1procpait01.bank.sbi nvidia.com/gpu=true:NoSchedule
oc adm taint node gpu-worker2.a1procpait01.bank.sbi nvidia.com/gpu=true:NoSchedule
oc label node gpu-worker1.a1procpait01.bank.sbi node-role.kubernetes.io/gpu=
oc label node gpu-worker2.a1procpait01.bank.sbi node-role.kubernetes.io/gpu=
```
The GPU Operator's own operand pods automatically tolerate this taint. Any workload pod you deploy later must explicitly add the matching toleration + `nvidia.com/gpu` resource request.

**Verification:**
```bash
oc get nodes -l node-role.kubernetes.io/gpu -o wide
oc describe node gpu-worker1.a1procpait01.bank.sbi | grep -A2 Taints
```

---

## 8. Install Red Hat OpenShift AI (RHOAI)

### 8.1 Dependencies — Serverless and Service Mesh

```bash
oc create namespace openshift-serverless --dry-run=client -o yaml | oc apply -f -
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: serverless-operator
  namespace: openshift-serverless
spec:
  channel: stable
  name: serverless-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: servicemeshoperator
  namespace: openshift-operators
spec:
  channel: stable
  name: servicemeshoperator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```
**Verification:**
```bash
oc get csv -A | grep -E "serverless|servicemesh"
```
**Expected output:** both `Succeeded`.

### 8.2 Install the RHOAI Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: redhat-ods-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  channel: stable
  name: rhods-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```
**Verification:**
```bash
oc get csv -n redhat-ods-operator
```
**Expected output:** `rhods-operator.3.x.x` — `Phase: Succeeded`.

### 8.3 Create the `DataScienceCluster`

```bash
cat <<EOF | oc apply -f -
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    dashboard:
      managementState: Managed
    workbenches:
      managementState: Managed
    kserve:
      managementState: Managed
      serving:
        managementState: Managed
        name: knative-serving
    modelmeshserving:
      managementState: Removed
    trainingoperator:
      managementState: Managed
    ray:
      managementState: Managed
    datasciencepipelines:
      managementState: Managed        # needed for Section 9.2 production RAG/fine-tuning pipelines
    dashboard-model-registry:
      managementState: Managed
EOF
```
**Verification:**
```bash
oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}{"\n"}'
oc get pods -n redhat-ods-applications
```
**Expected output:** `Ready`; dashboard/KServe/pipeline controller pods `Running`.

---

## 9. GUI Access — the RHOAI Dashboard

### 9.1 URL and port

```bash
oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}{"\n"}'
```
**Expected output:** something like `rhods-dashboard-redhat-ods-applications.apps.a1procpait01.bank.sbi` — served over standard HTTPS **port 443** through the AVI ingress VS from Document 1 (no special port to open; it rides the same `*.apps` wildcard and AVI apps Virtual Service already configured).

Log in with your IdP-backed OpenShift identity (same SSO as `oc login`/console) — no separate RHOAI-specific account.

**Verification:**
```bash
curl -skI https://$(oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}') | head -1
```
**Expected output:** `HTTP/1.1 200` or a redirect to your IdP login (`302`).

### 9.2 What bank end users actually see

- **Data scientists / power users:** get a **Data Science Project** in the dashboard, a **Workbench** (Jupyter) for exploration, and access to **Pipelines** for repeatable RAG/training runs.
- **Everyday bank end users who just need to *use* the model:** don't need the dashboard at all for day-to-day use — point them at either (a) the **Gen AI Studio → Playground** chat UI (Section 11) if you want them testing/using it interactively inside RHOAI, or (b) a simple internal web app your team builds against the model's OpenAI-compatible REST endpoint (Section 10.4) if you want a branded, locked-down interface instead of exposing the full RHOAI dashboard to non-technical staff.
- Control who sees what with standard OpenShift RBAC: grant bank end users the `edit`/`view` role scoped to just the `llm-serving` (or a dedicated end-user-facing) project/namespace, not cluster-admin or access to every Data Science Project.

```bash
oc adm policy add-role-to-group view bank-llm-users -n llm-serving
```
**Verification:**
```bash
oc get rolebinding -n llm-serving | grep bank-llm-users
```

---

## 10. Download and Deploy the Model — Llama 3.1 8B Instruct

### 10.1 Download URL and port (on the bastion, through the proxy)

Hugging Face is served over standard HTTPS **port 443** at `huggingface.co` (and its CDN, `cdn-lfs.huggingface.co`, also 443) — no special port required, same as any other HTTPS site:

```bash
export HTTP_PROXY=http://<proxy-host>:<proxy-port>
export HTTPS_PROXY=http://<proxy-host>:<proxy-port>
pip install --user huggingface_hub
huggingface-cli login   # paste your HF access token — only ever used on the bastion, never on the cluster
huggingface-cli download meta-llama/Llama-3.1-8B-Instruct --local-dir /home/ocpadmin/models/llama-3.1-8b-instruct
```
**Verification:**
```bash
du -sh /home/ocpadmin/models/llama-3.1-8b-instruct
ls /home/ocpadmin/models/llama-3.1-8b-instruct | head
```
**Expected output:** ~16 GB total; `config.json`, `*.safetensors`, `tokenizer.json` present.

### 10.2 Package as a ModelCar OCI image and push to the mirror (port 8443, same as every other image in this doc)

```bash
cat > /home/ocpadmin/models/Modelcar.Dockerfile <<'EOF'
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS base
COPY llama-3.1-8b-instruct /models/llama-3.1-8b-instruct
EOF

podman build -t registry.a1procpvirt01.bank.sbi:8443/models/llama-3.1-8b-instruct:latest \
  -f /home/ocpadmin/models/Modelcar.Dockerfile /home/ocpadmin/models/
podman push registry.a1procpvirt01.bank.sbi:8443/models/llama-3.1-8b-instruct:latest --cert-dir /etc/containers/certs.d
```
**Verification:**
```bash
skopeo inspect docker://registry.a1procpvirt01.bank.sbi:8443/models/llama-3.1-8b-instruct:latest --cert-dir /etc/containers/certs.d | python3 -m json.tool | grep -A2 '"Digest"'
```
**Expected output:** a valid manifest digest.

### 10.3 ServingRuntime + InferenceService

```bash
oc new-project llm-serving
```

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: vllm-runtime
  namespace: llm-serving
spec:
  supportedModelFormats:
    - name: vLLM
      autoSelect: true
  containers:
    - name: kserve-container
      image: registry.a1procpvirt01.bank.sbi:8443/vllm:rhoai-cuda   # mirror the vLLM runtime image from quay.io/modh via oc-mirror additionalImages, confirm current tag against RHOAI 3.x release notes
      args:
        - --model=/models/llama-3.1-8b-instruct
        - --served-model-name=llama-3-1-8b-instruct
        - --dtype=bfloat16
        - --max-model-len=8192
      resources:
        limits:
          nvidia.com/gpu: "1"
        requests:
          nvidia.com/gpu: "1"
      tolerations:
        - key: nvidia.com/gpu
          operator: Equal
          value: "true"
          effect: NoSchedule
      nodeSelector:
        node-role.kubernetes.io/gpu: ""
```

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama-3-1-8b-instruct
  namespace: llm-serving
spec:
  predictor:
    model:
      modelFormat:
        name: vLLM
      runtime: vllm-runtime
      storageUri: oci://registry.a1procpvirt01.bank.sbi:8443/models/llama-3.1-8b-instruct:latest
      resources:
        limits:
          nvidia.com/gpu: "1"
        requests:
          nvidia.com/gpu: "1"
      tolerations:
        - key: nvidia.com/gpu
          operator: Equal
          value: "true"
          effect: NoSchedule
      nodeSelector:
        node-role.kubernetes.io/gpu: ""
```

```bash
oc apply -f servingruntime.yaml
oc apply -f inferenceservice.yaml
```

**Verification:**
```bash
oc get inferenceservice -n llm-serving
oc get pods -n llm-serving -o wide
```
**Expected output:** `READY=True`; predictor pod `Running` on a GPU node.

```bash
oc exec -n nvidia-gpu-operator -it $(oc get pod -n nvidia-gpu-operator -l app=nvidia-driver-daemonset -o jsonpath='{.items[0].metadata.name}') -- nvidia-smi
```
**Expected output:** GPU memory now shows several GB allocated (loaded weights + KV cache).

### 10.4 The model's own endpoint — port and URL

```bash
oc get inferenceservice llama-3-1-8b-instruct -n llm-serving -o jsonpath='{.status.url}{"\n"}'
```
**Expected output:** `https://llama-3-1-8b-instruct-llm-serving.apps.a1procpait01.bank.sbi` — again, standard **HTTPS 443** through the same AVI apps VS, OpenAI-compatible `/v1/chat/completions` path.

```bash
ROUTE=$(oc get inferenceservice llama-3-1-8b-instruct -n llm-serving -o jsonpath='{.status.url}')
curl -sk "${ROUTE}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama-3-1-8b-instruct","messages":[{"role":"user","content":"In one sentence, what is OpenShift?"}],"max_tokens":100}' \
  | python3 -m json.tool
```
**Expected output:** a JSON completion with `choices[0].message.content`.

---

## 11. How Bank End Users Actually Use the Model — the GUI

### 11.1 Gen AI Studio → Playground (built into the RHOAI dashboard, no extra install)

This is Red Hat's own answer to "how do we give end users a GUI, and how do they upload docs to train/ground it" — it ships as part of RHOAI 3.x itself, no separate app to build:

1. Open the RHOAI dashboard (Section 9.1), select your Data Science Project.
2. **Gen AI Studio → Playground.**
3. Select your deployed model (`llama-3-1-8b-instruct`) from the model dropdown — any `InferenceService` in the project shows up automatically, no extra registration step.
4. Users chat with the model directly in the browser — this is the "how do we use it" answer for anyone who doesn't need API access.

**Verification:** have a colleague log in, select the model, and send a test message — confirm they get a response without needing `oc`, a token, or any CLI access at all.

### 11.2 Uploading documents and chunking — the easy way

This directly answers "how we got GUI for AI" and "upload document so it's divided into chunks":

1. In the Playground, toggle **RAG** on and expand the section (in RHOAI 3.4+ this is under a **Knowledge** tab instead — check which your installed version shows).
2. Click **Upload** → drag-and-drop or browse for a file.
3. **Supported file formats: PDF, DOC, and CSV.** Up to **10 files per session, 10 MB max per file.**
4. Optionally adjust **Maximum chunk length**, **Chunk overlap**, and **Delimiter** to match your document type (e.g., shorter chunks for dense policy tables, larger chunks for narrative text).
5. Click **Upload**, wait for the **"Source uploaded"** notification, and the file appears under **Uploaded files**.
6. Ask a question in the chat that the model wouldn't know without your document — if the answer correctly reflects your document's content, ingestion + chunking + retrieval all worked.

> **Important limitation:** this Playground RAG feature is a **Technology Preview** in RHOAI 3.0 through at least 3.4 — not covered by Red Hat's production support SLA, and it only supports an **inline, session-scoped vector database** (no external/persistent vector DB, and refreshing the browser or ending the session **loses all uploaded documents and chat history**). Treat it as the fast way for a business user or SME to test "does the model understand our docs" — not as the permanent, governed knowledge base for production bank use. For that, use Section 12.2.

**Verification:**
```bash
oc get pods -n <your-data-science-project> | grep -i playground
```
**Expected output:** a playground/RAG backend pod `Running` while a session is active — confirms the inline vector DB is actually running in-cluster (not phoning out anywhere), which matters for your disconnected requirement.

---

## 12. Grounding the Model on Your Documents — Two Ways to "Train" It

| Path | What it does | When to use it |
|---|---|---|
| **12.1 Playground RAG (GUI, Tech Preview)** | Covered above — fastest, zero setup, session-scoped | Quick SME validation, demos, one-off Q&A testing |
| **12.2 Production RAG pipeline (Docling + Kubeflow Pipelines + persistent vector DB)** | Documents are parsed (including complex PDFs/tables) by **Docling**, chunked, embedded, and stored persistently in a vector DB (Milvus/PGVector); retrieval happens at query time via Llama Stack | Real, governed, always-on knowledge base for bank staff/customers |
| **12.3 Fine-tuning (Kubeflow Trainer SFT)** | Actually updates the model's weights on your data | When you need consistent tone/format/behavior, not just facts lookup |

### 12.1 covered in Section 11.2.

### 12.2 Production RAG — file types and the pipeline

**Supported/recommended file types for the production Docling path** (broader than the Playground's PDF/DOC/CSV limit): PDF, DOCX, PPTX, XLSX, HTML, Markdown, and plain text — Docling specifically excels at PDFs with complex tables/layouts, which is exactly what a bank's policy/compliance documents tend to be.

```bash
oc new-project rag-pipeline
```
Deploy a persistent vector DB (Milvus, via the RHOAI-provided component, or your bank's already-approved vector store) and a Docling-serve instance, then run ingestion as a Data Science Pipeline (Tekton-backed, from Section 2's OpenShift Pipelines operator) so it's repeatable and auditable — re-run automatically whenever your source documents change, rather than a one-off manual upload:

```python
from llama_stack_client import LlamaStackClient

client = LlamaStackClient(base_url="http://llama-stack.rag-pipeline.svc:8321")
client.vector_dbs.register(vector_db_id="internal-docs", embedding_model="ibm-granite/granite-embedding-125m-english")

for doc_path in your_document_paths:
    client.tool_runtime.rag_tool.insert(
        documents=[{"document_id": doc_path, "content": open(doc_path).read()}],
        vector_db_id="internal-docs",
    )
```
**Verification:**
```python
resp = client.tool_runtime.rag_tool.query(vector_db_id="internal-docs", query="<a question you know the answer to>")
print(resp)
```
**Expected output:** the retrieved chunk(s) genuinely come from your source documents.

### 12.3 Fine-tuning — Kubeflow Trainer SFT pipeline

```bash
oc new-project model-training
```
Prepare instruction-formatted **JSONL** (`{"prompt": ..., "completion": ...}` pairs) — this is the required training file format; raw docs must be converted to this Q&A/instruction format first (this conversion step is your team's responsibility, the trainer consumes whatever you give it).

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: llama-sft-finetune
  namespace: model-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          tolerations:
            - key: nvidia.com/gpu
              operator: Equal
              value: "true"
              effect: NoSchedule
          nodeSelector:
            node-role.kubernetes.io/gpu: ""
          containers:
            - name: pytorch
              image: registry.a1procpvirt01.bank.sbi:8443/training/sft-trainer:latest
              args:
                - --model_name_or_path=/models/llama-3.1-8b-instruct
                - --dataset_path=/data/internal-docs-sft.jsonl
                - --output_dir=/output/llama-3.1-8b-sbi-tuned
                - --num_train_epochs=3
                - --per_device_train_batch_size=2
                - --learning_rate=2e-5
              resources:
                limits:
                  nvidia.com/gpu: "1"
EOF
```
```bash
oc apply -f llama-sft-job.yaml
```
**Verification:**
```bash
oc get pytorchjob llama-sft-finetune -n model-training
oc logs -n model-training -f $(oc get pods -n model-training -l job-name=llama-sft-finetune -o jsonpath='{.items[0].metadata.name}')
```
**Expected output:** training loss decreasing; job `Status: Succeeded`; checkpoint at `/output/llama-3.1-8b-sbi-tuned`. Package it as a new ModelCar (Section 10.2) and deploy as a second `InferenceService` for A/B comparison before cutting traffic over.

---

## 13. Manage and Monitor

```bash
oc adm top pod -n llm-serving
oc exec -n nvidia-gpu-operator -it $(oc get pod -n nvidia-gpu-operator -l app=nvidia-driver-daemonset -o jsonpath='{.items[0].metadata.name}') -- nvidia-smi dmon -c 5
```
**Expected output:** live SM/memory utilization while a request is in flight.

```bash
oc annotate inferenceservice llama-3-1-8b-instruct -n llm-serving serving.kserve.io/min-scale=1 --overwrite
```
**Verification:**
```bash
oc get inferenceservice llama-3-1-8b-instruct -n llm-serving -o jsonpath='{.metadata.annotations.serving\.kserve\.io/min-scale}{"\n"}'
```
**Expected output:** `1` (keeps one replica warm, avoids cold-start reload of a 16 GB model).

The RHOAI dashboard's **Model Serving** view also shows the deployed model, its endpoint, and basic latency/throughput metrics — the place to point application teams who shouldn't need `oc` access.

---

## 14. Troubleshooting Matrix

| Symptom | Likely cause | Fix |
|---|---|---|
| `oc mirror` fails partway | Proxy env vars not exported for that shell, or auth expired | Re-export, re-run — `oc-mirror` v2 resumes rather than restarting |
| GPU Operator subscription channel not found | Channel name changed since this document was written | `oc mirror list operators --catalog=...certified-operator-index:v4.22 --package=gpu-operator-certified` and use the live answer |
| Driver pod `CrashLoopBackOff` | Wrong driver tag for the RHCOS/kernel combo, or B300 not supported by the GPU Operator version installed (<v25.10.0) | Confirm GPU Operator ≥ v25.10.0; use a precompiled driver container matching your exact RHCOS build (Section 7.1 note) |
| `nvidia.com/gpu` missing from node Allocatable | NFD hasn't labeled the node, or driver/toolkit pods failed | `oc get pods -n nvidia-gpu-operator -o wide`; check pod logs |
| ClusterPolicy stuck `notReady` | Operand image not actually on the mirror (Section 7.2 skipped or tag wrong) | Re-run the `skopeo list-tags` verification in 7.2 |
| InferenceService stuck pending | Large ModelCar image still pulling, or GPU toleration/taint mismatch | `oc describe pod` on the predictor; confirm toleration matches the taint exactly |
| Playground RAG upload fails / no "Source uploaded" | File exceeds 10MB or isn't PDF/DOC/CSV, or session backend pod not running | Confirm file type/size; check `oc get pods` in the project for the playground backend |
| Playground documents/chat gone after refresh | Expected — Playground RAG is session-scoped/Technology Preview, not persistent | Use Section 12.2 (persistent vector DB) for anything that must survive a session |
| Fine-tuning job OOMs | Batch size too large for available GPU memory | Reduce `per_device_train_batch_size`, or use LoRA instead of full-parameter SFT |

---

## 15. Appendix — Quick Command Reference

```bash
# ---- Operator catalog mirroring ----
oc mirror -c imageset-config.yaml --workspace file:///home/ocpadmin/mirror/workspace \
  docker://registry.a1procpvirt01.bank.sbi:8443 --v2
oc apply -f /home/ocpadmin/mirror/workspace/working-dir/cluster-resources/

# ---- NGC operand image mirroring (nvcr.io, port 443, through proxy) ----
podman pull nvcr.io/nvidia/driver:580.126.20-rhcos4.22
podman tag nvcr.io/nvidia/driver:580.126.20-rhcos4.22 registry.a1procpvirt01.bank.sbi:8443/nvidia/driver:580.126.20-rhcos4.22
podman push registry.a1procpvirt01.bank.sbi:8443/nvidia/driver:580.126.20-rhcos4.22 --cert-dir /etc/containers/certs.d

# ---- GPU stack ----
oc get csv -n nvidia-gpu-operator
oc get pods -n nvidia-gpu-operator -o wide
oc exec -n nvidia-gpu-operator -it <driver-pod> -- nvidia-smi

# ---- RHOAI ----
oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}'
oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}'

# ---- Model download and serving ----
huggingface-cli download meta-llama/Llama-3.1-8B-Instruct --local-dir /home/ocpadmin/models/llama-3.1-8b-instruct
podman build -t registry.a1procpvirt01.bank.sbi:8443/models/llama-3.1-8b-instruct:latest -f Modelcar.Dockerfile .
podman push registry.a1procpvirt01.bank.sbi:8443/models/llama-3.1-8b-instruct:latest --cert-dir /etc/containers/certs.d
oc get inferenceservice -n llm-serving
curl -sk "$(oc get inferenceservice llama-3-1-8b-instruct -n llm-serving -o jsonpath='{.status.url}')/v1/chat/completions" \
  -H "Content-Type: application/json" -d '{"model":"llama-3-1-8b-instruct","messages":[{"role":"user","content":"hello"}]}'
```

---

*End of Document 2.*
