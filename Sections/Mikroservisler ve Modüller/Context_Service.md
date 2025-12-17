# 4) Context Service

---

- LLM’e giden prompt’u zenginleştirmek için gereken domain bilgisini yönetir.
    - Business rules
    - Golden examples(soru-sql çiftleri)
    - Kullanıcı feedback’leri
    - Favori sorgular
- nlp2sql-Service tarafından çağrıldığında tenant + soru bağlamına göre:
    - Vektör DB’den:
        - ilgili *schema parçaları (*table/view/column)
        - İlgili golden examples’lar
        - Ve ilgili business rule’ları bulur ve getirir.
- Metadata DB’sindeki context schemasının sahibidir.
- Önemli tablolar:
    - `context.business_rules`
    - `context.golden_qa`
    - `context.query_feedback`
    - `context.favorite_queries`