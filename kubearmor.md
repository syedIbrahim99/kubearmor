# KubeArmor Installation and Policy Enforcement Guide

## 📌 Official Documentation

* [KubeArmor Deployment Guide](https://docs.kubearmor.io/kubearmor/quick-links/deployment_guide?utm_source=chatgpt.com)

---

# 1️⃣ Install KubeArmor

## Add Helm Repository

```bash
helm repo add kubearmor https://kubearmor.github.io/charts
helm repo update kubearmor
```

---

## Install KubeArmor Operator

```bash
helm upgrade --install kubearmor-operator kubearmor/kubearmor-operator \
-n kubearmor --create-namespace
```

---

## Apply Sample Configuration

```bash
kubectl apply -f https://raw.githubusercontent.com/kubearmor/KubeArmor/main/pkg/KubeArmorOperator/config/samples/sample-config.yml
```

---

# 2️⃣ Install kArmor CLI (Optional)

```bash
curl -sfL http://get.kubearmor.io/ | sudo sh -s -- -b /usr/local/bin
```

---

# 3️⃣ Verify KubeArmor Installation

```bash
kubectl get pods -n kubearmor -o wide
```

---

# 4️⃣ Create KubeArmor Policy

## policy.yaml

This policy blocks shell access inside the protected pod.

```yaml
apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: block-shell-access
  namespace: codeshelf-dev

spec:
  selector:
    matchLabels:
      security: armor

  process:
    matchPaths:
    - path: /bin/bash
    - path: /usr/bin/bash
    - path: /bin/sh
    - path: /usr/bin/sh
    - path: /bin/dash
    - path: /usr/bin/dash

  action:
    Block
```

---

# 5️⃣ Create Test Pod

## pod.yaml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-pod
  labels:
    app: nginx
    security: armor

  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: runtime/default

spec:
  containers:
  - name: nginx
    image: nginx
```

---

# 6️⃣ View KubeArmor Logs

```bash
karmor logs -n default
```

---

## Sample Response

```text
== Alert / 2026-04-27 06:47:24.146870 ==
ClusterName: default
HostName: worker-1
NamespaceName: default
PodName: nginx-pod
Labels: app=nginx
ContainerName: nginx
ContainerID: 7eedee7b355743e2e4faea1300c99d947299f654b9c96daf49add6eb030f4c58
ContainerImage: docker.io/library/nginx:latest@sha256:6e23479198b998e5e25921dff8455837c7636a67111a04a635cf1bb363d199dc
Type: MatchedPolicy
PolicyName: block-shell-access
Source: /usr/bin/runc
Resource: /bin/bash
Operation: Process
Action: Block
Data: lsm=SECURITY_BPRM_CHECK
EventData: map[Lsm:SECURITY_BPRM_CHECK]
Enforcer: BPFLSM
Result: Permission denied
Cwd: /
ExecEvent: map[ExecID:38441054057213 ExecutableName:runc:[2:INIT]]
HostPID: 8950
HostPPID: 8936
KubeArmorVersion: v1.6.18-dirty
Owner: map[Name:nginx-pod Namespace:default]
PID: 41
PPID: 8936
ParentProcessName: /usr/bin/runc
ProcessName: /usr/bin/bash
TTY: pts0
UID: 0
```

---

# 7️⃣ Enable BPF LSM on Worker Nodes

## Edit GRUB Configuration

```bash
sudo nano /etc/default/grub
```

Update the following line:

```bash
GRUB_CMDLINE_LINUX="lsm=lockdown,yama,apparmor,bpf"
```

---

## Update GRUB and Reboot

```bash
sudo update-grub
sudo reboot
```

---

## Verify Enabled LSM Modules

```bash
cat /sys/kernel/security/lsm
```

### Expected Output

```text
lockdown,capability,yama,apparmor,bpf
```

---

# 8️⃣ Restart KubeArmor DaemonSet

```bash
kubectl rollout restart ds -n kubearmor
```

---

# 9️⃣ Verify KubeArmor Pods

```bash
kubectl get pods -n kubearmor -o wide -w
```

---

## Sample Output

```text
NAME                                    READY   STATUS    RESTARTS      AGE   IP               NODE       NOMINATED NODE   READINESS GATES
kubearmor-bpf-containerd-98c2c-vwxvx    1/1     Running   0             18m   192.168.30.202   worker-1   <none>           <none>
kubearmor-controller-69f896fc94-fp97b   1/1     Running   1 (19m ago)   49m   10.244.226.73    worker-1   <none>           <none>
kubearmor-operator-b789b76b5-8282w      1/1     Running   1 (19m ago)   50m   10.244.226.74    worker-1   <none>           <none>
kubearmor-relay-558f89b75d-qk8vl        1/1     Running   1 (19m ago)   50m   10.244.226.75    worker-1   <none>           <none>
```

---

# 🔟 Validate Policy Enforcement

## Attempt to Access Shell Inside Pod

```bash
kubectl exec -it nginx-pod -- bash
```

---

## Expected Output

```text
exec /usr/bin/bash: permission denied
command terminated with exit code 255
```

---

# 1️⃣1️⃣ Verify Policy Violation Logs

```bash
karmor logs -n default
```

---

## Sample Enforcement Log

```text
== Alert / 2026-04-27 06:47:24.146870 ==
ClusterName: default
HostName: worker-1
NamespaceName: default
PodName: nginx-pod
Labels: app=nginx
ContainerName: nginx
ContainerID: 7eedee7b355743e2e4faea1300c99d947299f654b9c96daf49add6eb030f4c58
ContainerImage: docker.io/library/nginx:latest@sha256:6e23479198b998e5e25921dff8455837c7636a67111a04a635cf1bb363d199dc
Type: MatchedPolicy
PolicyName: block-shell-access
Source: /usr/bin/runc
Resource: /bin/bash
Operation: Process
Action: Block
Data: lsm=SECURITY_BPRM_CHECK
EventData: map[Lsm:SECURITY_BPRM_CHECK]
Enforcer: BPFLSM
Result: Permission denied
Cwd: /
ExecEvent: map[ExecID:38441054057213 ExecutableName:runc:[2:INIT]]
HostPID: 8950
HostPPID: 8936
KubeArmorVersion: v1.6.18-dirty
Owner: map[Name:nginx-pod Namespace:default]
PID: 41
PPID: 8936
ParentProcessName: /usr/bin/runc
ProcessName: /usr/bin/bash
TTY: pts0
UID: 0
```

---


