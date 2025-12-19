# Subscription Service

Tenant’ların plan ve özellik (entitlement) bilgisinin tek doğrusu. Plan tanımlarını, tenant-plan ilişkisinin yönetimini ve kullanım sayaçlarının (quota) tutulmasını üstlenir; Gateway ve diğer servisler buradan okuma yapar. Ödeme sağlayıcısı daha sonra eklenecek; bugün mock/manuel atama ile çalışır.

## Görevler
- Plan tanımları: `Free / Pro / Enterprise` ve feature bayrakları (ör. `max_queries_per_day`, `max_rows`, `statement_timeout_ms`, `export_enabled`, `chart_enabled`, `insight_enabled`, `max_exports_per_day`, `max_charts_per_day`, `concurrency_limit`).
- Tenant-plan ilişkisinin tutulması: `tenant_plan` (tenant_id, plan_id, status=active).
- Opsiyonel override: tenant bazlı feature override (admin özel durumları).
- Kullanım sayaçları: sorgu/grafik/export kullanımını atomik olarak artırmak, limit kontrolü için Gateway ve servislerle birlikte çalışmak.
- Dışarıya API: plan/feature bilgisini dönen `GET /entitlements/tenant/:tenant_id`, admin-only `POST /entitlements/tenant/:tenant_id/assign-plan` (şimdilik mock provider gibi davranır).

## Veri Modeli (özet)
- `plans`: `id`, `name`, `feature_flags` (JSON).
- `tenant_plan`: `tenant_id`, `plan_id`, `status`, `assigned_by`, `assigned_at`.
- `feature_overrides` (opsiyonel): `tenant_id`, `feature_key`, `override_value`.
- `usage_counters` (opsiyonel, kalıcı): `tenant_id`, `metric` (queries/exports/charts), `period_start`, `period_end`, `value`.

## Redis Entegrasyonu
- Dağıtık, hızlı sayaç tutmak için Redis kullanılır: key örneği `tenant:{id}:usage:queries:{YYYYMMDD}`. Export ve chart için benzer key’ler.
- TTL periyotla uyumlu (günlük/aylık) set edilir; her istekte `INCR`/`INCRBY` + limit kontrolü.
- Periyodik job (cron/worker) Redis’ten değerleri okuyup `usage_counters` tablosuna flush eder (rapor/audit için kalıcılık).
- Schema/rule version kullanılan cache’lerde olduğu gibi Redis sadece sayaç ve ephemeral metadata için; satır verisi tutulmaz.

## API Gateway / Rate Limit Entegrasyonu
- Gateway (NGINX+Lua+Redis veya gateway içi rate limiter), plan’a göre rate limit ve günlük sorgu/usage limitlerini uygular: limit bilgisi Subscription Service’ten okunur, sayaçlar Redis’te tutulur.
- Limit aşıldığında 429/403 döner; audit log’a “entitlement limit exceeded” olayı yazılır.
- DbExecute/Export/Chart servisleri de plan parametrelerini kullanır:
  - `max_rows`, `statement_timeout_ms` gibi sınırlar,
  - `export_enabled`, `chart_enabled`, `insight_enabled` bayrakları,
  - Günlük `max_exports_per_day`, `max_charts_per_day` için sayaç kontrolü.

## Ödeme Sağlayıcısı (gelecek)
- `BillingProvider` arayüzü: assign/cancel webhook vs. Şu an `MockBillingProvider` ile admin endpoint plan atar.
- Gerçek provider (Stripe/iyzico vb.) geldiğinde: webhook → Subscription Service plan’ı günceller, Gateway/servisler otomatik yeni limitlere göre davranır.

## UI/Gating
- Frontend plan/feature bilgisini Subscription Service’ten alır; butonlar/özellikler plan izinlerine göre açılır veya disabled.
- Limit aşımlarında kullanıcıya net mesaj: kalan sorgu/export/grafik hakkı göstergesi.
