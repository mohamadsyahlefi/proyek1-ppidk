# Proyek 1 — Implementasi Stack Observability (Prometheus + Grafana) di Kubernetes

Implementasi ini membangun **stack observability** menggunakan **kube-prometheus-stack** (Prometheus Operator) untuk memantau **cluster Kubernetes** dan **aplikasi** secara proaktif, serta membuat **dashboard Grafana kustom**.

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

## 3. Menyiapkan Kubernetes

### Install kind
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind create cluster --name observability
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

```bash
kubectl -n proyek1-monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
```
Buka: **http://localhost:9090**

### 5.2 Port-forward Grafana

```bash
kubectl -n proyek1-monitoring port-forward svc/monitoring-grafana 3000:80
```
Buka: **http://localhost:3000**

### 5.3 Ambil kredensial Grafana
```bash
kubectl -n proyek1-monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```
Login ke Grafana menggunakan user: admin dan pass hasil decode tersebut.

---

## 6. Verifikasi Metrik Cluster (Node & Pod)

### 6.1 Cek Targets di Prometheus
Buka Prometheus: **Status → Targets** lalu pastikan komponen target **UP**

---


## 7. Membuat Dashboard Grafana Kustom
Import File .JSON tersebut

## 8. Import lewat UI Grafana
1. Buka Grafana → **Dashboards → New → Import**
2. Klik **Upload JSON file**
3. Pilih file JSON dari repo ini, misalnya:
   - `dashboards/custom-node-monitoring.json`
4. Pilih datasource: **Prometheus**
5. Klik **Import**

### 8.2 Export Dashboard JSON (Deliverable)
Untuk export:
1. Masuk dashboard → klik ikon **Share** (panah) atau **Dashboard settings (gear)**
2. Pilih **JSON model**
3. Klik **Download JSON**

Simpan file hasil export ke folder `dashboards/` dan commit ke GitHub.

## Lisensi
Gunakan sesuai kebutuhan pembelajaran/Studi Independen.
