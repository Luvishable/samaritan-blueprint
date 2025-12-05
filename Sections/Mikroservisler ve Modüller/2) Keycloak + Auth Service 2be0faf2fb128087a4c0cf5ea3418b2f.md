# 2) Keycloak + Auth Service

---

- Kullanıcı, rol, client, session ve token yönetimi bu serviste yapılır.
- Roller: ***system_admin, tenant_admin, tenant_user***
- Metadata DB’den tamamen farklı bir Postgres DB’si olacaktır.
- JWT doğrulama / JWKS cache görevleri bu servise aittir.
- Token’den tenant_id, user_id gibi claim’leri parse eder.
- Bazı helper ednpoint’lere sahip olacaktır: /me → current user bilgisi
- auth - service’in kendisine ait database’i yoktur: keycloak ile haberleşir.
- Bu serviste auth olaylarının loglanması için Kafka aracılığıyla:
    - *auth.login.succeeded*
    - *auth.login.failed*
    
    gibi event’ler üretilip audit-service’e gönderilebilir. 
    

[KEYCLOAK](2)%20Keycloak%20+%20Auth%20Service/KEYCLOAK%202be0faf2fb12801e9b22d5bfeae1d757.md)

[RATE-LIMIT](2)%20Keycloak%20+%20Auth%20Service/RATE-LIMIT%202be0faf2fb128042b3a3c68fc7c26a38.md)

---

## Keycloak ve Auth Service İşbirliği Akışı

### 1) Kim ne yapıyor?

- Keycloak: Kimlik sağlayıcı(IdP)
    - Kullanıcıların tamamı keycloak’un kendisine ait db’sinde durur.
    - e-posta, parola, MFA, account lock, brute-force koruması…
    - Roller: system_admin, tenant_admin, tenant_user
    - Token üretimi
        - Access Token (API için JWT)
        - ID Token (UI için profil bilgisi)
- Auth Service: Samaritan tarafındaki ince bağlayıcı katman. Görevleri
    - Davet süreçleri:
        - system_admin → tenant_admin
        - tenant_admin → tenant_user
    - Keycloak admin API’si ile konuşup:
        - Yeni kullanıcı oluşturma,
        - Kullanıcıya doğru role ve tenant bilgisini gömme
        - Davet linki üretme
        
        görevlerinde Keycloak ile işbirliği yapar.
        
    - Diğer microservice’ler için “authorization helper” rolünü oynar:
        - Çünkü token parsing işlemi auth-service tarafından yapılır
        - Bu token gerçekten şu tenant için geçerli mi sorusunu diğer microservice’ler için cevaplar
        - Bu kullanıcı bu tenant’ta admin mi gibi kontroller için tüm microservice’ler için ortak bir mantık sunar.

**NOT:** Kullanıcı profili sadece Keycloak’ta tutulacak, Metadata DB’de ayriyeten bir users tablosu olmayacak. 

### 2) Keycloak Yapısı: Realm, Client, Role, Tenant

- Realm : samaritan
- 3 tane client:
    - samaritan-ui (public client):
        - [app.samaritan.com](http://app.samaritan.com) üzerinden çalışan frontend
        - PKCE ile authentication code üretiminin akışı:
            - UI → Keycloak login → token → UI → API
    - samaritan-api (confidential / bearer only client)
        - [api.samaritan.com](http://api.samaritan.com) arakasındaki tüm backend servisler için
        - API Gateway gelen JWT’nin aud/azp claim’lerinin bu client’a uygun olduğunu doğrular.
- Realm seviyesinde roller:
    - system admin:
        - Tüm sistem üzerinde tam yetki
        - Yeni tenant onboarding’ini başlatabilir
    - tenant_admin:
        - Sadece kendi tenant’ı üzerinde:
            - onboarding wizard
            - datasource ekleme
            - business rule, golden example (doğru soru-sql çifti), tenant_user daveti gibi işleri yapmaya yetkilidir.
    - tenant_user:
        - Sadece soru sorma, sonuç görme, feedback verme, favorileme, alarm kurma işlemlerini yapar.
        - Bağlı olduğu tenant_admin yetki verirse business rule ve golden examples ekleyebilir.
    

### 3) auth-service’in Sorumlulukları

---

> auth-service’i Keycloak’u doğrudan UI’ya açmak istemediğimiz yerlerde kullandığımız bağlayıcı/yapıştırıcı katman gibi düşünebiliriz.
> 

### Yapacağı İşler

### 1) Davet akışlarını yönetir:

- `system_admin` → `tenant_admin` daveti
- `tenant_admin` → `tenant_user` daveti

Bu akışlarda:

- Davet isteği API’dan gelir (UI üzerinden).
- API Gateway → auth-service’e route eder.
- auth-service:
    - Metadata DB’de bir **davet kaydı** açar:
        - `invitation_id`, `email`, `invited_by`, `type`, `status`, `created_at`, `expires_at` vb.
    - Keycloak Admin API ile:
        - İlgili realm’de user oluşturur (veya varsa bulur),
        - Kullanıcıya uygun rolü (`tenant_admin` / `tenant_user`) atar,
        - Eğer tenant_admin davetiyse `tenant_id` henüz yoktur; bu kullanıcı, ilk login sonrası onboarding wizard’ını açacak şekilde işaretlenir (ör. user attribute: `onboarding_pending = true`).
        - Tenant_user için`tenant_id` attribute’u doğrudan set edilir.
    - Davet linkini üretir:
        - Bu link ya:
            - Keycloak’ın action link’i (set-password / verify-email),
            - ya da UI tarafında onboarding’i başlatan signed token olabilir.
    - Davet linkini:
        - Notification-service’e ileterek mail attırabilir,
        - veya `201 Created` response body’de UI’a geri verip, system_admin’e copy-paste yaptırabilirsin (senin kararın).

### 2) Keycloak admin-api ile entegrasyon:

- Yeni user oluşturma (`POST /admin/realms/.../users`)
- Role assignment:
    - `realm-management/` altında veya realm role’leri ile.
- User attribute set/get:
    - `tenant_id`, `onboarding_pending` gibi.

auth-service bu Admin API’yi kullanmak için Keycloak’ta ayrı bir “service account” client kullanır (ör. `samaritan-admin`).

### 3) Diğer mikroservislere “authorization helper” olmak:

Bazı servislerin işine yarayabilir:

- “Bu user bu tenant’ta admin mi?” sorusunu tek yerde çözmek.
- “Bu user’ın hangi tenant/role kombinasyonları var?” gibi advanced sorular.

Bu durumda auth-service, token’daki claim’lere ve gerekirse Keycloak Admin API’ye bakarak:

- `GET /auth/tenant-permissions?tenant_id=...` gibi endpoint’lerle net cevap dönebilir.

### 4) system_admin → tenant_admin Davet Akışı

---

- A**dım 1: system_admin UI’dan davet başlatır**
    1. system_admin, [app.samaritan.com](http://app.samaritan.com) üzerinde “Yeni tenant_admin davet et” formunu açar.
    2. e-posta adresini girer ve gönder buttonuna basar.
    3. UI → `POST https://api.samaritan.com/auth/invite-tenant-admin` çağrısını yapar.
    4. API Gateway:
        1. JWT’yi doğrular,
        2. system_admin rolünü kontrol eder,
        3. Request body + user bilgileriyle auth-service’e yönlendirilir.
- **Adım 2: auth-service davet kaydı ve Keycloak işlemleri**
    1. Metadata DB’de yeni kayıt açılır:
        - `id`
        - `email`
        - `invited_by_user_id` (Keycloak `sub`)
        - `type = tenant_admin`
        - `status = pending`
        - `created_at`, `expires_at`
        - tenant bilgisi henüz yoktur çünkü tenant bilgilerini tenant_admin girecek.
    2. Keycloak Admin API ile:
        1. Bu email ile kayıtlı bir user olup olmadığını kontrol eder. Yoksa enabled=False ile bir user oluşturur, varsa mevcut user’ı kullanır.
        2. Kullanıcıya realm role olarak tenant_admin atanır.
        3. user_attribute `onboarding_pending = true` set edilir.
    3. built-in required actions eklenebillir: VERIFY_EMAIL, UPDATE_PASSWORD, CONFIGURE_TOTP
    
- **Adım 3: action link**
    - Keycloak’tan action link (password set / verify email linki gibi) alınır.
        - Davet sırasında auth-service o User’ın required_actions listesini şu şekilde set eder:
            - VERIFY_EMAIL
            - UPDATE_PASSWORD
            - CONFIGURE_TOTP
        
        Kullanıcı davet linkine tıklayıp login olmaya çalıştığında:
        
        - Sırasıyla:
            - Mail doğrulama (zaten linkle halledilmiş olabilir),
            - Parola güncelleme,
            - TOTP setup ekranı
                
                zorunlu olarak gösterilir.
                
        - Bu akışla birlikte sadece davet edilen kullanıcılar için TOTP zorunlu olue; arka plandaki service_account user’lar için gerek kalmaz.
- **Adım 4: davet linki hazırlandıktan sonra**
    - Linki api alır ve uygulamanın mail’i aracılığıyla yollar.
    - Keycloak’un maili yollaması yerine ona **direct access token** ürettiririz.
    - Sonra bu token’i bir URL’ye ekler ve mailin içine gömeriz.
        - [`https://auth.samaritan.com/realms/.../login-actions/action-token?key=](https://auth.samaritan.com/realms/.../login-actions/action-token?key=)...`
- **Adım 5: tenant_admin davet linkine tıklar**
    1. link auth.samaritan.com’a gider
    2. Kullanıcı parolasını set eder, TOTP kurar ve hesabını aktif eder
    3. login sonrası `redirect_uri=https://app.samaritan.com/onboarding` ile UI’a döner.
    4. UI, token içindeki `onboarding_pending=true` attribute’unu görür, onboarding wizard’a yönlendirir.
    5. Onboarding wizard:
        - Şirket adı, ülke, sektör…
        - Bu bilgiler `catalog-service`’e gider → `catalog.tenants` kaydı oluşturulur.
        - `onboarding_pending=false`’a dönülür.
    

### 5) tenant_admin → tenant_user davet akışı

- system_admin’in tenant_admin’i davet etme akışı ile aynıdır.

## Keycloak’ta Admin API kullanımı

- samaritan-ui ve samaritan-api client’ları kullanıcıların tokenlerini temsil ediyor. Yani bir user login olduğunda o user adına bir token alınıyor ve bu tokenle API çağrıları, UI işleri vb. yapılabiliyor.
- Ama Admin API ile yapılacaklar daha başka olduğu için Keycloak tarafında özel bir service account lazım. Sonuçta herkesin user sil, ekle, rol ata gibi işlemleri yapamaması lazım.
- Bu sebeplerden ötürü Keycloak tarafında ayrı bir client oluşturulur***(samaritan-admin)***
- Bu client’ta `service_account=enabled` yapılır.
- Bu client’ın service account user’ına realm-management rolü verilir(manage-users, view-users vb.)
- auth service client credentials flow ile:
    - `client_id=samaritan-admin`, `client_secret=..` ile accessd token alır kendi adına ve bu tokenle admin api'yi çağırır.