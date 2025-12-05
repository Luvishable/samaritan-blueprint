# audit schema

---

<aside>
💡

“Kim, ne zaman, hangi soruyu sordu, hangi SQL çalıştı, ne sonuç geldi, hangi LLM prompt’u gitti?” gibi soruların cevabı burada.

</aside>

- **audit.query_runs**
    - Her soru çalıştırma denemesi için bir kayıt
    - `id` (uuid, PK)
    - `tenant_id` (uuid)
    - `user_id` (text)
        
        → Bu query’i kim tetikledi?
        
    - `datasource_id` (uuid, FK → `catalog.datasources.id`)
    - `question_text` (text)
        
        → Kullanıcının doğal dildeki sorusu.
        
    - `sql_text` (text, nullable)
        
        → LLM’in ürettiği SQL.
        
        Hata aldıysa bile kaydedebiliriz.
        
    - `status` (enum: `'succeeded' | 'failed' | 'cancelled'`)
    - `error_code` (text, nullable)
    - `error_message` (text, nullable)
        
        → DB’den gelen hata veya SqlGuard’tan gelen uyarı.
        
    - `row_count` (integer, nullable)
        
        → Kaç satır döndü?
        
    - `started_at` (timestamptz)
    - `finished_at` (timestamptz)
    - `duration_ms` (integer, nullable)
    - `request_id` (text)
        
        → Gateway’in her isteğe vereceği global ID.
        
        Log’ları mikroservisler arasında korele etmeye yarar.
        
    - `llm_session_id` (uuid, nullable)
        
        → İlgili `prompt_runs` ile ilişki kurmak için.
        
    
    > context.favorite_queries ve context.query_feedback bu tablodaki id’lere FK veriyor.
    > 
    > 
    > Böylece “hangi run favorilendi, hangi run’a feedback geldi?” net.
    > 
- **audit.prompt_runs**
    - LLM ile yapılan her etkileşim (agent node’larının çağrısı, context oluşturma vb.) için tutulan kayıt.
    - `id` (uuid, PK)
    - `tenant_id` (uuid, nullable olabilir)
        
        → Bazı admin/system çağrıları tenant dışı olabilir ama çoğu tenant scoped.
        
    - `user_id` (text, nullable)
        
        → Hangi user tetikledi (genellikle dolu, ama background işler için boş olabilir).
        
    - `model_name` (text)
        
        → Örn. `"gpt-4.1"`, `"deepseek-coder"`, `"local-llm-v1"`.
        
    - `prompt_template_name` (text)
        
        → `config.prompt_templates.name` ile eşleşir (örn. `"nlp2sql_base"`, `"nlp2sql_feedback"`, `"chart_planner"`).
        
    - `prompt_template_version` (integer)
        
        → Templates versioning için.
        
    - `rendered_prompt` (text)
        
        → LLM’e gönderilen *gerçek* prompt.
        
        Debug & audit için önemli (gerekirse truncate veya mask yaparız).
        
    - `role` (text)
        
        → Bu prompt’un amacı:
        
        - `nlp2sql_main`, `nlp2sql_feedback`, `chart_planner`, `explanation` vs.
    - `input_token_count` (integer, nullable)
    - `output_token_count` (integer, nullable)
    - `created_at` (timestamptz)
    - `request_id` (text)
        
        → `query_runs.request_id` ile aynı ID, trace için.
        
    
    > Böylece: “Bu query’nin arkasında hangi prompt gitmiş, hangi model, kaç token, ne cevap almışız?” hepsi audit’te görülebilir.
    >