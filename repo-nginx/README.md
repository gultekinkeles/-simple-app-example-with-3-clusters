# Nginx Web Server Repository

Bu repository, Nginx web server'ının Kubernetes (OpenShift) yapılandırmalarını içerir. Statik içerik sunmak için PersistentVolume kullanır ve her ortam için özelleştirilmiş index.html dosyaları ConfigMap aracılığıyla sağlanır.

## Klasör Yapısı

```
repo-nginx/
├── components/                         # Ortak Base Bileşenler
│   └── nginx/base/                    # Nginx web server bileşeni
│       ├── kustomization.yaml         # Base kustomization
│       ├── nginx-deployment.yaml      # Nginx deployment (replicas temel olarak 1)
│       ├── nginx-service.yaml         # ClusterIP Service (port 80)
│       ├── nginx-pvc.yaml             # PersistentVolumeClaim (1Gi default)
│       └── ...
│
└── overlays/                          # Ortam-Spesifik Yapılandırmalar
    ├── test/                          # Test Ortamı
    │   ├── kustomization.yaml         # Test ayarları: 1 replika
    │   │                              # ConfigMap: "Hello World from TEST Cluster"
    │   └── patches/
    │
    ├── pre-prod/                      # Pre-Production Ortamı
    │   ├── kustomization.yaml         # Pre-prod ayarları: 2 replika
    │   │                              # ConfigMap: "Hello World from PRE-PROD Cluster"
    │   └── patches/
    │
    └── prod/                          # Production Ortamı
        ├── kustomization.yaml         # Prod ayarları: 3 replika
        │                              # ConfigMap: "Hello World from PROD Cluster"
        └── patches/

└── kustomization.yaml                 # Root kustomization (base deployment)
```

## Kustomization Dosyaları Açıklaması

### Base Kustomization (`components/nginx/base/kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - nginx-deployment.yaml
  - nginx-service.yaml
  - nginx-pvc.yaml

configMapGenerator:
  - name: index-html
    literals:
      - index.html=<html><body><h1>Hello World from Nginx</h1></body></html>
```

**Açıklama:**
- Nginx deployment, service ve PVC'yi tanımlar
- ConfigMap oluşturarak index.html içeriğini sağlar
- Temel "Hello World" sayfası default olarak sunulur

### Nginx Deployment (`components/nginx/base/nginx-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        - name: index-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: nginx-html
      - name: index-config
        configMap:
          name: index-html
```

**Açıklama:**
- Nginx container'ı port 80'de çalışır
- **html** volume: PersistentVolumeClaim ile uzun vadeli depolama
- **index-config** volume: ConfigMap'ten index.html inject edilir
- ConfigMap'teki index.html dosyası PVC'nin üzerine yazılır

### Nginx Service (`components/nginx/base/nginx-service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: http
  type: ClusterIP
```

**Açıklama:**
- ClusterIP tipi Service
- Port 80'de erişilir
- Cluster içindeki diğer pod'lar tarafından erişilebilir

### Nginx PVC (`components/nginx/base/nginx-pvc.yaml`)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-html
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**Açıklama:**
- 1Gi boyutunda PersistentVolumeClaim
- ReadWriteOnce erişim modu (tek pod yazabilir)
- Static dosyaları depolamak için kullanılır

### Overlay Kustomizations

#### Test Ortamı (`overlays/test/kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../components/nginx/base

replicas:
  - name: nginx
    count: 1

commonLabels:
  environment: test

configMapGenerator:
  - name: index-html
    literals:
      - index.html=<html><body><h1>Hello World from TEST Cluster</h1></body></html>
    behavior: replace
```

**Özellikleri:**
- **1 Replika**: Test ortamında tek instance çalışır
- **Custom Content**: "Hello World from TEST Cluster" mesajı gösterilir
- **Environment Label**: Kaynaklar `environment: test` etiketi alırlar

#### Pre-Production Ortamı (`overlays/pre-prod/kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../components/nginx/base

replicas:
  - name: nginx
    count: 2

commonLabels:
  environment: pre-prod

configMapGenerator:
  - name: index-html
    literals:
      - index.html=<html><body><h1>Hello World from PRE-PROD Cluster</h1></body></html>
    behavior: replace
```

**Özellikleri:**
- **2 Replika**: Yüksek kullanılabilirlik sağlanır
- **Custom Content**: "Hello World from PRE-PROD Cluster" mesajı gösterilir
- **Environment Label**: Kaynaklar `environment: pre-prod` etiketi alırlar

#### Production Ortamı (`overlays/prod/kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../components/nginx/base

replicas:
  - name: nginx
    count: 3

commonLabels:
  environment: prod

configMapGenerator:
  - name: index-html
    literals:
      - index.html=<html><body><h1>Hello World from PROD Cluster</h1></body></html>
    behavior: replace
```

**Özellikleri:**
- **3 Replika**: Maximum yüksek kullanılabilirlik
- **Custom Content**: "Hello World from PROD Cluster" mesajı gösterilir
- **Environment Label**: Kaynaklar `environment: prod` etiketi alırlar

## Deployment

### Base Deployment (Geliştirme)
```bash
kubectl apply -k repo-nginx/
```

### Test Ortamı (1 Replika)
```bash
kubectl apply -k repo-nginx/overlays/test/
```

### Pre-Production Ortamı (2 Replika)
```bash
kubectl apply -k repo-nginx/overlays/pre-prod/
```

### Production Ortamı (3 Replika)
```bash
kubectl apply -k repo-nginx/overlays/prod/
```

## İçerik Yönetimi

### Mevcut index.html Sayfası

Test ortamında:
```html
<html>
  <body>
    <h1>Hello World from TEST Cluster</h1>
  </body>
</html>
```

Pre-prod ortamında:
```html
<html>
  <body>
    <h1>Hello World from PRE-PROD Cluster</h1>
  </body>
</html>
```

Production ortamında:
```html
<html>
  <body>
    <h1>Hello World from PROD Cluster</h1>
  </body>
</html>
```

### İçerik Güncelleme

Ortam-spesifik içeriği güncellemek için ilgili overlay'in kustomization.yaml'ındaki configMapGenerator'ı değiştirin:

```yaml
configMapGenerator:
  - name: index-html
    literals:
      - index.html=<html><body>YENİ İÇERİK</body></html>
    behavior: replace
```

## Önemli Notlar

- **Replika Sayıları**: Test=1, Pre-prod=2, Prod=3
- **Persistent Storage**: PVC tarafından managed, container restart'ında korunur
- **ConfigMap Injection**: Her ortam için farklı index.html ConfigMap'ten inject edilir
- **subPath Kullanımı**: ConfigMap dosyasının yalnızca index.html kısmı mount edilir
- **Environment Labels**: Kaynakları ortamlarına göre filtrelemek için kullanılabilir

## Volume Davranışı

- **PVC (html)**: Persistent storage, Pod yeniden başlansa da korunur
- **ConfigMap (index-config)**: Her deployment sırasında güncellenebilir
- ConfigMap'teki index.html, PVC üzerine yazılarak istenen içerik sağlanır

## Cluster Tanımlama

Her ortam kendi cluster ismini index.html'de gösterir:
- TEST Cluster
- PRE-PROD Cluster  
- PROD Cluster

Bu sayede hangi cluster'dan hizmet aldığınız açıkça görülür.

## ArgoCD Entegrasyonu

Bu repository ArgoCD ile GitOps workflow'u için hazırdır:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-test
spec:
  source:
    repoURL: https://github.com/your-org/your-repo
    path: repo-nginx/overlays/test
  destination:
    server: https://test-cluster-api
    namespace: default
```
