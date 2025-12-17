# 5) Embedding Service

---

- Kullanıcının doğal dilde sorduğu sorunun gerektirdiği business rule’larının, golden example’ların ve table/view/kolonların vektör database’den çekilebilmesi için söz konusu sorunun embedding tekniği ile vektörel bir sayısal ifadeye dönüştürülmesi gerekir.
- Çünkü vektör database’de bilgiler text olarak değil sayısal olarak tutulur. Bu yüzden similarity  search(benzerlik araması) yapılabilmesi için embedding şarttır.
- Embedding GPU veya güçlü CPU gerektiren yani hardware gücü isteyen bir işlem olduğundan bu noktada OpenAI, Deepseek gibi şirketlerin embedding API’lerini kullanıyoruz.
- Onboarding wizard sırasında girilen bilgiler (golden examples, table/view/kolon açıklamaları vb) embed edilerek vektör db’ye yazılır.
- Bu servisin Metadata DB’de kendisine ait bir schema’sı yoktur.