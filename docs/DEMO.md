# Verification demo — prova edilmiş akış

Prova tarihi: 2026-08-27. Aşağıdaki her adım ve süre gerçek koşudan alındı.

## Hikâye

Bozuk bir config push'lanır. Kubernetes ve Argo CD açısından her şey yolundadır
(pod `2/2 Running`), ama gerçek sağlık sinyali aksini söyler. Kargo Freight'i
doğrulanmış saymaz ve **prod'a geçişi durdurur.**

Anlatılacak asıl ayrım: `Healthy=True` iken `Verified=False`.

## Verification ne kontrol ediyor

`AnalysisTemplate/smoke-check`, metrik `nginx-healthy`:

```
query:            min(nginx_up{namespace="nginx-dev"}) or vector(0)
successCondition: result[0] == 1
count: 3, interval: 15s, failureLimit: 1
```

- `min()` — rolling update sırasında eski ve yeni pod birlikte yaşıyor.
  Ham `nginx_up` iki seri döndürür ve `result[0]` rastgele birini seçerdi;
  `min()` hepsinin sağlıklı olmasını şart koşar. Kırılmayı yakalayan şey bu.
- `or vector(0)` — hiç pod yoksa `min()` boş vektör döner ve `result[0]`
  patlar; ölçüm temiz bir `Failed` yerine `Error` verirdi.

Prometheus `scrape_interval: 15s` (varsayılan 1m ile kırılma, AnalysisRun
bitmeden metriğe yansımıyordu).

## Başlangıç durumu kontrolü

```sh
kubectl -n kargo-lab get stages
# dev / prod  -> HEALTH=Healthy  READY=True  "Freight has been verified"
```

## 1) KIRMA

```sh
sed -i '' 's|8080/stub_status|9999/stub_status|' base/deployment.yaml
git commit -am "break metrics endpoint" && git push origin main
kubectl -n kargo-lab annotate warehouse nginx "kargo.akuity.io/refresh=$(date +%s)" --overwrite
```

Warehouse `includePaths: [base]` olduğu için bu commit yeni Freight üretir;
`refresh` annotation'ı 1 dakikalık poll beklemesini atlatır.

Gözlenen zaman çizgisi:

| Zaman | Durum |
|-------|-------|
| T+10s | promotion `Succeeded`, yeni pod doğuyor, eski ayakta, `min=1` |
| T+30s | rollout bitti, tek pod `2/2 Running`, **`min=0`** |
| T+50s | **`Verified=False`**, `Healthy=True` |

Sonuç:

```
dev   8785161  Healthy  READY=False  Metric "nginx-healthy" assessed Failed (2) > failureLimit (1)
prod  7c45060  Healthy  READY=True   Freight has been verified
```

Bozuk Freight'in `status.verifiedIn` alanı **boş** — prod'a geçmesi yapısal
olarak imkânsız. Pod ise `2/2 Running`, yani sorun ancak metrikten görülüyor.

## 2) GERİ ALMA

```sh
git revert --no-edit HEAD && git push origin main
kubectl -n kargo-lab annotate warehouse nginx "kargo.akuity.io/refresh=$(date +%s)" --overwrite
```

⚠️ **Revert tek başına yeşile döndürmez.** Revert sonrası promotion, pod henüz
toparlanmamışken verification'ı başlatıyor ve o koşu da başarısız oluyor
(provada `min` ancak T+100s'de 1'e döndü). Kargo başarısız doğrulamayı
kendiliğinden tekrar denemez. `min(nginx_up)` 1'e döndükten sonra reverify
tetiklemek gerekir:

```sh
VID=$(kubectl -n kargo-lab get stage dev \
  -o jsonpath='{.status.freightHistory[0].verificationHistory[0].id}')
kubectl -n kargo-lab annotate stage dev \
  "kargo.akuity.io/reverify={\"id\":\"$VID\",\"actor\":\"admin\",\"controlPlane\":false}" \
  --overwrite
```

~50 saniye sonra `Verified=True`.

## Provada öğrenilen üç davranış

1. **Doğrulama Freight başına yapışkan ve asimetrik.** Zaten doğrulanmış bir
   Freight'te başarısız reverify hiçbir şeyi geri almaz (Stage yeşil kalır).
   Başarısız bir Freight'te ise reverify çalışır ve kurtarır.
   Bu yüzden kırmızı göstermek için **yeni bir Freight** şarttır; mevcut
   Freight'i reverify etmek yetmez.

2. **`selfHeal` elle yapılan sabotajı 10 saniyeden kısa sürede geri alır.**
   `kubectl scale --replicas=0` denendi; Prometheus değişikliği görmedi bile.
   Kırılma git üzerinden gitmek zorunda.

3. **Pod'u çökerten bir kırma işe yaramaz.** nginx crashloop'a girerse
   `argocd-update` sağlık bekler ve promotion asılı kalır; verification hiç
   koşmaz. Doğru lever, pod'u sağlıklı bırakıp metriği bozmaktır — exporter'ın
   scrape adresini kaydırmak tam bunu yapıyor.

## Geri dönüş kaçış yolu

Demo tamamen ters giderse verification'ı zararsız hale getirip devam et:

```sh
kubectl -n kargo-lab patch analysistemplate smoke-check --type=json \
  -p '[{"op":"replace","path":"/spec/metrics/0/provider/prometheus/query","value":"vector(1)"}]'
```

Bu sorgu her zaman geçer. Sonra `kargo/analysis-templates.yaml` dosyasını
apply ederek geri al.
