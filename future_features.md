# Gelecek Özellikler

Bu dosya, ileride eklemeyi planladığımız iki kritik özelliğin ayrıntılı tasarımını içerir: (1) normalize edilmiş soru cache’i ve (2) ücretli üyelik altyapısı.

---

## 1) Normalize Edilmiş Soru Cache’i

### Amaç
- Aynı sorunun tekrar sorulmasında LLM ve RAG maliyetini düşürmek, cevap gecikmesini azaltmak.
- Mümkün olduğunda LLM’i tamamen atlayıp önceki guard’dan geçmiş SQL’i doğrudan çalıştırmak.

### Akış (happy-path)
1. **Normalize et:** Kullanıcı mesajını lower/trim, fazla whitespace/punctuation temizliği. (Regex ile sadeleştir, stopword çıkarma yapma.)
2. **Key üret:** `hash(normalized_question) + tenant_id + datasource_id + schema_version + rule_version` (örn. SHA256).
3. **Redis lookup:** TTL’li (10–30 dk) cache’te ara. Hit varsa:
   - Guard’dan geçmiş SQL + kullanılan RAG snippet ID’leri + chart_plan (varsa) alın.
   - Schema_version uyumluysa LLM çağırmadan DbExecute çalıştır.
   - İsteğe bağlı: Hızlı bir SqlGuard tekrar kontrolü (policy değişikliği varsa).
4. **Miss durumunda:** Normal IntentRouter → RAG araması → LLM akışı çalışır; çıkan RAG snippet ID’leri ve SQL cache’e aynı key ile yazılır.

### Cache içeriği
- `normalized_question_hash`
- `tenant_id`, `datasource_id`, `schema_version`, `rule_version`
- `rag_snippet_ids` (table/column/business rule/golden example referansları)
- `sql` (guard’dan geçmiş hali)
- Opsiyonel: `chart_plan`, `insight_specs` (LLM çıktıları)
- `ttl`: 10–30 dk (konfigürasyona göre)

### Validasyon ve invalidasyon
- **Schema/rule versiyonu değişirse** cache yok sayılır.
- **Policy değişirse** (PII/izin değişimi) SqlGuard yeniden koşulur; fail olursa cache kullanma.
- **Timeout veya DB hatası yaşanırsa** cache kullanılmaz; teknik retry normal akışta devam eder.

### Güvenlik
- Sadece SELECT-only SQL’ler saklanır.
- Tenant/datasource scope’unda izole key; cross-tenant collision engellenir.
- Log’larda masked SQL tutulur.

### Metrikler
- Cache hit/miss oranı
- Hit’te LLM atlama sayısı
- Ortalama yanıt süresi düşüşü (p95)
- Schema_version uyumsuzluğu nedeniyle düşen cache oranı

---

## 2) Ücretli Üyelik (Subscription) Altyapısı

### Amaç
- Çok kiracılı (multi-tenant) yapıda plan bazlı kullanım ve faturalama.
- Rate limit, özellik erişimi, destek seviyesi, SLA gibi kısıtların planda tanımlanması.

### Bileşenler
- **Billing provider entegrasyonu:** Stripe/Braintree/Adyen (PCI yükünü azaltmak için hosted checkout + webhook). Türkiye/yerel ihtiyaçlar için iyzico/paytr opsiyon olarak değerlendirilebilir.
- **Billing service (yeni mikroservis):**
  - Plan tanımları (free/pro/enterprise)
  - Abonelik durumu (active/trial/cancelled/past_due)
  - Kullanım ölçümleri (request sayısı, veri boyutu, export/grafik limitleri)
  - Webhook tüketimi: provider’dan event alır, DB’yi günceller, Kafka event yayınlar.
- **API Gateway/Rate limit entegrasyonu:** Plan’a göre limit ve quota enforcement (NGINX+Lua+Redis veya Gateway’in yerleşik rate limiter’ı).
- **İzin/özellik bayrakları:** Feature flag veya plan bazlı capability’ler (ör. grafik/insight erişimi, export izni).
- **UI/Onboarding:** Plan seçimi, ödeme sayfası yönlendirmesi, fatura bilgileri. Trial başlatma ve plan yükseltme/düşürme.

### Veri modeli (özet)
- `billing_plans`: id, name, price, currency, period, limits (req/day, export limit, chart/insight erişimi), feature_flags (JSON)
- `subscriptions`: tenant_id, plan_id, status, current_period_start/end, cancel_at, trial_end, provider_subscription_id
- `usage_records`: tenant_id, metric_type (requests, charts, exports), value, period
- `invoices/payments`: provider’dan gelen invoice/payment intent referansları

### Akış
1. Tenant plan seçer → Billing provider checkout → başarı webhook’u → `subscriptions` güncellenir, status=active.
2. Gateway rate limiter, plan limitlerini Redis’ten okur; aşıldığında 429 döner.
3. Kullanım metrikleri (requests/charts/exports) Kafka’ya event; billing-service günlük/aylık agregasyon yazar.
4. Plan değişikliği: UI → billing-service → provider API → webhook ile doğrula → plan_id/limitler güncellenir.
5. Ödeme sorunu/past_due: webhook → status=past_due → gateway’de kısıtlı mod veya grace period.

### Güvenlik ve uyumluluk
- Kredi kartı bilgisi backend’de tutulmaz; hosted checkout + PCI DSS uyumlu provider kullanılır.
- Webhook doğrulama (imza/sig).
- Tenant verisi ile ödeme verisi ayrıştırılır; sadece referans ID’ler saklanır.

### Metrikler ve izleme
- Aktif abonelik sayısı, churn, MRR/ARR (basic metrik).
- Plan başına rate limit ihlalleri.
- Payment success/fail oranı, webhook hata oranı.

---

## Sonraki Adımlar
- Redis key şemasını ve TTL değerlerini kararlaştır; schema_version/rule_version üretimini standardize et.
- Billing provider seçimi (Stripe vs yerel) ve PoC: checkout → webhook → subscription güncelleme.
- API Gateway’de plan bazlı rate limit politikasını hazırlayıp billing-service’den konfigürasyon okumayı ekle.
