# 03 — Video Görsel Anlama: Ekranda Ne Var? (Persona 3 — Multimodal Video Analyst)

> Tarih: 2026-10-20 · Kapsam: YouTube + Instagram Reels videolarından **ekranda görünen** bilgiyi (URL, repo, kod, terminal komutu, araç adı, slayt, UI adımı) zaman damgasıyla, ucuz ve doğru çıkarmak.
> Fiyat/limit verileri resmi sayfalardan 2026-10-20'de alındı. Doğrulanamayan maddeler **"doğrulanamadı"** diye işaretlendi.

---

## 1) TL;DR — Net öneri

1. **H2 kısmen doğru.** YouTube URL'sini doğrudan Gemini'ye vermek çalışıyor: indirme gerekmiyor ve 15 dakikalık bir video **2–9 cent** tutuyor (gemini-3.5-flash-lite). Ancak üç önemli şart var:
   - **Varsayılan çözünürlük kod/URL okumaya yetmiyor.** Gemini 3.x'te varsayılan değer kare başına **70 token**. Google da yoğun metin/OCR için **`high`** ayarını (280 token/kare) öneriyor.
   - **Ucuz 2.5 Flash-Lite yeni projelere kapalı.** 2.5 serisi yalnızca daha önce kullanmış hesaplara açık. Gerçekçi seçenek **3.5 Flash-Lite**: tüm modaliteler $0,30/1M girdi.
   - **YouTube URL hâlâ "preview".** Yalnızca public videolarda çalışıyor. Instagram için URL desteği yok.
2. **Varsayılan motor: `gemini-3.5-flash-lite` (static mod).** Ayarlar şöyle:
   - Ekran/kod içeren videolar: `high`, 0,5–1 fps.
   - Konuşan kafa (talking head): `default`, 0,2 fps.
   - Okunamayan veya doğrulanamayan öğeler için ikinci tur: aynı modelde clip'li **3.8 Flash high** ya da yerel kare + `ultra_high` görsel.
3. **Instagram Reels:** mp4 indirilir (Persona 2), **inline bytes** ile gönderilir (dosyalar küçük; Files API ücretsiz, 48 saat saklıyor). Ayar her zaman `high`, 1 fps. Maliyeti **<1 cent/reel**.
4. **Routing zorunlu.** Önce metinle sınıflandırma yapılır (başlık, açıklama, bölümler, transcript; ~0,2 cent). Ardından tam görsel analiz yalnızca ekran paylaşımlı videolarda çalışır. 100 × 15 dk YouTube için tahmin **≈ $3,3** (Flex/Batch ile **≈ $1,6**). Her videoyu `high` ile işlemek $9,3, 3.8 Flash ile işlemek $22 tutuyor. Tasarruf **%65–85**.
5. **Doğrulama katmanı şart.** LLM, kısmen görünen kod ve URL'leri "tamamlayarak" uydurabiliyor. Bu yüzden:
   - Çıktı `verbatim` + `legibility` alanlarıyla kaydedilir.
   - Repo ve paketler GitHub REST (PAT ile 5.000 istek/saat) ve PyPI/npm üzerinden doğrulanır, açıklamadaki linklerle tekilleştirilir.
   - 404 dönen ya da `partial` işaretli öğeler hedefli olarak yeniden kontrol edilir.
6. **v1.1: yerel keyframe yolu.** Akış: ffmpeg `fps=1,mpdecimate` veya PySceneDetect `HashDetector` → 256-bit pHash ile tekilleştirme → RapidOCR → Claude Haiku 5.5 ya da Gemini görsel `high`. Maliyeti **~0,7–2,5 cent / 15 dk**, tam çözünürlük kontrolü sağlıyor ve zaman damgaları kesin. Kod yoğun kanallarda doğruluk için en iyi yol bu; mp4 indirmeyi gerektiriyor.

---

## 2) Değerlendirilen seçenekler

### 2.1 Gemini'nin video girdisi (Ekim 2026 durumu)

**Girdi yolları**

| Yol | Limit | Kaynak |
|---|---|---|
| YouTube URL | Preview, "available at no charge". Yalnızca **public** video (private/unlisted yok). Free tier'da **günde 8 saat**, paid tier'da süre limiti yok. Gemini 2.5 ve sonrası için istek başına **10 video**. | [video-understanding] |
| Files API | Dosya başına 2 GB, proje başına 20 GB, 48 saat saklama, **ücretsiz** | [files] |
| Inline bytes | İstek başına 100 MB; video sayfası <20 MB'ı öneriyor | [file-input], [video-understanding] |
| Harici HTTPS / signed URL | Dosya başına 15 MB, video/mp4 destekleniyor. IG CDN URL'leriyle çalışıp çalışmadığı **doğrulanamadı**. | [file-input] |

**Bağlam ve modlar**
- 1M bağlamda yaklaşık **3 saat (low)** veya **1 saat (high)** video sığıyor. 3.5 Flash-Lite ve 3.8 Flash'ın input limiti 1.048.576 token. [video-understanding], [model sayfaları]
- **Static mod** (varsayılan) 1 fps ile çalışır. `fps` ile kare hızı, `start_offset`/`end_offset` ile kırpma (clipping) ayarlanır. Bu ayarlar yalnızca static modda geçerli. generateContent API'sinde aynı işi `videoMetadata` görüyor ve YouTube URL'sinde de çalışıyor. [video-understanding], [gc-video]
- **Agentic mod** (3.8 Flash, 3.6 Flash, 3.5 Flash-Lite, 4 Argon): Model önce transcript'e bakıyor, sonra yalnızca ilgili kareleri yüklüyor. Uzun videolarda "%88'e kadar daha az token" kullanıyor. Ancak her şeyi eksiksiz çıkarma (exhaustive extraction) işinde bazı segmentleri atlama riski var. Bu yüzden **static** öneriyorum; agentic, 60 dakikadan uzun webinar'lar için pilotta denenmeli. [video-understanding]

**Token hesabı (Gemini 3.x/4)** [media-resolution]

| Ayar | Token |
|---|---|
| `default` / `low` / `medium` | **70/kare** (videoda bu üç ayar aynı) |
| `high` | **280/kare** |
| Ses | 25 token/sn |
| Görsel (image), `high` | 1120 token |
| Görsel (image), `ultra_high` (yalnızca per-item) | 2240 token |

- Toplam yaklaşık **~100 tok/sn (default)** ve **~300 tok/sn (high)**. [tokens]
- Gemini 2.5'te değerler 64/66, 256/258 ve ses 32 tok/sn. Dokümanlar kendi içinde tutarsız: tokens sayfası "263 tok/sn" diyor. Pilotta `countTokens` ve `usage` ile teyit edilmeli.

**Model fiyatları (Standard, $/1M token; Batch ve Flex ≈ %50)** [pricing]

| Model | Girdi (video) | Girdi (ses) | Çıktı (thinking dahil) | Not |
|---|---|---|---|---|
| **gemini-3.5-flash-lite** | 0,30 | 0,30 | 2,50 | Stabil, agentic destekli, varsayılan thinking: minimal |
| gemini-3.1-flash-lite | 0,25 | 0,50 | 1,50 | **Kapanış: 7 Mayıs 2027** [deprecations] |
| gemini-3.8-flash | 0,75 → **1,50 (1 Ocak 2027'den itibaren)** | aynı | 3,75 → 7,50 | Re-check / zor vakalar için |
| gemini-3-flash-preview | 0,50 | 1,00 | 3,00 | "Legacy" |
| gemini-2.5-flash-lite | 0,10 | 0,30 | 0,40 | **Yalnızca eski kullanıcılara açık** [deprecations] |

**Ek notlar**
- Free tier bu modellerde ücretsiz, fakat "content used to improve our products". RPD limitleri yalnızca AI Studio'da görünüyor, dokümanda yok (doğrulanamadı).
- Tier 1 için harcama limiti 10 dakikada $10. [rate-limits]
- "YouTube URL available at no charge" ifadesi, video token'larının faturalanmadığı anlamına mı geliyor? Bu **doğrulanamadı**. Bütçeyi faturalanıyormuş gibi hesapladım.
- Google'ın sunucusunun YouTube videosunu **hangi çözünürlükte** aldığı belgelenmemiş (**doğrulanamadı**). Kod okuma doğruluğu için bu kritik; pilotta 1080p upload ile karşılaştırılmalı.

### 2.2 Küçük metin/kod okuma — kanıt

- **Resmi kaynak:** Video için `low/medium` (70 token) "çoğu aksiyon tanıma/betimleme için yeterli". `high` (280 token) ise "yalnızca yoğun metin (OCR) veya küçük detay okunacaksa gerekli". [media-resolution]
- **VidText benchmark** (18 LMM): Modeller video içi metin görevlerinin çoğunda zorlanıyor. Performansı belirleyen faktörler **girdi çözünürlüğü** ve OCR yeteneği. Gemini'ye özel rakam özette yok. [vidtext]
- **Yerel deneyimim (ücretsiz, scratch venv):** 1280×720 boyutunda, 16 px monospace koyu temalı 8 satırlık kod ekranında RapidOCR (PP-OCR ONNX) çalıştırdım:

| Çözünürlük | Sonuç |
|---|---|
| 720p | Neredeyse kusursuz (`ChatOpenAI` → `ChatopenAI` gibi tek harf hataları) |
| 360p | `github.con`, `chromiun`, `fron`; bir satır tamamen kayıp |
| 240p | `brouser-use/broier-uie.git`, `opt-5-mins`: kullanılamaz |

  **Sonuç:** Kaynak karede ~720p altına inildiğinde URL ve komutlar bozuluyor. 70 token/kare, düşük bir piksel bütçesine karşılık geliyor (kesin piksel değeri belgelenmemiş). Bu nedenle ekran içerikli videolar için en az `high` gerekiyor; bazı karelerde `ultra_high` görsel gerekebiliyor.
- **Sahne/tekrar tespiti deneyimi** (40 sn'lik sentetik "terminal" videosu, 5 ekran):

| Yöntem | Sonuç |
|---|---|
| PySceneDetect `ContentDetector` (eşik 27/5/3) ve ffmpeg `select='gt(scene,0.02)'` | **0 kesme**. Koyu arka planda yalnızca metin değişiyor; HSV farkı çok küçük. |
| PySceneDetect 0.7.2 `HashDetector` | 5/5 doğru |
| ffmpeg `fps=1,mpdecimate` | 5/5 doğru |
| 64-bit pHash | Farklı iki komut ekranını (**`pip install …` ile `npx create-next-app…`**) mesafe **0** ile aynı kabul etti |
| 256-bit pHash (`hash_size=16`) | Ayırdı (mesafe 38) |

  **Ders:** Ekran kayıtlarında `HashDetector` ya da `mpdecimate` kullanılmalı. Hash'ler 256-bit seçilmeli, son tekilleştirme OCR metni farkına göre yapılmalı.

### 2.3 Karşılaştırma tablosu

Maliyetler: 15 dk video. Varsayımlar: prompt 1,5k token, çıktı (thinking dahil) 4k token, 40 keyframe. Hesaplar bölüm 5'te.

| # | Seçenek | Ekran metni/kod kalitesi | Zaman damgası | 15 dk maliyet | Karmaşıklık | Karar |
|---|---|---|---|---|---|---|
| A | YouTube URL → 3.5 FL **default** 1 fps | Düşük (küçük metin) | ~sn (model tahmini) | $0,036 | Çok düşük | Yalnızca talking head / light pass |
| B | YouTube URL → 3.5 FL **high** 0,5–1 fps | Orta-iyi | ~sn | $0,055–0,093 | Çok düşük | **MVP ana yol (heavy)** |
| C | YouTube URL → 3.8 Flash high | İyi (daha güçlü model) | ~sn | $0,22 (2027'de $0,44) | Çok düşük | Yalnızca clip'li re-check |
| D | Agentic mod (3.5 FL / 3.8 Flash) | Hedefli sorularda iyi, eksiksizlik riski | ~sn | İçeriğe göre değişir | Düşük | 60 dk üstü için pilot |
| E | mp4 indir → Files API / inline → B ayarları | B ile aynı, çözünürlük kontrolü bizde | ~sn | B ile aynı | Orta (indirme) | **IG için zorunlu**, YT için fallback |
| F | Yerel: ffmpeg + mpdecimate/HashDetector + pHash16 + RapidOCR/PaddleOCR | 720p+ kaynakta kod ve URL için iyi; anlam/UI adımı yok | **Kesin** (kare) | ~$0 (CPU) | Orta-yüksek | Doğrulayıcı / ipucu katmanı |
| G | F + vision LLM (keyframe görselleri) | **En iyi kontrol**: crop, `ultra_high` | Kesin | Haiku 5.5 **$0,0075**, Gemini 3.5 FL img-high $0,025, gpt-5.4-mini $0,055, Sonnet 5.5 $0,15 | Yüksek | **v1.1** (kod ağırlıklı kanallar) |
| H | Keyframe → Cloud Vision OCR / Mistral OCR 3 | Ham OCR, anlam yok | Kesin | $0,06 / $0,08 (batch $0,04) | Orta | Gereksiz; yerel OCR yeterli |
| I | Video Intelligence API Text Detection | Ham OCR | Kesin | $2,25 (ayda ilk 1000 dk ücretsiz) | Orta (GCS) | Hayır |
| J | Twelve Labs Pegasus 1.6 | İyi genel anlama | İyi | $0,44 + plan ($29/$99/ay; overage yalnızca Pro'da) | Düşük | Hayır (~10× pahalı) |
| K | Higgsfield `video_analysis_create` (MCP) | Sahne bazlı yaratıcı analiz; araç açıklamasına göre "uzun videoda doğruluk düşer" | Sahne | Kredi (doğrulanamadı; **çalıştırılmadı**) | Pipeline'a uygun değil (yalnızca MCP, 3–5 dk async) | Hayır |
| L | 2.5 Flash-Lite (legacy) | 258 tok/kare default | ~sn | $0,016–0,034 | Çok düşük | Hesap eski kullanıcıysa en ucuzu; yeni hesapta erişilemez |

---

## 3) Önerilen pipeline + routing

```
[P2 girdileri: metadata, açıklama, bölümler, transcript(+ts), IG mp4]
   │
   ├─(1) Link hasadı (regex, $0): açıklama + transcript → URL / owner/repo / pip|npm|npx|brew|uvx|docker
   │
   ├─(2) Router (3.5 FL, thinking=minimal, yalnızca metin, ~$0,002):
   │      visual_need ∈ {none, light, heavy}, beklenen öğe türleri, aday zaman pencereleri
   │
   ├─(3) Görsel geçiş (static mod, structured JSON çıktı)
   │      none  → atla (visual_status="skipped:router"; sonra tekrar çalıştırılabilir)
   │      light → default, fps 0,2   (slayt başlıkları, overlay metinler, araç logoları)
   │      heavy → high, fps 0,5 (canlı kodlama/terminal bölümü varsa 1)
   │      reel  → her zaman high, fps 1 (inline mp4)
   │      >45 dk + high → 20 dk'lık start/end_offset pencerelerine böl (1M bağlam sınırı)
   │
   ├─(4) Normalize + validate (kod): JSON schema, MM:SS→sn, clip offset düzeltmesi,
   │      URL kanonikleştirme, OCR-tipi düzeltme (".con"→".com" yalnızca doğrulanırsa),
   │      açıklama linkleriyle tekilleştirme (kanonik anahtar)
   │
   ├─(5) Zenginleştirme: GitHub REST /repos/{o}/{r} + /readme (PAT), PyPI/npm JSON;
   │      301 → yeni ad; 404 → /search/repositories (30/dk) → "unverified"
   │
   ├─(6) Hedefli re-check (video başına ≤5 aralık):
   │      tetikleyiciler: legibility≠clear olan url/repo/command/code, repo 404, recheck_ranges
   │      YT: clip [t-5, t+10] sn, 3.8 Flash high, fps 2
   │      mp4 varsa: o anın tam çözünürlüklü karesi → crop → image ultra_high + RapidOCR ipucu
   │
   └─(7) KB kayıtları (P1 şeması): provenance = video_id, ts, pass, model, prompt_version, confidence
```

**Router sinyalleri (heavy'ye doğru ağırlık):**
- Başlık veya bölüm adında şu ifadeler: tutorial, demo, setup/kurulum, "how to", GitHub, Claude Code, n8n, Cursor, "step by step"/"adım adım".
- Açıklamada github.com veya npm linki.
- Transcript'te komut benzeri token'lar (npm, pip, clone, terminal, prompt).
- Transcript'te deiktik ifadeler: "as you can see", "here", "this repo", "ekranda gördüğünüz gibi", "şuraya tıklıyoruz", "linki açıklamaya koydum".

**Tasarruf:** Örnek kanal karışımı %20 none / %50 light / %30 heavy, heavy videoların %30'unda re-check. Bu karışımla 100 video **$3,26**. Hepsini `high` ile işlemek $9,28 tutuyor (**−%65**), 3.8 Flash high ise $22,2 (**−%85**). Router hatasının maliyeti asimetrik: heavy bir videoyu light'a düşürmek veri kaybettirir. Bu yüzden emin olunamayan durumda **heavy** seçilmeli. Ayrıca light geçişte `text_density=high` çıkarsa video heavy'ye **otomatik yükseltilmeli**.

**Artımlı güncelleme:** Anahtar `(video_id, pipeline_version, prompt_version, model)`. Ham model JSON'u saklanır. Prompt veya model değişirse yalnızca etkilenen geçiş yeniden çalıştırılır.

---

## 4) Prompt taslağı + JSON şeması

Prompt İngilizce yazıldı (talimat takibi daha tutarlı). Ekrandaki metin orijinal dilinde (TR/EN) aynen korunuyor. Özet alanlarının dili `{output_lang}` ile seçiliyor (Persona 6 karar versin).

**Interactions API parametreleri:**
- `response_format={"type":"text","mime_type":"application/json","schema":…}` [structured-output]
- `thinking_level="low"`
- video part'ında `"resolution":"high"` ve `"processing":{"type":"static","fps":0.5}`
- Kurallar ve şema system instruction'a; video önce, metin sonra. [video-understanding]

```text
SYSTEM
You are a meticulous on-screen content extractor building a knowledge base from creator videos
(LinkedIn growth, marketing, sales, software engineering). You receive the video (frames + audio).
Your job: record what is VISIBLE ON SCREEN, with timestamps. Audio is context only.

RULES
1. Verbatim first. Copy on-screen text exactly as displayed (case, punctuation, spacing in code).
   NEVER complete, correct, translate or guess. Mark unreadable characters/segments with ⟨?⟩ and set
   legibility to "partial" or "unreadable". A wrong URL is worse than a missing one.
2. Priority targets: URLs; GitHub/GitLab repos (owner/repo); terminal commands; code; package names
   (pip/npm/npx/brew/uvx/docker); tool/product/site names visible in UI (tabs, logos, window titles);
   slide titles + bullets; numbers on charts/metrics; step-by-step UI actions (click/type/navigate).
3. Timestamps: "MM:SS" (or "H:MM:SS" if ≥1h) relative to the START OF THE PROVIDED VIDEO/CLIP.
   start = first frame the item is readable; end = when it disappears. If an item reappears,
   output it once and add extra ranges in "also_at".
4. "spoken": yes / partially / no / unknown — is the same information also said in the audio?
   Always include on-screen-only items; they are the most valuable.
5. Code: one item per visible block; "completeness" = full | partial_scrolled | partial_cropped.
   Commands: one item per command line, without prompt chars like "$" or ">".
6. UI steps: an ordered list of concrete actions (app, action, target, typed value).
7. Ignore presenter appearance, clothing, background, generic b-roll. Ignore subtitles/captions
   that merely duplicate speech (but DO capture on-screen titles/overlays that add information).
8. If text is too small/blurred to read reliably, add a recheck_ranges entry with the reason.
9. Context links below are hints for spelling only. Report an item ONLY if it is actually visible.
10. Write "summary"/"context" fields in {output_lang}; keep "verbatim" in the original language.
11. Output ONLY JSON matching the schema. Use empty arrays when nothing applies.

USER (after the video part)
Video: "{title}" — channel: {channel} — platform: {platform} — duration: {duration}
Clip offset (if clipped): {clip_start_s}s.  Router hint: visual_need={visual_need}, expected={expected_kinds}
Context links from description (hints only): {description_links}
Extract all on-screen information per the rules.
```

**JSON şeması** (Gemini structured output alt kümesi; `oneOf` kullanılmadı, tür bazlı alanlar opsiyonel):

```json
{
  "type": "object",
  "required": ["video_profile", "segments", "items", "recheck_ranges"],
  "properties": {
    "video_profile": {
      "type": "object",
      "required": ["visual_type", "screen_share_ratio", "text_density", "visual_value"],
      "properties": {
        "visual_type": {"type": "string", "enum": ["talking_head", "screen_share", "slides", "mixed", "b_roll", "whiteboard", "other"]},
        "screen_share_ratio": {"type": "number", "description": "0-1 share of time showing screen/slides"},
        "text_density": {"type": "string", "enum": ["none", "low", "medium", "high"]},
        "visual_value": {"type": "string", "enum": ["none", "low", "medium", "high"], "description": "info on screen NOT in audio"},
        "on_screen_languages": {"type": "array", "items": {"type": "string"}}
      }
    },
    "segments": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["start", "end", "scene_type", "summary"],
        "properties": {
          "start": {"type": "string"}, "end": {"type": "string"},
          "scene_type": {"type": "string", "enum": ["face", "code_editor", "terminal", "browser", "app_ui", "slide", "diagram", "chart", "b_roll", "other"]},
          "app_or_site": {"type": "string"},
          "summary": {"type": "string"}
        }
      }
    },
    "items": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "kind", "start", "verbatim", "legibility", "confidence", "spoken"],
        "properties": {
          "id": {"type": "string", "description": "i1, i2, ..."},
          "kind": {"type": "string", "enum": ["url", "repo", "command", "code", "package", "tool_or_product", "slide", "ui_step", "chart_or_metric", "other_text"]},
          "start": {"type": "string"}, "end": {"type": "string"},
          "also_at": {"type": "array", "items": {"type": "string", "description": "MM:SS-MM:SS"}},
          "verbatim": {"type": "string"},
          "normalized": {"type": "string", "description": "https URL | owner/repo | package name | tool name; empty if unsure"},
          "code_language": {"type": "string", "description": "bash, python, ts, yaml ..."},
          "completeness": {"type": "string", "enum": ["full", "partial_scrolled", "partial_cropped", "n/a"]},
          "slide_title": {"type": "string"},
          "slide_bullets": {"type": "array", "items": {"type": "string"}},
          "step_index": {"type": "integer"},
          "ui_action": {"type": "string", "enum": ["click", "type", "navigate", "select", "drag", "scroll", "shortcut", "other"]},
          "ui_target": {"type": "string"}, "ui_app": {"type": "string"}, "ui_value": {"type": "string"},
          "context": {"type": "string", "description": "1 sentence: what it is / why shown"},
          "spoken": {"type": "string", "enum": ["yes", "partially", "no", "unknown"]},
          "legibility": {"type": "string", "enum": ["clear", "partial", "unreadable"]},
          "confidence": {"type": "number"}
        }
      }
    },
    "recheck_ranges": {
      "type": "array",
      "items": {"type": "object", "required": ["start", "end", "reason"],
        "properties": {"start": {"type": "string"}, "end": {"type": "string"}, "reason": {"type": "string"}}}
    }
  }
}
```

**Pipeline zarfı** (kod tarafından eklenir, modelden istenmez):
- `schema_version`, `video_id`, `platform`, `source_url`, `pass` (light | heavy | recheck)
- `model`, `resolution`, `fps`, `clip_offset_s`, `prompt_version`, `usage` (token sayıları), `created_at`
- Her öğe için:
  - `t_start_s` / `t_end_s` ve derin link (ör. `https://youtu.be/{id}?t={s}`)
  - `sources[]`: `screen@ts`, `description`, `transcript@ts`
  - `validation`: `exists`, `redirected`, `not_found`, `unchecked`
  - `enrichment`: GitHub `full_name`, `description`, `stargazers_count`, `topics`, `license`, `pushed_at`, `archived`, `homepage`, ilk ~2k karakterlik README ve README sha, `fetched_at`

---

## 5) Maliyet

**Formül (Gemini 3.x, static):**
```
input ≈ süre_sn × (fps × {70 | 280} + 25) + prompt
```
Çıktı varsayımı (thinking dahil): 15 dk için 4k token, 60 sn için 1,2k token. Hesap betiği scratch'te. Kaynak fiyatlar: [pricing].

### 5.1 Tek video — 15 dk

Token sayıları: default@1fps ≈ 87k · high@1fps ≈ 276k · high@0.5fps ≈ 150k · default@0.2fps ≈ 37k

| Model | default @1fps | **high @1fps** | high @0,5fps | default @0,2fps |
|---|---|---|---|---|
| **3.5 Flash-Lite** | $0,036 | **$0,093** | $0,055 | $0,021 |
| 3.1 Flash-Lite | $0,033 | $0,081 | $0,049 | $0,021 |
| 3.8 Flash (2026) | $0,080 | $0,222 | $0,128 | $0,043 |
| 3.8 Flash (2027) | $0,161 | $0,444 | $0,255 | $0,085 |
| 3 Flash Preview | $0,067 | $0,161 | $0,098 | $0,042 |
| 2.5 Flash-Lite (legacy) | low $0,016 / default $0,034 | — | — | — |

### 5.2 Tek video — 60 sn reel

| Model | default @1fps | **high @1fps** |
|---|---|---|
| **3.5 Flash-Lite** | $0,0052 | **$0,0089** |
| 3.1 Flash-Lite | $0,0040 | $0,0071 |
| 3.8 Flash (2026) | $0,0099 | $0,0193 |
| Pegasus 1.6 | $0,029 (+plan) | — |

### 5.3 Keyframe yolu (15 dk, 40 kare 1280×720, transcript bağlamı dahil)

| Seçenek | Maliyet |
|---|---|
| Gemini 3.5 FL görsel `high` | $0,025 |
| Gemini 3.5 FL görsel `ultra_high` | $0,039 |
| **Claude Haiku 5.5** (≈1229 token/kare; ≤100k prompt için $0,10/$0,50) | **$0,0075** [claude-pricing], [claude-vision] |
| gpt-5.4-mini | $0,055 [openai-pricing], [openai-vision] |
| Claude Sonnet 5.5 | $0,149 |

Yerel OCR ve sahne tespiti ücretsiz (CPU, ~0,2–1 sn/kare).

### 5.4 100 video senaryoları

| Senaryo | Standard | Flex / Batch (−%50) |
|---|---|---|
| **S1:** 100 × 15 dk YouTube, routing (R0 $0,23 + light 50× $1,05 + heavy 30× $1,65 + re-check 9× ($0,036; 2 dk clip, 3.8 Flash) $0,33) | **$3,26** | **≈ $1,65** |
| S1-a: hepsi 3.5 FL high @1fps | $9,28 | $4,64 |
| S1-b: hepsi 3.8 Flash high | $22,2 (2027: $44,4) | $11,1 |
| S1-c: hepsi default (OCR zayıf) | $3,61 | $1,8 |
| **S2:** 100 × 60 sn reel, hepsi high @1fps | **$0,89** | **≈ $0,45** |
| S3: S1 + keyframe doğrulayıcı (heavy videolarda Haiku 5.5) | +$0,23 | — |
| GitHub/PyPI/npm zenginleştirme | $0 | — |

S1 için ek notlar:
- YouTube girdisi gerçekten ücretsizse S1 yalnızca çıktı ücretine iner: **≈ $0,6–1,0** (doğrulanamadı).
- Free tier'da YouTube için günde 8 saat (~32 adet 15 dakikalık video) ücretsiz. Pilot için ideal, ancak free tier verisi ürün geliştirmede kullanılıyor.

Flex hakkında: Preview aşamasında; gecikme değişken ve "best-effort". Bizim artımlı ve kişisel kullanımımıza uygun. [flex]

Zenginleştirme hesabı: 100 video × ~3 repo × 2 çağrı ≈ 600 istek. Kimlik doğrulamasız limit 60 istek/saat, bu yüzden **PAT şart** (5.000 istek/saat; search 30 istek/dk, code search 10 istek/dk). [gh-rate], [gh-search]

---

## 6) Riskler & azaltma

| Risk | Etki | Azaltma |
|---|---|---|
| Model kısmi kod/URL'yi "tamamlayıp" uyduruyor | KB'ye yanlış repo/komut girer | `verbatim` + `⟨?⟩`, `legibility`; repo ve paketleri API ile doğrula; doğrulanamayanı "unverified" işaretle, ajanlara düşük güvenle sun |
| Varsayılan çözünürlük metni okuyamıyor | Kod/URL kaçar | heavy ve reel için `high`; okunamayanda re-check (3.8 Flash clip ya da `ultra_high` crop) |
| YouTube URL preview: fiyat/limit değişebilir; private/unlisted/yaş kısıtlı/üyelik videoları çalışmaz | Kesinti | İndirme → Files API fallback'i (P2); hata türüne göre otomatik geçiş |
| YouTube ingest çözünürlüğü bilinmiyor | Kod okuma tavanı belirsiz | Pilot: 10 videoluk altın set, URL ile 1080p upload karşılaştırması (recall/precision) |
| Zaman damgası kayması (1 fps örnekleme, model tahmini, clip'te göreli/mutlak belirsizliği — doğrulanamadı) | Derin link sapar | ±5 sn tolerans; clip offset testi; keyframe yolunda kesin ts |
| 1M bağlam: 60 dk × high ≈ 1,1M token | İstek reddi | >45 dk'yı 20 dk pencerelere böl veya 0,5 fps |
| Fiyat ve ömür: 3.8 Flash 1 Ocak 2027'de 2×; 3.1 FL 7 Mayıs 2027'de kapanıyor; 2.5 erişimi kısıtlı | Bütçe/kırılma | Model adı config'te; varsayılan 3.5 FL; aylık fiyat kontrolü |
| Sahne tespiti metin değişimini kaçırıyor; 64-bit pHash farklı ekranları birleştiriyor (deneyle gösterildi) | Keyframe kaybı | `HashDetector`/`mpdecimate`, pHash16, OCR metin farkıyla final dedupe |
| Agentic mod bazı segmentleri atlayabilir | Eksik çıkarım | Eksiksiz çıkarımda static; agentic yalnızca hedefli sorgu / uzun video pilotu |
| Türkçe karakterler (ç, ğ, ı, İ, ş) yerel OCR'da düşebilir (RapidOCR varsayılan modeli zh/en; doğrulanamadı) | Slayt metni hatası | Gemini'yi esas al; yerel OCR'ı yalnızca ASCII kod/URL ipucu olarak kullan; Tesseract kullanılırsa `tur` dil paketi |
| Free tier'da veri eğitimde kullanılıyor; spend limiti (Tier 1: $10/10 dk) | Gizlilik / 429 | Paid tier; eşzamanlılık 5–10; exponential backoff |
| Cloud oturumlarında GitHub API proxy'li: bu oturumda bağlı olmayan repolar için `api.github.com` engellendi (gözlem) | Ajan sorgu anında zenginleştiremez | Zenginleştirmeyi ingest sırasında yap, README özetini KB'de sakla |
| ToS / telif (indirme, kişisel kullanım) | Hukuki | Persona 2/5 değerlendirmesi; ham video saklama, yalnızca türetilmiş metin |

---

## 7) Diğer personalara notlar / açık sorular

**P1 (KB/RAG)**
- Görsel öğeleri ayrı chunk türü olarak saklayın: `kind`, `verbatim`, `normalized`, `t_start_s`, derin link, `confidence`, `validation`.
- Repo ve araçlar için kanal genelinde bir **entity tablosu** tutun. Kanonik anahtarlar `github:owner/repo`, `pypi:name`, `npm:name`; kaynak listesi video×ts.
- Embedding metni = `verbatim` + `context` + başlık. Ham model JSON'u ve `prompt_version` versiyonlansın.

**P2 (veri edinimi)**
- IG için mp4 gerekli: ≥720p (OCR deneyi 360p'de bozuluyor), ideal olarak <20 MB. Reels'in azami süresi 3 dk mı, P2 teyit etsin.
- YouTube'da indirme yalnızca fallback ve re-check için. Bölümler (chapters), açıklama ve zaman damgalı transcript router'a girdi oluyor.
- Agentic modun YouTube URL'lerinde transcript'i nereden aldığı belgelenmemiş.

**P4 (agent/MCP)**
- Önerilen araçlar: `list_repos(channel)`, `get_on_screen(video_id, kind)`, `search_on_screen(query)`, `get_commands(topic)`. Hepsi derin link döndürsün.
- Cloud oturumlarında harici GitHub API'ye güvenmeyin; zenginleştirme KB'de hazır olmalı.

**P5 (maliyet)**
- Görsel katman için bütçe: 100 uzun video $1,6–3,3; 100 reel $0,45–0,9.
- "YouTube URL no charge" ifadesi bütçeyi yarıdan fazla düşürebilir; fatura ile doğrulanmalı.
- 1 Ocak 2027'deki 3.8 Flash fiyat artışına dikkat.

**P6 (ontoloji/prompt)**
- `kind` enum'ını ve UI-adım modelini ontolojiyle eşleyin.
- `output_lang` kararı: TR mi, EN mi, video dili mi?
- Slaytlardaki "framework" ve "hook" listeleri (LinkedIn içerikleri) ayrı bir `concept` türü olmalı mı?

**Açık sorular (kullanıcıya)**
1. UI adımlarının tam yeniden üretimi isteniyor mu? Çıktı token'ını yaklaşık 2× artırır.
2. Keyframe görselleri (küçük thumbnail) KB'de saklanıp ajanlara gösterilsin mi?
3. Kabul edilebilir zaman damgası hatası: ±5 sn yeterli mi?
4. Pilot için 10 videoluk altın seti (TR+EN, kod ağırlıklı + talking head) kim seçecek?

---

## 8) Kaynaklar

- [video-understanding] https://ai.google.dev/gemini-api/docs/video-understanding
- [gc-video] https://ai.google.dev/gemini-api/docs/generate-content/video-understanding.md.txt (videoMetadata, YouTube clipping/fps)
- [media-resolution] https://ai.google.dev/gemini-api/docs/media-resolution · 2.5 tablosu: https://ai.google.dev/gemini-api/docs/generate-content/media-resolution.md.txt
- [tokens] https://ai.google.dev/gemini-api/docs/tokens · https://ai.google.dev/gemini-api/docs/interactions/tokens.md.txt
- [pricing] https://ai.google.dev/gemini-api/docs/pricing
- [deprecations] https://ai.google.dev/gemini-api/docs/deprecations
- [models] https://ai.google.dev/gemini-api/docs/models · https://ai.google.dev/gemini-api/docs/models/gemini-3.5-flash-lite.md.txt · https://ai.google.dev/gemini-api/docs/models/gemini-3.8-flash.md.txt
- [rate-limits] https://ai.google.dev/gemini-api/docs/rate-limits
- [files] https://ai.google.dev/gemini-api/docs/files.md.txt
- [file-input] https://ai.google.dev/gemini-api/docs/file-input-methods.md.txt
- [flex] https://ai.google.dev/gemini-api/docs/flex-inference.md.txt
- [thinking] https://ai.google.dev/gemini-api/docs/thinking.md.txt
- [structured-output] https://ai.google.dev/gemini-api/docs/structured-output.md.txt
- [vidtext] https://arxiv.org/abs/2505.22810
- [claude-pricing] https://platform.claude.com/docs/en/about-claude/pricing · [claude-vision] https://platform.claude.com/docs/en/build-with-claude/vision
- [openai-pricing] https://developers.openai.com/api/docs/pricing · [openai-vision] https://developers.openai.com/api/docs/guides/images-vision
- Twelve Labs: https://www.twelvelabs.io/pricing · https://www.twelvelabs.io/overage
- Mistral OCR 3 ($2/1000 sayfa, batch $1): https://mistral.ai/news/mistral-ocr-3
- Cloud Vision: https://cloud.google.com/vision/pricing · Video Intelligence: https://cloud.google.com/video-intelligence/pricing
- [gh-rate] https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api · [gh-search] https://docs.github.com/en/rest/search/search
- PySceneDetect detektörleri: https://www.scenedetect.com/docs/latest/api/detectors.html
- PyPI sürümleri (2026-10-20): scenedetect 0.7.2, paddleocr 3.7.0, rapidocr 3.10.0, imagehash 4.3.2, yt-dlp 2026.8.19, instaloader 4.15.4, google-genai 2.31.0 — https://pypi.org/pypi/{paket}/json
- Higgsfield `video_analysis_create`: MCP araç tanımı (ToolSearch ile okundu, çalıştırılmadı)
- Yerel deneyler (ffmpeg 6.1.1, scenedetect 0.7.2, imagehash 4.3.2, rapidocr_onnxruntime 1.2.3): scratchpad `p3/scripts/{keyframes,hashdist,ocr,cost}.py`
