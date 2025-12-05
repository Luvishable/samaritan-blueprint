# Metadata DB

---

> Samaritan adlı uygulamamızın kendisine ait PostgreSQL database’idir.
> 

## 1) Genel Tasarım

- Veritabanı adı: samaritan_core (PostgreSQL)
- Schema’lar:
    - catalog → tenant + datasource + exposed schema (table/column/view)
    - context → Business kuralları, golden examples, feedback, favoriler
    - schedular → Zamanlanmış işler
    - audit → Query & prompt çalışma kayıtları
    - config → prompt template’leri ve global ayarlar
    - auth → davet kayıtları
- Her tabloda mümkün olduğunca:
    - `tenant_id` olacak (multi-tenant izolasyonu için)
    - created_at, updatede_at → versiyon ve izleme için
    - Bazı tablolarda user_id olacak → hangi kullanıcı yaptı sorusunun cevabı için

## 2) Schema’lar

---

[catalog schema](Metadata%20DB/catalog_schema.md)

[context schema](Metadata%20DB/context_schema.md)

[schedular schema](Metadata%20DB/schedular_schema.md)

[audit schema](Metadata%20DB/audit_schema.md)

[config schema](Metadata%20DB/config_schema.md)

[auth schema](Metadata%20DB/auth_schema.md)