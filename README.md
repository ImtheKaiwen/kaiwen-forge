# Kaiwen Forge

> **Modüler AI Model Geliştirme, Eğitim Orkestrasyonu, Deney Takibi, Değerlendirme ve Model Kayıt Altyapısı**<br>
> **Repository:** [`ImtheKaiwen/kaiwen-forge`](https://github.com/ImtheKaiwen/kaiwen-forge)<br>
> **Belge türü:** Proje README + V1 geliştirici başvuru rehberi<br>
> **Belge durumu:** `Proposed / implementation blueprint`<br>
> **İlk hedef entegrasyon:** **Kaiwen Speech**<br>
> **Güncelleme:** 30 Ekim 2026

> **D1 belge entegrasyonu:** Upstream başlangıç README'sindeki tanım — “Modular AI model development toolkit for training, experimentation, evaluation, and model management.” — ürün niyeti olarak korunur. İki satırlık orijinal dosya [`README.upstream-20261030.md`](README.upstream-20261030.md) yolunda byte-for-byte mevcuttur. Bu ana README'nin geri kalanı **PROPOSED / implementation blueprint** niteliğindedir; Forge kodu, CLI'sı, registry'si veya `kaiwen.model.release` validator'ı bugün uygulanmış sayılmaz.

> [!IMPORTANT]
> **Mevcut durum ile hedef mimari birbirinden farklıdır.** 30 Ekim 2026 tarihinde GitHub'daki `kaiwen-forge` reposu yalnızca kısa bir `README.md` içeriyor. Bu dosyadaki `kaiwen_forge` Python modülleri, `forge` CLI komutları, `recipes/`, `registry/` ve entegrasyon adaptörleri **henüz mevcut değil**. Aşağıdaki bazı örnekler açıkça **HEDEF / ÖNERİ** olarak gösterilir. Bunları bugün çalıştırılabilir komut sanmayın. Var olan eğitim çalışmaları [`kaiwen-speech`](https://github.com/ImtheKaiwen/kaiwen-speech) reposundadır. **Bu README, uygulanmış özelliklerin belgesi değil; Forge V1'in uygulama ve kabul sözleşmesidir.**

---

## 1. Forge neden var?

Bir modelin mimarisini yazmak ile o model üzerinde bilimsel ve tekrarlanabilir deneyler yürütmek farklı sorumluluklardır. Araştırma büyüdükçe aynı sorular yeniden ortaya çıkar:

- Hangi veri sürümü, codec sürümü ve model konfigürasyonu ile eğitim yaptık?
- Deneyin hangi commit'ten çalıştırıldığını ve hangi Python/CUDA ortamında yürütüldüğünü biliyor muyuz?
- Eğitim yarıda kalırsa hangi checkpoint güvenle devam ettirilebilir?
- Google Colab oturumu sona ererse hangi dosyalar gerçekten kaydedildi?
- Bir modelin daha düşük `loss` üretmesi, daha doğal Türkçe konuştuğunu kanıtlar mı?
- İki model arasında aynı test seti, donanım ve çıkarım ayarlarıyla karşılaştırma yapıyor muyuz?
- Windows'taki Kaiwen Live uygulamasına hangi model dosyasının **uyumlu ve onaylanmış** olduğunu nasıl söylüyoruz?
- Yeni bir model ailesi eklendiğinde bütün eğitim altyapısını baştan yazacak mıyız?

**Kaiwen Forge bu soruların ortak altyapısını sahiplenir.** Forge'un temel görevi kod yazan model araştırmacısını, güvenilir sonuç üreten bir deney sistemine taşımaktır. Bu amaca ulaşmak için ilk sürümde ağır bir dağıtık cluster veya genel amaçlı no-code makine öğrenmesi platformu kurmak gerekmiyor. İlk hedef, **tek geliştiricinin** kendi cihazında ya da hosted Colab ortamında aynı reçete ile deneyi başlatıp, durumu takip edip, sonucu ölçüp, gerektiğinde geri dönebilmesidir.

### 1.1 Bir cümlelik tanım

**Kaiwen Forge; modeli yeniden uygulamadan, model geliştirme işlerini sürümlü reçeteler, tekrarlanabilir çalıştırmalar, bütünlüğü doğrulanmış artifact'ler, karşılaştırılabilir metrikler ve kontrollü model yayınları etrafında birleştiren bağımsız bir Python araç setidir.**

### 1.2 Proje kimliği

- **Repository:** `kaiwen-forge` — ayrı Git deposu, ayrı paket ve sürüm yaşam döngüsü.
- **Dil:** Python 3.11+ hedef; somut desteklenen sürüm `pyproject.toml` ve CI eklendikten sonra kesinleşecek.
- **İlk platformlar:** Windows geliştirici bilgisayarı, Linux CPU CI, opsiyonel CUDA GPU, hosted Colab.
- **Depolama:** Önce yerel dosya sistemi; aktarım/senkronizasyon için Google Drive ve gerektiğinde başka artifact backend'i.
- **İlk model adaptörü:** `kaiwen-speech`; gelecekte yeni model ailesi eklemeye **uygun**, ama V1'de diğer ailelerin hayali implementasyonları yok.
- **Arayüz önceliği:** Python API + basit CLI; web paneli, dağıtık görev kuyruğu, Kubernetes, ücretli takip servisleri ve çok kullanıcılı rol sistemi V1 zorunluluğu değil.
- **Ürün yaklaşımı:** Ölçülebilir, bakımı kolay, az bağımlılıklı, yerelde çalışabilir, otomasyona uygun.

---

## 2. Güncel durum: ne var, ne yok?

Bu README'nin hazırlandığı tarihte repo kontrolü:

| Alan | Durum | Kanıt / açıklama |
|---|---|---|
| `kaiwen-forge` GitHub repository | **Mevcut** | `main` dalı ve kısa bir başlangıç README'si var. |
| Forge Python paketi (`kaiwen_forge`) | **Planlanan** | Henüz `src/`, `pyproject.toml` ve yayınlanmış Forge API'si yok. |
| Forge CLI (`forge`) | **Planlanan** | Aşağıdaki CLI sözdizimi tasarım önerisi. |
| Experiment run kayıtları | **Planlanan** | Dosya/SQLite tabanlı V1 izleme uygulanacak. |
| Speech training adapter | **Planlanan** | Gerçek Speech CLI/API ile uyum testinden sonra yazılacak. |
| Model registry, promotion | **Planlanan** | Release manifest, checksum, durum geçişleri tasarlanacak. |
| Artifact storage / Drive sync | **Planlanan** | Güvenilir, devam ettirilebilir aktarım V1+ kapsamında. |
| Forge CI/CD | **Planlanan** | Test, lint, type check, build ve güvenlik kontrolleri eklenecek. |
| Speech model ve eğitim altyapısı | **Başka repoda mevcut** | `kaiwen-speech` içinde gerçek kod ve testler bulunuyor. |

### 2.1 Var olan Kaiwen Speech altyapısı

Forge'un ilk kullanıcısı olacak `kaiwen-speech` içinde şu bileşenler bugün kod olarak bulunuyor:

- `src/kaiwen_voice/codec/base.py`: soyut `SpeechCodec` arayüzü ve `CodecMetadata`.
- `src/kaiwen_voice/codec/mimi_codec.py`: donmuş Mimi adaptörü; ilk araştırma baseline'ı.
- `src/kaiwen_voice/model/`: causal Transformer; parallel ve deneysel depth codebook modelleri.
- `src/kaiwen_voice/training/trainer.py`: AdamW, gradient clipping, opsiyonel warmup/cosine, validation ve checkpoint akışı.
- `src/kaiwen_voice/training/checkpoint.py`: model ve eğitim durumu kaydı/yükleme.
- `src/kaiwen_voice/data/token_cache.py`: `.pt` token shard'ları, codec metadata doğrulaması.
- `scripts/train_continuation.py`, `scripts/train_dialogue.py`: mevcut eğitim CLI'ları.
- `scripts/benchmark.py`: **dar kapsamlı** tam-forward performans ölçümü; gerçek streaming benchmark'ı değil.
- `.github/workflows/test.yml`: Speech reposuna ait lint/type/test CI.

**Dikkat:** Git repo adı `kaiwen-speech` olsa da bu kaynakların gerçek import adı `kaiwen_voice`, PyPI dağıtım tanımı ise bugün `kaiwen-voice` olarak geçiyor. Forge V1 bu gerçeğe göre adapter geliştirir; yalnızca repo adına bakıp `import kaiwen_speech` yazamaz.

### 2.2 Neler bugünkü Speech kabiliyeti değildir?

Kendi Kaiwen Codec V1, üretime hazır streaming inference SDK, GQA/KV-cache tabanlı jenerasyon ve 2 GB VRAM'de doğrulanmış uçtan uca sesli konuşma **henüz uygulanmış sonuçlar değildir**. Forge araştırma planına bunları yazabilir, ancak çalıştırma kayıtlarına `completed`/`verified` diyebilmesi için gerçek test ve artifact kanıtı gerekir.

---

## 3. Kaiwen AI ekosistemindeki konumu

Yerel geliştirme alanı `kaiwen-ai/` adında bir klasör olsa da Git monorepo **değildir**. Altındaki altı proje bağımsız Git repository'sidir:

```text
kaiwen-ai/                             # Yerel workspace, .git yok
├── AGENTS.md                          # Bütün projeler için Codex kuralları
├── README.md                          # Genel giriş
├── docs/
│   ├── OVERVIEW.md
│   ├── INTEGRATION.md
│   ├── ENGINEERING_STANDARDS.md
│   └── ROADMAP.md
│
├── kaiwen-corpus/                     # Ses verisini toplar, etiketler ve export eder
├── kaiwen-speech/                     # Codec, model, loss, inference, training primitives
├── kaiwen-forge/                      # BU REPO: eğitim işleri, deneyler, değerlendirme, registry
├── kaiwen-live/                       # Son kullanıcı gerçek zamanlı konuşma uygulaması
├── kaiwen-design/                     # Ortak React UI component'leri
└── kaiwen-agent/                      # Opsiyonel agent, tools ve policy sistemi
```

### 3.1 Entegrasyonun yönü

```text
Corpus → [Sürümlü veri manifesti] ─┐
                                   │
Speech → [Model/codec/trainer API] ├──→ FORGE
                                   │        │
Colab  → [Compute çalışma ortamı] ─┘        ├──→ Metrikler + deney raporu
                                            ├──→ Doğrulanmış checkpoint / artifacts
                                            └──→ Onaylanmış model release manifesti
                                                           │
                                                           ▼
                                                 Live + Speech runtime
```

Bu şema **veri akışı** ile **Python import bağımlılığını** birbirine karıştırmaz. Corpus, Forge'u import etmek zorunda değildir. Speech, Forge'u import etmez. Forge, Speech'in model-özel primitive'lerini bir adapter üzerinden kullanır; Live runtime için modelin algoritmik yükleyicisi yine Speech'in kontrolündedir.

| Repo | Sahip olduğu sözleşme | Forge'un sorumluluğu |
|---|---|---|
| **Corpus** | Ham ses, provenance, lisans, veri kalite/etiket kayıtları, export formatı | Şemayı ve lisansı doğrulayıp dataset snapshot kimliğiyle deneye bağlamak. |
| **Speech** | Codec, Transformer, model config, training primitives, checkpoint state, inference | Version-pinned adapter ile bu kodu çalıştırmak; kopya model yazmamak. |
| **Forge** | Recipe, run lifecycle, deney metrikleri, karşılaştırmalar, artifact katalogu, promotion | Bunların sürümlü ve izlenebilir tek yetki kaynağı olmak. |
| **Live** | Mikrofon, ses oturumu, uygulama UI, kullanıcı tercihleri | Live için uyumlu model yayınını ve kanıtını sunmak; Live'ın UI koduna bağlanmamak. |
| **Design** | Paylaşılan UI token/component paketleri | Forge V1 CLI için zorunlu bağımlılık değildir; ileride GUI gelirse tüketici olabilir. |
| **Agent** | Tools, izinler, reasoning, agent runtime | Forge'un V1 eğitim scheduler'ı ile karıştırılmaz. Opsiyonel entegrasyon ileride olabilir. |

**Sert sınır:** Forge **codec implementasyonu**, Speech Transformer katmanları, ses synthesis algoritması veya Agent tool policy yazmaz. Bunların yeri ilgili bağımsız projelerdir.

---

## 4. Temel çalışma ilkeleri

1. **Mevcut kodu tekrar kullan:** Kod zaten Speech'te varsa Forge'a taşımak yerine paket/CLI adaptörü üzerinden çağır.
2. **Reçete ve gerçek çalıştırmayı ayır:** Bir YAML dosyasının varlığı eğitim yapıldığı anlamına gelmez; `run` kanıtı gerekir.
3. **Sürümü sabitle:** Dataset, codec, model paketi, Git commit, config ve metrik şeması kayıt altına alınır.
4. **Başarısızlığı görünür yap:** Kesilen iş hiçbir zaman sessizce `completed` durumuna geçmez.
5. **Veriyi değiştirme:** Corpus'un immutable ham sesini Forge kendi adına overwrite etmez.
6. **Gizli indirme/hesaplama yok:** Komut hangi veriyi indirdiğini, nerede çalıştığını ve hangi maliyeti doğurabileceğini gösterir.
7. **Önce küçük kanıt, sonra ölçek:** Synthetic CPU smoke → küçük gerçek veri → tek Colab deneyi → daha geniş deney matrisi.
8. **Operasyonel sadelik:** İlk sürüm yerel, tek kullanıcı ve küçük sayıda eşzamanlı deney için anlaşılır olmalı.
9. **Modeli kaliteyle değerlendir:** Loss ve perplexity yeterli değil; ses anlaşılırlığı, doğal Türkçe konuşma ve gecikme ayrı ölçülür.
10. **Üretim ve araştırmayı ayır:** Bir `experiment` checkpoint'i otomatik olarak Live'a dağıtılabilecek `release` değildir.

---

## 5. Kullanıcı yolculuğu: ilk çalışan deney

İlk gerçek dikey kesitte hedef akış:

1. Corpus'ta lisansı ve orijinali doğrulanmış küçük bir Türkçe ses veri kümesi oluşturulur.
2. Dataset export'u sabit kimlik, sürüm, kayıt/split bilgileri ve byte hash'leriyle yayımlanır.
3. Forge, dataset'i staging alanına alır; schema, checksum, lisans ve train/val ayrımını kontrol eder.
4. Forge, seçilen `kaiwen-speech` commit/package sürümünü ve codec kimliğini doğrular.
5. Recipe + seed + dataset hash + code version birleşiminden değişmez bir **run spec snapshot** üretir.
6. Forge, mevcut Speech training primitive'ini/CLI'ını başlatır; stdout/stderr, metrikler ve checkpoint durumunu kaydeder.
7. İş bittikten sonra checkpoint yükleme ve küçük inference smoke yapılır. Yalnızca exit code `0` görmek yeterli değildir.
8. Değerlendirme sonuçları run'a kaydedilir, uygun adaylar karşılaştırılır.
9. İnsan onayıyla model release adayı hazırlanır; doğrulanmış manifest ve artifact dosyaları registry'ye eklenir.
10. Kaiwen Live daha sonra bu release manifestini okuyup Speech SDK aracılığıyla modeli güvenle yükler.

### 5.1 Başarılı ilk demo neye benzer?

- Aynı küçük dataset'in değişmeden kaldığı doğrulanabiliyor.
- Aynı training recipe'si yeniden çalıştırıldığında sonuçlar ölçülebiliyor; determinism sınırları raporlanıyor.
- Her çalışmanın çıktıları farklı klasöre yazılıyor; bir run diğerinin dosyasını ezmiyor.
- Çalışma durdurulduğunda son sağlam checkpoint korunuyor.
- Yanlış codec/dataset eşleşmesi eğitim başlamadan reddediliyor.
- Run ve experiment ID'sinden kaynak dataset, commit, environment ve metrik geçmişine ulaşılabiliyor.
- Model yayınını neyin onayladığı ve hangi kalite ölçütlerinin geçildiği görülebiliyor.

---

## 6. Başlangıç: bugün gerçekten çalıştırılabilenler

### 6.1 Bu repository'yi klonlama

```bash
git clone https://github.com/ImtheKaiwen/kaiwen-forge.git
cd kaiwen-forge
```

**Bugün:** Forge reposu ilk README dışında Python paket altyapısı taşımıyor. Burada `uv sync`, `python -m kaiwen_forge` veya `forge run ...` çalıştırılabilir olarak sunulmuyor. Paket hazırlanıp testler eklendiğinde bu bölüm gerçek kurulum komutlarıyla güncellenecek.

### 6.2 Mevcut Speech baseline'ını bağımsız çalıştırma

Forge uygulanmadan önce var olan çalışma yolunu doğrulamak isteyen geliştirici, **ayrı** Speech reposunda şu komutları kullanabilir:

```bash
# kaiwen-ai/kaiwen-speech/ dizini içinde
uv sync --frozen --extra dev
uv run pytest
uv run ruff check .
uv run pyright

# İndirme gerektirmeyen CPU smoke / synthetic overfit
uv run python scripts/train_continuation.py \
  --config configs/smoke.yaml \
  --synthetic --steps 100 --overfit
```

Bunlar Forge komutları **değildir**. Modelin mevcut eğitim çekirdeğini test eder. Entegrasyon başladığında Forge aynı motoru yeniden yazmadan kontrollü biçimde çalıştıracaktır. Yukarıdaki komutların çalışma sonucu burada ölçülmüş/garanti edilmiş değildir.

### 6.3 İleride kurulması hedeflenen Forge CLI — **henüz çalışmaz**

Aşağıdaki komutlar yalnızca **önerilen V1 kullanıcı deneyimini** gösterir:

```bash
# HEDEF API — Henüz implementasyon yok.
uv sync --extra dev
uv run forge doctor
uv run forge dataset validate datasets/turkish-pilot/dataset.lock
uv run forge recipe validate recipes/speech-continuation-pilot.yaml
uv run forge run start recipes/speech-continuation-pilot.yaml
uv run forge run status <run-id>
uv run forge run resume <run-id>
uv run forge evaluate <run-id> --suite speech-core-v1
uv run forge compare --experiment turkish-speech-pilot
uv run forge registry inspect <model-version>
```

Komut isimleri, API yüzeyi ve argümanlar **ilk CLI ADR/contract testleriyle kesinleştirilecek**. Bu aşamada dosya üretilmesi, komutun gerçek implementasyona sahip olduğu anlamına gelmez.

---

## 7. Önerilen V1 Python paket mimarisi

> **HEDEF KLASÖR YAPISI.** Bunların hiçbiri mevcut GitHub reposunda henüz yoktur.

```text
kaiwen-forge/
├── README.md
├── AGENTS.md
├── pyproject.toml
├── uv.lock
├── .gitignore
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release.yml                 # Yayın yapılması kararlaştırılınca
├── docs/
│   ├── TRAINING_PIPELINE.md
│   ├── EXPERIMENTS.md
│   ├── EVALUATION.md
│   └── MODEL_REGISTRY.md
├── configs/
│   └── forge.local.example.yaml
├── recipes/
│   ├── speech-codec-smoke.yaml
│   ├── speech-continuation-pilot.yaml
│   └── speech-dialogue-pilot.yaml
├── src/
│   └── kaiwen_forge/
│       ├── __init__.py
│       ├── cli.py                      # CLI yüzeyi, opsiyonel
│       ├── config.py                   # Tipli Forge ayarları
│       ├── domain/
│       │   ├── recipes.py
│       │   ├── runs.py
│       │   ├── artifacts.py
│       │   └── evaluation.py
│       ├── experiments/
│       │   ├── store.py               # V1: SQLite + dosya artifact'leri
│       │   └── metrics.py
│       ├── orchestration/
│       │   ├── runner.py
│       │   ├── lifecycle.py
│       │   ├── scheduler.py           # V1 local serial worker
│       │   └── cancellation.py
│       ├── datasets/
│       │   ├── manifest.py
│       │   ├── validation.py
│       │   └── splits.py
│       ├── artifacts/
│       │   ├── store.py
│       │   ├── checksums.py
│       │   └── registry.py
│       ├── evaluation/
│       │   ├── runner.py
│       │   ├── metrics.py
│       │   └── reports.py
│       └── integrations/
│           └── speech/
│               ├── adapter.py
│               ├── compatibility.py
│               └── fixtures.py
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contracts/
│   └── smoke/
└── examples/
    ├── recipes/
    └── reports/
```

### 7.1 Neden bu klasörler?

- `domain/`: Bağımsız, küçük, immutable veri sözleşmeleri ve durum makineleri; dış altyapıdan mümkün olduğunca bağımsız.
- `orchestration/`: Başlatma, yeniden deneme, iptal, sürdürme, sinyal ve process yaşam döngüsü.
- `integrations/speech/`: Speech'e özgü model/codec konfigürasyonları ve gerçek kütüphane/CLI köprüsü.
- `experiments/`: Run, metric, tag, artifact referansı; tarihçe ve sorgular.
- `datasets/`: Corpus export manifesti için şema ve integrity doğrulaması; Corpus'un veri etiketleyicisi burada tekrar yazılmaz.
- `artifacts/`: Checkpoint ve raporun fiziksel konumundan bağımsız katalog/aktarımı.
- `evaluation/`: Farklı değerlendiricileri aynı deneye bağlayan kayıt katmanı; Speech'e özgü ölçüm algoritması uygun backend'de kalır.
- `tests/contracts/`: Repo sınırlarının kırılmasını tespit eden fixture ve şema testleri.

**Aşırı soyutlama uyarısı:** Sadece bir implementasyon için her klasöre interface, factory, plugin manager ve service container doldurulmayacak. Bu ağaç nihai zorunlu mimari değil, büyüdükçe ayrılabilecek modül sınırlarını gösterir. İlk çalışan sürüm daha az dosyayla başlayabilir.

### 7.2 Paket/import sürümleme kararı

`kaiwen_forge` adı hedef olarak öneriliyor. `pyproject.toml` oluşturulmadan **resmî public import veya CLI adı kesinleştirilmiş değildir**. Karar verildiğinde isimler API belgelerinde, kurulum komutlarında, wheel metadata'sında ve CI'de birlikte doğrulanacaktır.

---

## 8. Recipe: bir deneyin tekrarlanabilir tarifi

**Recipe** yalnızca eğitim komut satırı değil; hangi modeli, hangi veriyi, hangi eğitim ayarlarını ve hangi değerlendirme suite'ini kullanacağını tanımlayan sürümlü girdidir. Recipe ve resolved run snapshot birbirinden ayrıdır: kullanıcı tarifi düzenlese bile başlamış deneyin snapshot'ı değişmez.

### 8.1 Önerilen recipe örneği — **şema taslağı**

```yaml
schema_version: "1.0"
recipe_id: "speech-continuation-pilot"
family: "speech"
backend: "kaiwen-speech"

source:
  git_repository: "https://github.com/ImtheKaiwen/kaiwen-speech"
  git_commit: "<resolved-full-commit-sha>"  # Çalıştırmada gerçek SHA ile değiştirilir

input:
  dataset_manifest: "datasets/turkish-pilot/dataset.lock"
  dataset_digest: "<verified-sha256>"        # Gerçek hash zorunludur
  split: "train"
  codec_id: "mimi-baseline"                   # Örnek; model config ile doğrulanır
  codec_revision: "<pinned-codec-revision>"

model:
  config: "../kaiwen-speech/configs/smoke.yaml"
  task: "continuation"

training:
  seed: 42
  max_steps: 100
  precision: "fp32"
  resume_policy: "explicit"

runtime:
  execution: "local"
  device: "cpu"
  max_wall_time_minutes: 30
  output_root: "artifacts/runs"

evaluation:
  suite: "speech-smoke-v1"
  require_checkpoint_load: true
```

Bu YAML **gerçek Forge parser'ında bugün geçerli değildir**. `source.git_commit` ve `input.dataset_digest` gibi placeholder'lar bir üretim çalıştırmasında kabul edilmemelidir. `codec_id` ancak doğrulanmış Speech metadata'sı ile eşleştiğinde geçerlidir; kullanıcı tarafından yazılmış rastgele isim tek başına kanıt sayılmaz.

### 8.2 Recipe doğrulama aşamaları

1. YAML güvenli yüklenir; unknown field politikası baştan tanımlanır. Kritik hatalı alanlar yok sayılmaz.
2. `schema_version` ve `family/backend` destekleniyor mu kontrol edilir.
3. Model config dosyası gerçekten mevcut ve okunabilir mi bakılır.
4. Paket/import/commit sürümü kayda alınır; dirty git worktree varsa kullanıcıya açık uyarı verilir veya politika gereği reddedilir.
5. Dataset export manifesti çözülür; kimlik, hash, lisans ve split doğrulanır.
6. Codec metadata'sı modelin codebook sayısı, vocab ve timebase ile karşılaştırılır.
7. Çalıştırma cihazı ve precision destekleniyor mu kontrol edilir.
8. Output dizininin güvenli, yazılabilir ve önceki run'larla çakışmasız olduğu doğrulanır.
9. `resolved_recipe` ve kullanılan environment bilgisi **iş başlamadan önce** dondurulur.
10. Bütün kontroller geçmeden GPU eğitimi veya büyük indirme başlatılmaz.

### 8.3 Recipe override politikası

`--set training.max_steps=...` benzeri CLI override'ları desteklenirse, override edilmiş değerler gerçek snapshot'ta saklanır. Orijinal YAML ayrı referanslanır. Gizli environment override'ları sonucu değiştiremez; etki eden tüm ayarlar raporlanır. Gizli token, API key ve kişisel veri override geçmişine yazılmaz.

---

## 9. Run, experiment ve study kavramları

| Kavram | Ne demek? | Kimlik |
|---|---|---|
| `recipe` | Neyi, nasıl çalıştırmak istiyoruz? | `recipe_id` + schema sürümü |
| `experiment` | Aynı araştırma sorusuna ait koşuların mantıksal grubu | `experiment_id` |
| `run` | Gerçekten başlatılmış tek deneme; kendi config ve artifact geçmişi var | `run_id` |
| `evaluation` | Sabit test seti, metrik tanımı ve protokolle yapılan ölçüm | `evaluation_id` |
| `artifact` | Checkpoint, log, rapor, ses örneği veya manifest gibi çıktılar | `artifact_id` + SHA-256 |
| `candidate` | Registry için aday gösterilmiş doğrulanmış artifact takımı | `candidate_id` |
| `release` | Uyumlu, onaylanmış, dağıtıma hazır sürümlü model paketi | `model_id` + `model_version` |

### 9.1 Experiment örneği

Araştırma sorusu: *Aynı veri ve codec ile depth codebook modellemesi, parallel baseline'a göre Türkçe ses kalitesini iyileştiriyor mu?*

```text
experiment: tr-depth-ablation-001
  run-A: parallel / baseline-30m / seed-42
  run-B: depth    / baseline-30m / seed-42
  run-C: parallel / baseline-30m / seed-43
  run-D: depth    / baseline-30m / seed-43

Held constant:
  dataset revision + hash
  codec model + revision
  hardware/precision/steps
  evaluation samples + decoding parameters

Variable under study:
  codebook modeling strategy
```

Yalnızca bir parametre değiştirildiği varsayımı, depth modelin parametre sayısını artırabileceği gerçeğiyle çatışabilir. Bu yüzden **gerçek parametre sayısı, eğitim FLOP veya ölçülen training süresi ve inference maliyeti** de rapora eklenir. Her karşılaştırmanın sınırlamaları açıkça belirtilir; performans iddiaları bir deney olmadan ilan edilmez.

### 9.2 Run durum makinesi

V1 önerisi:

```text
created ──validate──> validated ──enqueue──> queued ──start──> running
   │                    │                         │           │
   └──────> rejected     └──────> rejected         │           ├──> completed
                                                   │           ├──> failed
                                                   │           ├──> cancelling ──> cancelled
                                                   │           └──> interrupted ──> resumable/failed
                                                   └───────────────> cancelled
```

- Bir iş `completed` olduğunda gerekli son artifact'ler, metrik çıktıları ve finish marker doğrulanmış olmalı.
- `failed`, `interrupted` ve `cancelled` ayrı durumlardır. Kullanıcı iptali hata gibi gizlenmez.
- `interrupted`, yeni cihaz/oturumda otomatik olarak devam edildiği anlamına gelmez; `resume` ayrı kontrollü işlemdir.
- Yeniden başlatma sırasında yalnızca gerçekten bulunup checksum'ı doğrulanan checkpoint'ten devam edilir.
- Hata nedeni, exit code, zaman, stage, retry geçmişi ve son sağlıklı checkpoint kayda geçer.
- Çift başlatma engellenir; aynı `run_id` için iki worker eşzamanlı aynı checkpoint'e yazmaz.

### 9.3 İdempotence ve atomiklik

Run oluşturma isteği tekrarlanırsa kullanıcıya yanlışlıkla iki eğitim başlatılmamalıdır. Bunun için isteğe bağlı idempotency key veya tekillik anahtarı kullanılır. Artifact dosyaları önce geçici hedefe yazılır, kapandıktan sonra SHA-256 hesaplanır ve mümkünse atomik taşıma yapılır. Tamamlanmamış shard/checkpoint asla `completed` etiketi almaz.

---

## 10. Deney ve metrik kayıtları

Forge V1'de temel prensip **ham kanıtı kaybetmeden okunabilir rapor üretmektir**.

Önerilen kayıt modeli:

```json
{
  "schema_version": "1.0",
  "run_id": "run-2026-10-30-0001",
  "experiment_id": "tr-continuation-pilot",
  "status": "completed",
  "backend": "kaiwen-speech",
  "dataset": {
    "id": "tr-speech-pilot",
    "version": "v1",
    "manifest_sha256": "<sha256-replaced-at-runtime>"
  },
  "source": {
    "repository": "ImtheKaiwen/kaiwen-speech",
    "commit": "<full-commit-sha>"
  },
  "model": {
    "config_id": "baseline_30m",
    "codec_family": "mimi",
    "codec_revision": "<pinned-revision>"
  },
  "execution": {
    "device": "cuda",
    "seed": 42,
    "precision": "fp16",
    "python_version": "<recorded-version>"
  },
  "metrics": {
    "train_loss_last": 1.23,
    "validation_loss_last": 1.51
  },
  "artifacts": [
    {"role": "checkpoint", "uri": "artifacts/runs/<id>/checkpoints/latest.pt", "sha256": "<sha256>"}
  ]
}
```

**Bu kayıt ve sayılar sadece biçim örneğidir; gerçek eğitim çıktısı değildir.** `status=completed` ve `<...>` placeholder'ları gerçek registry doğrulamasında birlikte kabul edilmez. Tamamlanmış bir koşuda hash, version, kaynak ve durum alanlarının doğrulanmış olması zorunludur.

### 10.1 V1 metric formatı

Örnek metrik olaylarının özelliği:

- `run_id`, `step`, `timestamp_utc`, `metric_name`, `value`, `unit`, `split`, `source`.
- `NaN`, sonsuz değer, yanlış birim ve geriye giden step zamanları kontrollü ele alınır.
- `loss` gibi modele özgü metrikler `training` alanından; ses kalitesi gibi metrikler `evaluation` alanından gelir.
- Aynı isimle kaydedilmiş ama farklı tanımlı iki metrik otomatik karşılaştırılmaz; metrik tanımı da sürümlüdür.
- Logların tümü bellekte tutulmaz; stream ve disk/DB write batch kullanılır.
- Gizli bilgiler, kullanıcı sesleri veya tokenlar loglara kontrolsüz yazılmaz.

### 10.2 Depolama seçimi

İlk V1 için **SQLite (WAL) + dosya sistemi** güçlü varsayılan seçimdir: düşük operasyon maliyeti, kolay yedekleme ve yerel analiz. SQLite yalnızca metadata, küçük metrik kayıtları, referanslar ve durum geçişleri tutar. Büyük tensor, ses veya checkpoint binary'leri DB'ye gömülmez. Concurrency/lock ihtiyaçları ölçülmeden PostgreSQL veya Redis kurulmaz.

---

## 11. Corpus veri sözleşmesi ve veri yönetişimi

Corpus'un görevi **veriyi hazırlamak**, Forge'un görevi **belirli sürümünü kontrollü deneye bağlamak**. Forge; kaynak URL ve lisansı, hash'i, konuşmacı/konuşma gruplanmasını, kalite durumunu ve codec'e uygun gerçek örnekleme hızını doğrular.

### 11.1 Dataset kimliği

Bir dataset yalnızca dosya yoluyla tanımlanmaz:

```text
dataset_id
+ dataset_version
+ export_schema_version
+ manifest_sha256
+ source/provenance snapshot
```

Bir `dataset.lock` başka içerikle yeniden yazılırsa aynı sürüm kimliği altında sessizce kabul edilmez. İçerik farklıysa yeni versiyon veya yeni digest gerekir.

### 11.2 Audio kalitesi: 16 kHz ile 24 kHz farkı

Mevcut Corpus pipeline'ında bazı işlemler 16 kHz mono WAV üzerinden çalışıyor. Kaiwen Codec araştırma hedefi **24 kHz** ses. 16 kHz kaydı 24 kHz'e upsample etmek 8 kHz üzerindeki kayıp bandı geri oluşturmaz. Bu nedenle Forge, codec eğitimi için mümkünse gerçek 24 kHz veya üzeri örnekleme kalitesine sahip orijinal ses türevinin kaynak/provenance bilgisini istemelidir.

ASR için 16 kHz türevi makul olabilir; codec pretraining için onun tek kaynak olmasına izin vermeyiz. Veri pipeline'ı frekans dönüşümünü açıkça sürümlendirir; re-sample edilen dosyalar yeni artifact/hash üretir.

### 11.3 Split leakage

Train/validation/test ayrımında dosya ismi değil, gerçek örnekler arasındaki bağımlılık önemlidir. Aynı kayıttan koparılmış segmentlerin, aynı konuşma bölümlerinin veya yakın kopyaların farklı split'lere düşmesi raporu olduğundan iyi gösterebilir. Forge, Corpus'un group key ve dedup bilgilerini kontrol eder; kayıt/conversation/speaker temelli ayrımın hangi seviyede güvenilir olduğunu raporlar.

Özellikle speaker diarization önerisinin gerçek model sonucu olup olmadığı anlaşılmadan, `speaker_id` alanına göre split yapılan dataset `speaker-disjoint` diye etiketlenmez.

### 11.4 Lisans ve izinler

- Kaynak sesin kullanım/yeniden dağıtım/eğitim izinleri kaydedilmeden büyük çaplı otomatik indirme veya kamuya yayın yapılmaz.
- YouTube veya üçüncü taraf içeriklerin indirilebilir olması, otomatik olarak model eğitimi lisansı vermez.
- Veri kaynağına ait lisans belirsizse ilgili kayıt **review/blocked** durumuna alınabilir.
- Hassas konuşma verisi, kimlik bilgisi ve kişisel sesler demo raporlarına izinsiz eklenmez.
- Gerçek/sentetik veri ayrı etiketlenir; sentetik konuşma gerçek insan diyalogu gibi sunulmaz.
- Değerlendirme veri setinin eğitime sızması şema ve provenance kontrolleriyle önlenmeye çalışılır.

### 11.5 Corpus sözleşmesinin otoritesi

Entegrasyon genel ilkeleri `../docs/INTEGRATION.md`; gerçek export formatının nihai sahibi ise gelecekte yazılacak `kaiwen-corpus/docs/EXPORT_CONTRACT.md` olacak. Forge bu formatı **tüketen ve doğrulayan** taraftır. İki repo birbirinden bağımsız key isimleri icat ederse sözleşme testi başarısız olmalıdır.

---

## 12. Kaiwen Speech backend adapter tasarımı

V1 adapter'ı `kaiwen_voice` mevcut public API/CLI'ına dayanır. Adapter interface'i küçük tutulur; önce tek bir Speech trainer'ını çalıştırmak yeterlidir.

Önerilen sorumluluklar:

- `validate_compatibility(recipe, environment)` — veri/codec/model/precision uyumluluğu.
- `prepare_run(run_snapshot)` — output ve input referanslarını hazırla.
- `launch(run_snapshot, process_context)` — mevcut Speech training primitive'ini çağır.
- `parse_metrics(log_stream)` — yapılandırılmış metrikleri al; bozuk logu yutma.
- `collect_artifacts(run_directory)` — checkpoint/rapor/hash doğrula.
- `evaluate(run, suite)` — Speech'e özgü değerlendirme bileşenlerini uygula.
- `supports_resume(checkpoint)` — checkpoint uyumluluğunu kontrol et.

Bu isimler **önerilen adapter arayüzüdür, mevcut Python API değildir**.

### 12.1 Minimum entegrasyon örneği

İlk hedef adapter, `kaiwen-speech/scripts/train_continuation.py` içindeki gerçek argümanlardan yararlanabilir:

```bash
# Forge'un ileride çalıştırabileceği GEÇERLİ SPEECH CLI ÖRNEĞİ
# Bu komut kaiwen-speech repo kökünde yürütülür.
uv run python scripts/train_continuation.py \
  --config configs/smoke.yaml \
  --synthetic \
  --steps 100 \
  --output-dir /absolute/path/to/forge-run-0001/checkpoints
```

Adapter'ın görevi komutu başlatmakla sınırlı değildir: `cwd`, environment, timeout, subprocess exit code, stderr, dosya yolları, checkpoint gerçekliği, iptal ve run durumunu da yönetir. İlk sürümde `subprocess` tabanlı çağrı kabul edilebilir; daha sonra ihtiyaç kanıtlanırsa doğrudan tipli Python backend arayüzü eklenebilir.

### 12.2 Dikkat: training engine ownership

- Model forward, loss, backward ve optimizasyonun **algoritmik uygulaması** Speech'tedir.
- Forge hangi experiment ne zaman hangi ayarlarla çalıştı ve çıktısı nereye yazıldı sorularını çözer.
- Forge, Speech'in `Trainer` kodunu kopyalayıp kendi ikinci trainer'ını oluşturmaz.
- Speech config şeması Forge'da elle yeniden tanımlanmaz; uyumluluk için sahip repo şeması kullanılır veya sürümlü adapter doğrular.
- Yeni Kaiwen Codec yazıldığında, Speech repo sürümü ve codec kimliği değişimi Forge tarafından izlenir. Mimi tokenizer cache'i yeni codec için yeniden kullanılamaz.

### 12.3 Codec-token eşleşmesi

Token cache kontrolü için yalnızca `Q`, `K` ve `frame_rate` yeterli değildir. Aynı şekle sahip farklı codec'ler aynı token ID'lerine **aynı akustik anlamı** vermez. V1 run kayıtları en az codec aile/versiyon/revision, model kimliği, codebook düzeni, örnekleme hızı ve doğrulanabilir artifact özeti taşımalıdır. Bunlar uyuşmuyorsa run başlamadan veya checkpoint yüklenmeden **fail-fast** uygulanır.

### 12.4 Warm-start ve resume farkı

- **Resume:** Aynı eğitime optimizer/scheduler/scaler ve step ile devam; kaynak checkpoint'in yapı ve config uyumu gereklidir.
- **Warm-start:** Ağırlıkların bilinçli bir kısmını başka konfigürasyona aktarır; yeni run kimliği ve açık transfer raporu gerekir.
- **Import:** Harici artifact'i kaydetme işlemi; eğitim veya başarı kanıtı değildir.

Forge bu işlemleri tek `--resume` bayrağı altında gizlememelidir. Özellikle `parallel → depth` warm-start gibi geçişlerde hangi ağırlıkların yüklendiği, hangilerinin yeniden başlatıldığı kaydedilmelidir.

---

## 13. Eğitim orkestrasyonu ve kaynak yönetimi

### 13.1 V1 kapsamı

V1 için önerilen çalışma modeli:

```text
User CLI
   ↓
Validate recipe + dataset + environment
   ↓
Create run record + immutable snapshot
   ↓
Queue (single-machine, bounded)
   ↓
Launch model-specific backend
   ↓
Monitor stage, metrics, stdout, heartbeat
   ↓
Cancel / retry / resume / complete
   ↓
Verify checkpoint + update experiment store
```

Yalnızca tek kişi için demo hazırlanırken, veritabanı destekli dağıtık görev kuyruğu, birden fazla worker node, Kubernetes yönetimi ve remote GPU autoscaler zorunlu değildir. Önce `local executor` ve gerektiğinde Colab için **manuel/import tabanlı remote execution** yeterli olabilir.

### 13.2 Resource policy

Her run için açıkça kaydedilecekler:

- CPU/GPU device ve precision.
- Kullandığı model ve codec sürümleri.
- Beklenen bellek/boş disk için erken kontrol (ölçüm değil, tahmin).
- Process concurrency limiti; aynı GPU'da iki büyük işi istemeden başlatmama.
- Disk üst sınırı, log rotasyonu, checkpoint retention politikası.
- Wall-clock timeout ve heartbeat/recovery beklentisi.
- İptal sinyali, graceful shutdown ve son sağlam checkpoint'in korunması.

Memory estimate gerçek VRAM ölçümünün yerini tutmaz. OOM yakalandığında Forge sonucu başarısız olarak kaydeder, cihazı temizlemeyi dener ve kullanıcıya batch size/precision/sequence length gibi değişkenlerin etkisini önerir; fakat kendiliğinden farklı bir recipe ile tekrar çalışıp sonucu aynı run olarak göstermemelidir.

### 13.3 Hata sınıfları

| Hata | Beklenen davranış |
|---|---|
| Dataset bulunamadı | İş başlatılmaz, doğrulama hatası kaydı tutulur. |
| Schema/hash uyuşmazlığı | Fail-closed; kullanıcı onayıyla dahi doğrulanmamış veri sessizce kabul edilmez. |
| Codec/model uyumsuzluğu | Eğitimden önce reddedilir. |
| CUDA OOM | `failed` veya kontrollü `interrupted`; kaynaklar temizlenir. |
| Colab bağlantısı koptu | Son doğrulanmış artifact'ten sonra `interrupted`/unknown; otomatik başarı yok. |
| Checkpoint yarım yazıldı | Checksum/doğrulama başarısız; son sağlam sürüme dönüş. |
| Metric parser bozuk | Çalıştırma devam edebilir ama ölçüm eksikliği açık raporlanır; başarı kapıları sıkı olabilir. |
| İptal | İşlem öldürme, artifact koruma, `cancelled` geçişi ve audit kaydı. |
| Aynı run iki kez başlatıldı | İkincisi idempotent cevap alır veya net çakışma hatası. |

---

## 14. Google Colab ve Google Drive kullanımı

Kullanıcının araştırma ortamı için temel strateji: **büyük veri indirmeyi ve GPU/CPU işlerini hosted Colab üzerinde yapabilmek, dayanıklı çıktıları Google Drive'a aktarmak**. Bu tasarım, Colab oturumunun süresiz çalıştığı veya Drive mount'ın güvenilir yerel disk gibi davrandığı varsayımına dayanamaz.

### 14.1 Corpus ile Forge'un sınırı

- **Corpus:** Kaynağı keşfeder, izinlerini kaydeder, ham sesi indirir, işleme/etiketleme/export süreçlerini yönetir.
- **Forge:** Corpus tarafından yayımlanmış **sürüm ve içerik bütünlüğü belli** veri setini, eğitim için doğrular ve kullanır.
- **Speech:** Batch/streaming tokenization, codec/model eğitimi ve inference algoritmalarını uygular.

Forge, Corpus'un büyük dataset indirme GUI'sini veya YouTube keşif aracını yeniden yazmaz.

### 14.2 Colab çalışma ilkeleri

1. Hosted runtime türü ve kaynakları iş başında kaydet: Python, PyTorch, CUDA, GPU, disk, bellek.
2. Drive'ı kalıcı artifact hedefi olarak kullan; yüksek rastgele I/O gerektiren eğitim cache'ini mümkünse `/content` gibi yerel geçici depoda tut.
3. Uzaktaki licensed dataset parçalarını **Colab runtime üzerinden** indir; kullanıcının laptop bağlantısına otomatik geri dönme.
4. Download/copy işlerinde shard bazlı hash, boyut, tekrar deneme, son başarılı işlem imleci ve limit tutulur.
5. Aynı dosya adına farklı içerik gelirse üzerine sessizce yazma; conflict oluştur.
6. Yarım transfer edilen dosyayı geçici adla tut, doğrulanınca tamamlanmış say.
7. Colab kesilirse run durumu `interrupted` veya `unknown` olur; Drive'daki son checkpoint'ler ayrıca taranır.
8. Drive kota, mount kopması, yavaş ağ ve Colab depolama sınırları için erken hata/geri kazanım akışı geliştir.
9. Google Drive credential'ı, paylaşım linki veya OAuth token'ı log/checkpoint içine gömme.
10. Gizli/kişisel ses verilerini ve lisanslı verinin dağıtımını varsayılan olarak kapalı tut.

### 14.3 Önerilen artifact yolu — **hedef**

```text
MyDrive/KaiwenAI/
├── datasets/
│   └── <dataset-id>/<version>/
│       ├── dataset.lock
│       ├── READY
│       ├── segments.jsonl
│       └── shards/
├── forge/
│   ├── runs/<run-id>/
│   │   ├── resolved_recipe.yaml
│   │   ├── environment.json
│   │   ├── metrics.jsonl
│   │   ├── checkpoints/
│   │   ├── evaluation/
│   │   └── run.json
│   └── registry/<model-id>/<version>/
└── reports/
```

Bu klasör adları bugünkü Google Drive'ın gerçek klasörleri olarak doğrulanmadı; V1 storage sözleşmesi için **öneri**. Geçmiş `KaiwenVoice/` yolları bulunan veriler silinmeden, açık migration/alias kararı ile taşınmalıdır.

### 14.4 Colab reprodüksiyon paketi

Bir Colab çalışması sonrasında en az şu rapor bulunmalıdır:

```text
Run identity: run ID + experiment ID
Source: kaiwen-speech Git SHA + Forge Git SHA
Data: dataset version, manifest digest, codec revision
Config: resolved recipe ve model config
Environment: CPU/GPU, RAM/VRAM, CUDA/PyTorch/Python, dependencies
Outputs: checkpoint SHA-256, log, metrics, eval, termination reason
Recovery: son sağlam checkpoint ve continuation policy
```

Notebook tek başına bilimsel tekrarlanabilirlik sağlamaz: hücrelerin çalıştırılma sırası, ortamın sürümü, eklenen paketler ve dış dosyalar da belirleyicidir.

---

## 15. Artifact depolama ve checksum sözleşmesi

Forge'un taşıdığı binary'ler büyük olabilir; bu yüzden metadata ile byte içeriği ayrı tutulur.

### 15.1 Artifact türleri

- `checkpoint`: Speech'in ürettiği model/eğitim durumu.
- `weights`: yayınlanan inference ağırlıkları.
- `codec`: ayrı codec modeli, tokenizer metadata'sı, codebook sürümü.
- `config`: canonical resolved training/model config.
- `dataset_manifest`: immutable Corpus export kimliği.
- `metrics`: scalar, histograms, ölçüm raporu.
- `evaluation`: sonuç, fixture referansı, ses kalite raporu.
- `audio_sample`: mahremiyeti/lisansı onaylı değerlendirme örneği.
- `release_manifest`: Live'ın kabul edeceği sürümlü index.

### 15.2 Artifact kayıt alanları

Önerilen minimum alanlar:

```text
artifact_id
artifact_kind
created_at_utc
producer_run_id
origin_repository + git_sha
uri
sha256
size_bytes
media_type / format
schema_version
model_id + codec_id (uygunsa)
license / redistribution policy (uygunsa)
retention_policy
verified_at_utc
```

Hash sadece kaydedildiği için doğru kabul edilmez; transfer sonrası yeniden hesaplanır. Uygun güven modeli olmadan SHA-256 **kaynağın güvenilirliğini tek başına ispatlamaz**; beklenen hash'in güvenilir manifest/release kaynağından gelmesi gerekir.

### 15.3 Güvenlik

PyTorch `.pt` dosyaları uygun olmayan yükleme biçiminde güvensiz deserialize riski taşıyabilir. Speech'teki mevcut checkpoint loader `weights_only=True` kullanıyor; Forge yabancı kaynaklardan gelen checkpoint'i otomatik yükleyip çalıştırmamalı. İmza/signed manifest, güvenilir origin ve sabit hash politikası release tarafında tasarlanmalıdır. Checkpoint ile inference artifact'inin ihtiyaçları farklıdır: optimizer state içeren training dosyasını Live uygulamasına körlemesine dağıtmak doğru değildir.

---

## 16. Değerlendirme: loss düşmesi tek başına başarı değildir

Kaiwen Speech'in araştırma hedefi doğal Türkçe speech-native konuşma. Forge değerlendirme katmanı bu hedefin **ölçülebilir kanıtını** tutmalıdır.

### 16.1 V1 evaluation katmanları

| Seviye | Amaç | Örnek test |
|---|---|---|
| `E0` — Structural | Model/tensor/checkpoint bozulmamış mı? | Shape, dtype, codec metadata, finite loss, round-trip. |
| `E1` — Training | Eğitim akışı tutarlı mı? | Loss eğilimi, val loss, resume parity, seed kayıtları. |
| `E2` — Acoustic | Ses ne kadar anlaşılır ve doğal? | Kör dinleme paneli, codec distortion ölçümleri, tanımlı set. |
| `E3` — Conversational | Türkçe anlam, yanıt, turn-taking davranışı iyi mi? | Ayrılmış diyalog senaryoları, konuşma rubrikleri. |
| `E4` — Runtime | Kullanıcı açısından gerçekten akıcı mı? | TTFA, RTF, peak VRAM, uzun oturum, jitter. |
| `E5` — Safety/Release | Uyumlu, izinli ve regresyonsuz mu? | Manifest, license, security, rollback, Live smoke. |

### 16.2 Metriklerle dürüst ilişki

- Bir codec'in `bitrate` değeri ses kalitesinin yerini tutmaz.
- Daha düşük validation loss, daha doğal Türkçe konuşma ile eşanlamlı değildir.
- RTF `1.0` altında olsa bile ilk duyulan sesin gecikmesi yüksek olabilir.
- Synthetic dialogue verisiyle kazanılan başarıyı doğal insan konuşmasına doğrudan genelleyemeyiz.
- Farklı frame rate'ler model sequence length ve gecikme maliyetini değiştirir; yalnız model parametre sayısını kıyaslamak yeterli değildir.
- Speaker, emotion veya overlap için gerçek etiket/ölçüm yoksa `unsupported` ya da `unmeasured` yazılır.
- `2 GB VRAM` bir **hedef**; codec + Transformer + KV cache + runtime toplamının gerçek cihazda ölçülmesi gerekir.

### 16.3 Ayrık değerlendirme kümeleri

Deney karşılaştırmaları için train ve test seti kaynağının ayrıldığı, tekrar kullanılabilen bir benchmark seti gerekir. Bu setin hash, örnek/speaker/conversation grubu, dil kapsamı, anotasyon kalitesi, synthetic oranı ve kullanım izni sürümlendirilir. İnsan dinleme değerlendirmesinde sıralama rastgele olmalı ve değerlendirici mümkünse model kimliğini bilmemelidir.

### 16.4 Karşılaştırma raporu

Rapor asgari olarak şu tabloyu üretebilmelidir:

| Alan | Baseline A | Candidate B |
|---|---|---|
| Model ve codec sürümü | Kaydedilir | Kaydedilir |
| Dataset revision/hash | Kaydedilir | Aynı mı? |
| Parametre sayısı | Ölçülür | Ölçülür |
| Eğitim device/time | Kaydedilir | Kaydedilir |
| Validation loss | Ölçülür | Ölçülür |
| Türkçe anlaşılabilirlik | Ölçülür / unmeasured | Ölçülür / unmeasured |
| Latency TTFA p50/p95 | Ölçülür / unmeasured | Ölçülür / unmeasured |
| Peak total VRAM | Ölçülür / unmeasured | Ölçülür / unmeasured |
| Teknik risk ve sınırlar | Belirtilir | Belirtilir |

Unmeasured hücreler `0`, tahmini `100%` veya gerçek dışı örnek sayılarla doldurulmaz.

---

## 17. Model registry: araştırmadan ürüne geçiş kapısı

Training çıktısı `latest.pt` dosyasının oluşması, o modelin Kaiwen Live tarafından güvenle yüklenebileceği anlamına gelmez.

### 17.1 Önerilen model yaşam döngüsü

```text
Run checkpoint
    ↓  inspect + checksum + compatibility
Experiment artifact
    ↓  evaluation suite + comparison
Candidate
    ↓  human approval + policy gate
Validated release
    ↓  versioned manifest + immutable artifact
Available to Kaiwen Live
    ↓  telemetry + compatibility + optional rollback
Retired / superseded
```

- `candidate` ile `released` farklıdır.
- Release modeli üzerine yazılmaz; yeni sürüm üretilir.
- Önceki çalışan sürüm geri dönüş amacıyla tutulabilir.
- Modelin hangi codec ile kullanılması gerektiği açıkça yazılır.
- Her model/codec/release sürümü tekil ve değişmez hash'lerle doğrulanır.
- Model registry yalnızca dosya koleksiyonu değil; **kanıt, uyumluluk ve onay** kaydıdır.

### 17.2 Önerilen release manifest — **şema taslağı**

```json
{
  "schema_version": "1.0",
  "release_id": "kaiwen-speech-demo-0.1.0",
  "status": "candidate",
  "model": {
    "family": "speech",
    "name": "kaiwen-speech-demo",
    "version": "0.1.0",
    "architecture": "temporal-transformer-parallel",
    "sample_rate_hz": 24000
  },
  "codec": {
    "family": "mimi",
    "revision": "<exact-pinned-revision>",
    "num_codebooks": 8,
    "codebook_size": 2048,
    "frame_rate_hz": 12.5
  },
  "runtime": {
    "minimum_speech_api_version": "<validated-version>",
    "requires_streaming": false
  },
  "artifacts": [
    {
      "role": "model_weights",
      "uri": "<immutable-artifact-uri>",
      "sha256": "<verified-sha256>"
    }
  ],
  "evaluation_refs": [],
  "approved_by": null
}
```

Bu örnek `candidate` statüsündedir ve placeholder barındırır: Live tarafından otomatik **released** sayılmaz. Manifest formatı, Speech'in gerçek model/codec metadata'sıyla ve Live yükleme adaptörüyle birlikte contract testlerinden geçirildikten sonra kesinleşir. `model_weights` ile codec dosyalarının ayrı artifact olarak tutulması halinde her iki hash de doğrulanmalıdır.

### 17.3 Live'a aktarım sınırı

Forge'un görevi:

- Hangi release'in indirilebilir/uyumlu olduğunu belgelemek.
- Integrity ve sürüm metadata'sını sunmak.
- Onay ve kalite sonuçlarını kaydetmek.

Live'ın görevi:

- Kullanıcıya modeli seçtirmek veya gerekli paketi istemek.
- Dosyayı güvenle indirmek/önbelleğe almak.
- Speech SDK'sı ile modelin çalışabilirliğini doğrulamak.
- Streaming/session davranışını ve ses arayüzünü yönetmek.

Forge, son kullanıcı uygulamasındaki mikrofonu veya Electron IPC'yi yönetmez.

---

## 18. Tekrarlanabilirlik ve bilimsel izlenebilirlik

Her raporun arkasında **gerçek kaynak** olmalı:

```text
Question / hypothesis
   → Dataset & evaluation protocol
   → Recipe + resolved config
   → Code/package/codec revisions
   → Run + environment snapshot
   → Metrics + audio + benchmark evidence
   → Comparison + limitations
   → Accepted/rejected conclusion
```

### 18.1 Zorunlu minimum kayıtlar

- Research hypothesis veya deney amacı.
- Kullanılan dataset manifest + split + SHA.
- Kullanılan model/codec kaynak sürümü + config.
- Seed, determinism politikası, optimizer/precision ve önemli hyperparameter'lar.
- GPU/CPU, VRAM, OS, Python, framework ve bağımlılık sürümleri.
- Başlatma/bitiş saatleri, gerçek toplam süre, hata/iptal nedeni.
- Checkpoint ve sonuç dosyalarının içerik hash'leri.
- Evaluation suite sürümü, örnek sayıları, kabul kriterleri.
- Varsa insan değerlendirme protokolü ve veri koruma notları.

### 18.2 Determinism hakkında gerçekçi kural

Aynı seed, her donanım ve kütüphane kombinasyonunda bit-for-bit aynı sonucu garanti etmez. Özellikle GPU kernel'leri, mixed precision, bazı CUDA işlemleri ve Colab runtime değişiklikleri sonuçları etkileyebilir. Forge **determinism ayarını ve ihlal/istisnaları** kaydeder; doğrulanmamış eşdeğerlik iddia etmez.

### 18.3 Akademik kullanımı

Bitirme projesi raporlarında sonuç tablosu Forge run ID'leriyle eşleştirilmeli. Başarısız deneylerin de en azından başlangıç konfigürasyonu, hata nedeni ve sınırlamaları tutulmalı. Sadece iyi çıkan koşuları seçip başarısızları gizlemek araştırma kalitesini düşürür.

---

## 19. Güvenlik ve gizlilik sınırları

Kaiwen Forge model eğitimini yönettiği için dosya sistemi, process çalıştırma ve uzaktaki artifact konumlarına erişebilir. Bu, açık bir **güven sınırı** gerektirir.

### 19.1 Kod ve komut çalıştırma

- Varsayılan reçete parser'ı arbitrary Python `eval`, `exec`, unsafe YAML object construction veya şüpheli shell expansion yapmaz.
- Recipe argümanları mümkünse shell string birleştirmek yerine list-based subprocess çağrılarıyla iletilir.
- Model adapter'ları güvenilir, sürümü sabit koddan yüklenir; kullanıcı tarafından verilen her uzak URL çalıştırılabilir plugin değildir.
- Çıktı dizini ve artifact URI'si traversal/symlink escape gibi sorunlara karşı kontrol edilir.
- Remote kaynaklardan gelen binary/checkpoint'ler güven modeli ve format kontrolü olmadan deserialize edilmez.

### 19.2 Veri ve sırlar

- `HF_TOKEN`, Drive credential, API key gibi sırlar Git commit, recipe snapshot, log veya release manifestine girmemeli.
- Kişisel ses kaydı varsayılan olarak telemetriye gönderilmemeli.
- Lisanslı veri, model ağırlığı ve benchmark ses örnekleri otomatik kamuya açık release artifact'i yapılmamalı.
- GitHub Actions'ta minimum token permissions, dependency pinning ve güvenilir action kullanımı uygulanmalı.
- Artifact temizliği izin ve retention politikasıyla yapılmalı; kullanıcı onayı olmadan `rm -rf` gibi geniş silme işlemleri yok.

### 19.3 Politika / onay sınırları

Büyük remote dataset indirme, ücretli cloud job başlatma, dosya üzerine yazma, ortak registry'de `promote`, kamuya model yayınlama ve geniş toplu silme **açık insan onayı** gerektiren işlemler olarak ele alınmalı. Basit yerel CPU smoke ve test komutları böyle bir onay gerektirmez.

---

## 20. Test stratejisi ve CI/CD

Forge'un ilk PR'ı yalnızca bir README değil; gerçek paket altyapısı oluşturulduğunda aşağıdaki kapılar da eklenecek.

### 20.1 Katmanlı test matrisi

| Test katmanı | Örnek | CI beklentisi |
|---|---|---|
| Unit | Recipe schema, state machine, metrics normalization, checksum | CPU üzerinde her PR |
| Contract | Corpus manifest, Speech adapter version, model release manifest | Fixture tabanlı her PR |
| Integration | Local subprocess fake backend, checkpoint collection, SQLite recovery | CPU üzerinde her PR |
| Failure injection | Kısmi checkpoint, disk dolması, timeout, iptal, dosya yarışları | Her PR için seçilmiş senaryolar |
| Security | Unsafe YAML, traversal, shell injection, secret redaction | Her PR |
| E2E smoke | Küçük gerçek Speech synthetic training yolunu çalıştırma | Kaynak bütçesine göre seçilmiş CI veya manuel |
| GPU | CUDA/Colab, peak VRAM, resume ve inference | Ayrı, kontrollü/manuel GPU job |
| Release | Model manifest ve publish compatibility smoke | Yayın öncesi zorunlu |

### 20.2 Önerilen kalite komutları — **Forge paketine eklendikten sonra**

```bash
# HEDEF komutlar: pyproject.toml ve test altyapısı henüz yok.
uv sync --frozen --extra dev
uv run ruff check .
uv run pyright
uv run pytest
uv build
```

Hangi type checker veya build backend kullanılırsa kullanılsın, araç sürümleri lockfile ile sabitlenir ve CI ile yerel geliştirme aynı quality gate'i kullanır. Codec/model GPU testleri her CPU PR'da pahalı şekilde çalıştırılmaz; tekrarlanabilir ayrı workflow bulunur.

### 20.3 GitHub Actions önerisi

```text
Pull Request / Push
    ├── Python format/lint/typecheck
    ├── unit + integration + contract tests
    ├── security/dependency scanning
    └── package build

Manual/Tagged GPU Research
    └── pinned Speech backend + small evaluation + artifact report

Release (ayrı onaylı işlem)
    ├── changelog + version verification
    ├── artifact hashes + metadata
    └── package publishing / signed release
```

**CI yeşil demek modelin Türkçe konuşma kalitesi kanıtlandı demek değildir.** CI yazılım doğru çalışıyor mu kontrol eder; bilimsel kalitenin ayrıca değerlendirme kanıtı gerekir.

---

## 21. Performans ve operasyon bütçesi

Forge gerçek zamanlı ses üretim döngüsünde çalışmamalı. Bu yüzden Forge'un gecikme hedefleri ses frame milisaniyeleriyle değil, **işletim verimliliği ve kaynak güvenliği** ile ölçülür.

V1 için izlenecek metrikler:

- Recipe validation wall time ve başarısız validasyonun işe başlamadan yakalanma oranı.
- Run startup overhead (backend başlamadan önceki hazırlık maliyeti).
- Metric ingestion rate, memory tüketimi ve log dosyası büyümesi.
- Checkpoint doğrulama/Drive aktarım süresi, retry sayısı, veri bütünlüğü.
- Interrupted run recovery, resume öncesi doğrulama süresi.
- GPU runtime kullanım süresi (Forge orchestration overhead'inin training süresine oranı).
- Registry lookup ve manifest verification süresi.
- Experiment compare sorgularının ölçeklenebilirliği.

Sayısal SLO/eşikler, ilk yerel baseline ölçümleri yapıldıktan sonra `docs/EXPERIMENTS.md` ve `docs/TRAINING_PIPELINE.md` içinde **kanıta dayalı** belirlenecek. Şimdiden keyfî milisaniye limitlerini gerçek test şartı gibi göstermek doğru değil.

### 21.1 Hesaplama ve depolama maliyeti

Önerilen maliyet disiplini:

1. Dry-run ile amaçlanan iş, veri boyutu, disk gereksinimi ve execution target gösterilir.
2. CPU synthetic smoke ile recipe/backend doğrulanır.
3. Küçük gerçek dataset pilotu yapılır.
4. Büyük GPU/Colab deneyleri ancak küçük pilot başarılıysa ölçeklenir.
5. Aynı dataset'i gereksiz tekrar indirmek yerine hash tabanlı staging cache kullanılır.
6. Cache temizleme retention/quotaya bağlıdır; gizli veri paylaşımına yol açmaz.

---

## 22. V1 geliştirme aşamaları

Bu liste, sıradaki dört Forge teknik belgesiyle detaylandırılacak. **Sıra ve done kriterleri somuttur; süre tahmini veya bitiş garantisi değildir.**

| Aşama | Hedef | Tamamlanma kanıtı |
|---|---|---|
| **F0 — Repo foundation** | `pyproject.toml`, src layout, uv, lint/type/test/CI | `uv sync`, package import, CPU test ve CI geçer. |
| **F1 — Domain & recipe** | Sürüm şeması, recipe parser, fail-fast validator | Geçerli/geçersiz fixture testleri ve immutable run snapshot. |
| **F2 — Run store** | SQLite WAL + artifact dirs, state transition | Crash/restart sonrası kayıt korunur, aynı run iki kez çalışmaz. |
| **F3 — Local executor** | Güvenli subprocess, cancel, timeout, heartbeat | Fake backend E2E, hata sınıfları ve resource cleanup testleri. |
| **F4 — Speech adapter** | Mevcut `kaiwen_voice` training CLI entegrasyonu | Synthetic 100-step overfit run, checkpoint doğrulama, metrics. |
| **F5 — Corpus handoff** | Dataset schema/hash/license ve split kontrolü | Sabit pilot manifest ile doğrulama ve bozuk hash reddi. |
| **F6 — Evaluation V1** | Structural, training, acoustic ve runtime değerlendirme kaydı | Sürümle bağlı suite sonuçları ve karşılaştırma raporu. |
| **F7 — Registry V1** | Candidate/released yaşam döngüsü, release manifest | Uyumlu artifact promotion, rollback ve negatif testler. |
| **F8 — Colab/Drive** | Remote run import, verified artifact sync, resume metadata | Bilinçli bağlantı kesintisinde bozuk dosya `completed` olmaz. |
| **F9 — Stabilization** | Dokümantasyon, release kalite kapıları ve bitirme kanıtları | Tekrarlanabilir örnek demo, temiz çalışma alanı ve test raporu. |

### 22.1 Minimum kullanılabilir Forge (ilk kesit)

F0–F4 birlikte, tek CPU host üzerinde bir Speech smoke eğitimini başlatıp sonuçları kaydeden gerçek bir Forge sağlar. F5–F8 eklenmeden bunu **tam platform** diye sunmak doğru değildir. Bu sıralama, uzun süre soyut altyapı yazmak yerine erken çalışan sonuç almayı hedefler.

### 22.2 Paralel geliştirilebilecek işler

- Corpus, lisanslı veri pilotunu hazırlarken Forge dataset validator fixture'larını geliştirebilir.
- Speech kendi codec'ini eğitirken Forge mevcut Mimi baseline üzerinden deney takip sistemini kurabilir.
- Live, fake runtime ile tasarım/oturum sistemini geliştirirken Forge registry release manifest sözleşmesini hazırlayabilir.
- Değerlendirme protokolleri, bütün model eğitimi bitmeden tasarlanabilir; böylece sonuca göre test uydurulmaz.

**Forge, diğer projelerin bitmesini beklemek zorunda değildir.** Bununla birlikte kendi modelini veya dataset editörünü yazmaya başlamamalıdır.

---

## 23. Codex ile çalışma protokolü

Forge üzerinde Codex veya başka bir coding agent çalıştırıldığında:

1. Yerel workspace `../AGENTS.md`, bu README ve varsa `AGENTS.md` önce okunur.
2. `../docs/OVERVIEW.md`, `../docs/INTEGRATION.md`, `../docs/ENGINEERING_STANDARDS.md` ve `../docs/ROADMAP.md` ile çelişki kontrol edilir.
3. `kaiwen-speech` public API/CLI'ları ve mevcut testleri **gerçek koddan** doğrulanır; repo isimlerine bakıp import tahmin edilmez.
4. Henüz oluşturulmamış dosyalar/planned API gerçekmiş gibi kullanılmaz.
5. Tek görev için tek anlamlı değişiklik paketi hazırlanır; gereksiz cross-repo refactor yapılmaz.
6. Yeni implementasyonun en az unit ve contract testleri yazılır; gerekiyorsa CLI E2E smoke eklenir.
7. CI/CD, lockfile, hata yönetimi, logging ve güvenlik sınırları aynı PR'da kontrol edilir.
8. Büyük cloud maliyeti, veri silme, public yayın, model promotion ve özel dataset aktarımı için açık onay alınır.
9. Bir görev sonunda **değişen dosyalar, çalıştırılan komutlar, sonuçlar, eksik kalanlar ve riskler** açık raporlanır.
10. Başarı iddiaları yalnızca ölçüm/log/fixture ile desteklenir; kullanılmayan model kalitesini uydurmaz.

### 23.1 Tekrarlanacak sık hatalar

- `kaiwen-forge` repo adı değişince `kaiwen_voice` importunu otomatik olarak `kaiwen_speech` varsaymak.
- Geliştirilmemiş `forge` CLI'sını README'de aktif özellik gibi göstermek.
- README veya YAML yazmayı eğitim altyapısı tamamlandı saymak.
- Forge ile Speech'in ayrı `Trainer` implementasyonları üretmesi.
- İlk küçük deneyden önce dağıtık scheduler/DB cluster kurmak.
- 16 kHz sesin 24 kHz re-sample edilmiş hâlini kaliteli orijinal kayıt sanmak.
- Checkpoint dosyasını checksum/codec sürümü olmadan Live'a dağıtmak.
- `train_loss` düşüşünü konuşma kalitesi veya `2 GB VRAM` başarısı diye sunmak.
- Model/Dataset lisansını sadece Hugging Face URL'si bulunduğu için onaylanmış saymak.
- Colab kapandığında eksik artifact'leri başarılı gibi kaydetmek.

---

## 24. Dokümantasyon haritası

### 24.1 Ortak Kaiwen AI belgeleri

Aşağıdaki bağlantılar önerilen **yerel `kaiwen-ai/` workspace yapısında** çalışır. Bu repository GitHub'da tek başına klonlandığında komşu repoların bulunması garanti edilmez.

| Belge | Rol |
|---|---|
| [`../README.md`](../README.md) | Altı repo için genel başlangıç. |
| [`../AGENTS.md`](../AGENTS.md) | Workspace Codex çalışma kuralları. |
| [`../docs/OVERVIEW.md`](../docs/OVERVIEW.md) | Ekosistem vizyonu ve sahiplik sınırları. |
| [`../docs/INTEGRATION.md`](../docs/INTEGRATION.md) | Corpus/Forge/Speech/Live sözleşme yönleri. |
| [`../docs/ENGINEERING_STANDARDS.md`](../docs/ENGINEERING_STANDARDS.md) | Ortak kalite ve güvenlik standartları. |
| [`../docs/ROADMAP.md`](../docs/ROADMAP.md) | Altı repoda kilometre taşları. |

### 24.2 Forge teknik belge serisi — **henüz hazırlanacak**

| Planlanan dosya | Konusu |
|---|---|
| `AGENTS.md` | Forge'a özel Codex protokolü ve güven sınırları. |
| `docs/TRAINING_PIPELINE.md` | Orchestrator, execution, resume, Colab ve checkpoint. |
| `docs/EXPERIMENTS.md` | Run veri modeli, metrik, sorgu, karşılaştırma ve raporlama. |
| `docs/EVALUATION.md` | Suite, benchmark, test ayrımı ve bilimsel kanıt. |
| `docs/MODEL_REGISTRY.md` | Artifact, promotion, compatibility, release ve rollback. |

Bu tablo **Forge'un mevcut dosya listesi değildir**. Belgeler üretildikçe link verilecek ve durum etiketleri güncellenecektir.

### 24.3 Model tarafı

Kaiwen Speech ayrıntıları için ilgili repodaki şu dokümanlar rehberdir:

- [`../kaiwen-speech/docs/VISION.md`](../kaiwen-speech/docs/VISION.md): araştırma hedefleri ve Türkçe speech-native yaklaşım.
- [`../kaiwen-speech/docs/ARCHITECTURE.md`](../kaiwen-speech/docs/ARCHITECTURE.md): model/codec sınırları.
- [`../kaiwen-speech/docs/CODEC.md`](../kaiwen-speech/docs/CODEC.md): özgün causal codec ve RVQ.
- [`../kaiwen-speech/docs/SPEECH_MODEL.md`](../kaiwen-speech/docs/SPEECH_MODEL.md): Transformer, loss ve diyalog.
- [`../kaiwen-speech/docs/API.md`](../kaiwen-speech/docs/API.md): SDK ve inference sözleşmeleri.
- [`../kaiwen-speech/docs/PERFORMANCE.md`](../kaiwen-speech/docs/PERFORMANCE.md): VRAM ve gerçek zamanlı performans testleri.

Bu Speech belgeleri önceki adımlarda hazırlanan **yerel taslaklardır**; GitHub'daki repo üzerinde bütün değişikliklerin uygulanmış olduğu anlamına gelmez.

---

## 25. Yönetişim ve karar alma

Her önemli mimari değişiklik için kısa bir ADR veya issue kaydı açılmalı. ADR aşağıdakileri içermelidir:

- **Karar:** Ne değişiyor?
- **Sorun:** Hangi ölçülebilir eksiklik bunu gerektiriyor?
- **Alternatifler:** En az bir daha basit seçenek nelerdi?
- **Etkiler:** VRAM/performans/karmaşıklık/veri güvenliği/test/yayın etkisi.
- **Migration:** Mevcut run/checkpoint ve metadata nasıl korunacak?
- **Rollback:** Yanlış çıkarsa hangi önceki sürüme döneriz?
- **Kanıt:** Hangi test, benchmark veya kullanıcı ihtiyacı bu kararı destekliyor?

MLflow, Optuna, Ray, DVC, Weights & Biases, Prefect, MLflow Model Registry, Hydra gibi araçlar ileride **değerlendirme adayı** olabilir. Hepsini V1'e önceden dahil etmek profesyonellik değildir. Gerçek ihtiyaç varsa benchmark, lisans, offline erişim, bağımlılık ve mevcut çözümle fark üzerinden karar verilir. Özellikle kullanıcının temel hedefi için ilk çalışan deneyin ortaya çıkması, araç seçimlerini sonsuza dek araştırmaktan daha değerlidir.

---

## 26. Neler kapsam dışında?

V1 kapsamı **değildir**:

- Speech modelinin kendi codec/Transformer implementasyonunu Forge'a taşımak.
- Son kullanıcı mikrofon/ses UI'si geliştirmek.
- Agent araçları, genel bilgi retrieval, konuşma persona yönetimi.
- Büyük dağıtık Kubernetes eğitim cluster'ı veya otomatik cloud faturalama sistemi.
- Desteklenmeyen vision/text model aileleri için sahte adaptörler ve boş pipeline'lar.
- Çok kiracılı SaaS, hesap sistemi, rol tabanlı yönetim paneli ve ödeme altyapısı.
- CPU smoke çalışmadan tam ölçekli veri indirme ya da pahalı GPU deneyleri.
- Mevcut Corpus downloader, annotation UI veya Design bileşenlerini tekrar yazmak.

Söz konusu özelliklerden biri gelecekte somut değer sağlarsa yeni kapsam kararıyla eklenir; sadece yapılabilir oldukları için eklenmezler.

---

## 27. Definition of Done: Forge V1 hangi koşullarda hazır?

Bir README veya klasör ağacının tamamlanması **Forge V1'in bittiği anlamına gelmez**. Minimum çalışan sürüm için aşağıdaki kanıtlar gerekir:

- [ ] Temiz Python kurulumu ve type/lint/test/build CI geçiyor.
- [ ] Gerçek `kaiwen_forge` paket API'si ve/veya sürümlenmiş Forge CLI bulunuyor.
- [ ] Schema-valid ve schema-invalid recipe fixture'ları doğru sonuç üretiyor.
- [ ] Run yaratma, durumu güncelleme, idempotent başlatma, iptal ve kesinti sonrası toparlanma test ediliyor.
- [ ] Mevcut Speech CPU synthetic eğitim yolu Forge üzerinden başlatılıyor ve doğru sonuç kaydediliyor.
- [ ] Model/codec/dataset eşleşmezliği eğitim başlamadan reddediliyor.
- [ ] Bounded log/metrik kayıtları ve artifact SHA doğrulaması yapılıyor.
- [ ] Eğitim sonrasında checkpoint tekrar yüklenebiliyor; hata hâlinde `completed` yazılmıyor.
- [ ] Aynı baseline koşusunun recipe, commit, dataset ve environment kaydı rapordan yeniden bulunabiliyor.
- [ ] Corpus export contract'ının lisans, veri kalitesi ve split bilgileri kontrollü tüketiliyor.
- [ ] En az bir evaluation suite ile kıyaslama üretilebiliyor; `unmeasured` alanlar dürüst gösteriliyor.
- [ ] Bir model release adayı uyumluluk/doğrulama/onay kapısından geçmeden yayımlanamıyor.
- [ ] Colab/Drive varsa kısmi aktarım ve yeniden başlatma negatif testleri geçiyor.
- [ ] GitHub Actions ve README gerçek komutlarla uyumlu; demo adımları başka bir geliştirici tarafından yeniden uygulanabiliyor.

Bu maddelerin bir kısmı F0–F4 minimum kesitinden sonra tamamlanacak. Sağlanmayan madde açıkça `planned`/`incomplete` olarak raporlanır.

---

## 28. İlk sonraki görev

**İlk uygulama görevi:** `F0 — Repo foundation`. Yalnızca tek repo olan `kaiwen-forge` içinde minimal, çalıştırılabilir Python package + `uv` + kalite kontrolleri + basit CLI `doctor` komutunu kur. Sonra `F1 — Recipe validator`, `F2 — Run store` ile devam et. Doğrulanmış çalışma altyapısı oluşmadan kullanıcıyı yüzlerce servis ve soyut arayüzle meşgul etme.

Bu README'nin sonraki belge halkası **`kaiwen-forge/AGENTS.md`** olacak: Forge'a özel Codex görev sınırları, güvenli process/dataset erişimi, test kanıtı ve mimari karar protokolleri burada daha kesin biçimde tanımlanacak.

---

## Kaynak ve tutarlılık notları

Bu README aşağıdaki var olan veya taslak kaynaklarla uyumlu yazılmıştır:

1. GitHub'da 30 Ekim 2026 tarihli gerçek `ImtheKaiwen/kaiwen-forge` repo durumu (tek README).
2. GitHub'daki `ImtheKaiwen/kaiwen-speech` Python kaynak kodu ve mevcut eğitim CLI'ları.
3. Yerel `kaiwen-ai/docs/OVERVIEW.md`, `INTEGRATION.md`, `ENGINEERING_STANDARDS.md`, `ROADMAP.md` taslakları.
4. Yerel Kaiwen Speech dokümantasyon serisi ve güncel 2 GB VRAM / Türkçe speech-native araştırma kararı.

**Önemli:** Bir dosya, komut veya API gerçekten implemente edildiğinde bu README'nin `planlanan` etiketleri güncellenir. Kodla çelişen bir belge teknik gerçekliğin yerine geçmez; çatışma belgelenir, testle doğrulanır ve kontrollü migration uygulanır.
