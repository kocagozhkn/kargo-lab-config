# kargo-lab-config

Kargo öğrenme lab'i için GitOps config repo'su + **[kurulum rehberi](docs/SETUP.md)**.

macOS + Colima + kind üzerinde Kargo 1.11.2 ve Argo CD v3.5.1 ile uçtan uca
çalışan bir promotion pipeline'ı.

## Yapı

```
base/            nginx Deployment + Service
env/dev/         base -> nginx-dev namespace   | Argo CD app: kargo-lab-dev
env/prod/        base -> nginx-prod namespace  | Argo CD app: kargo-lab-prod
kargo/           cluster kaynakları (Kargo + Argo CD Application manifest'leri)
docs/SETUP.md    adım adım kurulum, doğrulama ve hata teşhisi rehberi
```

## Akış

- **Warehouse** `public.ecr.aws/nginx/nginx` imajını izler (`constraint:
  >=1.27.0 <1.29.0`, `interval: 1m`).
- **dev** Stage: yeni Freight geldiğinde **otomatik** promote edilir. Kargo
  `env/dev/kustomization.yaml` içindeki `images[].newTag` alanını günceller,
  commit'ler, `main`'e push eder ve Argo CD app'ini senkronlar.
- **prod** Stage: upstream `dev`. **Manuel** tetiklenir; Kargo değişikliği ayrı
  bir branch'e push eder, PR açar ve **PR merge edilene kadar bekler**. Merge
  sonrası Argo CD app'ini senkronlar.

> `env/*/kustomization.yaml` dosyalarındaki `newTag` değerlerini elle
> düzenlemeyin — onları Kargo yönetir.

## Hızlı başlangıç

Tam rehber: **[docs/SETUP.md](docs/SETUP.md)**

```bash
kubectl port-forward -n argocd svc/argocd-server 8081:80    # http://localhost:8081
kubectl port-forward -n kargo  svc/kargo-api     8082:443   # https://localhost:8082

kargo get stages --project kargo-lab
```

## ⚠️ Lab kimlik bilgileri

Bu repo'daki helm values dosyaları **lab amaçlıdır** (`admin/admin`, quickstart
`tokenSigningKey`). Üretimde kullanma — gerekçeleri
[docs/SETUP.md § 11](docs/SETUP.md#11-güvenlik-notları)'de.
