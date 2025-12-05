# Rate Limit Uygulanması

> Uygulamamızın bot ve DDOS saldırılarına karşı korunması, kötü niyetli kullanıcıların art arda istekler atarak server’a fazla yüklenmelerini engellemek adına iki katmanda rate limit uygulanacaktır: 
**Auth rate limit** → login/token endpointleri
**Domain rate limit** → samaritan business logic endpointleri; query, report vb.
> 

## 1) Auth Rate Limit

---

- Login ve token endpointlerine gelen istekleri sınırlamaya yarar.
- Keycloak’ı ve database’i şifre deneyen botlardan korur.
- Çok fazla yanlış şifre girilirse o hesabı kilitler.
- Bu rate limitin uygulanması iki aşamada gerçekleşecektir:
    - NGINX / API Gateway katmanı → IP bazlı rate limit
    - Keycloak → hesap bazlı brute-force detection

### a) Adım 1: NGINX’te IP bazlı limit (auth.samaritan.com)

> Hedef: Keycloak’a giden login/token istek sayısını IP’ye göre sınırlamak.
> 
- Uygulanacak fikir:
    - `client_ip` adresini al
    - Redis’te bir sayaç tut: `login:ip:{client_ip}`
    - Belirlediğimiz pencere içinde (ör: 60 sn) çok fazla istek varsa, isteği keycloak’a iletmeden 429 döndür.
- Nerede Uygulanacak?
    - Host: `auth.samaritan.com`
    - Path’ler:
        - `/realms/.../protocol/openid-connect/auth`
        - `/realms/.../protocol/openid-connect/token`
        - bunun yanında forgot-password gibi endpoint’lerde de uygulanır.
- Teknik implementasyon yöntemleri:
    - NGINX’in `limit_req` direktifleriyle
    - Daha gelişmiş yapı: NGINX + Lua + Redis
    - Veya API Gateway gömülü bir rate-limit özelliğiyle

### b) Adım 2: Keycloak içi brute-force detection (hesap bazlı)

- NGINX IP bazlı saldırıları kesti ama bazı saldırılar farklı IP’lerden gelebilir. Burada da devreye Keycloak’un kendi mekanizması devreye giriyor.
- Keycloak’un mantığı:
    - Belli bir sayıda yanlış login attempt’inden sonra kullanıcı hesabını kilitle ve belirli süre login olamasın
    - Admin panelinde event olarak göster
    - İleride CAPTCHA vb. gibi önlemler alınabilir.

**Özetle auth rate limit:**

- **NGINX / API Gateway** → “Bu IP çok saldırgan, içeri bile almıyorum.”
- **Keycloak** → “Bu kullanıcı hesabı çok zorlanıyor, hesabı kilitliyorum / geciktiriyorum.”

İkisini birlikte kullanınca:

> Hem IP bazlı volumetrik,
> 
> 
> hem **hesap bazlı hedefli** saldırıları yakalamış oluruz.
> 

## 2) Domain Rate Limit

---

> Login olmuş kullanıcının:
> 
> - `/query/execute`,
> - `/reports/run`,
> - `/reports/export`
> 
> gibi işlevsel endpointleri ne kadar kullandığını sınırlamak.
> 
- Amaç:
    - LLM maliyetini ve DB’ye giden yükü azaltıp, kaynakların kullanımını optimize etmek
    - İleride uygulamada ücretli paketlere geçildiğinde (free/plus/pro) kota uygulayabilmek
- Nerede uygulanır?
    - [`api.samaritan.com`](http://api.samaritan.com) üzerinde API Gateway + Redis ile
    - Kullanıcı veya tenant’ların belli bir pakete üye olmaları durumunda ise JWT claim’lerinden `user_id` ve `tenant_id` ile plan bilgisi okunur.
- Login aşamasından sonra elimizde bir access token olacağı için:
    - sub (user_id)
    - tenant_id
    - roles
    - plan(paket tipi)
    
    gibi claim’lere erişebiliriz ve böylece bir user veya tenant’ın belirli bir endpoint’i ne sıklıkla kullanabileceğini belirleyebiliriz.
    
- API Gateway ne yapacak?:
    - JWT’yi Keycloak ile doğrulamak,
    - Claim’lerden yukarıda bahsedilen bilgileri çıkarmak
    - Redis’e gidip ilgili sayacı artırmak
    - Limit aşılmışsa isteği ilgili microservice’e göndermeden 429 dönmek.
    
    ### 2.1) User bazlı domain rate limit
    
    ---
    
    - Örneğin her user’ın:
        - dakikada en fazla 20
        - saniyede en fazla 2 sorgu yapabilmesi gibi
    - 
        - 
    
    **Redis key pattern:**
    
    - `ratelimit:user:{user_id}:per_minute`
    - `ratelimit:user:{user_id}:per_second`
    
    ### 2.2) Tenant bazlı domain rate limit
    
    - Bir tenant’ın toplam LLM sorgusu:
        - Dakikada 100,
        - Saniyede 10 olsun.
    
    **Redis key pattern:**
    
    - `ratelimit:tenant:{tenant_id}:per_minute`
    - `ratelimit:tenant:{tenant_id}:per_second`
    - Bir tenant içindeki tüm kullanıcılar beraber saldırsalar bile tenant’ın istek veya ileride bütçe anlamında bir sınırı olduğu için tek bir tenant saldırısından dolayı tüm cluster çökmez.