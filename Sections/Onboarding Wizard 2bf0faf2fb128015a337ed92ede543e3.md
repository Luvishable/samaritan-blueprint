# Onboarding Wizard

---

## 1. Amaç ve Kullanım

---

> **Onboarding Wizard;** 
 
uygulamaya davet linki ile kaydolan bir tenant_admin’in kendi şirketine(tenant) ait bilgileri doldurduğu, 
kullanacakları database’e dair soruları cevapladığı, 
uygulamanın kullanımı sırasındaki tenant bazlı konfigürasyonlarını set ettiği aşamadır.
> 

- Sadece o şirkete ait ilk tenant_admin için zorunlu; diğer tenant_admin ve tenant_user’lar bu aşamayı görmeyecekler.

**Temel amaç:**

1. Yeni bir tenant’ın **şirket kaydını** oluşturmak (`catalog.tenants`).
2. Tenant’ın **ilk datasource’unu** tanımlamak (`catalog.datasources`).
3. Bu datasource’ta:
- Hangi **table / view’lerin** Samaritan’a açılacağını seçmek (`catalog.exposed_objects`).
- Bu object’lerde hangi **kolonların** kullanılacağını, hangilerinin PII olduğunu, semantiğini belirlemek (`catalog.exposed_columns`).
- Table/view ve kolonlar için **doğal dil açıklamaları** toplamak → daha sonra embedding-service ile vector DB’ye gidecek.

**NOT:**

Bu wizard bitmeden:

- Tenant “tamamlanmış” sayılmıyor,
- Bu tenant’a ait kimse soru soramıyor,
- nlp2sql / chart vs. uçları kullanılmıyor.

## 2. Onboarding Zorunluluğu

---

1. `system_admin` *auth_service* üzerinden bir e-posta adresine `tenant_admin` daveti gönderir.
2. *auth service:*
    1. `auth.invitations` tablosuna kayıt açar
    2. Keycloak Admin API ile user’ı oluşturur:
        1. realm_role: `tenant_admin`
        2. user attribute: `onboarding_pending = true`
        3. henüz `tenant_id` yok.
    3. Required Actions:
        1. `VERIFY_EMAIL`
        2. `UPDATE_PASSWORD`
        3. `CONFIGURE_TOTP`
3. tenant_admin davet linkine tıklar ve Keycloak:
    1. e-mail doğrular
    2. parola set ettirir
    3. TOTP kurdurur

**Login sonrası:**

- Kullanıcının access/ID token’ında:
    - `realm_access.roles` içinde `tenant_admin`
    - User attribute olarak `onboarding_pending = true`
1. UI login callback’inde token’de eğer `onboarding_pending = true` görürse kullanıcıyı direkt olarak Onboarding Wizard ekranına yönlendirir.

## 3. Onboarding’in Yarım Kalması Durumu

---

1. `catalog.tenants` içinde `onboarding_status` alanı var.
    1. `'pending'`   → tenant satırı bile açılmamış olabilir (ilk adım öncesi).
    2. `'incomplete'` → bazı adımlar tamam, bazıları eksik.
    3. `'completed'`  → tüm kritik adımlar bitti, sistem kullanılabilir.
2. Wizard her adımda ilgili bilgiyi postgres’e hemen yazar:
    - İlgili bilgiyi Postgres’e **hemen** yazar (step by step persist).
    - Örneğin:
        - Şirket bilgisi kaydedildiği anda `catalog.tenants` satırı oluşur.
        - Data source test edilip kaydedildiğinde `catalog.datasources` satırı oluşur.
    - `onboarding_status`:
        - İlk tenant row açıldığında `'incomplete'` olur,
        - Tüm zorunlu adımlar bitince `'completed'` yapılır.
3. Kullanıcı yarıda kapatırsa:
    1. Keycloak user attribute hâlâ `onboarding_pending = true` 
    2. `catalog.tenants.onboarding_status` `'incomplete'`
    3. Bir dahaki login’de:
        - UI tekrar wizard’a atar.
        - Wizard backend’den mevcut verileri çekip ilgili step’te formu **pre-fill** eder:
            - Şirket adı, datasource bilgisi, seçilmiş tablolar vb. aynen gelir.
            - Kullanıcı kaldığı yerden devam eder.

## 4. Adım Adım Onboarding Wizard Akışı

---

- **Adım 0 – Login + TOTP**
    - Tenant_admin davet linki ile gelir.
    - Keycloak:
        - e-mail doğrulama,
        - şifre belirleme,
        - TOTP kurulumu (required action).
    - Login tamamlandığında UI access + ID token alır:
        - Token’da `realm_access.roles` içinde `tenant_admin`,
        - User attribute olarak `onboarding_pending = true`.
    
    UI bu durumda:
    
    - Normal dashboard yerine:
        - `GET /catalog/onboarding/status` gibi bir endpoint ile duruma bakar:
            - `onboarding_status` + varsa `tenant_id` vs.
        - Cevap `'pending'` veya `'incomplete'` ise wizard’ı açar.

---

- **Adım 1 – Tenant (şirket) bilgileri**
    
    **Ekran:**
    
    “Şirket Bilgileri” sayfası.
    
    Tenant_admin’den istenenler:
    
    - Şirket adı (`tenant_name`)
    - Ülke (`country`)
    - Sektör (`sector`) vb.
    
    **Backend (catalog-service) ne yapar?**
    
    - Eğer bu tenant_admin için daha önce tenant row açılmamışsa:
        - `INSERT INTO catalog.tenants (...)`:
            - `id` → yeni uuid
            - `name` → formdan
            - `country`, `sector` → formdan
            - `created_by_user_id` → token’daki `sub`
            - `is_active = true`
            - `onboarding_status = 'incomplete'`
        - Bu `tenant_id` artık Samaritan tarafında resmi kimlik olur.
    - Bu noktada `auth-service` devreye girebilir ve Keycloak Admin API ile bu user için:
        - `tenant_id=<new_tenant_id>` attribute’unu set eder.
    - Böylece:
        - tenant_admin’in ID token/access token’ına `tenant_id` claim’i eklenebilir.
        - İleride bu user bu tenant’a bağlı tüm işlemlerini yapar.

---

- **Adım 2 – Datasource (DB bağlantısı) tanımı**
    
    **Ekran:**
    
    “Veri Kaynağı Tanımı” sayfası.
    
    tenant_admin’den istenenler:
    
    - DB türü (`db_type`): `postgres`, `oracle`, `sqlserver` vs.
    - Host, port
    - Database / service name
    - Kullanıcı adı, şifre
    - Gerekirse:
        - Schema owner, SID, vs. (DB tipine göre)
    
    UI, bu bilgileri **tek bir form** olarak toplar.
    
    **Backend akışı:**
    
    1. UI → catalog-service:
        - `POST /catalog/datasources/test-and-save`
        - Body içine tüm connection parametreleri eklenir.
    2. catalog-service:
        - Önce `db-execution-service`’e “test connection” çağrısı atar:
            - Bu servis, verilen parametrelerle DB’ye bağlanmayı dener.
            - Başarılıysa `ok`; değilse hata mesajı döner.
        - Eğer test başarılıysa:
            1. Gerçek connection string’i (veya param paketini) bir **secret manager**’a kaydeder:
                - Örn. `vault` / `gcp secret manager`:
                    - Secret adı: `tenant-{tenant_id}-primary-ds`
            2. `catalog.datasources` tablosuna kayıt yazar:
                - `id` (uuid)
                - `tenant_id` (Adım 1’de oluşturduğumuz tenant)
                - `name` (UI’dan alınan bir friendly name)
                - `db_type`
                - `connection_secret_ref = "tenant-{tenant_id}-primary-ds"`
                - `default = true`
                - `is_active = true`
                - `created_at`, `updated_at`
    
    Bu adım sonunda:
    
    - Tenant’ın sistemde **resmi bir datasource’u** oluşur.
    - Ama henüz hiçbir table/view seçilmediği için schema tarafı boştur.

---

- **Adım 3 – Table mı, View mü tercihi**
    
    **Ekran:**
    
    “Kullanım modu” sayfası.
    
    Burada alınacak karar:
    
    > Aynı datasource için karmaşanın önüne geçmek için başlangıçta ya table, ya view kullanılsın (ikisi birden değil).
    > 
    
    UI şu soruyu sorar:
    
    - “Analizleri hangi nesneler üzerinden yapalım?”
        - `Only tables`
        - `Only views`
    
    Seçilen moda göre:
    
    - catalog-service, discovery sırasında:
        - Ya `INFORMATION_SCHEMA.TABLES` benzeri tablo listesi,
        - Ya da `INFORMATION_SCHEMA.VIEWS` benzeri view listesi çekecek.
    
    Bu tercih:
    
    - Kalıcı bir kolon olarak tutulmak zorunda değil,
    - Ama `catalog.datasources` içinde:
        - `object_discovery_mode` (enum `'tables' | 'views'`) olarak saklanabilir.
    
    (Böylece ileride “hem table hem view” desteği açılabilir.)
    

---

- **Adım 4 – DB’den table/view discovery ve seçim**
    
    **Ekran:**
    
    “Tabloları / View’leri Seçin” sayfası.
    
    Akış:
    
    1. UI → `GET /catalog/schema/discover-objects?datasource_id=...&mode=tables|views`
    2. catalog-service:
        - `db-execution-service`’e “list tables” veya “list views” query’si attırır:
            - DB tipine göre uygun sistem view’larını kullanır.
            - Örn. Oracle için `ALL_TABLES` / `ALL_VIEWS`, Postgres için `information_schema.tables` vb.
        - Sonuç:
            - `db_schema_name` (örn. `PUBLIC`, `REPORTING`, `dbo`)
            - `object_name` (“EXPORTS”, “VW_MONTHLY_SALES”)
            - `object_type` (`table` / `view`)
    3. UI bu listeyi gösterir (filter/search ile):
        - Tenant_admin:
            - İşine yarayan object’leri işaretler:
                - Örn. `REPORTING.VW_MONTHLY_EXPORTS`
                - Örn. `DW.SALES_FACT`
    4. UI → `POST /catalog/schema/select-objects`:
        - Body: seçili object’lerin listesi.
    5. catalog-service:
        - Her seçilen object için `catalog.exposed_objects`’a satır yazar:
            - `id` (uuid)
            - `tenant_id`
            - `datasource_id`
            - `db_schema_name`
            - `object_name`
            - `object_type` (table/view)
            - `display_name` (şimdilik `object_name` ile aynı başlatılabilir)
            - `description` (şimdilik NULL)
            - `is_enabled = true`
            - `created_at`, `updated_at`
    
    > Bu adımda kolon yok, sadece hangi tablolar/view’ler “oyuna dahil” olacak onu seçtiriyoruz.
    > 

---

- **Adım 5 – Kolon discovery + PII + semantics**
    
    **Ekran:**
    
    “Kolonları Seçin & İşaretleyin” sayfası.
    
    Her exposed object için:
    
    1. UI → `GET /catalog/schema/discover-columns?exposed_object_id=...`
    2. catalog-service:
        - `db-execution-service` kullanarak ilgili table/view için kolon listesi çıkarır:
            - `column_name`
            - `data_type`
            - `ordinal_position`
            - Mümkünse PK/FK bilgisi.
    3. Bu kolonlar için `catalog.exposed_columns`’a kayıt açar (veya upsert):
        - `id`
        - `tenant_id`
        - `datasource_id`
        - `exposed_object_id`
        - `column_name`
        - `data_type`
        - `logical_type = NULL` (ilk başta)
        - `is_exposed = true` (varsayılan; istersen false da yapabilirsin)
        - `is_pii = false` (varsayılan)
        - `description = NULL`
        - `ordinal_position`
        - `is_primary_key`, `is_foreign_key` (varsa)
        - `created_at`, `updated_at`
    4. UI bu kolon listesini tenant_admin’e gösterir:
        - Her kolon için:
            - “Bu kolon kullanılsın mı?” → checkbox (`is_exposed`)
            - “Bu kolon hassas veri içerir mi (PII)?” → checkbox (`is_pii`)
            - “Kolon türü?” (opsiyonel enum):
                - amount / date / country_code / currency / text / id vs. → `logical_type`
    5. UI seçimleri → `POST /catalog/schema/update-columns`:
        - Body içinde kolonların `exposed_column_id` + `is_exposed`, `is_pii`, `logical_type` değerleri bulunur.
    6. catalog-service bu update’leri `catalog.exposed_columns`’ta işler.
    
    > Burada kolon tiplerini DB’den öğreneceğiz.
    > 
    > 
    > PII ve semantic (logical_type) bilgisini tenant_admin belirledi ve böylece
    > 
    > LLM ve chart-service için çok kritik bir zemin oluştu.
    > 

---

- **Adım 6 – Doğal dil açıklamaları (table/view + kolon)**
    
    **Ekran:**
    
    “Anlamlı İsim & Açıklama” sayfası.
    
    Amaç:
    
    - LLM’in schema’yı insan gibi anlayabilmesi için:
        - Table/view’lere ve kolonlara açıklama yazdırıyoruz.
    
    UI:
    
    - Her exposed object için:
        - `display_name` → “kullanıcı dostu isim” (ör: “Aylık Ülke Bazlı İhracat View’i”)
        - `description` → birkaç cümle:
            - “Her satır, bir ülkenin belirli bir ay için toplam ihracat tutarını ifade eder. Tutar USD cinsindendir…”
    - Her exposed column için:
        - `description`:
            - “İhracat yapılan ülkenin ISO 2 kodu (TR, DE, US…)”
            - “Satış tarihi (fatura tarihi ile aynı).”
        - `logical_type` zaten Adım 5’te seçilmiş olabilir; UI gerekirse burada topluca gösterip düzenletir.
    
    Backend:
    
    - UI → `POST /catalog/schema/update-descriptions`:
        - Body: object ve column ID’leri + display_name/description.
    - catalog-service:
        - `catalog.exposed_objects.display_name`, `catalog.exposed_objects.description` alanlarını günceller.
        - `catalog.exposed_columns.description` alanlarını günceller.
    
    Bu adım bittiğinde:
    
    - Metadata DB’deki **schema semantik olarak etiketlenmiş** olur.
    - `catalog-service`, Kafka’ya event atabilir:
        - `schema.exposed-object.updated`
        - `schema.exposed-column.updated`
    - Embedding-service bu event’leri tüketip:
        - Object ve column açıklamalarını embed edip vector DB’ye yazar.

---

- **Adım 7 – Özet + Onboarding’in tamamlanması**
    
    **Ekran:**
    
    “Özet” sayfası.
    
    - UI, backend’den toplu bir “onboarding summary” çeker:
        - Şirket adı
        - Datasource adı + db_type
        - Kaç object expose edilmiş (table/view)
        - Toplam kaç kolon expose edilmiş, kaç tanesi PII
    - Tenant_admin “Tamamla” butonuna basar.
    
    Backend tarafında:
    
    1. catalog-service:
        - `catalog.tenants.onboarding_status = 'completed'` olarak update eder.
        - Belki `onboarding_completed_at` gibi bir kolon varsa doldurur.
    2. auth-service:
        - Keycloak Admin API ile:
            - Kullanıcının attribute’lerini günceller:
                - `tenant_id = <tenant_id>`
                - `onboarding_pending = false`
    3. Bundan sonra:
        - UI token yenilendiğinde:
            - `onboarding_pending=false` görür,
            - Normal dashboard’a/uygulamaya girebilir.
        - Backend servisleri:
            - Bu tenant için:
                - `catalog.tenants.onboarding_status = 'completed'` olduğu için,
                - `nlp2sql`, `chart`, `scheduler` vb. uçlara artık izin verir.
    4. Vector DB:
        - Onboarding bittikten sonra o tenant için **tüm exposed schema** (seçilmiş table/view + kolon açıklamaları) indexlenmiş olur.
        - Bunu sağlamak için:
            - Onboarding adımları sırasında metadata DB’ye yazıyoruz,
            - Wizard “tamamlandı” noktasında **batch index** tetikleyen bir event çıkarıyoruz,
            - Embedding-service bu event’i tüketerek **o tenant’ın tüm şemasını** Vector DB’ye yüklüyor.