# Samaritan – Export Mimarisi & Politikaları v2

Bu doküman, Samaritan projesinde **grafik**, **tablo** ve **dashboard** seviyesindeki export davranışlarını hem **UI/UX politikası** hem de **backend mimarisi** açısından tanımlar.  
Bu dosya iki ana bölüme ayrılmıştır:

- **Bölüm A – UI / UX Export Politikası**: Export işlemlerinin UI kısmında nasıl ele alınacağı ve export sürecini tetikleyecek olan UI elemanları
- **Bölüm B – Backend Mimarisi ve Sistem Değişiklikleri**: Export fonksiyonelliğinin mikroservis mimarisine nasıl oturduğunu, hangi servis ve tabloların ekleneceğini ve senkron/asenkron akışları anlatır.

---

## Bölüm A – UI / UX Export Politikası

Bu doküman, Samaritan projesinde **grafik**, **tablo** ve **dashboard** seviyesindeki **HTML / PDF / CSV / Excel** export davranışlarını tanımlar.

Amaç:
- PDF’i **okunabilir rapor** formatı olarak konumlandırmak,
- Büyük tablolarda kullanıcıyı **Excel/CSV** gibi daha uygun formatlara yönlendirmek,
- Kullanıcıya tutarlı ve öngörülebilir bir UX sağlamak,
- Frontend ve backend tarafında **aynı kuralları** uygulamak.

### 1. Temel Kurallar

1. **Satır limiti (Row limit)**
   - PDF içinde gösterilecek tablolarda **maksimum 50 satır** kuralı uygulanır.
   - Bir tablonun satır sayısı `> 50` ise:
     - Bu tablo için **PDF export devre dışı** (disabled) olur.
     - CSV, Excel ve HTML export seçenekleri kullanılabilir.

2. **PDF’in rolü**
   - PDF, **insan gözüyle okunabilir rapor** içindir; ham veri dökümü için değildir.
   - Büyük veri setleri (50+ satırlı tablolar) için kullanıcı Excel/CSV ile tam veriye erişir.

3. **HTML dashboard**
   - Grafik + tablo + insight’ların birlikte görüldüğü **interaktif HTML dashboard**:
     - Satır sayısından bağımsız olarak **her zaman indirilebilir**.
   - HTML dashboard, Plotly grafikleri interaktif tutar; tabloları scroll/pagination ile gösterebilir.

4. **Global PDF butonu yok**
   - Uygulamada **global bir PDF butonu bulunmayacaktır**.
   - PDF aksiyonları:
     - **Grafik widget’ının kendi export menüsünde**,
     - **Tablo widget’ının kendi export menüsünde**,
     - **Dashboard export alanında (top-level dashboard PDF butonu)** tanımlanır.

### 2. Bileşenler ve Export Davranışı

#### 2.1. Grafik (Chart) Export Politikası

Her grafik widget’ının kendi yanında bir export menüsü bulunur (ör. `⋯` veya `Export` dropdown).

**Chart export seçenekleri:**
- `PNG olarak indir` (Plotly’nin kendi download mantığı veya custom)
- `PDF olarak indir` (grafik özel PDF – tek sayfa veya küçük rapor)
- (Opsiyonel) `HTML (sadece bu grafik) olarak indir`

**Kurallar:**
- Grafik export’unda **tablo satır limiti dikkate alınmaz**.
- **Chart → PDF seçeneği her zaman aktiftir (enabled).**
- Grafik PDF’i, sadece ilgili grafiği ve istenirse kısa bir başlık/açıklamayı içerir.
- Dashboard’taki tablo satır sayısından bağımsızdır.

**Özel not (Plotly PNG için backend yok):**
- Eğer kullanıcı Plotly’nin kendi yerleşik `Download plot as PNG` butonunu kullanıyorsa:
  - Bu işlem **tamamen frontend tarafında** gerçekleşir.
  - Backend (export-service vb.) bu senaryoda **hiç devreye girmez**.

#### 2.2. Tablo (Table) Export Politikası

Her tablo widget’ının kendi yanında bir export menüsü bulunur.

**Table export seçenekleri:**
- `CSV olarak indir`
- `Excel (.xlsx) olarak indir`
- `HTML (sadece bu tablo) olarak indir`
- `PDF (tablo raporu) indir`

**Davranış matrisi (UX seviyesi):**

| Satır sayısı (row_count) | CSV | Excel | HTML (tablo) | PDF (tablo) |
|--------------------------|-----|-------|--------------|-------------|
| `row_count ≤ 50`         | ✅  | ✅    | ✅           | ✅ (enabled)|
| `row_count > 50`         | ✅  | ✅    | ✅           | ⚪ Disabled |

- `PDF (tablo raporu) indir` butonu **her zaman görünür**, ancak:
  - `row_count ≤ 50` → **enabled**
  - `row_count > 50` → **disabled** (gri, tıklanamaz)
- Disabled durumunda hover tooltip mesajı (örn.):
  > “Bu tablo 50 satırdan fazla olduğu için PDF olarak indirilemez. Lütfen Excel veya CSV ile indirin.”

> Backend tarafında, **row_count ≤ 50** olan tablo export’ları için senkron HTTP stream;  
> diğer tüm tablo export senaryoları için (row_count > 50) Kafka + worker mimarisi uygulanacaktır.  
> Bu detaylar Bölüm B’de anlatılır.

#### 2.3. Dashboard Export Politikası (Grafik + Tablo + Insight Birlikte)

Dashboard seviyesinde bir Export alanı veya butonu bulunur (örn. ekranın sağ üstünde).

**Dashboard export seçenekleri:**
- `Interactive HTML dashboard indir`
- `PDF dashboard raporu indir`

**Davranış matrisi (dashboard düzeyi):**

| Koşul                                          | HTML Dashboard | PDF Dashboard butonu durumu |
|------------------------------------------------|----------------|------------------------------|
| Tüm tabloların satır sayısı `≤ 50`             | ✅ Enabled     | ✅ Enabled                   |
| En az bir tablonun satır sayısı `> 50`         | ✅ Enabled     | ⚪ Disabled                  |

- `Interactive HTML dashboard indir`
  - **Her zaman enabled**.
  - Grafik + Insight + Tablolar tek HTML dosyada sunulur.

- `PDF dashboard raporu indir`
  - **Her zaman görünür**, fakat:
    - Tüm tablolar `≤ 50` → **enabled**
    - En az bir tablo `> 50` → **disabled** (gri)
  - Disabled durumunda hover veya click ile **tatlı bir uyarı** gösterilir:
    - Hover tooltip örneği:
      > “Bu dashboard’taki bazı tablolar 50 satırdan fazla olduğu için PDF raporu oluşturulamıyor. Lütfen filtreleyerek satır sayısını azaltın veya veriyi Excel/CSV ile indirin.”
    - Eğer kullanıcı disabled butona tıklarsa (frontend izin veriyorsa) toast/modal:
      > “PDF raporu sadece 50 satır veya daha az satıra sahip tablolar için oluşturulabilir. Bu dashboard’ta 50’den fazla satırlı tablolar bulunduğu için PDF export devre dışı. Tam veriye Excel/CSV ile erişebilirsiniz.”

### 3. Backend Guardrail Politikası (UX dokümanından devralınan mantık)

Frontend doğru konfigüre edilse bile, **asıl doğrulama backend tarafında yapılmalıdır.**

#### 3.1. Tablo PDF Export Endpoint

- Endpoint örneği: `POST /export/table/pdf`
- Davranış:
  1. İstenen tablo dataframe olarak yüklenir.
  2. `row_count = len(df)` hesaplanır.
  3. Eğer `row_count > 50` ise:
     - **PDF üretme işlemi yapılmaz.**
     - HTTP 400/422 ile anlamlı bir hata döndürülür.

#### 3.2. Dashboard PDF Export Endpoint

- Endpoint örneği: `POST /export/dashboard/pdf`
- Davranış:
  1. Dashboard içeriği (grafikler, tablolar, insight’lar) derlenir.
  2. Dashboard’a ait tüm tabloların `row_count` değerleri kontrol edilir.
  3. Herhangi bir tablo için `row_count > 50` ise:
     - **PDF üretme işlemi yapılmaz.**
     - HTTP 400/422 ile anlamlı bir hata döndürülür.

#### 3.3. Grafik PDF Export Endpoint

- Endpoint örneği: `POST /export/chart/pdf`
- Grafik PDF export’unda tablo satır limiti yoktur.

---

## Bölüm B – Backend Mimarisi ve Sistem Değişiklikleri

Bu bölüm, yukarıdaki export UX/policy kurallarının Samaritan’ın **mikroservis mimarisi**, **Kafka event’leri** ve **metadata veritabanı** ile nasıl uyumlandırıldığını anlatır.

### B.1. Genel Mimari Resim

Export fonksiyonelliği şu bileşenler üzerinden çalışır:

- **export-service (yeni mikroservis)**  
  - Export isteklerini alır (HTTP veya Kafka).
  - Küçük tablolarda senkron HTTP stream olarak dosya üretir.
  - Büyük tablo / dashboard / scheduled export’larda Kafka üzerinden background worker gibi davranır.
- **db-execution-service**  
  - Export için gerekli SQL’i çalıştırır ve tabular veriyi export-service’e döner.
- **scheduler-service**  
  - Zamanlanmış raporlar (alarm/schedule ile kurulan export’lar) için job yönetimini yapar.
- **notification-service**  
  - Export tamamlandığında e-posta gönderir (özellikle scheduled export’lar).
- **Kafka**  
  - `export.requested`, `export.completed`, `export.failed` gibi event’leri taşır.
- **metadata DB (PostgreSQL)**  
  - Export isteklerinin ve üretilen dosyaların metadata’sını tutar:
    - `export.export_requests`
    - `export.export_artifacts`
- **Object storage (GCS / S3 / local)**  
  - Büyük export dosyalarının (Excel, PDF, HTML) saklandığı yer.

**Önemli istisna:**  
- Plotly’nin kendi `Download PNG` butonu tamamen frontend olduğu için backend bu senaryoda devreye girmez.  
- Küçük tablolarda (satır sayısı ≤ 50) “sadece tablo export” senaryosunda dosya **senkron HTTP stream** ile döndürülür ve kalıcı storage’a yazılmak zorunda değildir.

### B.2. Senkron vs Asenkron Export Kuralları (Uygulama Seviyesi)

Kritik kararların özet hali:

1. **Plotly PNG (grafik)**  
   - Tamamen frontend.  
   - Backend export pipeline’ı kullanılmaz.

2. **Sadece tablo export & `row_count ≤ 50`**  
   - UI’dan gelen istek, export-service tarafından **senkron** işlenir.
   - Dosya (CSV/Excel/HTML/PDF) HTTP response ile **stream** edilerek döndürülür.
   - Kalıcı storage’a yazmak zorunda değiliz (opsiyonel).  
   - Kafka event’i zorunlu değil; ama audit için `export_requests` kaydı atılabilir.

3. **Geri kalan tüm senaryolar** (**Kafka + worker + scheduler modeli**):
   - Büyük tablo export’ları (`row_count > 50`),  
   - Dashboard HTML/PDF export’ları (row count’tan bağımsız),  
   - Zamanlanmış (scheduled) export’lar (favori sorgular üzerinden),  
   - E-posta ile gönderilen raporlar,
   - Batch raporlar.
   - Bu senaryolarda ortak yaklaşım:
     1. Export isteği kaydedilir (`export_requests`).
     2. Kafka’ya `export.requested` event’i atılır.
     3. export-service veya ayrı worker instance’ları bu event’i tüketir.
     4. Dosya üretilir ve object storage’a yazılır.
     5. `export.completed` / `export.failed` event’i yayımlanır.
     6. notification-service mail veya diğer bildirimleri tetikler.

> Yani:  
> - **Küçük tablo export’u** = “direct HTTP stream” (lightweight, no queue).  
> - **Diğer her şey** = “Kafka + worker + (scheduler varsa) pipeline”.

### B.3. Export-Service Sorumlulukları

**export-service** şu rollerle çalışır:

1. **HTTP API ile senkron küçük tablo export’u**  
   Örnek endpoint:
   - `POST /export/table`
   - Body: `query_run_id`, `format`, `row_count` (isteğe bağlı), vb.

   Adımlar (küçük tablo senaryosu):

   1. `audit.query_runs` tablosundan `sql_text`, `datasource_id`, `tenant_id` bilgisi alınır.
   2. `db-execution-service` üzerinden SQL tekrar çalıştırılır.
   3. Sonuç setinin satır sayısı hesaplanır (`row_count`).
   4. Eğer **yalnızca tablo export** isteniyorsa ve `row_count ≤ 50` ise:
      - İstenen formata göre (CSV/Excel/HTML/PDF) dosya memory’de üretilir.
      - HTTP response ile **stream** edilir (direct download).
      - İsteğe bağlı olarak `export.export_requests`’e bir satır yazılır (`status='succeeded'`), storage_path boş kalabilir veya `inline` gibi bir işaret set edilebilir.
   5. Eğer `row_count > 50` ise:
      - Bu istek **async pipeline’a** yönlendirilir (aşağıdaki asenkron akış devreye girer).

2. **Kafka Consumer olarak background worker**  
   - `export.requested` topic’ini dinler.
   - Event içinden:
     - `tenant_id`, `user_id`
     - `source_type` (ad_hoc_query / favorite_query / scheduled_job)
     - `source_id` (query_run_id / favorite_query_id / job_id)
     - `format`, `delivery_channel` bilgilerini alır.
   - Gerekli SQL’i `audit.query_runs` veya `context.favorite_queries` üzerinden çıkarır.
   - `db-execution-service` ile sorguyu tekrar çalıştırır.
   - Dosyayı üretir (Excel, PDF, HTML dashboard vs.).
   - Dosyayı object storage’a yazar.
   - `export.export_artifacts` tablosuna kayıt ekler.
   - `export.export_requests.status` alanını günceller.
   - Kafka’ya `export.completed` veya `export.failed` event’i gönderir.

3. **Dashboard Export (her zaman async)**  
   - `POST /export/dashboard` veya Kafka üzerinden tetiklenir.
   - Dashboard içindeki tüm tabloların row_count’larını kontrol eder (db-execution-service veya önceden hesaplanan metadata üzerinden).
   - Eğer PDF istendi ve herhangi bir tablo `> 50` satır ise:
     - PDF üretmez, hata döner (veya `export.failed` event’i üretir).
   - HTML dashboard export’unda row_count limiti yoktur; fakat boyut çok büyükse yine async pipeline’da çalışır.

4. **Scheduled Export entegrasyonu**  
   - scheduler-service tarafından üretilen `export.requested` event’lerini tüketir.
   - Zamanlanmış job’lar için:
     - `scheduler.jobs` tablosundan job detayını,
     - `context.favorite_queries`’den ilgili sorguyu alır,
     - Export pipeline’ını çalıştırır.

### B.4. Metadata DB – Export Schema Tasarımı

Export ile ilgili metadata, PostgreSQL’de `export` şeması altında tutulur.

#### B.4.1. `export.export_requests`

Her export isteği için bir satır:

- `id` (uuid, PK)
- `tenant_id` (uuid)
- `user_id` (text – Keycloak `sub`)
- `source_type` (enum)
  - `'ad_hoc_query'`
  - `'favorite_query'`
  - `'scheduled_job'`
- `source_id` (uuid/text)
  - ad_hoc → `audit.query_runs.id`
  - favorite → `context.favorite_queries.id`
  - scheduled → `scheduler.jobs.id`
- `datasource_id` (uuid)
- `format` (enum: `excel`, `pdf`, `csv`, `html_dashboard` …)
- `delivery_channel` (enum: `download`, `email`)
- `status` (enum: `pending`, `running`, `succeeded`, `failed`)
- `error_message` (text, nullable)
- `requested_at` (timestamptz)
- `started_at` (timestamptz, nullable)
- `finished_at` (timestamptz, nullable)
- `request_id` (text, nullable – API gateway trace ID)

**Not:**  
- Küçük tablo senkron export’larında da audit amacıyla satır açılabilir.  
- Bu durumda `delivery_channel='download'`, `format` uygun şekilde set edilir, ancak `export_artifacts` kaydı zorunlu olmayabilir (çünkü dosya kalıcı storage’a yazılmamış olabilir).

#### B.4.2. `export.export_artifacts`

Her üretilen dosya için bir satır:

- `id` (uuid, PK)
- `tenant_id` (uuid)
- `export_request_id` (uuid, FK)
- `artifact_type` (enum: şimdilik `'file'`)
- `storage_provider` (enum: `gcs`, `s3`, `local`, `filesystem` …)
- `storage_path` (text)  
  Ör: `samaritan-exports/tenant-uuid/2025/12/07/req-uuid/report.xlsx`
- `file_name` (text)
- `mime_type` (text)
- `file_size_bytes` (bigint, nullable)
- `expires_at` (timestamptz, nullable)
- `created_at` (timestamptz)

Bu tablo sayesinde:

- Mail ile gönderilecek linkler üretilebilir,
- Admin panelde “hangi tenant ne kadar export üretiyor?” gibi raporlar alınabilir,
- Eski export dosyaları batch job’larla silinebilir.

### B.5. Kafka Topic’leri ve Export Akışları

Export için kullanılan başlıca Kafka topic’leri:

- `export.requested`
  - Producer:
    - scheduler-service (scheduled export)
    - export-service’in HTTP katmanı (büyük tablo veya dashboard export’u için interactive istekleri async’e çevirirken)
  - Consumer:
    - export-service (worker instance’ları)
- `export.completed`
  - Producer:
    - export-service
  - Consumer:
    - notification-service
    - audit/logging
- `export.failed`
  - Producer:
    - export-service
  - Consumer:
    - notification-service (hata maili istenirse)
    - audit/logging

#### B.5.1. Senaryo – Küçük tablo, anlık export (row_count ≤ 50)

1. Kullanıcı UI’da tabloyu görür, satır sayısı 50’den azdır.
2. `Export → Excel/PDF` butonuna basar.
3. UI, access token ile birlikte export-service’e HTTP isteği atar (`POST /export/table`).  
4. export-service:
   - `audit.query_runs`’tan SQL’i alır,
   - `db-execution-service` ile çalıştırır,
   - row_count’ı hesaplar,
   - `row_count ≤ 50` ise dosyayı üreterek **aynı response içinde stream eder**.
   - İsterse `export.export_requests`’e bir satır yazar (`status='succeeded'`), Kafka kullanmak **zorunda değildir**.

Bu akışta:

- Kafka ve scheduler devreye girmez.
- Storage kullanımı zorunlu değildir (tamamen “inline response”).

#### B.5.2. Senaryo – Büyük tablo (row_count > 50) export’u

1. Kullanıcı tabloda 5000 satır görmektedir.
2. `Export → Excel` butonuna basar.
3. UI yine `POST /export/table` çağrısını yapar.
4. export-service:
   - SQL’i çalıştırır veya row_count bilgisini alır.
   - `row_count > 50` olduğunu görür.
   - Kullanıcıyı bilgilendirmek üzere HTTP response’ta:
     - “Rapor arka planda hazırlanıyor, hazır olunca mail olarak alacaksınız” gibi bir mesaj döner.
   - `export.export_requests` tablosuna `status='pending'` ile satır yazar.
   - Kafka’ya `export.requested` event’ini gönderir.

5. export-service’in worker instance’ı (Kafka consumer):
   - `export.requested` event’ini okur.
   - SQL’i tekrar çalıştırır (veya query_run_id üzerinden çeker).
   - Excel dosyasını üretir, storage’a yazar.
   - `export.export_artifacts` kaydını oluşturur.
   - `export.export_requests.status='succeeded'` yapar.
   - Kafka’ya `export.completed` event’ini gönderir.

6. notification-service:
   - `export.completed` event’ini dinler,
   - Tenant konfigürasyonuna göre:
     - Mail gönderir (attachment veya link),  
     - veya ileride webhook çağrısı yapabilir.

#### B.5.3. Senaryo – Dashboard export (HTML/PDF)

Dashboard export’ları **her zaman async pipeline** ile çalışır (row_limit’ten bağımsız).

1. UI, `POST /export/dashboard` çağrısı veya doğrudan `export.requested` event’i ile isteği başlatır.
2. export-service isteği `export.export_requests`’a yazar ve Kafka’ya `export.requested` atar.
3. Worker:
   - Dashboard tanımını toplar (grafikler, tablolar, insight’lar).
   - Tabloların row_count’larını kontrol eder:
     - PDF istendi ve bir tablo `> 50` satır ise → PDF üretmez, `export.failed` event’i üretir.
     - HTML dashboard istendi ise row_count limiti yoktur.
   - Dosyayı üretip storage’a kaydeder ve `export.completed` event’i yollar.

### B.6. Storage Kullanımı ve HTTP Stream Mantığı

- **Sadece tablo & row_count ≤ 50**:
  - Dosya **HTTP response stream** olarak döndürülebilir.
  - Ekstra storage zorunlu değildir.
  - Sadece audit amaçlı metadata DB kaydı tutulur.

- **Mail, büyük tablolar ve dashboard export’ları**:
  - Dosya mutlaka storage’a yazılır.
  - E-posta ile indirilebilir link/attachment için bu gereklidir.
  - `export_artifacts` tablosu storage path ve meta bilgileri tutar.

Bu ayrım sayesinde:

- Küçük, sık yapılan tablo indirme işlemleri backend’i yormaz ve basit kalır.
- Büyük ve kompleks export’lar ise asenkron, dayanıklı ve tekrar izlenebilir bir pipeline üzerinden akar.

---

Bu doküman, Samaritan’daki export fonksiyonelliğinin hem **kullanıcı deneyimi** (Bölüm A) hem de **mikroservis mimarisi** (Bölüm B) açısından tek referans noktası olacak şekilde tasarlanmıştır.  
Yeni ihtiyaçlar ortaya çıktıkça (örneğin webhook ile dış sistemlere rapor iletmek), bu mimari Kafka event’leri ve export-service üzerinden genişletilebilir.
