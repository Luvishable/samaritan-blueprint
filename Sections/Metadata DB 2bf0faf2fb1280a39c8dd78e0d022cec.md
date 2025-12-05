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

[catalog schema](Metadata%20DB/catalog%20schema%202bf0faf2fb1280f2b3f1c982245c6513.md)

[context schema](Metadata%20DB/context%20schema%202bf0faf2fb128007b245f188a7e5cf41.md)

[schedular schema](Metadata%20DB/schedular%20schema%202bf0faf2fb1280f6b02ddb0c65068aa8.md)

[audit schema](Metadata%20DB/audit%20schema%202bf0faf2fb12802e8d6ffd8ebd5379d0.md)

[config schema](Metadata%20DB/config%20schema%202bf0faf2fb128045b46ac2cfef2a20a3.md)

[auth schema](Metadata%20DB/auth%20schema%202bf0faf2fb128026ad4ffebbe3cb2086.md)