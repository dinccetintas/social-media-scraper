# 01 — KB Depolama & Retrieval Mimarisi (Persona 1)

**Persona 1 — Lead AI Engineer / RAG Architect** · Tarih: 2026-10-20 · Kapsam: kanal başına KB'nin nasıl saklanacağı, indeksleneceği ve agent sorgularına nasıl servis edileceği.
Notasyon: fiyatlar ve limitler 2026-10-20 itibarıyla kaynak sayfalardan alındı. **doğrulanamadı** = birincil kaynakta teyit edilemedi. "(ikincil)" = yalnızca 3. taraf kaynakta bulundu.

---

## 1) TL;DR — Net öneri

1. **Ne klasik RAG ne de "LLM wiki" tek başına yeter. Öneri: derlenmiş wiki + hibrit arama indeksi.** Agent wiki ile *gezinir* (`overview` → `topic` → `playbook`), detay ve alıntı için *arar* (BM25 + vektör). Karpathy de `index.md` ile gezinmeyi "~100 kaynak" ölçeğiyle sınırlıyor ve sonrası için hibrit arama (qmd) öneriyor. 1000 videonun düz indeksi ~60k token tutar ve Claude Code'un 25k MCP çıktı limitine sığmaz. Türkçe ekler ve TR↔EN sorgular da grep/FTS'i kırar.
2. **Kanonik kaynak git repo'su (`kb-data`: JSONL + markdown). İndeksler bundan yeniden üretilir.** Serving: **Cloudflare Workers (MCP) + D1 (metadata + FTS5) + Vectorize + R2**. Lokal/CLI için aynı JSONL'den tek bir **SQLite dosyası (FTS5 + sqlite-vec)** üretilir. H4 büyük ölçüde doğru: ayrı bir vektör DB vendor'ı gerekmiyor. 100 video Free tier'a sığar; 1000 video ya da onlarca kanal ≈ **$5–8/ay** tutar.
3. **Chunk = timestamp hizalı pencere, ~300 token (≈60–90 sn).** Chapter sınırını asla geçmez, overlap bir cümle (≤%15) olur. Embedding ve BM25 için ~50–80 tokenlik **deterministik bağlam başlığı** eklenir (Contextual Retrieval'ın ücretsiz versiyonu). Parent = chapter (small-to-big). Birincil indeks birimleri üç tane: P6'nın **knowledge item**'ları, wiki sayfa bölümleri ve ham transcript/görsel segmentleri.
4. **Embedding: `voyage-4` (1024d, $0.06/1M, ilk 200M token ücretsiz)**, yani bu ölçekte fiilen $0. Türkçe için birincil kanıt yok; tek doğrudan kanıt TR-MTEB'de `text-embedding-3-small`'ın retrieval'da 64.99 ile en iyi çıkması. İlk iş: 30 soruluk TR/EN golden set üzerinde voyage-4, OpenAI 3-small, Gemini Embedding 2 ve Workers AI `qwen3-embedding-0.6b` karşılaştırması. Yeniden embed 1000 video için <$1.
5. **Rerank: `voyage rerank-2.5`** (çok dilli, talimat alabiliyor; ≈$0.001/sorgu, 200M ücretsiz). Asıl değeri MCP token bütçesi: 50 aday → 8 sonuç. Anthropic'te rerank, top-20 hatasını %2.9'dan %1.9'a indirdi.
6. **Knowledge graph MVP'de yok.** `items`/`resources`/`mentions` tabloları ve derlenmiş topic/overview sayfaları GraphRAG'in global özet faydasını ucuza verir. Graphiti ancak onlarca kanalda çok adımlı ve zamansal sorgular baskınlaşırsa. Kalite için kanal başına **golden set + Recall@k + LLM-judge** (bölüm 3.6).

---

## 2) Değerlendirilen seçenekler

### 2.1 Klasik RAG gerekli mi? Korpus boyutu ve kırılma noktaları

**Korpus hesabı.** Konuşma hızı ~150 kelime/dk (NCVS, VirtualSpeech üzerinden). 15 dakikalık bir video ≈ 2.250 kelime eder. Tokenizer fertility'si EN için ≈1.24 token/kelime, TR için ≈1.8 (GPT-4o) ile ≈2.5 (Llama-3.1) arası (arXiv 2603.05883). Claude ≥4.7 tokenizer'ı aynı metni ~%30 daha fazla token sayar (P6 notu). Ham transcript boyutları:

| | EN | TR |
|---|---|---|
| 1 video (15 dk) | ~2.8k token | ~4–5.6k token |
| 100 video | ~0.28M | ~0.4–0.56M |
| 1000 video | ~2.8M | ~4–5.6M |
| + görsel notlar, item'lar, wiki (~+%50) | 100 video ≈ 0.45–0.8M; 1000 video ≈ 4.5–8M | |
| Instagram Reel (30–90 sn) | 0.1–0.3k/reel | |

**Yaklaşım karşılaştırması** (tüketici, çok adımlı tool çağrısı yapabilen bir agent):

| Yaklaşım | İyi olduğu yer | Kırıldığı yer |
|---|---|---|
| **(a) LLM wiki** (Karpathy: `raw/` + LLM'in yazdığı wiki + şema; `index.md`, `log.md`; ingest/query/lint) | Sentez soruları ("kanalın ana tezleri?", "cold outreach playbook'u"). Bilgi bir kez derlenir ve birikir; "her soruda sıfırdan keşif" yok. | Index satırı başına ~60 token varsayımıyla 1000 video ≈ 60k token eder; 25k MCP limitine sığmaz. Özetler kayıplıdır, "hangi dakikada X dedi?" sorusunu cevaplayamaz. Derleme maliyeti ve kayma (drift) var. Karpathy de ölçek sınırını "~100 kaynak" olarak koyuyor. |
| **(a′) Agentic grep/FTS** (Claude Code deseni) | Özel isimler (araç, repo, kişi). İndeks bakımı gerektirmez. Kod için Anthropic "agentic search genelde daha iyi" diyor (Cherny, ikincil; resmi ölçüm değil, iç izlenim). | Türkçe ekler ("satışlarını" ≠ "satış"), parafraz, TR↔EN çapraz dil, ASR gürültüsü, çok isabette token israfı. claude.ai/Desktop'ta dosya sistemi yok; grep zaten bir MCP tool'u olarak sunulmak zorunda. |
| **(b) Vektör/hibrit RAG** | Detay ve alıntı soruları, çapraz dil (çok dilli embedding), ölçek. | Global/sentez soruları (top-k parça "kanalın tezi"ni vermez). Chunk bağlam kaybı. |
| **(b′) Tüm korpusu bağlama koymak** | Anthropic: <200k token KB'yi prompt'a koy, RAG gerekmez. | 100 TR video ≈ 0.4–0.55M token. Claude 1M bağlama sığar ama MCP tool çıktısıyla taşınamaz. Soru başına ≈ $0.6–1.1 (Sonnet 5.5, cache'siz). |
| **(c) Hibrit — ÖNERİ** | Wiki ile gezinme + hibrit arama + segment okuma. Agent hangisini kullanacağına kendisi karar verir. | İki katmanın tutarlılığı gerekiyor. Çözüm: wiki maddeleri item/segment ID'si taşır ve lint bunları doğrular. |

**Kanıt.** arXiv 2605.15184 (Mayıs 2026): LongMemEval'in 116 sorusunda grep genelde vektörden isabetli, ama harness etkisi de güçlü. arXiv 2602.23368: agentic keyword search, RAG metriklerinin >%90'ına ulaşıyor. İkisi de İngilizce; Türkçe ve çapraz dil test edilmemiş. Bu yüzden lexical arama **birinci sınıf** kalıyor, vektör TR/EN ve parafraz için ekleniyor.

### 2.2 Remote MCP'nin ucuza servis edebileceği depolama/indeks seçenekleri

| Seçenek | Ücretsiz / en düşük ücret | Ops yükü | Hibrit (BM25+vektör) | Metadata filtre | Gecikme | Taşınabilirlik (lokal dosya?) | Not |
|---|---|---|---|---|---|---|---|
| **SQLite + FTS5 + sqlite-vec** (VPS/Fly container'da) | Yazılım $0. Hetzner ~€5.99/ay'dan | Orta: sunucu, TLS, yedek | Var, SQL'de RRF ile (Garcia/Willison deseni) | SQL `WHERE`; vec0 metadata ve partition kolonları | Yerel disk, çok düşük (ölçülmedi) | **Mükemmel**: aynı dosya | sqlite-vec pre-v1 (v0.1.9 stabil; DiskANN v0.1.10-alpha'da) |
| **Cloudflare D1 + Vectorize (+R2)** | Free: D1 5GB toplam (DB başına 500MB), Vectorize 5M saklanan / 30M sorgulanan boyut. Paid $5/ay: 10M/50M dahil, sonra $0.05/100M ve $0.01/1M | **Çok düşük**, serverless. MCP host'u ile aynı yerde | Var ama iki ayrı sistem: D1 FTS5 + Vectorize, füzyon Worker'da | Vectorize: 10 metadata index, `$eq/$in/$gte…`, namespace. topK ≤50 (metadata ile) | Worker binding, aynı ağ (ölçülmedi) | **Kısmi**: D1 export FTS5 sanal tablolarını desteklemiyor. Lokal kopya SoT'den üretilir | Vectorize en fazla 1536 boyut. Metadata index'leri veri yüklemeden **önce** kurulmalı |
| **Turso (libSQL)** | Free: 100 DB, 5GB, 500M satır okuma, 10M yazma. Developer $4.99 | Düşük (HTTP; Worker'dan erişilebilir) | Var, tek DB'de: native vektör (DiskANN, `vector_top_k`) + FTS5 (önceden yüklü) | SQL. ANN ile `WHERE` sırası (pre/post filter) **doğrulanamadı** | Ölçülmedi | **İyi**: libSQL dosyası / embedded replica | Platform geçişte: yeni Rust motoru alpha, Tantivy FTS deneysel. sqlite-vec yalnız Pro/Enterprise'da |
| **Postgres + pgvector — Neon** | Free: proje başına 1GB, 100 CU-saat, 5 dk sonra scale-to-zero | Düşük | pgvector + Postgres FTS; **`turkish` snowball stemmer** var. Neon'da pg_search (BM25) deprecated, yerine `lakebase_text` | SQL | Soğuk başlangıç var (ölçülmedi) | pg_dump ile, "dosya" değil | Türkçe stemming tek artısı |
| **Supabase** | Free 500MB; **7 gün inaktivitede pause**. Pro $25 | Düşük | pgvector + FTS | SQL | Ölçülmedi | pg_dump | Kişisel ve seyrek kullanımda pause riski var |
| **LanceDB** (OSS, S3/R2 üzerinde) | OSS $0. Cloud fiyatı **doğrulanamadı** | Orta: native binary gerekir, Workers'ta çalışmaz | Native FTS (BM25) + vektör + rerank | SQL filtre | Object storage'dan okuma (ölçülmedi) | İyi (dizin) | Container gerektirir |
| **Turbopuffer** | Launch planı **$16/ay minimum** | Çok düşük | BM25 + vektör | Var | Ölçülmedi | Yok | Bu ölçek için gereksiz büyük |
| **Qdrant Cloud** | Free: 1GB RAM, 0.5 vCPU, 4GB disk. **1 hafta inaktivitede askı, 4 haftada silme** | Düşük | Sparse + dense füzyon (sayfada BM25 geçmiyor) | Var | Ölçülmedi | Self-host mümkün | Silme politikası kişisel KB için riskli |
| **Chroma Cloud** | Starter $0 + $5/ay kredi. Yazma $2.50/GiB, saklama $0.33/GiB-ay | Çok düşük | "vector, full-text, metadata search" | Var | Ölçülmedi | Chroma OSS lokal | Kredi bu ölçeğe yeter; vendor kilidi |
| **Upstash Vector** | Free: günlük 10K sorgu+güncelleme, 1GB, 100 namespace. PAYG $0.40/100K istek | Çok düşük (HTTP) | **Hybrid index** (dense + BM25 sparse, RRF/DBSF) | Var | Ölçülmedi | Yok | BM25 tokenizer'ının Türkçe davranışı **doğrulanamadı** |
| **Cloudflare AI Search** (eski AutoRAG; all-in-one) | **1 Kasım 2026'dan itibaren** aylık 1.000 semantik + 1.000 FTS sorgu ücretsiz, sonra 1.000 sorgu başına $0.75 / $0.10. Ingestion 5M tokendan sonra $0.75/1M. Saklama $2/GB-ay | En düşük | Var: tokenizer `porter` (yalnız İngilizce) veya `trigram` | **Yalnız 5 alan** | Ölçülmedi | Yok | Hazır MCP endpoint'i var. Timestamp/segment şeması üzerinde kontrol zayıf. H6 için en iyi "SaaS" rakibi |

**Karar: Cloudflare D1 + Vectorize + R2.** P4 MCP host'u olarak Workers'ı seçti; tek vendor, tek fatura ($5), sıfır ops ve metadata filtre geliyor. Turso daha zarif (tek DB'de hibrit SQL) ama motor geçişi ve filtre semantiği belirsiz. **Plan B:** sunucu bir Python container'ına taşınırsa aynı şemadaki SQLite dosyası doğrudan servis edilir ve Litestream ile R2'ye yedeklenir. Şema SQLite-native olduğu için iki yol da açık.

### 2.3 Chunking seçenekleri

| Yöntem | Kanıt | Bizim için |
|---|---|---|
| Sabit token penceresi | Chroma raporu: Recursive 200/0 her metrikte istikrarlı. OpenAI varsayılanı (800/400) verimlilikte kötü (precision 1.4–1.5) | Taban çizgisi |
| **Timestamp hizalı, cümle ve chapter sınırlı pencere** | Transcript'te doğal birim cue/cümle. Atıf için `t_start` gerekli | **Seçildi** |
| Semantik chunking | ClusterSemantic 200: en iyi precision/IoU (8.0). LLM chunker: en iyi recall (91.9), ama pahalı | Chapter'ı olmayan videolarda P6'nın konu segmentasyonu bu rolü görür |
| Parent–child (small-to-big) | Küçük chunk = isabet, parent = bağlam | Child ~300 token, parent = chapter (≤2k) |
| **Contextual Retrieval** (Anthropic) | Top-20 hata %5.7 → %3.7 (−%35, contextual embedding), → %2.9 (−%49, + contextual BM25), → %1.9 (−%67, + rerank). Maliyet ≈ $1.02/1M doküman tokeni | Ucuz versiyon: deterministik başlık + P6'nın chapter özeti, $0. LLM'li versiyon Haiku 5.5 ile ≈ $0.0026/video (opsiyonel, eval'e bağlı) |
| Contextualized chunk embeddings (`voyage-context-3/4`) | Vendor iddiası: Contextual Retrieval'a göre chunk düzeyinde +%20.5 | Eval'de alternatif kol olarak denenebilir ($0.12/1M) |

### 2.4 Embedding modelleri (TR + EN)

| Model | $/1M token | Boyut | Bağlam | Çok dilli / Türkçe kanıt | Not |
|---|---|---|---|---|---|
| **voyage-4** | $0.06 (ilk 200M ücretsiz; Batch −%33) | 1024 (256–2048, MRL; int8/binary) | 32k | Dokümanda "multilingual" deniyor. RTEB'de voyage-4-large OpenAI-3-large'dan +%14 (vendor). **TR verisi yok** | voyage-4-large/-lite ile aynı uzayı paylaşır |
| voyage-4-lite | $0.02 | aynı | 32k | Dokümanda multilingual olarak **anılmıyor** | Sorgu tarafında asimetrik kullanılabilir |
| voyage-4-large / context-4 | $0.12 | aynı | 32k | En yüksek kalite (vendor) | |
| **OpenAI text-embedding-3-small** | $0.02 | 1536 (MRL) | ~8k (bu turda doğrulanmadı) | **TR-MTEB Retrieval 64.99**: test edilenlerin en iyisi (m-e5-large 60.62) | Tek doğrudan Türkçe kanıt |
| text-embedding-3-large | $0.13 | 3072 | ~8k (bu turda doğrulanmadı) | — | |
| **Gemini Embedding 2** | $0.20 ($0.10 batch); ücretsiz tier var | 3072 (128–3072 MRL) | 8k (ikincil) | MTEB Multilingual 69.9 (Google tablosu, ikincil kaynaklar üzerinden; 001 = 68.4) | Multimodal (görüntü/video da embed eder; P3 için ilginç). Free tier'da veri kullanımı riski |
| Cohere Embed 5 Pro / Fast | $0.12 / $0.08 (ikincil, **doğrulanamadı**) | — | — | — | Embed v4'ün yerini almış görünüyor |
| Jina embeddings v5 text-small | ~$0.05 (ikincil) | — | — | MMTEB 67.0 (Jina) | Ağırlıklar CC-BY-NC |
| Qwen3-Embedding-8B / 0.6B | Açık (Apache-2.0). Workers AI'da **0.6B $0.012** | 4096 / 1024 | 32k | MMTEB 70.58 (8B) / 64.33 (0.6B), Mayıs 2025 | 0.6B, Cloudflare içinde dış API anahtarı gerektirmeden çalışır |
| bge-m3 | Workers AI **$0.012** | 1024 (+sparse) | ~8k (doğrulanamadı) | TR-MTEB'de yok (**doğrulanamadı**) | Upstash'te hosted sparse olarak da var |

### 2.5 Reranker'lar

| Model | Fiyat | Not |
|---|---|---|
| **voyage rerank-2.5** / -lite | $0.05 / $0.02 per 1M token, ilk 200M ücretsiz. Token = sorgu×doküman + doküman tokenleri | Çok dilli, talimat alabiliyor, 32k bağlam. 50 aday × ~350 token ≈ 19k token ≈ **$0.001/sorgu** |
| voyage rerank-3 / -lite | $0.05 / $0.02 | Dokümanda "highest accuracy" deniyor; çok dillilik **belirtilmemiş** |
| Cohere Rerank 4 Fast / Pro | $2.00 / $2.50 per 1.000 arama (ikincil). 1 arama = 1 sorgu + ≤100 doküman; >500 token doküman bölünür | 100+ dil, 32k bağlam (ikincil) |
| Jina reranker v3 (0.6B) | Fiyat **doğrulanamadı** | MIRACL 66.50, 18 dil (Jina) |
| Workers AI bge-reranker-base | $0.003/1M | Küçük, EN/ZH odaklı. TR için önerilmez |

**Bu ölçekte değer mi?** Evet. Sebebi doğruluktan çok **token ekonomisi**: agent'a 50 parça yerine 8 parça döndürmek P4'ün ~4k token bütçesini korur. Maliyet ihmal edilebilir. Anthropic'in ölçümünde rerank ek olarak %2.9 → %1.9 kazandırdı. Rerank `search` için varsayılan açık, `rerank:false` ile kapatılabilir.

### 2.6 Knowledge graph seçenekleri

| Sistem | Ne yapar | Maliyet / kanıt | Bizim için |
|---|---|---|---|
| Microsoft GraphRAG | Entity ve ilişki çıkarımı + community özetleri (global arama) | İndeksleme pahalı. LazyGraphRAG, full GraphRAG indeksleme maliyetinin %0.1'iyle (vektör RAG ile aynı) ve global aramadan >700× ucuz sorguyla benzer kalite iddia ediyor | Global özetleri zaten derlenmiş `overview`/`topic` sayfaları veriyor |
| LightRAG | Graf + vektör, düşük sorgu maliyeti | Tasarruf sorgu tarafında; indeksleme hâlâ chunk başına LLM çıkarımı (ikincil) | Gereksiz |
| Graphiti / Zep | Bi-temporal graf, artımlı, BM25 + embedding + traversal | Neo4j/FalkorDB/Kuzu gerekir | "Fikir zamanla değişti mi?" soruları ve onlarca kanal üzerinde çok adımlı sorgular baskınlaşırsa |
| Cognee | ECL pipeline, graf + vektör | Apache-2.0 (ikincil) | Gereksiz |

**Ne zaman değer?** (1) Kanallar arası, çok adımlı sorgular (kişi → şirket → araç). (2) Zamansal çelişki takibi. (3) Toplamda >~10k video. MVP'de bunun yerine SQL'de hafif bir graf yeterli: `items —about→ topics`, `items —uses→ resources`, `mentions`.

---

## 3) Önerilen yaklaşım

### 3.1 Katmanlar

```
[Ingestion (P2/P3/P6, GH Actions)]
      │  idempotent: (video_id, content_hash, pipeline_version)
      ▼
kb-data (git, kanonik)  raw/…/{meta,transcript,visual}.json · _data/items.jsonl · wiki *.md · chunks.jsonl(+emb)
      │  build_index (deterministik, yeniden üretilebilir)
      ├──► Cloudflare: D1 (tablolar + FTS5) · Vectorize (1 index, metadata: kb_id, kind, lang, published_at, video_id) · R2 (ham, kareler)
      └──► Lokal: kb.sqlite (aynı tablolar + FTS5 + sqlite-vec)  → CLI, offline, eval
```

Embedding'ler `chunks.jsonl` içinde (veya R2'de parquet olarak) saklanır, böylece yeniden yükleme ücretsizdir. Model değişirse `embedding_model` alanı ile yeniden embed edilir. Kanallar arası arama için **tek Vectorize index + `kb_id` metadata filtresi** kullanılır. Namespace kullanılmaz, çünkü bir sorgu tek namespace'e bakar.

### 3.2 Veri modeli (SQLite/D1 şeması; P4 ve P6 ile uyumlu adlar)

| Tablo | Anahtar alanlar |
|---|---|
| `channels` | `kb_id` PK (`yt-<handle>`, `ig-<handle>`), `platform`, `handle`, `url`, `title`, `lang_primary`, `last_synced_at`, `videos_total`, `videos_indexed`, `kb_version` |
| `videos` | `video_id` PK (`yt:ID`, `ig:SHORTCODE`), `kb_id`, `url`, `title`, `description`, `published_at`, `duration_s`, `lang`, `format` (short/long), `transcript_source` (manual/auto/asr), `has_chapters`, `visual_analyzed`, `content_hash`, `pipeline_version`, `status`, `ingested_at` |
| `chapters` | `video_id`, `idx`, `title`, `start_s`, `end_s`, `summary` (1–2 cümle; bağlam başlığına girer) |
| `segments` (atıf çapası) | `segment_id` (`<video_id>#s0042` konuşma / `#v0007` ekran), `kind`, `start_s`, `end_s`, `text` (birebir; OCR dahil), `visual_type` (slide/code/terminal/url/repo/diagram/overlay) |
| `chunks` (arama birimi) | `chunk_id` (`<video_id>:<start_ms>` veya `item:<id>` / `page:<path>#<h>`), `kb_id`, `video_id`, `type` (`transcript_chunk\|visual_note\|insight\|playbook\|resource\|page`), `seg_from`/`seg_to`, `start_s`/`end_s`, `chapter_idx`, `context_header`, `text`, `text_norm` (TR katlama), `lang`, `token_count`, `published_at`, `citation_url`, `parent_id`, `embedding_model`, `content_hash` |
| `chunks_fts` | FTS5 (external content): `context_header`, `text_norm`, `title` (bm25 ağırlıkları 0.5 / 1.0 / 2.0) + trigram yardımcı tablo |
| `chunks_vec` | Lokal: sqlite-vec `vec0(embedding float[1024], kb_id partition key, type)`. Remote: Vectorize |
| `items` | P6 KnowledgeItem: `item_id`, `kb_id`, `kind` (tactic/framework/playbook/concept/claim/example), `statement_en`, `quote_orig`, `polarity`, `speaker`, `specificity`, `platform_sensitive`, `steps` (JSON; her adım `segment_id` taşır), `evidence` (JSON `[segment_id]`), `verified` |
| `topics`, `item_topics` | `topic_slug`, `parent`, `aliases` (TR/EN) |
| `resources`, `mentions` | `resource_id`, `kind` (tool/repo/website/book/course/person/product), `canonical_name`, `url`, `aliases`; mention: `resource_id`, `segment_id`, `video_id`, `kb_id`, `stance`, `source` (spoken/onscreen/description) |
| `pages` | `path`, `kb_id`, `page_type` (index/overview/topic/framework/playbook/video/resources/glossary), `title`, `body_md`, `item_refs`, `content_hash`, `updated_at` |
| `ingest_runs` | `run_id`, `kb_id`, `requested`, `new_videos`, `skipped`, `errors`, `cost_usd`, `pipeline_version` |
| `eval_items`, `eval_runs` | `question`, `lang`, `type` (fact/quote/synthesis/xling), `gold_video_ids`, `gold_segments`, `reference_answer`; skorlar |

`citation_url` deterministik üretilir: `https://www.youtube.com/watch?v=ID&t={start_s}s`. Instagram'da timestamp deep-link yok, bu yüzden metne "(0:12)" eklenir (P4). LLM hiçbir zaman timestamp yazmaz.

### 3.3 Chunking reçetesi (somut)

- **Transcript chunk:** hedef 300 token (min 120, maks 450). Bu, TR ve EN'de ≈60–90 saniyeye denk gelir. Kesim noktası cümle sonu, ≥1.5 sn duraklama ya da cue sınırıdır. **Chapter sınırını geçmez.** Overlap: önceki chunk'ın son cümlesi (≤%15, ≤15 sn). Chroma güçlü embedder'larda overlap'in IoU'yu düşürdüğünü buldu, bu yüzden 0 ile 1 cümle eval'de A/B edilecek.
- **Chapter yoksa:** P6 extraction'ın konu segmentasyonu kullanılır, o da yoksa 5 dakikalık sabit "pseudo-chapter".
- **Bağlam başlığı (~50–80 token, embedding ve FTS'e girer, kullanıcıya gösterilmez):** `Kanal: … | Video: … | Bölüm: … (mm:ss–mm:ss) | Tarih: 2026-03 | Bölüm özeti: …`.
- **Görsel not:** her `v####` olayı (kod bloğu, slayt, repo URL'si, UI adımı) ayrı bir chunk. Metin = birebir OCR + kısa açıklama.
- **Reel:** tek chunk (caption + transcript + overlay metni).
- **Item:** tek chunk (`statement_en` + `quote_orig` + adımlar).
- **Wiki sayfası:** H2/H3 bazında 300–600 token.
- **Small-to-big:** `search` child döndürür. `expand=chapter|±n` ile parent veya komşular okunur.

### 3.4 Türkçe lexical arama

- FTS5'in `porter` stemmer'ı yalnız İngilizce. `unicode61 remove_diacritics 2` ş/ç/ğ/ö/ü'yü katlar, ama **ı/İ** sorunu uygulamada çözülmeli: önce Türkçe-duyarlı lowercase (İ→i, I→ı), sonra ı→i katlaması. Bu `text_norm` alanına yazılır ve sorgu da aynı fonksiyondan geçer.
- Ekler için `prefix='2 3'` index'i ve sorguda kök + `*` kullanılır (`satış*`). Trigram yardımcı tablo alt dizgi eşleşmesi sağlar (en az 3 karakter).
- Çapraz dil: semantik tarafı çok dilli embedding taşır. Persona skill'i agent'a anahtar terimleri **TR ve EN iki sorguyla** aramasını söyler (P4/P6).
- Not: Postgres'in `turkish` stemmer'ı tek gerçek "stemming" seçeneği. Eval'de lexical recall düşük çıkarsa Neon kolu yeniden değerlendirilir.

### 3.5 Retrieval pipeline (agent sorgusu)

```
Agent (Claude Code cloud/CLI, Desktop, claude.ai) + kanal persona skill'i
  │ 1. get_channel_overview(kb_id)  → overview + index (≤5k token; 1000+ videoda hiyerarşik: topic → video)
  │ 2. search(query, kb_ids, types?, date_from?, k=8)
  ▼
Worker /search
  ├─ q_norm = tr_fold(query) ; q_vec = embed(query)            (voyage-4 / Workers AI)
  ├─ paralel: D1 FTS5 bm25 top-50  (WHERE kb_id IN …, type IN …)
  │           Vectorize top-50     (filter kb_id/type/published_at)
  ├─ RRF füzyonu (k=60; ağırlıklar eval ile)        → top-50 aday
  ├─ rerank-2.5 (instruction: "uygulanabilir adım/kanıt içeren parçayı öne al") → top-k
  ├─ dedupe: video başına ≤2 parça; group_by kb|video opsiyonel
  └─ yanıt: snippet(≤250 tok) + video_title + start_s/end_s + citation + chunk_id  (toplam ≤~4k)
  │ 3. get_transcript(video_id, start_s, end_s) / expand=chapter   (birebir alıntı)
  │ 4. kb_read(path) / get_playbook / find_resources                (wiki gezinme)
  ▼
Agent sentezler → her iddia için [Video @ mm:ss](…&t=Ns)
```

Gecikme bütçesi (tahmini, **ölçülmedi**): embed ~100–200 ms + D1/Vectorize paralel ~50–150 ms + rerank ~150–300 ms. Toplam <1 sn hedefleniyor.

### 3.6 Minimal eval döngüsü (kanal başına)

1. **Golden set (~30 soru, ~$0.1).** 20 olgu/alıntı sorusu: rastgele chunk'lardan Haiku 5.5 ile soru + referans cevap + gold `segment_id` üretilir. 5 sentez sorusu: overview ve topic sayfalarından, rubrikli. 5 çapraz dil sorusu: TR kanala EN soru ve tersi. İnsan 10 tanesini hızlıca kontrol eder. `eval_items` tablosuna yazılır.
2. **Retrieval metrikleri (deterministik, ücretsiz):** video düzeyinde Recall@5/@10 ve MRR, segment düzeyinde isabet (gold aralıkla ±30 sn örtüşme).
3. **Cevap metrikleri:** LLM-judge (Sonnet 5.5; 30 soru ≈ $0.3) ile doğruluk (referansa göre) ve faithfulness (her iddia döndürülen parçalarla destekleniyor mu). Ayrıca **atıf doğrulama** kodla yapılır: URL var mı, `t` süre içinde mi, alıntı segmentte geçiyor mu. RAGAS metrikleri (context precision/recall, faithfulness, response relevancy) hazır alternatif.
4. **Ne zaman koşulur:** her ingest'ten sonra (CI) ve her konfigürasyon değişikliğinde (chunk boyutu, overlap, embedding modeli, rerank, RRF ağırlıkları). Sonuçlar `eval_runs` tablosuna yazılır ve gerilemede alarm verilir.
5. **İlk deney matrisi:** {voyage-4, oai-3-small, gemini-emb-2, qwen3-0.6b} × {rerank açık/kapalı} × {overlap 0 / 1 cümle}.

---

## 4) Maliyet

**Birim fiyatlar (retrieval katmanı):**

| Kalem | Fiyat |
|---|---|
| Cloudflare Workers Paid | $5/ay minimum (10M istek dahil) |
| D1 | Free: 5M satır okuma/gün, 100k yazma/gün, 5GB. Paid: 25B okuma ve 50M yazma dahil, 5GB dahil, sonra $0.75/GB-ay. DB başına en fazla 10GB |
| Vectorize | Free: 30M sorgulanan + 5M saklanan boyut. Paid: 50M + 10M dahil, sonra $0.01/1M sorgulanan, $0.05/100M saklanan |
| R2 | 10GB-ay ücretsiz, egress ücretsiz, sonra $0.015/GB-ay |
| Workers AI | Günde 10k neuron ücretsiz. bge-m3 ve qwen3-emb-0.6b $0.012/1M |
| voyage-4 / rerank-2.5 | $0.06 / $0.05 per 1M (her biri için ilk 200M ücretsiz) |
| Claude Haiku 5.5 / Sonnet 5.5 | $0.10/$0.50 ve $2/$10 per 1M (giriş/çıkış). Cache okuma $0.01 ve $0.10. Batch −%50 |

**Senaryolar** (15 dk ortalama, TR/EN karışık, video başına ~20 vektör, 1024 boyut):

| | 100 video (1 kanal) | 1000 video (veya ~10 kanal × 100) | ~9.000 video (30 kanal) |
|---|---|---|---|
| Embed edilen token | ~0.7M → $0.04 (200M ücretsiz kotada $0) | ~7M → $0.42 ($0) | ~60M → $3.6 ($0) |
| Vektör sayısı / saklanan boyut | ~2k / 2.0M → **Free tier'a sığar** | ~20k / 20.5M → $0.005/ay | ~180k / 184M → ~$1.5/ay (sorgu boyutu dahil) |
| D1 boyutu | ~10MB | ~100MB | ~1GB |
| Rerank (aylık 500 / 3.000 / 3.000 sorgu) | $0.48 (ücretsiz kotada) | $2.85 (ücretsiz kotada) | $2.85 |
| Opsiyonel LLM bağlam başlığı (Haiku 5.5) | $0.26 bir kez | $2.6 bir kez | ~$24 bir kez |
| **Aylık serving** | **$0** (Free) – $5 | **≈ $5–8** | **≈ $7–10** |

Karşılaştırma için: "her soruda tüm korpusu bağlama koy" yaklaşımı 100 TR video için soru başına ≈ $0.6–1.1 tutar (Sonnet 5.5, cache'siz). Wiki derleme ve extraction maliyeti P6'nın tahminine dahil (100 video ≈ $3–5).

---

## 5) Riskler & azaltma

| Risk | Azaltma |
|---|---|
| Türkçe lexical recall düşük (ekler, ı/İ) | `text_norm` katlama, prefix ve trigram, çok dilli vektör, eval'de TR alt kümesi. Gerekirse Postgres `turkish` stemmer |
| Embedding modelinin Türkçe kalitesi bilinmiyor | Model-agnostik şema (`embedding_model`), ucuz yeniden embed, ilk hafta eval matrisi |
| MCP çıktı limitleri (Claude Code: 10k uyarı, 25k varsayılan; claude.ai limiti **doğrulanamadı**) | Sonuç başına ≤250 token, k=8, cursor, `expand` ile tembel okuma, hiyerarşik index |
| Wiki ile transcript arasında kayma ve halüsinasyon | Her wiki maddesi item/segment ID'si taşır. Lint, ID'leri ve alıntı eşleşmesini doğrular. Timestamp'ler koddan üretilir |
| Vendor ve fiyat değişimi (Turso motor geçişi, Neon pg_search'ün kaldırılması, AI Search faturalamasının 1 Kasım 2026'da başlaması) | Kanonik git SoT, indeksler türev ve yeniden üretilebilir. Lokal SQLite yolu her zaman hazır |
| D1 export'un FTS5 sanal tablolarını içermemesi | Yedek D1'den değil SoT'den alınır. FTS build'den yeniden kurulur |
| Vectorize'da metadata index'in veriden sonra kurulması | Index'ler build script'inde ilk adım olarak kurulur. Aksi halde yeniden upsert gerekir |
| Free tier pause/silme (Supabase 7 gün; Qdrant 1 hafta askı, 4 hafta silme) | Bu servisler birincil olarak seçilmedi |
| sqlite-vec'in pre-v1 olması | Sürüm sabitlenir. Yalnız lokalde kullanılır. Brute-force KNN ~20k vektörde yeterli |
| Transcript içinden prompt injection | Snippet'ler veri olarak işaretlenip sarılır (P4). Tool'lar salt okunur |
| Gemini free tier'da verinin ürün geliştirmede kullanılması | Ücretli tier veya Voyage/OpenAI tercih edilir |

---

## 6) Diğer personalara notlar / açık sorular

- **P2 (acquisition):** cue (tercihen kelime) düzeyinde `start/end`, `chapters` (yt-dlp), `published_at`, `duration`, caption dili ve kaynağı (manual/auto/ASR), açıklama linkleri, `content_hash`. Idempotency anahtarı: `(video_id, content_hash, pipeline_version)`.
- **P3 (visual):** her olay `v####` ID'si, `start_s/end_s`, `visual_type` ve birebir OCR taşımalı; her biri ayrı chunk olur. Gemini Embedding 2 kareleri de embed edebilir (MVP dışı).
- **P4 (MCP):** pipeline P4'ün tool setiyle (`search`, `get_transcript`, `get_channel_overview`…) uyumlu. Chunk ID `<video_id>:<start_ms>`, tek index + `kb_id` filtresi, `kb_ids:["*"]` destekli. `search` için `rerank` ve `expand` parametreleri öneriyorum. Cloud oturumlar `kb-data`'yı klonlayıp grep de yapabilir.
- **P5 (cost):** bölüm 4. Baskın kalem Workers Paid ($5); embedding ve rerank ücretsiz kotada. Cloudflare AI Search H6'nın all-in-one alternatifi, ama porter tokenizer ve 5 metadata alanı yüzünden birincil değil.
- **P6 (ontology):** üç indeks birimi ve filtre alanları şemada. `statement_en` + `quote_orig` birlikte embed ve FTS'e girer; topic `aliases` (TR/EN) FTS'e eklenir.
- **Açık sorular:** (1) claude.ai/Desktop'ın tool sonucu boyut limiti? (2) Beklenen aylık sorgu hacmi? (rerank kotası ve Vectorize tahmini buna bağlı.) (3) Cross-channel arama MVP'de varsayılan mı olsun? (4) Wiki derlemesi Karpathy gibi denetimli mi yapılsın, tam otomatik mi? (5) Embedding seçimi eval sonucuna kadar `voyage-4` mü, Cloudflare-içi `qwen3-0.6b` mi? (Gizlilik ve tek-vendor tercihi.)

**Hipotez kararları (bu persona açısından):**
- **H4:** büyük ölçüde doğru. Ayrı bir vektör DB vendor'ı gerekmiyor, ama remote servis için D1 + Vectorize (veya bir container'da SQLite) şart.
- **H5:** tek MCP + tek index + `kb_id` filtresi destekleniyor.
- **H6:** retrieval katmanında doğru. Hazır SaaS (AI Search) Türkçe ve timestamp kontrolünde zayıf kalıyor.

---

## 7) Kaynaklar

- Karpathy, LLM Wiki gist: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- Karpathy deseni, ikincil özet (Nisan 2026): https://codex.danielvaughan.com/2026/04/11/karpathy-llm-knowledge-bases-codex-resources-flywheel/
- qmd (FTS5 + sqlite-vec + rerank, MCP): https://github.com/tobi/qmd
- Anthropic, Contextual Retrieval: https://www.anthropic.com/news/contextual-retrieval
- Claude Code agentic search / Cherny (ikincil): https://officechai.com/ai/claude-researcher-explains-how-agentic-search-performed-better-than-rag-for-code-generation/ · https://smartscope.blog/en/ai-development/practices/rag-debate-agentic-search-code-exploration/
- Karşı görüş (vendor): https://zilliz.com/blog/why-im-against-claude-codes-grep-only-retrieval-it-just-burns-too-many-tokens
- arXiv 2605.15184 "Is Grep All You Need?": https://arxiv.org/abs/2605.15184
- arXiv 2602.23368 "Keyword search is all you need": https://arxiv.org/abs/2602.23368v1
- Claude Code MCP limitleri: https://code.claude.com/docs/en/mcp
- Claude fiyatları: https://platform.claude.com/docs/en/about-claude/pricing
- Cloudflare Vectorize fiyat: https://developers.cloudflare.com/vectorize/platform/pricing/ · limitler: https://developers.cloudflare.com/vectorize/platform/limits/ · metadata filtre: https://developers.cloudflare.com/vectorize/reference/metadata-filtering/
- Cloudflare D1 fiyat: https://developers.cloudflare.com/d1/platform/pricing/ · limitler: https://developers.cloudflare.com/d1/platform/limits/ · FTS5: https://developers.cloudflare.com/d1/sql-api/sql-statements/ · export kısıtı: https://developers.cloudflare.com/d1/best-practices/import-export-data/
- Cloudflare Workers fiyat: https://developers.cloudflare.com/workers/platform/pricing/ · R2: https://developers.cloudflare.com/r2/pricing/ · Workers AI: https://developers.cloudflare.com/workers-ai/platform/pricing/
- Cloudflare AI Search fiyat/limit: https://developers.cloudflare.com/ai-search/platform/limits-pricing/ · hibrit: https://developers.cloudflare.com/changelog/post/2026-04-16-hybrid-search-and-relevance-boosting/
- Turso fiyat: https://turso.tech/pricing · vektör: https://docs.turso.tech/features/ai-and-embeddings · eklentiler: https://docs.turso.tech/features/sqlite-extensions · FTS: https://docs.turso.tech/sql-reference/functions/fts
- sqlite-vec: https://github.com/asg017/sqlite-vec · sürümler: https://github.com/asg017/sqlite-vec/releases · hibrit SQL: https://simonwillison.net/2024/Oct/4/hybrid-full-text-search-and-vector-search-with-sqlite/
- SQLite FTS5: https://www.sqlite.org/fts5.html
- Supabase: https://supabase.com/pricing · Neon: https://neon.com/pricing · Neon pg_search deprecated: https://neon.com/docs/extensions/migrate-pg-search-to-lakebase-text
- Postgres Türkçe sözlük: https://www.postgresql.org/docs/9/textsearch-psql.html
- LanceDB: https://lancedb.com/docs/search · https://docs.lancedb.com/features
- Turbopuffer: https://turbopuffer.com/pricing · Qdrant: https://qdrant.tech/pricing/ · https://qdrant.tech/documentation/cloud/create-cluster/ · Chroma: https://www.trychroma.com/pricing · Upstash: https://upstash.com/pricing/vector · https://upstash.com/docs/vector/features/hybridindexes
- Hetzner: https://www.hetzner.com/cloud/
- Chroma chunking raporu: https://www.trychroma.com/research/evaluating-chunking
- Voyage fiyat: https://docs.voyageai.com/docs/pricing · modeller: https://docs.voyageai.com/docs/embeddings · reranker: https://docs.voyageai.com/docs/reranker · contextualized: https://docs.voyageai.com/docs/contextualized-chunk-embeddings · voyage-4 blog: https://blog.voyageai.com/2026/01/15/voyage-4/ · voyage-context-3 (vendor): https://mongodb.com/company/blog/product-release-announcements/voyage-context-3-focused-chunk-level-details-global-document-context
- OpenAI fiyat: https://developers.openai.com/api/docs/pricing
- Gemini fiyat: https://ai.google.dev/gemini-api/docs/pricing · Gemini Embedding 2: https://deepmind.google/models/gemini/embedding/
- Cohere fiyat: https://cohere.com/pricing · Embed 5 (ikincil): https://www.eesel.ai/blog/cohere-embed-5-pricing · Rerank 4 (ikincil): https://openrouter.ai/cohere/rerank-4-pro · https://developer.puter.com/tutorials/cohere-api-pricing/
- Jina v5: https://jina.ai/news/jina-embeddings-v5-text-distilling-4b-quality-into-sub-1b-multilingual-embeddings · reranker v3: https://jina.ai/models/jina-reranker-v3/
- Qwen3-Embedding: https://huggingface.co/Qwen/Qwen3-Embedding-8B · OpenRouter (ikincil): https://openrouter.ai/models/qwen/qwen3-embedding-8b
- TR-MTEB (EMNLP Findings 2025): https://aclanthology.org/2025.findings-emnlp.471.pdf
- Tokenizer fertility TR/EN: https://arxiv.org/pdf/2603.05883
- Konuşma hızı: https://virtualspeech.com/blog/average-speaking-rate-words-per-minute
- LazyGraphRAG: https://www.microsoft.com/en-us/research/blog/lazygraphrag/ · LightRAG maliyet analizi (ikincil): https://particula.tech/blog/lightrag-vs-graphrag · Graphiti: https://cdn.jsdelivr.net/gh/getzep/graphiti@main/README.md · Zep makalesi: https://arxiv.org/abs/2501.13956 · Cognee: https://pypi.org/project/cognee/
- RAGAS metrikleri: https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/
