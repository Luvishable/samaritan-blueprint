# Agentic Workflow Tasarımı – LangGraph / LangChain Analizi

Bu doküman, Samaritan / nlp2sql projesinde kullanacağımız **agentic mimariyi**, LangGraph / LangChain açısından ayrıntılı olarak anlatır:

- Hangi **node**’lar (agent / tool node) var,
- Bu node’ların **görevleri**, input / output’ları,
- Aralarındaki **ilişkiler** ve **event** akışı,
- Hangi node’ların **LLM çağrısı** yaptıkları,
- Hangi node’ların **deterministik** olduğu,
- **Teknik retry** (DB hatası) ve **business retry** (kullanıcı feedback’i) akışları,
- Grafik çizdirme (ChartPlannerLLMAgent + ChartRuntimeTool + InsightRuntimeTool),
- 5 adet **edge case** interaction senaryosu.

> Not: Bu mimari, **query sonuç satırlarını kalıcı olarak saklamayan**, sadece SQL ve metadata’yı tutan (örn. `query_runs` tablosu) bir tasarıma göre düşünülmüştür. Excel/PDF export için sonuç satırları gerektiğinde SQL tekrar çalıştırılır.

---

## 1. Yüksek Seviye Resim

### 1.1. Genel akış

Her yeni kullanıcı mesajı geldiğinde:

1. Mesaj önce **IntentRouterAgent (LLM)**’a gider.
2. IntentRouter, mesajın niyetini belirler:
   - `NEW_QUESTION`
   - `FOLLOWUP_QUESTION`
   - `DRAW_CHART`
   - `BUSINESS_ERROR_FEEDBACK`
   - `HELP` / `META_QUESTION`
   - `OTHER`
3. Bu intent’e göre LangGraph, ilgili node’lara yönlendirir:
   - Soruysa → **Nlp2SqlAgent → SqlGuardAgent → DbExecuteTool**,
   - Grafik isteğiyse → **ChartPlannerLLMAgent → ChartRuntimeTool → InsightRuntimeTool**,
   - Business feedback ise → **Nlp2SqlAgent** yeni prompt ile tekrar çalışır,
   - Yardım / meta ise → **ConversationAgent** doğrudan açıklama döner.
4. Her akışın sonunda **ConversationAgent** son cevabı üretir ve UI’daki chat + dashboard’a gönderilecek payload’ı hazırlar.

Teknik hatalarda (timeout, syntax error) **RetryController** devreye girer ve sınırlı sayıda tekrar LLM’e SQL revizyonu yaptırır. Business feedback durumlarında ise kullanıcıdan gelen yeni mesaj, IntentRouter üzerinden yeniden yorumlanır ve Nlp2SqlAgent’a “feedback içeren yeni prompt” olarak gider.

### 1.2. LangGraph açısından model

- Bir **StateGraph** kullanıyoruz.
- Global state: `ConversationState` (TypedDict benzeri bir yapı).
- Node’lar:
  - Bazıları **LLM node** (LangChain `ChatModel` + structured output),
  - Bazıları **tool node** (deterministik Python fonksiyonları),
  - Bazıları da basit **control node** (RetryController gibi).

Routing, iki temel sinyale bakarak yapılır:

1. `state["intent"]` – IntentRouter’ın ürettiği intent label,
2. `state["last_event"]` – En son node’un ürettiği event label.

LangChain tarafında:

- LLM çağrıları için `ChatOpenAI` / benzeri bir `BaseChatModel`,
- Structured output için `Pydantic` modelleri veya LangChain’in `with_structured_output` API’si,
- Tool’lar için klasik `@tool` veya normal Python fonksiyonu + LangGraph node bağlama.

---

## 2. Global State ve Event Modeli

### 2.1. ConversationState taslağı

```python
class ConversationState(TypedDict):
    messages: list                       # Chat geçmişi (user + assistant)
    intent: str | None                   # NEW_QUESTION, DRAW_CHART, ...
    sql: str | None                      # Nlp2SqlAgent’ın ürettiği son SQL
    table_result: dict | None            # UI’da gösterilen tablo (satırlar + kolonlar)
    query_run_id: str | None             # query_runs tablosundaki kayıt ID’si
    db_error_message: str | None         # Son DB hatası (timeout, syntax vb.)
    technical_retry_count: int           # Teknik retry sayısı (0–2)
    business_feedback: str | None        # Kullanıcının “bu sonuç yanlış” feedback mesajı
    chart_plan: dict | None              # ChartPlannerLLMAgent’ın ürettiği chart plan
    chart_spec: dict | None              # ChartRuntimeTool’un ürettiği Plotly spec
    insight_specs: list | None           # LLM’den gelen insight şablonları
    insights: list | None                # InsightRuntimeTool ile doldurulmuş insight cümleleri
    last_event: str | None               # Son node’un ürettiği event label
    error: str | None                    # Kullanıcıya gösterilecek hata mesajı (varsa)
```

> Not: `table_result`, sadece **geçici UI state** içindir; sonuç satırlarını metadata DB’de tutmayız. Kalıcı tarafta sadece `query_runs` tablosunda SQL + metadata tutulur.

### 2.2. Event ve event label listesi

Ana event label’lar:

- IntentRouter / general:
  - `INTENT_DETECTED`
  - `USER_ERROR_NO_TABLE` (grafik isteniyor ama tablo yok)
- Nlp2SqlAgent:
  - `SQL_GENERATED`
- SqlGuardAgent:
  - `SQL_VALIDATED`
  - `SQL_REJECTED`
- DbExecuteTool:
  - `QUERY_EXECUTED`
  - `QUERY_TIMEOUT`
  - `QUERY_FAILED`
- RetryController:
  - `SQL_RETRY_REQUESTED`            # Tekrar Nlp2SqlAgent’a git
  - `SQL_RETRY_LIMIT_REACHED`        # Artık ConversationAgent’a dön
- ChartPlannerLLMAgent:
  - `CHART_PLAN_READY`
- ChartRuntimeTool:
  - `CHART_READY`
  - `CHART_ERROR`
- InsightRuntimeTool:
  - `INSIGHTS_READY`
  - (Opsiyonel) `INSIGHTS_PARTIAL`
- ConversationAgent:
  - `RESPONSE_READY`

Event’lerin tamamı **bizim tarafımızdan** set edilir; LLM sadece intent ve bazı structured output alanlarını üretir.

---

## 3. Node Listesi ve Görevleri

### 3.1. IntentRouterAgent (LLM node)

**Rol:** Kullanıcı mesajının niyetini sınıflandıran LLM agent’ı.

- **Tip:** LLM node (LangChain `ChatModel` + structured output).
- **Input:**
  - Son user mesajı (`latest_user_message`),
  - Son 1–2 mesajlık context,
  - State özeti:
    - `has_result = table_result is not None`
    - `last_intent`
    - `last_event`
- **Prompt içeriği (özet):**
  - Sistem neler yapabiliyor (soru → tablo, tablo → grafik, feedback → SQL revizyonu, yardım vs.),
  - Desteklenen intent label’ları ve anlamları:
    - `NEW_QUESTION`
    - `FOLLOWUP_QUESTION`
    - `DRAW_CHART`
    - `BUSINESS_ERROR_FEEDBACK`
    - `HELP`
    - `OTHER`
  - Örnek user cümleleri → örnek intent label’ları.
- **LLM output format:**
  ```json
  { "intent": "<LABEL>", "reason": "<short explanation>" }
  ```
- **Node çıktısı / state update:**
  ```python
  state["intent"] = result.intent
  state["last_event"] = "INTENT_DETECTED"
  if result.intent == "BUSINESS_ERROR_FEEDBACK":
      state["business_feedback"] = latest_user_message
  ```
- **Sonraki node:**
  - `intent` değerine göre conditional edge:
    - `NEW_QUESTION`, `FOLLOWUP_QUESTION`, `BUSINESS_ERROR_FEEDBACK` → `Nlp2SqlAgent`
    - `DRAW_CHART` → `ChartPlannerLLMAgent`
    - `HELP` → `ConversationAgent`
    - `OTHER` → `ConversationAgent` (veya future node).

### 3.2. Nlp2SqlAgent (LLM node)

**Rol:**

- `NEW_QUESTION`, `FOLLOWUP_QUESTION` ve `BUSINESS_ERROR_FEEDBACK` intent’lerinde,
- Kullanıcı sorusundan (ve gerekiyorsa feedback’ten), **SELECT odaklı SQL** üretir veya revize eder.

Ayrıca **teknik retry** sırasında DB hata mesajını da prompt’a alır.

- **Tip:** LLM node.
- **Input:**
  - Orijinal user sorusu,
  - Intent (`NEW_QUESTION` / `FOLLOWUP_QUESTION` / `BUSINESS_ERROR_FEEDBACK`),
  - Seçilmiş şema (ilgili tablolar + kolonlar),
  - Business rule context (RAG, vb.),
  - Eğer `technical_retry_count > 0` ise:
    - `db_error_message` (DB’den gelen hata),
    - Son üretilen SQL (`state.sql`),
  - Eğer `business_feedback` varsa:
    - Kullanıcının “bu sonuç yanlış” açıklaması.

- **Prompt çeşitleri:**
  - **Normal mod (first attempt):**
    - “Bu soruyu, verilen şemaya göre tek bir SELECT ile cevapla. INSERT/UPDATE/DELETE kullanma.”
  - **Teknik retry modunda:**
    - “Önceki SQL şu DB hatasını üretti: … Aynı soruyu, bu hatayı düzeltecek şekilde yeniden yaz.”
  - **Business feedback modunda:**
    - “Kullanıcı sonucun yanlış olduğunu söylüyor ve şu açıklamayı veriyor: ‘…’. Orijinal soru ve şemaya göre, bu feedback’i dikkate alarak yeni bir SELECT yaz.”

- **LLM output:**
  ```json
  { "sql": "SELECT ...", "explanation": "short optional explanation" }
  ```
- **Node çıktısı / state update:**
  ```python
  state["sql"] = result.sql
  # explanation istersek logging veya UI için saklanabilir
  state["last_event"] = "SQL_GENERATED"
  ```
- **Sonraki node:** `SqlGuardAgent`.

### 3.3. SqlGuardAgent (tool node)

**Rol:** Üretilen SQL’in policy’lere uyup uymadığını kontrol eder.

- **Tip:** Deterministik tool node.
- **Input:**
  - `state.sql`,
  - Schema + yetki kuralları (hangi tablo/kolonlara izin var, PII vs.).
- **Kontroller:**
  - Sadece `SELECT` mi? (DDL/DML yasak)
  - Yasak tablo / kolon içeriyor mu?
  - Tenant’ın yetkisi olmayan tabloyu kullanıyor mu?
- **Output / state update:**
  - Geçtiyse:
    ```python
    state["last_event"] = "SQL_VALIDATED"
    ```
  - Kaldıysa:
    ```python
    state["error"] = "Bu sorguya izin verilmiyor. (sebep ...)"
    state["last_event"] = "SQL_REJECTED"
    ```
- **Sonraki node:**
  - `SQL_VALIDATED` → `DbExecuteTool`,
  - `SQL_REJECTED` → `ConversationAgent`.

### 3.4. DbExecuteTool (tool node)

**Rol:** SQL’i tenant DB’de çalıştırır (timeout + row limit ile).

- **Tip:** Deterministik tool node.
- **Input:**
  - `state.sql`,
  - Tenant DB connection bilgileri,
  - `statement_timeout`, `row_limit` konfigürasyonu.
- **Davranış:**
  - SQL’i çalıştırır.
  - Başarılıysa:
    - İlk N satırı `table_result` olarak state’e yazar,
    - `query_runs` tablosunda SQL + metadata kaydeder,
    - `query_run_id`’yi state’e yazar.
- **Output / state update:**
  - Başarı:
    ```python
    state["table_result"] = {...}
    state["query_run_id"] = new_id
    state["last_event"] = "QUERY_EXECUTED"
    state["db_error_message"] = None
    ```
  - Timeout:
    ```python
    state["db_error_message"] = raw_error
    state["last_event"] = "QUERY_TIMEOUT"
    ```
  - Diğer teknik hata:
    ```python
    state["db_error_message"] = raw_error
    state["last_event"] = "QUERY_FAILED"
    ```
- **Sonraki node:** `route_after_db_execute` fonksiyonuna göre:
  - `QUERY_EXECUTED` → `ConversationAgent` (veya opsiyonel post-process),
  - `QUERY_TIMEOUT` / `QUERY_FAILED` → `RetryController`.

### 3.5. RetryController (control node)

**Rol:** Teknik hatalarda sınırlı sayıda LLM’e yeniden SQL ürettiren kontrol node’u.

- **Tip:** Deterministik node.
- **Input:**
  - `last_event` (`QUERY_TIMEOUT` / `QUERY_FAILED`),
  - `technical_retry_count`,
  - `db_error_message`.
- **Davranış:**
  - Eğer `technical_retry_count < 2` ise:
    ```python
    state["technical_retry_count"] += 1
    state["last_event"] = "SQL_RETRY_REQUESTED"
    ```
  - Aksi halde:
    ```python
    state["last_event"] = "SQL_RETRY_LIMIT_REACHED"
    ```
- **Sonraki node:**
  - `SQL_RETRY_REQUESTED` → `Nlp2SqlAgent` (teknik retry prompt’u ile),
  - `SQL_RETRY_LIMIT_REACHED` → `ConversationAgent` (kullanıcıya “3 defa denedik” mesajı).

### 3.6. ChartPlannerLLMAgent (LLM node)

**Rol:** Grafik isteği geldiğinde, hangi tür grafik, hangi X/Y eksenleri, hangi group_by ve hangi parametrik insight şablonlarının üretileceğini planlayan LLM agent.

- **Tip:** LLM node.
- **Input:**
  - Kullanıcının chart isteği mesajı,
  - Son soru ve `sql` string’i,
  - Mevcut `table_result`’ın **kolon isimleri ve tipleri** (değerler yok),
  - `logical_type` bilgileri (numeric, date, categorical, id vs.).
- **Önemli:** `chart_role` gibi manuel alanlar yok; X/Y ekseni ve chart type tamamen buradan çıkar.

- **LLM’den istenen output örneği:**
  ```json
  {
    "chart": {
      "type": "bar",
      "x_axis": "country",
      "y_axis": "total_amount",
      "group_by": null,
      "aggregation": "sum"
    },
    "insights": [
      {
        "template": "Haziran ayında en çok alışveriş yapan müşteri $X.",
        "placeholder": "$X",
        "sql": "SELECT customer_name FROM ... LIMIT 1"
      }
    ]
  }
  ```

- **Node çıktısı / state update:**
  ```python
  state["chart_plan"] = result.chart
  state["insight_specs"] = result.insights
  state["last_event"] = "CHART_PLAN_READY"
  ```
- **Sonraki node:**
  - `CHART_PLAN_READY` → `ChartRuntimeTool`,
  - Aynı zamanda (veya sonrasında) → `InsightRuntimeTool`.

### 3.7. ChartRuntimeTool (tool node)

**Rol:** ChartPlannerLLMAgent’ın ürettiği `chart_plan` + `table_result`’ı kullanarak Plotly chart spec üretmek.

- **Tip:** Deterministik tool node.
- **Input:**
  - `chart_plan` (type, x_axis, y_axis, group_by, aggregation),
  - `table_result` (kolonlar + satırlar).
- **Davranış:**
  - Uygun kolonları tablo içinden bularak Plotly JSON config oluşturur.
  - Eğer uygun numeric kolon vs. yoksa `CHART_ERROR` üretebilir.
- **Output / state update:**
  - Başarılı:
    ```python
    state["chart_spec"] = plotly_spec
    state["last_event"] = "CHART_READY"
    ```
  - Hata:
    ```python
    state["error"] = "Bu tablo için istenen tipte grafik oluşturulamadı."
    state["last_event"] = "CHART_ERROR"
    ```
- **Sonraki node:**
  - `CHART_READY` → `InsightRuntimeTool` veya doğrudan `ConversationAgent`,
  - `CHART_ERROR` → `ConversationAgent`.

### 3.8. InsightRuntimeTool (tool node)

**Rol:** ChartPlannerLLMAgent’tan gelen `insight_specs` içindeki SQL’leri çalıştırarak parametrik insight cümlelerini doldurmak.

- **Tip:** Deterministik tool node.
- **Input:**
  - `insight_specs` listesi:
    - `template`, `placeholder`, `sql`
- **Davranış:**
  - Her insight için:
    1. `sql` → `SqlGuardAgent` (SELECT-only ve security kontrolü),
    2. Sonra `DbExecuteTool` ile (mümkünse daha küçük timeout/limit ile) çalıştır,
    3. Dönen satırdan placeholder değerini çıkar (`$X` → `"ABC A.Ş."` gibi),
    4. `template` string içinde placeholder ile değiştir.
- **Output / state update:**
  - Başarılı tüm insight’lar:
    ```python
    state["insights"] = [
        "Haziran ayında en çok alışveriş yapan müşteri ABC A.Ş.",
        ...
    ]
    state["last_event"] = "INSIGHTS_READY"
    ```
  - Bazıları hata alırsa (isteğe bağlı):
    - Hata alanları loglayıp, kalanları doldurur → `INSIGHTS_PARTIAL`.
- **Sonraki node:**
  - `INSIGHTS_READY` / `INSIGHTS_PARTIAL` → `ConversationAgent`.

### 3.9. ConversationAgent (tool node – finalizer)

**Rol:** Tüm state’i okuyup UI’ya gidecek **chat cevabını** ve ek payload’ı üretir.

- **Tip:** Deterministik node (isteğe bağlı mini LLM yardımı olabilir ama core mantık deterministik).
- **Input:**
  - `messages` (history),
  - `intent`, `last_event`,
  - `table_result`, `chart_spec`, `insights`,
  - `error` (varsa),
  - `query_run_id` (export butonları için link üretmek istersek).

- **Davranış:**
  - `last_event` ve `intent`’e göre farklı template’ler:
    - `QUERY_EXECUTED` → “Tablon hazır, istersen grafiğe dönüştürebiliriz...” + tabloyu UI’a gönder.
    - `CHART_READY` + `INSIGHTS_READY` → “Grafiği oluşturdum, bazı insight’lar: …”
    - `USER_ERROR_NO_TABLE` → “Henüz bir tablo yok, önce soru sormalısın.”
    - `SQL_RETRY_LIMIT_REACHED` → “3 kez denedik, teknik hata devam ediyor.”
    - `SQL_REJECTED` → Policy temelli kibar bir uyarı.
  - UI tarafı:
    - Chat balonunda metin,
    - Yanında/altında tablo, grafik ve insight komponentleri.

- **Output / state update:**
  ```python
  state["messages"].append(ai_message)
  state["last_event"] = "RESPONSE_READY"
  ```

Bu node’dan sonra frontend cevabı gösterir, yeni user mesajı geldiğinde döngü baştan IntentRouter ile tekrar başlar.

---

## UX ve İdempotency Notları

- **Sorgula butonu:** Kullanıcı sorgu gönderdikten sonra `RESPONSE_READY` (veya hata) olana kadar UI’de “Sorgula” butonu disabled kalmalı; çift tıklamayla aynı isteğin tekrarlanması önlenir.
- **Idempotency için request_id:** İsteklerde (sorgu, export, grafik) opsiyonel `request_id`/`Idempotency-Key` taşınabilir; backend aynı `request_id` ile gelen tekrarlara daha önceki sonucu döndürür, usage sayaçları iki kez artmaz.
- **Satır verisi saklama:** Row data metadata DB/Redis’te tutulmaz; tekrar çizim gerekirse SQL yeniden çalıştırılır.
- **Timeout/limit sınırları:** Tekrar denemelerde (teknik retry) zaman aşımı kademeli artırılır; usage artışı yalnızca başarılı çalıştırmada yapılmalıdır.
- **Query/Grafik/Export idempotency:** Query, grafik ve export uçları `request_id` kabul eder; aynı `request_id` ile gelen tekrarlarda mevcut sonuç döndürülür veya job durumu raporlanır, yeni DB çalıştırması/usage artışı yapılmaz.

---

## 4. LLM’in Intent Routing’e Yardımı

### 4.1. IntentRouter system prompt mantığı

IntentRouter’ın LLM prompt’u, sistemin **konsept olarak** neler yapabildiğini anlatır:

- Soru → SQL → tablo üretmek,
- Var olan tabloyu grafiğe dönüştürmek,
- Mevcut sonucun yanlış olduğuna dair feedback alıp SQL’i revize etmek,
- Yardım / meta sorularını cevaplamak.

Ve desteklenen intent label’lerini şu şekilde tarif eder:

1. `NEW_QUESTION` – Sistemin yeni bir tablo üretmesini isteyen bağımsız soru.
2. `FOLLOWUP_QUESTION` – Mevcut sonucu referans alan, ama yeni tablo üretmek isteyen soru.
3. `DRAW_CHART` – Mevcut tabloyu grafiğe dönüştürme isteği.
4. `BUSINESS_ERROR_FEEDBACK` – Mevcut sonucun mantık olarak yanlış olduğunu, nedenleriyle açıklayan feedback.
5. `HELP` – Sistemle ilgili yardım / açıklama isteği.
6. `OTHER` – Yukarıdakilerin hiçbirine uymayan durum.

LLM’den sadece:

```json
{ "intent": "<LABEL>", "reason": "<short explanation>" }
```

dönmesi istenir. **Event label’ları LLM üretmez**; event’ler tamamen bizim kod tarafımızda set edilir.

### 4.2. Business feedback’in yakalanması

- IntentRouter, user mesajında “bu sonuç yanlış”, “X olmalıydı” gibi pattern’ler ve context’e (mevcut tablo var mı?) bakarak yüksek ihtimalle `BUSINESS_ERROR_FEEDBACK` seçer.
- Biz de:
  ```python
  state["business_feedback"] = latest_user_message
  ```
  ile bu feedback’i Nlp2SqlAgent’a taşırız.

### 4.3. Teknik retry’de IntentRouter’ın devre dışı kalması

- Teknik retry senaryosunda **yeni bir user mesajı yoktur**.
- DB hatası → `DbExecuteTool` → `RetryController` → `Nlp2SqlAgent` döngüsü, tamamen graph içinde döner.
- IntentRouter sadece “kullanıcı mesajı geldiğinde” çalışır; teknik retry’de devreye girmez.

---

## 5. Edge Case Senaryoları – 5 Örnek Akış

Aşağıdaki senaryolarda, kullanıcı ↔ sistem arasındaki interaction ve event akışını özetliyoruz.

### 5.1. Edge Case 1 – Tablo yokken grafik isteği (USER_ERROR_NO_TABLE)

**Durum:** Kullanıcı sistemi açar açmaz:  
> “Bana ihracat grafiği çiz.”

1. Yeni user mesajı → **IntentRouterAgent (LLM)**:
   - Context: `table_result is None`.
   - LLM: Bu cümle “grafik isteği” → `intent = "DRAW_CHART"`.
   - State: `intent="DRAW_CHART"`, `last_event="INTENT_DETECTED"`.
2. Routing:
   - Normalde `DRAW_CHART` → `ChartPlannerLLMAgent`, ama biz önce rule check yaparız:
     - `if intent == "DRAW_CHART" and table_result is None:`
       ```python
       state["last_event"] = "USER_ERROR_NO_TABLE"
       ```
3. `USER_ERROR_NO_TABLE` → **ConversationAgent**:
   - Cevap:
     > “Şu anda grafik çizebileceğim bir tablo yok. Önce bir soru sorup tablo oluşturalım, sonra ‘bu tabloyu grafiğe dök’ diyebilirsin.”
   - `last_event="RESPONSE_READY"`.

ChartPlannerLLMAgent ve ChartRuntimeTool hiç çağrılmaz.

---

### 5.2. Edge Case 2 – DB timeout, teknik retry ile ikinci denemede başarı

**Durum:** Kullanıcı:  
> “Son 12 ayda ülkelere göre aylık toplam ihracatımı göster.”

İlk denemede sorgu ağır geliyor ve timeout oluyor, ikinci denemede Nlp2SqlAgent daha optimize SQL üretiyor.

1. User mesajı → **IntentRouterAgent**:
   - `intent="NEW_QUESTION"`, `INTENT_DETECTED`.
2. → **Nlp2SqlAgent**:
   - İlk SQL’i üretir → `SQL_GENERATED`.
3. → **SqlGuardAgent**:
   - Policy OK → `SQL_VALIDATED`.
4. → **DbExecuteTool**:
   - DB’de timeout alır:
     ```python
     state["db_error_message"] = "statement timeout ..."
     state["last_event"] = "QUERY_TIMEOUT"
     ```
5. `QUERY_TIMEOUT` → **RetryController**:
   - `technical_retry_count=0` → `< 2`
   - `technical_retry_count = 1`,
   - `last_event = "SQL_RETRY_REQUESTED"`.
6. Routing: → **Nlp2SqlAgent** (retry modunda):
   - Prompt’ta:
     - Orijinal soru,
     - İlk SQL,
     - `db_error_message` (timeout açıklaması),
     - `technical_retry_count = 1` bilgisi.
   - LLM daha basit / daha dar bir SQL üretir → `SQL_GENERATED`.
7. → **SqlGuardAgent**:
   - Yine OK → `SQL_VALIDATED`.
8. → **DbExecuteTool**:
   - Bu sefer query zamanında biter → `QUERY_EXECUTED`, `table_result`, `query_run_id`.
9. → **ConversationAgent**:
   - Tabloyu UI’a gönderir,
   - “İstersen bunu grafiğe dönüştürebiliriz” tarzı metin.
   - `RESPONSE_READY`.

Teknik retry, IntentRouter’ı hiçbir aşamada yeniden çağırmaz.

---

### 5.3. Edge Case 3 – İş mantığı feedback’i (BUSINESS_ERROR_FEEDBACK) ile SQL revizyonu

**Durum:**

1. Kullanıcı soru sorar → tablo gelir.
2. Kullanıcı sonucunu inceler ve chat’e yazar:  
   > “Bu tablo yanlış, Haziran ayında en çok satış yaptığım müşteri X olmalıydı, ama burada görünmüyor.”

Akış:

1. İlk soru ve tablo üretimi normal flow ile yapılır (5.2’ye benzer).
2. Yeni user mesajı (feedback) → **IntentRouterAgent**:
   - Context: `has_result = True`, `last_intent="NEW_QUESTION"`,
   - Mesaj: “Bu tablo yanlış …” → çok büyük ihtimalle `intent="BUSINESS_ERROR_FEEDBACK"`.
   - State:
     ```python
     state["intent"] = "BUSINESS_ERROR_FEEDBACK"
     state["business_feedback"] = latest_user_message
     state["last_event"] = "INTENT_DETECTED"
     ```
3. Routing: `BUSINESS_ERROR_FEEDBACK` → **Nlp2SqlAgent** (business retry modunda):
   - Prompt’ta:
     - Orijinal soru,
     - Önceki SQL,
     - (İstersek) Önceki tablonun özet bilgisi,
     - Kullanıcının feedback’i (`business_feedback`).
   - LLM, yeni bir SELECT üretir → `SQL_GENERATED`.
4. → **SqlGuardAgent**:
   - Policy check → `SQL_VALIDATED`.
5. → **DbExecuteTool**:
   - Yeni SQL çalıştırılır → `QUERY_EXECUTED`, `table_result` güncellenir, yeni `query_run_id` yazılır.
6. → **ConversationAgent**:
   - Kullanıcıya:
     > “Feedback’inizi dikkate alarak sorguyu revize ettim, yeni tabloyu aşağıda görebilirsiniz.”  
   - `RESPONSE_READY`.

Teknik retry’den farkı:

- Yeni user mesajı olduğu için IntentRouter devrede,
- `business_feedback` state üzerinden Nlp2SqlAgent’a taşınıyor,
- Prompt’ta DB hata mesajı yok, sadece kullanıcı feedback’i var.

---

### 5.4. Edge Case 4 – Grafik isteği + insight, ama uygun numeric kolon yok (CHART_ERROR)

**Durum:**

1. Kullanıcı bir tablo üretir ama tablonun tüm kolonları metinsel / kategorik veya ID’dir.
2. Sonra:  
   > “Bunu bar chart yap ve anlamlı 2–3 insight üret.”

Akış:

1. Tablo üretimi → normal flow (`NEW_QUESTION` → `Nlp2SqlAgent` → `DbExecuteTool` → `table_result`).
2. Yeni user mesajı → **IntentRouterAgent**:
   - `intent="DRAW_CHART"`, `INTENT_DETECTED`.
3. Routing → **ChartPlannerLLMAgent**:
   - Input: chart isteği + kolon isimleri + logical_type.
   - LLM, belki yine bar chart planı üretir ama numeric y-axis bulmaya çalışır:
     - Eğer numeric kolon yoksa, prompt’ta belirttiğimiz kurala göre:
       - Boş `chart_plan` veya `chart.type = "count_per_category"` gibi bir fallback döndürebilir.
4. → **ChartRuntimeTool**:
   - `chart_plan`’ı uygulamaya çalışır.
   - Eğer gerçekçi bir numeric kolon bulunamazsa:
     ```python
     state["error"] = "Bu tablo için istenen tipte anlamlı bir grafik oluşturulamadı."
     state["last_event"] = "CHART_ERROR"
     ```
5. Routing: `CHART_ERROR` → **ConversationAgent**:
   - Kullanıcıya:
     > “Bu tablodaki kolonlar sayısal bir metrik içermediği için anlamlı bir bar chart üretemiyorum. İstersen önce toplam/ortalama gibi bir metrik üreten bir sorgu çalıştırıp onu grafiğe dönüştürebiliriz.”
   - `RESPONSE_READY`.

Burada, **InsightRuntimeTool** hiç devreye girmez; önce grafik planı bile kurulamazsa insight üretmenin anlamı yok.

---

### 5.5. Edge Case 5 – 3 teknik denemeden sonra hâlâ DB hatası (SQL_RETRY_LIMIT_REACHED)

**Durum:**

- Kullanıcı çok karmaşık veya tutarsız bir soru soruyor,
- Nlp2SqlAgent 3 farklı SQL üretiyor,
- Her seferinde DB tarafında teknik hata oluşuyor (örneğin eksik kolon, yanlış şema varsayımı, vs.).

Akış:

1. User mesajı → IntentRouter (`NEW_QUESTION`) → Nlp2SqlAgent → SqlGuardAgent → DbExecuteTool.
2. İlk deneme: `QUERY_FAILED` + `db_error_message="column X does not exist"`.
3. → **RetryController**:
   - `technical_retry_count=0` → `SQL_RETRY_REQUESTED`, `technical_retry_count=1`.
4. → **Nlp2SqlAgent** (retry #1):
   - Prompt’ta eski SQL + DB error + orijinal soru.
   - Yeni SQL → `SQL_GENERATED` → SqlGuard → DbExecute.
5. İkinci deneme: yine `QUERY_FAILED` (başka bir kolon error’u).
6. → **RetryController**:
   - `technical_retry_count=1` → `SQL_RETRY_REQUESTED`, `technical_retry_count=2`.
7. → **Nlp2SqlAgent** (retry #2):
   - Tekrar yeni bir SQL.
8. Üçüncü deneme: yine `QUERY_FAILED`.
9. → **RetryController**:
   - `technical_retry_count=2` → artık `< 2` değil,
   - `last_event = "SQL_RETRY_LIMIT_REACHED"`.
10. Routing: `SQL_RETRY_LIMIT_REACHED` → **ConversationAgent**:
    - Chat cevabı:
      > “Üzgünüz, bu isteğiniz için arka planda 3 kez sorgu üretmeyi denedik ancak her seferinde veritabanı teknik hata döndürdü. Sorguyu daha basit parçalara ayırarak tekrar deneyebilir veya IT ekibinizle görüşebilirsiniz.”
    - `RESPONSE_READY`.

Bu senaryoda:

- IntentRouter sadece ilk user mesajında devrede,
- Tüm retry mekanizması `DbExecuteTool` → `RetryController` → `Nlp2SqlAgent` döngüsü içinde,
- Kullanıcıya gereksiz teknik detay vermeden “3 kez denedik, olmadı” mesajı gösteriliyor.

---

## 6. Özet

Bu mimaride:

- **LLM node’ları**:
  - `IntentRouterAgent` – intent routing,
  - `Nlp2SqlAgent` – SQL üretimi / revizyonu,
  - `ChartPlannerLLMAgent` – chart planı + parametrik insight şablonları.
- **Deterministik node’lar / tool’lar**:
  - `SqlGuardAgent` – güvenlik ve policy,
  - `DbExecuteTool` – veritabanı erişimi,
  - `RetryController` – teknik retry kontrolü,
  - `ChartRuntimeTool` – Plotly chart spec üretimi,
  - `InsightRuntimeTool` – parametrik insight’ları doldurma,
  - `ConversationAgent` – son cevabı UI’ya hazırlama.
- **Event tabanlı routing** sayesinde:
  - Intent routing net, deterministik,
  - Teknik retry ve business retry birbirine karışmıyor,
  - Grafik ve insight süreçleri LLM plan + deterministik icra şeklinde kurgulanıyor,
  - Query sonuç satırlarını kalıcı saklamadan, sadece SQL + metadata üzerinden export ve loglama yapılabiliyor.

Bu doküman, LangGraph’te node’ları ve edge’leri kurarken, ayrıca LangChain tarafında hangi LLM çağrılarının nasıl yapılandırılacağını tasarlarken “single source of truth” olarak kullanılabilir.
