# 10) Schedular Service

---

- Favori bazlı zamanlanmış sorgular için:
    - Kullanıcı bir alarm/schedule tanımlar
    - schedular_jobs tablosuna yazılır
    - Background worker next_run_at alanına göre job’ları tetikler
- Job zamanı geldiğinde:
    - İlgili favorite_query sorgusunu çalıştırmak için nlp2sql ve db execution service’lerini tetikler veya daha basit ve uygulamamızda da kullanacağımız gibi daha önceden favorilenmiş stored sql’i direkt çalıştırır.
    - Export işlemleri (PDF/Excel) ve mail gönderimi için notification service’i çağırır.
- Metadata DB’de schedular schema’sının sahibidir.