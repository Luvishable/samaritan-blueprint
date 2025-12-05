# config schema

---

<aside>
💡

nlp2sql, chart planner vb. için kullanılan prompt şablonlarını ve bazı global ayarları tutmak için oluşturulacak olan schema.

</aside>

- **config.prompt_templates**
    - Her bir LLM kullanımında oluşturulan prompt’un daha sonra incelenebilmesi adına oluşturulacak olan tablo.
    - `id` (uuid, PK)
    - `name` (text)
        
        → Örn:
        
        - `"nlp2sql_main"`
        - `"nlp2sql_feedback"`
        - `"chart_planner"`
        - `"explanation"`
    - `version` (integer)
        
        → Aynı isim için birden çok versiyon tutarsın; son versiyon `is_active=true` olur.
        
    - `description` (text, nullable)
        
        → Template’in ne işe yaradığı, hangi alanlara doldurulduğu.
        
    - `template_text` (text)
        
        → Prompt şablonunun kendisi.
        
        Örnek:
        
        - `"You are a text-to-SQL assistant. Given the user's question: {{question}} and schema: {{schema_snippet}} ..."`
    - `is_active` (boolean)
    - `created_at`, `updated_at`