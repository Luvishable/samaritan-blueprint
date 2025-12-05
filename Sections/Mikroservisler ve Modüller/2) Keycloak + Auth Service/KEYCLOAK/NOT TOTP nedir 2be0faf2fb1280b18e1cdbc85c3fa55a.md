# NOT: TOTP nedir?

---

> TOTP: **Time-based One-Time Password,** zaman bazlı, tek kullanımlık, genelde 6 haneli kod üreten algoritma. 
Her 30 saniyede bir yeni kod oluşur.
> 
- Kullanıcı önce şifre girer, sonra telefonundaki uygulamadan (Örn: Google Authenticator) 6 haneli kodu girer.

**Nasıl Çalışır?**

---

- Sunucu ile kullanıcının telefonu arasında ortak bir gizli anahtar vardır (secret key).
    - Bu secret rastgele üretilir ve Base32 formatındadır.
- Hem sunucu hem telefon bu secret’ı bilir.
- İkisi de kendi taraflarında şu anki zamanı kullanarak aşağıdaki code’u üretir:
    - code = HMAC(secret, current_time_slot) → 6 haneli sayı
    - current_time_slot = unix_time / 30 sn yani zaman 30 saniyelik parçalara bölünür.
- Oluşturulan bu kod sunucudan telefona gönderilmez, sadece her ikisi de aynı zaman slotuna baktığı için aynı kodu üretir.

**Google Authenticator Neden Kullanılıyor?:**

---

- Google Authenticator aslında sadece bir TOTP client.
- Telefonda offline şekilde çalışabilir ve kod üretir.
- Authy gibi alternatifler de vardır fakat Google Authenticator ücretsizdir.
- Google Authenticator kurulumu sırasında; sunucu bir QR kod ile telefonundaki Google Authenticator app’ine secret key’i gösterir ve app bunu telefona kaydeder.