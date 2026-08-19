# kargo-lab-config

Kargo öğrenme lab'i için GitOps config repo'su.

```
base/            nginx Deployment + Service (image: public.ecr.aws/nginx/nginx:1.27.0)
env/dev/         base'i nginx-dev namespace'ine render eder  -> Argo CD app: kargo-lab-dev
env/prod/        base'i nginx-prod namespace'ine render eder -> Argo CD app: kargo-lab-prod
```

## Akış

- **Warehouse** `public.ecr.aws/nginx/nginx` imajını izler (semverConstraint: `>=1.27.0 <1.29.0`).
- **dev** Stage: yeni Freight geldiğinde otomatik promote edilir. Kargo
  `env/dev/kustomization.yaml` içindeki `images[].newTag` alanını günceller,
  commit'ler, `main` branch'ine push eder ve Argo CD app'ini refresh/sync eder.
- **prod** Stage: upstream `dev`. Manuel promote tetiklenir; Kargo değişikliği
  ayrı bir target branch'e push eder, GitLab'da MR açar ve MR merge edilene
  kadar bekler. Merge sonrası Argo CD app'ini sync eder.

`env/*/kustomization.yaml` dosyalarındaki `newTag` değerlerini elle düzenlemeyin —
onları Kargo yönetir.
