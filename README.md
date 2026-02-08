# DevOps Case: MERN + ETL (Docker, Kubernetes, CI/CD ve Terraform)

Bu repo **iki proje** ve uçtan uca bir DevOps setup içeriyor:

- **Proje 1 — MERN stack**: React (client) + Node/Express (server) + MongoDB
- **Proje 2 — ETL (Python)**: Kubernetes üzerinde **CronJob** olarak çalışan containerize ETL işi
- **Altyapı (IaC)**: AWS kaynaklarını kurmak için Terraform (EKS + VPC) (lokal çalıştırma için opsiyonel)
- **CI/CD**: Docker image build + Kubernetes manifest doğrulama yapan GitHub Actions pipeline’ları


---

## Repo Yapısı

```
.
├── mern-project/
│   ├── client/                 # React app (+ Dockerfile, nginx.conf)
│   └── server/                 # Node/Express API (+ Dockerfile)
├── python-project/             # ETL job (+ Dockerfile)
├── k8s/                        # Kubernetes manifestleri (namespace, deployment, service, cronjob)
├── infra/terraform/            # Terraform (AWS EKS + VPC) - opsiyonel
├── .github/workflows/          # CI pipeline’ları (MERN + ETL)
├── docker-compose.yml          # Lokal MERN için Docker Compose
├── kind-config.yaml            # (Opsiyonel) kind cluster config
└── README.txt
```

---

## Mimari

### 1) MERN Stack (Web App)

**Bileşenler**
- **Client (React)**: Build alınıp statik dosyalar **Nginx** ile servis edilir
- **Server (Node/Express)**: `5050` portundan REST API yayınlar
- **MongoDB**: Kalıcı veri depolama

**İletişim**
- Client, Kubernetes içinde Service DNS ile Server’a istek atar
- Server, MongoDB’ye cluster-internal `mongo` servisi üzerinden bağlanır

### 2) ETL (Python CronJob)

- Kubernetes **CronJob** ile belirli aralıklarla çalışır
- akış: fetch → transform → load (ör. DB’ye ya da bir API’ye yazma)
- İstenirse manuel Job olarak tetiklenebilir

### Diyagram

```
            (Kullanıcı Tarayıcı)
                 |
              NodePort
                 |
          +----------------+
          |  client (nginx)|
          +----------------+
                 |
          ClusterIP Service
                 |
          +----------------+
          | server (API)   |
          +----------------+
                 |
          ClusterIP Service
                 |
          +----------------+
          | mongo (DB)     |
          +----------------+

     +-----------------------------+
     | ETL CronJob (Python)        |
     | schedule ile çalışır        |
     +-----------------------------+
```

---

## Dağıtım Süreci (Lokal)

Lokal için **iki seçenek** var:
1) **Docker Compose** (en hızlı lokal MERN)
2) **kind ile Kubernetes** 

### Seçenek A — Docker Compose (Lokal MERN)

Repo root’ta:

```bash
docker compose up --build
```

- Client: http://localhost:8080
- Server healthcheck: http://localhost:5050/healthcheck
- MongoDB: localhost:27017

Durdurmak:

```bash
docker compose down
```

### Seçenek B — kind ile Kubernetes (Lokal K8s)

#### 1) kind cluster oluşturulur

```bash
kind create cluster --name devopscase
```

Kontrol:

```bash
kubectl get nodes
```

#### 2) Namespace ve bileşenleri deploy edilir

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -n devopscase -f k8s/mongo.yaml
kubectl apply -n devopscase -f k8s/server.yaml
kubectl apply -n devopscase -f k8s/client.yaml
kubectl apply -n devopscase -f k8s/etl-cronjob.yaml
```

Doğrulama:

```bash
kubectl -n devopscase get all
kubectl -n devopscase get cronjob
```

#### 3) Client’a erişim

Client **NodePort** ile dışarı açılır:

```bash
kubectl -n devopscase get svc client
```

Tarayıcıdan:

- http://localhost:30080  (manifest’teki NodePort)

> NodePort farklıysa `kubectl get svc` çıktısındaki portu kullan.

#### 4) ETL manuel çalıştırma (opsiyonel)

CronJob’dan tek seferlik Job üret:

```bash
kubectl -n devopscase create job --from=cronjob/etl-hourly etl-manual-test
kubectl -n devopscase logs job/etl-manual-test
```

Cleanup:

```bash
kubectl -n devopscase delete job etl-manual-test
```

#### 5) Kapatma

```bash
kind delete cluster --name devopscase
```

---

## CI/CD (GitHub Actions)

İki workflow vardır:

- **CI - MERN (client + server)**: Docker image build (client/server) + K8s manifest doğrulama
- **CI - ETL (python cronjob)**: Docker image build (etl) + CronJob manifest doğrulama

### Neden cluster olmadan doğrulama?

GitHub Actions runner’larında default olarak Kubernetes API Server yoktur.  
Bu nedenle manifest doğrulaması `kubectl apply` ile gerçek cluster’a gitmek yerine **schema tabanlı doğrulama (kubeconform)** ile yapılır.

---

## Altyapı Kod (Terraform / AWS EKS)

> Lokal değerlendirme için **opsiyonel**. “IaC scripts” şartını sağlamak için eklenmiştir.


infra/terraform
```

### Init + validate (güvenli)

```bash
terraform init
terraform validate
```

### Plan (AWS credential ister)

```bash
terraform plan
```

### Apply (gerçek cloud kaynağı oluşturduğu için apply uygulanmadı)

```bash
terraform apply
```

> **Önemli:** AWS’te VPC/EKS oluşturur. AWS hesabı/region/credential ayarların doğru olmalı.

### Destroy (temizleme)

```bash
terraform destroy
```

---

## Güvenlik ve Best-Practice Notları

- **Containerization**: Her bileşen için ayrı Dockerfile; build’ler tekrarlanabilir
- **En az dışa açılım**:
  - MongoDB Kubernetes’te **ClusterIP** ile sadece içeride açık
  - Client lokal erişim için **NodePort** ile dışa açık
- **Separation of concerns**: Manifestler bileşen bazlı ayrı
- **CI**: Push/PR’de otomatik build + doğrulama
- **Secret yok**:
  - Token’lar GitHub Secrets ile
  - Runtime config için env değişkenleri

---

## Karşılaşılan Zorluklar ve Çözümler

### 1) Windows’ta Node/npm sürüm & PATH karmaşası
- Admin PowerShell vs VSCode terminalinde farklı PATH ve Node sürümü görünüyordu.
- `nvm` ile sürüm standardize edilip doğru terminal oturumunda PATH düzeltildi.

### 2) Docker Desktop engine bağlantı hataları
- Docker engine kapalıyken veya context karışınca “docker API” hataları oluştu.
- Docker Desktop’ın çalıştığı ve doğru Linux engine kullanıldığı doğrulandı.

### 3) kind cluster oluşturma / Node Ready sorunları
- kubelet/cgroups/network hazır değilken kind init hata verebiliyor.
- Docker Desktop/WSL2 stabilize olduktan sonra tekrar denenip `kubectl get nodes/pods` ile doğrulandı.

### 4) GitHub Actions’ta kubectl dry-run hatası (cluster yok)
- `kubectl apply --dry-run=client` OpenAPI şemasını `localhost:8080`’dan çekmeye çalışıyordu.
- CI doğrulaması **kubeconform** ile değiştirildi (cluster gerektirmez).

---


## Lisans

Case/interview değerlendirme amaçlıdır.

================================================================================

# DevOps Case: MERN + ETL with Docker, Kubernetes, CI/CD and Terraform

This repository contains **two projects** and an end-to-end DevOps setup:

- **Project 1 — MERN stack**: React (client) + Node/Express (server) + MongoDB
- **Project 2 — ETL (Python)**: A containerized ETL job running as a **Kubernetes CronJob**
- **Infrastructure (IaC)**: Terraform code to provision AWS resources (EKS + VPC) (optional for local run)
- **CI/CD**: GitHub Actions pipelines that build Docker images and validate Kubernetes manifests


---

## Repository Structure

```
.
├── mern-project/
│   ├── client/                 # React app (+ Dockerfile, nginx.conf)
│   └── server/                 # Node/Express API (+ Dockerfile)
├── python-project/             # ETL job (+ Dockerfile)
├── k8s/                        # Kubernetes manifests (namespace, deployments, services, cronjob)
├── infra/terraform/            # Terraform (AWS EKS + VPC) - optional
├── .github/workflows/          # CI pipelines (MERN + ETL)
├── docker-compose.yml          # Local Docker Compose for MERN stack
├── kind-config.yaml            # (Optional) kind cluster config
└── README.txt
```

---

## Architecture

### 1) MERN Stack (Web App)

**Components**
- **Client (React)**: Built into static files and served via **Nginx**
- **Server (Node/Express)**: Exposes REST API on port `5050`
- **MongoDB**: Database for persistence

**Communication**
- Client calls Server over internal Kubernetes networking (Service DNS)
- Server talks to MongoDB via cluster-internal service (`mongo`)

### 2) ETL (Python CronJob)

- Runs on a schedule via **Kubernetes CronJob**
- Typically: fetch → transform → load (e.g., to DB or external API)
- Can also be triggered manually (Job)

### High-Level Diagram 

```
            (User Browser)
                 |
              NodePort
                 |
          +----------------+
          |  client (nginx)|
          +----------------+
                 |
          ClusterIP Service
                 |
          +----------------+
          | server (API)   |
          +----------------+
                 |
          ClusterIP Service
                 |
          +----------------+
          | mongo (DB)     |
          +----------------+

     +-----------------------------+
     | ETL CronJob (Python)        |
     | runs on schedule (hourly)   |
     +-----------------------------+
```

---

## Deployment Process (Local)

There are **two local deployment options**:
1) **Docker Compose** (fastest for local MERN)
2) **Kubernetes via kind** (matches production-style deployment)

### Option A — Docker Compose (Local MERN)

From repo root:

```bash
docker compose up --build
```

- Client: http://localhost:8080
- Server healthcheck: http://localhost:5050/healthcheck
- MongoDB: localhost:27017

Stop:

```bash
docker compose down
```

### Option B — Kubernetes with kind (Local K8s)

#### 1) Create a kind cluster

```bash
kind create cluster --name devopscase
```

Check:

```bash
kubectl get nodes
```

#### 2) Create namespace and deploy workloads

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -n devopscase -f k8s/mongo.yaml
kubectl apply -n devopscase -f k8s/server.yaml
kubectl apply -n devopscase -f k8s/client.yaml
kubectl apply -n devopscase -f k8s/etl-cronjob.yaml
```

Verify:

```bash
kubectl -n devopscase get all
kubectl -n devopscase get cronjob
```

#### 3) Access the client

The client is exposed as **NodePort**. Example output:

```bash
kubectl -n devopscase get svc client
```

Open in browser:

- http://localhost:30080  (NodePort used in manifests)

> If the NodePort differs, use the one shown by `kubectl get svc`.

#### 4) ETL manual run (optional)

Create a one-off Job from CronJob:

```bash
kubectl -n devopscase create job --from=cronjob/etl-hourly etl-manual-test
kubectl -n devopscase logs job/etl-manual-test
```

Cleanup the job:

```bash
kubectl -n devopscase delete job etl-manual-test
```

#### 5) Tear down

```bash
kind delete cluster --name devopscase
```

---

## CI/CD (GitHub Actions)

Two workflows exist:

- **CI - MERN (client + server)**: builds Docker images (client/server) + validates K8s manifests
- **CI - ETL (python cronjob)**: builds Docker image (etl) + validates CronJob manifest

### Why manifest validation is done without a cluster

GitHub Actions runners do not have a Kubernetes API server by default.  
Therefore the pipeline validates manifests using **schema-based validation (kubeconform)** instead of running `kubectl apply` against a real cluster.

This keeps CI fast and reliable while still catching invalid YAML/schema issues.

---

## Infrastructure as Code (Terraform / AWS EKS)

> This part is **optional for local evaluation**. It is included to satisfy the “IaC scripts” requirement.


infra/terraform
```

### Initialize + validate (safe)

```bash
terraform init
terraform validate
```

### Plan (requires AWS credentials)

```bash
terraform plan
```

### Apply (creates real cloud resources, so it has not created)

```bash
terraform apply
```

> **Important:** Applying will create AWS resources (VPC/EKS). Ensure your AWS account, region, and credentials are configured properly.

### Destroy (cleanup)

```bash
terraform destroy
```

---

## Security & Best Practices Notes

- **Containerization**: Each component has its own Dockerfile. Builds are reproducible.
- **Least exposure**:
  - MongoDB is exposed internally in Kubernetes via **ClusterIP** only.
  - Client is exposed externally via **NodePort** for local access.
- **Separation of concerns**: k8s manifests are separate per component.
- **CI**: builds and validates automatically on push/PR.
- **No secrets in Git**:
  - Use GitHub Secrets for tokens if pushing images.
  - Use environment variables for runtime configuration.

---

## Challenges Encountered (and How They Were Solved)

### 1) Node/npm version & PATH issues on Windows
- Different shells (Admin PowerShell vs VSCode terminal) had inconsistent PATH and Node versions.
- Resolved by standardizing Node version (via `nvm`) and ensuring correct PATH in the active terminal session.

### 2) Docker Desktop engine connection issues
- Initial “failed to connect to the docker API” errors occurred when Docker Desktop engine was not running or context mismatch.
- Resolved by verifying Docker Desktop is running and using the correct Linux engine.

### 3) kind cluster creation failures / node not ready
- kind sometimes failed when kubelet/cgroups networking was not ready.
- Resolved by retrying after Docker Desktop/WSL2 stabilized; verified cluster health via `kubectl get nodes` and system pods.

### 4) GitHub Actions kubectl validation failures (no cluster)
- `kubectl apply --dry-run=client` attempted to fetch OpenAPI schema from `localhost:8080`.
- Resolved by switching CI validation to **kubeconform**, which works without a Kubernetes API server.



## License

For case/interview evaluation purposes.
