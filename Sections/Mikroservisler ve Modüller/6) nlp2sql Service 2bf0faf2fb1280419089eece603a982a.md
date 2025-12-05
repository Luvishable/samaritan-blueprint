# 6) nlp2sql Service

---

> Langgraph tabanlı agentic workflow’un çalıştığı esas servis. 
Uygulamanın beyin kısmı.
Diğer servisler, bu servisin ihtiyaç duyduğu görevleri icra etmek için varlar.
> 

AKIŞ:

- Kullanıcı sorusunu alır
- Context service + embedding service ikilisi ile gereken context zenginleştirmesini yapar.
- İçindeki agent’lardan birinin HTTP çağrısı ile LLM’den SQL üretir.
- Bir diğer agent üretilen SQL’in validasyonunu yapar.
- Başka bir agent onaylanan SQL’i database’de çalıştırır.
- Sonucu UI’a gönderir ve graph state’i (langgraph’ın memory’si) “feedback bekliyor” moduna alır.
- Gelen feedback’e göre ya golden example üretir ya da LLM’e kullanıcıdan gelen feedback veya tenant’ın DB’sindeki hata kodu ile tekrar çağrıda bulunur.
- Metadata DB’mizde kendisine ait bir schema yoktur; sadece okuyucudur.
- Audit log için audit service ile koordineli çalışır.
- Kullanıcının grafik çizdirme talebine karşılık chart service ile de konuşur.