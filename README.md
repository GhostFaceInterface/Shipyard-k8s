# SynergyChat Kubernetes Öğrenme Projesi

Bu repo, Kubernetes çalışma mantığını pratik ederek öğrenmek amacıyla hazırlanmış bir çok‑bileşenli uygulama örneğidir. Manifester; Deployment, Service, Ingress, ConfigMap, PersistentVolumeClaim (PVC), HorizontalPodAutoscaler (HPA) ve çok konteynerli Pod gibi temel kavramları bir arada gösterir. Amaç, gerçekçi bir senaryoda bu kaynakların nasıl birlikte çalıştığını deneyimlemek ve paylaşılabilir bir örnek ortaya koymaktır.

## Mimari Genel Bakış
- Web: Kullanıcı arayüzü. ConfigMap ile yapılandırılır, Service ile küme içinde yayınlanır, HPA ile ölçeklenir.
- API: İş mantığı. PVC ile kalıcı depolama kullanır, NodePort Service ile erişilebilir, Crawler’a servis DNS’iyle ulaşır.
- Crawler: Ayrı bir namespace altında çalışan ve aynı Pod içinde üç konteyner barındıran tarayıcı bileşeni (paylaşımlı cache ile).
- Ingress: Web ve API trafiğini host tabanlı kurallarla yönlendirir.
- Deney Pod’ları: CPU ve bellek tüketimi davranışını test etmek için küçük yardımcı iş yükleri.

Basit akış: Web arayüzü API’ye `synchatapi.internal` üzerinden ulaşır. API, Crawler’ı küme içi DNS ile çağırır (`crawler-service.crawler.svc.cluster.local`). API verileri PVC ile `/persist` altında saklar.

## Kaynaklar ve Dosyalar
- Web
  - Deployment: `web-deployment.yaml`
  - Service: `web-service.yaml`
  - ConfigMap: `web-configmap.yaml`
  - HPA: `web-hpa.yaml`
- API
  - Deployment: `api-deployment.yaml`
  - Service (NodePort 30080): `api-service.yaml`
  - ConfigMap: `api-configmap.yaml`
  - PVC (1Gi): `api-pvc.yaml`
- Crawler (namespace: `crawler`)
  - Deployment (3 konteyner: 8080/8081/8082): `crawler-deployment.yaml`
  - Service (80 → 8080): `crawler-service.yaml`
  - ConfigMap: `crawler-configmap.yaml`
- Ingress
  - Hostlar: `synchat.internal` (web), `synchatapi.internal` (api)
  - Manifest: `synergychat-ingress.yaml`
- Deney Pod’ları
  - CPU test: `testcpu-deployment.yaml`, `testcpu-hpa.yaml`
  - RAM test: `testram-deployment.yaml`, `testram-configmap.yaml`

## Önemli Yapılandırmalar
- API kalıcı depolama: PVC, `/persist` altına bağlanır. `API_DB_FILEPATH=/persist/db.json`.
- API → Crawler: `CRAWLER_BASE_URL=http://crawler-service.crawler.svc.cluster.local`.
- Web → API: `API_URL="http://synchatapi.internal"`.
- Crawler Pod’u: 3 konteyner aynı Pod’da, paylaşımlı `emptyDir` cache (`/cache`).

## Önkoşullar
- `kubectl` kurulumu ve erişilebilir bir Kubernetes kümesi (Minikube, Kind, k3s veya bulut).
- Ingress Controller (ör. NGINX Ingress Controller).
- Metrics Server (HPA metrikleri için).

## Kurulum
1) Namespace (crawler):
```bash
kubectl create ns crawler
```
2) ConfigMap’ler:
```bash
kubectl apply -f crawler-configmap.yaml -n crawler
kubectl apply -f api-configmap.yaml
kubectl apply -f web-configmap.yaml
kubectl apply -f testram-configmap.yaml
```
3) Depolama (API için PVC):
```bash
kubectl apply -f api-pvc.yaml
```
4) Crawler (namespace ile):
```bash
kubectl apply -f crawler-deployment.yaml -n crawler
kubectl apply -f crawler-service.yaml -n crawler
```
5) API:
```bash
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml
```
6) Web:
```bash
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
```
7) Ingress:
```bash
kubectl apply -f synergychat-ingress.yaml
```

## Erişim
- Ingress ile erişim için `/etc/hosts` dosyanıza küme giriş IP’sini şu hostlarla eşleyin:
  - `synchat.internal`
  - `synchatapi.internal`
- Web: `http://synchat.internal`
- API: `http://synchatapi.internal`
- Alternatif olarak API NodePort:
  - `http://<NodeIP>:30080`

## Ölçekleme ve Metrikler
- Web HPA: 1–4 replika, hedef CPU kullanımı %50 (`web-hpa.yaml`).
- Test CPU HPA: 1 replika ile sabit, metrik gözlem amaçlı (`testcpu-hpa.yaml`).
- HPA’nın çalışması için Metrics Server gereklidir.

## Kaynak İstekleri ve Limitler
- `testram`: Bellek sınırı ve isteği 4000Mi, CPU limiti 500m. `MEGABYTES` değeriyle kullanım davranışı ayarlanır (`testram-configmap.yaml`).
- `testcpu`: CPU limiti 10m (yük altında CPU metriklerini gözlemlemek için minimal bir örnek).

## Temizlik
- Tek tek dosyalar:
```bash
kubectl delete -f <dosya>.yaml
```
- Crawler namespace’i:
```bash
kubectl delete ns crawler
```

## Öğrenme Notları
Bu çalışma kapsamında aşağıdaki Kubernetes kavramlarını uygulamalı olarak denedim:
- Deployment ve çok konteynerli Pod düzenleri
- Service türleri: ClusterIP ve NodePort
- Ingress ile host tabanlı yönlendirme
- ConfigMap ile ortam değişkeni yönetimi
- PersistentVolumeClaim ile kalıcı depolama bağlama
- HorizontalPodAutoscaler ve Metrics Server ile otomatik ölçekleme
- Namespace izolasyonu ve küme içi DNS keşfi

## Yol Haritası (Gelecek İyileştirmeler)
- Health/Liveness/Readiness probelarını eklemek
- Kaynak istek/limitlerini gözlemlere göre ayarlamak
- Crawler için ayrı Service/Endpoint stratejileri denemek (örn. Headless Service)
- CI/CD boru hattı ile otomatik uygulama

---
Bu repo, Kubernetes’e giriş ve pratik yapmak isteyenler için örnek bir başlangıç noktasıdır. Geliştirme önerilerine ve geribildirimlere açığım.

