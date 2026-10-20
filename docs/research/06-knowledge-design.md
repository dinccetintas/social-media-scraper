# 06 — Bilgi Tasarımı: Ontoloji, Çıkarım, Sentez ve LLM Wiki

**Persona 6 — Knowledge Engineer** · Tarih: 2026-10-20 · Kapsam: KB'ye *hangi bilginin* girdiği ve agent'ın onu uzman gibi kullanabilmesi için *nasıl yapılandırıldığı*.

---

## 1) TL;DR — net öneri

1. **H4 doğrulandı, bir düzeltmeyle: iki katman.** (a) *Makine katmanı* = doğrulanmış **knowledge item**'lar (`items.jsonl`). Her item bir video + segment + orijinal dilde birebir alıntı taşır ve gerçeğin kaynağı budur. (b) *Wiki katmanı* = bu item'lardan **derlenen** git-versiyonlu markdown sayfalar (Karpathy "LLM Wiki" deseni). Sayfalar item'lardan yeniden derlenir; LLM düzyazının üstüne yama yapmaz. Böylece ardışık yeniden yazımlarda atıflar kaybolmaz ve kayma birikmez. Hybrid index üç şeyi indeksler: sayfalar, item'lar ve ham segmentler. Karpathy'nin kendisi `index.md` ile gezinmenin yalnızca "~100 kaynak / birkaç yüz sayfa" ölçeğine kadar iyi çalıştığını söylüyor. "Tüm videolar" senaryosu (500+ video) bu yüzden arama gerektirir.
2. **Minimal ontoloji: 5 tip.** `Channel`, `Video` (+ gömülü `Segment`), `Topic`, `KnowledgeItem` (polimorfik `kind`: tactic | framework | playbook | concept | claim | example) ve `Resource` (`kind`: tool | repo | website | book | course | person | product). Ayrı Quote, Person veya Metric tipi yok; bunlar alan olarak yaşar.
3. **Video başına tek LLM çağrısı** (structured output). LLM zaman damgası yazmaz, **segment ID** (`s0042`, `v0007`) yazar. Kod alıntıyı transkriptte veya OCR'da fuzzy eşleştirerek doğrular ve zamanı segmentten türetir. Böylece zaman damgası halüsinasyonu yapısal olarak ortadan kalkar. Kota da yok: kural "hepsi ve sadece". Fabric'in "en az 20 madde" kalıbı kısa Reel'lerde uydurmaya yol açar; bir Reel'den 0–3 item çıkması normaldir.
4. **Dil kuralı.** Alıntılar her zaman orijinal dilde (TR/EN) saklanır, çünkü doğrulama onlarla yapılır. Normalize alanlar (`statement_en`) ve wiki düzyazısı İngilizce olur; kanallar arası birleştirme bu sayede çalışır. Türkçe karşılıklar `aliases` olarak tutulur. Agent kullanıcıya her zaman kullanıcının dilinde cevap verir.
5. **Artımlı güncelleme.** Kanonik slug kayıt defteri (`registry.json`) extraction prompt'una eklenir; yeni video ya mevcut bir slug'a bağlanır ya da yeni bir slug önerir. Yalnızca "kirlenen" (dirty) sayfalar yeniden derlenir. `overview.md` ancak içeriğin %10'u yenilendiğinde ya da aylık lint'te yeniden derlenir.
6. **Maliyet.** 100 videoluk bir kanalın ilk kurulumu yaklaşık 145 çağrı ve ~1.2M giriş / ~0.3M çıkış tokeni tutar. Bu, Sonnet 5.5 ile ≈ **$5.5**, Batch ile ≈ **$2.8** demek. Sonrasında her yeni video ~6 çağrı ve ≈ $0.03–0.08.

---

## 2) Ontoloji

### Varlıklar

| Tip | Ne | Saklandığı yer | Çıkar? |
|---|---|---|---|
| **Channel** | platform, handle, dil, kitle varsayımı, tez | `overview.md` | ✔ (sentezle) |
| **Video** | id, url, published_at, süre, format (short ≤90 sn / long), dil, content_type, chapters | `videos/<id>.md` + frontmatter | ✔ |
| **Segment** | `s####` (konuşma) / `v####` (ekran: slide/code/url/overlay); start–end sn | ham katman (`raw/…/transcript.json`, `visual.json`) | atıf çapası, sayfa değil |
| **Topic** | global slug (`linkedin-hooks`), 2 seviyeyi geçmeyen taksonomi | `topics/<slug>.md`, `_topics/<slug>.md` | ✔ |
| **KnowledgeItem** | `kind` aşağıdaki gibi; `statement_en`, `polarity`, `speaker`, `specificity`, `evidence[]` | `items.jsonl`; sayfalarda derlenmiş hâli | ✔ çekirdek |
| **Resource** | tool / repo / website / book / course / person / product; `stance` | `resources.md` | ✔ |

**KnowledgeItem.kind:**
- `tactic`: atomik do/don't tavsiye ("İlk satır ≤12 kelime"). Hatalar ve "yapma"lar ayrı bir tip değildir; `polarity: dont` ile işaretlenir.
- `framework`: adlandırılmış ya da açıkça yapılandırılmış bir model ve bileşenleri ("5 hook kalıbı", "Hook-Story-Offer").
- `playbook`: bir sonuca giden, en az 3 adımlı, sıralı prosedür. Ekrandaki adım adım demolar da buraya girer. Adımlar `steps[]` içinde gömülüdür ve her adımın kendi segment ID'si vardır.
- `concept`: kanalın tanımladığı ya da özel anlamda kullandığı terim. `glossary.md` dosyasına gider.
- `claim`: ampirik veya olgusal iddia; sayılar, sonuçlar, benchmark'lar `numbers[]` alanında tutulur. Metric/Benchmark bu tipin içinde yaşar.
- `example`: vaka, hikâye ya da önce/sonra karşılaştırması. `example_of` ile başka bir item'a bağlanır.

### İlişkiler (yalnızca bunlar)
`Item —evidence→ Video@Segment` (zorunlu, 1..3) · `Item —about→ Topic` (1..3) · `Item —uses→ Resource` · `Item —example_of→ Item` · sentez aşamasında `Item —refines | contradicts | supersedes→ Item` · `Video —covers→ Topic`. Framework bileşenleri ve playbook adımları item'ın içine gömülüdür; ayrı varlık olmazlar.

### Kritik meta alanlar
- `speaker` (host | guest | unknown + ad): röportajlarda "kanal diyor" ile "konuk diyor" ayrımını mümkün kılar.
- `stance` (Resource): recommended | used_on_screen | mentioned | warned_against | sponsored. Sponsorlu araç önerisi, organik öneriyle aynı ağırlıkta tutulamaz.
- `specificity` (1 genel klişe · 2 spesifik · 3 sayı / kesin adım / kod): platitude filtresi.
- `platform_sensitive` (bool): algoritma, fiyat, UI veya API sürümüne bağlı iddia demek. 12 ay sonra bayatlamış sayılır.

### Gürültü: çıkarılmayanlar
Selam/kapanış, CTA'lar ("yorum yaz REHBER", like/subscribe), sponsor metninin kendisi (yalnızca `stance: sponsored` bilgisi tutulur), espriler, ders çıkarılmamış kişisel hikâye, spesifik olmayan klişeler ("tutarlı ol"), alanla ilgisiz genel dünya bilgisi. Fabric `extract_wisdom` deseninin HABITS ve FACTS bölümlerini almıyoruz; amaç bilgelik derlemek değil, uygulanabilir uzmanlık. Instagram'a özgü bir durum: içerik bir lead magnet'in arkasındaysa (`content_gated: true`) `knowledge_density: low` olarak işaretlenir.

---

## 3) Video başına çıkarım

### Girdiler (prompt'a XML ile; Anthropic önerisi: uzun veri üstte, talimat sonda)
- `<video_meta>`: platform, url, title, published_at, duration_s, lang, native_chapters
- `<registry>`: kanalın mevcut kanonik slug'ları (`slug | kind | aliases`) ve global topic listesi. ~2–5k token, **prompt cache**'lenir.
- `<transcript>`: satır başına `[s0042 03:12] metin`
- `<screen>`: Persona 3 çıktısı, `[v0007 03:15 slide|code|url|terminal|overlay] OCR metni`
- `<description>`: açıklama, sabit yorum, linkler

### JSON şeması (structured outputs, `output_config.format`)
Claude structured outputs yalnızca belirli JSON Schema özelliklerini destekliyor. `minLength`/`maxItems`/`minimum` desteklenmiyor. İsteğe bağlı parametre (24) ve union tipi (16) sayısında da limit var. Bu yüzden **bütün alanlar `required`** ve boş değer `""` ya da `[]` olarak verilir (nullable union kullanılmaz). Sınırlar şemada değil, kodda doğrulanır.

```jsonc
{
  "content_type": "tutorial|advice|interview|case_study|demo|listicle|promo|other",
  "knowledge_density": "none|low|medium|high",
  "content_gated": false,
  "summary_en": "≤60 words",
  "takeaway_en": "one sentence",
  "topics": [{ "slug": "linkedin-hooks", "is_new": false }],
  "items": [{
    "id": "i03",
    "kind": "tactic|framework|playbook|concept|claim|example",
    "title_en": "Keep hook line 1 under ~12 words",
    "statement_en": "Self-contained, specific; keeps creator's hedges/conditions",
    "polarity": "do|dont|neutral",
    "speaker": "host|guest|unknown", "speaker_name": "",
    "topics": ["linkedin-hooks"],
    "canonical": "tactic/hook-line-length | new:<proposed-slug>",
    "example_of": "",                         // item id or ""
    "specificity": 3,                         // enum [1,2,3]
    "platform_sensitive": false,
    "steps":   [{ "n": 1, "text_en": "…", "ref": "s0051" }],       // playbook/framework
    "numbers": [{ "value": "2x", "what_en": "see-more clicks", "basis": "self-test n=40" }],
    "resources": ["r02"],
    "evidence": [{ "ref": "s0041", "source": "speech|screen|description",
                   "quote": "İlk satır on iki kelimeyi geçmesin…" }]   // verbatim, original lang
  }],
  "resources": [{ "id": "r02", "kind": "tool|repo|website|book|course|person|product",
                  "name": "AuthoredUp", "url": "", "stance": "recommended|used_on_screen|mentioned|warned_against|sponsored",
                  "why_en": "preview mobile cut-off", "evidence": [{ "ref": "s0090", "source": "speech", "quote": "…" }] }],
  "chapters": [{ "ref": "s0000", "title_en": "…" }],       // only if no native chapters & long
  "notable_quotes": [{ "ref": "s0120", "quote": "…" }]     // ≤3; for persona "voice"
}
```

### Extraction prompt taslağı (model-facing, İngilizce)

```text
SYSTEM: You are a knowledge-extraction engine. You convert ONE video's transcript,
on-screen content and description into verifiable knowledge items that an AI expert
agent will later use to advise a practitioner. Fidelity beats coverage: an item you
cannot support with a verbatim quote must not exist.

USER:
<video_meta>…</video_meta>
<registry>…</registry>
<transcript>…</transcript>
<screen>…</screen>
<description>…</description>

<instructions>
1. Read everything first. Set content_type and knowledge_density. Promotional,
   gated or entertainment-only content may yield ZERO items — that is a valid answer.
2. Extract ALL and ONLY knowledge a practitioner could act on, or needs to judge advice:
   tactic (one atomic do/don't), framework (named/structured model + components),
   playbook (ordered procedure ≥3 steps toward an outcome, incl. on-screen demos;
   give each step its own ref), concept (term the channel defines/uses specially),
   claim (empirical/factual assertion; capture numbers + their basis), example
   (concrete case illustrating another item; set example_of).
   Resources: every tool/repo/site/book/course/person recommended, used on screen,
   or warned against. stance=sponsored for paid reads or disclosed affiliates.
3. Do NOT extract greetings, CTAs, sponsor copy, jokes, life stories without a lesson,
   generic platitudes, or world facts outside the channel's domain.
4. Evidence first: for each item, locate the supporting span(s) and copy them VERBATIM
   in the ORIGINAL language (do not translate, do not fix ASR errors, ≤40 words) with
   the s####/v#### ref where they occur. Never invent refs. Max 3 evidence spans.
5. statement_en: specific, self-contained English sentence. Keep numbers, conditions,
   audience and hedges ("in my test", "for B2B SaaS"). Add nothing not in the video.
6. Attribute speaker (host vs guest; name if stated).
7. Canonicalize against <registry>: reuse an existing slug when the item expresses the
   same idea (even in another language); otherwise "new:<kebab-slug>". Prefer existing topics.
8. specificity: 1 generic · 2 specific · 3 specific + numbers/exact steps/code.
   platform_sensitive=true if it depends on an algorithm, pricing, UI or API version.
9. Scale to content, never pad: a 30-60s reel usually yields 0-3 items; 10 min ≈ 5-15;
   60 min ≈ 15-40. Merge restatements into ONE item with multiple evidence spans.
10. Before answering, check: every ref exists in the input; quotes are verbatim;
    no two items say the same thing; nothing is asserted that the video does not say.
</instructions>
```

Prompt'u besleyen kaynaklar şunlar. Fabric'in `extract_instructions` (adımlar), `extract_recommendations`, `extract_references` (kaynaklar), `extract_sponsors` (organik/sponsorlu ayrımı) ve `create_video_chapters` (zaman damgaları sıralı olmalı ve süre içinde kalmalı) desenleri. Fabric'ten bilerek **almadıklarımız**: sabit adetler ("20–50 IDEAS", "tam 16 kelime"), markdown çıktı ve kaynak/zaman bağlantısının olmaması. Anthropic'ten aldıklarımız: *uzun veri üstte*, *önce birebir alıntı*, *alıntı bulunamazsa iddiayı geri çek*, *harici bilgiyi kısıtla*.

### Kodla doğrulama (LLM sonrası, deterministik)
1. **Ref kontrolü:** ID girdide var mı?
2. **Alıntı doğrulama:** Önce normalize edilir: NFKC, Türkçe'ye duyarlı küçük harf (`İ→i`, `I→ı`; Python `casefold` burada hatalı sonuç verir), noktalama silme, boşluk sıkıştırma. Ardından `[ref−1, ref+2]` segment penceresinde exact substring aranır, bulunamazsa `rapidfuzz.partial_ratio ≥ 90` denenir; ASR hataları yüzünden fuzzy gerekiyor. Bu da tutmazsa tüm transkript taranır. Alıntı orada bulunursa ref **düzeltilir** (zaman damgası tamiri); bulunmazsa `verified:false` işaretlenir. Ekran alıntıları OCR metnine karşı aynı yöntemle doğrulanır.
3. **Karar:** Hiç doğrulanmış kanıtı olmayan item düşürülür ve `log.md` dosyasına yazılır. Videonun `verified_ratio` değeri 0.8'in altındaysa bir kez daha güçlü bir modelle çalıştırılır, yine düşükse `review: pending` işaretlenir.
4. **Zaman:** İlk doğrulanmış segmentin `start − 1 sn` değeri kullanılır. Kelime düzeyinde zaman damgası varsa alıntının ilk kelimesine hizalanır. Link biçimleri: YouTube için `https://www.youtube.com/watch?v=ID&t=192s` (Shorts da bu formatla açılır). Instagram Reel'lerde zaman parametresi olduğu **doğrulanamadı**, bu yüzden düz URL ve metin içinde "(0:12)" kullanılır.
5. **Kaynak normalizasyonu:** URL kanonikleştirilir, kısaltılmış link çözülür, GitHub için `owner/repo` biçimine indirilir ve API ile var mı, arşivli mi, son push ne zaman, kaç star diye kontrol edilir (`repo_status`).

### Kısa ve uzun içerik
- **Reel/Short (≤90 sn):** Aynı şema kullanılır, ama ucuz modelle (Haiku 5.5) çalıştırılır ve 0–3 item beklenir. Çoğu zaman konuşma yoktur (müzik + metin overlay), dolayısıyla birincil kaynak `<screen>` olur. Tekrar eden tavsiyeler frekans sinyali için değerlidir.
- **Uzun tutorial:** Transkript ~60k tokene kadar (yaklaşık 3 saat) **tek çağrıda** işlenir; 1 saatlik bir video ~15–25k token tutar. Daha uzun içerik chapter'lara göre ~40 dakikalık, 2 dakika örtüşmeli pencerelere bölünür. Her pencere ayrı çıkarılır, sonra kanonik slug ile deterministik dedupe ve tek bir ucuz birleştirme çağrısı yapılır.

---

## 4) Kanal sentezi ve artımlı güncelleme

### Kanonikleştirme (entity resolution)
- **Resource:** deterministik. Normalize URL, `owner/repo` ve ad+alias haritasıyla eşlenir; LLM gerekmez.
- **Concept, framework, playbook, tactic:** Extractor zaten registry'ye bakarak slug önerir. `new:` önerileri için registry'ye karşı embedding top-3 benzerlik hesaplanır. Benzerlik ≥0.85 ise **Haiku 5.5** "aynı mı / refines / farklı" kararını verir. GraphRAG'in "aynı başlık + tip → açıklama listesi → LLM tek açıklama" yaklaşımı buradaki rehberdir.
- **Sayfa açma eşiği:** Bir framework ya da playbook ancak en az 2 videoda geçiyorsa **veya** (specificity=3 ve en az 3 adım) koşulunu sağlıyorsa kendi sayfasını alır. Aksi hâlde topic sayfasında madde olarak kalır. Böylece sayfa patlaması önlenir.

### Topic taksonomisi
İlk ≥10 video işlendiğinde tek bir çağrı yapılır: girdi item başlıkları ve önerilen topic'ler, çıktı 6–20 topic'lik (en fazla 2 seviye) bir taksonomi (slug, açıklama, TR/EN alias). Slug'lar **global** tutulur (`kb/_meta/topics.json`), böylece `linkedin-hooks` her kanalda aynı anlama gelir. Sonraki atamalar extractor ve alias haritasıyla yapılır. Hiçbir yere oturmayan item'lar `_unsorted` altına düşer; lint aynı temada en az 3 item birikince yeni topic önerir. HDBSCAN gibi kümeleme algoritmalarına gerek yok (YAGNI).

### Sinyaller (LLM değil, kod hesaplar)
Her kanonik için `support_videos`, `first_seen`, `last_seen`, host/guest dağılımı, `verified_ratio`, `max_specificity` tutulur. Sayfada `★4 video · 2024-11→2026-08` biçiminde gösterilir. LLM'in kendi beyan ettiği "confidence" değerini kalibre olmadığı için **kullanmıyoruz**.

### Çelişki ve evrim
Sayfa derleyicisine item'lar tarih sırasıyla, speaker bilgisiyle birlikte verilir. Derleyici iki çıktı üretir: sayfa ve `relations[]` (refines | contradicts | supersedes). Kurallar:
- Taktik veya platforma duyarlı tavsiyede **yeni olan kazanır**. Eski item `superseded` olur ve sayfanın "Evolution" bölümünde tarihleriyle birlikte görünür.
- İlke düzeyinde çatışma varsa her iki görüş de "Contested" olarak sunulur.
- Host ile konuk çeliştiğinde host'un görüşü "kanal görüşü" sayılır.

### Kanal tezi (`overview.md`)
Bölümler:
- **Thesis:** en az 3 videoda tekrar eden 3–5 temel inanç.
- **Signature frameworks**
- **What this channel says NO to:** `polarity: dont` item'ları; persona'nın "anti-tavsiye" sesi buradan gelir.
- **Assumed context:** kitle, pazar, aşama. Tavsiyenin geçerlilik koşulu budur.
- **Coverage map:** topic başına item ve video sayısı.
- **Blind spots:** komşu ama kanalın işlemediği konular; kapsam dışı sorular için kullanılır.
- **Freshness**

### Artımlı akış (yeni video veya batch geldiğinde)
1. Extract (registry enjekte edilmiş) → doğrula → `items.jsonl` dosyasına ekle → `videos/<id>.md` yaz (şablon + LLM özeti).
2. Kanonikleştir → registry'yi güncelle → **dirty set** çıkar: item'ların topic, framework ve playbook sayfaları, `resources.md`, `glossary.md`, `index.md`.
3. Dirty sayfaları **o sayfanın tüm item'larından** yeniden derle. Derleme stateless'tır ve kaymaz; bir batch içinde her sayfa yalnızca bir kez derlenir. Sayfa 60 item'ı aşarsa önce item'lar özetlenir (map), sonra sayfa derlenir (reduce).
4. `overview.md`: yeni videolar toplamın %10'unu aştığında veya aylık olarak yeniden derlenir. Tez değişiklikleri `log.md` dosyasına yazılır.
5. `index.md` kodla üretilir (frontmatter'daki tek satırlık özetlerden).
6. Lint (her 20 videoda veya haftalık): çelişkiler, bayat item'lar, yetim sayfalar, kırık wiki linkleri, doğrulanmamış ref'ler, öneri bekleyen `_unsorted` kümeleri. Ardından `git commit -m "ingest(@kanal): 3 video"`.
7. Extractor prompt sürümü değişirse yalnızca etkilenen videolar yeniden çıkarılır (`extractor` alanındaki prompt hash'ine göre).

### Kanallar arası sayfalar
Bir topic slug'ı en az 2 kanalda varsa `kb/_topics/<slug>.md` üretilir. Girdisi ham item'lar değil, kanalların **topic sayfaları**dır (ucuz). Bölümler: kanal başına duruş tablosu, Consensus, Disagreements, "Hangi soru için hangi kanal".

---

## 5) LLM wiki dizin yapısı

Karpathy'nin üç katmanı (immutable `raw/`, LLM'in sahibi olduğu wiki, kuralları koyan şema dosyası) ile `index.md` ve `log.md` konvansiyonlarını aynen alıyoruz.

```
kb/
  AGENTS.md                  # şema: bakımcı-LLM kuralları + persona kullanım kuralları
  _index.md                  # tüm KB'ler: kanal başına 1 satır
  _meta/topics.json          # global topic taksonomisi + alias
  _topics/<slug>.md          # kanallar arası derlenmiş sayfalar
  <platform>/<handle>/       # örn. youtube/linkedin-ile-buyu, instagram/...
    index.md                 # her sayfa + 1 satır özet (ilk okunacak)
    overview.md              # tez, imza framework'ler, "hayır dediği", bağlam, kapsam, tazelik
    topics/<slug>.md
    frameworks/<slug>.md
    playbooks/<slug>.md
    resources.md             # araç/repo/kitap tablosu (stance, repo_status, kanıt)
    glossary.md
    videos/<video-id>.md     # video notu
    log.md                   # append-only: "## [2026-10-20] ingest | <başlık>"
    _data/                   # makine katmanı (agent normalde okumaz, arama indeksler)
      items.jsonl  registry.json  relations.jsonl  overrides.yaml
      extracts/<video-id>.json
raw/<platform>/<handle>/<video-id>/{meta.json, transcript.json, visual.json}   # immutable
```

Sayfa bütçeleri: `overview` ≤1.5k token, `topic` ≤2k, `playbook` ≤1k, `video` ≤0.8k (60 dakikalık videoda ≤2.5k). Her derlenmiş madde gizli bir item referansı taşır (`<!-- yt-Q7xK2mP9aLs#i03 -->`). Lint bunu kullanarak "her madde doğrulanmış bir item'a bağlı mı" kontrolünü otomatik yapar. İnsan düzenlemeleri `<!-- human:start --> … <!-- human:end -->` blokları arasında durur ve derleyici bunlara dokunmaz. Yanlış çıkarılan item'ı bastırmak için `overrides.yaml` kullanılır.

### Örnek 1: `videos/yt-Q7xK2mP9aLs.md` (kurgusal kanal)
```markdown
---
type: video
id: yt-Q7xK2mP9aLs
url: https://www.youtube.com/watch?v=Q7xK2mP9aLs
channel: linkedin-ile-buyu
title: "LinkedIn'de ilk 2 satır: 5 hook kalıbı"
published: 2026-03-14
duration_s: 742
format: long
lang: tr
content_type: tutorial
density: high
topics: [linkedin-hooks, linkedin-post-format]
items: 9
verified_ratio: 1.0
extractor: v0.3/sonnet-5.5/1a2b3c
---
# LinkedIn'de ilk 2 satır: 5 hook kalıbı
**TL;DR:** "See more" tıklamasını ilk iki satır belirler; kanal 5 hook kalıbı, ≤12 kelimelik ilk satır ve fikir başına 3 hook varyantı öneriyor.

- **[framework]** 5 hook kalıbı (itiraz, sayı, ortadan başlayan hikâye, hata itirafı, sonuç) → [[frameworks/five-hook-patterns]] · [▶ 01:05](https://www.youtube.com/watch?v=Q7xK2mP9aLs&t=65s)
  > "İlk satırda ya bir itiraz, ya bir sayı, ya da bir hata itirafı olmalı."
- **[tactic·do]** İlk satır ≤ ~12 kelime; 2. satır merak boşluğu açar · [▶ 03:40](https://www.youtube.com/watch?v=Q7xK2mP9aLs&t=220s)
- **[claim]** Hook'ta sayı olan postlar ~2x "see more" aldı (kendi testi, n=40) · [▶ 06:12](https://www.youtube.com/watch?v=Q7xK2mP9aLs&t=372s)
- **[tool·recommended]** AuthoredUp: mobil kesilme noktasını önizleme · [▶ 09:30](https://www.youtube.com/watch?v=Q7xK2mP9aLs&t=570s)

**Chapters:** 00:00 Neden ilk 2 satır · 01:05 5 kalıp · 06:12 Kendi testim · 09:30 Araçlar
```

### Örnek 2: `playbooks/write-linkedin-hook.md`
```markdown
---
type: playbook
slug: write-linkedin-hook
channel: linkedin-ile-buyu
topics: [linkedin-hooks]
support: { videos: 4, first: 2024-11-02, last: 2026-08-21 }
status: current
platform_sensitive: true
review: none
updated: 2026-10-20
---
# Write a LinkedIn hook (channel method)
**When:** every text post; matters most for reaching non-followers. **Outcome:** higher "see more" rate.
1. Write the body first, the hook last. [▶ Q7x 02:10](…&t=130s) <!-- yt-Q7x…#i02 -->
2. Draft 3 hooks from different patterns in [[frameworks/five-hook-patterns]]. [▶ Q7x 01:05](…) [▶ ig C9x 0:08](https://www.instagram.com/reel/C9x…/)
3. Line 1 ≤ ~12 words, no emoji; line 2 opens a curiosity gap. [▶ Q7x 03:40](…)
4. Preview the mobile cut-off in AuthoredUp before posting. [▶ Q7x 09:30](…)
5. After 24h judge by impressions from non-followers, not likes. [▶ Lm3 04:55](…)
**Don't:** "Did you know…?" question hooks; channel says they erode trust. [▶ Hh7 07:20](…)
**Evolution:** 2024-11: "start every post with a question" [▶ Ab1 03:00](…) → **superseded** 2026-08 (explicitly reversed in [▶ Lm3 02:10](…)).
```

### Örnek 3: `overview.md` (kısaltılmış)
```markdown
# @linkedin-ile-buyu — overview  (68 videos · 2023-05→2026-09 · TR · B2B solo founders, TR/EU market)
## Thesis
1. Distribution beats polish: post daily-ish, iterate on hooks, not on design. ★14 videos
2. Comments on others' posts are the cheapest growth lever before 1k followers. ★9
## Signature frameworks
[[frameworks/five-hook-patterns]] ★6 · [[playbooks/comment-before-you-post]] ★5
## Says NO to
Engagement pods; carousels for brand-new accounts; buying followers. ★7
## Blind spots
Paid LinkedIn ads, Sales Navigator workflows → out of scope.
## Freshness
3 topics are platform-sensitive; last algorithm-related content: 2026-08.
```

---

## 6) Persona agent talimatı taslağı (SKILL.md veya MCP `instructions`)

```text
You are an expert advisor grounded in the knowledge base of {CHANNEL} ({PLATFORM}).
You are NOT the creator; never speak as them ("According to this channel…", not "I…").

START OF SESSION: read kb/{p}/{h}/overview.md and index.md (≈4k tokens). They tell you
the channel's thesis, assumed context, signature frameworks and blind spots.

ROUTING:
- Strategy / "what should I focus on" → overview + the relevant topics/*.md
- "How do I…" → search kind=playbook|framework first; answer as numbered steps
- "Which tool/repo…" → resources.md (respect stance: sponsored ≠ recommended; check repo_status)
- "What did they say about X / in which video" → kb_search over items, then open videos/*.md
- Several channels / comparison → kb/_topics/<slug>.md, then each channel's topic page
Search before you say "not covered". Read pages before quoting them.

GROUNDING & CITATIONS:
- Every channel-derived statement carries a citation: [Video title @ mm:ss](url&t=Ns).
  Instagram: plain URL + "(m:ss)".
- Only quote verbatim text returned by kb tools (use kb_get_segment to check); never
  reconstruct quotes from memory. Keep Turkish quotes in Turkish; answer in the user's language.
- Label anything not from the KB as "General knowledge (not from this channel):".
- Distinguish host vs guest opinions; mention support strength (★ videos) when it matters.

FRESHNESS & CONFLICTS:
- If an item is platform_sensitive and older than 12 months, or marked superseded,
  say so and prefer the newest position; show the evolution briefly when relevant.
- If the channel contradicts itself or another channel, present both with dates.

OUT OF SCOPE: if the KB has nothing relevant, say so plainly, check overview "Blind spots",
suggest another KB if one fits, and only then offer clearly-labelled general knowledge.

ANSWER SHAPE (default): 1-2 sentence answer → steps/playbook tailored to the user's
context (ask ONE clarifying question if context such as audience/stage is missing and
changes the advice) → why (channel's reasoning) → sources list. Be concrete; no filler.
```

Persona varyantları (LinkedIn büyüme uzmanı, B2B satış koçu, mühendislik mentoru) bu metne yalnızca 3–5 satırlık bir "rol odağı" ekler. Örneğin mühendislik mentoru `resources.md` dosyasındaki repolara ve `repo_status` alanına öncelik verir.

---

## 7) Kalite ve eval

**Otomatik kontroller (her ingest'te):**
- Şema ve ref var mı kontrolü.
- **Alıntı doğrulama oranı:** hedef ≥%95; videonun oranı <%80 ise yeniden çalıştırılır.
- Zaman damgası süre içinde mi, monoton mu.
- Derlenmiş sayfadaki her madde doğrulanmış bir item'a bağlı mı (gizli ref'ler sayesinde).
- Wiki link ve URL kontrolü, GitHub repo var mı / arşivli mi.
- Frontmatter geçerli mi.
- "Specificity=1 oranı" alarmı: klişe sızıyorsa uyarır.

**Örneklemeli LLM yargıcı:** Item'ların %5'i, extractor'dan *farklı* bir modelle "statement_en, alıntı + ±30 sn bağlamdan çıkarılabilir mi" (entailment) sorusuyla değerlendirilir. Aynı yargıç, çevrilen sayı ve koşullarda kayma olup olmadığına da bakar.

**Tazelik:** `platform_sensitive` alanı ve `last_seen` tarihi kullanılır. 12 ayı geçen item sayfada ⚠ ile işaretlenir; ayrıca repo son push tarihi izlenir.

**Versiyonlama:** Her ingest batch'i bir git commit'idir. Video notunda `extractor` (sürüm/model/prompt hash) tutulur. `log.md` append-only'dir.

**İnsan inceleme kancaları:** Çelişki veya düşük doğrulama varsa `review: pending` işaretlenir ve `REVIEW.md` kuyruğu oluşur. Okuma ortamı Obsidian'dır (graph view yetim sayfaları gösterir). Kalıcı insan düzeltmeleri `overrides.yaml` dosyasında ve human blokları içinde tutulur.

**Eval seti (kanal başına 20 soru-cevap):**

| Kategori | Adet | Ölçülen |
|---|---|---|
| Olgusal hatırlama ("hook için kaç kelime?") | 5 | doğruluk, atıf hassasiyeti |
| How-to / playbook | 4 | adım kapsamı, uygulanabilirlik (1–5) |
| Araç/repo bulma | 3 | doğru kaynak + stance |
| Videolar arası sentez | 2 | tamlık, çoklu atıf |
| Çelişki / evrim | 2 | güncel pozisyon + tarih |
| Kapsam dışı (beklenen: çekimser) | 3 | doğru çekimserlik, uydurma yok |
| Konuk ve host ayrımı | 1 | doğru atfetme |

Gold veri her soru için cevap ve gold atıflardan (video + zaman aralığı) oluşur. Metrikler: atıf hassasiyeti (atıf yapılan segment iddiayı destekliyor mu), atıf recall'u, zaman damgası hatası (≤15 sn), çekimserlik doğruluğu, groundedness (yargıç) ve aksiyon puanı. Üretim yarı otomatiktir: LLM item'lardan soru taslağı çıkarır, kullanıcı kanal başına yaklaşık 20 dakikada onaylar. Ayrıca 10 soruluk bir kanallar arası set hazırlanır. Eval prompt/extractor/model her değiştiğinde regresyon olarak çalışır.

---

## 8) Boyut ve çağrı tahminleri (maliyet analisti için)

Varsayımlar: konuşma ~130–150 kelime/dk. Claude ≥4.7 tokenizer'ı aynı metin için ~%30 daha fazla token üretiyor; bu, EN için ≈ 250–300 token/dk eder. TR için bu değer **doğrulanamadı**; Türkçe eklemeli bir dil olduğu için daha yüksek olması bekleniyor ve `count_tokens` ile ölçülmeli. Prompt + registry ~4k tokendir ve cache'lenir (Sonnet 5.5 için cache okuma maliyeti 0.05x).

| Format | Girdi (transkript + ekran + açıklama) | Çıkış JSON | Item | Video notu |
|---|---|---|---|---|
| Reel/Short 30–60 sn | 0.3–0.8k | 0.3–0.8k | 0–3 | 0.2–0.4k |
| 10–15 dk | 3–6k | 1.5–3k | 5–15 | 0.6–1k |
| 60 dk tutorial | 15–25k | 5–9k | 15–40 | 1.5–2.5k |

**Çağrılar:**
- Yeni video başına (artımlı) ≈ **6 çağrı**: 1 extraction, ~0.15 tamir, ~0.3 kanonik karar (Haiku), 4–6 sayfa derlemesi (batch içinde paylaşılır), ~0.1 overview (amortize).
- **İlk kurulum, 100 video (ort. 12 dk):** 100 extraction (≈0.7M giriş + 0.4M cache, 0.22M çıkış) + ~45 sentez çağrısı (taksonomi 1, topic ~15, framework/playbook ~25, resources/glossary 2, overview 1, lint 1–2; ≈0.5M giriş, 0.09M çıkış) = **~145 çağrı, ~1.2M giriş / ~0.31M çıkış**.

**Maliyet** (fiyat sayfası, 2026-10; $/MTok giriş/çıkış: Sonnet 5.5 2/10, Haiku 5.5 0.10/0.50 [≤100k prompt], Opus 5.5 4/20; Batch −%50):
- 100 video, hepsi Sonnet 5.5: ≈ $5.5; Batch ile ≈ $2.8.
- Hepsi Haiku 5.5: ≈ $0.3.
- **Önerilen karışım:** uzun videolarda extraction ve sentez Sonnet 5.5, Reel'ler ve kanonik kararlar Haiku 5.5. Toplam ≈ $3–5; overview için Opus 5.5 kullanılırsa +$0.1.
- 1000 videoluk kanal ("tümü"): ≈ $30–50, Batch ile ≈ $15–25. Sentez maliyeti alt-doğrusal artar.
- Yeni video başına: ≈ $0.03–0.08.
- Görsel çıkarım (vision) maliyeti dahil değil (Persona 3).

**KB boyutu (100 video):** video notları ~80k, `items.jsonl` ~250k, derlenmiş sayfalar ~50 sayfa × ~1k ≈ 50k token. Agent oturum başında yalnızca ~4–5k token okur.

---

## 9) Diğer personalara notlar

- **P1 (Storage/Retrieval):** Üç indeks birimi var: **items** (birincil), **sayfalar** ve **segmentler** (birebir alıntı için yedek). Filtre alanları: channel, kind, topic, published_at, speaker, stance, verified, platform_sensitive. TR+EN için **çok dilli embedding** şart (qmd'nin varsayılan modeli değiştirilebilir). Chunk'lara deterministik bağlam başlığı (`Kanal / Video / Chapter / mm:ss`) eklenmeli; bu, Contextual Retrieval fikrinin ucuz bir versiyonu. Doğruluk kaynağı `items.jsonl` olmalı, Markdown sayfalar türetilmiş. Embedding'ler kanonikleştirmede de kullanılıyor.
- **P2 (Acquisition):** Gerekenler: kararlı video ID'leri, `published_at`, segment düzeyinde (tercihen kelime düzeyinde) zaman damgaları, caption dili, chapters, açıklama + sabit yorum + linkler, başlık/açıklamadan konuk adı. Instagram'da ayrıca caption metni, carousel slayt sırası ve sesi olmayan Reel'lerin işaretlenmesi.
- **P3 (Visual):** Çıktı `v####` ID'li, zamanlı ve `type ∈ {slide, code, terminal, url, repo, diagram, overlay}` olmalı. OCR metni **birebir** kalmalı, çünkü ekran alıntıları ona karşı doğrulanıyor. Aynı kare tekrarları elenmeli. Reel'lerde metin overlay'leri birincil kaynak.
- **P4 (MCP/Agent):** Önerilen araçlar: `kb_list_channels`, `kb_overview(channel)`, `kb_search(query, channel?, kind?, topic?, since?)`, `kb_read(path)`, `kb_get_segment(video_id, t, window)`. Arama sonuçları API tarafında `search_result` blokları olarak döndürülebilir; `source` alanı zaman damgalı URL'dir ve Claude bunlarla yerel atıf yapar. Bu özelliğin Desktop/claude.ai MCP istemcilerinde nasıl render edildiği **doğrulanamadı**. Persona metni (Bölüm 6) Skill ya da MCP `instructions` olarak dağıtılabilir.
- **P5 (Cost):** Bölüm 8. Görsel çıkarım ve agent sorgu maliyeti ayrıca eklenmeli.

---

## 10) Kaynaklar

- Karpathy, "LLM Wiki" idea file (gist, 2026-04-04): https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f. Raw / wiki / şema katmanları, ingest / query / lint operasyonları, `index.md` ve `log.md`, "~100 kaynak / yüzlerce sayfaya kadar embedding RAG gerekmeden", qmd ve Obsidian.
- Karpathy, "LLM Knowledge Bases" X postu (2026-04 başı): https://x.com/karpathy/status/2039805659525644595. Link VentureBeat üzerinden alındı; x.com içeriği doğrudan **doğrulanamadı**.
- VentureBeat, "Karpathy shares LLM knowledge base architecture…": https://venturebeat.com/data/karpathy-shares-llm-knowledge-base-architecture-that-bypasses-rag-with-an ("You rarely ever write or edit the wiki manually; it's the domain of the LLM.")
- Fabric patterns (commit 0511907, 2026-10-20): https://github.com/danielmiessler/Fabric/tree/main/data/patterns. İncelenenler: `extract_wisdom`, `extract_wisdom_with_attribution`, `extract_instructions`, `extract_recommendations`, `extract_references`, `extract_sponsors`, `extract_affiliate_products`, `create_video_chapters`, `youtube_summary`, `analyze_claims`.
- basic-memory (markdown gözlem/ilişki sözdizimi, MCP, hybrid arama): https://github.com/basicmachines-co/basic-memory
- qmd (BM25 + vektör + RRF + rerank, MCP, markdown): https://github.com/tobi/qmd
- Anthropic, Reduce hallucinations: https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations
- Anthropic, Prompting best practices (long context, XML, quote grounding): https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- Anthropic, Structured outputs (şema kısıtları): https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- Anthropic, Search result content blocks: https://platform.claude.com/docs/en/build-with-claude/search-results
- Anthropic, Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Anthropic, Contextual Retrieval: https://www.anthropic.com/news/contextual-retrieval
- Microsoft GraphRAG default dataflow (açıklama birleştirme, claim/covariate, community reports): https://microsoft.github.io/graphrag/index/default_dataflow/
- Instagram Reel zaman damgalı deep link: **doğrulanamadı** (destek görünmüyor; test edilmeli).
