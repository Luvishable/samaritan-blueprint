# Redis

---

- Redis projede iki ana amaçla kullanılacak:
    1. Rate limiting / sayaç 
        1. Login denemeleri (auth rate limit)
        2. API çağrıları (domain/endpoint rate limit)
    2. Metadata Caching
        1. Tenant’ın exposed table/view + column bilgisi
        2. Bunların prompt oluşturma sırasında hızla çekilmesi
    
    **NOT**: *Golden example*’lar ve *business rule’*lar Redis’te tutulmayacak. Redis projede performans artırıcı ve rate limit için yardımcı olarak kullanılacaktır.
    

---

## 1. Rate limit tarafında Redis kullanımı

---

### Auth rate limit:

- Hedef: `auth.samaritan.com` üstündeki login uçlarını kaba brute-force’tan korumak.
- Ana anahtarlar:
    - `ratelimit:auth:ip:{ip_address}`
        
        → Bu IP’den 1 dakikada kaç login denemesi geldi?
        
- Akış:
    - NGINX / API Gateway:
        - İstek geldiğinde `INCR` yapar,
        - İlk kez görüyorsa `EXPIRE` 60 saniye,
        - Sayaç belli bir eşiği aşarsa 429 döndürür.

Bu sayede Keycloak’a ulaşmadan önce kaba saldırılar boğuluyor.

### Domain / endpoint rate limit:

- Hedef: `api.samaritan.com` tarafındaki **pahalı endpoint’leri** sınırlamak:
    - `/nlp2sql/execute`
    - `/reports/export`
    - Belki `/chart/plan` vs.
- Ana key pattern’ler:
    - User bazlı:
        - `ratelimit:user:{user_id}:nlp2sql_per_minute`
    - Tenant bazlı:
        - `ratelimit:tenant:{tenant_id}:nlp2sql_per_minute`
    - Endpoint bazlı:
        - `ratelimit:user:{user_id}:export_per_hour`
- API Gateway:
    - Her istek için uygun key’leri `INCR` + `EXPIRE` ediyor.
    - Limit aşılırsa 429 ile geri dönüyor.

**Burada Redis** sadece çok hızlı sayaç / TTL deposu.

## 2. Schema cache tarafında Redis kullanımı

---

- `catalog.exposed_objects` + `catalog.exposed_columns` birleşiminden oluşan schema parçalarını saklayaacğız.
- Golden example’lar ve business rule’lar Metadata DB’de duracak ve embedding service ile semantic benzerlik ölçüsüne göre çekilecek.
- Redis’in buradaki görevi, bilinen bir exposed table/view + column metadatasını Postgres’e inmeden hızlıca çekebilmek.
    
    ### Key yapısı:
    
    - tenant ve datasource bazında isimlendirme yapacağız.
    - schema:tenant:{tenant_id}:ds:{datasource_id}:objects:
        
        ```json
        [
          {
            "exposed_object_id": "uuid-1",
            "db_schema_name": "REPORTING",
            "object_name": "EXPORTS_MONTHLY",
            "object_type": "view",
            "display_name": "Aylık İhracat Görünümü"
          },
          {
            "exposed_object_id": "uuid-2",
            "db_schema_name": "PUBLIC",
            "object_name": "CUSTOMERS",
            "object_type": "table",
            "display_name": "Müşteriler"
          }
        ]
        ```
        
    - Tek bir object için full detay:
        - schema:tenant:{tenant_id}:ds:{datasource_id}:object:{exposed_object_id}
        
        ```json
        {
          "exposed_object_id": "uuid-1",
          "db_schema_name": "REPORTING",
          "object_name": "EXPORTS_MONTHLY",
          "object_type": "view",
          "display_name": "Aylık İhracat Görünümü",
          "description": "Her satır ülkeler bazında aylık toplam ihracatı temsil eder.",
          "columns": [
            {
              "exposed_column_id": "col-1",
              "column_name": "MONTH",
              "data_type": "DATE",
              "logical_type": "date",
              "is_exposed": true,
              "is_pii": false,
              "description": "Ayın ilk günü",
              "ordinal_position": 1
            },
            {
              "exposed_column_id": "col-2",
              "column_name": "COUNTRY_CODE",
              "data_type": "VARCHAR2(2)",
              "logical_type": "country_code",
              "is_exposed": true,
              "is_pii": false,
              "description": "ISO ülke kodu (TR, DE, US...)",
              "ordinal_position": 2
            },
            {
              "exposed_column_id": "col-3",
              "column_name": "TOTAL_AMOUNT",
              "data_type": "NUMBER(18,2)",
              "logical_type": "amount",
              "is_exposed": true,
              "is_pii": false,
              "description": "İhracat tutarı (USD)",
              "ordinal_position": 3
            }
          ]
        }
        ```
        
    
    ### Cache’leme ne zaman yapılacak? :
    
    - Kullanıcı soru sorduğunda redis açısından gerçekleşen akış:
        1. Kullanıcı soru sorar ve nlp2sql service devreye girer
        2. nlp2sql embedding service’ten “bu soruyla en alakalı schema parçaları hangileri?” diye sorar.
        3. Vektör DB’den ilgili schema parçalarının ID’leri alınır: `[uuid-1, uuid-2]`
        4. context service:
            1. Her object için Redis’e bakar:
                1. `GET schema:tenant:T:datasource:D:object:uuid1`
                2. `GET schema:tenant:T:datasource:D:object:uuid2`
            2. Eğer hit olursa yani Redis’te schema parçaları mevcutsa SJON direkt kullanılır
            3. Eğer miss olursa yani Redis’te schema parçaları yoksa gereken schema parçaları `catalog.exposed_objects` + `catalog.exposed_columns`’tan çekilir ve prompt’a verilir.
    
    ### Kim Redis’e yazacak? :
    
    - schema cache worker `schema.exposed-object.changed`, `schema.exposed-column.changed` event’lerini dinler ve ilgili object için gereken veriyi Postgres’ten alıp Redis’e yazar.
    
    ### Redis key’lerinin yaşam süresi:
    
    - Schema key’leri onboarding sonrası çok sık değişmeyeceği için agresif bir TTL’e gerek yok
    - Schema değişince *Kafka event’leri* ile **var olan schema parçaları invalid edilir. *(explicit invalidation)*
        - Rate limit key’leri içinse kısa TTL’ler tercih edilecektir.
            - IP bazlı login → 60 saniye
            - User bazlı query → 60 saniye / 1 saat
            - Bu anahtarlar sık sık oluşturulup silinecekler.