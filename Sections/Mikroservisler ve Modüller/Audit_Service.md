# 12) Audit Service

---

- Uygulama bazında tüm önemli aksiyonları tek yerde loglar:
    - Hangi kullanıcı hangi tenant için hangi soruyu sormuş
    - Hangi SQL çalışmış, kaç satır dönmüş, ne kadar sürmüş
    - Hangi prompt hangi LLM modeline gitmiş, ne cevap gelmiş
- Hem debugging hem de compliance (uyumluluk) için loglama yapar.
- Metadata DB’deki audit scheması’nın sahibidir.