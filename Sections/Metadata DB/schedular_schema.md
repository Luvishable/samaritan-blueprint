# schedular schema

---

<aside>
💡

Favorilenen sorguların kullanıcının talep ettiği sıklıkta ve zamanda kullanıcıya mail olarak gönderilmesini sağlayan service’e ait database.

</aside>

- **schedular.jobs**
    - `id` (uuid, PK)
    - `tenant_id` (uuid)
        
        → Job hangi tenant’a ait?
        
    - `user_id` (text)
        
        → Job’u kim oluşturdu?
        
    - `favorite_query_id` (uuid, FK → `context.favorite_queries.id`)
        
        → Hangi favori sorgu düzenli çalıştırılacak?
        
    - `schedule_type` (enum: `'cron' | 'interval' | 'once'`)
        
        → Cron string mi, belli aralık mı, tek seferlik mi?
        
    - `cron_expression` (text, nullable)
        
        → `schedule_type='cron'` ise dolu.
        
    - `interval_minutes` (integer, nullable)
        
        → `schedule_type='interval'` ise dolu.
        
    - `time_zone` (text)
        
        → Job hangi timezone’a göre çalışacak? (Örn. `Europe/Istanbul`)
        
    - `next_run_at` (timestamptz)
        
        → Bir sonraki çalıştırma zamanı.
        
        → Worker, `WHERE next_run_at <= now()` ile sıradaki job’ları seçiyor.
        
    - `last_run_at` (timestamptz, nullable)
    - `is_active` (boolean)
        
        → Job pause/resume için.
        
    - `delivery_channel` (enum: `'email'` vs., şimdilik neredeyse hep `'email'`)
    - `delivery_format` (enum: `'table' | 'excel' | 'pdf'`)
        
        → Sonucun nasıl gönderileceği.
        
    - `created_at`, `updated_at`
    
    > job_runs gibi bir tabloyu ileride ekleyebiliriz (her çalışmanın log’u).
    >