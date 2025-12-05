# PROJECT SAMARITAN

> *Samaritan*: Yardımsever*.* 

Şirketlerin; iş süreçlerinde hızlı ama tutarlılıktan taviz vermeden, verilerine doğal dilde sordukları sorular ile erişebilmelerini ve sonuçları daha iyi okuyabilmeleri için veriyi modern araçlarla görselleştiren ve en önemlisi de tüm bu süreçlerde hiçbir 3. parti yazılıma veri sızdırmayan, **AI-native** bir proje.
> 

---

### Hedef Kullanıcılar ve Kullanım Senaryoları:

- İş geliştirme ve izleme süreçlerinde gerek çalışanların gerekse yöneticilerin database üzerinden verilere erişebilmesi için SQL bağımlılığını ortadan kaldıran bu uygulama sayesinde her türlü ihtiyaç veya alınacak aksiyon daha çabuk karara bağlanacaktır.
- İstenmeyen gidişat erken fark edilecek ve böylece finansal zararlar minimize edilecektir.
- Kısacası 21. yüzyılın petrolü olan dataya erişimi kolaylaştıran ve verimliliği artırmaya yardımcı olacak bir araç.
- Dinamik, modüler, esnek ve genişletilebilir yapısı sayesinde database’lere bağlanmayı ve veri onlar üzerinden çıkarımlarda bulunmayı isteyen gerek yönetici, işveren ve daha bilimum pozisyondaki çalışanın iş süreçlerini kolaylaştıran bir yardımcı.

---

### Ana Özellikler:

- Kullanıcıların database ile kendi ana dillerinde doğal bir şekilde sohbet ederek sonuçları görüntüleyebilmelerini sağlar.
- Kullanıcının doğal dilde sorduğu soru, LLM sayesinde database’den o soruya karşılık gelen sonucun çekilebilmesi için SQL koduna dönüştürülür.
- Üretilen SQL kullanıcının bağlı veya sahip olduğu database üzerinde çalıştırılır ve sonuç tablo formatında dönülür.
- Kullanıcı uygulama ile bir chat sayesinde iletişime geçer ve üretilen tablo üzerinden ekstra bir talepte bulunmak istediğinde yine chat aracılığıyla talebini iletir.
- Üretilen tablonun doğruluğu kullanıcı tarafından onaylandıktan sonra grafik çizdirmek için kullanıcı chat’e talebini iletebilir. Fakat kullanıcının arzu etmediği bir sonucun üretilmesi durumunda kullanıcının feedback vermesi zorunlu kılınır ve verdiği feedback ile sonucun doğru hale getirilmesi amacıyla LLM tekrar devreye girer.
- Kullanıcılar uygulama düzeyinde iki temel role’a sahip olacaklardır: **tenant_admin** ve **tenant_user.**
- System admini tarafından uygulamayı kullanması için bir sign-up linki ile davet edilen tenant_admin’ler kendi şirketlerine ve kullanacakları database’e ait bilgileri girdikten sonra isterlerse kendi şirketlerine ait diğer kullanıcıları da tenant_admin veya tenant_user olarak ekleyebilirler.
- Sisteme kaydolma aşamasında tenant_admin ***onboarding wizard*** ***(kurulum sihirbazı)*** adlı bir aşamadan geçer ve bu aşama sayesinde kullanacakları database/ler’in hakkında talep edilen bilgileri doldurur ve uygulamanın database’de read-only yetkiye sahip connection string’ini set eder.
- Gerek onboarding wizard gerekse de istenilen herhangi bir zamanda tenant_admin veya onun yetki verdiği tenant_user’lar business rule adında, LLM’in daha doğru SQL üretebilmesini sağlayacak iş kuralları veya tanımları set edebilirler.
- Onboarding wizard sırasında tenant_admin database’lerinde erişimin table mı yoksa view’ler üzerinden mi olacağını başlangıçta seçer ve program buna göre bir table/view listesi getirir. Daha sonra tenant_admin’den, bu table/view ve sahip oldukları column’lara iş mantığının güçlendirilmesi açısından kısa açıklamalar girmeleri istenir. Fakat database’in üretim aşamasında *CREATE TABLE* veya *CREATE VIEW* komutları sırasında bu table/view’lere ait açıklamalar girilmişse program bunu otomatik görür ve tenant_admin’in onayına sunar.
- Uzun vadede daha doğru ve tutarlı sonuçların üretilmesi için kullanıcının doğru onayı verdiği her ***soru-sql*** çifti arka planda uygulamamızın kendi database’i (bu database *metadata DB* ile anılacaktır ileriki bölümlerde) ve yine uygulamamıza ait vektör database’imize kaydedilecektir.
- Authentication tamamen **Keycloak** üzerinden yönetilecek olup **Multi-Factor Authentication (MFA)** prensibi uygulanacaktır. Böylece kullanıcıların sisteme kaydolması ve giriş yapması en modern güvenlik prensipleri gözetilerek sağlanacaktır.
- Database’lere ait connection string’ler encrypte edildikten sonra bulut ortamındaki Secrets Manager’larda tutulacaktır. Dolayısıyla database üzerinde sadece Secrets Manager’daki ID bilgileri tutulacaktır. Buna ek olarak uygulamanın zaten sadece veri okuma (read-only) düzeyinde database’de yetki sahibi olması da güvenliği artıran bir diğer unsurdur.
- **LLM aracılığıyla SQL, Grafik ve Çıkarım üretiminde hiçbir satır verisi (row data) LLM’e verilmez. Grafik ve insight üretimi tamamen deterministik şekilde backend tarafında yapılır.**
- Başlıklarda bu özelliklerin tümü detaylandırılacaktır.

---

### Mimari ve Uygulamada Kullanılan Araçlar

[Mikroservisler ve Modüller](PROJECT%20SAMARITAN/Mikroservisler%20ve%20Mod%C3%BCller%202be0faf2fb1280a3bb5cc257b73472ff.md)

[Rate Limit Uygulanması](PROJECT%20SAMARITAN/Rate%20Limit%20Uygulanmas%C4%B1%202bf0faf2fb1280c5b661f2fcbc881feb.md)

[Metadata DB](PROJECT%20SAMARITAN/Metadata%20DB%202bf0faf2fb1280a39c8dd78e0d022cec.md)

[Redis](PROJECT%20SAMARITAN/Redis%202bf0faf2fb128008b2ebee6f5e0a8b56.md)

[Onboarding Wizard ](PROJECT%20SAMARITAN/Onboarding%20Wizard%202bf0faf2fb128015a337ed92ede543e3.md)

[Kafka](PROJECT%20SAMARITAN/Kafka%202bf0faf2fb1280728f04c6f0afb183be.md)

---

### Agentic Workflow

---

### Export Stratejisi

---

### Metrics & Audit & Observability

---

---

### CI / CD Entegrasyonu

---

### Deployment

---