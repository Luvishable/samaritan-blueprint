# Kafka

---

## 0. Kafka’nın Samaritan’daki Rolü

---

Kafka’yı üç amaçla kullanıyoruz.

- **1. Metadata & Schema değişikliklerini dağıtmak:**
    - Onboarding bittiğinde ve tenant scheması değiştiğinde:
        - embedding-service ve schema-cache-worker gibi servisler bunları asenkron olarak öğreniyor.
- **2. Context değişimlerini dağıtmak:**
    - Yeni business rule eklendiğinde,
    - Kullanıcı tabloyu onayladığında (golden example kaydı),
    - Kullanıcı tabloyu onaylamayıp feedback yazdığında
    
    embedding-service Vektör DB’yi güncelliyor.
    
- **3. Schedular & Notification & Audit tarafında sinyalleşme:**
    - Zamanlanmış işler,
    - e-posta gönderme istekleri
    - query run tamamlandığında oluşan audit/metrics event’leri

## 1. Hangi mikroservis Kafka ile konuşuyor?

---

Kafka ile ilgili ana oyuncular:

- **catalog-service**
- **embedding-service**
- **context-service**
- **nlp2sql-service**
- **scheduler-service**
- **notification-service**
- **schema-cache-worker**
- (dolaylı olarak) **auth-service**, **audit/logging** (bazı event’leri dinleyebilir)

NGINX/API Gateway, UI, db-execution-service gibi servisler Kafka’ya doğrudan dokunmuyor.

## 

## 2. Tenant & Onboarding & Schema event’leri

---

### 2.1. `tenant.onboarding.completed`

- Onboarding Wizard’ın son adımında tenant_admin tamamla butonuna bastığında bu event produce edilir.
    - Producer → `catalog-service`
    - Consumers → `auth-service`, `notification-service`
- **Örnek event:**
    
    ```json
    {
      "event_type": "tenant.onboarding.completed",
      "tenant_id": "tenant-uuid",
      "datasource_ids": ["ds-uuid-1"],
      "completed_at": "2025-12-04T09:30:00Z",
      "triggered_by_user_id": "keycloak-sub-of-tenant-admin"
    }
    ```
    
- **Akış:**
    1. `catalog-service`:
        - `catalog.tenants.onboarding_status = ‘completed’`
        - `onboarding_completed_at = now()`
        - Kafka’y `tenant.onboarding.completed` event’ini yazar.
    2. `auth-service`(consumer):
        - Event’i alır ve Keycloak Admin API ile:
            - Bu *tenant_admin* user’ına `tenant_id` attribute’unu set eder.
            - `onboarding_pending = false` yapar.
        - auth.invitations tablosunda `status = ‘accepted’` olarak günceller.
    3. `notification-service`:
        - İsteğe bağlı olarak:
            - “Hoşgeldiniz, onboarding tamamlandı” gibi bir mail atabilir.
        

### 2.2 `schema.index.requested`

- Onboarding Wizard süreci bittiğinde Metadata DB’deki catalog schemasındaki bilgilerin Vektör DB’ye indekslenmesi için gereken event.
    - Producer → `catalog-service`
    - Consumer → `embedding-service`
- **Örnek event:**
    
    ```json
    {
      "event_type": "schema.index.requested",
      "tenant_id": "tenant-uuid",
      "datasource_ids": ["ds-uuid-1"],
      "reason": "onboarding_completed",
      "requested_at": "2025-12-04T09:30:05Z"
    }
    ```
    
- **Akış:**
    1. `catalog-service`:
        - Onboarding tamamlandığında Kafka’ya `schema.index.requested` event’ini yazar.
    2. `embedding-service` (consumer):
        - Event’i alır.
        - Metadata DB’ye gider:
            - `catalog.exposed_objects` (tenant_id = X, is_enabled = true)
            - `catalog.exposed_columns` (tenant_id = X, is_exposed = true)
        - Her table/view için bir **schema açıklama metni** oluşturur:
            - Object name, display_name, description
            - Kolon adları, tipleri, logical_type, column description
        - Her metin için embedding üretir (OpenAI embedding API veya başka model).
        - Vector DB’de:
            - `schema_objects` koleksiyonuna object-level vektörler,
            - `schema_columns` koleksiyonuna column-level vektörler yazar.
    
    > Böylece Onboarding sonrası, o tenant’ın tüm şeması Vektör DB’de indekslenmiş olur. Soru geldiğinde embedding-service sadece sorunun embedding’ini üretir, schema’yı baştan embed etmez.
    > 

### 2.3 `schema.object.changed` & `schema.column.changed`

- Onboarding sonrasında tenant_admin UI’dan:
    - Yeni table/view expose edebilir,
    - Mevcut object’i disable edebilir,
    - Kolonlarda PII/is_exposed/description değişikliği yapabilir.
- Bu durumda:
    - **Producer:** yine `catalog-service`
    - **Consumers:** `embedding-service`, `schema-cache-worker` (opsiyonel), belki audit/logging
- Örnek event:
    - *object seviyesinde:*
    
    ```json
    {
      "event_type": "schema.object.changed",
      "tenant_id": "tenant-uuid",
      "datasource_id": "ds-uuid-1",
      "exposed_object_id": "obj-uuid",
      "change_type": "updated", // "created" | "deleted"
      "changed_at": "2025-12-05T10:00:00Z"
    }
    ```
    
    - *column seviyesinde:*
        
        ```json
        {
          "event_type": "schema.column.changed",
          "tenant_id": "tenant-uuid",
          "datasource_id": "ds-uuid-1",
          "exposed_object_id": "obj-uuid",
          "exposed_column_id": "col-uuid",
          "change_type": "updated", // "created" | "deleted"
          "changed_at": "2025-12-05T10:05:00Z"
        }
        ```
        
- **Akış:**
    1. catalog-service:
        - Her metadata değişikliğinde ilgili event’i yazar.
    2. embedding-service (consumer):
        - Event’te gelen `exposed_object_id` / `exposed_column_id` ile metadata DB’den *sadece o kaydı* çeker.
        - O object/column için yeni bir embedding üretir.
        - Vector DB’de ilgili vektörü **upsert** eder.
        - Böylece tüm schema’yı tekrar indexlemeye gerek kalmaz.
    3. `schema-cache-worker` (isteğe bağlı consumer):
        - Redis’te tutulan schema cache’i güncellemek için bu event’leri dinleyebilir:
            - `schema.object.changed` geldiğinde:
                - İlgili object’in JSON’unu metadata DB’den yeniden alır → Redis’e yazar.
            - `schema.column.changed` geldiğinde:
                - İlgili object’in Redis JSON’unu update eder veya komple invalid eder.

## 3. Context: Business Rule ve Golden QA event’leri

---

### 3.1. `context.business-rule.changed`

**Ne zaman?**

- Tenant_admin veya yetkili tenant_user:
    - yeni business rule eklediğinde,
    - Var olanı güncellediğinde,
    - Gerekirse sildiğinde.

**Producer:** `context-service`

**Consumer:** `embedding-service`

**Örnek event:**

```json
{
  "event_type": "context.business-rule.changed",
  "tenant_id": "tenant-uuid",
  "business_rule_id": "rule-uuid",
  "change_type": "created", // "updated" | "deleted"
  "changed_at": "2025-12-06T09:15:00Z"
}
```

**Akış:**

1. context-service:
    - `context.business_rules` tablosunda CRUD işlemi yapar.
    - İşlem sonrası Kafka’ya bu event’i yazar.
2. embedding-service:
    - `change_type` `deleted` ise → ilgili vektörü Vector DB’den siler.
    - Aksi halde:
        - `context.business_rules` tablosundan ilgili `business_rule_id`’yi okur:
            - `name`
            - `description` (natural language açıklama)
        - Bu açıklama üzerinden embedding üretir.
        - Vector DB’de `business_rules` koleksiyonuna upsert eder.

### 3.2 `context.golden-qa.created`

**Ne zaman?**

- Kullanıcı tabloyu gördükten sonra “✅ Doğru” butonuna bastığında,
- Bizim kuralımıza göre **bu soru–SQL çifti otomatik “golden” oluyor.**

**Producer:** `nlp2sql-service` 

- `nlp2sql-service`:
    - İlgili `query_run` bilgisini zaten biliyor,
    - “Doğru” feedback event’ini aldıktan sonra:
        - `context.golden_qa` tablosuna insert yapıyor,
        - Kafka’ya event’i yolluyor.

**Consumer:** `embedding-service`

**Örnek event:**

```json
{
  "event_type": "context.golden-qa.created",
  "tenant_id": "tenant-uuid",
  "datasource_id": "ds-uuid-1",
  "golden_qa_id": "golden-uuid",
  "query_run_id": "query-run-uuid",
  "created_by_user_id": "keycloak-sub",
  "created_at": "2025-12-06T10:00:00Z"
}
```

**Akış:**

1. Kullanıcı:
    - UI’da tabloyu görür,
    - “✅ Doğru” butonuna basar.
2. UI → `POST /context/query/{query_run_id}/mark-correct`
3. nlp2sql-service
    - `audit.query_runs`’tan:
        - `question_text`
        - `sql_text`
        - `datasource_id`
            
            gibi bilgileri alır.
            
    - `context.golden_qa` tablosuna yeni satır yazar.
    - Kafka’ya `context.golden-qa.created` event’ini gönderir.
4. embedding-service:
    - `golden_qa_id` ile DB’den:
        - `question_text`
        - `sql_text`
        
        bilgilerini alır.
        
    - Vector DB’de `golden_qa` koleksiyonuna upsert eder.

> Sonuç:
> 
> 
> Kullanıcının “Doğru” dediği her örnek,
> 
> gelecekteki sorularda **referans golden örnek** olarak semantic search’e dahil olur.
> 

 

### 3.3 `context.query-feedback.created`

**Ne zaman?**

- Kullanıcı üretilen tablo için “❌ Yanlış” der,
- Sonra chat üzerinden “Neden yanlış?” feedback’i yazar.

**Producer:** `context-service`

**Consumers:** `embedding-service` (opsiyonel), belki `audit-service`

---

**NOT:** 

Eğer ileride LLM’e bir soru-sql çiftinin neden yanlış olduğunu göndermek istersek embedding-service aksi halde audit-service producer olacak. Şu an için negatif örnek yollanmayacağı için audit-service consumer olacak. 

---

**Örnek event:**

```json
{
  "event_type": "context.query-feedback.created",
  "tenant_id": "tenant-uuid",
  "query_run_id": "query-run-uuid",
  "feedback_id": "feedback-uuid",
  "feedback_type": "business_wrong", // "sql_error" | "performance" | ...
  "created_by_user_id": "keycloak-sub",
  "created_at": "2025-12-06T10:05:00Z"
}
```

**Akış:**

1. Kullanıcı:
    - “❌ Yanlış” butonuna basar.
    - UI chat açar, kullanıcı “Şu ülkeleri içermemeli” vs. açıklama yazar.
2. UI → `POST /context/query/{query_run_id}/feedback`
3. context-service:
    - `context.query_feedback` tablosuna satır yazar.
    - Kafka’ya `context.query-feedback.created` event yazar.
4. embedding-service (opsiyonel):
    - Feedback metnini embed eder,
    - Vector DB’de ayrı bir koleksiyonda (`negative_examples` veya `query_feedback`) saklayabilir.

> Bu sayede ileride:
> 
> - “Bu soru tipi daha önce yanlış üretilmişti, ne hatası vardı?”
>     
>     gibi analizler yapılabilir,
>     
> - Ayrıca fine-tuning/offline evaluation için veri kaynağı olur.

## 4. Query ve Audit event’leri

---

### 4.1`query.run.completed`:

**Ne zaman?**

- Agentic workflow içindeki nlp2sql + db-execution süreci tamamlandığında (başarılı veya başarısız).

**Producer:** `nlp2sql-service`

**Consumers:** `audit-service` (veya direkt audit tablosuna yazan yine nlp2sql), `scheduler-service` (opsiyonel), observability pipeline’ları.

- nlp2sql-service direkt `audit.query_runs` tablosuna yazar, Kafka’ya sadece “bilgilendirme” event’i atar.

**Örnek event:**

```json
{
  "event_type": "query.run.completed",
  "tenant_id": "tenant-uuid",
  "user_id": "keycloak-sub",
  "query_run_id": "query-run-uuid",
  "datasource_id": "ds-uuid-1",
  "status": "succeeded", // or "failed"
  "row_count": 123,
  "error_code": null,
  "error_message": null,
  "started_at": "2025-12-06T10:10:00Z",
  "finished_at": "2025-12-06T10:10:02Z",
  "request_id": "req-uuid"
}
```

**Akış:**

1. nlp2sql-service:
    - LLM + SqlGuard + db-execution pipeline’ını çalıştırır.
    - Sonucu audit tablosuna (`audit.query_runs`) yazar.
    - Kafka’ya `query.run.completed` event’ini atar.
2. `audit/logging`/metrics sistemi (consumer):
    - Bu event’i tiempo serisi metric’lere (Prometheus/Loki vs.) çevirebilir.

---

## 5. Schedular & Notification event’leri

### 5.1 `schedular.job.due`

Alt seviye tasarımda, scheduler-service zaten kendi DB’sine bakarak:

- `scheduler.jobs` tablosunda `next_run_at <= now()` olanları seçip çalıştıracak.

Kafka’ya mutlaka event göndermesi şart değil fakat:

- Her tetiklenecek job için internal bir `scheduler.job.due` event’i üretebilir.

**Producer:** scheduler-service

**Consumers:** scheduler-service’in kendi worker’ı (aynı servis), belki başka analitik servisler.

### 5.2 `scheduler.job.completed` / `scheduler.job.failed`

- Bir job çalışıp:
    - Query çalıştı,
    - Excel/PDF üretildi,
    - Mail gönderildi
        
        vs. tamamlandığında veya hata aldığında loglamak.
        

**Producer:** `scheduler-service`

**Consumers:** `notification-service , audit/logging`.

**Örnek event:**

```json
{
  "event_type": "scheduler.job.completed",
  "tenant_id": "tenant-uuid",
  "job_id": "job-uuid",
  "favorite_query_id": "fav-uuid",
  "run_at": "2025-12-07T07:00:00Z",
  "status": "succeeded",
  "delivery_channel": "email",
  "delivery_format": "excel",
  "request_id": "req-uuid"
}
```

veya hata durumunda:

```json
{
  "event_type": "scheduler.job.failed",
  "tenant_id": "tenant-uuid",
  "job_id": "job-uuid",
  "favorite_query_id": "fav-uuid",
  "run_at": "2025-12-07T07:00:00Z",
  "status": "failed",
  "error_message": "DB timeout",
  "request_id": "req-uuid"
}
```

### 5.3. `notification.email.requested` & `notification.email.sent/failed`

Mail gönderimi için Kafka kullanımı:

- Sorgu sonucunu PDF/Excel/export edip gönderme,
- Davet mailleri,
- Onboarding tamamlandı maili,
- Alarm maili (scheduler)
    
    gibi tüm mail işleri için ortak pattern:
    

**Producer:** `scheduler-service`, `auth-service`

**Consumer:** `notification-service`

**email.requested örneği:**

```json
{
  "event_type": "notification.email.requested",
  "tenant_id": "tenant-uuid",
  "to": "user@example.com",
  "subject": "Aylık ihracat raporunuz hazır",
  "body_template": "monthly_export_report",
  "body_params": {
    "month": "2025-11",
    "country_count": 12
  },
  "attachments": [
    {
      "type": "excel",
      "storage_url": "s3://.../tenant-uuid/job-uuid/result.xlsx"
    }
  ],
  "requested_by": "scheduler",
  "requested_at": "2025-12-07T07:00:01Z"
}
```

**Akış:**

1. scheduler-service:
    - Job çalıştı,
    - Query sonucu + export oluşturuldu,
    - Storage’a (S3 / GCS / local) kaydedildi.
    - Kafka’ya `notification.email.requested` event’i yazar.
2. notification-service:
    - Event’i okur,
    - Uygun mail provider üzerinden (SMTP / SendGrid / Mailgun / GMail API / free seçenekler) maili gönderir.
    - Sonuç başarılı/başarısız olursa:
        - `notification.email.sent` veya `notification.email.failed` event’ini yayınlayabilir.

Bu sayede:

- Mail provider değiştirmek,
- Mail işini başka bir queue’ya taşımak,
- Retry mekanizmaları eklemek
    
    son derece kolaylaşır.