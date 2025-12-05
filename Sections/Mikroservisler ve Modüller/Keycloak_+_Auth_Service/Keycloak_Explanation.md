# KEYCLOAK

<aside>
💡

Keycloak, modern web uygulamaları ve microservice mimarileri içim kimlik ve erişim yönetimi sağlayan, açık kaynaklı bir Single Sign-On sunucusudur.

Yani kullanıcılar, roller, token login/logout, Multi-Factor Authentication gibi tüm fonksiyonlar tek bir merkezde toplanır.

</aside>

### 1) Authentication (Kimlik Doğrulama)

---

- Kullanıcının kim olduğunu doğrular.
    - Kullanıcı adı / parola kombinasyonu,
    - Google, Github,Facebook Microsoft gibi social login’ler sunar.
    - OTP, SMS, e-posta gibi 2FA ve MFA seçenekleri sunar.
- Login formunu uygulamaların içinden çıkarıp Keycloak’un kendi ekranına taşır (*redirect* ile*)*

### 2) Single Sign-On (SSO)

---

- Kullanıcı bir kere Keycloak’a login olur ve aynı realm içindeki diğer uygulamalara tekrar parola girmeden girebilir.
- Logout olduğunda da hepsinden çıkabilir.
- Google’a giriş yapıldığında otomatikmen diğer Google ürünlerine de giriş yapılabilmesi gibi..

### 3) Authorization (Yetkilendirme)

---

- Kullanıcılara role, group, ve hatta daha ileri seviye izinler iş kurallarımıza göre tanımlanabilir.
- Keycloak’un fine-graiend authorization özelliğiyle “şu kullanıcının şu resource’a şu saatte şu aksiyonu yapabilmesi” gibi çok detaylı kurallar tanımlanabilir.
- 

### 4) Token Servisi

---

- ***OpenID Connect (OIDC***) ve ***OAuth2*** protokollerine uygun biçimde:
    - ***access token*** (genelde JWT)
    - refresh token
    - id_token
    
    üretir.
    
- Yani mikroservis mimarisinde API çağrılarında taşınılan token’in kaynağı Keycloak’tur.

### 5) Kullanıcı Yönetimi ( User Management)

---

- Admin panelinden user oluşturma, silme, disable etme işlemleri yapılabilir.
- Parola reset, e-posta doğrulama, profil alanları, zorunlu action’lar (ilk girişte parola değiştir, TOTP ayarla vb.) Keycloak tarafından yönetilebilir.
- Keycloak kullanıcıları kendi database’inde tutar.

### 6) Kavramlar ve Açıklamaları

[**NOT: TOTP nedir?**](KEYCLOAK/NOT_TOTP_nedir.md)

[**NOT: Realm ve Client Kavramları**](KEYCLOAK/NOT_Realm_ve_Client_Kavramları.md)

[Token Saklama](KEYCLOAK/Token_Saklama.md)