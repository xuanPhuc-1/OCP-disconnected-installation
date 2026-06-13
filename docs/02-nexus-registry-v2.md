# Chương 02 - Nexus Registry (Disconnected)

> Phiên bản: OCP 4.22
> 
> Kiểu triển khai: UPI Traditional
>
> Registry: Nexus Community + Nginx TLS

## Mục tiêu

Sau khi hoàn thành chương này:

- [ ] Nexus hoạt động
- [ ] Docker Hosted Repository hoạt động
- [ ] HTTPS hoạt động
- [ ] Podman login thành công
- [ ] Push/Pull image thành công

## 1. Cài Podman

### Mục đích

Cài runtime để chạy Nexus.

### Thực hiện

```bash
dnf install -y podman
```

### Kiểm tra

```bash
podman version
```

### Kết quả mong đợi

Hiển thị thông tin version.

### Checklist

- [ ] Podman OK

## 2. Pull Nexus image (OCMIRROR)

### Thực hiện

```bash
podman pull docker.io/sonatype/nexus3:latest

podman save   -o nexus3.tar   docker.io/sonatype/nexus3:latest
```

### Kiểm tra

```bash
ls -lh nexus3.tar
```

### Checklist

- [ ] nexus3.tar tồn tại

## 3. Copy sang Nexus

```bash
scp nexus3.tar root@<IP_NEXUS>:/tmp/
```

Kiểm tra:

```bash
ssh root@<IP_NEXUS> ls -lh /tmp/nexus3.tar
```

- [ ] Copy thành công

## 4. Import Nexus

```bash
podman load -i /tmp/nexus3.tar
```

Kiểm tra:

```bash
podman images | grep nexus
```

- [ ] Image tồn tại

## 5. Chuẩn bị dữ liệu

```bash
mkdir -p /data/nexus
chown -R 200:200 /data/nexus
```

Kiểm tra:

```bash
ls -ld /data/nexus
```

- [ ] Permission đúng

## 6. Khởi động Nexus

```bash
podman run -d   --name nexus   --restart=always   -p 8081:8081   -p 8082:8082   -v /data/nexus:/nexus-data:Z   docker.io/sonatype/nexus3:latest
```

Kiểm tra:

```bash
podman ps
podman logs -f nexus
```

Kết quả mong đợi:

Started Sonatype Nexus

- [ ] Nexus Running

## 7. First Login

Truy cập:

http://<IP_NEXUS>:8081

Lấy password:

```bash
cat /data/nexus/admin.password
```

Tắt anonymous access.

- [ ] Login thành công

## 8. Docker Hosted Repository

Repositories
→ Create repository
→ docker(hosted)

Cấu hình:

- Name: ocp4
- HTTP: 8082
- HTTPS: Disabled
- Deployment Policy: Allow redeploy

Kiểm tra:

```bash
curl http://localhost:8082/v2/
```

Không được trả về 404.

- [ ] Docker Hosted OK

## 9. Nginx TLS

### Pull

```bash
podman pull docker.io/library/nginx:alpine

podman save   -o nginx.tar   docker.io/library/nginx:alpine
```

### Import

```bash
podman load -i nginx.tar
```

- [ ] nginx image OK

## 10. Tạo CA

```bash
openssl genrsa -out ca.key 4096
```

```bash
openssl req -x509   -new   -nodes   -key ca.key   -sha256   -days 3650   -out ca.crt   -subj "/CN=Local Registry CA"
```

Kiểm tra:

```bash
openssl x509 -in ca.crt -text -noout
```

- [ ] CA OK

## 11. Firewall

```bash
firewall-cmd   --permanent   --add-port={8081,8443}/tcp

firewall-cmd --reload
```

Kiểm tra:

```bash
firewall-cmd --list-ports
```

- [ ] Firewall OK

## 12. Trust CA

```bash
mkdir -p /etc/containers/certs.d/nexus.<base_domain>:8443

cp ca.crt /etc/containers/certs.d/nexus.<base_domain>:8443/
```

- [ ] Trust thành công

## 13. Login

```bash
podman login nexus.<base_domain>:8443
```

- [ ] Login thành công

## 14. Push/Pull Test

```bash
podman pull docker.io/library/nginx:latest

podman tag docker.io/library/nginx:latest nexus.<base_domain>:8443/ocp4/nginx:test

podman push nexus.<base_domain>:8443/ocp4/nginx:test

podman pull nexus.<base_domain>:8443/ocp4/nginx:test
```

- [ ] Push OK
- [ ] Pull OK

## Troubleshooting

| Lỗi | Kiểm tra |
|---|---|
| 404 /v2/ | Docker Hosted |
| x509 | CA |
| Push fail | Deployment Policy |
| Login fail | TLS |

## Checklist tổng

- [ ] Nexus Running
- [ ] Docker Hosted OK
- [ ] TLS OK
- [ ] Login OK
- [ ] Push OK
- [ ] Pull OK
