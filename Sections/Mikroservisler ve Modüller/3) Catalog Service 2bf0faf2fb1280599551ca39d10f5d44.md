# 3) Catalog Service

- Her tenant için:
    - Hangi DB tipi(Oracle, Postgres vb.) kullanılıyor,
    - Hangi connection secret kullanılıyor (Vault referansı, string’in kendisi değil)
    - Tenant “tablolarla mı viewlerle mi çalışacak” tercihi
    - Hangi tablolar ve view’lerin exposed edileceği
    - Hangi kolonlar PII veya sensitive
    - Tablo/view/kolonların doğal dildeki açıklamaları
    - Kolonların tip bilgileri

gibi tüm bilgileri Metadata DB’mizde kendisine ait şemadaki tenants tablosunda tutar.

- Onboarding Wizard’ın bütün API’leri catalog-service üzerinden döner.
    - connection string kaydetme
    - table/view discovery
    - kolon discovery
    - PII flag’leri
    - Açıklama alanları