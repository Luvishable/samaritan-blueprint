# catalog schema

<aside>
💡

Tenant’ın şirket bilgileri, bağlı olduğu veri kaynakları ve “Samaritan’a açılmış” tablo/view/kolon listesini tutmak.
Tenant DB’den gelen *gerçek data* yok, sadece “ne var ve hangilerini kullanabiliriz” bilgisi var.

</aside>

- **catalog.tenants**
    - Samaritan’daki her müşteri için bir kayıt
    - `id` (uuid, PK)
        
        → Tenant’ın ana kimliği. Tüm diğer tablolarda `tenant_id` buraya referans.
        
    - `name` (text)
        
        → Şirket adı (UI’da görünen isim).
        
    - `slug` (text, unique, nullable)
        
        → URL veya dahili isimlendirme için kısa kod. (Zorunlu değil ama faydalı.)
        
    - `country` (text / iso code)
        
        → İleride raporlamada veya lokalizasyon için kullanılabilir.
        
    - `sector` (text, nullable)
        
        → Sektör (ihracat, lojistik, gıda vs.)
        
    - `created_at` (timestamptz)
    - `created_by_user_id` (text, nullable)
        
        → Bu tenant’ı kim oluşturdu (genelde `tenant_admin` veya `system_admin`).
        
    - `is_active` (boolean)
        
        → Tenant dondurulduğunda, uygulamada pasif hale getirilebilir.
        
    
    > Not:
    > 
    > 
    > Şimdilik **plan_tier** gibi bir tablo yok. 
    > 
    
     
    
- **catalog.datasources**
    - Bir tenant’ın bağlandığı veri kaynaklarını (DB connection’larını) temsil ediyor.
    - Ör: bir tenant’ın Oracle imalat ve Postgres rapor adlı iki tane DB’si olabilir.
    - `id` (uuid, PK)
    - `tenant_id` (uuid, FK → `catalog.tenants.id`)
        
        → Bu datasource hangi tenant’a ait?
        
    - `name` (text)
        
        → UI’da gösterilen “Data Source Name” (örn. “ERP Oracle Production”).
        
    - `db_type` (text / enum)
        
        → Örn. `'postgres'`, `'oracle'`, `'sqlserver'`…
        
        LLM tarafında SQL dialekt seçimi için de kullanılacak.
        
    - `connection_secret_ref` (text)
        
        → Direkt connection string **TUTMUYORUZ**.
        
        Bunun yerine:
        
        - Vault / GCP Secret Manager’daki secret’ın adı/id’sini tutuyoruz.
        - `db-execution-service` bu referansla gerçek connection string’i çekiyor.
    - `default` (boolean)
        
        → Tenant’ın varsayılan data source’u mu?
        
    - `is_active` (boolean)
        
        → Bu data source kullanıma açık mı?
        
    - `created_at`, `updated_at` (timestamptz)
- **catalog.exposed_objects**
    - Onboarding wizard sırasında tenant_admin’in “Samaritan’da kullanılabilir olsun” dediği tablo/view’leri temsil ediyor.
    - `id` (uuid, PK)
    - `tenant_id` (uuid, FK → `catalog.tenants.id`)
    - `datasource_id` (uuid, FK → `catalog.datasources.id`)
    - `db_schema_name` (text)
        
        → Tenant DB içindeki schema (örn. `"public"`, `"dbo"`, `"REPORTING"`).
        
    - `object_name` (text)
        
        → DB’deki gerçek table/view ismi (örn. `"EXPORTS"`, `"ORDERS"`, `"VW_MONTHLY_SALES"`).
        
    - `object_type` (enum: `'table' | 'view'`)
        
        → Onboarding’de “tablo mu, view mi kullanılsın?” tercihini buradan ayırt ediyoruz.
        
        *“ya tablo ya view kullanılacak” kararını aldık ama ileride esnetilebilir; ve kolon buna hazır.*
        
    - `display_name` (text, nullable)
        
        → UI için daha anlamlı isim (“Aylık İhracat”, “Siparişler Tablosu” gibi).
        
    - `description` (text, nullable)
        
        → Onboarding wizard’da tenant_admin’in yazdığı doğal dil açıklama.
        
        Bu açıklama, **vector DB’ye embedding** için gönderiliyor.
        
    - `is_enabled` (boolean)
        
        → Tenant bu object’i geçici olarak devre dışı bırakmak isterse.
        
    - `created_at`, `updated_at`
    
    **İlişki:**
    
    - 1 tenant → N datasource
    - 1 datasource → N exposed_object
    - 1 exposed_object → N exposed_column
- **catalog.exposed_columns**
    - exposed_objects içindeki kolonların detayları ve hangi kolonların LLM’e açıldığının kaydı tutuluyor.
    - `id` (uuid, PK)
    - `tenant_id` (uuid, FK → `catalog.tenants.id`)
    - `datasource_id` (uuid, FK → `catalog.datasources.id`)
    - `exposed_object_id` (uuid, FK → `catalog.exposed_objects.id`)
    - `column_name` (text)
        
        → DB’deki gerçek kolon ismi.
        
    - `data_type` (text)
        
        → DB’nin tipini raw şekilde saklıyoruz (örn. `"NUMBER"`, `"VARCHAR2(100)"`, `"DATE"`, `"NUMERIC(18,2)"`).
        
        `db-execution-service` için doğru SQL üretiminde kullanışlı.
        
    - `logical_type` (text / enum, nullable)
        
        → Tenant_admin onboarding’de isterse seçebilir:
        
        - `"amount"`, `"date"`, `"country_code"`, `"currency"`, `"percentage"` gibi…
            
            Bu, chart-service ve LLM için *semantik ipucu*.
            
    - `is_exposed` (boolean)
        
        → Bu kolon LLM ve UI tarafından kullanılabilir mi?
        
        Bazı kolonlar (internal id vs.) sadece teknik amaçlı olabilir.
        
    - `is_pii` (boolean)
        
        → “Bu kolon hassas veri içeriyor mu?”
        
        Onboarding sırasında tenant_admin işaretliyor.
        
        LLM context’ine nasıl yansıtılacağına (mask, aggregate only vs.) biz karar veririz.
        
    - `description` (text, nullable)
        
        → Kolonun doğal dil açıklaması (“Gönderici ülke ISO kodu”, “Faturanın KDV’siz tutarı” gibi).
        
        Bu açıklamalar da vector DB’ye embedding için gidiyor.
        
    - `ordinal_position` (integer)
        
        → Tablo içindeki sırası. UI’da kolon sıralaması için işe yarar.
        
    - `is_primary_key` (boolean, nullable)
    - `is_foreign_key` (boolean, nullable)
        
        → Discovery sırasında tespit edebilirsek, LLM’e relation hint’i vermek için.
        
    - `created_at`, `updated_at`
    
    **Önemli Not:**
    
    Redis cache’inde tutulan “schema JSON” tam olarak bu **exposed_objects + exposed_columns** birleşiminin sıkıştırılmış hali.
    
    Vector DB’ye giden embedding’ler ise `description` alanlarından geliyor.