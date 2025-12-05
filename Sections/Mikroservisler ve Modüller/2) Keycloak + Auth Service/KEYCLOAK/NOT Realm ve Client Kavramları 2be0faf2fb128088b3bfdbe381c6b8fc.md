# NOT: Realm ve Client Kavramları

---

> Realm, keycloak içindeki izole kimlik evreni diğer bir deyişle güvenlik bölgesidir.
Bu evrenin içinde kullanıcılar, roller, client’lar, gruplar, login ayarları vb. vardır ve hepsi diğer realm’lerden tamamen bağımsızdır.
> 

- **Realm Neyi Yönetir?:**
    
    Bir realm’in içinde şunlar var:
    
    - **Users (kullanıcılar)**
    - **Credentials** (parola, OTP, WebAuthn, vb.)
    - **Roles & Groups** (roller, gruplar)
    - **Clients** (uygulamalar / API’ler)
    - **Identity Providers** (Google, LDAP, AD vb.)
    - **Authentication/Authorization flow’lar** (login akışları, MFA, zorunlu action’lar)
    
    Bir kullanıcı bir realm’e aittir ve login olurken o realm’e login olur. 
    

- **Tek Keycloak; Birden Fazla Realm:**
    1. Environment ayırmak için:
        1. samaritan-dev
        2. samaritan-staging
        3. samaritan-prod
        
        Hepsinin ayrı kullanıcıları, client’ları ve Redirect URL’leri olabilir. 
        

- **Client Nedir?:**
    - Keycloak’ta client bir uygulama kimliğidir.
        - `samaritan-ui` → browser’da çalışan SPA (frontend)
        - `samaritan-api` → backend API’lerin
        - `samaritan-admin` → sadece auth-service’in admin işleri için kullanacağı teknik hesap
    - Keycloak “bu isteği yapan uygulama kim” sorusunu client üzerinden cevaplıyor.
    - **Public Client:**
        - Sır saklayamayan uygulama
        - Kullanıcının elinde öalışan uygulamalar (frontend, mobil, desktop app vb)
        - Bu uygulamalarda ortak özellik kullanıcının kodu görebilir, dosyaları kopyalayabilir ve process’i debug edebilir. Yani gizli kalması gereken bir client_secret hemen çalınır.
        
        Bu yüzden public client’larda:
        
        - **Client secret YOKTUR** (veya varmış gibi gözükse bile güvenilemez).
        - Keycloak, bu client’tan gelen istekleri şu mantıkla değerlendirir:
            - “Bu uygulama kendini **ispat edemez**, sadece user’ı ispat edebilir.”
        - Kullanılan flow genelde:
            - **Authorization Code + PKCE** (SPA’ler için önerilen yöntem):
                - Kullanıcı login olur,
                - Keycloak code üretir,
                - UI bu code + PKCE ile access token alır.
        - Ama client asla
            
            `client_id + client_secret`
            
            ile “ben gerçekten X uygulamasıyım” diye ispatlayamaz.
            
    - **Confidential Client:**
        - secret’ın güvenle tutulabileceği, sunucu tarafında çalışan uygulama: API’ler, mikroservisler, background worker’lar
        - Keycloak’a istek atarken:
            - `Authorization: Basic base64(client_id:client_secret)`
            - veya *form_body*’de `client_secret=…` göndererek “ Ben X client’ıyım, işte id ve  secret_key’im diyip Keycloak tarafından onaylanabiliyor.
        - Yani public client’tan farklı olarak herhangi bir user olmadan da bu uygulamalar token alabiliyor.

- **Realm vs Client:**
    - Realm güvenlik domain’idir, yönetim yapılır.
    - Client ise bu realm tarafından korunacak olan tekil uygulama veya API
    - Örn: Realm → samaritan-prod, Client → samaritan-ui
    - Aynı kullanıcı aynı login ile samaritan api’sine de erişebilir tabi rolünün izni olursa.

- **Master Realm ve Diğer Realm’ler:**
    - Keycloak kurulduğunda bir tane özel realm vardır: ***master realm***
    - Bu realm yönetim içindir yani keycloak admin gibi düşünülebilir.
    - Bu realm aracılığıyla yeni realm oluşturma, admin kullanıcıları ve global ayarlar yapılabilir.