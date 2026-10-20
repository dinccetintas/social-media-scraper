# 07 — Devil's Advocate / Principal Reviewer: Eleştirel İnceleme

**Persona 7 — Principal Reviewer** · Tarih: 2026-10-20 · Girdi: 01–06 numaralı raporların tamamı. Bu rapor diğerlerini değiştirmiyor. Çelişkileri çözüyor, riskleri sıralıyor ve MVP'yi daraltıyor. Bölüm atıfları "P3 §2.2" biçiminde. Doğrulamalar ve kaynakları §5'te.

---

## 0) TL;DR

1. Altı rapor tek tek bakıldığında sağlam, ama yan yana konunca **iki ayrı sistem** tarif ediyorlar:
   - P5'in "ucuz-iyi" stack'i: Gemini 3.1 FL, default çözünürlük, Haiku, Supadata, SQLite/pgvector.
   - P1/P3/P4/P6'nın stack'i: 3.5 FL `high`, Sonnet ile extraction, D1 + Vectorize + voyage + rerank, 11 tool, plugin marketplace.

   İkisi arasında maliyet ve karmaşıklık farkı 2–3 kat.
2. **En büyük tek risk, YouTube altyazısının datacenter IP'den alınması.** GitHub Actions runner'ları Azure IP'si kullanıyor ve `web_embedded` yolu orada hiç test edilmedi. P4'ün "ana tetik GH Actions" kararı P2'nin bulgusuyla çatışıyor.
3. **İkinci SPOF, Gemini YouTube URL özelliği (preview).** Buluttan video indirilemiyor (P2 T6/T10). Bu özellik kalkarsa bulutta YouTube görsel analizinin **yedeği kalmıyor**. P3'ün önerdiği "indir → Files API" fallback'i yalnızca laptop'ta işe yarıyor.
4. **MVP fazla mühendislik içeriyor.** Çekirdek değer şu: "kanal X'ten timestamp'li atıfla cevap". Bunun için gerekenler: altyazı, Sonnet ile extraction, kodla alıntı doğrulama, FTS5, tek bir remote MCP ve tek bir skill. Vektör, rerank, üç ayrı görsel geçiş, Batch, wiki derleyicisi ve per-KB plugin sonraya kalabilir.
5. **Gerçekçi maliyet daha yüksek.** 100 × 15 dk YouTube videosu için P5'in $1.7–3.3 tahmini yerine **≈ $2.6 (Haiku) – $6.2 (Sonnet)** çıkıyor. Workers Free ile ayda ≈ $2.5–5.5 tutuyor. Bütçe yetiyor, ama "all" seçeneği ve uzun canlı yayınlar için ön tahmin ve `--max-usd` kapısı şart.

---

## 1) Raporlar arası çelişkiler ve çözüm

| # | Konu | Kim ne diyor | Karar |
|---|---|---|---|
| 1 | Gemini varsayılan modeli | P3 §1: **3.5 Flash-Lite** ($0.30 girdi / $2.50 çıktı). P5 §1.4, §5.1: **3.1 Flash-Lite** ($0.25 / $1.50) | **3.5 FL.** 3.1 FL 7 Mayıs 2027'de kapanıyor (§5'te doğrulandı). P5'in kendi "fiyat riski" argümanı da 3.1'i eliyor. İkisi arasındaki fark 100 videoda $1'ın altında. |
| 2 | Görsel çözünürlük | P3 §2.2: kod/URL okumak için `high` (280 tok/kare) şart, deneyle gösterilmiş. P5 §4.3: maliyet `default` (70) ile hesaplanmış | **`high` @0.5 fps**, yalnızca işaretli videolarda. P5'teki görsel kalem yaklaşık 2 kat düşük; bütçe P3 §5.4'e göre alınmalı. |
| 3 | Token oranları | P3: 70/280 tok/kare, ses 25 tok/sn. P2 ve P5 §4.1: ses 32 tok/sn | Google'ın kendi sayfaları da çelişkili; video-understanding sayfası hâlâ 66/258 ve 32 diyor (§5). Pilotta `countTokens` ile ölçülmeli. |
| 4 | Görsel geçişe giren pay | P5 §4.1: videoların %40'ı. P3 §3: %80 (light %50 + heavy %30) | **Light geçiş kaldırılsın (§3).** Seçim ikili olsun: none ya da heavy. Heavy hedefi %30–40. |
| 5 | Extraction modeli | P5 §1.3: Haiku 5.5 Batch ($0.09/100 video). P6 §8: uzun videoda Sonnet 5.5, reel'de Haiku ($3–5/100) | **v0.1'de Sonnet 5.5, standart API.** KB kalitesini belirleyen tek çağrı bu ve fark 100 videoda yaklaşık $3.6. P5 ve P6'nın ikisinin de önerdiği 5 videoluk A/B testinden sonra Haiku'ya geçilebilir. |
| 6 | Embedding | P1 §2.4: voyage-4 (1024d). P5 §4.3: Gemini Embedding 2 veya bge-m3, Vectorize hesabı 768d ile | **v0.x'te embedding yok (§3).** Sonradan eklenirse seçimi P1 §3.6'daki eval matrisi yapar. |
| 7 | Depolama | P1 §2.2 ve P4 §3: D1 + Vectorize + R2. P5 §5.1: Workers üzerinde "SQLite + sqlite-vec veya pgvector" (Workers sqlite-vec yükleyemez). P5 §5.2: Hetzner | **D1 FTS5 + lokal `kb.sqlite`, ikisi aynı şemada.** P5'in ifadesi teknik olarak tutarsız. |
| 8 | Workers planı | P1 §4: 100 video Free'ye sığar. P4 §1.2: Paid ($5) gerekli, çünkü "Vectorize free'de belirsiz" | **Free ile başlanmalı.** Vectorize Free planda mevcut (§5). 10 ms CPU sınırına takılınca Paid'e geçilir. $5, $10'luk bütçenin yarısı. |
| 9 | Ingestion nerede çalışacak | P4 §1.5, §7: GH Actions ana tetik. P2 §4.1: GH Actions da datacenter IP'si; ücretsiz yol yalnızca kırılgan `web_embedded`, ses indirme yok. P5 §7: bulutta Supadata | **v0.1'de Claude Code cloud oturumu** (P2 testleri bu ortamda yapıldı). GH Actions v1'e kalır; önce bir smoke test (flat-playlist + `web_embedded` altyazı) geçmeli. |
| 10 | Transkript sağlayıcısı | P2 §4.1: ScrapeCreators (süresiz kredi, altyazı yoksa ücret yok, IG fallback'i de olur). P5 §5.1: Supadata (ayda 100 ücretsiz kredi) | **Tek adapter arayüzü, tek aktif sağlayıcı.** Pilotta iki sağlayıcının ücretsiz kotalarıyla `web_embedded` başarı oranı ölçülür, karar ondan sonra verilir (§7, soru 2). |
| 11 | Altyazısız video | P2 §4.1: laptop'ta Groq, bulutta Supadata generate veya Gemini. P5 §5.1: Gemini tam geçiş | **Gemini URL ile transkript** (`transcript_source=gemini`, düşük güven etiketiyle). IG için de Gemini'de tek çağrı yapılırsa **Groq tamamen çıkar.** |
| 12 | vidIQ | P4 §1.6 ve §6: 1–10 video ingestion için uygun. P2 §3 T13: bakiye 0, çağrı başına 5 kredi, IG'de en fazla 12 reel | **Pipeline'dan çıkarılmalı.** P2'nin test sonucu P4'teki tabloyu geçersiz kılıyor. |
| 13 | ID'ler ve dizin yolları | P1/P4: `kb_id=yt-<handle>`, `video_id=yt:ID`, segment `yt:ID#s0042`, chunk `<video_id>:<start_ms>`. P6 §5: `yt-Q7x…`, `#i03`, `kb/youtube/<handle>/`. P4 §7: `kb/<kb_id>/videos/<id>.json` + `manifest.json` | **Tek kural:** `video_key = yt-<ID>` / `ig-<shortcode>`. İki nokta üst üste dosya adlarını Windows'ta ve Obsidian'da bozuyor. Ref'ler `<video_key>#s0042`, `#v0007`, `#i03`; dizin `kb/<kb_id>/…`. **Birincil anahtar değişmeyen `channel_id` (UC…) olmalı.** Handle değişebildiği için `kb_id` yalnızca bir slug ve alias. |
| 14 | Tool adları | P4 §4: 11 tool. P6 §9: `kb_search`, `kb_read`, `kb_get_segment`. P3 §7: 4 ayrı görsel tool. P6'nın persona'sı dosya yolu okuyor (`kb/{p}/{h}/overview.md`), P4'ün skill'i tool çağırıyor | **6 tool (§3.1).** Persona talimatında yalnızca tool'lar geçmeli. Dosya okuma, repo klonlanmış ortamlarda yedek yol olarak kalır. |
| 15 | Tip enum'ları | P1/P4'te `insight` tipi var, P6 ontolojisinde yok. P4'te `template` var, P6'da yok. P3'ün `kind` alanında `package`, `tool_or_product`, `ui_step` var. P3'ün görsel öğe ID'leri (`i1…`) P6'nın item ID'leriyle çakışıyor | `chunk.type ∈ {segment, visual, item, page}` ve bir `item_kind` filtresi; P6 enum'u kanonik kabul edilir. P3'ün çıktısı normalize edilirken `v####` segmentlerine dönüştürülür. Eşleme tablosu tek yerde, `schema/` altında durur. |
| 16 | Derin link biçimi | P3 §4: `youtu.be/{id}?t=`. Diğerleri: `watch?v=ID&t=Ns` | İkincisi. Tek bir kod fonksiyonu üretir. |
| 17 | Timestamp'i kim yazar | P6 §1.3: "LLM timestamp yazmaz, segment ID yazar". P3 §4'teki prompt modelden MM:SS istiyor | Görselde bu kaçınılmaz, çünkü önceden segment yok. Ancak `v####` zamanları model tahmini (±5 sn). Ekran alıntısı modelin kendi OCR'ına karşı "doğrulandığı" için doğrulama döngüsel. Bu yüzden `verified` yerine `screen_model` etiketi kullanılmalı ve atıfta "≈" gösterilmeli. |
| 18 | Cloud oturum limitleri | P4 §7: "yaklaşık 2 dk foreground / 30 dk background" | Bunlar bu oturumdaki Bash tool'unun **varsayılan** değerleri. Tool tanımına göre üst sınır foreground için 10 dk, background için 2 saat (gözlem). 100 video background'da sığar; "all" sığmaz, bu yüzden resume şart. |
| 19 | Embedding'ler git'te | P1 §3.1: `chunks.jsonl(+emb)` SoT repo'sunda tutuluyor | **Reddedilmeli.** 20k × 1024 float JSON olarak ~150–200 MB eder ve her yeniden embed'de git şişer. Embedding türev veridir: R2/parquet'te tutulur ya da yeniden üretilir. |

### 1.1 Uzlaştırılmış maliyet (100 × 15 dk YouTube, standart API)

| Kalem | Haiku yolu | Sonnet yolu |
|---|---|---|
| Edinim, bulut, en kötü durum (P2 §4.3) | $0.2 | $0.2 |
| Altyazısız %5 → Gemini URL | $0.2 | $0.2 |
| Görsel heavy: %35 × $0.055 (3.5 FL, high @0.5 fps) | $1.9 | $1.9 |
| Extraction (~6.5k girdi / 2.5k çıktı) | $0.2 | $3.8 |
| Overview + index | $0.1 | $0.1 |
| **Toplam** | **≈ $2.6** | **≈ $6.2** |

Notlar:
- YouTube URL girdisi gerçekten ücretsizse görsel kalem yaklaşık %80 düşer (doğrulanmadı).
- Geliştirme sırasındaki yeniden işlemeler için +%30–100 eklenmeli.
- Aylık (~87 video): $2.3–5.4. Workers Paid eklenirse $7–10.5 ile tavana dayanıyor.
- IG, 100 reel: ≈ $1.2 (Apify $0.26 + Gemini high $0.89 + Haiku).

---

## 2) En zayıf varsayımlar ve SPOF'lar (olasılık × etki, 1–5)

| # | Varsayım / SPOF | O | E | Skor | Neden zayıf | Azaltma |
|---|---|---|---|---|---|---|
| 1 | Datacenter IP'den YouTube altyazısı (`web_embedded`) | 5 | 4 | **20** | Resmî bir yol değil; yalnızca embed'e izin veren videolarda çalışıyor; yt-dlp her hafta kırılıyor (P2 §5). GH runner'da test edilmedi. youtube-transcript-api README'sine göre Azure IP'leri bloklu. | Vendor adapter'ı v0.1'de hazır olmalı. Her run'da `transcript_source` dağılımı raporlanmalı. Haftalık canary testi. |
| 2 | Gemini YouTube URL (preview; "pricing and rate limits are likely to change") | 3 | 4 | **12** | Buluttan indirme yapılamadığı için yedeği yok. Google'ın videoyu hangi çözünürlükte aldığı belgelenmemiş, dolayısıyla kod okuma tavanı bilinmiyor (P3 §2.1). | 10 videoluk altın setle kod/URL recall ölçülmeli. Plan B: ev makinesinde self-hosted runner + Files API. Görsel katman isteğe bağlı tasarlanmalı; KB onsuz da çalışmalı. |
| 3 | "Doğrulanmış alıntı" garantisi | 3 | 3 | **9** | Altyazı yoksa transkripti Gemini üretiyor; ekran alıntıları modelin kendi OCR'ı. İkisinde de doğrulama döngüsel. Auto-caption'da ASR hataları var. | Güven sırası: manuel > auto-orig > vendor > gemini. Bu sıra `evidence.source` alanına yazılır ve agent'a "≈ (ASR/model)" etiketiyle gösterilir. |
| 4 | Remote MCP auth zinciri | 3 | 3 | **9** | claude.ai ve Desktop için tek yol bu. Hazır template'ler deprecated `McpAgent` kullanıyor. MCP 2026-07-28 Cloudflare'de hâlâ RC. `workers-oauth-provider`'ın CIMD desteği doğrulanmadı. | **Hafta 1'de spike:** Worker + GitHub OAuth + claude.ai connector + Claude Code loopback. DCR yeterli; CIMD alanları yoksa Claude DCR'a düşüyor (§5). |
| 5 | Prompt injection | 3 | 3 | **9** | AI ve Claude Code kanallarında ekranda ve konuşmada **prompt metni olması kesin** (demolar). Bu metin wiki'ye "güvenilir sayfa" olarak sızabilir. Scheduled task'larda connector tool'ları onay almadan çalışıyor (P4 §2). | Extraction çağrısı tool'suz yapılmalı (yalnızca structured output). MCP yanıtında içerik `untrusted` alanında dönmeli. Yazma tool'u `requiresUserInteraction` ve bütçe sınırı arkasında olmalı. Sohbet ortamlarında API anahtarı bulunmamalı (§4). |
| 6 | Maliyet patlaması | 4 | 2 | **8** | "all" ile 2 saatlik yayınlar `high` @0.5 fps'te ≈ 1.2M token tutar; pencereleme gerekir ve video başı ~$0.4'e çıkar. Prompt iterasyonları yeniden işleme demek. Thinking token'ları çıktı fiyatından ($2.50/M) faturalanıyor. | `--dry-run` ile ön tahmin, `--max-usd`, varsayılan `max_duration=90dk`, Shorts ve canlı yayınlar opt-in. |
| 7 | Instagram edinimi | 3 | 2 | **6** | Apify actor'ı değişebilir; transkript add-on'unun fiyatı listelenmemiş. Logged-out vendor kullanıldığında hukuki risk düşük (P2 §5). | Kendi hesabıyla login'li scraping yasak. ScrapeCreators v1.x'te ikinci adapter olur. |
| 8 | Türkçe lexical arama | 3 | 2 | **6** | FTS5'te Türkçe stemmer yok; TR soru ↔ EN kanal eşleşmesi zor. | `statement_en` + TR alias'lar köprü görevi görür. Skill her aramayı TR ve EN iki sorguyla yaptırır. Eval'de Recall@10 < 0.8 çıkarsa vektör eklenir. |
| 9 | Belgelenmemiş davranışlar | 3 | 2 | **6** | Synced plugin'in cloud oturumuna gelmesi yalnızca gözlemlendi (P4 §9.3). claude.ai'nin tool sonucu limiti bilinmiyor. | Skill hem claude.ai skill'i hem repo içi `.claude/skills` olarak dağıtılır. Yanıtlar ≤4k token tutulur. |
| 10 | Model ömrü ve fiyat değişimi | 5 | 1 | 5 | 3.1 FL Mayıs 2027'de kapanıyor; 3.6/3.8 Flash'ın fiyatı 1 Ocak 2027'de 2 katına çıkıyor (§5). | Model adı ve fiyat `config.yaml`'da. Maliyet gerçek `usage` verisinden hesaplanır. |
| 11 | Handle'a bağlı kimlik | 2 | 2 | 4 | P1 §3.2'de `channels` PK'si `yt-<handle>`. | `channel_id` PK yapılmalı. |

---

## 3) YAGNI: ne kesilir, ince dikey dilim, yol haritası

### 3.1 Kes veya ertele

| Bileşen | Kaynak | Karar | Gerekçe |
|---|---|---|---|
| Vectorize + voyage-4 + rerank-2.5 | P1 §1.4–5 | v1 sonrasına, eval kapısına bağlı | 100 video yaklaşık 2k chunk eder; agent zaten çok adımlı arama yapıyor. Item'lardaki `statement_en` çapraz dil köprüsünü kuruyor. Rerank'in asıl gerekçesi token bütçesiydi; k=8 ve ≤250 token'lık snippet'ler bunu zaten çözüyor. |
| Lokal sqlite-vec | P1 §2.2 | Kes | Yukarıdakiyle aynı gerekçe. |
| Light görsel geçiş (default, 0.2 fps) | P3 §3 | Kes | Default çözünürlük küçük metni okuyamıyor (P3 §2.2). Geçiş ya heavy olmalı ya hiç olmamalı. |
| Re-check (3.8 Flash), keyframe/pHash/RapidOCR yolu | P3 §1.6, §3(6) | v1.1 | Buluttan YouTube indirilemediği için keyframe yolu zaten çalışmıyor. |
| LLM router | P3 §3(2) | Heuristik router + `--visual=auto/all/none` | Tamamen deterministik. Router şüpheye düştüğünde heavy seçer. |
| Batch API, Gemini Flex | P3 §5, P5 §5 | v1.x | Tasarruf 100 videoda ~$2–3. Karşılığında asenkron iş takibi, 24 saate varan gecikme ve resume karmaşıklığı geliyor. |
| Wiki derleyicisi (topic/framework/playbook sayfaları, registry, relations/evolution, lint, `_topics`) | P6 §4–5 | v1.x | MVP'de yeterli olanlar: video notları + `overview.md` + `index.md` + kodla üretilen `resources.md`. Kanonikleştirme olmayınca tekrarlar olur, ama arama bunları tolere eder. |
| 11 tool | P4 §4 | **6'ya indir** | `list_kbs`, `get_overview(kb_id)` (overview + index), `search(query, kb_ids, types, item_kinds, date_from, k, cursor)`, `get_transcript(video_key, start_s, end_s)`, `read_page(kb_id, path)`. v1'de bunlara `ingest_channel` eklenir. Playbook, resource ve P3'ün görsel tool'ları `search(types=…)` ve `read_page` ile karşılanıyor. |
| Per-KB persona plugin, marketplace, subagent | P4 §5 | v1.x | Tek bir generic `kb-expert` skill'i + MCP `instructions` yeterli. Persona = `kb_id` + overview. |
| Groq, HikerAPI, Business Discovery, Instaloader, proxy, Modal, CF Cron | P2, P4 | Kes | Her vendor bir secret, bir adapter ve bir kırılma noktası daha demek. |
| Lokal stdio MCP | — | Kes | claude.ai girişli CLI ve Desktop connector'ı zaten görüyor (P4 §2). |

### 3.2 Remote MCP ilk gün gerekli mi?

**v0.1 için gerekli değil, v0.2 için gerekli.** Git'e klonlanmış KB + skill + `smk search` ikilisi, *bu repo'da* çalışan cloud ve CLI oturumlarına yetiyor. Ancak kullanıcının agent'ları **başka repo'larda** açılan cloud oturumlarında çalışacak. Orada KB'ye ulaşmanın iki yolu var: her seferinde `add_repo` yapmak ya da connector kullanmak. Çok repo'lu oturumda `.mcp.json` ve hook'lar yüklenmiyor (P4 §2 P7). claude.ai ve Desktop için de başka bir yol yok. Bu nedenle remote MCP v0.2'nin çekirdeği olmalı. En belirsiz entegrasyon bu olduğu için auth spike'ı ise **hafta 1'de, paralel olarak** yapılmalı.

### 3.3 İnce dikey dilim ve yol haritası

**v0.1: tek kanal, 10 video, atıflı cevap (~3–4 gün)**
- `smk ingest <yt-url> --limit 10` komutu cloud oturumunda çalışır. Sıra şöyle:
  - flat-playlist (`lang=orig`) + Data API ile `published_at`
  - `web_embedded` ile altyazı
  - olmazsa tek vendor
  - o da olmazsa Gemini URL ile transkript
  - açıklamadaki linkler regex ile toplanır
- Altyazı cue'ları cümle bazlı, ≤10 sn'lik segmentlere birleştirilir ve `s####` ID'leri atanır.
- Sonnet 5.5 ile extraction yapılır (P6 şemasının alt kümesi). Alıntılar kodla doğrulanır, timestamp kodla türetilir.
- `kb-data`'ya yazılanlar: budanmış raw, `items.jsonl`, `videos/*.md`, `overview.md`, `index.md`.
- `kb.sqlite` FTS5 (TR katlamalı) + `smk search` + `.claude/skills/kb-expert/SKILL.md`.
- 15 soruluk eval + atıfların kodla kontrolü (URL, `t` aralığı, alıntının segmentte geçip geçmediği).
- Paralel spike: Worker + GitHub OAuth + claude.ai connector'da bir "ping" tool'u.
- **Çıkış kriteri:** 15 sorunun en az 12'si doğru ve geçerli atıflı.

**v0.2: her yüzeyden erişim + görsel + Instagram (~1 hafta)**
- Remote MCP: `createMcpHandler` + `workers-oauth-provider` + GitHub allowlist, Workers Free üzerinde, D1 FTS5 ile. 5 tool, claude.ai connector olarak eklenir; böylece dört yüzeyin hepsinden erişilir.
- Görsel heavy geçiş: yalnızca işaretli videolarda; `v####` segmentleri üretilir, repo'lar GitHub REST ile doğrulanır.
- Instagram: Apify → mp4 → Gemini 3.5 FL'ye inline tek çağrı (transkript + ekran) → Haiku ile extraction.
- `costs.jsonl`, `--dry-run`, `--max-usd`.

**v1: MVP tamam (~1–2 hafta)**
- `ingest_channel` tool'u. Smoke test geçtiyse GH Actions `workflow_dispatch` üzerinden tetiklenir, geçmediyse cloud oturumu kullanılır. Gece artımlı RSS taraması.
- Resume'lu durum makinesi; "all" seçeneği için parçalı run'lar.
- 20 soruluk birleşik eval CI'da koşar. Haiku/Sonnet A/B testi ve vektör kararı burada verilir.
- `smk remove` komutu.

**v1.x:** vektör + rerank (eval gerektirirse), Batch/Flex, wiki derleyicisi + registry + evolution, re-check/keyframe yolu, per-KB plugin, kanallar arası `_topics`.

---

## 4) Kimsenin kapsamadığı eksikler

- **Secrets ve ortam ayrımı.** Gereken anahtarlar: YouTube Data API, Gemini, Anthropic, transkript vendor'ı, Apify, GitHub PAT, Cloudflare API token, OAuth client secret, cookie key. Kural: ingestion **ayrı bir cloud environment'ta** çalışmalı. Bu environment'ta network secrets ve vendor domain'lerini içeren Custom allowlist tanımlı olmalı: `generativelanguage.googleapis.com`, `api.anthropic.com`, `www.googleapis.com`, vendor'lar. KB'yle sohbet eden oturumların environment'ında **hiçbir anahtar olmamalı**; aksi hâlde transkript içine gömülü bir injection, Bash erişimli oturumdan anahtar sızdırabilir. Lokalde `.env` + `gitleaks` pre-commit kullanılmalı.
- **Hata, retry ve resume.**
  - Video başına durum makinesi `manifest.json`'da tutulur: `discovered → transcript → visual → extracted → verified → published`, ayrıca `failed:<aşama>:<neden>` ve `skipped:<neden>` durumları.
  - Her aşama idempotent olmalı ve çıktısını cache'lemeli; yeniden çalıştırma yalnızca eksik aşamaları koşar.
  - 429 hatalarında backoff + jitter uygulanır; 3 denemeden sonra iş dead-letter'a düşer.
  - Her 10 videoda bir ara commit atılır, böylece oturum ya da runner ölürse iş kaybolmaz.
  - Kanal başına kilit konur (GHA'da `concurrency`).
- **Gözlemlenebilirlik ve maliyet.** Her API çağrısı `costs.jsonl`'a yazılmalı: vendor, model, token, $, `video_key`. Run sonunda bir özet çıkmalı: video sayısı, `transcript_source` dağılımı, doğrulama oranı, harcanan $. P1 §3.2'de `ingest_runs.cost_usd` alanı tanımlı ama onu dolduran kimse yok. Aylık eşik aşılınca run durmalı.
- **Sohbetten tetikleme UX'i.** Örnek akış: "@x'ten 100 video ekle" → agent önce `estimate` sonucunu gösterir (video sayısı, toplam süre, $, ne kadar süreceği) → kullanıcı onaylar → `ingest_channel` çağrılır → `get_ingest_status` ile takip edilir. "count" ifadesinin anlamı tanımsız (§7).
- **Silme ve unutma.** `smk remove --kb x [--video y]` komutu kb-data'dan ve D1'den siler, indeksi yeniden kurar. Kayıt git geçmişinde kalır; gerekirse `git filter-repo`. Kaynakta silinmiş ya da private'a çekilmiş videolar artımlı taramada `source_status: removed` olarak işaretlenir.
- **Test stratejisi.**
  - (a) Kayıtlı fixture'larla ağsız unit testler: json3, Apify ve LLM yanıtları üzerinden segment birleştirme, `İ/ı` katlama, alıntı doğrulama, link üretimi.
  - (b) Python ↔ TS arasında JSON Schema kontrat testleri.
  - (c) MCP Inspector + snapshot testleri.
  - (d) **Haftalık canary:** bilinen 3 videoda altyazı edinimi test edilir, yt-dlp kırılması kullanıcı fark etmeden yakalanır.
  - (e) Eval regresyonu.
- **Repo düzeni ve dil.** Pipeline Python'da yazılmalı (yt-dlp, rapidfuzz, google-genai, anthropic hep Python-native). Worker TS'te yazılmalı (Cloudflare MCP SDK'sı TS-first). İki dil arasındaki kaymayı önlemek için:
  - **Worker yalnızca okuma yapar.**
  - Şema `schema/*.json`'da tek kaynaktır; pydantic ve TS tipleri buradan codegen ile üretilir.
  - Arama SQL'i tek bir dosyadadır (`sql/search.sql`) ve hem `smk search` hem Worker onu kullanır.

  ```
  social-media-scraper/   pipeline/ (uv, Py 3.13, smk CLI) · server/ (wrangler, TS) · schema/ · sql/ · skills/kb-expert/ · evals/ · .github/workflows/
  kb-data/ (ayrı private repo)   kb/<kb_id>/… · raw/<kb_id>/<video_key>/…
  ```
- **Segment granülerliği.** P2 T8'de tek bir videoda 1077 json3 event'i ve "rolling" tekrarlar görüldü. Her cue'ya `s####` verilirse prompt'a binlerce ID eklenir. P1 ve P6 birleştirilmiş segment varsayıyor, ama algoritmayı kimse tanımlamamış: dedupe + cümle sınırı + ≤10 sn.
- **IG için derin link yok.** Atıf "(0:12)" metniyle veriliyor. Kullanıcı bunu kabul ediyor mu, sorulmadı.

---

## 5) Kritik iddiaların doğrulanması (2026-10-20)

| İddia | Sonuç | Not | Kaynak |
|---|---|---|---|
| Gemini YouTube URL: preview, ücretsiz, yalnızca public video, free tier'da 8 saat/gün, paid'de süre limiti yok, 2.5 ve sonrasında istek başına 10 video (P2/P3/P5) | **Doğrulandı** | Sayfada ayrıca "Pricing and rate limits are likely to change" yazıyor. Aynı sayfa 66/258 tok/kare + 32 tok/sn veriyor; bu P3'ün 70/280/25 değerleriyle çelişiyor. | https://ai.google.dev/gemini-api/docs/video-understanding |
| Gemini fiyat ve ömür: 3.5 FL $0.30/$0.30/$2.50; 3.1 FL $0.25/$0.50/$1.50 ve 7 Mayıs 2027'de kapanış; 3.6/3.8 Flash $0.75/$3.75, 1 Ocak 2027'den itibaren $1.50/$7.50; 2.5 yalnızca önceden kullananlara açık | **Doğrulandı** | P5'in 3.1 FL tercihi bu nedenle zayıflıyor. | https://ai.google.dev/gemini-api/docs/pricing · https://ai.google.dev/gemini-api/docs/deprecations |
| Claude Haiku 5.5 $0.10/$0.50 (≤100K prompt), >100K'da $0.50/$2.50; Sonnet 5.5 $2/$10; Opus 5.5 $4/$20; Batch −%50; Sonnet/Opus 5.5'te cache okuma 0.05× | **Doğrulandı** | Haiku 5.5, 1M bağlamı standart fiyattan sunmayan tek istisna. Per-video çağrılar <100K tutulmalı (P5 haklı). | https://platform.claude.com/docs/en/about-claude/pricing |
| claude.ai custom connector: Free/Pro/Max/Team/Enterprise'da var, Free'de 1 connector; OAuth DCR ve CIMD varsayılan; `static_headers` beta ve sınırlı; CIMD iki metadata alanı yoksa DCR'a düşüyor; zaman aşımları 10 sn / 30 sn; callback `claude.ai/api/mcp/auth_callback` + port-bağımsız loopback; aynı altyapı tüm yüzeylerde | **Doğrulandı** | P4 §2–3 doğru. Ek olarak "Use your own OAuth client" seçeneği de var. | https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp · https://claude.com/docs/connectors/building/authentication |
| Cloudflare: `McpAgent` deprecated ve feature-frozen (agents v0.20.0); `createMcpHandler(factory)` stateless çalışıyor; Vectorize Free planda var (30M sorgulanan / 5M saklanan boyut) | **Doğrulandı** | P4'ün "Vectorize free'de belirsiz" notu çözüldü. Cloudflare 2026-07-28'i hâlâ "release candidate" diye anıyor. `workers-oauth-provider` CIMD desteği **doğrulanamadı**. | https://developers.cloudflare.com/changelog/post/2026-07-27-agents-sdk-v0.20.0-mcp-sdk-v2/ · https://developers.cloudflare.com/vectorize/platform/pricing/ |
| Apify reel scraper: Free planda $2.60/1k, Starter'da $2.30/1k; mp4 URL + caption + timestamp + süre dönüyor; transkript add-on'u dakika başı ücretli (fiyat yazmıyor); yalnızca public reel | **Doğrulandı** | — | https://apify.com/apify/instagram-reel-scraper |
| GH Actions runner'ından YouTube altyazısı alınabilir (P4'ün örtük varsayımı) | **Belirsiz** | Doğrudan kanıt yok. youtube-transcript-api README'si AWS/GCP/**Azure** IP'lerinin bloklandığını söylüyor; GitHub-hosted runner'lar Azure'da. Smoke test şart. | https://github.com/jdepoix/youtube-transcript-api · https://github.com/jdepoix/youtube-transcript-api/issues/593 |

---

## 6) Önerilen nihai mimari karar listesi

1. **Monorepo, iki dil:** Python pipeline + salt okunur TS Worker; şema ve SQL tek kaynakta. Gerekçe: her dil kendi ekosisteminde güçlü, kayma noktası daraltılmış oluyor.
2. **SoT, private `kb-data` git repo'su** (JSONL + MD, embedding yok). Gerekçe: tüm indeksler yeniden üretilebilir, vendor değiştirmek ucuz.
3. **Kimlik:** `channel_id` PK, `kb_id` slug; `yt-<ID>` / `ig-<shortcode>`; ref'ler `#s/#v/#i`. Gerekçe: dosya sistemine güvenli ve handle değişimine dayanıklı.
4. **Keşif:** yt-dlp flat-playlist + Data API + RSS. Gerekçe: hepsi buluttan test edildi ve ücretsiz.
5. **YT transkript:** `web_embedded` → tek vendor adapter → Gemini URL, `transcript_source` güven etiketiyle. Gerekçe: ücretsiz yol önce, kırıldığında otomatik yedek.
6. **IG:** Apify → mp4 → Gemini 3.5 FL'de tek çağrı. Gerekçe: Groq'a gerek kalmıyor, tek ek vendor yeterli.
7. **Görsel:** Gemini 3.5 FL, YouTube URL, `high` @0.5 fps, yalnızca heuristik router'ın işaretlediği videolarda. Gerekçe: kod okumayı kanıtlanmış tek ayar bu, maliyet de kontrol altında.
8. **Extraction:** uzun videoda Sonnet 5.5, reel'de Haiku 5.5; P6 şeması + kodla alıntı doğrulama. Gerekçe: çekirdek değer bu adımda üretiliyor; maliyeti 100 videoda ~$4.
9. **MVP wiki:** video notları + overview + index + `resources.md`. Gerekçe: derleyici ve registry'nin getirisi henüz kanıtlanmadı.
10. **Arama:** D1/SQLite FTS5 + TR katlama, aynı SQL. Vektör yalnızca eval kapısı geçilemezse eklenir.
11. **Serving:** tek remote MCP; Workers Free, `createMcpHandler`, `workers-oauth-provider` + GitHub allowlist (DCR); 5 okuma + 1 yazma tool'u. Gerekçe: tek entegrasyonla dört yüzeyin hepsine ulaşılıyor.
12. **Persona:** tek `kb-expert` skill'i + MCP `instructions`. Gerekçe: belgelenmemiş plugin sync davranışına bağımlı kalmamak.
13. **Ingestion yeri:** v0.x'te ayrı bir ingest environment'ında cloud oturumu; v1'de smoke test geçerse GH Actions + gece RSS taraması. Her iki durumda da dry-run ve `--max-usd`.
14. **Güvenlik:** tool'lar salt okunur; yazma `requiresUserInteraction` arkasında; içerik `untrusted` olarak etiketli; sohbet ortamlarında anahtar yok.
15. **Config:** model adları ve fiyatlar `config.yaml`'da; maliyet gerçek `usage` verisinden. Gerekçe: Ocak ve Mayıs 2027'deki fiyat/ömür değişiklikleri.
16. **Kalite:** kanal başına 20 soruluk birleşik eval (P1 §3.6 + P6 §7) ve kodla atıf kontrolü CI'da.

---

## 7) Kullanıcıya 5 açık soru

1. **Ingestion nerede koşabilir?** Evde sürekli açık bir makine (residential IP) var mı? Varsa ücretsiz altyazı ve YouTube indirme mümkün olur, Gemini URL'nin de yedeği olur. Yoksa her şey tamamen bulutta mı çalışacak?
2. **Bütçe neyi kapsıyor?** $0–10/ay yalnızca API + hosting için mi, yani Claude Pro/Max aboneliği hariç mi? Tek seferlik ~$47'lık süresiz kredi paketi (ScrapeCreators) kabul edilebilir mi, yoksa aylık ≤$5'lık bir abonelik (Supadata) mi tercih edilir?
3. **"count" ne demek?** Son N video mu, en çok izlenen N video mu? Shorts ve canlı yayınlar dahil mi? Video süresine bir üst sınır konulsun mu (ör. 90 dk)?
4. **Pilot kanallar hangileri?** İlk iki kanal seçilmeli: biri TR, biri EN, en az biri kod ağırlıklı. Ekrandaki kod/repo çıkarımı bu kanallarda ilk günden gerekli mi?
5. **Wiki ve notlar hangi dilde olsun?** İngilizce mi (P6'ya göre kanallar arası birleştirme daha kolay) yoksa Türkçe mi? Notları Obsidian'da kendin de okuyacak mısın, yoksa yalnızca agent'lar mı kullanacak?
