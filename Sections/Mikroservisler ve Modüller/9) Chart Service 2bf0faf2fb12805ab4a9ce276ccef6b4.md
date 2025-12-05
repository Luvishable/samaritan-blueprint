# 9) Chart Service

---

- SQL sonucu oluşturulan tablodan grafik planı ve Plotly spec’i üretir.
- ChartPlannerAgent:
    - Kullanıcının chart isteği + tablo schema’sı + orijinal soruyu LLM’e yollar ve `ChartPlan` JSON’unu üretir.
- ChartRuntime:
    - ChartPlan’i valide eder (kolon var mı, tipler uyumlu mu)
    - Daha önceden yazılmış, farklı sayısal değerler üzerinden farklı grafikler oluşturulabilmesini sağlayan fonksiyonlar ile Plotly figure spec’i üretir.
- UI’a hazı plotly figure JSON’u döner
- Metadata DB’de kendisine ait schema’sı yoktur.