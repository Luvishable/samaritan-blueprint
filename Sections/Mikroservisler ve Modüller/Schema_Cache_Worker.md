# 13) Schema Cache Worker

---

- Metadata DB’deki schema bilgilerini Redis cache ile senkron tutmak:
    - catalog.exposed_objects ve catalog.exposed_columns üzerinde değişiklik olduğunda event alır.
    - Redis’te tenant + datasource bazlı exposed_schema json’unu tutar.
- context service veya nlp2sql service schema bilgisine ihtiyaç duyduğunda:
    - Önce redis’ten alır
    - miss olursa metadata DB’den okuyup Redis’e yazılmasını tetikler