# Proyek 1 — Implementasi Stack Observability (Prometheus + Grafana) di Kubernetes

Implementasi ini membangun **stack observability** menggunakan **kube-prometheus-stack** (Prometheus Operator) untuk memantau **cluster Kubernetes** dan **aplikasi** secara proaktif, serta membuat **dashboard Grafana kustom**.

> Catatan: **File dashboard JSON sudah tersedia di repo ini** (sudah kamu upload sebelumnya). README ini menjelaskan langkah dari **nol sampai selesai** agar sistem bisa dijalankan ulang kapan pun.

---

## 1. Arsitektur Singkat

- **Prometheus Operator / kube-prometheus-stack**: instalasi Prometheus, Alertmanager, Grafana, exporters, dan ServiceMonitor bawaan.
- **Node Exporter + kube-state-metrics**: metrik node & objek Kubernetes (pod, deployment, dsb).
- **Aplikasi contoh (NGINX)** + **nginx-prometheus-exporter**: metrik request rate/traffic untuk kebutuhan panel aplikasi.
- **Grafana**: visualisasi metrik + import dashboard JSON kustom.

---

## 2. Prasyarat

- OS: Ubuntu (VM) / Linux setara
- Kubernetes lokal salah satu:
  - **kind** (direkomendasikan untuk VM) *atau*
  - **minikube**
- Tools:
  - `kubectl`
  - `helm`
  - `docker`

### 2.1 Instalasi Tools (Ubuntu)

> Jika sudah terpasang, lewati.

**Docker**
```bash
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
docker version
```

**kubectl**
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubectl
kubectl version --client
```

**Helm**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 3. Menyiapkan Kubernetes (pilih salah satu)

### Opsi A — kind (rekomendasi)
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind create cluster --name observability
kubectl cluster-info
```

### Opsi B — minikube
```bash
sudo snap install minikube
minikube start --driver=docker
kubectl cluster-info
```

---

## 4. Instalasi kube-prometheus-stack (Prometheus + Grafana)

### 4.1 Tambahkan repo Helm
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 4.2 Buat namespace monitoring
```bash
kubectl create namespace proyek1-monitoring
```

### 4.3 Install chart
```bash
helm install monitoring prometheus-community/kube-prometheus-stack -n proyek1-monitoring
```

### 4.4 Validasi pod
Tunggu sampai komponen utama **Running**:
```bash
kubectl get pods -n proyek1-monitoring
```

Jika ada yang `Pending/CrashLoopBackOff`, cek:
```bash
kubectl describe pod <nama-pod> -n proyek1-monitoring
kubectl get events -n proyek1-monitoring --sort-by=.lastTimestamp | tail -n 50
```

---

## 5. Akses Prometheus & Grafana

### 5.1 Port-forward Prometheus
Jika port 9090 bentrok, gunakan 9091 (disarankan):
```bash
kubectl -n proyek1-monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9091:9090
```
Buka: **http://localhost:9091**

### 5.2 Port-forward Grafana
Jika port 3000 bentrok, gunakan 3001:
```bash
kubectl -n proyek1-monitoring port-forward svc/monitoring-grafana 3001:80
```
Buka: **http://localhost:3001**

### 5.3 Ambil kredensial Grafana
```bash
kubectl -n proyek1-monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-user}" | base64 -d; echo
kubectl -n proyek1-monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```
Login ke Grafana menggunakan user/pass tersebut.

---

## 6. Verifikasi Metrik Cluster (Node & Pod)

### 6.1 Cek Targets di Prometheus
Buka Prometheus: **Status → Targets** lalu pastikan komponen ini **UP**:
- `kube-state-metrics`
- `node-exporter`
- `kubelet` / komponen kube-prometheus-stack lainnya

### 6.2 Query cepat di Prometheus
Contoh cek metrik node exporter:
```promql
up{job="node-exporter"}
```

---

## 7. Monitoring Aplikasi (NGINX + nginx-exporter)

> Bagian ini diperlukan jika kamu ingin panel aplikasi seperti **request rate / traffic** (misalnya `nginx_http_requests_total`).  
> Jika fokus hanya metrik cluster (CPU/RAM node), bagian ini boleh dilewati.

### 7.1 Deploy NGINX (aplikasi contoh) di namespace default
```bash
kubectl create deployment web --image=nginx:latest
kubectl expose deployment web --port=80 --name=web
kubectl get pods
kubectl get svc web
```

### 7.2 Deploy nginx-prometheus-exporter
Kita jalankan exporter yang membaca status NGINX via endpoint stub_status. Cara termudah untuk latihan:
- Gunakan image exporter resmi
- Jalankan exporter menargetkan service `web`

> Catatan: exporter butuh endpoint NGINX yang expose metric/status. Pada latihan sederhana, kamu bisa tetap gunakan exporter sebagai contoh endpoint `/metrics`, lalu fokus pada mekanisme ServiceMonitor dan dashboard. Jika kamu ingin metrik request yang benar-benar real, pastikan NGINX punya `stub_status` aktif.

Contoh deployment exporter (sederhana):
```bash
kubectl apply -f k8s/nginx-exporter.yaml
kubectl get pods
kubectl get svc nginx-exporter
```

> Jika file `k8s/nginx-exporter.yaml` belum ada, buat dengan label `app: nginx-exporter` dan service port `9113`.

---

## 8. Membuat ServiceMonitor untuk nginx-exporter

Agar Prometheus Operator *scrape* nginx-exporter, buat ServiceMonitor di namespace monitoring.

```bash
kubectl apply -f k8s/servicemonitor-nginx-exporter.yaml
kubectl get servicemonitor -n proyek1-monitoring
```

**Checklist ServiceMonitor benar**
- `namespaceSelector.matchNames` berisi namespace aplikasi (mis. `default`)
- `selector.matchLabels` sama dengan label pada **Service** exporter (mis. `app: nginx-exporter`)
- `endpoints.port` sama dengan **nama port** di Service (mis. `metrics`)
- `path: /metrics`

### 8.1 Bukti target exporter UP
Buka Prometheus → **Status → Targets**, cari `nginx` lalu pastikan `nginx-exporter` statusnya **UP**.

Query validasi:
```promql
nginx_up
```
atau cek semua metrik nginx:
```promql
{__name__=~"nginx_.*"}
```

---

## 9. Membuat Dashboard Grafana Kustom

Kamu sudah punya panel CPU & Memory. Untuk melengkapi sesuai instruksi, tambahkan panel untuk:
- **Disk Usage**
- **Network Traffic**
- (Opsional) **Request Rate (NGINX)** bila metrik nginx tersedia

### 9.1 Query rekomendasi (Node Exporter)

**Disk Usage (%)** (gunakan mount root `/`)
```promql
100 - (node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"} 
/ node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"} * 100)
```

**Network Traffic (Receive bytes/s)**
```promql
sum by (instance) (rate(node_network_receive_bytes_total{device!~"lo"}[5m]))
```

**Network Traffic (Transmit bytes/s)**
```promql
sum by (instance) (rate(node_network_transmit_bytes_total{device!~"lo"}[5m]))
```

> Tips: untuk Network, kamu bisa buat 2 query dalam 1 panel (RX & TX) atau 2 panel terpisah.

### 9.2 (Opsional) Query untuk NGINX Request Rate
Jika exporter sudah benar-benar mengirim metrik request:
```promql
sum(rate(nginx_http_requests_total[1m]))
```

Jika masih `No data`, kemungkinan metrik request belum tersedia dari konfigurasi NGINX/exporter. Yang penting untuk deliverable adalah mekanisme ServiceMonitor + bukti metrik exporter muncul.

---

## 10. Import Dashboard JSON (File Sudah Ada di Repo)

### 10.1 Import lewat UI Grafana
1. Buka Grafana → **Dashboards → New → Import**
2. Klik **Upload JSON file**
3. Pilih file JSON dari repo ini, misalnya:
   - `dashboards/custom-node-monitoring.json` *(sesuaikan dengan nama file di repo kamu)*
4. Pilih datasource: **Prometheus**
5. Klik **Import**

### 10.2 Export Dashboard JSON (Deliverable)
Untuk export:
1. Masuk dashboard → klik ikon **Share** (panah) atau **Dashboard settings (gear)**
2. Pilih **JSON model**
3. Klik **Download JSON**

Simpan file hasil export ke folder `dashboards/` dan commit ke GitHub.

---

## 11. Bukti Hasil (Checklist)

- [ ] **Prometheus** dapat diakses (port-forward) dan menampilkan Targets
- [ ] Targets **node-exporter** dan **kube-state-metrics** status **UP**
- [ ] **Grafana** dapat diakses (port-forward) dan login berhasil
- [ ] Dashboard kustom berisi panel:
  - [ ] CPU Usage
  - [ ] Memory Usage
  - [ ] Disk Usage
  - [ ] Network Traffic
- [ ] (Opsional) Target **nginx-exporter** UP dan query `{__name__=~"nginx_.*"}` menampilkan metrik
- [ ] **File JSON export** dashboard tersedia di repo (folder `dashboards/`)

---

## 12. Troubleshooting Cepat

### 12.1 Port-forward “address already in use”
Gunakan port lokal lain:
- Prometheus: `9091:9090`
- Grafana: `3001:80`

### 12.2 Pod Prometheus Pending
Cek storage class / PV:
```bash
kubectl get pvc -n proyek1-monitoring
kubectl describe pvc <nama-pvc> -n proyek1-monitoring
```

### 12.3 Query Grafana “No data”
- Pastikan datasource Grafana = Prometheus yang benar
- Pastikan rentang waktu di kanan atas (mis. **Last 1 hour**)
- Uji query di Prometheus UI dahulu

---

## 13. Struktur Repo (Rekomendasi)

```
.
├── README.md
├── k8s/
│   ├── nginx-exporter.yaml
│   └── servicemonitor-nginx-exporter.yaml
└── dashboards/
    └── custom-node-monitoring.json
```

---

## Lisensi
Gunakan sesuai kebutuhan pembelajaran/Studi Independen.
