# context schema

---

<aside>
💡

LLM’e giden prompt’u zenginleştirmek için gerekli semantic context’i ve kullanıcı etkileşimlerini tutmak. 

</aside>

- **context.business_rules**
    - Tenant’ın iş kurallarını doğal dil ile saklar.
    - `id` (uuid, PK)
    - `tenant_id` (uuid, FK → `catalog.tenants.id`)
    - `datasource_id` (uuid, FK → `catalog.datasources.id`, nullable olabilir)
        
        → Kural spesifik bir datasource’a mı ait, yoksa tenant genel kuralı mı?
        
    - `name` (text)
        
        → Kısa başlık (“Cancel edilen siparişleri hariç tut”, “İade kayıtlarını toplamlara dahil etme” gibi).
        
    - `description` (text)
        
        → Doğal dil açıklama, LLM context’ini bununla zenginleştiriyoruz.
        
    - `is_active` (boolean)
    - `created_by_user_id` (text)
        
        → Kim ekledi?
        
    - `created_at`, `updated_at`
    
    > LLM prompt’una giderken:
    > 
    > 
    > Temel olarak `name` + `description` kullanıyoruz.
    > 
    
- **context.golden_qa**
    - Kullanıcı, sorusuna karşılık üretilen tablonun doğruluğunu onaylarsa LLM doğru bir şekilde SQL üretmiş demektir. Durum böyle olunca kullanıcının da doğru feedback’inin de onayıyla sorulan soru + üretilen sql çiftini ileride LLM’e eğer aynı veya benzer soru gelirse context zenginleştirmesi amacıyla kayıt altına alıyoruz.
    - DB’ye kaydedildikten sonta üretilen event ile embedding service’in de bu event’i tüketmesi ile bu golden example aynı zamanda Vektör DB’ye de kaydedilir.
    - tenant_admin ve onun yetki verdiği tenant_user’lar gerek onboarding wizard aşamasında gerekse de uygulamayı başka zamanlarda kullanırken golden example çifti kaydedebilirler.
    - `id` (uuid, PK)
    - `tenant_id` (uuid)
    - `datasource_id` (uuid)
    - `question_text` (text)
        
        → Kullanıcının sorduğu doğal dildeki soru.
        
    - `sql_text` (text)
        
        → LLM’in ürettiği ve doğru olduğu kabul edilen SQL.
        
    - `created_by_user_id` (text, nullable)
        
        → Kaynağı:
        
        - Kullanıcı onayı ile `tenant_user` / `tenant_admin`,
        - Seed data ise `system_admin` vb.
    - `source` (enum: `'user_verified' | 'system_seed' | 'manual'`, vs.)
        
        → Bu golden’ın nereden geldiği.
        
    - `created_at`
- **context.favorite_queries**
    - Kullanıcının favorilediği sorgular.
    - Schedular buradan referans alır.
    - `id` (uuid, PK)
    - `tenant_id` (uuid)
    - `user_id` (text; Keycloak `sub`)
        
        → Bu favori kimin?
        
    - `query_run_id` (uuid, FK → `audit.query_runs.id`)
        
        → Hangi query çalışmasının sonucunu favorilemiş?
        
    - `name` (text, nullable)
        
        → Kullanıcı favoriye özel isim verebilir (“Aylık ülke bazlı ihracat” gibi).
        
    - `created_at`
    - `is_active` (boolean)
        
        → Favori listeden kaldırmak için flag.
        
- **context.query_feedback**
    
    Kullanıcı ❌ bastığında, neden yanlış olduğunu açıklayan metinleri saklıyoruz; bunlar LLM için “negatif örnek” gibi.
    
    **Kolonlar:**
    
    - `id` (uuid, PK)
    - `tenant_id` (uuid)
    - `user_id` (text)
    - `query_run_id` (uuid, FK → `audit.query_runs.id`)
        
        → Hangi query’ye feedback verildi?
        
    - `feedback_type` (enum: `'business_wrong' | 'performance' | 'other'`)
        
        → Hata türü:
        
        - İş mantığı yanlış,
        - Çok yavaş geldi,
        - vb.
    - `feedback_text` (text)
        
        → Kullanıcının serbest metin açıklaması.
        
    - `created_at`
    
    > Bu tabloda tutulan açıklamalar da embedding-service tarafından vektör DB’ye atılabilir (ileride negative examples / reranking için).
    > 
    
    ---