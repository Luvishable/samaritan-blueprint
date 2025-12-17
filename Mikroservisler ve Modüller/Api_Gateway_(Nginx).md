# 1) Api - Gateway (Nginx)

---

- Tüm HTTP trafiği için tek giriş noktası olacaktır, **reverse-proxy** görevi görecektir.
- Hangi hostname’i hangi uygulamaya yönlendirmesi gerektiğini bilir:
    - [app.samaritan.com](http://app.samaritan.com) → UI ( SPA - Single Page Application)
    - [api.samaritan.com](http://api.samaritan.com) → backend API Gateway
    - [auth.samaritan.com](http://auth.samaritan.com) → Keycloak
- Bunların hepsi aynı IP’ye çözülüyor (edge NGINX)
- samaritan-ui sadece api-gateway üzerinden backend’e ulaşabilir.
- TLS terminasyonu (HTTPS) bu katmanda yapılacaktır.
- IP bazlı rate-limit nginx, domain bazlı rate-limit ise API Gateway tarafında uygulanacaktır. (rate-limit konusu ayrıntılı olarak  ele alınıyor)
- Bu servise ait bir database yoktur, nginx’in kendisine ait nginx.conf dosyasının konfigüre edilmesi ile bu servis çalışır ve güvenlli hale getirilir.
- API Gateway’e bir request geldiğinde:
    1. Request API Gateway Nginx tarafından proxy’lenir.
    2. API Gateway Authorization: Bearer <token> header’ını kontrol eder. 
        1. Token yoksa ve eğer endpoint public değilse 401 döner.
        2. Token varsa:
            1. Keycloak JWKS endpoint’inden aldığı public key ile imzayı doğrular
            2. exp, iss, aud claim’lerini kontrol eder
            3. sub (user_id), tenant_id, roles (system_admin, tenant_admin, tenant_user) gibi claim’leri de gerekirse extract edebilir. 
    3. Doğrulama başarılıysa ilgili route’un permission kurallarına bakar. Örn:
        1.  /auth/invite-tenant-admin → sadece system admin’e açık
        2. /catalog/* → tenant-admin
        3. /query/execute → tenant_user + tenant_admin
        
        Uygunsa isteği gereken mikroservise yönlendirir.
        
        ***NOT***: Bu sayede içteki mikroservisler token parsing ile uğraşmak zorunda kalmaz ve API Gateway bu servislere:
        
        X - User - Id
        
        X - Tenant - Id
        
        X - User - Roles
        
        X - Request - Id
        
        gibi header’lar aracılığıyla kimlik bilgilerini normalize edilmiş halde iletir.