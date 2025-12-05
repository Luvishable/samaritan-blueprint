# Token Saklama

**Access Token:**

- kısa sürelidir(10 dk)
- API çağrılarında kullanılır
- JWT içinde sub(kullanıcı id), email, tenant_id, role, exp vs. var.
- JS tarafında **bellekte** tutulur (değişkende, state’te vs.).
- LocalStorage’ta tutmamaya çalışırız; çünkü XSS’de çalınması daha kolay olur.
- Zaten kısa ömürlü olduğundan dolayı ele geçirilse bile çok uzun yaşayamaz.

**Refresh token**:

- Daha uzun süreli (ör: 30 gün),
- Access token süresi dolunca yenilemek için kullanılacak.
- JS tarafının görmemesi için, backend üzerinden **HttpOnly + Secure cookie** olarak ayarlanır.
- Frontend, token’ları aldıktan sonra backend’e “set_refresh_token” gibi bir endpoint’e gönderir.
- Backend bu refresh token’ı alır ve response’ta aşağıdaki gibi cookie olarak set eder.

```jsx
Set-Cookie: samaritan_rt=REFRESH_TOKEN_HERE; HttpOnly; Secure; SameSite=Lax; Path=/;
```

- Bu sayede tarayıcı bu cookie’yi diske yazar ve JS kodu document.cookie ile erişemez(HttpOnly)
- Sadece backend’e istek gittiğinde HTTP katmanında otomatik gönderilir.

**ID Token:**

- Kullanıcı kimliği bilgisi(OIDC kısmı)
- Genelde frontend için kullanılır

## Access token süresi bitince ne oluyor? (Refresh akışı)

Access token kısa ömürlü. Diyelim 10 dakika sonra süresi doldu.

İki şekilde durum fark edilir:

- Ya backend “401 Unauthorized, token expired” döner,
- Ya frontend token’ın `exp` alanına bakıp süre dolmak üzereyken yenileme yapar.

Frontent’in yapacağı şey:

1. Arka planda (kullanıcı fark etmeden) bir “refresh token” isteği atar.
2. Bu istek backend’e gider:
    
    ```jsx
    POST https://api.samaritan.com/auth/refresh
    Cookie: samaritan_rt=REFRESH_TOKEN_BURADA
    ```
    
- `POST https://api.samaritan.com/auth/refresh
Cookie: samaritan_rt=REFRESH_TOKEN_BURADA`
    - Refresh token cookie’de otomatik geldi (`HttpOnly` olduğu için JS bu cookie’yi elle koymadı, tarayıcı koydu).
- Backend bu endpoint’te:
    - Cookie’den refresh token’ı alır,
    - Keycloak’ın token endpoint’ine bir istek atar:
        
        ```
        POST https://auth.samaritan.com/realms/samaritan/protocol/openid-connect/token
        grant_type=refresh_token
        refresh_token=REFRESH_TOKEN_BURADA
        client_id=samaritan-frontend
        ```
        
1. 
    - `POST https://auth.samaritan.com/realms/samaritan/protocol/openid-connect/token
    grant_type=refresh_token
    refresh_token=REFRESH_TOKEN_BURADA
    client_id=samaritan-frontend`
2. Keycloak refresh token’ı kontrol eder:
    - Geçerli mi?
    - Süresi dolmuş mu?
    - Revoked mu?
    - Token rotation vs.
3. Her şey yolundaysa yeni bir `access_token` + gerekirse yeni bir `refresh_token` döner.
4. Backend:
    - Yeni access token’ı JSON olarak frontend’e döner,
    - Yeni refresh token geldiyse tekrar `Set-Cookie` ile cookie’yi günceller.

Bu akış sayesinde:

- Kullanıcı tekrar şifre/MFA girmeden uzun süre çalışabilir,
- Ama XSS ile yalnızca access token çalınırsa:
    - Refresh token JS’den görünmez,
    - Çalınan access token birkaç dakika içinde ölü olur.

**NOT:** Cookie, tarayıcının (Chrome, Firefox, Edge vs.) **profil klasöründe tuttuğu küçük bir key–value deposu**.

- Her cookie bir domain’e bağlıdır:
    - Örneğin `Domain=.samaritan.com`