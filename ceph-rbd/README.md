---

# **1. Ceph Storage Cluster  접속**

- **Bastion 접속 :  `203.0.113.10`**

```bash
root@bastion:~$ ssh ubuntu@203.0.113.10 -p 8522
ubuntu@203.0.113.10's password: CHANGE_ME_SSH_PASSWORD
```

---

- **horizon_drallinone1 접속 정보 확인**
    - `cat /etc/haproxy/haproxy.cfg`

```bash
listen horizon_drallinone1
  bind *:33443 ssl crt /opt/certificate/internal-ca.pem ssl verify none
  mode tcp
  server vip 192.168.30.90:443 check inter 2000 rise 2 fall 5 ssl crt /opt/certificate/internal-ca.pem ssl verify none
```

---

- **horizon_drallinone1 (=Ceph Storage) 접속**
    - 실제 물리장비 이므로 작업 할때 주의
        - **Ceph Storage 접속 :** `192.168.30.90`

```bash
root@bastion:~$ ssh root@192.168.30.90
root@192.168.30.90's password: CHANGE_ME_SSH_PASSWORD
```

---

- Ceph 상태 확인
    - ceph version : `17.2.6`

```bash
root@drallinone1:~# ceph -s
  cluster:
    id:     60ba6e1e-2e4c-40e6-a2b7-98341db296c3
    health: HEALTH_WARN
            1 pool(s) do not have an application enabled

  services:
    mon: 1 daemons, quorum drallinone1 (age 7d)
    mgr: drallinone1(active, since 10d)
    mds: 1/1 daemons up
    osd: 24 osds: 24 up (since 10d), 24 in (since 10d)
    rgw: 1 daemon active (1 hosts, 1 zones)

  data:
    volumes: 1/1 healthy
    pools:   13 pools, 433 pgs
    objects: 78.57k objects, 294 GiB
    usage:   571 GiB used, 167 TiB / 168 TiB avail
    pgs:     433 active+clean

  io:
    client:   0 B/s rd, 594 KiB/s wr, 0 op/s rd, 126 op/s wr
```

---

# 2. **k8s-ceph rbd 연동**

- RBD는 Ceph에서 제공하는 블록 스토리지입니다. ReadWriteOnce 특성의 PVC를 생성할 수 있습니다. 대체적으로 CephFS 보다 성능이 높지만, 볼륨 공유가 안되는 블록 스토리지입니다.

## 2-1. Ceph CSI GitHub 저장소 클론

- Ceph CSI(컨테이너 스토리지 인터페이스) 플러그인 소스코드를 클론합니다.
- CSI는 Kubernetes가 Ceph RBD 및 CephFS와 통신하게 해주는 드라이버입니다.
- 클론된 소스코드는 배포 매니페스트 예제 등을 제공하며, 실제 배포에 활용됩니다.

```bash
git clone https://github.com/ceph/ceph-csi.git
```

---

## 2-2. Ceph 풀 생성

- `kubernetes`라는 이름의 **RADOS pool**을 생성합니다.
- 이 풀은 RBD 이미지들이 저장되는 공간이며, Kubernetes에서 PV(PersistentVolume)를 생성할 때 사용됩니다.

```bash
ceph osd pool create kubernetes
pool 'kubernetes' created

# 생성된 풀 리스트 확인 
ceph osd pool ls | grep kubernetes
kubernetes

# 풀 상세 정보 확인
ceph osd pool get kubernetes all
size: 2
min_size: 1
pg_num: 32
pgp_num: 32
crush_rule: replicated_rule
hashpspool: true
nodelete: false
nopgchange: false
nosizechange: false
write_fadvise_dontneed: false
noscrub: false
nodeep-scrub: false
use_gmt_hitset: 1
fast_read: 0
pg_autoscale_mode: on
eio: false
bulk: false
```

---

- `kubernetes` 풀에 RBD 메타데이터용 Object를 초기화합니다.
- 이 작업은 CSI나 RBD 커맨드를 사용하기 위해 **필수**입니다. (librbd가 사용할 OMAP 메타데이터 저장소 등 준비)

```bash
rbd pool init kubernetes

# 아직 아무 이미지가 없으면 출력 없음
rbd ls -p kubernetes
```

---

## 2-3. Ceph 사용자 인증 생성

```bash
ceph auth get-or-create client.kubernetes \
  mon 'profile rbd' \
  osd 'profile rbd pool=kubernetes' \
  mgr 'profile rbd pool=kubernetes'
  
[client.kubernetes]
	key = CHANGE_ME_CEPH_RBD_USER_KEY
```

→ ceph mon dump로 나온 fsid 값 필요.

```bash
ceph fsid
60ba6e1e-2e4c-40e6-a2b7-98341db296c3
```

---

## 2-4. Secret 생성

- Kubernetes Cluster 에서 생성

```bash
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: kubernetes               # client.kubernetes
  userKey: CHANGE_ME_CEPH_RBD_USER_KEY
```

→ 위의 값에서 ceph auth로 생성된 키 값 입력.

```bash
kubectl get secrets -n default
NAME             TYPE     DATA   AGE
csi-rbd-secret   Opaque   2      21s
```

---

## 2-5. RBAC 생성

- vi 2.csi-provisioner-rbac.yaml

```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-csi-provisioner
  # replace with non-default namespace name
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-external-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims/status"]
    verbs: ["update", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots"]
    verbs: ["get", "list", "patch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots/status"]
    verbs: ["get", "list", "patch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotcontents"]
    verbs: ["create", "get", "list", "watch", "update", "delete", "patch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments/status"]
    verbs: ["patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["csinodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotcontents/status"]
    verbs: ["update", "patch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-csi-provisioner-role
subjects:
  - kind: ServiceAccount
    name: rbd-csi-provisioner
    # replace with non-default namespace name
    namespace: default
roleRef:
  kind: ClusterRole
  name: rbd-external-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # replace with non-default namespace name
  namespace: default
  name: rbd-external-provisioner-cfg
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "watch", "list", "delete", "update", "create"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-csi-provisioner-role-cfg
  # replace with non-default namespace name
  namespace: default
subjects:
  - kind: ServiceAccount
    name: rbd-csi-provisioner
    # replace with non-default namespace name
    namespace: default
roleRef:
  kind: Role
  name: rbd-external-provisioner-cfg
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f 2.csi-provisioner-rbac.yaml
```

- vi 3.csi-nodeplugin-rbac.yaml

```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-csi-nodeplugin
  # replace with non-default namespace name
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-csi-nodeplugin
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get"]
  # allow to read Vault Token and connection options from the Tenants namespace
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["list", "get"]
  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-csi-nodeplugin
subjects:
  - kind: ServiceAccount
    name: rbd-csi-nodeplugin
    # replace with non-default namespace name
    namespace: default
roleRef:
  kind: ClusterRole
  name: rbd-csi-nodeplugin
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f 3.csi-nodeplugin-rbac.yaml
```

---

## 2-6. **Key Management Service (키 관리 서비스)** 설정

- vi kms-config

```bash
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
#        "vault-test": {
#        "encryptionKMSType": "vault",
#        "vaultAddress": "http://vault.default.svc.cluster.local:8200",
#        "vaultAuthPath": "/v1/auth/kubernetes/login",
#        "vaultRole": "csi-kubernetes",
#        "vaultPassphraseRoot": "/v1/secret",
#        "vaultPassphrasePath": "ceph-csi/",
#        "vaultCAVerify": "false"
#      }
metadata:
  name: ceph-csi-encryption-kms-config
```

```bash
kubectl create -f 4.kms-config.yaml
```

---

## 2-7. `ceph-csi-config` 생성

```bash
ceph mon dump

epoch 1
fsid 60ba6e1e-2e4c-40e6-a2b7-98341db296c3
last_changed 2025-04-03T17:31:42.448468+0900
created 2025-04-03T17:31:42.448468+0900
min_mon_release 17 (quincy)
election_strategy: 1
0: [v2:192.168.20.14:3300/0,v1:192.168.20.14:6789/0] mon.drallinone1
dumped monmap epoch 1
```

- vi 5.ceph-csi-config

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: ceph-csi-config
  namespace: default
data:
  config.json: |-
    {
      "clusterID": "60ba6e1e-2e4c-40e6-a2b7-98341db296c3",
      "monitors": ["192.168.20.14:6789"],
      "storageClass": "ceph-rbd"
    }
```

```bash
kubectl create -f 5.ceph-csi-config.yaml
```

---

## 2-8. `Plugin-provisioner`

- vi 6.csi-rbdplugin-provisioner.yaml

```bash
---
kind: Service
apiVersion: v1
metadata:
  name: csi-rbdplugin-provisioner
  # replace with non-default namespace name
  namespace: default
  labels:
    app: csi-metrics
spec:
  selector:
    app: csi-rbdplugin-provisioner
  ports:
    - name: http-metrics
      port: 8080
      protocol: TCP
      targetPort: 8680
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: csi-rbdplugin-provisioner
  # replace with non-default namespace name
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: csi-rbdplugin-provisioner
  template:
    metadata:
      labels:
        app: csi-rbdplugin-provisioner
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - csi-rbdplugin-provisioner
              topologyKey: "kubernetes.io/hostname"
      serviceAccountName: rbd-csi-provisioner
      priorityClassName: system-cluster-critical
      containers:
        - name: csi-provisioner
          image: registry.k8s.io/sig-storage/csi-provisioner:v3.3.0
          args:
            - "--csi-address=$(ADDRESS)"
            - "--v=1"
            - "--timeout=150s"
            - "--retry-interval-start=500ms"
            - "--leader-election=true"
            #  set it to true to use topology based provisioning
            - "--feature-gates=Topology=false"
            - "--feature-gates=HonorPVReclaimPolicy=true"
            - "--prevent-volume-mode-conversion=true"
            # if fstype is not specified in storageclass, ext4 is default
            - "--default-fstype=ext4"
            - "--extra-create-metadata=true"
          env:
            - name: ADDRESS
              value: unix:///csi/csi-provisioner.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
        - name: csi-snapshotter
          image: registry.k8s.io/sig-storage/csi-snapshotter:v6.1.0
          args:
            - "--csi-address=$(ADDRESS)"
            - "--v=1"
            - "--timeout=150s"
            - "--leader-election=true"
            - "--extra-create-metadata=true"
          env:
            - name: ADDRESS
              value: unix:///csi/csi-provisioner.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
        - name: csi-attacher
          image: registry.k8s.io/sig-storage/csi-attacher:v4.0.0
          args:
            - "--v=1"
            - "--csi-address=$(ADDRESS)"
            - "--leader-election=true"
            - "--retry-interval-start=500ms"
            - "--default-fstype=ext4"
          env:
            - name: ADDRESS
              value: /csi/csi-provisioner.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
        - name: csi-resizer
          image: registry.k8s.io/sig-storage/csi-resizer:v1.6.0
          args:
            - "--csi-address=$(ADDRESS)"
            - "--v=1"
            - "--timeout=150s"
            - "--leader-election"
            - "--retry-interval-start=500ms"
            - "--handle-volume-inuse-error=false"
            - "--feature-gates=RecoverVolumeExpansionFailure=true"
          env:
            - name: ADDRESS
              value: unix:///csi/csi-provisioner.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
        - name: csi-rbdplugin
          # for stable functionality replace canary with latest release version
          image: quay.io/cephcsi/cephcsi:canary
          args:
            - "--nodeid=$(NODE_ID)"
            - "--type=rbd"
            - "--controllerserver=true"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--csi-addons-endpoint=$(CSI_ADDONS_ENDPOINT)"
            - "--v=5"
            - "--drivername=rbd.csi.ceph.com"
            - "--pidlimit=-1"
            - "--rbdhardmaxclonedepth=8"
            - "--rbdsoftmaxclonedepth=4"
            - "--enableprofiling=false"
            - "--setmetadata=true"
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            # - name: KMS_CONFIGMAP_NAME
            #   value: encryptionConfig
            - name: CSI_ENDPOINT
              value: unix:///csi/csi-provisioner.sock
            - name: CSI_ADDONS_ENDPOINT
              value: unix:///csi/csi-addons.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - mountPath: /dev
              name: host-dev
            - mountPath: /sys
              name: host-sys
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - name: ceph-csi-config
              mountPath: /etc/ceph-csi-config/
            - name: ceph-csi-encryption-kms-config
              mountPath: /etc/ceph-csi-encryption-kms-config/
            - name: keys-tmp-dir
              mountPath: /tmp/csi/keys
#            - name: ceph-config
#              mountPath: /etc/ceph/
            - name: oidc-token
              mountPath: /run/secrets/tokens
              readOnly: true
        - name: csi-rbdplugin-controller
          # for stable functionality replace canary with latest release version
          image: quay.io/cephcsi/cephcsi:canary
          args:
            - "--type=controller"
            - "--v=5"
            - "--drivername=rbd.csi.ceph.com"
            - "--drivernamespace=$(DRIVER_NAMESPACE)"
            - "--setmetadata=true"
          env:
            - name: DRIVER_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: ceph-csi-config
              mountPath: /etc/ceph-csi-config/
            - name: keys-tmp-dir
              mountPath: /tmp/csi/keys
#            - name: ceph-config
#              mountPath: /etc/ceph/
        - name: liveness-prometheus
          image: quay.io/cephcsi/cephcsi:canary
          args:
            - "--type=liveness"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--metricsport=8680"
            - "--metricspath=/metrics"
            - "--polltime=60s"
            - "--timeout=3s"
          env:
            - name: CSI_ENDPOINT
              value: unix:///csi/csi-provisioner.sock
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
          imagePullPolicy: "IfNotPresent"
      volumes:
        - name: host-dev
          hostPath:
            path: /dev
        - name: host-sys
          hostPath:
            path: /sys
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: socket-dir
          emptyDir: {
            medium: "Memory"
          }
#        - name: ceph-config
#          configMap:
#            name: ceph-config
        - name: ceph-csi-config
          configMap:
            name: ceph-csi-config
        - name: ceph-csi-encryption-kms-config
          configMap:
            name: ceph-csi-encryption-kms-config
        - name: keys-tmp-dir
          emptyDir: {
            medium: "Memory"
          }
        - name: oidc-token
          projected:
            sources:
              - serviceAccountToken:
                  path: oidc-token
                  expirationSeconds: 3600
                  audience: ceph-csi-kms

```

```bash
kubectl create -f 6.csi-rbdplugin-provisioner.yaml
```

- 배포 확인

```bash
kubectl get svc,pods
NAME                                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/csi-rbdplugin-provisioner   ClusterIP   10.10.0.250   <none>        8080/TCP   119m
service/kubernetes                  ClusterIP   10.10.0.1     <none>        443/TCP    6d22h

NAME                                             READY   STATUS    RESTARTS   AGE
pod/csi-rbdplugin-provisioner-5dd6987856-8l95j   7/7     Running   0          8m53s
pod/csi-rbdplugin-provisioner-5dd6987856-b4n9k   7/7     Running   0          8m53s
pod/csi-rbdplugin-provisioner-5dd6987856-ntwng   7/7     Running   0          8m53s
```

---

## 2-9. Plugin

- vi 7.csi-rbdplugin.yaml

```bash
---
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: csi-rbdplugin
  # replace with non-default namespace name
  namespace: default
spec:
  selector:
    matchLabels:
      app: csi-rbdplugin
  template:
    metadata:
      labels:
        app: csi-rbdplugin
    spec:
      serviceAccountName: rbd-csi-nodeplugin
      hostNetwork: true
      hostPID: true
      priorityClassName: system-node-critical
      # to use e.g. Rook orchestrated cluster, and mons' FQDN is
      # resolved through k8s service, set dns policy to cluster first
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: driver-registrar
          # This is necessary only for systems with SELinux, where
          # non-privileged sidecar containers cannot access unix domain socket
          # created by privileged CSI driver container.
          securityContext:
            privileged: true
            allowPrivilegeEscalation: true
          image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.6.2
          args:
            - "--v=1"
            - "--csi-address=/csi/csi.sock"
            - "--kubelet-registration-path=/var/lib/kubelet/plugins/rbd.csi.ceph.com/csi.sock"
          env:
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - name: registration-dir
              mountPath: /registration
        - name: csi-rbdplugin
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          # for stable functionality replace canary with latest release version
          image: quay.io/cephcsi/cephcsi:canary
          args:
            - "--nodeid=$(NODE_ID)"
            - "--pluginpath=/var/lib/kubelet/plugins"
            - "--stagingpath=/var/lib/kubelet/plugins/kubernetes.io/csi/"
            - "--type=rbd"
            - "--nodeserver=true"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--csi-addons-endpoint=$(CSI_ADDONS_ENDPOINT)"
            - "--v=5"
            - "--drivername=rbd.csi.ceph.com"
            - "--enableprofiling=false"
            # If topology based provisioning is desired, configure required
            # node labels representing the nodes topology domain
            # and pass the label names below, for CSI to consume and advertise
            # its equivalent topology domain
            # - "--domainlabels=failure-domain/region,failure-domain/zone"
            #
            # Options to enable read affinity.
            # If enabled Ceph CSI will fetch labels from kubernetes node and
            # pass `read_from_replica=localize,crush_location=type:value` during
            # rbd map command. refer:
            # https://docs.ceph.com/en/latest/man/8/rbd/#kernel-rbd-krbd-options
            # for more details.
            # - "--enable-read-affinity=true"
            # - "--crush-location-labels=topology.io/zone,topology.io/rack"
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            # - name: KMS_CONFIGMAP_NAME
            #   value: encryptionConfig
            - name: CSI_ENDPOINT
              value: unix:///csi/csi.sock
            - name: CSI_ADDONS_ENDPOINT
              value: unix:///csi/csi-addons.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - mountPath: /dev
              name: host-dev
            - mountPath: /sys
              name: host-sys
            - mountPath: /run/mount
              name: host-mount
            - mountPath: /etc/selinux
              name: etc-selinux
              readOnly: true
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - name: ceph-csi-config
              mountPath: /etc/ceph-csi-config/
            - name: ceph-csi-encryption-kms-config
              mountPath: /etc/ceph-csi-encryption-kms-config/
            - name: plugin-dir
              mountPath: /var/lib/kubelet/plugins
              mountPropagation: "Bidirectional"
            - name: mountpoint-dir
              mountPath: /var/lib/kubelet/pods
              mountPropagation: "Bidirectional"
            - name: keys-tmp-dir
              mountPath: /tmp/csi/keys
            - name: ceph-logdir
              mountPath: /var/log/ceph
#            - name: ceph-config
#              mountPath: /etc/ceph/
            - name: oidc-token
              mountPath: /run/secrets/tokens
              readOnly: true
        - name: liveness-prometheus
          securityContext:
            privileged: true
            allowPrivilegeEscalation: true
          image: quay.io/cephcsi/cephcsi:canary
          args:
            - "--type=liveness"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--metricsport=8680"
            - "--metricspath=/metrics"
            - "--polltime=60s"
            - "--timeout=3s"
          env:
            - name: CSI_ENDPOINT
              value: unix:///csi/csi.sock
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
          imagePullPolicy: "IfNotPresent"
      volumes:
        - name: socket-dir
          hostPath:
            path: /var/lib/kubelet/plugins/rbd.csi.ceph.com
            type: DirectoryOrCreate
        - name: plugin-dir
          hostPath:
            path: /var/lib/kubelet/plugins
            type: Directory
        - name: mountpoint-dir
          hostPath:
            path: /var/lib/kubelet/pods
            type: DirectoryOrCreate
        - name: ceph-logdir
          hostPath:
            path: /var/log/ceph
            type: DirectoryOrCreate
        - name: registration-dir
          hostPath:
            path: /var/lib/kubelet/plugins_registry/
            type: Directory
        - name: host-dev
          hostPath:
            path: /dev
        - name: host-sys
          hostPath:
            path: /sys
        - name: etc-selinux
          hostPath:
            path: /etc/selinux
        - name: host-mount
          hostPath:
            path: /run/mount
        - name: lib-modules
          hostPath:
            path: /lib/modules
#        - name: ceph-config
#          configMap:
#            name: ceph-config
        - name: ceph-csi-config
          configMap:
            name: ceph-csi-config
        - name: ceph-csi-encryption-kms-config
          configMap:
            name: ceph-csi-encryption-kms-config
        - name: keys-tmp-dir
          emptyDir: {
            medium: "Memory"
          }
        - name: oidc-token
          projected:
            sources:
              - serviceAccountToken:
                  path: oidc-token
                  expirationSeconds: 3600
                  audience: ceph-csi-kms
---
# This is a service to expose the liveness metrics
apiVersion: v1
kind: Service
metadata:
  name: csi-metrics-rbdplugin
  # replace with non-default namespace name
  namespace: default
  labels:
    app: csi-metrics
spec:
  ports:
    - name: http-metrics
      port: 8080
      protocol: TCP
      targetPort: 8680
  selector:
    app: csi-rbdplugin
```

```bash
kubectl create -f 7.csi-rbdplugin.yaml
```

- 배포 조회

```bash
kubectl get pods
NAME                                         READY   STATUS    RESTARTS   AGE
csi-rbdplugin-6fwl5                          3/3     Running   0          23s
csi-rbdplugin-dzjfd                          3/3     Running   0          23s
csi-rbdplugin-fdlbg                          3/3     Running   0          23s
```

---

## 2-10. Storage Class

- vi 8.csi-rbd-sc.yaml

```bash
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
allowVolumeExpansion: true
parameters:
   clusterID: 60ba6e1e-2e4c-40e6-a2b7-98341db296c3
   pool: kubernetes
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
mountOptions:
   - discard
```

```bash
kubectl create -f 8.csi-rbd-sc.yaml
```

- 배포 조회

```bash
kubectl get sc
NAME         PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
csi-rbd-sc   rbd.csi.ceph.com   Delete          Immediate           true                   48s
```

---

## 2-11. pvc

- vi 9.raw-block-pvc.yaml

```bash
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
```

```bash
kubectl create -f 9.raw-block-pvc.yaml
```

- 배포 조회

```bash
kubectl get pvc
NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
raw-block-pvc   Pending                                      csi-rbd-sc     <unset>                 1s
```

---

## 2-12. CSI-driver

- vi 10.csi-driver.yaml

```bash
---
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: "rbd.csi.ceph.com"
spec:
  attachRequired: true
  podInfoOnMount: false
  fsGroupPolicy: File
```

```bash
kubectl create -f 10.csi-driver.yaml
```

```bash
kubectl get pvc
```

→ 위 명령어 입력시 status 값이 Bound이고 volume 값이 있으면 완료.

---

# 3. PVC Pending 상태일 경우 조치방법

## 3-1. kubectl describe pvc 에러 상태

```bash
  Type     Reason                Age                From                                                                                              Message
  ----     ------                ----               ----                                                                                              -------
  Normal   ExternalProvisioning  3m (x42 over 13m)  persistentvolume-controller                                                                       Waiting for a volume to be created either by the external provisioner 'rbd.csi.ceph.com' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
  Warning  ProvisioningFailed    40s (x5 over 10m)  rbd.csi.ceph.com_csi-rbdplugin-provisioner-5986ff5c78-rwjzc_3e6339ea-df7d-4f04-9ab9-8d918e54df98  failed to provision volume with StorageClass "csi-rbd-sc": rpc error: code = DeadlineExceeded desc = context deadline exceeded
  Normal   Provisioning          32s (x6 over 13m)  rbd.csi.ceph.com_csi-rbdplugin-provisioner-5986ff5c78-rwjzc_3e6339ea-df7d-4f04-9ab9-8d918e54df98  External provisioner is provisioning volume for claim "default/raw-block-pvc"
```

- 노드 → ceph 쪽 통신이 안되서 발생하는 오류
    - Openstack에서 ceph-network Interface 추가

![image.png](attachment:c6d4a8f7-4042-42a6-8bd5-bcd08b2685b6:image.png)

---

- 각 노드들에 접속 후 dhclient 명령어 실행
    - DHCP 클라이언트를 수동으로 실행해서 IP를 재요청할 때 사용하는 명령어

```bash
dhclient
```

---

- 추가된 Ceph Interface 확인 및 ping, telnet 테스트

```bash
ip a | grep 172.168

root@k8s-worker02:~# ping 192.168.20.14
PING 192.168.20.14 (192.168.20.14) 56(84) bytes of data.
64 bytes from 192.168.20.14: icmp_seq=2 ttl=63 time=1.46 ms
64 bytes from 192.168.20.14: icmp_seq=3 ttl=63 time=0.475 ms
^C
--- 192.168.20.14 ping statistics ---
3 packets transmitted, 2 received, 33.3333% packet loss, time 2026ms
rtt min/avg/max/mdev = 0.475/0.965/1.456/0.490 ms
```

---

## 3-2. 미해결시

1. **각 워커노드에 ceph-interface 추가**

```bash
network:
    version: 2
    ethernets:
        ens3:
            dhcp4: true
            match:
                macaddress: fa:16:3e:35:e3:41
            mtu: 1450
            set-name: ens3
        ens7:
            dhcp4: false
            addresses: [192.168.21.210/24] # 노드 IP 
            match:
                macaddress: fa:16:3e:a3:7d:66 # 노드 MacAddress
            mtu: 1500
            routes:
              - to: 192.168.21.0/24 # Ceph IP 대역
                via: 192.168.21.1 # Ceph Gateway
              - to: 192.168.20.0/24 # Ceph 사설 IP (Private IP) 주소 
                via: 192.168.21.1
            set-name: ens7
```

1. **Deploy:** csi-rbdplugin-provisioner  에 아래 내용 추가

```bash
      hostNetwork: true
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
      - effect: NoSchedule
        operator: Exists
```

1. **PVC Bound 상태 확인**

```bash
root@bastion:~# kubectl get pvc
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
raw-block-pvc   Bound    pvc-2e3ea337-d99c-48db-b9ac-9b33ba84ca80   1Gi        RWO            csi-rbd-sc     <unset>                 14h
```
