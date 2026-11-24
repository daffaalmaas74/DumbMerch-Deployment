# DumbMerch Deployment

Repository ini berisi konfigurasi infrastructure as code dan CI/CD untuk deployment aplikasi DumbMerch. Infrastruktur dibuat di Microsoft Azure menggunakan Terraform, konfigurasi server dilakukan dengan Ansible, aplikasi dijalankan melalui Docker atau K3s, dan proses build/deploy dikelola oleh Jenkins.

## Arsitektur

- **Terraform** menyediakan Resource Group, Virtual Network, subnet, Network Security Group, Public IP, VM Ubuntu 22.04, dan managed disk di Azure.
- **Ansible** menginstal Docker, Jenkins, Jenkins dependencies, K3s, PostgreSQL, private Docker Registry, SonarQube, Prometheus, Grafana, Node Exporter, cAdvisor, Nginx, dan Certbot.
- **Jenkins** melakukan quality analysis, quality gate, build Docker image, vulnerability scan dengan Trivy, push image ke private registry, deployment, dan smoke test.
- **Staging** menggunakan Docker Compose pada appserver dan server lain.
- **Production** menggunakan cluster K3s pada master dan worker nodes.
- **Gateway** menyediakan reverse proxy HTTPS dan load balancing staging.
- **Monitoring** menggunakan Prometheus, Grafana, Node Exporter, dan cAdvisor.

## Struktur Direktori

```text
DumbMerch-Deployment/
├── automation/
│   ├── Terraform/
│   │   ├── server/
│   │   │   ├── main.tf             # Resource Azure untuk VM layanan dan staging
│   │   │   ├── providers.tf        # Konfigurasi provider Terraform
│   │   │   └── variables.tf        # Variabel server, region, dan disk
│   │   └── kubernetes-server/
│   │       ├── main.tf             # Resource Azure untuk VM master dan worker K3s
│   │       ├── providers.tf        # Konfigurasi provider Terraform
│   │       └── variables.tf        # Variabel cluster, jaringan, VM, dan disk
│   └── Ansible/
│       ├── Inventory                       # Daftar host dan grup server Ansible
│       ├── ansible.cfg                     # Konfigurasi default Ansible
│       ├── create-user.yaml                # Membuat user dan mengatur autentikasi/port SSH
│       ├── deploy-be-dumbmerch-staging.yaml    # Deploy backend staging dengan Docker Compose
│       ├── deploy-be-dumbmerch-production.yaml # Deploy backend production ke K3s
│       ├── deploy-fe-dumbmerch-staging.yaml    # Deploy frontend staging dengan Docker Compose
│       ├── deploy-fe-dumbmerch-production.yaml # Deploy frontend production ke K3s
│       ├── install-cadvisor.yaml           # Memasang cAdvisor untuk metrik container
│       ├── install-docker-registry-private.yaml # Memasang private Docker Registry
│       ├── install-docker.yaml              # Memasang dan mengaktifkan Docker
│       ├── install-grafana.yaml             # Memasang Grafana
│       ├── install-jenkins.yaml             # Memasang Jenkins
│       ├── install-kubernetes.yaml          # Memasang K3s server dan agent
│       ├── install-node-exporter.yaml       # Memasang Node Exporter untuk metrik host
│       ├── install-postgresql-k3s.yaml      # Menjalankan PostgreSQL di cluster K3s
│       ├── install-postgresql.yaml          # Menjalankan PostgreSQL dengan Docker di server database
│       ├── install-prometheus.yaml          # Memasang Prometheus dan konfigurasi monitoring
│       ├── install-sonarqube.yaml           # Memasang SonarQube
│       ├── renew-certificates.yaml          # Menjadwalkan perpanjangan sertifikat SSL
│       ├── setup-ingress-certmanager-ssl.yaml # Mengatur Ingress NGINX, Cert-Manager, dan SSL di K3s
│       └── setup-nginx-certbot-ssl.yaml     # Mengatur Nginx reverse proxy dan SSL untuk server gateway
├── staging/
│   ├── be-dumbmerch/
│   │   └── Jenkinsfile             # Pipeline CI/CD backend staging ke server staging
│   └── fe-dumbmerch/
│       └── Jenkinsfile             # Pipeline CI/CD frontend staging ke server staging
└── production/
    ├── be-dumbmerch/
    │   └── Jenkinsfile             # Pipeline CI/CD backend production ke cluster K3s
    └── fe-dumbmerch/
        └── Jenkinsfile             # Pipeline CI/CD frontend production ke cluster K3s
```

## Infrastruktur Server

Konfigurasi VM menggunakan Ubuntu 22.04 LTS, ukuran `Standard_B2als_v2`, dan managed disk 10 GB.

### VM layanan

| Host | Lokasi | Fungsi |
|---|---|---|
| `appserver` | West US 2 | Aplikasi staging frontend/backend |
| `gateway` | West US 2 | Nginx reverse proxy, load balancer, dan SSL |
| `database` | East Asia | Database staging |
| `monitoring` | East Asia | Prometheus dan Grafana |
| `jenkins` | Korea Central | CI/CD |
| `additional-depedencies` | Korea Central | Private Docker Registry dan SonarQube |

### VM Kubernetes

| Host | Lokasi | Fungsi |
|---|---|---|
| `master` | Australia East | K3s control plane dan deployment aplikasi production |
| `worker-1` | Australia East | K3s worker node |
| `worker-2` | Japan East | K3s worker node |
