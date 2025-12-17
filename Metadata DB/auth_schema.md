# auth schema

---

<aside>
💡

system_admin ve tenant_admin tarafından oluşturulan davetlerin kaydını izlemek adına oluşturulacak olan schema.

**NOT:** User kayıtları Keycloak’a ait database’de persist edilecektir. Bizim bu schema’yı tutma amacımız davet akışını takip edebilmek.

</aside>

- **auth.invitations**
    - Her davet maili için bir row.
    - `id` (uuid, PK)
    - `email` (text)
        
        → Davet edilen kişinin mail adresi.
        
    - `type` (enum: `'tenant_admin' | 'tenant_user'`)
        
        → Kimi davet ediyoruz?
        
    - `tenant_id` (uuid, nullable)
        
        → `tenant_user` davetlerinde DOLU (davet eden tenant_admin’in bağlı olduğu tenant).
        
        → `tenant_admin` davetinde **başta boş**, onboarding sonrası set edilebilir.
        
    - `invited_by_user_id` (text)
        
        → Daveti kim yaptı? (Keycloak `sub` ile daveti yapanın ID’si çekilir)
        
    - `keycloak_user_id` (text, nullable)
        
        → Davet sırasında Keycloak’ta oluşturulan user’ın ID’si.
        
    - `status` (enum: `'pending' | 'accepted' | 'expired' | 'revoked'`)
    - `token_hash` (text, nullable)
        
        → Kendi ürettiğimiz davet token’ını saklamak istersek (güvenli olması için hash).
        
    - `created_at` (timestamptz)
    - `expires_at` (timestamptz, nullable)
        
        → Davet geçerlilik süresi.
        
    - `accepted_at` (timestamptz, nullable)
        
        → Kullanıcı daveti ne zaman kullandı?
        
    
    > Kullanıcı TOTP + şifre + onboarding’i tamamladığında bu kayıt accepted durumuna çekilebilir.
    > 
    > 
    > Böylece hangi davetler “ölü”, hangileri aktif, hepsini metadata DB’den izlenebilir.
    >