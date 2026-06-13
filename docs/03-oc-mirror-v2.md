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
