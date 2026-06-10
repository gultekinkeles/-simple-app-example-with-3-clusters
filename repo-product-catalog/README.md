# Product Catalog Repository

Bu repository, Product Catalog uygulamasının Kubernetes (OpenShift) yapılandırmalarını içerir. Üç katmanlı bir mimari ile Database, Server (Backend) ve Client (Frontend) bileşenlerinden oluşur.

## Klasör Yapısı

```
repo-product-catalog/
├── components/                          # Ortak Base Bileşenler
│   ├── client/base/                    # Frontend bileşeni
│   │   ├── kustomization.yaml
│   │   ├── client-deployment.yaml      # Client deployment (replicas temel olarak 1)
│   │   ├── client-route.yaml           # OpenShift Route - HTTPS edge termination
│   │   ├── client-service.yaml         # ClusterIP Service
│   │   └── config/
│   │       └── config.js               # Client konfigürasyonu
│   │
│   ├── server/base/                    # Backend bileşeni
│   │   ├── kustomization.yaml
│   │   ├── server-deployment.yaml      # Server deployment (replicas temel olarak 1)
│   │   ├── server-route.yaml           # OpenShift Route - HTTPS edge termination
│   │   ├── server-service.yaml         # ClusterIP Service
│   │   ├── default-view-rolebinding.yaml # RBAC configuration
│   │   └── config/
│   │       └── application.properties  # Server konfigürasyonu
│   │
│   └── database/base/                  # Database bileşeni
│       ├── kustomization.yaml
│       ├── db-deployment.yaml          # PostgreSQL deployment (replicas temel olarak 1)
│       ├── db-service.yaml             # Headless Service
│       ├── db-pvc.yaml                 # PersistentVolumeClaim (1Gi default)
│       ├── db-secret.yaml              # Database credentials
│       └── config/
│           ├── 90-init-database.sh     # Database initialization script
│           ├── import.sql              # Data import
│           └── schema.sql              # Database schema
│
└── overlays/                           # Ortam-Spesifik Yapılandırmalar
    ├── test/                           # Test Ortamı
    │   ├── kustomization.yaml          # Test ayarları: 1 replika her bileşen
    │   └── patches/
    │       └── route-host.yaml         # Route host adresleri (*.test.example.com)
    │
    ├── pre-prod/                       # Pre-Production Ortamı
    │   └── kustomization.yaml          # Pre-prod ayarları: 2 replika, environment label
    │
    └── prod/                           # Production Ortamı
        ├── kustomization.yaml          # Prod ayarları: 4 replika, environment label
        └── patches/
            ├── db-pvc-size.yaml        # PVC boyutlandırma (10Gi)
            └── secure-route.yaml       # Güvenli Route configuration

└── kustomization.yaml                  # Root kustomization (base deployment)
```

## Kustomization Dosyaları Açıklaması

### Base Kustomization (`components/*/base/kustomization.yaml`)

#### Client Base
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - client-service.yaml
  - client-route.yaml
  - client-deployment.yaml
```
- Client deployment, service ve route'u tanımlar
- ConfigMap generator (commented out) client config dosyasını inject edebilir

#### Server Base
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

generatorOptions:
  disableNameSuffixHash: true
  labels:
    app.kubernetes.io/part-of: product-catalog

configMapGenerator:
  - name: server
    files:
      - config/application.properties

resources:
  - default-view-rolebinding.yaml
  - server-service.yaml
  - server-route.yaml
  - server-deployment.yaml
```
- Server deployment, service, route ve RBAC konfigürasyonunu tanımlar
- application.properties dosyasını ConfigMap olarak oluşturur

#### Database Base
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

generatorOptions:
  disableNameSuffixHash: true
  labels:
    app.kubernetes.io/part-of: product-catalog

configMapGenerator:
- name: productdb-init
  files:
    - config/90-init-database.sh
    - config/import.sql
    - config/schema.sql

resources:
  - db-pvc.yaml
  - db-secret.yaml
  - db-service.yaml
  - db-deployment.yaml
```
- Database deployment, service, PVC ve Secret'i tanımlar
- Initialization scriptleri ve SQL dosyalarını ConfigMap olarak oluşturur

### Overlay Kustomizations

#### Test Ortamı (`overlays/test/kustomization.yaml`)
```yaml
resources:
  - ../../components/database/base
  - ../../components/client/base
  - ../../components/server/base

replicas:
  - name: server
    count: 1
  - name: client
    count: 1

patchesStrategicMerge:
  - patches/route-host.yaml
```
- Test ortamı için 1 replika kullanır
- Route hostları test.example.com üzerinden ayarlanır

#### Pre-Production Ortamı (`overlays/pre-prod/kustomization.yaml`)
```yaml
resources:
  - ../../components/database/base
  - ../../components/client/base
  - ../../components/server/base

commonLabels:
  environment: pre-prod

replicas:
  - name: server
    count: 2
  - name: client
    count: 2
```
- Pre-prod ortamı için 2 replika kullanır
- Tüm kaynaklar `environment: pre-prod` labeli alırlar

#### Production Ortamı (`overlays/prod/kustomization.yaml`)
```yaml
resources:
  - ../../components/database/base
  - ../../components/client/base
  - ../../components/server/base

commonLabels:
  environment: prod

replicas:
  - name: server
    count: 4
  - name: client
    count: 4

patchesStrategicMerge:
  - patches/db-pvc-size.yaml
  - patches/secure-route.yaml
```
- Production ortamı için 4 replika kullanır
- Database PVC boyutu 10Gi'ye artırılır
- Route yapılandırması güvenli hale getirilir (passthrough termination)

## Deployment

### Base Deployment (Geliştirme)
```bash
kubectl apply -k repo-product-catalog/
```

### Test Ortamı
```bash
kubectl apply -k repo-product-catalog/overlays/test/
```

### Pre-Production Ortamı
```bash
kubectl apply -k repo-product-catalog/overlays/pre-prod/
```

### Production Ortamı
```bash
kubectl apply -k repo-product-catalog/overlays/prod/
```

## Önemli Notlar

- **Database Persistence**: Database deployment PVC tarafından korunur
- **Configuration Management**: Tüm konfigürasyonlar ConfigMap ve Secret olarak yönetilir
- **TLS/SSL**: Routes edge termination ile HTTPS destekler
- **Replication**: Ortamlar arasında replika sayıları farklılık gösterir
- **Self-Healing**: Kubernetes deployment'ları replica sayısını otomatik olarak korur

## GitOps Prensipleri

Bu repository GitOps ilkelerine uyar:
1. **Tüm konfigürasyonlar Git'te** - Version control ve audit trail
2. **Self-Healing** - Replika sayısı otomatik olarak düzeltilir
3. **Drift Detection** - ArgoCD gibi tools ile çalışır
4. **Ortam Yönetimi** - Overlays ile ortamlar arasında geçiş yapılır
