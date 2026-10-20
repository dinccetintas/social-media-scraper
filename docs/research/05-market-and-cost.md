# 05 — Piyasa Taraması ve Maliyet Modeli (Persona 5: Build-vs-Buy & Maliyet Analisti)

> Tarih: 2026-10-20. Her fiyat `[Kn]` etiketiyle §8'deki kaynağa bağlı. **"doğrulanamadı"** = resmi sayfada teyit edilemedi (üçüncü taraf kaynak ya da hiç kaynak yok). Tüm $ = USD, vergiler hariç. Hesap betiği: scratchpad `p5/cost.py` (repo dışı).

---

## 1) TL;DR — net öneri

1. **H6 doğrulandı, bir düzeltmeyle:** YouTube, Instagram, görsel analiz, özel ontoloji ve uzak MCP'yi birlikte karşılayan bir SaaS yok. Bu ihtiyaca en yakın "Buy" yolları ayda **$50–140** tutuyor ve veri edinimini yine bize bırakıyor. "Assemble" yolu (B) 15 dk'lık **100 video için ≈ $1.7–3.3**, sürekli kullanımda (10 kanal, haftada ~20 video) **≈ $3–8/ay**. Düzeltme şu: **edinim katmanını satın al, KB ve MCP'yi kendin kur.** Transkript ve IG listeleme için Supadata, Apify ya da ScrapeCreators gibi değiştirilebilir küçük API'ler kullanılmalı. Kendi scraper'ımızı ve residential proxy'mizi bakımda tutmaya değmez.
2. **"Ucuz-iyi" stack:** Önce caption alınır (Supadata, evden çalışılıyorsa youtube-transcript-api). Caption yoksa **Gemini 3.1 Flash-Lite'a YouTube URL'si verilir** ve transkript ile görsel notlar **tek çağrıda** çıkar (≈ $0.039/video). Ayrı görsel geçiş yalnızca "ekran içerikli" videolarda (~%40) çalışır. Video başına zenginleştirme **Claude Haiku 5.5 + Batch** ile yapılır (≈ $0.0009/video). Kanal sentezi **Sonnet 5.5 Batch** ile, embedding Gemini Embedding 2 ile, depolama ve MCP SQLite/pgvector ile Cloudflare Workers üzerinde.
3. **Ana maliyet kalemi görsel analiz.** Ucuz stack'te toplamın ~%60–80'i buradan geliyor. Claude 5 nesliyle LLM zenginleştirme neredeyse bedavaya düştü (Haiku 5.5 $0.10/$0.50 [K1]). Asıl kalite ayarı şu: video başı zenginleştirmeyi Sonnet 5.5'e çıkarmak 100 video başına yalnızca **+$1.7** ekliyor. 5 videoluk bir A/B eval ile karar verin.
4. **Fiyat riski somut:** Gemini 3.6/3.8 Flash fiyatı **1 Ocak 2027'de iki katına çıkıyor** [K2]. Bu yüzden varsayılan model 3.1 Flash-Lite olmalı. YouTube URL özelliği "preview, ücretsiz, değişebilir" durumda [K3]. Graphlit'in yeni kayıtları kapattığı ve kapanma sürecinde olduğu görüldü [K19]; satıcıya kilitlenme riskinin canlı bir örneği.
5. **vidIQ ve Higgsfield MCP pipeline'a değil, ad-hoc kullanıma uygun.** Kredi havuzları diğer özelliklerle paylaşımlı ve vidIQ top-up satmıyor [K14]. Bu araçları Claude oturumunda tek tük keşif ve QA için kullanın; toplu ingest için kullanmayın.

---

## 2) Piyasa taraması

Uygunluk puanı (1–5): YT+IG kanal ingest, görsel, özel KB, Claude'a uzak MCP, TR içerik ve düşük maliyet ölçütlerine göre verildi. "Bileşen" = yalnızca bir pipeline katmanı olarak uygun.

| Ürün | Ne yapar | YT / IG | Transkript / Görsel | API / MCP | Fiyat (2026) | Lock-in | Uygunluk |
|---|---|---|---|---|---|---|---|
| **NotebookLM (tüketici)** | Kaynaklar üzerinde sohbet, podcast | YT URL ✓ (yalnızca caption'lı, public) / IG ✗ | Caption / ✗ | Resmî API yok; resmî MCP yok | Std: 100 notebook × 50 kaynak; Plus 100; Pro 300; Ultra 500–600 kaynak [K16] | Yüksek (export zayıf) | **2** |
| NotebookLM Enterprise API | `notebooks.sources.batchCreate`, `videoContent.youtubeUrl` | YT ✓ / IG ✗ | Caption / ✗ | REST **v1alpha**, MCP yok | ~$9/lisans/ay, 15 lisans minimum (**doğrulanamadı**) [K16] | Yüksek | 2 |
| Resmî olmayan NotebookLM MCP'leri (PleasePrompto vb.) | Claude'dan NotebookLM'e soru sorma | NotebookLM'e bağlı | NotebookLM'e bağlı | Yerel MCP (cookie ile oturum) | Ücretsiz [K16] | Kırılgan, ToS gri | 2 |
| **Supermemory** | Bellek ve RAG API'si | YT ingest **doğrulanamadı** | "Rich content" (video dahil) | API + plugin; MCP planlarda yalnız "Enterprise MCP" olarak geçiyor | Free $5 kredi; Pro $19 (+$20 kullanım); SuperRAG $0.002/1K token (rich) [K17] | Orta | 3 (KB backend) |
| **Ragie** | Yönetilen RAG, audio/video ingest, zaman damgalı chunk | Dosya ✓; YT URL **doğrulanamadı** / IG ✗ | ✓ / video modu ✓ | API + MCP sayfası var | Dev ücretsiz (1K sayfa, 10 saat işleme); Starter $100/ay; audio $0.0067/dk, video $0.025/dk [K18] | Orta-yüksek | 3 |
| Graphlit | Yönetilen içerik ve RAG + MCP | Video ✓ | ✓ | MCP (Hobby+) | Hobby $49 + $0.10/kredi; **yeni kayıt kapalı, kapanma sürecinde** [K19] | — | **1** |
| **Pinecone Assistant** | Yönetilen RAG; her asistanın kendi uzak MCP'si var | Dosya yükleme / ✗ | Metin / ✗ | ✓ Remote MCP (streamable HTTP) [K20] | Starter ücretsiz (1 GB, 500K input token/ay); Standard min **$50/ay**; context $5/M token [K20] | Orta | 3 (KB+MCP) |
| Vectara | Kurumsal RAG | ✗ | ✗ | API | SaaS **$100K/yıl'dan başlıyor** [K21] | Yüksek | 1 |
| **Twelve Labs** | Video-native arama ve analiz (Marengo/Pegasus) | Dosya / ✗ | ✓ / ✓✓ | API; MCP **doğrulanamadı** | Plus $29/ay; Pegasus $0.0294/dk [K22] | Orta | 2 (görsel bileşen) |
| VideoDB | Video DB: konuşma ve sahne indeksi | YT URL ✓ (SDK) / ✗ | ✓ / ✓ (sahne indeksi beta) | Director MCP ✓ | Free $20 tek seferlik; Pro $20/ay (Capterra, **doğrulanamadı**) [K23] | Orta | 3 |
| **Supadata** | Transkript API'si (YT, TikTok, IG, X, FB) + AI fallback | ✓ / ✓ | ✓ (AI STT fallback) / ✗ | API + MCP ✓ | Free 100 kredi; Basic $5 (300); Pro $17 (3.000); 1 kredi/transkript, AI transkript 2 kredi/dk, kanal listesi 1 kredi [K8] | Düşük | **5 (bileşen)** |
| **Apify** (+MCP) | Actor pazaryeri; IG Reel Scraper | ✓ / ✓ | Caption + ücretli transkript eklentisi / ✗ | Remote MCP `mcp.apify.com` [K11] | Free $5/ay kredi; Reel Scraper $2.60/1K (free), $2.30/1K (Starter) [K9][K10] | Düşük | **4 (bileşen)** |
| Bright Data (+MCP) | Scraper API'leri + Web MCP | ✓ / ✓ | — | MCP ✓ (5.000 istek/ay ücretsiz) [K13] | Scraper API "$1/1K kayıttan başlıyor" [K13]; IG fiyatı **doğrulanamadı** | Orta | 3 |
| **ScrapeCreators** | Sosyal medya API'leri (IG 20, YT 17 endpoint) | ✓ / ✓ | IG transkripti yalnız <2 dk videolar (3. taraf) [K12] | MCP `api.scrapecreators.com/mcp` | $47 = 25K kredi ($1.88/1K); krediler **süresiz**; 1 kredi ≈ 1 istek [K12] | Düşük | 4 (bileşen) |
| **vidIQ MCP** (bağlı) | Kanal videoları, transkript, IG reels, video "watch" | ✓ / ✓ | ✓ / ✓ (watch) | Yalnız MCP (Claude oturumu içinden) | Transkript **5**, Video Watch **25**, short-form **10** kredi [K14]; Free 150 kr/ay, Boost 2.000, Max 6.000 [K14]; Boost $19 / Max $49 (3. taraf, çelişkili) | Orta; top-up yok | 3 (ad-hoc) |
| Higgsfield MCP (bağlı) | Üretim platformu, yan özellik olarak `video_analysis` | URL/medya | — / ✓ | MCP | Basic $9/120 kr, Pro $29/600, Max $79/1.800 (3. taraf) [K15]; analiz başına kredi **doğrulanamadı** | Yüksek | 1 |
| Recall (recall.it) | Kişisel KB, özet | YT ✓ / IG ✗ | Özet / ✗ | **API + MCP Free planda** [K24] | Free (10 özet/ay); Plus $10/ay; Max $38/ay [K24] | Orta | 2–3 |
| Readwise Reader | Okuma ve highlight; YT transkripti | YT ✓ / ✗ | ✓ / ✗ | Resmî remote MCP `mcp2.readwise.io` [K25] | $9.99/ay (yıllık), $12.99/ay (aylık) [K25] | Orta | 2–3 |
| Glasp | YT özeti, highlight | YT ✓ / ✗ | ✓ / ✗ | MCP connector ✓ | Pro $10, Unlimited $25 (3. taraf, 2025) [K26] | Orta | 2 |
| Eightify | Chrome özet eklentisi | YT ✓ / ✗ | Özet / ✗ | ✗ | ~$6.99–9.99/ay (3. taraf, çelişkili) [K26] | — | 1 |
| **Fabric** (danielmiessler) | CLI prompt kalıpları (`extract_wisdom`), `yt -y` | YT ✓ / ✗ | Caption / ✗ | REST `--serve`; MCP ✗ | MIT, ücretsiz [K27] | Yok | 3 (prompt kaynağı) |
| AnythingLLM | Yerel workspace RAG | YT kollektörü (eski kaynak) / ✗ | ✓ / ✗ | MCP **client**, server değil [K28] | MIT, desktop ücretsiz | Düşük | 2–3 |
| Open WebUI | Self-host sohbet + RAG | YT loader ✓ (`YOUTUBE_LOADER_PROXY_URL`) [K29] / ✗ | ✓ / ✗ | MCP **client** (streamable HTTP) | Ücretsiz (kendi lisansı) | Düşük | 2–3 |
| Khoj | Self-host "second brain" | ✗ / ✗ | — | MCP belirtilmemiş | AGPL-3.0; bulut fiyatı **doğrulanamadı** [K30] | Düşük | 2 |
| Morphik | Multimodal (ColPali) RAG | ? / ✗ | ✓ / görüntü ✓; video **doğrulanamadı** | MCP server ✓ (glama) | Free 200 sayfa; Pro $59/ay; $0.03/sayfa aşım [K31] | Orta | 2–3 |
| CustomGPT.ai | Kanal → chatbot | YT ✓ (auto-sync Premium'da) / ✗ | Caption / ✗ | RAG API; MCP **doğrulanamadı** | Standard $99, Premium $499/ay [K32] | Yüksek | 2 |
| Delphi (delphi.co) | "Digital mind" klonu | Video/sosyal ✓ / ? | ✓ / ✗ | API yalnız Enterprise; MCP ✗ | Builder $79, Scaler $299/ay [K33] | Yüksek | 1–2 |
| Chipp / SageTube / OneAI (2025–26) | Playlist/kanal → zaman damgalı alıntılı chatbot | YT ✓ / ✗ | Caption / ✗ | Widget odaklı | **doğrulanamadı** [K41] | Yüksek | 2 |

**Tablonun özeti:** YT ve IG'yi birlikte yalnızca **edinim** sağlayıcıları (Supadata, Apify, ScrapeCreators, Bright Data, vidIQ) kapsıyor; hiçbiri KB kurmuyor. KB+MCP sağlayıcıları (Pinecone Assistant, Ragie, Supermemory, Recall, Readwise) ise veriyi dışarıdan bekliyor. Görsel analiz sunan tek ciddi seçenek Twelve Labs, o da Gemini'nin 10–15 katı pahalı (§4.3). **Uçtan uca "Buy" seçeneği yok.**

---

## 3) Strateji karşılaştırması

| | **A — Buy (SaaS)** | **B — Assemble (önerilen)** | **C — Tam self-host/yerel** |
|---|---|---|---|
| Örnek | NotebookLM (+resmî olmayan MCP) veya Ragie/Pinecone Assistant (KB+MCP) + Supadata | Supadata/Apify + Gemini 3.1 FL + Claude Haiku/Sonnet 5.5 Batch + kendi MCP'miz (Workers/Hetzner) | yt-dlp + faster-whisper + yerel VLM (Qwen-VL sınıfı) + yerel LLM + bge-m3 + yerel MCP + Cloudflare Tunnel |
| Değere ulaşma süresi | Saatler (NotebookLM) – 1–2 gün | MVP 3–7 gün (Claude Code ile), sonra ~1–2 saat/ay bakım | 1–3 hafta; makul hız için GPU şart |
| S1 maliyeti (100 video) | $0 (NotebookLM, ama gereksinimleri karşılamıyor) / **$110–138** (Ragie Starter + video dk) / $73 (Twelve Labs, yalnız görsel) | **≈ $1.7–3.3** | ≈ $0 API + elektrik/donanım (doğrulanamadı) |
| Steady state/ay | $50–140 (Pinecone min $50; Ragie $100+; CustomGPT $99) | **≈ $3–8** | ≈ $0–2 (+ makinenin sürekli açık olması) |
| IG Reels | ✗ (edinim ayrıca kurulmalı) | ✓ | ✓ (scraping kırılgan) |
| Görsel (kod, slayt, repo) | ✗ veya pahalı | ✓ (Gemini, gating ile) | Zayıf/yavaş (yerel VLM'in OCR kalitesi düşük) |
| Ontoloji/şema kontrolü | Düşük | Tam | Tam |
| TR kalitesi | Ürüne bağlı | Yüksek (Claude/Gemini) | Orta (yerel modeller) |
| Riskler | Kapanma (Graphlit), fiyat, export | Edinim sağlayıcısı ToS, YouTube IP engeli, fiyat değişimi | Donanım, bakım, uzak erişim için ev makinesi |
| Karar | Hızlı demo için NotebookLM tamam; ürün olarak ✗ | **✓ Önerilen** | "Neredeyse bedava" varyantın parçaları buradan alınır |

---

## 4) Maliyet modeli

### 4.1 Varsayımlar
- YT videosu ortalama **D = 15 dk (900 sn)**. Transkript **≈ 4.500 token** (TR/EN karışık, zaman damgalı; ~2.000 kelime; TR tokenizasyonu EN'nin ~1.7 katı). Bu bir varsayım.
- Zenginleştirme çağrısı: **girdi 8.000** token (1.5K sistem/şema + 4.5K transkript + 1.5K görsel not + 0.5K metadata), **çıktı 2.000** token. Thinking kapalı ya da düşük effort varsayıldı; Haiku 5.5'te "disabled" yalnız effort ≤ high iken geçerli [K1].
- Görsel çıktı 1.500 token/video. Embedding girdisi: (4.5K + 2K + 1.5K) × 1.15 overlap ≈ **9.200 token/video**.
- Gemini video tokenizasyonu (1 fps): Gemini 3.x'te kare başına 70 token (low=medium=varsayılan), high'da 280 [K4]; ses 32 token/sn [K3]. 2.5 serisinde low 66, diğer ayarlarda 258 token/kare [K3].
- Kanal sentezi: video başına 1.5K'lık özetlerin map-reduce'u; 100 video → 150K girdi + 8K çıktı.
- **Gating:** ayrı görsel geçiş, ekran içeriği olan videoların ~%40'ında çalışır (transkriptte "ekranda", "kod", "GitHub", "slayt" gibi sinyaller). Bu bir varsayım; Persona 3 kalibre etmeli.
- S4: 20 video/hafta × 52/12 = **86.7 video/ay**; bunların %20'sinin caption'sız olduğu varsayıldı. Hesap muhafazakâr kalsın diye hepsi 15 dk'lık YT videosu sayıldı.

### 4.2 Formüller
- Gemini görsel/tam geçiş: `C = sn × (tok_kare × p_video + 32 × p_ses)/1e6 + çıktı_tok × p_çıktı/1e6`
- LLM: `C = (girdi × p_in + çıktı × p_out)/1e6 × (Batch ? 0.5 : 1)`. Claude Batch −50%, cache okuma 0.1× (Opus/Sonnet 5.5'te 0.05×) [K1].
- STT: `C = dakika × p_dk` (Groq'ta `saat × p_saat`). Proxy üzerinden ses indirme: `MB × $3.50/GB` (~1 MB/dk varsayımı).
- Toplam (YT): `N × [cap × c_caption + (1−cap) × c_nocap + cap × g × c_görsel + c_zengin + c_emb] + c_sentez + sabit`

### 4.3 Birim maliyet tabloları (15 dk'lık video başına)

**Görsel analiz** (Gemini'de YouTube URL kullanımı; standart tarife; Batch'in YouTube URL'lerini destekleyip desteklemediği **doğrulanamadı**):

| Seçenek | Fiyat dayanağı | $/video | $/100 |
|---|---|---|---|
| Gemini 2.5 Flash-Lite, low (66 tok/kare) | $0.10 video, $0.30 ses, $0.40 çıktı [K2] | 0.015 | 1.52 |
| **Gemini 3.1 Flash-Lite, varsayılan (70)** | $0.25 video, $0.50 ses, $1.50 çıktı [K2] | **0.032** | **3.24** |
| Gemini 3.1 Flash-Lite, high (280) | aynı | 0.080 | 7.96 |
| Gemini 3.6 Flash, varsayılan | $0.75 / $3.75 (2027'de $1.50 / $7.50) [K2] | 0.075 (2027: 0.149) | 7.45 |
| Gemini 3.6 Flash, high | aynı | 0.216 | 21.6 |
| Twelve Labs Pegasus | $0.0294/dk [K22] | 0.44 | 44 |
| Ragie video ingest | $0.025/dk [K18] | 0.375 | 37.5 |

**Caption yoksa yapılacak transkripsiyon** (+ bulut IP'sinden çalışılıyorsa proxy üzerinden ses indirme ≈ **$0.051/video**, [K36] ve varsayım):

| Seçenek | $/video | Not |
|---|---|---|
| Groq whisper-large-v3-turbo $0.04/saat [K6] | 0.010 | 10 sn minimum faturalama (3. taraf) |
| Groq whisper-large-v3 $0.111/saat (3. taraf) [K6] | 0.028 | TR'de daha isabetli |
| OpenAI gpt-4o-mini-transcribe $0.003/dk [K5] | 0.045 | |
| OpenAI gpt-4o-transcribe $0.006/dk [K5] | 0.090 | |
| Deepgram Nova-3 multilingual $0.0052/dk [K7] | 0.078 | $200 başlangıç kredisi |
| Supadata AI transkript, 2 kr/dk (Pro oranı) [K8] | 0.17 | İndirme ve proxy işi dahil |
| **Gemini 3.1 FL tam geçiş (transkript + görsel)** [K2][K3] | **0.039** | Proxy ve indirme yok; görsel analiz de aynı çağrıda çıkıyor |

**Zenginleştirme** (8K girdi / 2K çıktı):

| Model | $/M girdi/çıktı [K1][K2] | $/video std | Batch | $/100 Batch |
|---|---|---|---|---|
| **Claude Haiku 5.5** (≤100K prompt) | 0.10 / 0.50 | 0.0018 | 0.0009 | **0.09** |
| Claude Haiku 4.5 | 1 / 5 | 0.018 | 0.009 | 0.90 |
| **Claude Sonnet 5.5** | 2 / 10 | 0.036 | 0.018 | **1.80** |
| Claude Opus 5.5 | 4 / 20 | 0.072 | 0.036 | 3.60 |
| Gemini 3.6 Flash | 0.75 / 3.75 | 0.0135 | 0.0067 | 0.67 |
| Gemini 3.1 Flash-Lite | 0.25 / 1.50 | 0.005 | 0.0025 | 0.25 |

Not: Haiku 5.5'te prompt 100K token'ı aşınca fiyat **$0.50/$2.50**'ye çıkıyor [K1]. Video başına çağrıları 100K'nın altında tutun. Per-video prompt caching kazancı ihmal edilebilir (transkriptler benzersiz); asıl kazanç Batch'ten geliyor.

**Kanal sentezi (100 video):** Sonnet 5.5 $0.38 (Batch $0.19) · Opus 5.5 $0.76 · Haiku 5.5 $0.095 (>100K kademesi) [K1].
**Embedding:** OpenAI text-embedding-3-small $0.02/M → $0.018/100 video [K5]; Gemini Embedding 2 $0.20/M (Batch $0.10) → $0.18 (0.09)/100 video [K2]; bge-m3 yerelde $0. **Maliyet karar kriteri değil; TR kalitesine göre seçin.**
**Listeleme/metadata:** YouTube Data API ücretsiz, günlük 10.000 birim. `playlistItems.list` ve `videos.list` 1 birim [K34]; 1.000 videoluk bir kanal ~40 birim tutuyor. Artımlı takip için kanal RSS'i 0 kota harcar. `captions.download` yalnız kendi videolarınızda çalışır, üçüncü taraf kanallar için kullanılamaz.
**Proxy:** Webshare rotating residential en küçük paket 1 GB = $3.50; hacim arttıkça $1.40/GB'ye iner [K36]. youtube-transcript-api, bulut IP'lerinin YouTube tarafından engellendiğini belirtiyor [K35].
**Hosting:** Cloudflare Workers Paid en az $5/ay [K37]. Vectorize (Paid) ayda 10M depolanan boyut ve 50M sorgulanan boyut dahil [K37]; bu, 1.000 videonun 768 boyutlu vektörlerine yetiyor. Supabase Free: 500 MB DB, ama 1 hafta hareketsiz kalınca duraklatılıyor [K38]. Hetzner €5.99/ay'dan başlıyor [K39].
**IG listeleme (1K başına):** Apify Reel Scraper $2.30–2.60 [K10] · ScrapeCreators $1.88 (Freelance) / $0.99 (Business) [K12] · Bright Data ~$1–1.5 (doğrulanamadı) [K13] · IG transkript actor'ları: TubeText $0.006/reel, Firsthand $6/1K [K40].

### 4.4 Senaryolar (3 stack × S1–S4)

**Stack tanımları**
- **Ucuz-iyi:** Caption için Supadata (Pro oranı $0.0057/kredi; S1 Free planın 100 kredisine sığıyor). Caption yoksa Gemini 3.1 FL tam geçiş. Görsel 3.1 FL varsayılan çözünürlükte, %40 gating ile. Zenginleştirme Haiku 5.5 Batch, sentez Sonnet 5.5 Batch, embedding Gemini Emb 2 Batch.
- **Premium:** Caption için Supadata. Caption yoksa Groq whisper-large-v3 ve proxy ile ses indirme. Görsel tüm videolarda Gemini 3.6 Flash; ayrıca videoların %20'sinde kod segmentleri high çözünürlükte. Zenginleştirme Sonnet 5.5 Batch, sentez Opus 5.5, embedding Gemini Emb 2.
- **Neredeyse bedava:** Ev IP'sinden youtube-transcript-api. Caption yoksa ve görsel için Gemini **free tier** (YouTube'dan günde 8 saat sınırı [K3]; free tier verisi ürün geliştirmede kullanılıyor [K2]) ya da yerel faster-whisper. Zenginleştirme ve sentez Haiku 5.5 Batch; embedding yerel bge-m3; hosting Workers Free ya da Cloudflare Tunnel.

| Senaryo | Ucuz-iyi | Premium | Neredeyse bedava |
|---|---|---|---|
| **S1** 100 YT × 15 dk, hepsi caption'lı | **$1.67** (Supadata Free) – $2.23 (Pro oranı). Görsel $1.30 · zenginleştirme $0.09 · sentez $0.19 · embedding $0.09 · caption $0–0.57 | **$15.08** (görsel $11.77 · zenginleştirme $1.80 · Opus sentez $0.76) | **$0.14** |
| **S1-b** %50 caption'sız | **$2.98–3.26** (Gemini tam geçiş $1.96 · görsel $0.65) | **$18.75** (+STT ve proxy $3.95) | $0.14 (free tier 8 saat/gün → 100 videoluk geri dolum 2–4 gün sürer) |
| **S2** 1.000 YT video | **$22** marjinal; plan basamaklarıyla $23–33 (Supadata Pro $17 ya da Webshare 2 GB $7) | $149 (2027'de görsel ×2 → ~$267) | $1.29 (free tier'da ~4 hafta) |
| **S2-b** %50 caption'sız | **$32** marjinal (~$46 Supadata Pro planıyla) | $186 | $1.29 |
| **S3** IG 100 reel × 60 sn | **$0.67**: listeleme $0.26 (Apify'ın $5'lık free kredisine sığıyor) + Gemini 3.1 FL transkript+görsel $0.35 + Haiku $0.03 + embedding $0.03. Buluttan proxy ile indirme +$1.71 | ~$1.9 (Gemini 3.6 Flash $0.83 + Sonnet 5.5 $0.60 + Groq v3 $0.19) | ~$0.03 |
| **S4** 10 kanal, ~87 video/ay | **$3.1 değişken + $0–5 hosting = $3–8/ay**: görsel $0.90 · caption'sız $0.68 · aylık kanal profil güncellemesi (Sonnet 5.5) $1.00 · caption $0 (Free plan) | **$15.7 + $5 = ~$21/ay** (2027: ~$31) | **$0.13/ay** + $0 hosting |

**Okuma notları:** (1) Ucuz-iyi stack'te maliyet sırası görsel > caption'sız geçiş > sentez > geri kalan. Görsel gating'i %40'tan %100'e çıkarmak S1'e +$1.9 ekler. (2) Premium'un ~%80'i Gemini 3.6 Flash. Bunun yerine 3.1 FL kullanılırsa (varsayılan çözünürlük + %20 high) S1 ≈ $8'e iner. (3) Değerler "yeniden işleme yok" varsayımıyla hesaplandı; prompt/şema iterasyonu için +%20–30 tampon ayırın.

### 4.5 "Buy" karşılaştırması (aynı iş yükü)
- **Ragie:** S1 = Starter $100 + video 1.500 dk × $0.025 = **$137.5** (yalnız audio modunda $110). S4 = $100 + 1.300 dk × ($0.0067–0.025) = **$109–133/ay** [K18]. Videoyu yine biz indiriyoruz.
- **Pinecone Assistant:** Starter $0 (ayda 500K girdi + 500K context token sınırı; S1 sınırda kalır) ya da Standard en az **$50/ay** [K20]. Transkripti yine biz çıkarıyoruz.
- **Twelve Labs:** S1 = 1.500 dk × $0.0294 + Plus $29 ≈ **$73**; yalnız görsel katmanı karşılıyor [K22].
- **vidIQ (Claude içinden):** S1 = 100 × 5 (transkript) + 40 × 25 (watch) = 1.500 kredi → Boost (2.000 kredi/ay). KB yok, top-up yok [K14].
- **NotebookLM:** S1 $0, ama 50 kaynak/notebook yüzünden 2 notebook gerekiyor; IG yok, görsel yok, resmî MCP yok [K16].

**Sonuç:** B, maliyette A'dan **~10–50 kat** ucuz (S4: $3–8 karşısında $50–140). A'nın tek avantajı hızlı başlangıç.

---

## 5) Önerilen stack'ler

### 5.1 "Ucuz-iyi" (önerilen) — **≈ $1.7–3.3 / 100 video · ≈ $3–8 / ay**
1. **Listeleme:** YouTube Data API ya da `yt-dlp --flat-playlist`, artımlı takip için RSS. IG için Apify Instagram Reel Scraper (pay-per-result; aylık $5 free kredi) [K9][K10]. Yedek: ScrapeCreators (krediler süresiz) [K12].
2. **Transkript:** Pipeline bulutta çalışıyorsa Supadata (1 kredi/video; Free 100/ay, Pro $17/3.000) [K8]. Ev makinesinde çalışıyorsa youtube-transcript-api ücretsiz [K35].
3. **Caption yoksa ve tüm IG reel'leri için:** Gemini 3.1 Flash-Lite **tam geçiş**: YT'de URL ile, IG'de Files API ile. Transkript ve görsel notlar tek JSON'da döner. Video başına $0.039, reel başına $0.0035 [K2][K3].
4. **Görsel:** Gating'den geçen videolarda 3.1 FL varsayılan çözünürlük ($0.032/video). Kod yoğun segmentler `start/end_offset` ile kırpılıp high çözünürlükte işlenir.
5. **Zenginleştirme:** Claude **Haiku 5.5 + Batch** ($0.0009/video). Kanal playbook'u ve sentez için **Sonnet 5.5 Batch** ($0.19/100 video) [K1]. Bütçe izin veriyorsa ilk ayarlanacak şey per-video zenginleştirmeyi Sonnet 5.5'e çıkarmak (+$1.71/100).
6. **Embedding/depolama:** Gemini Embedding 2 (Batch) ya da bge-m3; SQLite + sqlite-vec veya Postgres/pgvector.
7. **MCP:** Cloudflare Workers (Free ya da $5) üzerinde remote MCP. Sorgu anında sunucu tarafında LLM çağrısı yapılmaz; chunk'lar döndürülür ve muhakemeyi ajan yapar. Sorgu başına maliyet ≈ $0.

### 5.2 Premium — **≈ $15–19 / 100 video · ≈ $21/ay (2027'de ~$31)**
Gemini 3.6 Flash görsel analizi tüm videolarda çalışır, ayrıca segmentlerin %20'si high çözünürlükte. Caption'sız videolarda Groq whisper-large-v3 (TR isabeti) ya da gpt-4o-transcribe. Zenginleştirme Sonnet 5.5 Batch, kanal sentezi Opus 5.5. Embedding Gemini Embedding 2 + reranker. Hosting Hetzner €5.99 (DB + MCP + kuyruk tek kutuda).

### 5.3 Neredeyse bedava — **≈ $0.14 / 100 video · <$1/ay**
Ev makinesinde youtube-transcript-api ve yt-dlp çalışır. Caption'sız videolar için yerel faster-whisper (turbo) ya da Gemini free tier. Görsel analiz Gemini free tier'da, yalnız gating'den geçenlere uygulanır (günde 8 saat YT). Zenginleştirme Haiku 5.5 Batch (≈ $0.09/100 video; tek ücretli kalem). Embedding bge-m3, MCP için Cloudflare Tunnel. **Bedeli:** free tier verisi Google ürünlerini geliştirmede kullanılıyor [K2], kotalar düşük, makinenin açık kalması gerekiyor ve görsel kuyruğu günlerce sürebilir.

---

## 6) Riskler

| Risk | Kanıt | Etki | Önlem |
|---|---|---|---|
| **Fiyat değişikliği** | Gemini 3.6/3.8 Flash 1 Ocak 2027'de 2× [K2]; YouTube URL "preview, ücretsiz, fiyat değişebilir" [K3]; Bright Data 2026'da iki kez zam yaptı (3. taraf) [K13] | Premium görsel maliyeti ×2 | Varsayılan 3.1 FL; model adı config'te; aylık maliyet raporu |
| **Satıcının kapanması/kilitlenme** | Graphlit yeni kayıtları kapattı [K19]; vidIQ MCP'nin "launch süresince ücretsiz" olduğu söyleniyor (3. taraf) [K14] | Pipeline kesintisi | Sağlayıcı arayüzü (`TranscriptProvider`, `IgLister`); KB verisi bizde (SQLite/Postgres); her sağlayıcıya en az 1 yedek |
| **ToS: YouTube** | Bulut IP'leri engelleniyor, residential proxy gerekiyor [K35]; resmî caption API'si yalnız kendi videoların için [K34] | Engellenme, gri alan | Kişisel ve özel kullanım; resmî API ile metadata; edinimi satın alınan API'ye devretmek |
| **ToS: Instagram/Meta** | Otomatik veri toplamayı yasaklıyor; IG transkript endpoint'leri <2 dk ile sınırlı [K12] | Actor/endpoint aniden bozulabilir | İki sağlayıcı (Apify + ScrapeCreators); oturum açmadan yalnız public veri; içerik yeniden yayımlanmaz |
| **Kredi havuzu tükenmesi** | vidIQ top-up satmıyor; MCP kredileri diğer özelliklerle paylaşımlı [K14] | Ajan iş ortasında durur | vidIQ ve Higgsfield'ı pipeline'ın kritik yoluna koymayın |
| **Gizlilik** | Gemini free tier verisi eğitimde kullanılıyor [K2] | İçerik public olduğu için düşük | Paid tier'da "not used" [K2] |
| **Kalite (TR)** | Gemini transkriptinin zaman damgaları Whisper kadar hassas değil (varsayım) | Alıntı ve zaman bağlantıları kayabilir | Caption varken caption'ı esas alın; 5 videoluk WER ve özet eval'i |

---

## 7) Diğer personalara notlar

- **P1 (KB/RAG):** Embedding ve depolama maliyeti ihmal edilebilir (<$0.2/100 video). Seçimi TR retrieval kalitesine göre yapın. 1.000 video yaklaşık 10K chunk eder; Vectorize/pgvector'ün free ve paid dahil kotalarına sığar [K37][K38]. Sorgu anında LLM çağırmayın, chunk ve özet döndürün. Haiku 5.5'te prompt >100K olduğunda fiyat 5 katına çıkıyor [K1].
- **P2 (edinim):** Claude Code cloud oturumları da bulut IP'si sayılır; YouTube engeli burada da geçerli [K35]. Bu yüzden bulutta Supadata ya da Webshare ($3.50/GB) kullanın, yoksa pipeline'ı ev makinesinde çalıştırın. Artımlı takip için RSS (0 kota). `captions.list` 50 birim tutuyor ve üçüncü taraf videolarında işe yaramıyor [K34]. IG: Apify Reel Scraper $2.30–2.60/1K, video indirme eklentisi MB başına ücretli [K10].
- **P3 (görsel):** YouTube URL yalnız **public** videolarda çalışıyor; free tier'da günde 8 saat sınırı, paid tier'da süre limiti yok; istek başına 10 video [K3]. Gemini 3.x'te low, medium ve varsayılan aynı (70 tok/kare); high 280 [K4]. Kırpma ve özel fps yalnız "static" modda [K3]. Bütçe çapası: $0.032 (3.1 FL varsayılan) – $0.080 (high) /15 dk. Gating eşiğini kalibre edin.
- **P4 (MCP):** Remote MCP hosting $0–5/ay. Hazır bir alternatif istenirse Pinecone Assistant asistan başına remote MCP sunuyor (Standard en az $50) [K20]. vidIQ MCP tool'ları (transkript 5 kr, watch 25 kr) ajanlar için "canlı keşif" aracı olarak sunulabilir.
- **P6 (ontoloji):** Sonnet/Opus'ta çıktı token'ı girdinin 5 katı fiyatlı [K1]. Şemayı kompakt tutun; modeldeki 2K çıktı/video varsayımını aşan şemalar maliyeti doğrusal artırır. Kanal düzeyindeki "playbook" alanlarını per-video'da değil, sentez aşamasında üretin.

---

## 8) Kaynaklar
- [K1] Claude pricing — https://platform.claude.com/docs/en/about-claude/pricing
- [K2] Gemini API pricing — https://ai.google.dev/gemini-api/docs/pricing
- [K3] Gemini video understanding — https://ai.google.dev/gemini-api/docs/video-understanding
- [K4] Gemini media resolution — https://ai.google.dev/gemini-api/docs/media-resolution
- [K5] OpenAI pricing — https://developers.openai.com/api/docs/pricing
- [K6] Groq whisper-large-v3-turbo — https://console.groq.com/docs/model/whisper-large-v3-turbo ; large-v3 ve 10 sn minimum (3. taraf) — https://www.morphllm.com/groq-pricing
- [K7] Deepgram pricing — https://deepgram.com/pricing
- [K8] Supadata — https://supadata.ai/pricing ; https://supadata.ai/
- [K9] Apify pricing — https://apify.com/pricing
- [K10] Apify Instagram Reel Scraper — https://apify.com/apify/instagram-reel-scraper
- [K11] Apify MCP — https://docs.apify.com/platform/integrations/mcp
- [K12] ScrapeCreators — https://scrapecreators.com/ ; IG transkript <2 dk (3. taraf) — https://www.socialcrawl.dev/blog/best-instagram-data-apis-2026
- [K13] Bright Data MCP — https://brightdata.com/ai/mcp-server ; scraper fiyatı — https://brightdata.com/blog/web-data/best-youtube-scrapers ; zamlar (3. taraf) — https://costbench.com/changelog/bright-data-price-increase-2026-08/
- [K14] vidIQ MCP — https://support.vidiq.com/en/articles/15082430-vidiq-mcp ; plan kredileri — https://support.vidiq.com/en/articles/14979673 ; fiyat (3. taraf) — https://outlierkit.com/resources/vidiq-mcp/
- [K15] Higgsfield (3. taraf) — https://blog.segmind.com/higgsfield-credits-explained-expiry-rules-and-real-cost/
- [K16] NotebookLM limitleri — https://support.google.com/notebooklm/answer/16213268 ; Enterprise API — https://docs.cloud.google.com/gemini/enterprise/notebooklm-enterprise/docs/api-notebooks-sources ; Enterprise fiyatı (3. taraf) — https://claw.aguidetocloud.com/google/notebooklm/enterprise/ ; resmî olmayan MCP — https://claudemarketplaces.com/mcp/pleaseprompto/notebooklm-mcp
- [K17] Supermemory — https://supermemory.ai/pricing
- [K18] Ragie — https://www.ragie.ai/pricing
- [K19] Graphlit — https://www.graphlit.com/pricing
- [K20] Pinecone — https://www.pinecone.io/pricing/ ; Assistant MCP — https://docs.pinecone.io/guides/assistant/mcp-server
- [K21] Vectara — https://www.vectara.com/pricing
- [K22] Twelve Labs — https://www.twelvelabs.io/pricing ; https://www.twelvelabs.io/pricing-calculator
- [K23] VideoDB — https://videodb.io/pricing ; https://www.capterra.com.sg/software/1096187/VideoDB ; https://playbooks.com/mcp/videodb-director
- [K24] Recall — https://www.recall.it/pricing
- [K25] Readwise — https://readwise.io/pricing/reader ; MCP — https://docs.readwise.io/tools/mcp
- [K26] Glasp (3. taraf) — https://zoftwarehub.com/products/glasp/pricing ; Eightify (3. taraf) — https://creatoreconomytools.com/tool/eightify
- [K27] Fabric — https://github.com/danielmiessler/Fabric
- [K28] AnythingLLM MCP — https://docs.anythingllm.com/mcp-compatibility/overview
- [K29] Open WebUI — https://docs.openwebui.com/features/ ; https://raw.githubusercontent.com/open-webui/open-webui/main/backend/open_webui/config.py
- [K30] Khoj — https://github.com/khoj-ai/khoj
- [K31] Morphik — https://dev.morphik.ai/pricing
- [K32] CustomGPT.ai — https://customgpt.ai/pricing/
- [K33] Delphi — https://www.delphi.co/pricing
- [K34] YouTube Data API kotası — https://developers.google.com/youtube/v3/determine_quota_cost
- [K35] youtube-transcript-api — https://github.com/jdepoix/youtube-transcript-api
- [K36] Webshare — https://www.webshare.io/pricing
- [K37] Cloudflare Workers — https://developers.cloudflare.com/workers/platform/pricing/ ; Vectorize — https://developers.cloudflare.com/vectorize/platform/pricing/
- [K38] Supabase — https://supabase.com/pricing
- [K39] Hetzner Cloud — https://www.hetzner.com/cloud
- [K40] Apify IG transkript actor'ları — https://apify.com/tubetext/instagram-reels-transcript ; https://apify.com/firsthand/instagram-reel-transcripts
- [K41] Yeni "kanal → chatbot" SaaS'ları — https://chipp.ai/integrations/youtube ; https://www.producthunt.com/products/sagetube/makers ; https://customgpt.ai/youtube-videos-into-an-ai-assistant/
