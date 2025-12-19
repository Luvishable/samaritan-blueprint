# Samaritan Güvenlik Blueprint’i (Endüstriyel Seviye)

> Bu doküman, Samaritan (multi-tenant, microservice tabanlı, text-to-SQL) projesi için **uygulanabilir güvenlik önlemleri** ve bu önlemlerin **neden/nerede/nasıl** uygulanacağına dair kapsamlı bir referanstır.  
> Odak: **kimlik doğrulama/ yetkilendirme**, **web güvenliği**, **API güvenliği**, **text-to-SQL’e özgü korumalar**, **altyapı/CI-CD/supply chain**, **izlenebilirlik ve denetim**.

---

## İçindekiler

1. [Güvenlik hedefleri ve tehdit modeli](#güvenlik-hedefleri-ve-tehdit-modeli)  
2. [Katmanlı güvenlik yaklaşımı (defense-in-depth)](#katmanlı-güvenlik-yaklaşımı-defense-in-depth)  
3. [Kimlik doğrulama (Auth) ve hesap güvenliği](#kimlik-doğrulama-auth-ve-hesap-güvenliği)  
4. [Yetkilendirme ve tenant izolasyonu (RBAC/ABAC, RLS/CLS)](#yetkilendirme-ve-tenant-izolasyonu-rbacabac-rlscls)  
5. [Token yönetimi (Keycloak) ve Next.js BFF + Session modeli](#token-yönetimi-keycloak-ve-nextjs-bff--session-modeli)  
6. [Session kavramı, SessionID, cookie ayarları](#session-kavramı-sessionid-cookie-ayarları)  
7. [CSRF ve SameSite](#csrf-ve-samesite)  
8. [CORS ve Preflight (OPTIONS)](#cors-ve-preflight-options)  
9. [XSS, Clickjacking ve CSP](#xss-clickjacking-ve-csp)  
10. [Replay protection (nonce/timestamp/imza)](#replay-protection-noncetimestampimza)  
11. [API güvenliği: validation, rate limiting, idempotency, abuse prevention](#api-güvenliği-validation-rate-limiting-idempotency-abuse-prevention)  
12. [DoS/DDoS ve kaynak tüketimi savunmaları](#dosddos-ve-kaynak-tüketimi-savunmaları)  
13. [HTTP temelleri: request/response parçaları, body nedir](#http-temelleri-requestresponse-parçaları-body-nedir)  
14. [Text-to-SQL’e özgü güvenlik: SQL policy, sandbox, parser, limitler](#text-to-sqle-özgü-güvenlik-sql-policy-sandbox-parser-limitler)  
15. [Secrets yönetimi (Vault/Secrets Manager), anahtar döndürme](#secrets-yönetimi-vaultsecrets-manager-anahtar-döndürme)  
16. [Servisler arası güvenlik: mTLS, network policies, gateway/WAF](#servisler-arası-güvenlik-mtls-network-policies-gatewaywaf)  
17. [CI/CD ve Supply chain güvenliği](#cicd-ve-supply-chain-güvenliği)  
18. [Logging, monitoring, observability ve audit](#logging-monitoring-observability-ve-audit)  
19. [Kademeli rollout, feature flags ve rollback](#kademeli-rollout-feature-flags-ve-rollback)  
20. [Uygulama planı: MVP → Production Hardening](#uygulama-planı-mvp--production-hardening)  
21. [Kontrol listeleri (checklists)](#kontrol-listeleri-checklists)

---

## Güvenlik hedefleri ve tehdit modeli

### Hedefler
- **Tenant izolasyonu:** Bir tenant’ın kullanıcıları başka tenant’ın verisini **asla** görememeli.  
- **Yetki sınırları:** Kullanıcı/rol yetkileri **UI’a değil**, server/DB katmanına dayalı olmalı.  
- **Veri koruma:** Hassas veri (kredi kartı/kimlik vb. varsa) korunmalı; en azından **in transit** ve **at rest** şifreleme.  
- **Sistem sürekliliği:** Kötü niyetli ya da hatalı kullanımla sistemin devre dışı kalması engellenmeli (DoS, abuse).  
- **Denetlenebilirlik:** “Kim, ne zaman, hangi prompt ile hangi SQL’i üretti ve çalıştırdı?” izlenebilir olmalı.  
- **LLM kaynaklı risk kontrolü:** Model yanlış SQL üretebilir; **policy + validator + limitler** ile hasar sınırlandırılmalı.

### Ana tehdit sınıfları
- **Kimlik hırsızlığı / hesap ele geçirme:** brute force, credential stuffing, MFA bypass denemeleri  
- **XSS/Clickjacking/CSRF:** tarayıcı tabanlı saldırılar  
- **Yetki yükseltme / tenant escape:** yanlış context taşıma, eksik RBAC, yanlış RLS  
- **Abuse / DoS:** pahalı sorgular, export bombaları, aşırı istek  
- **Supply chain:** zafiyetli dependency, image zafiyetleri, sızmış secret  
- **Observability boşlukları:** saldırı tespit edilememesi, audit eksikliği

---

## Katmanlı güvenlik yaklaşımı (defense-in-depth)

Samaritan’da güvenlik tek bir noktaya (token) yüklenmez. Katmanlar:

1. **Tarayıcı katmanı:** CSP, clickjacking/CSRF kontrolleri, cookie politikaları  
2. **UI/BFF katmanı (Next.js):** session yönetimi, CSRF, input validation, rate limit  
3. **API gateway katmanı:** auth doğrulama, request limitleri, WAF benzeri kontroller  
4. **Microservices:** RBAC/ABAC enforcement, request validation, business rule checks  
5. **DB katmanı:** RLS/CLS, yetkili DB kullanıcıları, timeouts/row limits  
6. **Infra:** network policies, mTLS, secrets, image scanning  
7. **Observability:** audit logs, anomali alarmları, trace/log korelasyonu

---

## Kimlik doğrulama (Auth) ve hesap güvenliği

### Keycloak (OIDC/OAuth2) ile temel prensipler
- **Authorization Code Flow (PKCE)**: tarayıcı tabanlı istemciler için modern varsayılan.  
- **MFA/TOTP**: özellikle `tenant_admin` ve `system_admin` için zorunlu.  
- **Brute force koruması:** deneme limitleri, geçici lockout, exponential backoff.  
- **Session yönetimi:** aktif oturumları yönetebilme (invalidate).  
- **Riskli aksiyonlarda step-up auth:** rol değişimi, DB bağlantısı ekleme gibi aksiyonlarda ek doğrulama.

### Kullanıcı güvenliği (minimum standart)
- Güçlü parola politikası + parola deneme limiti  
- MFA (rol bazlı zorunlu)  
- Şüpheli girişleri izleme (IP/cihaz anomalisi)  
- Audit: login success/fail, MFA events

---

## Yetkilendirme ve tenant izolasyonu (RBAC/ABAC, RLS/CLS)

### RBAC (Role Based Access Control)
Roller örnek:
- `system_admin`: sistem genel ayarlar, tenant lifecycle  
- `tenant_admin`: tenant içi yönetim (metadata, kullanıcılar, yetkiler)  
- `tenant_user`: sorgu çalıştırma/raporlama gibi sınırlı yetkiler

**Kural:** Yetki kontrolü **backend’de** zorunlu; UI sadece kullanıcı deneyimidir.

### ABAC (Attribute Based Access Control) – ihtiyaç varsa
- Kullanıcı, departman, lokasyon, proje gibi attribute’lara göre karar  
- Özellikle row-level güvenlik ihtiyaçlarında ABAC yardımcı olur.

### Tenant boundary
- Tenant context **her istekte** doğrulanmalı.
- Tenant bilgisini yalnızca header’dan almak **tek başına yeterli değildir**; token/session ile bağlanmalı.
- Servisler arası çağrılarda tenant context propagate edilir; **servis her zaman doğrular**.

### RLS/CLS (Row/Column-Level Security)
- Veriyi “UI filtreleriyle” gizlemek güvenlik değildir.
- DB (veya query layer) seviyesinde row/column erişimi kısıtlanır.
- Özellikle text-to-SQL’te, yanlış SQL üretildiğinde bile tenant izolasyonu DB katmanında korunmalıdır.

---

## Token yönetimi (Keycloak) ve Next.js BFF + Session modeli

### Neden “Next.js BFF + Session”?
Hedef: **Token’ı tarayıcı JS’inden uzak tutmak** ve XSS etkisini azaltmak.

#### Model A – SPA tarzı (token tarayıcıda)
- Browser token alır, API’ye `Authorization: Bearer ...` ile gider.
- Risk: XSS varsa token’a erişim riski artar (özellikle localStorage).

#### Model B – Next.js BFF + Session (önerilen)
1) Kullanıcı Keycloak’a yönlenir, auth code döner.  
2) **Next.js server** auth code’u alır, Keycloak ile token değişimini yapar.  
3) Browser’a token yerine **httpOnly session cookie** verilir.  
4) Browser artık sadece **Next’e** çağrı yapar.  
5) Next server microservices/gateway’e çağrıları **server-side** yapar (access token server’da kullanılır).  
6) Access token biterse Next server refresh token ile yeniler.

**Nerede üretilir?**
- Access/Refresh token’lar **Keycloak** tarafından üretilir.

**Nerede saklanır? (önerilen)**
- Browser: sadece httpOnly session cookie (JS okuyamaz)  
- Next.js server: session store içinde refresh token (ve kısa ömürlü access token cache)

> Not: Cookie-session modelinde **CSRF** koruması gereklidir (aşağıda detaylı).

---

## Session kavramı, SessionID, cookie ayarları

### Session nedir?
HTTP stateless’tir. Session, “kullanıcının oturum bilgisini” birden fazla istekte tutma mekanizmasıdır.

### SessionID nasıl atanır?
- Sunucu (Next) **rastgele, tahmin edilemez** bir SessionID üretir.
- Session store’da kullanıcı/tenant bilgisiyle ilişkilendirir.
- Browser’a `Set-Cookie` ile döner.

### Neden session’a ihtiyaç var?
- “Kullanıcı giriş yaptı mı?”
- “Hangi tenant aktif?”
- CSRF token/state bilgisi
- Refresh token’ı browser’dan saklama

### Cookie güvenlik ayarları (kritik)
- `HttpOnly`: JS cookie’yi okuyamasın (XSS etkisini azaltır)  
- `Secure`: sadece HTTPS üzerinden gitsin  
- `SameSite`: cross-site gönderim politikasını belirler  
- `Path`, `Domain`: kapsamı daralt  
- Kısa/orta TTL; refresh stratejisi net olmalı

---

## CSRF ve SameSite

### CSRF nedir?
Cookie tabanlı auth kullanıyorsan, başka bir site kullanıcının tarayıcısına “senin siteye istek attırabilir”. CSRF, “kullanıcının bilgisi olmadan state-changing işlem yaptırma” riskidir.

### SameSite nedir?
Cookie’nin cross-site isteklerde gönderilip gönderilmeyeceğini belirler:
- `Strict`: en sıkı (çoğu cross-site durumda göndermez)
- `Lax`: pratik varsayılan (bazı top-level navigation durumlarında gönderir)
- `None`: cross-site gönderir; **Secure zorunlu**

### CSRF savunma stratejileri (önerilen kombinasyon)
- `SameSite=Lax/Strict` (mümkünse)  
- State-changing isteklerde **CSRF token** (double submit veya server-side token)  
- `Origin` / `Referer` doğrulaması (özellikle kritik endpoint’lerde)  
- “Tehlikeli” aksiyonlarda ek onay/step-up auth

---

## CORS ve Preflight (OPTIONS)

### CORS nedir?
Tarayıcı güvenlik mekanizmasıdır: bir origin’deki sayfa, başka origin’deki API’ye istek atınca tarayıcı izin kontrolü yapar.

### Preflight (OPTIONS) nedir?
Bazı cross-origin isteklerde tarayıcı, gerçek isteği atmadan önce API’ye bir **OPTIONS** isteği atar.

- OPTIONS **kime gider?**  
  Asıl çağırmak istediğin API endpoint’ine gider (aynı URL).
- Tarayıcı “şu method ve header’larla istek atabilir miyim?” diye sorar:
  - `Origin`  
  - `Access-Control-Request-Method`  
  - `Access-Control-Request-Headers`

Sunucu “izin var” derse şu header’ları döner:
- `Access-Control-Allow-Origin`  
- `Access-Control-Allow-Methods`  
- `Access-Control-Allow-Headers`  
- (Cookie taşıyorsan) `Access-Control-Allow-Credentials: true`

### Samaritan’da pratik sonuç
- Eğer browser doğrudan `api.samaritan.com` çağırırsa: CORS/preflight sık görülür.  
- Eğer Next.js BFF ile browser tek origin’e konuşursa: CORS ihtiyacı ciddi azalır; Next’in server-to-server çağrıları CORS’a tabi değildir.

---

## XSS, Clickjacking ve CSP

### XSS (Cross-Site Scripting)
Saldırgan, senin domain’in altında kötü niyetli JS çalıştırır. Sonuç:
- UI manipülasyonu, veri okuma, kullanıcı adına aksiyon
- Eğer token JS erişimine açıksa token hırsızlığı

**Savunma**
- Output encoding (escape): kullanıcı girdisini HTML olarak basma  
- Tehlikeli API’lerden kaçın: ör. raw HTML injection (React’te `dangerouslySetInnerHTML`)  
- CSP ile script kaynaklarını daralt  
- HttpOnly cookie (token çalınmasını zorlaştırır)

### Clickjacking
Sitenin bir iframe içine gömülüp kullanıcıya kandırılarak tıklama yaptırılması.

**Savunma**
- `X-Frame-Options: DENY` veya `SAMEORIGIN`  
- CSP `frame-ancestors` ile izin verilen origin’leri kısıtla  
- Kritik aksiyonlarda ek onay ve step-up auth

### CSP (Content Security Policy) uygulama hedefleri
- `script-src` sadece `'self'` ve izinli kaynaklar  
- Inline script’leri minimize et; gerekiyorsa nonce/hash ile kontrollü izin ver  
- `frame-ancestors` ile clickjacking’i engelle  
- `default-src` ile genel kaynak politikasını daralt

---

## Replay protection (nonce/timestamp/imza)

### Replay nedir?
Geçerli bir isteğin (örn. export başlatma) yakalanıp tekrar gönderilmesiyle aynı aksiyonun tekrar tetiklenmesi.

### Strateji
- **Timestamp**: istek “freshness window” içinde mi? (örn. 60 sn)  
- **Nonce**: tek kullanımlık rastgele değer; sunucu TTL ile saklar; tekrar gelirse reddeder  
- **İmza**: timestamp + nonce + body üzerinde HMAC/Signature ile bütünlük

### Nerede uygulanır?
- Export job başlatma  
- Role/permission değişimi  
- Webhook endpoint’leri  
- “Mali/geri döndürülemez” işlemler

---

## API güvenliği: validation, rate limiting, idempotency, abuse prevention

### Input validation
- Tüm endpoint’lerde şema bazlı doğrulama (FastAPI/Pydantic tipik).
- “Beklenmeyen alanları” reddet (strict schema).
- Parametre limitleri: string uzunluğu, array uzunluğu, max nested depth.

### Rate limiting / throttling
- Login endpoint: brute force’a karşı  
- Query-run endpoint: abuse + DoS’a karşı  
- Export start endpoint: pahalı işlere karşı  
- Tenant bazlı limit: multi-tenant’ta şart

### Idempotency
Aynı isteğin tekrar gelmesi (retry) durumunda “çift job” oluşmasın:
- Idempotency key ile “aynı job zaten var” kontrolü  
- Özellikle export/async işlerde kritik

### Güvenlik kontrolleri
- Request size limit (body/header)  
- Timeout’lar  
- Robust error handling (hata mesajında hassas bilgi sızdırma)  
- Standard HTTP status kullanımı (401/403 ayrımı)

---

## DoS/DDoS ve kaynak tüketimi savunmaları

### DoS nedir?
Servisi yavaşlatmak/çöketmek için kaynak tüketimi:
- Volumetric (çok trafik)
- Application-layer (az ama pahalı istek)

### Savunma seti
- Rate limit + burst control  
- Body size limit + upload limit  
- Query timeouts / max row limits  
- Async queue + worker (export gibi)  
- Circuit breaker / bulkhead (servis izolasyonu)  
- WAF/DDoS koruması (özellikle public internet)

---

## HTTP temelleri: request/response parçaları, body nedir

### HTTP request parçaları
1) Request line: `METHOD /path HTTP/1.1`  
2) Headers: `Authorization`, `Content-Type`, `Cookie`, vb.  
3) Body (opsiyonel): JSON/form-data/dosya

### HTTP response parçaları
1) Status line: `HTTP/1.1 200 OK`  
2) Headers: `Set-Cookie`, `Content-Type`, güvenlik header’ları  
3) Body: JSON/HTML/dosya

### Body nedir?
Request/response yüküdür; JSON payload, dosya, form verisi vb. taşır.

---

## Text-to-SQL’e özgü güvenlik: SQL policy, sandbox, parser, limitler

Bu bölüm Samaritan için “yüksek öncelik”tir.

### 1) SQL komut tipi politikası (allow/deny)
- Varsayılan: **read-only** (SELECT)  
- Yasak: `DROP`, `ALTER`, `TRUNCATE`, `DELETE`, `UPDATE`, `INSERT`, `COPY`, `CREATE` vb.  
- Eğer yazma gerekiyorsa: ayrı rol + ayrı onay akışı + ayrı servis/DB user.

### 2) SQL doğrulama (parser/validator)
- Üretilen SQL çalıştırılmadan önce parse edilir:
  - Komut tipi doğrulanır (SELECT mi?)  
  - Kullanıcı/tenant izinlerine göre tablo/kolon erişimi kontrol edilir  
  - Join’ler, subquery’ler ve tehlikeli fonksiyonlar denetlenir

### 3) Query sandbox
- **statement timeout** (DB)  
- max row limit (örn. 10k)  
- concurrency limit (aynı tenant kaç query?)  
- max result size (response limit)  
- explain/cost guard (mümkünse)

### 4) Tenant izolasyonunu DB’de zorla
- RLS/tenant predicate zorunlu olsun.
- LLM yanlış sorgu ürettiğinde bile “tenant dışı veri” gelmemeli.

### 5) Audit
Her query için:
- tenant_id, user_id  
- prompt  
- üretilen SQL  
- duration, row_count  
- status (success/fail)  
- trace_id

---

## Secrets yönetimi (Vault/Secrets Manager), anahtar döndürme

### Prensipler
- Repo’da secret **asla** bulunmaz.  
- Secrets merkezi yönetime taşınır (Vault veya cloud secret manager).  
- Least privilege: her servis için ayrı credential.

### Operasyonlar
- Düzenli **rotation** (DB password, client secret)  
- Access logging (secret’a kim erişti?)  
- K8s secret’ları tek başına yeterli sayılmayabilir; Vault entegrasyonu değerlidir.

---

## Servisler arası güvenlik: mTLS, network policies, gateway/WAF

### mTLS (service-to-service)
- Servisler arası trafik şifreli ve kimliği doğrulanmış olur.
- Service mesh (ör. Istio/Linkerd) ile uygulanabilir.

### NetworkPolicies
- “Her pod her pod’a konuşmasın.”
- Sadece gerekli servisler konuşsun (east-west traffic kısıtlanır).

### Gateway / WAF
- Request size limit  
- Rate limit  
- Basic WAF kuralları (path/method anomalies)  
- Auth enforcement (JWT validation, header normalization)

---

## CI/CD ve Supply chain güvenliği

### CI (Pull Request)
- Lint/format  
- Unit tests  
- Typecheck  
- Docker build (en azından “build edilebilir mi?” kontrolü)  
- Dependency scan + secret scan  
- Container image scan

### CD (tag/release veya main)
- Tag geldiğinde: image build + sign + push (ileri seviye)  
- GitOps (Argo CD) ileride uygulanabilir  
- Reproducible builds (aynı commit aynı artifact)

### SBOM ve imzalama (ileri seviye)
- “Bu image’ın içinde hangi paketler var?”  
- Provenance / supply chain güvenliği için değerlidir.

---

## Logging, monitoring, observability ve audit

### Structured logging
- JSON log formatı
- Alanlar: `service`, `env`, `tenant_id`, `user_id`, `request_id`, `trace_id`, `action`

### Metrics
- Request rate, error rate, latency (p95/p99)  
- DB query latency, pool saturation  
- Worker queue depth, job duration  
- Kafka lag (varsa)

### Tracing (distributed)
- Request lifecycle’ı gateway → services → DB boyunca izlemek
- Trace/log korelasyonu: trace_id loglarda taşınmalı

### Audit log (ayrı kategori)
- Auth events, RBAC changes, metadata edits, export starts
- Append-only mantığı tercih edilir (değiştirilemezlik)

### Alerting
- 401/403 spike  
- query-run abuse  
- export failure spike  
- latency anomalisi

---

## Kademeli rollout, feature flags ve rollback

### Feature flag
Kod main’de olabilir ama flag kapalıysa kullanıcı görmez.  
- Release flags: bitmemiş işi gizle  
- Operational flags: Redis/cache gibi bileşenleri kapat/aç  
- Experiment flags: A/B test

### Rollback stratejileri
- “Tag” ile çalışan sürümü işaretleme  
- Revert kültürü (main’i hızlı yeşile döndür)  
- Canary/blue-green (ileri aşamada)

---

## Uygulama planı: MVP → Production Hardening

### MVP (minimum güvenli ürün)
1) Keycloak OIDC + MFA (admin)  
2) Next.js BFF + session cookie (httpOnly, secure, sameSite)  
3) RBAC ve tenant isolation (server-side)  
4) Text-to-SQL: SELECT-only + timeout + row limit + audit  
5) Rate limit: login + query-run + export start  
6) CSP temel politikası + clickjacking koruması  
7) Structured logs + audit logs

### Hardening (production)
- RLS/CLS tam oturtma  
- mTLS + network policies  
- CI security scanning + SBOM  
- Tracing + anomali alertleri  
- Replay protection (kritik aksiyonlar)  
- Secrets manager + rotation

---

## Kontrol listeleri (checklists)

### Web güvenliği
- [ ] CSP (script-src daraltıldı, inline minimize)  
- [ ] X-Frame-Options / frame-ancestors  
- [ ] CSRF stratejisi (SameSite + token + origin checks)  
- [ ] Secure, HttpOnly cookie  
- [ ] Input/output encoding (XSS)  
- [ ] Request size limits

### Auth & RBAC
- [ ] Keycloak realm/clients doğru  
- [ ] MFA (admin)  
- [ ] Role matrix net  
- [ ] Tenant boundary her request’te doğrulanıyor  
- [ ] Audit: role changes, login events

### Text-to-SQL
- [ ] SELECT-only policy enforced  
- [ ] SQL parser/validator  
- [ ] statement timeout  
- [ ] max rows / max result size  
- [ ] tenant isolation DB-level (RLS ideal)  
- [ ] query audit logs

### Infra
- [ ] Secrets manager  
- [ ] NetworkPolicy  
- [ ] mTLS/service mesh (opsiyonel)  
- [ ] Image scanning  
- [ ] Backups + restore test

---

## Ek: Örnek akış diyagramı (Next.js BFF + Session)

```text
Browser
  |
  | 1) Login (redirect)
  v
Keycloak  --(auth code)-->  Next.js (server)
  |                           |
  | 2) Token exchange         | 3) Set-Cookie: session_id (httpOnly)
  v                           v
Keycloak tokens          Browser (cookie only)
                                |
                                | 4) API calls to Next
                                v
                          Next.js (BFF)
                                |
                                | 5) Server-to-server calls w/ access token
                                v
                           API Gateway / Microservices
```

---

## Notlar
- Bu doküman “tüm önlemleri” tek seferde zorunlu kılmaz; **fazlara bölerek** uygulamak idealdir.
- En kritik nokta: **tenant izolasyonu + text-to-SQL policy + audit** (Samaritan’ın ayırt edici riskleri burada).
