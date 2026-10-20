# Kanal Knowledge Base Sistemi — Tasarım Spec'i

**Tarih:** 2026-10-20 · **Durum:** Taslak, kullanıcı incelemesi bekliyor
**Dayanak:** `docs/research/00-synthesis.md` (Seçenek A) ve 01–07 raporları.

**Kapsam notu:** Bu spec sistemin tamamını tarif eder, ama yalnızca **v0.1 ayrıntılı** tasarlanmıştır. İlk uygulama planı da sadece v0.1'i kapsar. v0.2 ve v1 kendi spec ekleri ve planlarıyla gelecek.

---

## 1. Amaç

Kullanıcı bir YouTube veya Instagram kanal linki ve bir sayı verir. Sistem:
1. kanalın videolarını keşfeder,
2. transkripti (ve gerektiğinde ekran içeriğini) çıkarır,
3. LLM ile doğrulanabilir bilgi parçalarına (knowledge item) dönüştürür,
4. kanal başına bir knowledge base (KB) yazar,
5. KB'yi kullanıcının agent'larına arama ve okuma araçlarıyla sunar.

Agent her kanal kaynaklı iddiayı **video + zaman damgası** linkiyle kaynaklandırır.

**Başarı (v0.1 çıkış kriteri):** Pilot kanal için hazırlanan 15 sorunun **en az 12'si** doğru cevaplanır, her cevap geçerli atıf taşır. Geçerli atıf üç koşulu sağlar:
- URL doğrudur,
- `t` değeri videonun süresi içindedir,
- alıntı ilgili segmentte geçer.

### Kullanıcı kararları (onaylı)

| Konu | Karar |
|---|---|
| Mimari | Seçenek A: yalın, aşamalı |
| Platform | YouTube + Instagram. Instagram v0.2'de |
| Erişim | Claude Code cloud, lokal CLI, Desktop, claude.ai. Dördü birden v0.2'de remote MCP ile |
| Çalışma yeri | Tamamen bulut. Ev makinesi yok |
| "Sayı" | `--order latest` (varsayılan) ve `--order popular` seçilebilir |
| KB dili | İngilizce. Birebir alıntılar orijinal dilde kalır, çünkü doğrulama onlarla yapılır. Agent kullanıcıya kendi dilinde cevap verir |
| Bütçe | Ucuz ücretli API'ler kabul. Hedef 100 video için birkaç dolar, ayda $0–10 |

### Hedef dışı
- Bilgiyi yeniden yayınlamak veya KB'yi herkese açmak. KB **özeldir**.
- Kullanıcının kendi Instagram hesabıyla login'li scraping.
- v0.1'de: vektör arama, rerank, Batch API, wiki derleyicisi (topic, framework ve playbook sayfaları), kanal başına plugin.

---

## 2. Aşamalar

| Aşama | Kapsam | Çıkış kriteri |
|---|---|---|
| **v0.1** | 1 YouTube kanalı, ≤10 video: keşif → altyazı → extraction → doğrulama → `kb-data` → SQLite FTS5 → `smk search` → `kb-expert` skill → eval. **Opsiyonel paralel spike:** remote MCP OAuth (bkz. §11) | §1'deki 12/15 eşiği |
| **v0.2** | Remote MCP (Cloudflare Worker + D1 FTS5, 5 okuma tool'u, claude.ai connector) · görsel geçiş (Gemini 3.5 Flash-Lite, `high`) · Instagram (Apify → Gemini) · maliyet raporu | Aynı soruya dört yüzeyden atıflı cevap |
| **v1** | `ingest_channel` tool'u (sohbetten tetikleme + tahmin ve onay) · resume'lu toplu çalıştırma ("all") · RSS ile artımlı tarama · CI'da eval · `smk remove` | Sohbetten "@x'ten 100 video ekle" komutu uçtan uca çalışır |
| **v1.x** | Eval gerektirirse vektör + rerank · Batch · wiki derleyicisi · kanallar arası `_topics` · persona plugin'leri | — |

---

## 3. Mimari (v0.1)

```
smk ingest <kanal-url> --limit N --order latest|popular
   │
   ├─ discovery      kanal → video listesi (yt-dlp flat-playlist; varsa YouTube Data API ile tarih)
   ├─ transcripts    sağlayıcı zinciri: web_embedded → vendor (Supadata) → Gemini URL (anahtar varsa)
   ├─ segmenter      altyazı cue'ları → cümle sınırlı, ≤10 sn segmentler (s0000…)
   ├─ extractor      Claude Sonnet 5.5, structured output → items/resources (JSON)
   ├─ verifier       alıntı doğrulama (TR duyarlı normalize + rapidfuzz), ref düzeltme, zaman türetme
   ├─ writer         kb-data: raw/ + kb/<kb_id>/{videos/*.md, overview.md, index.md, resources.md, _data/…}
   └─ indexer        kb.sqlite: tablolar + FTS5 (TR katlama)

smk search "<sorgu>" --kb <kb_id|*> --type item,segment --k 8 [--json]
skills/kb-expert/SKILL.md  → agent'a overview oku → ara → alıntıla → atıfla akışını öğretir
```

### 3.1 Birimler ve arayüzler

Her birim tek bir işten sorumludur. Birimler yalnızca `schema/` altındaki veri tipleriyle konuşur ve ağ olmadan, fixture'larla test edilebilir.

| Birim (modül) | Girdi → Çıktı | Bağımlılık |
|---|---|---|
| `config` | `config.yaml` + env → `Settings` | — |
| `ids` | platform, ham ID → `kb_id`, `video_key`, ref; citation URL üretimi | — |
| `discovery` | kanal URL'si, `limit`, `order`, filtreler → `list[VideoStub]` | yt-dlp, (opsiyonel) YouTube Data API |
| `transcripts` | `VideoStub` → `Transcript` (cue'lar, dil, `source`) | `TranscriptProvider` protokolü: `WebEmbeddedProvider`, `SupadataProvider`, `GeminiProvider` |
| `segmenter` | `Transcript` → `list[Segment]` | — |
| `extractor` | `VideoMeta` + `list[Segment]` + açıklama → `Extraction` (ham LLM JSON) | Anthropic SDK |
| `verifier` | `Extraction` + `list[Segment]` → `VerifiedExtraction` (`verified` bayrakları, düzeltilmiş ref'ler, `t_start`) | rapidfuzz |
| `writer` | doğrulanmış veri → `kb-data` dosyaları (idempotent) | Jinja2 şablonları |
| `synth` | kanalın tüm item'ları → `overview.md` (LLM) | Anthropic SDK |
| `indexer` | `kb-data` → `kb.sqlite` (her seferinde sıfırdan, deterministik) | sqlite3 (FTS5) |
| `search` | sorgu + filtreler → `list[SearchHit]` (hazır `citation` ile) | `sql/search.sql` |
| `manifest` | video başına durum makinesi, resume | — |
| `costs` | her API çağrısının kullanımı → `costs.jsonl`, run özeti, bütçe kapısı | `config.yaml` fiyatları |
| `cli` (`smk`) | `ingest`, `estimate`, `list`, `search`, `build-index`, `synth`, `status` | yukarıdakilerin hepsi |

### 3.2 Repo düzeni

```
social-media-scraper/
  pipeline/                # Python 3.13, uv, paket adı: smk
    pyproject.toml
    src/smk/{cli,config,ids,discovery,transcripts/,segmenter,extractor,verifier,writer,synth,indexer,search,manifest,costs}.py
    src/smk/prompts/extract_v1.md, overview_v1.md
    src/smk/templates/video.md.j2, index.md.j2, resources.md.j2
    tests/  tests/fixtures/
  schema/                  # JSON Schema: tek kaynak (Python modelleri bunlarla uyumlu tutulur)
    extraction.schema.json  item.schema.json  video.schema.json
  sql/schema.sql  sql/search.sql
  skills/kb-expert/SKILL.md   (+ .claude/skills/kb-expert → aynı içerik)
  evals/<kb_id>/questions.yaml  evals/run_eval.py
  config.yaml
  server/                  # v0.2 (TypeScript Worker). v0.1'de yalnızca opsiyonel spike
  docs/
```

---

## 4. Veri modeli

### 4.1 Kimlikler (tek kural)

| Kavram | Biçim | Örnek |
|---|---|---|
| Kanal PK | platformun değişmez ID'si | `UCsBjURrPoezykLs9EqgamOA` |
| `kb_id` (slug ve alias) | `yt-<handle-slug>` / `ig-<username>` | `yt-barisozcan` |
| `video_key` | `yt-<videoId>` / `ig-<shortcode>` | `yt-nnNkf7ttuGc` |
| Segment ref | `<video_key>#s0042` (konuşma), `#v0007` (ekran, v0.2) | `yt-nnNkf7ttuGc#s0042` |
| Item ref | `<video_key>#i03` | |
| Citation URL | `https://www.youtube.com/watch?v=<id>&t=<start_s>s` | Yalnızca `ids.citation_url()` üretir |

Dosya adlarında `:` kullanılmaz. Handle değişirse `kb_id` alias olarak güncellenir; PK değişmez.

### 4.2 `kb-data` düzeni

`kb-data` özel bir git repo'sudur. Yolu `KB_DATA_DIR` env değişkeniyle verilir, varsayılan `./kb-data` (bu repo'da gitignore'lu).

```
kb-data/
  kb/_index.md                         # tüm KB'ler: kanal başına 1 satır (kod üretir)
  kb/<kb_id>/
    index.md                           # video listesi + 1 satır özet (kod)
    overview.md                        # tez, imza fikirler, "hayır dedikleri", kapsam (LLM, Sonnet)
    resources.md                       # araç/repo/kitap tablosu, stance + kanıt (kod)
    videos/<video_key>.md              # frontmatter + TL;DR + item'lar + atıflar (şablon)
    log.md                             # append-only run günlüğü
    _data/channel.json  manifest.json  items.jsonl  resources.jsonl  costs.jsonl
    _data/extracts/<video_key>.json    # ham LLM çıktısı + extractor sürümü
  raw/<kb_id>/<video_key>/
    meta.json                          # budanmış metadata (açıklama, chapters, linkler, süre, tarih)
    transcript.json                    # cue'lar + source + lang
    segments.json                      # s#### segmentleri
```

Embedding'ler ve `kb.sqlite` git'e **girmez**; ikisi de türev veridir ve yeniden üretilir.

### 4.3 Ontoloji (06 §2; v0.1 alt kümesi)

- **KnowledgeItem:**
  - `kind ∈ {tactic, framework, playbook, concept, claim, example}`
  - `title_en`, `statement_en`, `polarity ∈ {do, dont, neutral}`, `speaker ∈ {host, guest, unknown}` + `speaker_name`
  - `topics[]` (serbest kebab-case slug; v0.1'de registry yok)
  - `specificity ∈ {1,2,3}`, `platform_sensitive`
  - `steps[]` (her adımda `ref`), `numbers[]`
  - `evidence[]` (1–3 adet; `ref`, `source ∈ {speech, description}`, `quote`: birebir, orijinal dilde)
  - `resources[]`
- **Resource:** `kind ∈ {tool, repo, website, book, course, person, product}`, `name`, `url`, `stance ∈ {recommended, used_on_screen, mentioned, warned_against, sponsored}`, `why_en`, `evidence[]`.
- **Video düzeyi:** `content_type`, `knowledge_density`, `summary_en` (≤60 kelime), `takeaway_en`, `topics[]`, `notable_quotes[]` (≤3).

Extraction şeması `schema/extraction.schema.json`'dadır. Claude structured outputs kısıtları nedeniyle tüm alanlar `required`, boş değer `""` ya da `[]` olarak verilir. Sayısal sınırlar şemada değil, kodda doğrulanır.

### 4.4 SQLite şeması (`sql/schema.sql`)

| Tablo | Alanlar |
|---|---|
| `channels` | `channel_id` PK, `kb_id` UNIQUE, `platform`, `handle`, `title`, `url`, `lang`, `last_synced_at` |
| `videos` | `video_key` PK, `kb_id`, `url`, `title`, `published_at`, `duration_s`, `lang`, `format`, `transcript_source`, `status`, `pipeline_version` |
| `segments` | `ref` PK, `video_key`, `start_s`, `end_s`, `text` |
| `items` | `ref` PK, `kb_id`, `video_key`, `kind`, `title_en`, `statement_en`, `polarity`, `speaker`, `specificity`, `platform_sensitive`, `topics` (JSON), `steps` (JSON), `evidence` (JSON), `verified`, `t_start`, `citation_url` |
| `resources` | `resource_id` PK, `kb_id`, `kind`, `name`, `url`, `stance`, `evidence` (JSON) |
| `chunks` | `chunk_id` PK, `kb_id`, `video_key`, `type ∈ {segment, item, page}`, `item_kind`, `start_s`, `end_s`, `context_header`, `text`, `text_norm`, `title`, `citation_url`, `published_at` |
| `chunks_fts` | FTS5, external content = `chunks`; kolonlar `title`, `context_header`, `text_norm`; tokenizer `unicode61 remove_diacritics 2`; `prefix='2 3'` |

**Chunk'lar:**
- **Transkript chunk'ı:** ardışık segmentlerden ~300 token (min 120, maks 450). Chapter sınırını geçmez. Overlap 1 cümle.
- **Item chunk'ı:** `statement_en` + `quote` + adımlar.
- **Sayfa chunk'ı:** `overview.md` H2 bölümleri.
- **`context_header`:** `Channel: … | Video: … | Chapter: … (mm:ss–mm:ss) | Date: YYYY-MM`.

**Türkçe katlama (`text_norm`):** NFKC → Türkçe'ye duyarlı küçük harf (`İ→i`, `I→ı`) → `ı→i` → aksan katlama → noktalama silme. Sorgu da aynı fonksiyondan geçer. Kelimeler `kök*` prefix sorgusuna çevrilir.

### 4.5 Arama sonucu biçimi

`smk search --json` ve v0.2'deki MCP `search` aynı şekli döndürür:

```json
{"query": "...", "kb_ids": ["yt-x"], "results": [
  {"type": "item", "kb_id": "yt-x", "video_key": "yt-ID", "video_title": "...", "published_at": "2026-03-14",
   "item_kind": "tactic", "start_s": 754, "end_s": 812, "snippet": "≤250 token",
   "citation": "[Video başlığı @ 12:34](https://www.youtube.com/watch?v=ID&t=754s)", "ref": "yt-ID#i03"}],
 "next_cursor": null, "content_trust": "untrusted-third-party-transcript"}
```

Sıralama: FTS5 `bm25` (ağırlıklar title 2.0, context 0.5, text 1.0). Aynı videodan en fazla 2 sonuç. Varsayılan `k=8`.

---

## 5. Pipeline ayrıntıları (v0.1)

### 5.1 Keşif
- `yt-dlp --flat-playlist -J --extractor-args "youtube:lang=<orig>" <kanal>/videos`
  - Varsayılan olarak yalnız `/videos` sekmesi alınır. `--include-shorts` ve `--include-live` ile `/shorts` ve `/streams` eklenir.
  - `youtube:lang` olmadan başlıklar İngilizce'ye çevrilmiş gelir (02 T3/T4).
- `--order latest`: sekmenin doğal sırası, ilk N video. `--order popular`: tüm liste çekilir (856 ID ≈ 11 sn), `view_count`'a göre azalan sıralanır, ilk N alınır.
- `--max-duration 90m` (varsayılan) ile daha uzun videolar `skipped:too_long` olur.
- `YOUTUBE_API_KEY` varsa `videos.list` ile kesin `published_at` alınır. Yoksa `approximate_date` kullanılır ve `published_at_approx=true` işaretlenir.
- Video metadata'sı (açıklama, chapters, tags) `web_embedded` altyazı çağrısıyla aynı `info.json`'dan alınır ve budanır (≤50 KB).

### 5.2 Transkript zinciri

Sağlayıcılar sırayla denenir; ilk başarılı olan kazanır. **Çeviri track'i asla alınmaz.**

| Sıra | Sağlayıcı | Ne zaman | `source` etiketi | Güven |
|---|---|---|---|---|
| 1 | yt-dlp `player_client=web_embedded`, `json3`. Öncelik: manuel orijinal dil > `<lang>-orig` otomatik | Her zaman | `yt_manual` / `yt_auto` | 1 / 2 |
| 2 | **Supadata** transcript API (`SUPADATA_API_KEY`; Free 100 kredi/ay) | 1 başarısızsa | `vendor` | 3 |
| 3 | Gemini 3.5 Flash-Lite + YouTube URL, "verbatim transcript with timestamps" | 1–2 başarısızsa ve `GEMINI_API_KEY` varsa | `gemini` | 4 (en düşük) |

- Hiçbiri başarılı olmazsa video `failed:transcript:<neden>` olur.
- Her run'ın sonunda `source` dağılımı raporlanır. Kırılan bir yol böylece görünür hâle gelir.
- Vendor adapter'ı `TranscriptProvider` protokolüne uyar. ScrapeCreators v0.2'de ikinci adapter olarak eklenir.

### 5.3 Segmentasyon
- **json3 normalize:** Otomatik altyazıdaki "rolling" tekrarlar temizlenir; önceki event'in sonekiyle örtüşen önek atılır.
- **Birleştirme:** Cue'lar cümle sonu (`.?!…`) ya da ≥1.5 sn duraklamada kesilir. Bir segment ≤10 sn ve ≤40 kelime olur.
- **ID:** Segmentlere sırayla `s0000…` verilir. `start_s` ve `end_s` cue zamanlarından gelir.
- **Kanıt (02 T8):** Tek bir videoda 1077 json3 event'i vardı. Birleştirme olmazsa prompt binlerce ID ile şişer.

### 5.4 Extraction
- **Model:** `claude-sonnet-5-5`. Reel ve ≤90 sn içerik için `claude-haiku-5-5` (v0.2). Model adları `config.yaml`'dadır.
- **Prompt (`prompts/extract_v1.md`):** 06 §3'teki taslak. Girdi XML bloklarında, veri üstte, talimat sonda:
  - `<video_meta>`
  - `<transcript>` (`[s0042 03:12] metin` satırları)
  - `<description>`
- **Kurallar:**
  - Kota yok; "hepsi ve sadece" kuralı geçerli.
  - Alıntı birebir ve orijinal dilde, ≤40 kelime.
  - `statement_en` İngilizce.
  - Ref uydurmak yasak.
- **Structured output:** `schema/extraction.schema.json`. Çağrı **tool'suz** yapılır (prompt injection yüzeyini daraltmak için).
- **Uzun videolar:** Transkript ~60k token'ı aşarsa chapter'lara göre ~40 dk'lık, 2 dk örtüşmeli pencerelere bölünür. Pencere çıktıları `statement_en` normalizasyonuyla deterministik olarak tekilleştirilir. v0.1'de varsayılan `max_duration` zaten 90 dk.
- **Önbellek:** `_data/extracts/<video_key>.json` dosyası `extractor_version = <prompt_hash>/<model>` alanını taşır. Bu değer aynıysa çağrı tekrar yapılmaz.

### 5.5 Doğrulama (deterministik)
1. Ref girdide var mı? Yoksa yakın segment aranır.
2. Alıntı ve segment metni aynı şekilde normalize edilir (§4.4 katlama). Arama sırası:
   - `[ref−1, ref+2]` penceresinde exact substring,
   - yoksa `rapidfuzz.partial_ratio ≥ 90`,
   - yoksa tüm transkript.

   Alıntı başka bir yerde bulunursa ref **düzeltilir**. Hiçbir yerde bulunamazsa `verified=false`.
3. Hiç doğrulanmış kanıtı olmayan item düşürülür ve `log.md`'ye yazılır.
4. `t_start = ilk doğrulanmış segmentin start_s − 1` (en az 0). Citation `ids.citation_url()` ile üretilir.
5. Videonun `verified_ratio` değeri < 0.8 ise video `review: pending` işaretlenir. v0.1'de yeniden çalıştırma yok, yalnız rapor.
6. `source=gemini` olan transkriptlerde doğrulama döngüseldir (07 §2.3). Bu item'lar `evidence_trust=model` ile işaretlenir ve atıfta "≈" gösterilir.

### 5.6 Yazım ve sentez
- `videos/<video_key>.md`: Jinja2 şablonundan üretilir; LLM çağrısı yok.
  - Frontmatter: id, url, title, published, duration, lang, `transcript_source`, topics, items, `verified_ratio`, extractor.
  - TL;DR ve item listesi; her item için `[▶ mm:ss](citation)` ve birebir alıntı.
- `resources.md` ve `index.md`: koddan üretilir.
- `overview.md`: tüm item'lardan **tek bir Sonnet çağrısıyla** yazılır.
  - Bölümler: Thesis, Signature ideas, Says NO to, Assumed context, Coverage, Blind spots, Freshness.
  - Her madde item ref'leri taşır (`<!-- yt-ID#i03 -->`).
  - Yeniden üretim: yeni videolar toplamın %10'unu aştığında ya da `smk synth --force` ile.
- `log.md`'ye run satırı eklenir.
- `kb-data` bir git repo'su ise ve `--commit` verildiyse her 10 videoda bir commit atılır.

### 5.7 Durum makinesi, hata ve resume
- **Durumlar** (`manifest.json`, video başına): `discovered → transcribed → extracted → verified → written → indexed`. Ayrıca `failed:<aşama>:<neden>` ve `skipped:<neden>`.
- **Resume:** Her aşama idempotenttir ve çıktısını diske yazar. Yeniden çalıştırma yalnızca eksik aşamaları koşar.
- **Idempotency anahtarı:** `(video_key, content_hash, pipeline_version)`.
- **Retry:** 429 ve 5xx hatalarında üstel backoff + jitter, en fazla 3 deneme. Sonra `failed`, run devam eder.
- **Kilit:** `kb-data/kb/<kb_id>/.lock` ile kanal başına aynı anda tek ingest.

### 5.8 Maliyet ve bütçe kapısı
- Her API çağrısı `costs.jsonl`'a yazılır: zaman, `video_key`, vendor, model, girdi/çıktı/cache token, `usd`. `usd`, `config.yaml` fiyatlarıyla gerçek `usage` verisinden hesaplanır.
- `smk estimate <url> --limit N` ve `smk ingest --dry-run`: toplam süre ve tahmini $ (video başı token tahmini × fiyat).
- `--max-usd` (varsayılan 5): kümülatif harcama sınırı aşarsa run temiz biçimde durur (resume edilebilir).
- Run sonu özeti: işlenen / atlanan / başarısız video, `transcript_source` dağılımı, `verified_ratio` ortalaması, toplam $.

---

## 6. Agent erişimi

### 6.1 v0.1: skill + CLI
- `skills/kb-expert/SKILL.md` (06 §6'daki persona talimatına dayanır). Akış:
  1. `smk list` ya da `kb/_index.md` ile hangi KB'lerin var olduğunu öğren.
  2. İlgili KB'nin `overview.md` ve `index.md` dosyalarını oku.
  3. `smk search` ile ara. Kritik terimleri **TR ve EN iki sorguyla** dene.
  4. Alıntı yapmadan önce `videos/<video_key>.md` ya da segmenti oku.
  5. Cevap ver. Her kanal kaynaklı iddia `[Başlık @ mm:ss](url&t=Ns)` atıfı taşır. KB dışı bilgi "General knowledge (not from this channel):" etiketiyle verilir.
- **Çalıştığı yer:** Bu repo ile `kb-data`'nın birlikte bulunduğu Claude Code oturumları (cloud ve CLI).
- **Kısıt:** Başka repo'daki oturumlar ve claude.ai/Desktop **v0.2'yi bekler**.

### 6.2 v0.2: remote MCP (özet; ayrıntılar v0.2 spec ekinde)
- Cloudflare Worker (Free plan): `createMcpHandler` + `@cloudflare/workers-oauth-provider` + GitHub login allowlist (DCR).
- Veri: D1, aynı `sql/schema.sql` ve `sql/search.sql`.
- Tool'lar: `list_kbs`, `get_overview`, `search`, `get_transcript`, `read_page`. v1'de buna `ingest_channel` eklenir (ayrı scope ve kullanıcı onayı ile).
- Kullanıcı connector'ı claude.ai'ye bir kez ekler; dört yüzeyde de görünür.
- Skill, MCP tool'ları varsa onları, yoksa CLI'ı kullanacak şekilde iki yolu da anlatır.

---

## 7. Güvenlik ve gizlilik

- **Veri:** `kb-data` özel repo'dur. Transkriptler kişisel kullanım içindir, yeniden yayınlanmaz.
- **Prompt injection:**
  - Extraction çağrıları tool'suz yapılır.
  - Transkript ve açıklama metni prompt'ta veri bloğu olarak işaretlenir.
  - Arama çıktısı `content_trust: untrusted-third-party-transcript` taşır.
  - Skill, KB içeriğindeki talimatları **asla uygulamamasını** açıkça söyler.
- **Anahtarlar:** Yalnızca environment ayarlarında (network secrets / env) tutulur. Sohbete yapıştırılmaz, repo'ya yazılmaz.
  - Lokalde `.env` (gitignore).
  - v0.2 ile birlikte veri çekme işi (ingest) **ayrı bir cloud environment'ına** taşınır; KB ile sohbet eden ortamlarda anahtar bulunmaz.
  - v0.1'de bu ayrım yoktur; bilinen risk olarak kabul edilir.
- **Env değişkenleri:** `ANTHROPIC_API_KEY`, `YOUTUBE_API_KEY` (opsiyonel), `SUPADATA_API_KEY` (opsiyonel), `GEMINI_API_KEY` (opsiyonel), `KB_DATA_DIR`.

---

## 8. Yapılandırma (`config.yaml`)

```yaml
pipeline_version: 1
models:
  extract_long: claude-sonnet-5-5
  extract_short: claude-haiku-5-5
  synth: claude-sonnet-5-5
  gemini_transcript: gemini-3.5-flash-lite
prices_usd_per_mtok:              # 2026-10-20, docs/research/07 §5; değişince burası güncellenir
  claude-sonnet-5-5: {in: 2.0, out: 10.0, cache_read: 0.10}
  claude-haiku-5-5:  {in: 0.10, out: 0.50, cache_read: 0.01}
  gemini-3.5-flash-lite: {in: 0.30, out: 2.50}
discovery: {order: latest, include_shorts: false, include_live: false, max_duration_min: 90}
transcripts: {providers: [web_embedded, supadata, gemini]}
verify: {fuzzy_threshold: 90, min_verified_ratio: 0.8}
budget: {max_usd_per_run: 5.0}
search: {k: 8, max_per_video: 2, snippet_tokens: 250}
```

---

## 9. Test stratejisi

1. **Unit testler (ağsız, pytest):** Fixture'lar `tests/fixtures/`'ta (gerçek bir json3 örneği, Supadata yanıtı, kayıtlı Sonnet extraction çıktısı). Kapsadığı konular:
   - json3 normalize ve segment birleştirme,
   - Türkçe katlama (`İSTANBUL`→`istanbul`, `ılık`→`ilik`),
   - alıntı doğrulama (exact, fuzzy, ref düzeltme, düşürme),
   - citation URL üretimi,
   - durum makinesi ve resume,
   - bütçe kapısı,
   - `--order popular` sıralaması,
   - FTS sorgu üretimi ve bm25 sıralaması (geçici SQLite üzerinde),
   - şablon çıktısı.
2. **Sözleşme testleri:** Kayıtlı LLM çıktıları `schema/extraction.schema.json`'a karşı doğrulanır.
3. **Ağ testleri** (`@pytest.mark.network`, opt-in):
   - 1 videoda `web_embedded` altyazı,
   - 1 videoda Supadata,
   - flat-playlist listeleme.

   Aynı set ileride haftalık canary olarak kullanılacak.
4. **Eval (`evals/run_eval.py`):** Pilot kanal için 15 soru (`questions.yaml`). Sorular item'lardan LLM ile taslak olarak üretilir, kullanıcı onaylar. Kategoriler:
   - 5 olgusal,
   - 4 how-to,
   - 2 araç/repo,
   - 2 videolar arası sentez,
   - 2 kapsam dışı (doğru davranış: çekimser kalmak).

   Her soru küçük bir tool-use döngüsüyle cevaplanır: Sonnet 5.5, `SKILL.md` system prompt'u ve iki tool (`search` → `smk.search`, `read_page` → KB dosyası okuma), en fazla 6 tur. Bu döngü, gerçek agent deneyimini taklit eder. Skorlama:
   - **Atıflar kodla kontrol edilir:** URL, `t` aralığı ve alıntının segmentte geçip geçmediği.
   - **Doğruluk Sonnet ile yargılanır:** cevap, referans cevaba ve item'lara göre değerlendirilir.

   Çıkış kriteri ≥12/15.

---

## 10. Maliyet beklentisi (v0.1)

10 video × ~15 dk, Sonnet extraction + 1 overview çağrısı ≈ **$0.5–0.8**. Altyazı yolu ücretsiz; Supadata yedeği Free kotada. Prompt iterasyonu için ×2 tampon ile v0.1 pilotu **≤ $2**.

---

## 11. Opsiyonel paralel spike: remote MCP OAuth (v0.1 haftası)

- **Amaç:** v0.2'nin en belirsiz entegrasyonunu erkenden doğrulamak (07 §2.4).
- **Kapsam:** `server/spike/` altında bir Worker. İçerik:
  - `createMcpHandler`,
  - `workers-oauth-provider`,
  - GitHub OAuth App ve `ALLOWED_GITHUB_LOGINS`,
  - tek bir `ping` tool'u.
- **Test:** Kullanıcı Worker'ı claude.ai'ye custom connector olarak ekler. `ping` aracı claude.ai, lokal CLI ve bir cloud oturumundan çağrılır.
- **Ön koşul:** Kullanıcının Cloudflare hesabı (`CLOUDFLARE_API_TOKEN`) ve bir GitHub OAuth App'i. Bunlar yoksa spike v0.2'ye kayar; v0.1 bloklanmaz.

---

## 12. Açık konular

| Konu | Karar zamanı | Varsayılan |
|---|---|---|
| Pilot kanal(lar) | Plan öncesi, kullanıcıdan | 1 kanal. Tercihen eval sorularını kullanıcının kendisinin değerlendirebileceği bir kanal |
| Transkript vendor'ı: Supadata mı, ScrapeCreators mı | Pilot sonrası (`source` dağılımı ve başarı oranına göre) | Supadata (Free kota) |
| Haiku 5.5 extraction yeterli mi | v1'deki 5 videoluk A/B testi | Sonnet 5.5 |
| Vektör arama gerekli mi | Eval'de TR soruların Recall@10 < 0.8 çıkarsa | Hayır |
| Instagram derin link | v0.2'de test | Düz URL + "(m:ss)" |
