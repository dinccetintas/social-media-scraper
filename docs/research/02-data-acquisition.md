# 02 — Veri Edinimi (Data Acquisition) Araştırması

**Persona 2 — Data Acquisition Engineer** · Tarih: 2026-10-20 · Fizibilite testleri bu Claude Code cloud container'ında (datacenter IP, HTTPS proxy arkası) koşuldu. Test log'u ve ham çıktılar: `/tmp/claude-0/-home-user-social-media-scraper/858654e1-d422-5d9f-99a5-b9b8ddb87133/scratchpad/p2/` (repo'ya hiçbir veri commit'lenmedi).

---

## 1) TL;DR: önerim

1. **H1 doğrulandı, ama bir nüansla.** Datacenter IP'den `youtube-transcript-api` → `IpBlocked`, `yt-dlp`'nin varsayılan client'ları → *"Sign in to confirm you're not a bot"* hatası verdi. Buna karşın **listeleme (`--flat-playlist`), RSS ve yt-dlp'nin `web_embedded` player client'ıyla altyazı indirme çalıştı.** Türkçe manuel altyazı ve `tr-orig` otomatik altyazı bu yolla indi. Bu yol ücretsiz ama kırılgan: resmî değil ve yalnızca embed'e izin veren videolarda çalışıyor. **Ses ve video indirme cloud'da tamamen bloklu** (SABR, PO token, bot kontrolü).
2. **YouTube zinciri.** Keşif için RSS (artımlı) + Data API v3 uploads playlist (tam liste, resmî, bedava) kullanılır. Transcript için öncelik **laptop'ta `youtube-transcript-api`/yt-dlp**, **cloud'da önce `web_embedded`, o olmazsa ScrapeCreators** (1 kredi ≈ $0.0019; altyazı yoksa ücret alınmıyor). Altyazısız videolarda Groq Whisper devreye girer (laptop) ya da Gemini'ye doğrudan YouTube URL'i verilir (cloud, preview, şu an ücretsiz). **100 video: laptop ≈ $0–0.15, cloud ≈ $0.2–0.6.**
3. **H3 doğrulandı.** Resmî Graph API (Business Discovery) başka hesapların medyasını okuyabiliyor, ama kullanıcının **kendi IG profesyonel hesabı + FB Page + Meta app** sahibi olmasını şart koşuyor. Telifli müzik içeren ya da indirmesi kapatılmış reel'lerde `media_url` dönmüyor. Anonim Instaloader cloud'da **302 → login** yönlendirmesiyle düştü.
4. **Instagram zinciri.** Ana yol **Apify `apify/instagram-reel-scraper`** (caption + mp4 URL + timestamp tek kayıtta, $2.3–2.6/1k, aylık $5 ücretsiz kredi) ve ardından ffmpeg + **Groq whisper-large-v3** ile STT. Fallback **ScrapeCreators**: reels + post + AI transcript; YouTube fallback'iyle aynı vendor, krediler süresiz. Ucuz alternatif **HikerAPI** (~$0.001/istek). **100 reel ≈ $0.15–0.40.**
5. **STT.** Varsayılan **Groq whisper-large-v3** ($0.111/saat, word/segment timestamp, URL input). İngilizce için daha ucuz turbo modeli ($0.04/saat). Türkçe kalite kritikse **ElevenLabs Scribe v2** ($0.22/saat). `gpt-4o-mini-transcribe` ucuz ($0.003/dk) ama **timestamp vermiyor**, bu yüzden KB derin linkleri için uygun değil.
6. **vidIQ MCP ingestion kaynağı olarak elverişli değil.** Bakiye **0 kredi** (Free plan, 150/ay, 2026-11-06'da yenileniyor). Test çağrısı "Not enough credits… No credits were charged" ile reddedildi. Transcript çağrısı 5 kredi, IG reels en fazla 12 kayıt ve sayfalama yok. Ad-hoc agent aracı olarak kalabilir.

---

## 2) Değerlendirilen seçenekler

### 2.1 YouTube: keşif ve metadata

| Yöntem | Kapsam | Maliyet / limit | Datacenter IP | Not |
|---|---|---|---|---|
| `yt-dlp --flat-playlist` `/videos`, `/shorts`, `/streams` | Tüm liste: id, title, duration, view_count | Ücretsiz | **✔ test** (856 ID / 11 sn) | Tarih yok; `youtubetab:approximate_date` yaklaşık tarih veriyor. **Başlıklar otomatik İngilizceye çevriliyor**, `youtube:lang=tr` ile düzeliyor (test) |
| Data API v3: uploads playlist (`UU…`) `playlistItems.list` + `videos.list` (50 ID/çağrı) | Tüm upload'lar. snippet (title, description, tags, publishedAt, defaultAudioLanguage), contentDetails, statistics | Her biri 1 unit; günlük 10.000 unit [1]. 1000 video ≈ 40 unit | ✔ (API key gerekli; erişim testi 403 "unregistered caller") | En stabil ve resmî yol. **Chapters yok**, description'daki zaman damgalarından parse edilmeli |
| RSS `feeds/videos.xml?channel_id=` | **Son 15** öğe; tam description, views, published | Ücretsiz, anahtar yok | **✔ test** | `playlist_id=UULF…` yalnız long-form, `UUSH…` yalnız Shorts döndürüyor (test, ikisi de 15 öğe; resmî değil). 2025 sonu kesinti iddiaları var [37] |
| Supadata `youtube/channel/videos` | ≤5000 ID; videoIds/shortIds/liveIds | 1 kredi/liste [8][9] | ✔ | |
| ScrapeCreators `channel-videos` | Sayfalı | 1 kredi/istek [10] | ✔ | |
| Captions API (resmî) | — | `captions.list` 50 unit [1]; `captions.download` 200 unit, **videoyu düzenleme yetkisi şart** [2] | — | Başkasının kanalı için **kullanılamaz** |

**Toplanacak metadata:** title (orijinal dilde), description (GitHub/repo linkleri için regex), chapters (`info.json` ya da description parse), publishedAt, duration, views/likes/comments, tags, `defaultAudioLanguage` (tr/en yönlendirmesi için).
**Yorumlar:** `commentThreads.list` 1 unit [1], `order=relevance` ile ilk 10–20 yorum ve kanal sahibinin yorumları. Ucuz ve faydalı (kaynak linkleri genelde pinned yorumda), ama P2 (öncelik 2). Data API'de "pinned" alanı var mı doğrulanamadı; yt-dlp comment çıktısında `is_pinned` var.

### 2.2 YouTube: transcript

| Seçenek | Fiyat | Datacenter IP | TR | Timestamp | MCP | Not |
|---|---|---|---|---|---|---|
| `youtube-transcript-api` 1.2.4 | Ücretsiz | **✘ IpBlocked** (test) | ✔ | segment | — | README cloud IP'lerin bloklandığını söylüyor; Webshare residential entegrasyonu var. Cookie auth "şu an bozuk" [3] |
| yt-dlp subs (varsayılan client'lar) | Ücretsiz | **✘** bot kontrolü (test) | ✔ | json3/vtt | — | Önerilen: JS runtime (deno ≥2.3) [5] + PO token plugin (bgutil) [4] |
| yt-dlp `player_client=web_embedded` | Ücretsiz | **✔ (test)**; çeviri altyazısı 429 | ✔ `tr`, `tr-orig` | json3 | — | PO token istemiyor ama **yalnız embed'lenebilir videolar** [4]. Kırılgan |
| yt-dlp + residential proxy | Webshare $3.50/GB (1 GB plan) → $1.40/GB [7] | ✔ (beklenen, test edilmedi) | ✔ | ✔ | — | Video başına ~1–2 MB, yani 100 video ≈ $0.5. Bakım yükü getiriyor |
| **ScrapeCreators** | $47 → 25k kredi ($1.88/1k), $497 → 500k ($0.99/1k), **krediler süresiz**, 100 ücretsiz [10] | ✔ | `language`, `original_audio` | startMs/endMs | ✔ | Canlı istek 1 kredi; **altyazı yoksa 0 kredi**; cache 0 kredi [11] |
| **Supadata** | Free 100 kredi/ay; Basic $5/ay (yıllık); Pro $17/3000; Mega $47/30k; native 1 kredi, **AI 2 kredi/dk**; kredi devretmiyor [8] | ✔ | `lang` | offset/duration (ms) | ✔ remote + `npx @supadata/mcp` [9] | `mode=native|generate|auto`. Büyük videoda async job [9]. IG/TikTok/X de destekleniyor. Rakip vendor'a göre "transcript unavailable" yanıtı da 1 kredi [36] |
| TranscriptAPI | $5/ay → 1000 kredi; top-up $2.5/1k (yıllıkta $1.5/1k) [14] | ✔ | doğrulanamadı | doğrulanamadı | ✔ | Kanal listeleme 1 kredi/sayfa; RSS takibi ücretsiz [14] |
| SearchAPI.io `youtube_transcripts` | $40/ay → 10k istek ($4/1k) [15] | ✔ | `lang`, `transcript_type=auto|manual` | start/duration | doğrulanamadı | Sadece transcript için pahalı |
| Apify actor'ları | $2–10/1k. Örn. pintostudio $10/1k, %80.8 başarı [16] | ✔ | değişken | değişken | Apify MCP | Kalite actor'a göre değişiyor |
| vidIQ MCP | 5 kredi/çağrı. Free 150/ay; Boost 2–3k; Max 6–10k kredi [32] | ✔ (MCP) | `language` | doğrulanamadı | ✔ (kendisi) | Bakiye 0. Ayrı bir REST API'si yok |

**2025–2026 anti-bot durumu:** YouTube PO token zorunluluğunu client bazında yayıyor. `mweb`/`tv_simply`/`web_safari` GVS token'ı, `web` ise altyazı token'ı istiyor; `tv`, `web_embedded` ve `android_vr` istemiyor. Token artık video ID'sine bağlı [4]. yt-dlp, JS challenge çözümü için harici runtime (EJS: deno/node) istiyor [5]. Yönergeler istekler arasında 5–10 sn bekleme öneriyor ve guest için ~300 video/saat, hesapla ~2000 video/saat sınırı veriyor. "Hesabınızla kullanmak ban riski taşır" uyarısı da var [6]. **Testim: deno kuruluyken bile `web_embedded`'de formatlar "SABR-only" diye atlandı, varsayılan client'larda bot kontrolü çıktı. Yani cloud'dan ses indirilemiyor.**

### 2.3 Speech-to-text fallback

Varsayım: YouTube video ort. 15 dk, reel ort. 45 sn.

| Model | $/saat | Türkçe | Timestamp | Dosya limiti | Not |
|---|---|---|---|---|---|
| **Groq whisper-large-v3** | **$0.111** [19] | ✔ (vendor tablosunda FLEURS-TR WER %7.5 [23]) | segment + word (`verbose_json`) [19] | 25 MB free / 100 MB dev; `url` input [19] | Min. 10 sn faturalama [19] |
| Groq whisper-large-v3-turbo | **$0.04** [19] | ✔ (v3'ten biraz kötü; doğrulanamadı) | aynı | aynı | EN için ideal |
| OpenAI gpt-4o-mini-transcribe | $0.18 ($0.003/dk) [20] | ✔ | **✘** (timestamp yalnız whisper-1) [20] | 25 MB [20] | gpt-transcribe $0.0045/dk, whisper-1/gpt-4o-transcribe $0.006/dk [20] |
| Deepgram Nova-3 (monolingual `tr`) | $0.258 ($0.0043/dk) [21] | ✔ yalnız mono; `multi` TR içermiyor [21] | word | — | $200 ücretsiz kredi [21] |
| AssemblyAI Universal-2 | $0.15 (U-3.5 Pro $0.21) [22] | doğrulanamadı | word | — | $50 ücretsiz [22] |
| ElevenLabs Scribe v2 | $0.22 [23] | ✔ (vendor iddiası: FLEURS-TR WER %3.8 [23]) | doğrulanamadı (sayfada yalnız Realtime için yazıyor) | — | En iyi TR kalitesi iddiası |
| Gemini 2.5 Flash-Lite (audio) | ≈$0.035 input (32 token/sn × $0.30/1M) + output [24] | ✔ | prompt'la MM:SS, hassas değil | 9.5 saat/prompt, 20 MB inline [24] | **YouTube URL'ini doğrudan alıyor**: preview, şu an ücretsiz, yalnız public videolar, free tier'da 8 saat/gün [24] |

### 2.4 Instagram

| Seçenek | Ne dönüyor | Fiyat | "Tüm reels" | Risk | Datacenter IP |
|---|---|---|---|---|---|
| **Graph API Business Discovery** (Facebook Login) | Başka **Business/Creator** hesapların medyası: caption, media_type, `media_url`, permalink, timestamp, like/comment, `view_count` [25] | Ücretsiz. Kendi IG profesyonel hesabı + FB Page + Meta app; izinler `instagram_basic`, `instagram_manage_insights`, `pages_read_engagement` [25]. Rate limit ikincil kaynaklara göre 200 × kullanıcı/saat (doğrulanamadı) [26] | Cursor sayfalama; geriye ne kadar gittiği doğrulanamadı | Resmî. **Telifli/kütüphane müziği olan videolarda ve indirmesi kapalı reel'lerde `media_url` yok**; age-gated hesaplar dönmüyor [25] | ✔ |
| Instaloader 4.15.4 | Her şey | Ücretsiz | ✔ | Anonim erişim cloud/VPN IP'lerinde "çok daha sıkı" limitli [27]. **Test: 302 → login**. Login checkpoint/ban riski taşıyor | **✘** |
| instagrapi | Private API | Ücretsiz | ✔ | README: "testing, research…" için uygun, prod'da kırılgan; HikerAPI'yi öneriyor [28] | Proxy ister |
| **Apify `instagram-reel-scraper`** | caption, **mp4 URL**, timestamp, views, duration, hashtags. Opsiyonel transcript (dk başına add-on, fiyatı listelenmemiş) [18] | Free plan $2.60/1k, Starter $2.30/1k, "from $1.00/1k" [18]. **Her ay $5 ücretsiz kredi** [17] | `resultsLimit` | Riski vendor taşıyor; logged-out görünüm [18] | ✔ |
| **ScrapeCreators** | `user/reels`: `video_versions` mp4, play_count, taken_at. **caption yok** → `/v1/instagram/post`; pinned reel'ler hariç [13]. `media/transcript`: AI [12] | 1 kredi/istek. Transcript ≤120 sn için 1 kredi, daha uzunda `allow_long_transcription=true` ile 10 kredi/dk [12]. $1.88/1k [10] | `max_id` sayfalama (sayfa boyu doğrulanamadı) | Vendor | ✔ |
| **HikerAPI** | `/v1/user/clips/chunk`: `video_url`, `caption_text`, `play_count`, `video_duration` [29] | $0.0006–0.001/istek; 100 ücretsiz [29] | `end_cursor` | Hosted private API; ToS gri alan | ✔ |
| Bright Data | Reels dataset: url, description, hashtags, date, likes, views. `video_url` doğrulanamadı [31] | $1.5/1k kayıt. 5000 aylık ücretsiz kredi (≈$7.5); MCP [31] | "Collect all Reels… (without the post timestamp)" [31] | Vendor | ✔ |
| EnsembleData | Reels/posts (post başına unit) | **Min. $100/ay** [30] | ✔ | Vendor | ✔ |
| vidIQ `ig_profile_reels` | ≤12 reel, caption, sayaçlar | 5 kredi (tool açıklaması) | **✘ sayfalama yok** | — | MCP |
| RapidAPI marketplace | — | doğrulanamadı | — | Kısa ömürlü sağlayıcılar | — |

---

## 3) Fizibilite test sonuçları (bu container, 2026-10-20)

Ortam: yt-dlp 2026.8.19, youtube-transcript-api 1.2.4, instaloader 4.15.4, deno 2.9.7 (pip), Python 3.13.

| # | Komut (özet) | Sonuç |
|---|---|---|
| T1 | `yt-dlp --flat-playlist --playlist-end 5 -J https://www.youtube.com/@Fireship/videos` | **OK**, 1.46 sn. id/title/duration/view_count geldi, timestamp `None` |
| T2 | Aynısı `/shorts` ve `/streams` sekmeleri için | **OK** |
| T3 | `@BarisOzcan/videos` tam liste | **OK**, 856 ID / 11.3 sn. Başlıklar **İngilizce** geldi ("Why Do You Forget What You Read?") |
| T4 | `--extractor-args "youtube:lang=tr"` | Orijinal başlıklar geldi ("Okuduğunu Neden Unutuyorsun?"). `youtubetab:approximate_date` tek başına yaklaşık `upload_date` veriyor |
| T5 | `curl feeds/videos.xml?channel_id=UCsBjURrPoezykLs9EqgamOA` | **200**, 15 entry, description + views içeriyor. `playlist_id=UULF…` / `UUSH…` → 200, 15'er entry, long/shorts ayrımı doğru |
| T6 | `yt-dlp -J watch?v=l3L_lETiSzc` (default, `tv_simply`, `mweb`) | **FAIL**: `HTTP Error 429` + `ERROR: Sign in to confirm you're not a bot. Use --cookies-from-browser or --cookies…` |
| T7 | `yt-dlp --skip-download --ignore-no-formats-error --extractor-args "youtube:player_client=web_embedded" --write-auto-subs --sub-langs en-orig,tr --sub-format json3` | **Kısmi OK**: `en-orig` json3 indi (343 event). Çeviri `tr` → `HTTP Error 429` |
| T8 | Aynı ayarlar + `lang=tr`, Türkçe video `nnNkf7ttuGc` (`-i --write-info-json`, `tr,tr-orig`) | **OK**: manuel `tr` (367 event) + otomatik `tr-orig` (1077 event, noktalama zayıf) + info.json (chapters, tags, 3776 karakter description, likes/comments). **`upload_date` boş** (watch sayfası 429). info.json 1.4 MB, budanması lazım |
| T9 | `YouTubeTranscriptApi().list/fetch` | **FAIL**: `IpBlocked`. Mesaj: "…requests from an IP belonging to a cloud provider…" |
| T10 | deno kurulu, `yt-dlp -F` (`web_embedded` ve default) | **FAIL**: "formats have been skipped… SABR-only" / bot kontrolü. **Ses indirme yok** |
| T11 | Data API anahtarsız | 403 "unregistered callers". Host erişilebilir, key gerekiyor |
| T12 | `instaloader.Profile.from_username(ctx,"garyvee")` (anonim, tek deneme) | **FAIL**: `ConnectionException: 302 Found when accessing https://www.instagram.com/garyvee/` (login'e yönlendirme) |
| T13 | `vidiq_balance` → `vidiq_channel_videos(@Fireship, long, popular=false)` | Bakiye `{"totalCredits":0,"maxRenewableCredits":150,"renewableResetsAt":"2026-11-06…"}`. Çağrı: *"Not enough credits. This tool costs 5 credits. No credits were charged."* **Tüketim 0 kredi.** Diğer vidIQ çağrıları aynı sonuçla düşeceği için denenmedi |

**vidIQ değerlendirmesi:** Free planda 150 kredi/ay ≈ 30 çağrı [32]. "Tüm videolar" için sayfalama yok (`ig_profile_reels` ≤12). Ayrı bir REST API'si yok; MCP endpoint'i `https://mcp.vidiq.com/mcp` ve Cursor/Gemini CLI gibi client'larda **API key** ile kullanılabiliyor [32]. Yani teknik olarak claude.ai connector'ı dışında da çağrılabilir, ama kredi başına maliyet ve limitler ingestion için uygun değil. Kullanım yeri: agent'ın ad-hoc analizi (ör. `vidiq_watch_shortform_content`, 10 kredi).

---

## 4) Önerilen zincir ve maliyet

### 4.1 YouTube

**Keşif (her ortam):**
1. Artımlı tarama için RSS `UULF` + `UUSH` (son 15'er). Son görülen ID listede varsa dur.
2. Listede yoksa ya da "all" istendiyse Data API `playlistItems.list` (uploads `UU…`, 50/sayfa). Anahtar yoksa `yt-dlp --flat-playlist` + `youtube:lang=<kanal dili>`.
3. Metadata için `videos.list` (50 ID/çağrı) kullanılır. Chapters description'dan parse edilir.

**Transcript (dil önceliği: manuel orijinal > `<lang>-orig` otomatik > STT; çeviri track'i asla alınmaz):**

| Ortam | Primary | Fallback | Last resort (altyazı yok) |
|---|---|---|---|
| **Laptop** (residential IP) | `youtube-transcript-api` | yt-dlp subs (deno + 5–10 sn sleep) | yt-dlp `bestaudio` (orijinal ses track'i) + Groq whisper-large-v3 |
| **Cloud container / Claude scheduled task** | yt-dlp `web_embedded` subs | **ScrapeCreators** transcript (alternatif: Supadata/TranscriptAPI) | Supadata `mode=generate` (2 kredi/dk) **ya da** Gemini'ye YouTube URL + "verbatim transcript" prompt'u (preview, ücretsiz) **ya da** Webshare residential + Groq |
| **Harici scheduled job** (GH Actions vb.) | Cloud ile aynı (datacenter IP) | ScrapeCreators | Supadata / Gemini URL |

### 4.2 Instagram

| Sıra | Yöntem | Ortam |
|---|---|---|
| Primary | Apify `instagram-reel-scraper` (profil URL'i + `resultsLimit`). mp4 **hemen** indirilir (imzalı CDN URL'leri süreli; Apify'ın `downloadedVideo` linki 3 gün geçerli [18]). Sonra `ffmpeg -vn` + Groq whisper-large-v3 | Her yerde |
| Fallback 1 | ScrapeCreators `user/reels` (sayfalı) + `post` (caption) + `media/transcript` (≤120 sn için 1 kredi) | Her yerde |
| Fallback 2 | HikerAPI `clips/chunk` + Groq | Her yerde |
| Resmî tamamlayıcı | Business Discovery. Kullanıcının IG profesyonel hesabı varsa caption/timestamp/view doğrulaması için, `media_url` gelmezse scraper'a düşülür | Her yerde |
| Last resort | Instaloader + **ikincil (burner) hesap session'ı**, yalnız laptop'tan, yavaş tempoda | Laptop |
| Önerilmez | Cloud'dan anonim scraping, ana hesapla instagrapi, vidIQ | — |

### 4.3 Maliyet: 100 video / 100 reel (yalnız edinim + STT; LLM ve görsel analiz hariç)

| Senaryo | Hesap | Toplam |
|---|---|---|
| YouTube, laptop (%95 altyazılı) | Keşif/metadata/transcript $0 + 5 video × 15 dk = 1.25 saat × $0.111 | **≈ $0.14** (video başı ≈ $0.0014) |
| YouTube, cloud, en kötü durum (`web_embedded` tamamen düştü) | 100 × 1 kredi × $0.00188 = $0.19 + altyazısız 5 video: Gemini URL ($0, preview) ya da Supadata 150 kredi (Pro'da ≈ $0.85) ya da proxy ≈ $0.26 + Groq $0.14 | **≈ $0.2–1.0** (video başı ≈ $0.002–0.01) |
| Instagram, Apify + Groq | 100 × $0.0026 = $0.26 (aylık $5 krediden düşer) + 1.25 saat × $0.111 = $0.14 | **≈ $0.40** (aylık ~1900 reel'e kadar fiilen $0) |
| Instagram, ScrapeCreators | ~10 sayfa + 100 post + 100 transcript ≈ 210 kredi × $0.00188 | **≈ $0.40** |
| Instagram, HikerAPI + Groq | ~10 istek × $0.001 + $0.14 | **≈ $0.15** |

Hedef bütçe olan "100 video için birkaç dolar" rahatça tutuyor. Asıl maliyet P3 ve LLM zenginleştirme tarafında olacak.

**Ortam matrisi:**

| | Cloud container | Laptop | Harici scheduled job |
|---|---|---|---|
| YT listeleme/RSS/Data API | ✔ | ✔ | ✔ |
| YT altyazı (ücretsiz) | yalnız `web_embedded` (kırılgan) | ✔ | yalnız `web_embedded` |
| YT ses/video indirme | ✘ | ✔ | ✘ (proxy ile ✔) |
| IG ücretsiz | ✘ | Instaloader + login (riskli) | ✘ |
| Ücretli API'ler | ✔ (env network policy vendor domain'lerine izin vermeli) | ✔ | ✔ |

---

## 5) Riskler ve yasal notlar (hukuki tavsiye değildir)

- **YouTube ToS** "robots, botnets or scrapers" gibi otomatik erişimi ve YouTube'un açıkça izin vermediği indirmeyi yasaklıyor; istisnalar robots.txt'e uyan public arama motorları ve yazılı izin [33]. yt-dlp ve youtube-transcript-api ToS'a aykırı. Pratikteki risk IP ya da hesap bloku, hukuki risk ise kişisel kullanımda düşük. **Azaltma:** keşif ve metadata için resmî Data API; istekler arasında 5–10 sn sleep; kendi hesabının cookie'si kullanılmamalı [6]; transcript'ler yalnız kişisel KB'de tutulmalı, yeniden yayımlanmamalı.
- **YouTube API politikaları:** API verisinin başka kaynaklarla birleştirilmesi ve izin verilenin ötesinde saklanması kısıtlı. Kaynaklar kullanıcıya ayırt edilerek gösterilmeli [34]. KB'de `source` alanı (`api` / `scrape` / `vendor`) tutulması önerilir.
- **Meta/Instagram:** Meta şartları "automated means" ile veri toplamayı yasaklıyor. *Meta v. Bright Data* (N.D. Cal., Hakim Chen) kararında **logged-out** public scraping şartların ihlali sayılmadı, çünkü şartlar hesap kullanımına bağlandı [35]. Sonuç: **login'li scraping (Instaloader/instagrapi + kendi hesabın) sözleşme ve ban riski taşıyor.** Logged-out çalışan vendor'lar ve resmî Business Discovery daha güvenli.
- **Telif:** Transcript ve video içerik sahibinin eseri. Kişisel ve özel kullanım dışında paylaşılmamalı; agent cevaplarında uzun alıntı yerine özet + timestamp linki kullanılmalı.
- **KVKK/GDPR:** Kişisel ve ev içi kullanım istisnası (KVKK m.28/1-a; kaynak bu çalışmada doğrulanamadı). KB paylaşılırsa istisna düşer.
- **Teknik riskler:** yt-dlp ve PO token kırılmaları haftalık oranda yaşanıyor; `web_embedded` her an kapanabilir. Vendor'lar kapanabilir veya fiyat değiştirebilir, bu yüzden adapter katmanı ve ≥2 sağlayıcı şart. IG CDN URL'lerinin süresi doluyor. Auto-translate edilen başlıklar ve **auto-dub ses track'leri** (STT'de orijinal track seçilmeli) ayrı birer tuzak. json3 auto-caption'larda tekrarlanan "rolling" satırlar var, dedupe gerekiyor.

---

## 6) Diğer personalara notlar / açık sorular

- **P1 (KB/RAG):** Kayıt anahtarı `platform:video_id` (idempotent) olmalı. Raw katmanda `info.json` (budanmış) + transcript segmentleri (`start_ms`, `end_ms`, `text`) + `transcript_source ∈ {manual, auto_orig, vendor_X, asr_groq_v3, gemini}` + `lang` + `fetched_at` tutulmalı. Kanal başına `last_seen_ids` (RSS checkpoint). Timestamp'lar `youtu.be/ID?t=123` derin linkleri için şart; `gpt-4o-mini-transcribe` bu yüzden elendi.
- **P3 (görsel):** **Cloud'dan YouTube videosu indirilemiyor** (T6, T10). Seçenekler: Gemini'ye YouTube URL'i (public, preview, free tier 8 saat/gün, ~100 token/sn düşük çözünürlükte [24]), laptop'ta yt-dlp ya da residential proxy. IG mp4'leri scraper'dan geliyor ve indirilebilir.
- **P4 (agent/MCP):** ScrapeCreators, Supadata, TranscriptAPI, Bright Data, Apify ve vidIQ'nun MCP sunucusu var, yani agent'lar KB'de olmayan bir videoyu anlık çekebilir. Cloud ortamının network policy'sinde vendor domain'lerine izin verilmeli; API key'ler environment secret olarak saklanmalı.
- **P5 (maliyet):** Fiyatlar 2026-10 itibarıyla kaynaklı. Groq fiyatı Groq'un fiyat sayfasından değil, duyuru + üçüncü taraftan doğrulandı.
- **P6 (bilgi tasarımı):** Chapters doğal bölüm sınırları olarak kullanılabilir. Description ve pinned yorumlardaki GitHub/kaynak linkleri ayrı entity olarak işlenmeli.
- **Açık sorular (kullanıcıya):**
  1. Ingestion nerede koşacak (laptop mı, cloud mu)? Bu karar ücretsiz yol ile API yolu arasındaki seçimi belirliyor.
  2. IG profesyonel hesabı + FB Page var mı (Business Discovery için)?
  3. ScrapeCreators'ın $47'lık süresiz kredisi mi, Supadata/TranscriptAPI aylık aboneliği mi tercih edilir?
  4. Google Cloud projesi ve YouTube Data API key açılabilir mi?
  5. vidIQ planı yükseltilecek mi? (Önerim: hayır.)

---

## 7) Kaynaklar

[1] https://developers.google.com/youtube/v3/determine_quota_cost
[2] https://developers.google.com/youtube/v3/docs/captions/download
[3] https://github.com/jdepoix/youtube-transcript-api
[4] https://github.com/yt-dlp/yt-dlp/wiki/PO-Token-Guide
[5] https://github.com/yt-dlp/yt-dlp/wiki/EJS
[6] https://github.com/yt-dlp/yt-dlp/wiki/Extractors
[7] https://www.webshare.io/pricing
[8] https://supadata.ai/pricing
[9] https://docs.supadata.ai/api-reference/endpoint/transcript/transcript.md · https://docs.supadata.ai/api-reference/endpoint/youtube/channel-videos.md · https://docs.supadata.ai/integrations/mcp.md
[10] https://scrapecreators.com/
[11] https://docs.scrapecreators.com/v1/youtube/video/transcript
[12] https://docs.scrapecreators.com/v2/instagram/media/transcript
[13] https://docs.scrapecreators.com/v1/instagram/user/reels
[14] https://transcriptapi.com/pricing
[15] https://www.searchapi.io/pricing · https://www.searchapi.io/docs/youtube-transcripts
[16] https://apify.com/pintostudio/youtube-transcript-scraper · https://use-apify.com/docs/best-apify-actors/best-youtube-scrapers
[17] https://apify.com/pricing
[18] https://apify.com/apify/instagram-reel-scraper
[19] https://console.groq.com/docs/speech-to-text · https://groq.com/whisper-large-v3-turbo-now-available-on-groq-combining-speed-quality-for-speech-recognition/ · https://www.morphllm.com/groq-pricing
[20] https://developers.openai.com/api/docs/pricing · https://developers.openai.com/api/docs/guides/speech-to-text
[21] https://deepgram.com/pricing · https://developers.deepgram.com/docs/models-languages-overview
[22] https://www.assemblyai.com/pricing
[23] https://elevenlabs.io/pricing/api · https://elevenlabs.io/speech-to-text/turkish (vendor benchmark)
[24] https://ai.google.dev/gemini-api/docs/pricing · https://ai.google.dev/gemini-api/docs/audio · https://ai.google.dev/gemini-api/docs/video-understanding
[25] https://developers.facebook.com/docs/instagram-platform/instagram-api-with-facebook-login/business-discovery · https://developers.facebook.com/documentation/instagram-platform/instagram-graph-api/reference/ig-user/business_discovery
[26] https://www.getphyllo.com/post/instagram-api-rate-limits-explained-and-how-to-scale-beyond-them-2026 (rate limit; ikincil kaynak; Business Discovery için "200 × kullanıcı/saat" iddiası doğrulanamadı)
[27] https://instaloader.github.io/troubleshooting.html
[28] https://github.com/subzeroid/instagrapi
[29] https://hikerapi.com/pricing · https://hiker-doc.readthedocs.io/en/latest/api-reference/v1/user/
[30] https://ensembledata.com/pricing
[31] https://brightdata.com/products/web-scraper/instagram · https://docs.brightdata.com/mcp-server/overview
[32] https://support.vidiq.com/en/articles/15082430-vidiq-mcp · https://vidiq.com/pricing/
[33] https://www.youtube.com/static?template=terms
[34] https://developers.google.com/youtube/terms/developer-policies-guide
[35] https://www.fbm.com/post/102kqw7/ (Meta v. Bright Data yorumu)
[36] https://transcriptapi.com/blog/youtube-transcript-api-comparison (vendor kaynaklı karşılaştırma)
[37] https://hn.nuxt.dev/item/48030964 (RSS kesinti tartışması, doğrulanamadı)
