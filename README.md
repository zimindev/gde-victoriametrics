## 🚀 **VictoriaMetrics Setup & Configuration Guide**  

### ✅ **What is VictoriaMetrics?**  
**VictoriaMetrics** is a **high-performance**, **open-source** time-series database (TSDB) and monitoring solution compatible with **Prometheus**.  
✔ **Faster & more efficient** than vanilla Prometheus  
✔ **Low resource usage** (CPU/RAM/disk)  
✔ **Long-term storage** with downsampling  
✔ **Supports PromQL, MetricsQL, and Graphite queries**  

Ideal for:  
- Large-scale Prometheus deployments  
- Cost-effective long-term metric storage  
- High-cardinality data (e.g., Kubernetes, microservices)  

---

## 🛠️ **Step 1: Install VictoriaMetrics**  

### **Linux (Standalone Binary)**  
```bash
# Download and run
wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.93.4/victoria-metrics-linux-amd64-v1.93.4.tar.gz
tar -xzf victoria-metrics-*.tar.gz
./victoria-metrics-prod -storageDataPath=/var/lib/victoria-metrics-data -retentionPeriod=12m
```
*Flags:*  
- `-storageDataPath`: Where to store data (default: `./vm-data`)  
- `-retentionPeriod`: Retention in months (`12m` = 1 year)  

### **Docker**  
```bash
docker run -d -p 8428:8428 -v vm-data:/storage victoriametrics/victoria-metrics
```

### **Kubernetes (Helm)**  
```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm install vm-single vm/victoria-metrics-single
```

---

## ⚙️ **Step 2: Configure Data Sources**  

### **Option 1: Replace Prometheus**  
Point VictoriaMetrics to scrape targets directly (edit `prometheus.yml`-equivalent):  
```yaml
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100"]
```
Start VictoriaMetrics with:  
```bash
./victoria-metrics-prod -promscrape.config=./scrape.yml
```

### **Option 2: Remote Write from Prometheus**  
Add to Prometheus config (`prometheus.yml`):  
```yaml
remote_write:
  - url: http://victoriametrics:8428/api/v1/write
```
Restart Prometheus:  
```bash
sudo systemctl restart prometheus
```

---

## 📊 **Step 3: Query Data**  

### **Web UI**  
Access at `http://localhost:8428/vmui`:  
- Run **PromQL/MetricsQL** queries  
- Visualize metrics with built-in graphs  

### **Grafana Integration**  
1. Add VictoriaMetrics as a **Prometheus-type** data source  
2. URL: `http://localhost:8428/` (or your VM instance)  

Example query:  
```promql
sum(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)
```

---

## 🔔 **Step 4: Set Up Alerts (vmalert)**  

### **Install vmalert**  
```bash
wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.93.4/vmalert-linux-amd64-v1.93.4.tar.gz
tar -xzf vmalert-*.tar.gz
```

### **Configure Alerts**  
Create `alerts.yml`:  
```yaml
groups:
  - name: example
    rules:
      - alert: HighCPU
        expr: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
```

### **Run vmalert**  
```bash
./vmalert -rule=./alerts.yml -notifier.url=http://alertmanager:9093 -datasource.url=http://victoriametrics:8428
```
*Replace `alertmanager:9093` with your Alertmanager URL.*  

---

## 🔧 **Step 5: Advanced Features**  

### **Cluster Mode**  
Deploy with **multiple components** for scalability:  
- **vmstorage**: Stores data  
- **vminsert**: Handles writes  
- **vmselect**: Handles queries  

Example (docker-compose.yml):  
```yaml
services:
  vmstorage:
    image: victoriametrics/vmstorage
    command: -retentionPeriod=12m
  vminsert:
    image: victoriametrics/vminsert
    command: -storageNode=vmstorage:8400
  vmselect:
    image: victoriametrics/vmselect
    command: -storageNode=vmstorage:8401
```

### **Downsampling**  
Reduce storage usage for long-term metrics:  
```bash
./victoria-metrics-prod -downsampling.period=30d -downsampling.offset=1d
```

### **Backup/Restore**  
```bash
# Backup
./victoria-metrics-prod -snapshot.createURL=http://localhost:8428/snapshot/create

# Restore
./victoria-metrics-prod -storageDataPath=/new/data -snapshot=/path/to/snapshot
```

---

## 📚 **Cheat Sheet**  

| Task                      | Command/Query                     |  
|--------------------------|----------------------------------|  
| List metrics             | `curl http://localhost:8428/api/v1/labels` |  
| Delete metrics           | `curl -X POST http://localhost:8428/api/v1/admin/tsdb/delete_series -d '{"match[]":"{job=prometheus}"}'` |  
| Check health             | `curl http://localhost:8428/health` |  

---

## 🚀 **Pro Tips**  
- **Use `-search.maxQueryLen`** to limit heavy queries.  
- **Enable `-httpAuth.*` flags** for basic authentication.  
- **Combine with Grafana** for dashboards (use `-http.pathPrefix=/victoria` if behind a proxy).  

---

Now you have a **blazing-fast Prometheus alternative**! 🎯  
- **For small setups**: Use standalone mode.  
- **For large-scale**: Deploy cluster mode + vmalert.  
- **For long-term storage**: Enable downsampling.
