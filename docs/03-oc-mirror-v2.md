# Chương 03 - OCMirror (Disconnected)

> OCP 4.22 + OpenShift Virtualization Production Lite

## Mục tiêu
- Mirror OCP release image
- Mirror Operators cần thiết cho OCPV
- Mirror theo mô hình Internet → Disk → Nexus
- Backup cluster-resources
- Verify image trên Nexus

## Kiến trúc

OCMirror (Internet + LAN)
↓
Mirror to Disk
↓
Trust Nexus CA
↓
Disk → Nexus (8443)
↓
Bastion / OCP

## Kiểm tra công cụ
```bash
oc version --client
oc mirror version
```

## Kiểm tra Pull Secret
```bash
podman login registry.redhat.io --authfile pull-secret.json
```
Expected: Login Succeeded!

## Kiểm tra dung lượng đĩa
```bash
df -h
```
Khuyến nghị >= 500GB trống.

## Kiểm tra kết nối internet từ server chạy oc-mirror lấy tài nguyên từ registry của Redhat

### Sau khi sử dụng lệnh df -h, thấy phân vùng nào còn trống nhiều thì thực hiện tạo thư mục workspace chứa dữ liệu
cấu trúc file nên để như sau

/home/oc-mirror-workspace/
├── bin/
├── auth/
├── configs/
├── mirror/
│   ├── mirror_seq1_000000.tar
│   ├── mirror_seq1_000001.tar
│   ├── oc-mirror-workspace/
│   │   ├── results-*.yaml
│   │   └── mapping.txt
│   └── publish/
├── logs/
└── archive/

```bash
mkdir -p /root/oc-mirror-workspace/
mkdir -p /root/oc-mirror-workspace/auth/
mkdir -p /root/oc-mirror-workspace/configs/
mkdir -p /root/oc-mirror-workspace/mirror/
mkdir -p /root/oc-mirror-workspace/logs/
mkdir -p /root/oc-mirror-workspace/archive/
mkdir -p /root/oc-mirror-workspace/bin
```

## Truy cập vào Redhat để lấy thông tin pull secret bằng lệnh
```bash
https://console.redhat.com/openshift/install/pull-secret
```

## Tải file pull secret sau đó copy sang bastion

```bash
scp pull-secret.txt root@192.168.0.15:/root/pull-secret.json
```

## Kiểm tra file pull secret
```bash
ls -l /root/pull-secret.json
jq . /root/pull-secret.json >/dev/null && echo "Pull secret OK"
```
## Phase 2 – Cài oc và oc-mirror

### 1. Tạo workspace
```bash
mkdir -p /home/oc-mirror-workspace
cd /home/oc-mirror-workspace
```

Kiểm tra:
```bash
pwd
```

Kết quả mong muốn:
```
/home/oc-mirror-workspace
```

### 2. Download OpenShift Client (oc + kubectl)
```bash
curl -LO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.21.0/openshift-client-linux.tar.gz
```

Kiểm tra:
```bash
ls -lh openshift-client-linux.tar.gz
```

### 3. Giải nén
```bash
tar -xzf openshift-client-linux.tar.gz
```

Kiểm tra:
```bash
ls -l
```

Kết quả mong muốn:
```
kubectl
oc
README.md
openshift-client-linux.tar.gz
```

### 4. Cài oc và kubectl
```bash
install -m 755 oc /usr/local/bin/oc
install -m 755 kubectl /usr/local/bin/kubectl
```

### 5. Thêm PATH

Do RHEL của bạn thiếu `/usr/local/bin`:

```bash
echo 'export PATH=/usr/local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Kiểm tra:
```bash
which oc
oc version --client
```

Kết quả mong muốn:
``` bash
/usr/local/bin/oc

Client Version: 4.22.0
Kustomize Version: ...
```

### 6. Download oc-mirror

Tại thời điểm hiện tại, oc-mirror được phát hành cùng client.

Download:

```bash
cd /root/oc-mirror-workspace/bin/
curl -LO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.21.0/oc-mirror.tar.gz
```

Kiểm tra:
```bash
ls -lh oc-mirror.tar.gz
```

### 7. Giải nén oc-mirror
```bash
tar -xzf oc-mirror.tar.gz
```

Kiểm tra:
```bash
ls -l
```

Bạn sẽ thấy thêm:
```
oc-mirror
```

### 8. Cài oc-mirror
```bash
install -m 755 oc-mirror /usr/local/bin/oc-mirror
```

### 9. Kiểm tra
```bash
which oc-mirror
oc-mirror version
```

Kết quả mong muốn:
```
/usr/local/bin/oc-mirror
...
```

### 10. Kiểm tra plugin mirror của oc
```bash
oc mirror --help
```

Nếu hiện ra phần hướng dẫn sử dụng thì mọi thứ đã sẵn sàng.

### Sau khi hoàn thành

Chạy:
```bash
oc version --client
oc-mirror version
```
và gửi mình output.


## ImageSetConfiguration
```yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
archiveSize: 50

mirror:
  platform:
    graph: true
    channels:
      - name: stable-4.22
        minVersion: <OCP_VERSION>
        maxVersion: <OCP_VERSION>

  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.22
      packages:
        - name: kubevirt-hyperconverged
          channels:
            - name: stable
        - name: kubernetes-nmstate-operator
          channels:
            - name: stable
        - name: local-storage-operator
          channels:
            - name: stable
        - name: metallb-operator
          channels:
            - name: stable
```

## Mirror Internet → Disk
```bash
oc mirror   --v2   --authfile pull-secret.json   -c imageset-config.yaml   file:///root/oc-mirror/mirror-data
```

## Kiểm tra kết quả
```bash
echo $?
ls /root/oc-mirror/mirror-data
du -sh /root/oc-mirror/mirror-data
```
Expected:
- exit code = 0
- Có cluster-resources
- Có mirror_seq*

## Backup cluster-resources
```bash
cp -r /root/oc-mirror/mirror-data/cluster-resources /root/backup-cluster-resources

ls /root/backup-cluster-resources
```
Expected:
- idms-oc-mirror.yaml
- itms-oc-mirror.yaml
- catalogSource-*.yaml

## Trust Nexus CA
```bash
mkdir -p /etc/containers/certs.d/nexus.<base_domain>:8443
cp ca.crt /etc/containers/certs.d/nexus.<base_domain>:8443/ca.crt
```

## Login Nexus
```bash
podman login nexus.<base_domain>:8443
```
Expected: Login Succeeded!

## Mirror Disk → Nexus
```bash
oc mirror   --v2   --from file:///root/oc-mirror/mirror-data   docker://nexus.<base_domain>:8443/ocp4
```

## Verify Nexus
```bash
echo $?
```
Expected: 0

Kiểm tra trên Nexus GUI:
- Browse
- ocp4
- Có openshift-release-dev
- Có redhat

## Troubleshooting
| Lỗi | Kiểm tra |
|---|---|
| unauthorized | Pull Secret |
| x509 | CA |
| no space left | Disk |
| catalogSource thiếu | cluster-resources |
| push thất bại | Nexus/TLS |

## Checklist
- [ ] Pull Secret OK
- [ ] Mirror to Disk OK
- [ ] cluster-resources backed up
- [ ] Login Nexus OK
- [ ] Disk to Nexus OK
- [ ] Nexus có đầy đủ image
