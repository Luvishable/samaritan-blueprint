# 8)DB Execution Service

---

- Tenant DB’de onaylanmış SQL’i güvenli şekilde çalıştırır.
- Doğru tenant + datasource için connection string’i Secret Manager’dan alır.
- Query’yi çalıştırır, satırları stream veya chunk olarak döner.
- Row-limit, timeout, retry gibi kontrolleri yapar.
- Sonuçları nlp2sql service’e döner, kalıcı olarak tutmaz.
- Metadata DB’de kendisine ait schema’sı yoktur.
- Query runtime metriklerini loglamak için event veya HTTP ile bilgi yollayabilir.