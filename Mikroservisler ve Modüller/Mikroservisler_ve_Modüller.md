# Mikroservisler ve Modüller

## Genel Bakış

---

<aside>
💡

Multi-tenant, mikroservis tabanlı, event-driven yapıya sahip olacak uygulamamızda, her servisin kendisine ait sorumluluğu olacak ve ***servisler arası async iletişim için Kafka*** kullanılacaktır. 

</aside>

- Servislerin hepsi *docker container* olarak çalışacak.
- Servislerdeki data Metadata DB’deki schema’larda tutulacak. Başlangıçta her bir servis için ayrı database olmayacak.

[1) Api - Gateway (Nginx)](Mikroservisler%20ve%20Mod%C3%BCller/Api_Gateway_(Nginx).md)

[2) Keycloak + Auth Service ](Mikroservisler%20ve%20Mod%C3%BCller/Keycloak_+_Auth_Service.md)

[3) Catalog Service](Mikroservisler%20ve%20Mod%C3%BCller/Catalog_Service.md)

[4) Context Service](Mikroservisler%20ve%20Mod%C3%BCller/Context_Service.md)

[5) Embedding Service](Mikroservisler%20ve%20Mod%C3%BCller/Embedding_Service.md)

[6) nlp2sql Service](Mikroservisler%20ve%20Mod%C3%BCller/nlp2sql_Service.md)

[7) SQL Validation Service](Mikroservisler%20ve%20Mod%C3%BCller/SQL_Validation_Service.md)

[8)DB Execution Service](Mikroservisler%20ve%20Mod%C3%BCller/DB_Execution_Service.md)

[9) Chart Service](Mikroservisler%20ve%20Mod%C3%BCller/Chart_Service.md)

[10) Schedular Service](Mikroservisler%20ve%20Mod%C3%BCller/Schedular_Service.md)

[11) Notification Service](Mikroservisler%20ve%20Mod%C3%BCller/Notification_Service.md)

[12) Audit Service](Mikroservisler%20ve%20Mod%C3%BCller/Audit_Service.md)

[13) Schema Cache Worker](Mikroservisler%20ve%20Mod%C3%BCller/Schema_Cache_Worker.md)