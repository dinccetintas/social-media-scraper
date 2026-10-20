# 00 — Sentez: Kanal → Knowledge Base → Agent Sistemi

**Tarih:** 2026-10-20 · **Durum:** Araştırma tamam; mimari seçimi kullanıcı onayı bekliyor.
**Girdi:** 6 persona raporu (01–06) ve Devil's Advocate incelemesi (07). Fiyat ve iddiaların kaynakları ilgili raporlarda.

| # | Rapor | Persona |
|---|---|---|
| 01 | [rag-architecture](01-rag-architecture.md) | Lead AI Engineer / RAG Architect |
| 02 | [data-acquisition](02-data-acquisition.md) | Data Acquisition Engineer (bu container'da canlı testlerle) |
| 03 | [video-understanding](03-video-understanding.md) | Multimodal Video Analyst |
| 04 | [agent-integration](04-agent-integration.md) | Agent Platform & MCP Engineer |
| 05 | [market-and-cost](05-market-and-cost.md) | Build-vs-Buy & Cost Analyst |
| 06 | [knowledge-design](06-knowledge-design.md) | Knowledge Engineer |
| 07 | [devils-advocate-review](07-devils-advocate-review.md) | Principal Reviewer (çelişki çözümü, YAGNI, doğrulama) |

---

## 1. Hedef ve onaylanmış kararlar

**Hedef:** Kullanıcı bir YouTube veya Instagram kanal linki ve bir sayı verir. Sistem videoları çeker, transkript çıkarır, ekranda gösterilen kodu, repo'ları ve slaytları analiz eder, bilgiyi yapılandırır ve **kanal başına bir knowledge base** oluşturur. Kullanıcının agent'ları bu KB ile "konuşur" ve her iddiayı **video + zaman damgası** ile kaynaklandırır.

**Onaylanan kararlar:**
- İlk sürüm YouTube ve Instagram'ı birlikte kapsıyor.
- Erişim üç yüzeyden olacak: Claude Code cloud, lokal CLI, Claude Desktop/claude.ai.
- Bütçe: ucuz ücretli API'ler kabul.
- Kişisel kullanım, artımlı güncelleme.

**Başarı tanımı:** "Kanal X hakkında sorduğum soruya agent'ım, kanalın gerçekten söylediği şeyle ve tıklanabilir `&t=` linkiyle cevap veriyor."

---

## 2. Hipotezlerin akıbeti

| H | Hipotez | Sonuç | Kanıt |
|---|---|---|---|
| H1 | YouTube için en ucuz yol yt-dlp + hazır altyazı; cloud IP'leri engelleniyor | ✅ **Doğru, bir nüansla** | Bu container'da test edildi (02 §3). Listeleme ve RSS çalışıyor. `youtube-transcript-api` → `IpBlocked`. yt-dlp varsayılan modu → "Sign in to confirm you're not a bot". **`web_embedded` client ile altyazı indirildi** (TR manuel + otomatik), ama yol resmî değil, kırılgan. Ses/video indirme buluttan **tamamen bloklu**. |
| H2 | Gemini'ye YouTube URL'si vererek ucuz görsel analiz | ⚠️ **Kısmen** | Çalışıyor; preview aşamasında, şimdilik ücretsiz, yalnız public videolar (07 §5'te doğrulandı). Ancak kod/URL okumak için `high` çözünürlük şart (03 §2.2). 2.5 Flash-Lite yeni projelere kapalı; doğru model **3.5 Flash-Lite**. Instagram için URL desteği yok. |
| H3 | Instagram için ücretli scraper gerekli | ✅ **Doğru** | Anonim erişim bu container'dan login'e yönlendirildi (02 T12). Resmî Business Discovery, kullanıcının kendi IG Business hesabı + FB App'ini istiyor. **Apify reel scraper** $2.3–2.6 / 1.000 reel (07 §5'te doğrulandı). |
| H4 | Ayrı vektör DB gerekmez; git'te markdown + SQLite hibrit arama yeter | ✅ **Büyük ölçüde doğru** | Kanonik veri git'te (JSONL + MD). MVP'de arama FTS5 + Türkçe katlama ile yapılıyor. Vektör yalnız eval gerektirirse eklenecek (01, 07 §3). |
| H5 | Tek MCP server + kanal başına skill/plugin | ✅ **Düzeltmeyle** | **Remote MCP, claude.ai custom connector olarak bir kez eklenir.** Böylece cloud, CLI, Desktop ve claude.ai'nin hepsinde otomatik görünür (04 §2, 07 §5'te doğrulandı). Persona tarafında kanal başına plugin yerine **tek bir `kb-expert` skill** yeterli. |
| H6 | Hazır SaaS yerine ucuz API'lerle kendimiz kurmak | ✅ **Doğru** | Uçtan uca hazır ürün yok; en yakın seçenekler ayda $50–140 ve onlar da veri çekmeyi bize bırakıyor. Karar: **veri çekme katmanını satın al (Apify, transkript vendor'ı), KB ve MCP'yi kendin kur** (05 §3). |

**Elenenler:**
- **vidIQ MCP:** bakiye 0, çağrı başı 5 kredi, en fazla 12 reel, sayfalama yok.
- **Higgsfield:** pipeline için uygun değil.
- **NotebookLM:** resmî API/MCP yok, IG yok.
- **Graphlit:** kapanıyor.
- **Twelve Labs:** Gemini'ye göre ~10× pahalı.

---

## 3. Mimari seçenekler

### Seçenek A — "Yalın, remote-first, aşamalı" ⭐ ÖNERİLEN
Devil's Advocate'in daraltılmış dikey dilimi. Önce çekirdek değer kurulur: altyazı → Sonnet extraction → kodla doğrulanmış alıntılar → FTS5 arama → tek skill. Remote MCP **v0.2**'de gelir; OAuth spike'ı ise ilk hafta paralel yapılır. Görsel analiz ve Instagram v0.2'de. Vektör, rerank, wiki derleyicisi ve kanal başına plugin'ler yalnızca eval ihtiyaç gösterirse eklenir.
- **Artı:** ~3–4 günde ilk kullanılabilir sonuç. Az vendor, az secret, az kırılma noktası. Her katman yeniden üretilebilir.
- **Eksi:** v0.1'de claude.ai/Desktop erişimi yok; o v0.2'de geliyor. Çapraz dil aramada vektörsüz başlangıç, Türkçe recall'u eval'de izlemeyi gerektiriyor.

### Seçenek B — "Tam özellikli ilk gün"
Raporların tam önerisi: D1 + Vectorize + voyage-4 + rerank, üç görsel geçiş, wiki derleyicisi + registry, 11 tool, kanal başına plugin marketplace, GitHub Actions ana tetik.
- **Artı:** Teoride daha iyi retrieval ve daha zengin wiki.
- **Eksi:** 2–3 kat karmaşıklık ve maliyet. Faydası henüz kanıtlanmamış bileşenler var. GitHub Actions runner'ları Azure IP'si olduğu için YouTube engelline takılabilir (doğrulanmadı).

### Seçenek C — "Hazır ürünle hızlı başlangıç"
NotebookLM veya Pinecone Assistant + Supadata.
- **Artı:** Saatler içinde demo.
- **Eksi:** Instagram yok, görsel analiz yok, timestamp'li atıf zayıf. Ayda $50+, vendor bağımlılığı. Gereksinimleri karşılamıyor.

**Öneri: A.** Çekirdek değeri en hızlı, en ucuz ve en düşük riskle teslim ediyor. B'nin parçaları, eval veriye dayanarak gerektirdiğinde v1.x'te eklenebilir.

---

## 4. Önerilen mimari (Seçenek A)

```
                ┌──────────── INGESTION (Python CLI: smk) ─────────────┐
 kanal URL ──►  │ keşif ─► transkript ─► [görsel] ─► extraction ─► verify │ ──► kb-data (private git: JSONL + MD)
 + limit        │ yt-dlp   web_embedded   Gemini 3.5   Sonnet 5.5    kod │            │ build
                │ DataAPI  → vendor       FL (high)    (reel: Haiku)     │            ▼
                │ RSS      → Gemini URL   IG: Apify→mp4→Gemini           │   kb.sqlite (lokal, FTS5)
                └────────────────────────────────────────────────────────┘   D1 (remote, FTS5)
                                                                                     │
   Claude Code cloud ─┐                                                              ▼
   Claude Code CLI ───┼── claude.ai custom connector (OAuth, GitHub allowlist) ──► kb-mcp Worker (Cloudflare, salt okunur)
   Desktop/claude.ai ─┘        + kb-expert skill (persona talimatı)                  5 okuma + 1 yazma tool'u
```

| Katman | Seçim | Gerekçe | Yedek |
|---|---|---|---|
| Keşif | `yt-dlp --flat-playlist` + YouTube Data API v3 + RSS (artımlı) | Hepsi buluttan test edildi, ücretsiz | — |
| YT transkript | `web_embedded` → **tek vendor adapter** (ScrapeCreators veya Supadata; pilotta ölçülüp seçilecek) → Gemini URL | Önce ücretsiz yol; kırılırsa otomatik yedek. `transcript_source` güven etiketi tutulur | — |
| Instagram | Apify reel scraper → mp4 → **tek Gemini 3.5 FL çağrısı** (transkript + ekran) | Tek ek vendor; Groq gerekmiyor | ScrapeCreators |
| Görsel | Gemini 3.5 FL, YouTube URL, `high` @0.5 fps, **yalnız heuristik router'ın işaretlediği** videolarda | Kod okumayı sağlayan tek ayar; maliyet kontrolü router'da | Görsel katman opsiyonel; KB onsuz da çalışır |
| Extraction | **Sonnet 5.5** (uzun video), **Haiku 5.5** (reel); structured output; alıntılar kodla doğrulanır, timestamp'i kod türetir | KB kalitesini belirleyen tek çağrı. Haiku'ya geçiş için 5 videoluk A/B testi | — |
| Ontoloji | 5 tip: Channel, Video (+Segment), Topic, KnowledgeItem (tactic/framework/playbook/concept/claim/example), Resource | Minimal ama yeterli (06 §2) | — |
| Kanonik kaynak | Private `kb-data` git repo'su (JSONL + MD; embedding yok) | Tüm indeksler bundan yeniden üretilebilir | — |
| Arama | SQLite/D1 **FTS5** + Türkçe ı/İ katlama; lokal ve remote aynı SQL | Ucuz ve yeterli. Recall@10 < 0.8 çıkarsa vektör eklenir | voyage-4 + Vectorize |
| Serving | Cloudflare Workers (Free) remote MCP: `createMcpHandler` + `workers-oauth-provider` + GitHub login allowlist | Tek entegrasyonla dört yüzeyin hepsi | Workers Paid $5/ay |
| Tool'lar | `list_kbs`, `get_overview`, `search`, `get_transcript`, `read_page` + (v1) `ingest_channel` | 11 tool yerine 6; yanıtlar ≤4k token, sayfalı, hazır `citation` alanı | — |
| Persona | Tek `kb-expert` skill + MCP `instructions` | Belgelenmemiş plugin-sync davranışına bağımlı değil | Kanal başına plugin (v1.x) |
| Ingestion yeri | v0.x: **ayrı ingest environment'ında cloud oturumu** · v1: smoke test geçerse GitHub Actions + gece RSS taraması | Testler cloud ortamında yapıldı; GH Actions doğrulanmadı | Ev makinesi (self-hosted runner) |
| Dil/repo | `pipeline/` (Python 3.13, uv) · `server/` (TS, wrangler) · `schema/` (tek kaynak, codegen) · `sql/` · `skills/` · `evals/` | Her dil kendi ekosisteminde | — |

**Güvenlik ilkeleri:**
- MCP tool'ları salt okunur. Yazma işlemi (`ingest_channel`) ayrı bir OAuth scope'ta ve kullanıcı onayı gerektiriyor.
- Transkript içeriği `untrusted` olarak etiketlenir (prompt injection riski).
- Sohbet ortamlarında API anahtarı tutulmaz; anahtarlar yalnız ingest ortamında.

---

## 5. Maliyet (uzlaştırılmış, 07 §1.1)

| Senaryo | Tahmin |
|---|---|
| 100 YouTube videosu × 15 dk (Haiku extraction) | ≈ **$2.6** |
| 100 YouTube videosu × 15 dk (Sonnet extraction; önerilen başlangıç) | ≈ **$6.2** |
| 100 Instagram reel × 60 sn | ≈ **$1.2** |
| Sürekli kullanım (10 kanal, ~87 yeni video/ay) | ≈ **$2.5–5.5/ay** (Workers Paid'e geçilirse +$5) |
| Geliştirme sırasında yeniden işleme | **+%30–100** tampon |

**Notlar:**
- Maliyetin çoğu görsel analiz ve Sonnet extraction'dan geliyor.
- Gemini YouTube URL girdisi gerçekten ücretsiz kalırsa görsel kalem ~%80 düşer.
- **Koruma:** `--dry-run` ön tahmin, `--max-usd` limiti, varsayılan `max_duration=90 dk`, her çağrının `costs.jsonl`'a kaydı.

**Fiyat takvimi:** Gemini 3.6/3.8 Flash fiyatı 1 Ocak 2027'de iki katına çıkıyor; 3.1 FL 7 Mayıs 2027'de kapanıyor. Bu yüzden model adları ve fiyatlar `config.yaml`'da tutulacak.

---

## 6. Yol haritası

| Aşama | Kapsam | Çıkış kriteri |
|---|---|---|
| **v0.1** (~3–4 gün) | 1 YouTube kanalı, 10 video: keşif → altyazı → Sonnet extraction → alıntı doğrulama → `kb-data` → FTS5 + `smk search` → `kb-expert` skill → 15 soruluk eval. **Paralel:** Worker + GitHub OAuth + claude.ai connector "ping" spike'ı | 15 sorunun en az 12'si doğru ve geçerli atıflı |
| **v0.2** (~1 hafta) | Remote MCP (D1 FTS5, 5 tool) dört yüzeyden erişilebilir · görsel heavy geçiş + GitHub repo doğrulama · Instagram (Apify → Gemini → Haiku) · maliyet takibi, dry-run, max-usd | Aynı soruya claude.ai, Desktop, CLI ve cloud'dan atıflı cevap |
| **v1** (~1–2 hafta) | `ingest_channel` (sohbetten tetikleme + tahmin/onay) · resume'lu durum makinesi, "all" için parçalı çalıştırma · GH Actions (smoke test geçerse) + gece RSS · 20 soruluk eval CI'da · Haiku/Sonnet A/B · `smk remove` | "@x'ten 100 video ekle" sohbetten uçtan uca çalışıyor |
| **v1.x** | Eval gerektirirse vektör + rerank · Batch/Flex · wiki derleyicisi (topic/playbook sayfaları, kanonikleştirme) · kanallar arası `_topics` · kanal başına persona plugin'leri · keyframe/OCR yolu | — |

---

## 7. En büyük 5 risk

1. **Datacenter IP'den YouTube altyazısı** (skor 20/25). `web_embedded` her an kırılabilir.
   **Önlem:** vendor adapter v0.1'de hazır, haftalık canary testi, her run'da `transcript_source` dağılımı raporu.
2. **Gemini YouTube URL'nin preview olması** (12). Buluttan video indirilemediği için yedeği yok.
   **Önlem:** görsel katman opsiyonel; ev makinesi varsa Files API yedeği.
3. **Döngüsel doğrulama** (9). Gemini'nin ürettiği transkript ve OCR, yine Gemini'nin çıktısına karşı doğrulanıyor.
   **Önlem:** güven sırası manuel > auto > vendor > gemini; agent'a "≈" etiketiyle gösterim.
4. **Remote MCP OAuth zinciri** (9). Hazır template'ler deprecated `McpAgent` kullanıyor.
   **Önlem:** ilk hafta spike, DCR yeterli.
5. **Prompt injection** (9). AI/Claude Code kanallarında prompt metni kesin olarak bulunacak.
   **Önlem:** extraction tool'suz yapılır; içerik `untrusted` etiketli; sohbet ortamında anahtar yok.

---

## 8. Uygulama başlarken kullanıcıdan gerekecekler

Anahtarlar **hiçbir zaman sohbete yapıştırılmaz**. Cloud environment ayarlarına (oturum başlığındaki environment menüsü → Edit → *Network secrets* veya environment variable) eklenir; yeni oturum otomatik okur. Değişken adları spec'te netleşecek.

| Ne | Ne zaman | Not |
|---|---|---|
| Anthropic API key | v0.1 | Extraction (Sonnet/Haiku 5.5) |
| YouTube Data API v3 key | v0.1 | Ücretsiz (Google Cloud Console) |
| Transkript vendor'ı (ScrapeCreators **veya** Supadata) | v0.1 | Biri yeter; pilotta seçilecek |
| Gemini API key (paid tier önerilir) | v0.1 (yedek) / v0.2 (görsel) | Free tier'da veri ürün geliştirmede kullanılıyor |
| Private `kb-data` GitHub repo'su | v0.1 | Bu session'a erişim olarak eklenmeli |
| Cloudflare hesabı + GitHub OAuth App | v0.2 (spike: v0.1) | Remote MCP için |
| Apify token | v0.2 | Instagram |

**Network erişimi:** Gerekirse environment'ın *Network access* ayarında vendor domain'leri izin listesine eklenir: `api.anthropic.com`, `generativelanguage.googleapis.com`, `www.googleapis.com`, `api.apify.com`, vendor API host'u. Bu container'dan YouTube'a erişim testlerde çalıştı.

---

## 9. Kullanıcıya açık sorular (07 §7'den)

1. Mimari seçenek A ve yol haritası onaylanıyor mu?
2. Evde sürekli açık bir makine (residential IP) var mı? Varsa ücretsiz altyazı ve video indirme mümkün olur, Gemini URL'nin de yedeği olur.
3. "Sayı" ne demek: son N video mu, en çok izlenen N video mu? Shorts ve canlı yayınlar dahil mi? Süre üst sınırı olsun mu?
4. Pilot kanallar hangileri? İdeali biri TR, biri EN ve en az biri kod/repo ağırlıklı.
5. Wiki ve notlar hangi dilde olsun? Kullanıcı notları Obsidian'da kendisi de okuyacak mı?
