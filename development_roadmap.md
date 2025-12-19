# Samaritan Development Yol Haritası (Monorepo, Microservices, GCP + Kubernetes)

Bu doküman, Samaritan’ı **monorepo** ile başlatıp **microservices** hedefiyle büyütürken development sürecini küçük ve yönetilebilir fazlara böler. Her faz; **hedef**, **yapılacaklar**, **çıktılar / başarı kriterleri** ve **sonraki faza geçiş koşulları** içerir.

> Not: Bu yol haritası “takvim” değil, **milestone** odaklıdır. Bir faz bitmeden bir sonrakine geçmek, genelde ileride pahalı refactor doğurur.

---

## Faz 0 — Repo iskeleti, kurallar ve development ortamı

### Hedef
İlk günden “endüstriyel disiplin”: repo düzeni, kontratlar, kalite kapıları, local bağımlılıklar.

### Yapılacaklar
- **Monorepo yapısı**
  - `services/*` (her servis bağımsız Python package)
  - `apps/ui` (Next.js)
  - `libs/*` (ortak: auth, error model, observability, config)
  - `infra/compose` (local bağımlılıklar)
  - `infra/k8s` (şablon/placeholder)
- **Çekirdek kontrat mini-spec’leri**
  - Tenant context standardı (header/claim, kim üretir/kim doğrular)
  - Standart hata zarfı (error envelope)
  - Correlation/trace standardı (`request_id`, `traceparent`)
  - API versiyonlama (`/v1/...`)
- **Kalite kapıları**
  - Local: pre-commit (format/lint/typecheck/test)
  - CI: GitHub Actions (PR üzerinde aynı kontroller)
- **Local bağımlılıklar (docker-compose)**
  - PostgreSQL
  - Keycloak
  - (Opsiyonel) Redis (şimdilik “boş” olabilir)
- **Observability minimum**
  - stdout’a **JSON structured logging**
  - `request_id` üretme/taşıma middleware
  - Log redaction: token/secret/authorization header asla loglanmaz

### Çıktılar / Başarı kriterleri
- `docker compose up` → Postgres + Keycloak (ve varsa Redis) ayağa kalkıyor.
- Servis template’lerinde `/healthz` ve `/readyz` var.
- Her request’te `request_id` loglanıyor ve servisler arası taşınıyor.

### Sonraki faza geçiş koşulu
- Repo yapısı ve minimum kontratlar (tenant context, error, trace) **yazılı** ve uygulanmış olmalı.

---

## Faz 1 — Walking Skeleton (uçtan uca en ince çalışan hat)

### Hedef
Sistem uçtan uca çalışsın: **UI → Gateway → Query-service → DB → UI**.

### Yapılacaklar
- **Gateway (minimal)**
  - Tek endpoint forward/proxy (ör. `POST /v1/query/run`)
  - `request_id` üretir ve downstream’e taşır
- **Query-service (minimal)**
  - DB’ye bağlanıp “hello query” çalıştırır (`SELECT 1` veya küçük sabit sorgu)
  - Standart response + hata formatı
- **UI (minimal)**
  - Tek sayfa: “Run” → sonucu ekranda göster
- **Auth (minimum iskelet)**
  - Keycloak token doğrulama (en azından “token var mı + imza doğrulama”)
  - Tenant context doğrulama (minimum tenant boundary)
- **Health**
  - Her serviste `/healthz`, `/readyz`

### Çıktılar / Başarı kriterleri
- UI’dan istek atılınca sonuç döner.
- Zincir boyunca aynı `request_id/trace_id` ile log/trace takip edilebilir.

### Sonraki faza geçiş koşulu
- Walking skeleton “gerçekten” çalışıyor olmalı; sadece teorik değil.

---

## Faz 2 — Platform çekirdeği (Tenant, AuthZ, Audit, Entitlement)

### Hedef
Multi-tenant güvenlik ve yönetişim temelini oturtmak (sonradan eklemek pahalıdır).

### Yapılacaklar
- **Tenant yönetimi (minimum)**
  - Tenant oluşturma (system_admin)
  - Tenant context doğrulama kurallarını kesinleştirme
- **Authorization (RBAC) iskeleti**
  - `system_admin`, `tenant_admin`, `tenant_user`
  - Endpoint-level guard’lar
- **Entitlement (ödeme olmadan da)**
  - `free/pro/enterprise` + feature/limit anahtarları
  - Admin-only plan atama (mock)
  - Servislerde enforcement (özellikle query/export)
- **Audit logging**
  - `tenant_id`, `user_id`, `action`, `resource`, `decision`, `trace_id`
  - Prompt/SQL için `hash` alanları (ileride PII politikasına uygun)
- **Error/hijyen standardizasyonu**
  - 401/403/429 semantiği
  - Sensitive data redaction

### Çıktılar / Başarı kriterleri
- Tenant boundary deterministik ve test edilmiş.
- RBAC guard’ları her kritik endpoint’te var.
- Audit log şeması ve kayıtları oluşuyor.

### Sonraki faza geçiş koşulu
- “Tenant izolasyonu + authz” olmadan core feature’a (metadata/query) geçilmez.

---

## Faz 3 — Tenant DB Connection + Metadata Ingest

### Hedef
Tenant’ın veri kaynağına bağlanmak ve şemayı keşfetmek: text-to-SQL’in deterministik girdisi.

### Yapılacaklar
- **DB connection yönetimi**
  - Tenant admin DB bağlantısı ekler
  - “Test connection” endpoint’i (timeout + güvenli hata mesajları)
- **Metadata keşfi (introspection)**
  - table/column çıkarımı
  - metadata tablolarına yazma
- **UI akışı**
  - Basit wizard: connection → test → ingest → ready
- **Guardrails**
  - Bağlantı sayısı/ingest limitleri (entitlement ile)

### Çıktılar / Başarı kriterleri
- Tenant admin DB bağlayıp tablo/kolon listesini UI’da görür.
- Metadata “sistem kaydı” haline gelir.

### Sonraki faza geçiş koşulu
- Metadata modeli stabil bir “v0” seviyesine gelmiş olmalı.

---

## Faz 4 — Query Pipeline v0 (Prompt → SQL → Policy → Execute → Result)

### Hedef
Samaritan’ın kalbi: güvenli ve gözlemlenebilir query çalıştırma hattı.

### Yapılacaklar
- **SQL üretimi (v0)**
  - İlk etap minimal LLM veya stub/heuristic (kontrat sabit)
- **SQL policy enforcement (zorunlu)**
  - SELECT-only (varsayılan)
  - statement timeout
  - row limit
  - yasaklı keyword/fonksiyon listesi (v0)
- **Execute + sonuç formatı**
  - `columns`, `rows`, `types`, `row_count`
  - pagination/limit semantiği
- **Query history**
  - prompt_hash, sql_hash, duration, status, trace_id
- **Hata sanitize**
  - DB hata mesajları kontrol/maskeleme (schema sızıntısını engelle)

### Çıktılar / Başarı kriterleri
- Gerçek prompt ile query çalışıyor.
- Timeout/limit devrede; kötü query sistemi kilitlemiyor.
- Her query audit ediliyor.

### Sonraki faza geçiş koşulu
- Policy ve gözlemlenebilirlik (log/trace/metrics) minimum seviyede tamam.

---

## Faz 5 — Operasyonel sağlamlık (Rate limit, Cache, Concurrency, Background Jobs temeli)

### Hedef
Sistem gerçek kullanımı kaldıracak hale gelsin; abuse etkisi sınırlansın; maliyet kontrolü başlasın.

### Yapılacaklar
- **Rate limiting**
  - tenant bazlı + endpoint bazlı (entitlement uyumlu)
  - önce in-memory, sonra Redis implementasyonu (interface hazır)
- **Concurrency kontrolü**
  - tenant başına max concurrent query
  - DB pool saturasyon kontrolleri
- **Cache**
  - metadata cache
  - entitlement cache (kısa TTL)
- **Background job temeli**
  - job state machine + idempotency (export gibi işler için hazırlık)

### Çıktılar / Başarı kriterleri
- Yük altında sistem öngörülebilir şekilde degrade olur (429/timeouts).
- Redis’e geçiş “implementation swap” düzeyinde kalır.

### Sonraki faza geçiş koşulu
- Rate limit + concurrency + cache stratejileri devrede ve ölçümlenebilir.

---

## Faz 6 — Export / Reporting (asenkron tasarım)

### Hedef
Export pahalıdır; ana API’leri kilitlemeden güvenli şekilde üretmek.

### Yapılacaklar
- **Export API**
  - küçük sonuçlarda sync, büyüklerde async (stratejiye uygun)
- **Queue + Worker**
  - job publish/consume, progress takibi
  - idempotency ve retry stratejisi
- **Depolama (GCP hedefli)**
  - çıktılar object storage (GCS)
  - download için signed URL veya benzeri kısa ömürlü erişim
- **Audit**
  - export request ve download trail

### Çıktılar / Başarı kriterleri
- Büyük export’lar synchronous request’i kilitlemiyor.
- Kullanıcı job durumunu görüyor; çıktı güvenli indiriliyor.

### Sonraki faza geçiş koşulu
- Worker/queue ve güvenli dağıtım modeli test edilmiş olmalı.

---

## Faz 7 — Observability hardening + GKE’ye hazırlık

### Hedef
GCP + Kubernetes’e geçiş sürpriz olmasın; prod’a yakın operasyonel kontratlar otursun.

### Yapılacaklar
- **OpenTelemetry standardizasyonu**
  - HTTP client/server spans, DB spans
  - trace propagation doğrulama
  - sampling stratejisi (dev yüksek, prod kontrollü)
- **Metrics & dashboard**
  - golden signals (traffic/errors/latency/saturation)
  - queue depth, worker metrikleri
- **Log stratejisi**
  - stdout JSON (dosyaya log yok)
  - retention ve PII/prompt/sql log politikası
- **K8s readiness/liveness**
  - graceful shutdown (SIGTERM)
  - resource requests/limits taslakları
- **Secrets & IAM**
  - hedef: GCP Secret Manager + Workload Identity
  - dev’de env, prod’da managed secret
- **CI güvenlik adımları**
  - dependency scan, secret scan, image scan (minimum)

### Çıktılar / Başarı kriterleri
- Local ve GKE davranışı yakın: config/health/shutdown.
- GCP’de “gözlemleme körlüğü” yok; trace + metrics + logs işler.

---

# Development boyunca uyulacak temel kurallar (GCP/GKE odaklı)

1. **Stateless servis**: state DB/Redis/GCS gibi dış sistemlerde tutulur.  
2. **12-factor config**: env var’lar; dev/prod isimleri tutarlı.  
3. **Logs stdout**: K8s/GCP toplar; dosyaya log yazma yok.  
4. **Metrik label kardinalitesi**: `tenant_id` metrics label olmaz (log/trace’te taşınır).  
5. **Kontrat değişikliği disiplin ister**: versiyonlama / backward compatibility.  
6. **Enforcement feature ile birlikte gelir**: auth/tenant/entitlement kontrolü sonradan eklenmez.  
7. **CI yeşil değilse merge yok**: trunk-based çalışma disiplini.

---

# Kısa “Faz geçiş” özeti

- Faz 0 → 1: repo + kontratlar + local bağımlılıklar hazır  
- Faz 1 → 2: walking skeleton uçtan uca gerçek çalışıyor  
- Faz 2 → 3: tenant boundary + authz + audit temel oturdu  
- Faz 3 → 4: metadata v0 stabil  
- Faz 4 → 5: policy + query hatları ölçümlenebilir ve güvenli  
- Faz 5 → 6: operasyonel limitler ve job temeli hazır  
- Faz 6 → 7: export pipeline çalışıyor; prod’a hazırlık başlıyor
