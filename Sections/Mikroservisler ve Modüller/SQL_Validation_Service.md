# 7) SQL Validation Service

---

- LLM’in ürettiği SQL’in tenant DB’de çalışmasından önce kontrol eder.
- Görevleri:
    - Sadece izin verilen verb’leri onaylar(Bizim durumumuzda sadece SELECT)
    - DROP/DELETE/UPDATE yok
    - Tenant DB sınırları dışına çıkan cross-db sorgu yok
    - Limit, timeout gibi guardrail’ler de uygulanır
    - Potansiyel riskli pattern’ları belirleyip reject etmek
    - Metadata DB’de kendisine ait schema yoktur.