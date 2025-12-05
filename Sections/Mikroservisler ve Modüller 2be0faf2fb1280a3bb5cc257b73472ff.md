# Mikroservisler ve Modüller

## Genel Bakış

---

<aside>
💡

Multi-tenant, mikroservis tabanlı, event-driven yapıya sahip olacak uygulamamızda, her servisin kendisine ait sorumluluğu olacak ve ***servisler arası async iletişim için Kafka*** kullanılacaktır. 

</aside>

- Servislerin hepsi *docker container* olarak çalışacak.
- Servislerdeki data Metadata DB’deki schema’larda tutulacak. Başlangıçta her bir servis için ayrı database olmayacak.

[1) Api - Gateway (Nginx)](Mikroservisler%20ve%20Mod%C3%BCller/1)%20Api%20-%20Gateway%20(Nginx)%202be0faf2fb1280a58335e199655e01ed.md)

[2) Keycloak + Auth Service ](Mikroservisler%20ve%20Mod%C3%BCller/2)%20Keycloak%20+%20Auth%20Service%202be0faf2fb128087a4c0cf5ea3418b2f.md)

[3) Catalog Service](Mikroservisler%20ve%20Mod%C3%BCller/3)%20Catalog%20Service%202bf0faf2fb1280599551ca39d10f5d44.md)

[4) Context Service](Mikroservisler%20ve%20Mod%C3%BCller/4)%20Context%20Service%202bf0faf2fb128000b3d9f7e678189edb.md)

[5) Embedding Service](Mikroservisler%20ve%20Mod%C3%BCller/5)%20Embedding%20Service%202bf0faf2fb128054a991f4c71fb1d381.md)

[6) nlp2sql Service](Mikroservisler%20ve%20Mod%C3%BCller/6)%20nlp2sql%20Service%202bf0faf2fb1280419089eece603a982a.md)

[7) SQL Validation Service](Mikroservisler%20ve%20Mod%C3%BCller/7)%20SQL%20Validation%20Service%202bf0faf2fb12808e9ef7e3d7072aff6c.md)

[8)DB Execution Service](Mikroservisler%20ve%20Mod%C3%BCller/8)DB%20Execution%20Service%202bf0faf2fb128010884dcd10f89ff0c0.md)

[9) Chart Service](Mikroservisler%20ve%20Mod%C3%BCller/9)%20Chart%20Service%202bf0faf2fb12805ab4a9ce276ccef6b4.md)

[10) Schedular Service](Mikroservisler%20ve%20Mod%C3%BCller/10)%20Schedular%20Service%202bf0faf2fb1280328c51d116156b01d8.md)

[11) Notification Service](Mikroservisler%20ve%20Mod%C3%BCller/11)%20Notification%20Service%202bf0faf2fb1280828441ced13220c77d.md)

[12) Audit Service](Mikroservisler%20ve%20Mod%C3%BCller/12)%20Audit%20Service%202bf0faf2fb12802aa6eff0b8b9782f82.md)

[13) Schema Cache Worker](Mikroservisler%20ve%20Mod%C3%BCller/13)%20Schema%20Cache%20Worker%202bf0faf2fb128082acb2d4dcebb1edd1.md)