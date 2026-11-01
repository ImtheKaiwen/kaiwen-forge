# AGENTS.md — Kaiwen Forge Engineering & Research Agent Contract

> **Belge kimliği:** `KAIWEN-FORGE-AGENT-001`  
> **Konum:** `kaiwen-ai/kaiwen-forge/AGENTS.md`  
> **Belge sürümü:** `1.0.0-draft`  
> **Son repo inceleme tarihi:** 30 Ekim 2026  
> **Durum:** `PROPOSED` — Uygulanması hedeflenen geliştirme ajanı çalışma sözleşmesi; uygulanan özelliklerin kanıtı değildir.  
> **Geçerlilik alanı:** `kaiwen-forge` deposunda çalışan Codex, otomatik kodlama ajanları ve geliştiriciler.  
> **Kullanıcıyla iletişim:** Türkçe; kod sembolleri, dosya adları, API'lar ve araç çıktıları gerektiğinde İngilizce.

**Temel talimat:** Kaiwen Forge'u model uygulamasını kopyalamadan, araştırma deneylerini **doğru, tekrarlanabilir, geri alınabilir, izlenebilir ve güvenli** biçimde yürütmeye yarayan bağımsız bir Python araç seti olarak geliştir. Her görevi mevcut kaynak kodu okuyarak, küçük doğrulanabilir değişikliklerle, ölçülmüş kanıtlarla tamamla. Mimariyi geleceğe açık tut; henüz ispatlanmamış gelecekteki gereksinimler için erken soyutlama inşa etme.

Bu belge bir kullanıcı kılavuzundan farklıdır. [`README.md`](README.md) projenin ne olduğunu, hedef arayüzlerini ve yol haritasını anlatır. **Bu `AGENTS.md`, Codex'in belirli bir iş verildiğinde neye bakacağını, neleri değiştirmeyeceğini, neyi test edeceğini, hangi durumda çalışmayı durdurup onay isteyeceğini ve nasıl raporlayacağını söyler.** Teknik uygulama detaylarının uzun biçimi ileride tamamlanacak `docs/` belgelerine aittir.

---

## 0. Kaynak gerçekliği, yetki ve talimat sırası

**30 Ekim 2026 kaynak incelemesi:** [`ImtheKaiwen/kaiwen-forge`](https://github.com/ImtheKaiwen/kaiwen-forge) deposunun `main` dalında yalnızca başlangıç amaçlı, kısa bir `README.md` vardı. Başka bir kaynak, `pyproject.toml`, `src/kaiwen_forge`, çalışan `forge` CLI'sı veya Forge CI workflow'u **yoktu**. Bu tarihsel tespit sabit bir gelecek garantisi değildir; **her yeni görevde repoyu yeniden incele.** Bu doküman ve genişletilmiş README birer uygulama taslağıdır.

Ajan şu bağlamı sırayla değerlendirir:

1. Çalışma ortamının yüksek öncelikli kuralları ve kullanıcının **mevcut somut isteği**.
2. Mevcutsa çalışma alanı kökündeki [`../AGENTS.md`](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/AGENTS.md), ortak [`../docs/ENGINEERING_STANDARDS.md`](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ENGINEERING_STANDARDS.md) ve [`../docs/INTEGRATION.md`](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/INTEGRATION.md).
3. Bu repository'deki **bu** `AGENTS.md`, [`README.md`](README.md), görevle ilgili teknik belge ve karar kayıtları.
4. **Gerçek uygulama durumu için** mevcut dosyalar, Git diff/durum, `pyproject.toml`, lockfile, workflow'lar, testler ve bağımlı kütüphanelerin gerçek API'ları.
5. İşin kapsamını aşan veya belgeler arasında çelişen bir durum varsa değişikliğe başlamadan önce farkı ve seçenekleri açıkla.

Bir dokümanın `planned` demesi kodda özelliğin bulunduğu anlamına gelmez. Kodda bir fonksiyon bulunması da test edildiği ya da üretim kalitesinde olduğu anlamına gelmez. README veya plan, kullanıcının bugünkü isteğini genişletmek için yetki değildir. Belge çelişkisi çözülmeden eski bilgiyi sessizce silme.

### 0.1 Durum sınıflandırması

| Etiket | Kullanım şartı | Kabul edilebilir kanıt |
|---|---|---|
| `IMPLEMENTED` | Somut kod yazıldı ve ilgili doğrulama geçti | Commit/diff, komut, test sonucu |
| `IMPLEMENTED_UNVERIFIED` | Kod var, bu çalışma oturumunda test edilmedi | Dosya/satır kanıtı; başarı iddiası yok |
| `PLANNED` | Onaylı tasarım, henüz kod yok | Karar belgesi ve kabul kriteri |
| `PROPOSED` | Tartışılabilir bir alternatif | Açık varsayım, risk, kanıtlama planı |
| `EXPERIMENTAL` | Kanıt bekleyen yaklaşım | Deney protokolü ve metrik |
| `LEGACY` | Tarihsel/uyumluluk amacıyla korunan yol | Migrasyon gerekçesi ve mevcut kullanıcı |
| `BLOCKED` | Dış veri, erişim, onay ya da teknik önkoşul eksik | Engel, etkisi ve güvenli sonraki adım |
| `DEPRECATED` | Değiştirilen ve kullanım dışına çıkacak API | Sürüm, geçiş yolu, destek süresi |

**Başarı iddiaları:** Başarılı bir `pytest` yalnızca test edilen davranış için kanıttır. Bir modelin Türkçe kalitesi, 2 GB VRAM'de çalışması, Colab'da sınırsız süre çalışması, registry'nin üretime uygunluğu ve tam deterministik eğitim **ölçülmeden iddia edilemez.**

### 0.2 Yerel workspace ve Git sınırı

```text
kaiwen-ai/                      # ortak yerel çalışma alanı; kendisi Git deposu değil
├── README.md
├── AGENTS.md
├── docs/
│   ├── OVERVIEW.md
│   ├── INTEGRATION.md
│   ├── ENGINEERING_STANDARDS.md
│   └── ROADMAP.md
├── kaiwen-speech/              # ayrı Git repository
├── kaiwen-forge/               # bu repository
├── kaiwen-corpus/              # ayrı Git repository
├── kaiwen-live/                # ayrı Git repository
├── kaiwen-design/              # ayrı Git repository
└── kaiwen-agent/               # ayrı Git repository
```

Bu düzen hedeftir. Başka bir ortamda tek başına `kaiwen-forge` açılmışsa kardeş repoların veya üst dizin belgelerinin bulunduğunu varsayma; erişilemeyen kaynağı uydurma. Görev yalnızca Forge ise Speech, Corpus, Live, Design veya Agent kodunu sessizce değiştirme. Kardeş repolara gereken değişiklikleri **ayrı görev / ayrı PR** ve ilgili sahibin onayıyla yap.

### 0.3 İlk araç çalıştırma kontrolü

> Aşağıdakiler **yerel Git klasörü mevcutsa çalıştırılacak keşif komutlarıdır**. Forge'da henüz `uv` paketi olmadığı için bunlar `uv run` yerine kullanılabilir başlangıç kontrolleridir.

```bash
pwd
git status --short
git branch --show-current
git remote -v
git log -1 --oneline
git ls-files
```

Ardından varsa `pyproject.toml`, `.github/workflows/`, `tests/`, `README.md`, `AGENTS.md`, `docs/` ve `src/` dosyalarını incele. Dosya yoksa neden yok olduğunu açıkça bildir; sahte kurulum çıktısı veya test raporu üretme. Çalışma alanında kullanıcıya ait kaydedilmemiş değişiklik varsa bunları koru; değişiklikleri izinsiz sıfırlama.

---

## 1. Proje sorumluluğu: Forge'un sahibi olduğu işler

Forge'un görevi şunlardır:

- Sürüm kontrollü **recipe** doğrulama, çözümleme ve salt-okunur `resolved_recipe` anlık görüntüsü.
- **Run / experiment** oluşturma, kimlik atama, durum geçişleri, süreç yürütme ve iptal.
- Çalıştırma ortamı, kaynak commit'ler, yapılandırma, dataset ve codec kimliklerini kayda alma.
- Deney metriklerini, yapısal raporları ve değerlendirme referanslarını sürümlü biçimde saklama.
- **Artifact** dosyalarını içerik hash'i, sahiplik, saklama politikası ve aktarım bütünlüğüyle yönetme.
- Sürüm uyumlu ve onaylı **model candidate / release registry** yaşam döngüsü.
- **Kaiwen Speech backend adapter** üzerinden mevcut gerçek model eğitimi ve değerlendirmesiyle entegrasyon.
- Yerel CPU işlerinden başlayıp gerektiğinde hosted Colab / Drive gibi uzak ortamlarla güvenli ve dayanıklı aktarım.
- **Sahte sonuç üretmeyen** CLI, Python API, raporlama, test ve CI altyapısı.

Forge'un görevi **değildir**:

- Kaiwen Neural Audio Codec encoder/decoder, RVQ veya codec kayıp fonksiyonunu yeniden yazmak.
- Speech Transformer, codebook prediction, masked loss, sampler veya attention algoritması geliştirmek.
- Speech'in optimizer/trainer'ını ayrı, kalıcı ve çatallanan bir `ForgeTrainer` ile yeniden uygulamak.
- Corpus'un YouTube/ASR/diarization/veri etiketleme masaüstü uygulamasını taşımak.
- Live'ın mikrofon akışını, Electron UI'sini, ses oynatmasını veya kullanıcı oturumlarını yönetmek.
- Agent'ın web arama, bellek, tool permission veya planner iş mantığını sahiplenmek.
- Kaiwen Design'ı depoya kopyalamak, web dashboard veya enterprise SaaS platformunu V1 için zorunlu yapmak.

**Altın kural:** *Speech modelin nasıl eğitileceğini bilir; Forge hangi işi, hangi veriyle, hangi koşulda, hangi kanıtlarla çalıştırdığını bilir.* Bir bileşen iki tarafta da görünüyorsa onu **hangi repo sahipleniyor** sorusuyla mimari sınırı çöz.

### 1.1 Bağımlılık yönü

```text
Kaiwen Corpus  -- versioned dataset export -->  Kaiwen Forge
Kaiwen Forge   -- backend adapter / call ---->  Kaiwen Speech
Kaiwen Forge   -- verified release manifest ->  Kaiwen Live
Kaiwen Live    -- speech runtime interface -->  Kaiwen Speech
Kaiwen Live    -- optional task bridge ----->  Kaiwen Agent
Kaiwen Design  -- reusable UI packages ---->  Live / Corpus
```

- **Forge, Speech'in public API veya mevcut script'lerini kullanabilir; Speech'in Forge'u import etmesi V1 için yasaktır.**
- Forge'un çekirdek domain paketinin Speech import etmesi gerekmez; Speech'e özel kod `integrations/speech/` sınırında tutulur.
- Forge, Live veya Design'a runtime bağımlılığı eklemez. Registry manifesti ortak sözleşmeyle üretilir.
- Corpus'un API ve manifest şeması veri sahibinin sürümlü kontratıdır; Forge yalnızca tüketici, doğrulayıcı ve deney referansı sahibi olur.
- Birden fazla model ailesi desteği ileride hedeflenir, fakat V1'de **yalnızca çalışan Speech adapter'ı** geliştirilir.

### 1.2 Model ailesi eklendiğinde ne yapılır?

Yeni bir backend eklemek için önce mevcut Speech adapter'ının darboğazlarını, gerçek farklılıkları ve ikinci backend gereksinimlerini göster. Bunun ardından ortak protokolde **en az gerekli** yeni alanları tasarla. Tek backend varken 15 soyut factory, plugin marketplace, dynamic import sandbox veya model-agnostic eğitim DSL'i kurma. İleride ihtiyaç doğarsa küçük sınıfları geriye dönük uyumluluğu koruyarak çıkar.

---

## 2. Başlamadan önce zorunlu keşif — 10 maddelik çalışma protokolü

Bir görev geldiğinde şu kontrol listesini uygula:

1. **Somut çıktıyı tanımla:** Kullanıcı hangi davranışı istiyor, bu görev sadece belge mi yoksa çalışan kod mu?
2. **Git durumunu belirle:** Branch, kirli çalışma ağacı, değişen dosyalar, alt modüller/checkout'lar, mevcut PR bağlamı.
3. **Mevcut kaynakları oku:** README ve ilgili teknik belge ile `src/tests/configs` içindeki gerçek kod ve testler.
4. **Sahiplik sınırını kontrol et:** Değişiklik Forge'da mı, Speech veya Corpus'ta mı olmalı?
5. **Teknik belirsizlikleri tespit et:** Gerçek CLI argümanları, dataset manifest sürümü, checkpoint I/O, cihaz davranışı, erişimler.
6. **Minimum doğrulanabilir dilimi belirle:** İlk küçük testle gösterilecek çalışan sonuç nedir?
7. **Risk ve onay gereksinimini yaz:** Bulut ücreti, internetten veri alma, repo push, release, veri silme, kimlik bilgisi, kişisel ses verisi.
8. **Hangi testlerin yazılacağını seç:** Negatif durum, crash/restart, idempotence, checksum, schema uyumu dahil.
9. **Önerilen diff'i sınırla:** Gereksiz dependency, taşıma, rename ve component ekleme.
10. **İş sonunda kanıt üret:** Yalnızca gerçekten koşan komutları, exit code'ları ve sonuçlarını raporla.

Kullanıcının isteği küçükse bu kontrolü kısa yap; uzun plan yazma zorunluluğu bir işi geciktirmek için bahane değildir. Güvenlik ve veri bütünlüğü kontrolleri kapsam küçük diye atlanmaz.

### 2.1 İlk görev olan repo temeli için sıra

Forge deposu boşsa ilk hedef: gerçek, kurulabilir, test edilen küçük Python paketi.

1. `pyproject.toml` ve `src/kaiwen_forge/` ekle; hedef Python sürümünü desteklenebilen ortamla doğrula.
2. `uv.lock` oluştur ve CI'de aynı bağımlılık çözümünü kullan.
3. Minimal `__init__.py`, tipli domain modeli ve birim testi ekle.
4. `uv sync --extra dev`, `uv run pytest`, `uv run ruff check .`, `uv run pyright` akışını gerçekten çalıştır.
5. GitHub Actions CI ekle; başarıyı CI raporundan doğrula, yalnızca YAML yazmayı başarı sayma.
6. Sonraki görevde recipe parser ve run store; daha sonra local executor ve Speech adapter.

**Tek hamlede F0–F9 uygulanacak diye paket temelini gereksiz büyük tasarlama.** İlk doğru çalışan dilimi elde et; sonra testlerle genişlet.

---

## 3. Kod stili ve Python kalitesi

**Hedef:** Python 3.11+ destekli, `src` layout kullanan, bağımsız kurulan, sıkı tiplendirmeli, az gizli yan etkili paket. Kesin desteklenen Python sürümleri kod/CI doğrulamasıyla belirlenir; PyTorch/CUDA paketleri uygun platformda ayrı ele alınır.

- Yeni public fonksiyonlar dönüş/girdi tipleriyle ve amaç/hata davranışıyla açıklanır.
- Domain modellerinde `dataclass(frozen=True, slots=True)` veya ihtiyaca uygun doğrulanan tipli modeller tercih edilir; değişmez snapshot kuralları esastır.
- Gizli global mutable state, import sırasında ağ çağrısı, model indirme veya CUDA başlatma yapılmaz.
- Dosya yollarını `pathlib.Path` ile yönet; Windows/Linux ayrımını ve separator farklarını test et.
- Yapılandırma için `yaml.safe_load` veya güvenli doğrulayıcı kullan; keyfi Python nesnesi oluşturabilecek loader kullanma.
- Hata mesajlarında hassas bilgi yazmadan **hangi invariant'ın bozulduğunu** belirt.
- Bir çalışanı (worker) çağırmadan önce girdiyi doğrula; geçersiz recipe ile GPU başlatma.
- Kapatılmış DB/stream/process'e tekrar erişim sessiz başarısızlık olarak geçmez.
- Sessiz `except Exception: pass`, kontrolsüz `assert`, maskeleme için `# type: ignore` ve bilinmeyen metrikleri `0` yapma alışkanlığını reddet.
- Büyük JSON, log, checkpoint veya ses dosyalarını tümüyle belleğe almak yerine stream/chunk kullan.
- Kodu test edilebilir bağımlılık sınırlarıyla yaz; ayrı framework icat ederek basit I/O'yu anlaşılmazlaştırma.
- Dependency pin/lock, sürüm çakışması ve opsiyonel GPU/Speech extras'ı bilinçli yönet.

### 3.1 Başlangıç kalite komutları — paket kurulunca geçerli

```bash
uv sync --extra dev
uv run ruff check .
uv run pyright
uv run pytest
uv build
```

**Bugün geçerli Forge komutları oldukları iddia edilmez.** `pyproject.toml`, lockfile ve scriptler oluşturulup doğrulanınca bu komutlar CI'de kullanılacak. Araç yoksa nedenini belirt; başarısız testin üzerine “geçti” yazma.

### 3.2 Public API değiştirme kuralı

Bir public isim/alan değişikliğinde:

1. Mevcut API kullanımlarını ara.
2. Gerekirse geriye dönük alias/deprecation tasarla.
3. Recipe, run store, release manifest ve test fixture'larındaki etkileri göster.
4. SemVer politikasına göre sürüm yükselt, migration yaz.
5. İkinci repo kullanıcılarını (Speech/Live) bilgilendir; otomatik çapraz repo değiştirme.
6. Eski fixture ile uyum testini çalıştır; kırıcı değişiklik gerekçesini kaydet.

---

## 4. Domain sözleşmeleri: recipe, run, experiment, artifact

**Forge'un V1 domain'i** beş temel nesneyle anlaşılabilir: `Recipe`, `ResolvedRun`, `RunState`, `MetricEvent`, `ArtifactRef`. Modelin tamamını hiçbirini uygulamadan 30 abstract class olarak yazma. İhtiyaç oluşunca `Evaluation` ve `ModelRelease` ekle.

### 4.1 Recipe'nin değişmezleri

- `schema_version` zorunlu, desteklenmeyen major sürüm reddedilir.
- `recipe_id` ve hedef `family/backend` doğrulanır; V1 yalnızca Speech backend'in **gerçek desteklediği** görevleri kabul eder.
- Dataset ve codec referansları manifest + sabit digest ile kimliklenir; `latest`, `main` veya rastgele URL yalnızca **çözümleme girdisi**, kesin çalışma kimliği değildir.
- Kullanıcıya ait özgün YAML ile CLI/env override sonrası **çözülmüş immutable snapshot** ayrı tutulur.
- Sır, API token ve oturum kimliği snapshot'a girmez.
- `precision`, `device`, timeout, output root ve dataset split çalıştırma başlamadan kontrol edilir.
- Bilinmeyen kritik alanlar sessizce yutulmaz; hatalı field adı için açık validation error verilir.
- Duplicate YAML key, bozuk tip, negatif step, sıfır batch, bilinmeyen scheduler açıkça reddedilir.
- Recipe path'leri çalışma dizini rastgele değişince farklı veriyi işaret etmeyecek biçimde normalize edilir; çözülmüş yolların erişim alanı doğrulanır.
- Recipe değişikliği çalışan run'a sızmaz. Yeniden deneme, mevcut run snapshot'ı ile uyumluluk kontrolünden sonra olur.

### 4.2 Hedef recipe örneği — **tasarım, çalışır CLI garantisi değil**

```yaml
schema_version: "1.0"
recipe_id: "speech-continuation-cpu-smoke"
family: "speech"
backend: "kaiwen-speech"
source:
  git_repository: "https://github.com/ImtheKaiwen/kaiwen-speech"
  git_commit: "<resolve-before-run>"
input:
  dataset_manifest: "datasets/turkish-pilot/dataset.lock"
  dataset_digest: "<verify-before-run>"
  split: "train"
  codec_id: "mimi-baseline"
  codec_revision: "<pin-before-run>"
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

`<...>` alanları **geliştirici taslağıdır**; bunlar çözülmeden eğitim başlatılamaz. Bu örneğin `dataset_manifest` gerektirmesiyle Speech'in mevcut `--synthetic` smoke yolu farklıdır. İlk Forge smoke adapter'ı sentetik kaynağı açık bir `input.kind: synthetic` gibi **ayrı sürümlü recipe şemasına** tanımlayabilir; var olmayan dataset için sahte manifest üretme. Bu farklılığı yeni şema tanımında ve testlerde çöz; doğrulanmamış alanları kendiliğinden “destekleniyor” ilan etme.

### 4.3 Run kimliği ve olayları

- `run_id` benzersiz ve kullanıcıdan bağımsız üretilir; aynı iş başlatma isteğine ait idempotency key kullanılır.
- `experiment_id` mantıksal çalışma grubudur; `run_id` onun gerçek bir tekrarıdır.
- Her run için `created_at_utc`, `updated_at_utc`, `status`, `recipe_digest`, `backend_revision`, `dataset_digest`, `error_class`, `last_checkpoint_ref` tutulması hedeflenir.
- Zaman damgaları UTC ve timezone-aware olur; terminalde yerel saat gösterilebilir.
- Olaylar sıra numarası ve kaynak kimliği ile takip edilir; worker yeniden bağlanınca eski olaylar yanlışlıkla iki kez işlenmez.
- Durum değişimini DB transaction ile yap; terminal durumu yalnızca doğrulanmış gözleme bağla.

### 4.4 Metric event geçerliliği

Her metrik için minimum alanlar: `run_id`, `step`, `timestamp_utc`, `name`, `value`, `unit`, `split`, `metric_definition_version`, `source`.

- `NaN`, `inf` veya string'i sessizce floating-point başarı metriklerine dönüştürme.
- `training/loss` ile `evaluation/mos` farklı kanıtlardır.
- GPU peak memory `MB` mi `MiB` mi; süre `ms` mi `s` mi açık olmalı.
- Veri yoksa `null`/`unmeasured` raporla; tahmini `0 ms` veya `100%` yazma.
- Tek bir sonuç ölçüm için kullanılan bütün örnekleme/seed/cihaz protokolünün yerini tutmaz.
- Metrik writer yarıda kalırsa son doğrulanmış cursor'dan devam eder; duplicate olayları idempotent ele alır.

### 4.5 Artifact kimliği

`artifact_id`, `role`, `size_bytes`, `sha256`, `content_type/format`, `location_uri`, `owner_run_id`, `created_at_utc`, `verified_at_utc` ve gerekiyorsa model/codec sürümleri tutulur. Yalnızca dosya uzantısı modelin güvenilirliği hakkında kanıt değildir. Artifact ID ile dosyanın fiziksel konumu farklı kimliklerdir; yolu değişen ama byte içeriği değişmeyen dosya aynı digest'i korumalıdır.

---

## 5. Run durum makinesi ve idempotence kuralları

Başlangıç önerisi:

```text
created --validate--> validated --enqueue--> queued --start--> running
   |                  |                           |             |
 rejected           rejected                   cancelled        +--> completed
                                                               +--> failed
                                                               +--> interrupted
                                                               +--> cancelling --> cancelled
```

Gerçek implementasyon için **izinli geçiş tablosu** tek bir yerde tanımlanır. DB, CLI ve worker aynı transition validator'ı kullanır.

1. `running -> completed` ancak süreç çıkışı, zorunlu artifact ve gerekli doğrulama birlikte başarılıysa olabilir.
2. `running -> interrupted` süreç/heartbeat kaybında uygundur; otomatik `completed` çıkarımı yasaktır.
3. `cancelling -> cancelled` worker/alt süreç kapatılıp kaynaklar bırakılmadan kabul edilmez.
4. `failed` veya `cancelled` terminal durumu, aynı run'ın sessizce yeniden `running` yapılmasıyla silinmez. Resume denemesi yeni attempt/generation veya açık sürümlemeyle belgelenir.
5. `queued` işler aynı anda iki worker tarafından sahiplenilmez; atomik claim ve lease mekanizması uygulanır.
6. Aynı `run_id` ve aynı attempt'e iki farklı checkpoint yazarı atanmaz.
7. `completed` run'ın snapshot, logs, metrics ve artifacts'ı sonradan sessizce değiştirilmez.
8. Başlatma, iptal, devam, dış işlem sonlandırma, güç kaybı ve backend exception için negatif test yazılır.

### 5.1 Crash recovery için doğrulama yöntemi

Testte gerçek süreci ortasından öldür; yalnızca bir exception fırlatıp recovery çalıştı deme. Yeniden başlatılan Forge, DB'deki `running` durumu ile OS process ve heartbeat kanıtını uzlaştırmalı; orphan subprocess'i yanlışlıkla yeni worker sanmamalı. Crash sırasında bitmeyen `.tmp` artifact'i aktif artifact olarak kaydetme. Yeni koşuda mevcut checkpoint'i yalnızca hash ve format doğrulamasından sonra `resume` için seç.

### 5.2 İptal davranışı

- İptal iki aşamalıdır: **isteği kaydet → çalışan süreci uygun yöntemle sonlandır → durumu doğrula**.
- Önce grace period ile temiz sonlandırma; timeout sonrası OS-aile process cleanup politikasını uygula.
- Windows ve Linux'ta process tree yönetimi farklıdır; gerçek platform testleri olmadan genel iddia kurma.
- Worker iptali checkpoint oluşturmuşsa checkpoint'in bütünlüğü ayrı doğrulanır; dosya varlığını başarı sayma.
- `cancel` tekrar çağrıldığında idempotent ve anlaşılır sonuç üret; log veya artifact'i yetkisiz silme.

---

## 6. Execution backend — güvenli yerel worker ile başla

**V1 varsayılanı:** tek makine, tek local executor, sınırlı concurrency. SQLite + subprocess + açık timeout gerçek ihtiyacı karşılıyorsa Redis, Celery, Kubernetes veya dağıtık lock servisi ekleme. Yük artarsa profil ve gerekçe ile genişlet.

### 6.1 Komut oluşturma güvenliği

- Kullanıcıdan gelen recipe'yi shell string'ine doğrudan ekleme: `shell=True` veya `sh -c` ile keyfi çalıştırma yok.
- Argüman listesiyle `subprocess` çağır; executable, working directory, environment ve dosya yazma kökünü allowlist/politika üzerinden seç.
- Kullanıcının verdiği uzak Git URL'sini veya arbitrary Python module path'ini güvenilir plugin gibi otomatik çalıştırma.
- İlgili Speech commit'i ve bağımlılıkları önceden doğrulanmış çalışma ortamında çağır.
- Girdi recipe'si doğrulanmadıysa iş `queued/running` durumuna geçemez.
- İşin CPU/GPU, maksimum süre ve çıkış alanı limitlerini önceden belirle.
- STDOUT/STDERR stream et; sınırsız RAM tamponu oluşturma ve hassas verileri rapora kopyalama.
- CLI exit code, termination reason, duration ve failure stage kaydedilir.
- Aynı subprocess’in stdout sırasına göre event reconstruction yapılıyorsa log formatına sürüm ekle; `print` çıktısı kırılgan bir bağımlılıktır.

### 6.2 Hata sınıfları

| Sınıf | Örnek | Politika |
|---|---|---|
| `ValidationError` | Recipe schema, dtype veya split yanlış | Fail-fast; otomatik retry yok |
| `CompatibilityError` | Codec/model/token uyuşmuyor | Fail-fast; kullanıcı düzeltmesi |
| `PermissionError` | Artifact dizinine yazılamıyor | Güvenli hata, secret sızmadan rapor |
| `ResourceError` | Disk dolu, CUDA OOM | Kaynakları serbest bırak, ortamı kaydet, kör retry yok |
| `TransientIOError` | Kısa ağ/mount kesintisi | Sınırlı exponential backoff + jitter |
| `BackendFailure` | Speech CLI nonzero exit | Exit code/stderr özeti, aşama ve log referansı |
| `UserCancelled` | Açık iptal isteği | Temiz cleanup, `cancelled` durumu |
| `Interrupted` | Worker/Colab bağlantı kaybı | Son doğrulanmış checkpoint, kontrol edilmeden başarı yok |
| `IntegrityError` | Hash veya boyut değişmiş | Artifact quarantine, release engeli |

Çok genel bir `Exception` yakalamak gerekirse sınırda loglayıp tipli domain hatasına çevir; istisnayı yok etme. Retry yalnızca **tekrar edilmesi güvenli ve idempotent** işlemler için kullanılır. Bir eğitim adımını körlemesine yeniden yürütmek, aynı optimizer state'ini farklılaştırabilir; `resume` her zaman checkpoint kontrollüdür.

### 6.3 Kaynak rezervasyonu ve bütçe

- GPU modeli, VRAM ve precision sorgula; `torch.cuda.is_available()` tek başına kapasite garantisi değildir.
- 2 GB VRAM **Kaiwen Speech inference araştırma hedefidir**, Forge eğitim işinin 2 GB VRAM ile tamamlanacağı garantisi değildir.
- İş için tahmini disk/cache alanı ve expected artifact hacmi göster; yazma sırasında disk dolma durumunu test et.
- Bir GPU'ya iki ağır eğitim otomatik yığılmaz; concurrency varsayılanı ölçümle değiştirilir.
- Büyük indirme veya ücretli GPU işi için açık onay ve ölçek/bütçe görünürlüğü gerekir.
- Execution timeout sonrası gerçek worker ve child process kalmadığı doğrulanır.

---

## 7. Kaiwen Speech adapter: hangi kod gerçekten mevcut?

**2026-10-30 kaynak incelemesine göre** Speech GitHub repository adı `kaiwen-speech` fakat Python dağıtımının adı `kaiwen-voice`, modül adı **`kaiwen_voice`** idi. GitHub repo rename Python importunu değiştirmez.

Speech tarafında aşağıdaki mevcut kaynakları referans al:

```text
../kaiwen-speech/pyproject.toml
../kaiwen-speech/src/kaiwen_voice/model/config.py
../kaiwen-speech/src/kaiwen_voice/model/transformer.py
../kaiwen-speech/src/kaiwen_voice/training/trainer.py
../kaiwen-speech/src/kaiwen_voice/training/checkpoint.py
../kaiwen-speech/src/kaiwen_voice/data/token_cache.py
../kaiwen-speech/scripts/train_continuation.py
../kaiwen-speech/scripts/train_dialogue.py
../kaiwen-speech/scripts/benchmark.py
../kaiwen-speech/.github/workflows/test.yml
```

**Önce yeniden doğrula:** Bu yollar geçmiş incelemedeki somut dosyalardır; sonradan migration yapılmış olabilir.

### 7.1 Mevcut çalıştırılabilir baseline — Speech repo root'unda

```bash
uv sync --extra dev
uv run ruff check .
uv run pyright
uv run pytest
uv run python scripts/train_continuation.py \
  --config configs/smoke.yaml \
  --synthetic \
  --steps 100 \
  --overfit
```

Bu komutlar **Speech'e aittir**, bugün çalışan Forge CLI'ları değildir. `uv` kurulumu, Python ve bağımlılık çözümü ortamdan etkilenebilir; test sonucunu gerçekten çalıştırmadan raporlama. Baseline'daki overfit, modelin doğal Türkçe konuştuğunu ispatlamaz.

### 7.2 Adapter uygulama kuralı

1. Speech'in mevcut CLI veya public API argümanlarını **kaynak kodundan** oku.
2. V1 için `train_continuation.py --synthetic` veya kontrollü token-cache eğitimlerinden biriyle dar bir entegrasyon başlat.
3. `SpeechBackendAdapter` kontratında örneğin `validate`, `prepare_command`, `collect_metrics`, `collect_artifacts`, `probe_capabilities` gibi küçük işlevleri kullan; bu isimlerin henüz Forge'da yazılı olmadığını açıkla.
4. `prepare_command` allowlist argüman listesi döndürür; modellenen config ve CLI argümanları gerçek script ile uyumlu olur.
5. `collect_metrics` kaynağını (structured output, log parser, event dosyası) versiyonla; terminal çıktısı değiştiğinde test hata vermelidir.
6. Checkpoint için Speech'in gerçek format ve metadata'sını doğrula; Forge kendi hayali checkpoint formatını yazmasın.
7. Real codec meta (`name`, `sample_rate`, `frame_rate`, `num_codebooks`, `codebook_size`) model config'iyle karşılaştır; **eşit şekil, eşit semantik değil**.
8. `--resume` ile `--init-from` arasındaki farkı koru: tam eğitim state'ini sürdürme ve ağırlıklarla başlangıç yapma aynı değildir.
9. Speech'in kendi trainer/loss/optimizer kodunu kopyalama. Ortak eksiklik varsa Speech repo için ayrı issue/PR öner.
10. Adapter'ın mümkün olduğunca backend sözleşmesini fixture/fake ile test et; gerçek model gerektiren testleri opsiyonel etiketle.

### 7.3 Training config ve recipe arası çeviri

Forge recipe katmanı, Speech `TrainingConfig` ve `SpeechTransformerConfig` nesnelerine kontrollü eşleme yapar. Speech config şemasını Forge'da sabit bir Python sınıfına kopyalayıp iki farklı kaynak oluşturma. Mümkünse Speech'in yayınlanmış doğrulamasını kullan; değilse sürümü belli, testli adapter uyumluluk katmanı yaz.

- `model.codebook_size`, `num_codebooks`, `codebook_modeling`, `acoustic_delay_frames` ve codec metadata uyuşmazsa fail-fast.
- `max_frames` ve dataset sekans uzunluğu bir arada doğrulanır.
- `--synthetic` ve `--token-cache` birbirini dışlayan gerçek CLI seçenekleridir; geçersiz kombinasyon üretme.
- Resume checkpoint modeli ile yeni recipe'nin config uyumu doğrulanır.
- Gelecekte GQA veya native KV cache gelmiş gibi çalıştırma bayrağı üretme.

---

## 8. Corpus dataset handoff ve etik veriler

Corpus'un görevi kaynakları toplamak, lisans/provenance ve kalite etiketleriyle sürümlü dataset **export** hazırlamak; Forge'un görevi bu export'u **eğitime kabul edilmeden önce** doğrulamaktır. Veri erişimi yalnızca manifest bulunduğu için otomatik onaylanmış sayılmaz.

### 8.1 Giriş doğrulama listesi

- Manifest `schema_version`, `dataset_id`, `revision`, `export_id`, `created_at_utc` ve `sha256` kimliği.
- Her kayıt için güvenilir source ID, içerik hash'i, ses formatı, sample rate, kanal, süre, split ve kalite/review bilgisi.
- Lisans koşulları, attribution ve kullanım hakkı; özel veri/onay ve paylaşım sınırları.
- Aynı kaynağın train/test'e düşmesini engelleyen leakage grupları (speaker, conversation, source, duplicate family).
- Kayıt bozulması, eksik dosya, bozuk JSONL, aynı kayıt ID'si, path traversal, hash mismatch.
- Asgari konuşma dili `tr` ve uygun veri türü; otomatik etiketin insan doğrulaması gibi sunulmaması.
- Codec için gereken **orijinal 24 kHz ya da üstü gerçek ses kaydı**; Corpus'un bugünkü 16 kHz işleme çıktısı orijinal yüksek frekans içeriği yerine geçmez.
- Yaşam döngüsü: `approved`, `rejected`, `quarantined` ve `unknown` gibi durumlar doğru ele alınır.

### 8.2 Veri sızıntısını önleme

`record_id` hash'ine göre rastgele split tek başına yeterli olmayabilir. Aynı kayıt farklı bölümlere bölünmüş, aynı ses başka video olarak yüklenmiş veya aynı konuşmacı çok sayıda kayıtta yer almış olabilir. Dedupe ve leakage family kuralları Corpus tarafında uygulanır; Forge **export'un split kanıtını** ve değerlendirme protokolünü doğrular. Korunması gereken grup birimi deney protokolünde açık yazılmalıdır.

### 8.3 Veri erişim politikası

- Özel ses kayıtlarını ve kişisel bilgileri varsayılan olarak uzak servise yükleme.
- Dataset lisansı bilinmiyorsa eğitim başlamadan işaretle veya reddet.
- Transcript ve speaker ID kişisel veri taşıyabilir; gereksiz loglama ve herkese açık artifact yok.
- Derlenen dataset manifesti, kaynak ses ile aynı paylaşım hakkına otomatik sahip değildir.
- Kullanıcı lisans ihlali doğurabilecek toplu veri çekimi istediğinde hukuk/izin doğrulaması olmadan otomasyona sokma.

---

## 9. Artifact store ve dosya sistemi güvenliği

**Önce yerel dosya sistemi ve SQLite WAL.** Artifact içerikleri büyük binary dosyalardır; veritabanında yalnızca kimlik, metadata, hash, durum ve referanslarını tut.

### 9.1 Atomik yazma ve doğrulama

1. Yazma izni olan özel run kökünde benzersiz `.partial`/`.tmp` yoluna yaz.
2. Dosyayı kapat; gerekirse dayanıklılık politikası kapsamında flush/fsync uygula.
3. Boyut, format ve SHA-256 doğrula.
4. Aynı dosya sistemi içindeyse atomik rename ile son konuma geçir.
5. Son dosyayı ve digest'i doğruladıktan sonra DB transaction ile görünür `artifact_ref` kaydı oluştur.
6. Crash sonrası `.partial` dosyaları otomatik `valid` sayma; quarantine/cleanup politikası uygula.
7. Yerel veya uzak aktarım tekrarlanırsa içerik digest'i ile **idempotent** davran.
8. Var olan, doğrulanmış artifact dosyasını aynı isimli yeni içerikle sessizce üzerine yazma.

### 9.2 Yollar ve silme

- `root / user_relative_path` kullanmadan önce resolved path'in izin verilen kök içinde kaldığını denetle.
- Symlink/junction ve Windows path özel durumlarına karşı test ekle.
- README, model veya ses dosyasını silmek için yalnızca kullanıcı tarafından verilen string path'e güvenme.
- Auto-retention cleanup yalnızca açık politika ve `dry-run` ile başlar; yayınlanmış release'leri yanlışlıkla silmez.
- Git'e ağırlık, WAV, checkpoint, `.env`, HF cache, token veya gerçek kullanıcı ses verisi commit etme.
- Anonimleştirilmiş örnek bile gerçekten anonim değilse herkese açık PR'a eklenmez.

### 9.3 PyTorch güvenliği

Eski Speech checkpoint'leri `.pt` olarak saklanabilir. `torch.load` deserialization saldırı yüzeyi olduğundan belirsiz kaynaktan checkpoint yürütme/restore yapma. Güvenilir origin + sabit SHA + izin politikası uygula, uygun olduğunda `weights_only=True` gibi sınırlı yükleme kullan. `weights_only=True` tek başına güvenilirlik kanıtı değildir. Eğitim checkpoint'inde optimizer state bulunabilir; Live'ın inference paketine gerekli olmayan training state'i yerleştirme.

---

## 10. Deney kayıtları ve bilimsel tekrarlanabilirlik

Bir run'ın tekrar edilebilirliği yalnızca `seed=42` yazmak değildir. En az şu bilgiler tamamlandığında sonuç izlenebilir kabul edilir:

| Alan | Kaynak | Kontrol |
|---|---|---|
| Forge commit + paket sürümü | Forge Git/metadata | Full SHA + dirty state |
| Speech commit + backend API | Speech Git/install | Full SHA + import yolu |
| Recipe digest ve resolved snapshot | Validator | İmmutable + secret redaction |
| Dataset export ID + SHA + split | Corpus export | İçerik doğrulanmış |
| Codec family/revision/token layout | Speech codec metadata | Checkpoint uyumu |
| Model config + parametre sayısı | Speech | Gerçek modelden ölçülmüş |
| Seed + deterministic ayarları | Çalıştırma | İstisnalar kayıtlı |
| OS, Python, PyTorch, CUDA, GPU | Çalıştırma | Gerçek runtime probe |
| Optimizer/scheduler/precision | Speech trainer | Resolved değerler |
| Adım, süre, hata, son checkpoint | Worker | Olay logu ve hash |
| Evaluation suite version | Evaluation | Dataset/metric/protokol |
| Artifact boyutu ve hash | Store | Transfer sonrası tekrar doğrulanmış |

### 10.1 Deterministik sonuç konusunda dürüstlük

- RNG seed'leri ve sample ordering sabitlenir; ancak GPU kernel nondeterminism, mixed precision, driver ve runtime farkları bit-for-bit tekrarı engelleyebilir.
- Tek bir eğitim tekrarını “bilimsel olarak tamamen deterministik” diye raporlama.
- Deney karşılaştırmasında değişen bağımsız değişken dışında koşullar mümkün olduğunca sabit tutulur.
- Depth model farklı parametre sayısına sahipse maliyet/performance farkını tabloda belirt.
- Eksik deneyleri 0 olarak kaydetme; `unmeasured`, `failed`, `skipped` ayrı olabilir.
- İstatistiksel varyansı görmek için gerektiğinde birden fazla seed, güven aralığı ve öznel ses kalitesinde kör değerlendirme kullan.

### 10.2 Deney karşılaştırma kapısı

İki run karşılaştırılmadan önce şu alanları kontrol et: dataset digest, codec revision, evaluation suite version, input profile, decoding sampling parametreleri, cihaz, precision ve benchmark koşulları. Eşdeğer değillerse “karşılaştırılamaz” veya sınırlamalarını açıkça göster. İki eğitim `loss` değerinin farklı tokenizasyon/codebook konfigürasyonlarında doğrudan anlamlı olmadığını belirt.

---

## 11. Değerlendirme ve benchmark sorumlulukları

Forge, **değerlendirme protokolünü yürütür ve ölçümü kaydeder**; Speech'in ses kalitesini hesaplamak için gereken tüm algoritmaları Forge'a kopyalamaz.

| Değerlendirme | Sahip olan implementasyon | Forge rolü |
|---|---|---|
| Checkpoint yapısal doğrulama | Speech checkpoint I/O | Uyum kontrolü ve rapor |
| Training/validation loss | Speech trainer | Metrik serisini ve dataset kimliğini kaydetme |
| Codec reconstruction | Speech codec eval | Aynı test setinde çıktı/rapor toplama |
| Türkçe konuşma anlaşılırlığı | Speech eval + bağımsız test seti | Suite, ölçüm protokolü, insan değerlendirmesi referansı |
| Streaming TTFA/RTF/VRAM | Speech runtime + ölçüm harness | Ortamı sabitleme ve sonuç karşılaştırması |
| Registry release gate | Forge | Gerekli test referanslarını ve approval kontrolü |

### 11.1 Ölçümler için kurallar

- Benchmark'ta model warm-up, CUDA synchronization, tekrar sayısı ve cihaz kimliğini kaydet.
- TTFA tanımını netleştir: ilk üretilen token mı, ilk decode edilen PCM mi, hoparlörden ilk duyulan ses mi? Bu üçü aynı değildir.
- RTF'i codec encoder, model ve decoder ile uçtan uca değerlendir; hangi bileşenlerin dahil edildiği açık olsun.
- `peak_vram` GPU process / framework allocation / nvidia-smi gözlemleri olarak ayrılabilir; birbirine karıştırma.
- CPU-only CI'de GPU hızını sahte sayı üretip raporlama.
- Sesi dinleyen insanlarla test yapılıyorsa izin, örneklerin anonimliği, kör karşılaştırma ve değerlendirici sayısı yazılır.
- Test seti eğitim kararlarına sürekli ayar yapmak için kullanılacaksa ayrılmış nihai holdout seti korunur.

---

## 12. Model registry, release ve rollback

Forge model çıktısını kayda almakla kalmaz; Kaiwen Live'a **hangi model ile hangi codec'in birlikte yüklenebileceğini** güvenli biçimde tarif eder. Ama bir checkpoint dosyası bulunması, **release tamam** demek değildir.

### 12.1 Yaşam döngüsü

```text
training checkpoint
  -> verified artifact
  -> evaluated candidate
  -> explicit approval
  -> immutable released manifest
  -> Kaiwen Live compatibility validation
  -> retained previous release / rollback
```

- `candidate` ve `released` farklı durumlardır.
- Promotion için model ve codec artifact hash'leri, schema/API sürümleri, kalite suite referansları ve gerekirse insan onayı gereklidir.
- Placeholder içeren manifest **release olamaz**.
- Release immutable'dır; düzeltme gerektiğinde yeni version oluştur.
- Promotion sırasında registry metadata ile disk içeriği atomik ve tutarlı güncellenir.
- Model ağırlığı ile codec checkpoint ayrı artifact olabilir; ikisinin de doğrulaması gerekir.
- Speech paket dağıtım adı ve Python import yolu migration aşamasında farklı olabilir; manifestte gerçek uyum bilgisi bulunur.
- `codec.num_codebooks=4` ile `model.num_codebooks=4` eşit diye semantik uyumluluk kanıtlanmaz; trained codec revision kimliği gerekir.
- Revoked/superseded sürümlere geri dönüş bilinçli ve kayıtlı olmalıdır; güvenlik nedeniyle iptal edilmiş sürüm otomatik rollback hedefi değildir.

### 12.2 Güvenli promotion önkoşulları

```text
[ ] Model ağırlıkları güvenilir origin'den geliyor
[ ] SHA-256 ve boyut doğrulandı
[ ] Codec artifact ve revision gerçekten mevcut
[ ] Model–codec compatibility testi geçti
[ ] İlgili evaluation suite tamamlandı
[ ] Release metadata'da placeholder yok
[ ] Live runtime desteklenen API ve platforma sahip
[ ] Approval ve audit kaydı var
[ ] Önceki onaylı release'e dönüş yolu belli
```

**Bir kodlama ajanı kendi hazırladığı aday modeli kullanıcı onayı veya açık tanımlı güvenli promotion politikası olmadan otomatik olarak herkese açık release etmez.** `main` branch'e push veya GitHub Release oluşturma ayrı yetkiler gerektirir.

---

## 13. Hosted Colab ve Google Drive çalışma protokolü

İlk sürümde Colab desteği **manuel başlatılmış, tekrarlanabilir notebook/script + dışarıdan artifact import** şeklinde başlayabilir. Colab, sonsuza kadar yaşayan worker veya güvenilir yerel filesystem olarak ele alınmaz.

### 13.1 Hazırlık ve çalıştırma

1. Hosted Colab runtime'ın gerçekten kullanıldığını doğrula. Kullanıcının laptopuna veri indirmeyi gizlice fallback yapma.
2. Dataset erişimi/indirme iznini ve kaynağın lisansını kontrol et.
3. Runtime disk boşluğu, Drive hedefi ve cache gereksinimini hesapla.
4. Paket ve model sürümlerini, commit SHA'yı ve resolved recipe'yi sabitle.
5. Büyük veri aktarımını küçük parçalarla, manifest ve hash kaydıyla yap.
6. Hızlı scratch I/O gereken eğitim adımlarını mümkün olduğunca runtime yerel diskinde yap; dayanıklı artifact'leri Drive'a taşı.
7. Checkpoint upload tamamlanmadıkça run `completed` olmaz; `verified_at_utc` kanıtı gerekir.
8. Notebook bağlantısı koparsa **bağlantı kaybı**, **worker'ın ölmesi** ve **eğitimin başarıyla bitmesi** birbirinden ayrılmalıdır.
9. Drive mount kesintisi, kota veya hız düşüklüğünde fail-safe davran; eksik dosyayı kullanma.
10. Uzaktan aktarılan run manifesti, yerelde checksum ve schema doğrulanmadan registry adayı yapılmaz.

### 13.2 Kesinti senaryoları

| Senaryo | Beklenen davranış |
|---|---|
| Colab süresi doldu | Son doğrulanmış checkpoint ve event dizisiyle `interrupted` |
| Drive mount düştü | Upload başarısız; partial dosya quarantine; metadata doğrulanır |
| Hash yanlış | İndirme/aktarım reddedilir; sınırlı retry yalnızca idempotent |
| Disk doldu | Kontrolsüz overwrite veya sessiz veri kaybı yok |
| Notebook tekrar çalıştırıldı | Çift run oluşturulmaz; açık resume veya yeni attempt |
| Kaynak dataset değişti | Eski run digest'i korunur; veri yeni sürüm kabul edilir |
| Colab GPU sağlanmadı | İş başarısız/blocked veya kullanıcı onayıyla CPU'a geçiş |
| Kullanıcı tokenı yok | Yetkisiz erişim yerine açıklayıcı auth hatası |

**İzin ve gizlilik:** Drive alanının büyük olması veri aktarımının yasal olduğu anlamına gelmez. İşleme izinleri, lisans, kişisel veri ve depolama güvenliği ayrı doğrulanır. Colab CPU/GPU runtime ve hizmet kotaları zamanla değişebilir; kod sabit kapasite garantisi içermez.

### 13.3 Remote worker güvenlik sınırı

Açık kullanıcının istediği gerçek uzaktan yürütme yapılmadıkça otomatik zamanlanmış Colab job, gizli ağ servisi veya kimlik bilgisi isteyen agent kurma. Drive mount yolu ve OAuth tokenları artifact manifestine kaydetme. Remote ortamdan dönen notebook dosyasını doğrudan yürütülebilir kod olarak güvenme; sonuç manifestini doğrula.

---

## 14. SQLite ve kalıcılık standartları

V1 için `SQLite WAL` yeterli başlangıç noktasıdır; eşzamanlılık gereksinimini ölçmeden uzak veritabanı ekleme.

- Schema version ve **migrasyon dosyaları** bulunur. Kolon eklemek rastgele startup SQL ile geçiştirilmez.
- Foreign key, uniqueness, idempotency ve durum geçişleri uygulama + DB düzeyinde korunur.
- Metric event veya run state update DB transaction ile yazılır; kısmi başarı tutarsız state oluşturmaz.
- Uzun süreli DB transaction içinde subprocess bekleme veya GPU eğitimi yapma.
- WAL checkpoint, backup/restore ve uygulama kapanışı stratejisi test edilir.
- DB bağlantıları context manager veya açık lifecycle ile yönetilir; test teardown'da kapatılır.
- Bir kullanıcıda bozuk metadata bulunursa sessiz yeniden kurma ve veri silme yok; migration backup ve açıklayıcı hata.
- Yalnızca metadata/indeks DB'de; büyük audio/checkpoint/metrics dump dosyada tutulur.
- Windows file-lock koşulları ve Linux path/permission farkları fixture ve integration testlerinde doğrulanır.
- Şema değişikliği release manifest dışa aktarımlarına etki ediyorsa sürüm yükseltme zorunlu.

---

## 15. Yapılandırma ve secret yönetimi

- Git'e `.env`, API key, GitHub token, HuggingFace access token, Google Drive credential veya kişisel dosya yolu commit etme.
- `*.example.yaml` gibi örnek dosyalarda yalnızca etkisiz placeholder kullan; gerçek secret veya kullanıcı hesabı bulunmasın.
- Ortam değişkenlerini yalnızca ihtiyaç anında oku; başlatma loguna değerlerini yazma.
- Redaction birim testinde bilinen secret fixture'ları raw/JSON/log/exception çıktılarında bulunmamalıdır.
- Secret olan bir recipe alanı varsa resolved snapshot'a değer yerine `redacted`/secret reference girer.
- CLI argümanlarında secret vermek mümkünse kaçın; OS process listesinde görünmesini önle.
- GitHub Actions job'larına least privilege `permissions` tanımla; public repo'dan gelen PR'a publishing credentials sunma.
- İndirilen paket ve üçüncü taraf eylemleri sabit sürüm/revizyon ile kontrol et; CI supply-chain riskini değerlendir.
- Gizli ses datası ve model ağırlıklarının ticari/özel lisansını public release'e otomatik taşımaya çalışma.
- Audit log karar ve metadata tutar; ham ses kaydı veya kullanıcı konuşma metni default olarak loglanmaz.

---

## 16. Test matrisi — hangi özellik nasıl doğrulanır?

| Katman | Minimum test | Başarısızlık örneği |
|---|---|---|
| Recipe | Schema validation, unknown fields, immutable snapshot | `max_steps=-1`, bilinmeyen backend |
| Path/I/O | Windows/Linux path, traversal, symlink, atomic files | `../private` export |
| Run state | Her geçiş + yasak geçiş + idempotence | `failed -> completed` izinsiz |
| Store | WAL transaction, recovery, backup/restore | Ortada worker öldürme |
| Executor | Fake process, timeout, cancellation, cleanup | Çocuk process arkada kalıyor |
| Metrics | NaN/inf, unit, duplicate, ordering | Aynı step iki kez sayılıyor |
| Artifacts | Hash, tam/yarım transfer, immutable ref | `.partial` release'e giriyor |
| Corpus contract | Version, license, split, sample rate | Eksik lisans, 16k resample iddiası |
| Speech adapter | Gerçek CLI mapping, codec/checkpoint | `--synthetic --token-cache` birlikte |
| Evaluation | Sabit suite, source attribution, fairness | Farklı codec loss'ları eşdeğer sanılıyor |
| Registry | Candidate→release, approval, rollback | Placeholder hash ile promotion |
| Colab/Drive | Resume/import, part retry, quota | Hash mismatch kabul ediliyor |
| Security | Secret redaction, unsafe inputs | Log içinde HF_TOKEN |
| Packaging | `uv build`, wheel import, no side effect | Import sırasında GPU/model indiriliyor |
| CI | Ruff, pyright, pytest, build ve contract | Başarısız testle merge |

### 16.1 Test türleri ve kullanım sırası

1. **Unit:** Domain, schema, state transition ve hash fonksiyonları; ağ/GPU gerekmez.
2. **Contract:** Speech/Corpus/Live'ın sabit sürümlü fixture'larıyla compat; sahte sonuç üretmeden.
3. **Integration:** SQLite + gerçek subprocess + temp artifact dizini + fake backend.
4. **Smoke:** Gerçek Speech tiny synthetic eğitim (CPU) ve checkpoint inspect; gerektiğinde ayrı test etiketi.
5. **Fault injection:** `SIGTERM`/Windows termination, disk dolu, kesilmiş stream, bozuk dosya.
6. **GPU/Colab:** Runner/ortam/VRAM ve gerçek model kalite testi; PR başına CPU CI yerine kontrollü ayrı iş.
7. **Release:** Candidate doğrulama, manifest ve Live compatibility smoke.

### 16.2 Test etmeme bahanesi olarak maliyet gösterme

GPU testleri pahalıysa CPU'da mock yapıp bunun **GPU doğrulaması olmadığı** raporlanır. Erişilemeyen hosted Colab servisi gerçek Colab test edilmiş gibi yazılmaz. Test verileri küçük, lisanslı veya tamamen sentetik ve PII içermeyen fixture'lardan seçilir. Büyük model ağırlıkları CI'a gereksiz indirilmez.

### 16.3 Temel kalite kapıları

- Değişen her domain davranışına en az bir pozitif, bir negatif test.
- Concurrency, resume, release ve hassas veri işlemlerinde başarı yolu kadar kesinti/hata senaryoları.
- CI'de ruff, type check, unit ve contract testleri, package build; E2E uygun maliyette çalışır.
- Lint'i susturmak yerine hatayı çöz; test atlama listesi, reason/owner/review tarihi olmadan büyümez.
- Mevcut CI kırmızıysa kullanıcının isteğiyle ilgisi açıkça bildirilir; yeni değişiklik sebebiyle mi başladığını ayır.
- Benchmark, performans regresyonuna duyarlı olmalı ama değişken paylaşımlı CI runner'larda kör performans eşiği dayatma; gerçek hardware karşılaştırmasını sabitle.

---

## 17. CI/CD ve değişiklik yönetimi

**İlk CI hedefi:** Ubuntu CPU job'unda temiz checkout + locked dependency install + lint/type/test/build. Windows path/process gereksinimleri için Windows CI matrix gerekebilir; ihtiyaç test kapsamına göre ekle. Release yalnızca ayrı doğrulanmış ve izinli workflow ile yapılır.

### 17.1 GitHub Actions politika kuralları

- `on: push` ve `pull_request` tetikleyicilerinde sürüme kilitlenmiş bağımlılıklar.
- Üçüncü taraf Actions sürüm/commit pin güvenliği, least privilege `permissions`.
- Fork PR'larında write token, yayın credential'ı veya private dataset erişimi yok.
- `pull_request_target` üzerinden doğrulanmamış kod çalıştırma yok.
- Büyük GPU/Colab işleri default PR job'u değildir; kontrollü manual/scheduled pipeline.
- Test başarısı, wheel build başarısı ve security check ayrı, görülebilir job çıktılarıdır.
- Release tag'leri paket sürümüyle uyumlu ve mümkünse immutable artifact metadata'sı ile bağlıdır.
- Docs-only PR olsa bile Markdown link/lint/contract kontrolü yapılır; sahte end-to-end başarı metni basılmaz.
- Public repo'ya büyük token cache, checkpoint, Drive mount yolu veya lisansı belirsiz veri ekleme.

### 17.2 PR oluşturma disiplini

Her PR'ın gövdesinde en az:

```text
Amaç:
Değişen modüller:
Yapılmayan/kapsam dışı konular:
Doğrulama komutları ve gerçek sonuç:
Uyumluluk ve güvenlik etkisi:
Artifact/dataset/release migration ihtiyacı:
Rollback yöntemi:
Açık riskler / takip işleri:
```

Modül bazlı commitler mantıklıdır, fakat gereksiz bölünmüş PR'larla entegrasyonu kırma. **Kullanıcı açıkça istemedikçe** remote push, model release, paket yayınlama, ücretli Colab/GPU başlatma, repo reset ve geri dönülmez silme işlemlerini otomatik gerçekleştirme.

---

## 18. Dokümantasyon, ADR ve kapsam yönetimi

**Forge temel belge kümesi:**

- [`README.md`](README.md) — proje giriş kılavuzu ve mevcut/hedef durum.
- `AGENTS.md` — bu geliştirme ajanı operasyon sözleşmesi.
- [`docs/TRAINING_PIPELINE.md`](docs/TRAINING_PIPELINE.md) — worker, scheduling, resume, Colab/Drive; mevcut tasarım belgesi, uygulama henüz planlanan.
- [`docs/EXPERIMENTS.md`](docs/EXPERIMENTS.md) — run store, metric, comparison; mevcut tasarım belgesi, uygulama henüz planlanan.
- [`docs/EVALUATION.md`](docs/EVALUATION.md) — suite, benchmark ve bilimsel doğrulama; mevcut tasarım belgesi, uygulama henüz planlanan.
- [`docs/MODEL_REGISTRY.md`](docs/MODEL_REGISTRY.md) — promotion, integrity ve release; mevcut tasarım belgesi, uygulama henüz planlanan.
- [`../docs/ENGINEERING_STANDARDS.md`](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ENGINEERING_STANDARDS.md) — tüm repolara ait ortak standartlar.
- [`../docs/INTEGRATION.md`](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/INTEGRATION.md) — sözleşmeler ve bağımlılık yönleri.
- [`../docs/ROADMAP.md`](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ROADMAP.md) — milestone ve diğer projelerle plan.

**Önemli bağlantı uyarısı:** Cross-repo dokümantasyon bağlantıları D8'de commit-sabitli GitHub URL'lerine dönüştürülmüştür. PRIVATE meta ve Speech hedefleri yetkili GitHub erişimi gerektirir; URL metinlerinin görünür olması hedef belge içeriğini anonim kullanıcılara açmaz. Dört Forge teknik Markdown belgesi mevcuttur; Forge CLI, training orchestration ve model registry/release kodu hâlâ planlanmaktadır.

### 18.1 ADR gerektiren kararlar

Aşağıdaki kararlarda küçük, somut bir Architecture Decision Record veya issue karşılaştırması üret:

- SQLite'tan başka veritabanına, local executor'dan dağıtık scheduler'a geçiş.
- Speech public API değişikliği veya yeni model ailesi adapter'ı.
- Recipe/manifest/checkpoint için kırıcı schema sürümü.
- Model promotion yetkileri veya secret yönetimi değişikliği.
- Default olarak otomatik bulut çalıştırma veya ücretli GPU kullanımı.
- Model artifact formatının değiştirilmesi, cross-repo storage API kararı.

ADR gerekçesi: problem, seçenekler, neden seçildi, alternatif riskler, geçiş, test ve rollback. Sırf profesyonel görünmek için her küçük fonksiyona ADR yazma.

### 18.2 Yeni teknoloji değerlendirme kapısı

Bir ajan yeni bir tool/framework öneriyorsa önce şu soruları somut cevapla:

1. **Bugün** hangi problemi yaşıyoruz? Ölçüm/bug raporu var mı?
2. Mevcut Python standard library + SQLite + subprocess bu problemi çözebiliyor mu?
3. Yeni teknoloji bakımı, güvenliği ve CI maliyetini ne kadar artırıyor?
4. Bir çalışan küçük prototip ile faydası ölçülebilir mi?
5. Alternatifin lock-in, veri taşınabilirliği ve geri alma maliyeti nedir?
6. Bu karar Forge'un gerçek F0–F4 kilometre taşlarını ilerletiyor mu, yoksa geciktiriyor mu?

Örnek: MLflow, Weights & Biases, Ray, Celery, Redis, PostgreSQL, Kubernetes, object storage, OpenTelemetry, DVC veya model-serving framework'ü yalnızca popüler olduğu için ekleme. Gerekiyorsa önce **dış sistem adaptörü** olarak ve ölçülen ihtiyaca göre değerlendir. Bu liste yasak teknolojiler listesi değildir; kanıtlanmamış erken karmaşıklığın önüne geçer.

---

## 19. İşleri aşamalara ayırma — F0–F9

| Faz | Codex görevi | Gerçek kabul kanıtı |
|---|---|---|
| **F0** Repo temelini kur | `pyproject`, `src`, `uv.lock`, lint/type/test/build, CI | Temiz kurulumdan wheel import ve CI yeşil |
| **F1** Recipe/domain | Versioned schema, parser, validation, immutable snapshot | Pozitif/negatif schema ve override testleri |
| **F2** Run store | SQLite WAL, transitions, metric events, artifact refs | Crash/restore ve duplicate run testi |
| **F3** Local executor | Subprocess safe args, timeout, cancellation, cleanup | Fake backend + gerçek process kill testi |
| **F4** Speech adapter | Var olan Speech CLI ile synthetic smoke | Gerçek 100-step çalışma, checkpoint doğrulama |
| **F5** Corpus handoff | Manifest schema, license/split/hash validation | Bozuk hash/lisans/split reddi |
| **F6** Evaluation | Suite version, report, apples-to-apples check | Aynı koşullu karşılaştırma, incompatible reject |
| **F7** Registry | Candidate/released, approval, release manifest | Promotion/rollback ve negative test |
| **F8** Colab/Drive | Verified import/sync, resume ve partial handling | Kesinti sonrası doğrulanmış artifact |
| **F9** Stabilization | Docs, CI/release, graduation evidence | Tekrarlanabilir pilot, açık rapor ve regression |

**Minimum kullanılabilir Forge:** F0–F4. İlk gerçek ürün başarısı “Forge üzerinden doğrulanmış bir Speech smoke run başlatıldı, run metadata/metrics/checkpoint kaydedildi ve aynı sonuç protokolü tekrar oynatılabiliyor” olmalıdır. F5–F8 yokken `production-grade full ML platform` gibi iddiadan kaçın.

### 19.1 Paralel geliştirme yaklaşımı

Corpus pilot dataset hazırlarken Forge manifest fixture'ı ve validation modülünü geliştirebilir. Speech codec eğitilirken Forge mevcut Mimi baseline ile iş yürütebilir. Live fake ses runtime ile arayüzü geliştirirken Forge registry sözleşmesini tamamlayabilir. **Forge, diğerlerinin bitmesini beklemez; fakat onların sahibine ait algoritmayı kendi içine kopyalamaz.**

---

## 20. Kodlama ajanının her görev sonundaki zorunlu raporu

Codex işi tamamladığında şu kısa ama kanıtlı formatı kullan:

```text
Görev:
Kapsam ve değiştirdiğim dosyalar:
Çalışan mevcut özellik / yeni planlanan özellik ayrımı:
Çalıştırdığım komutlar (gerçek çıktıyla):
  - lint: geçti / başarısız / çalıştırılamadı (neden)
  - typecheck: geçti / başarısız / çalıştırılamadı (neden)
  - unit/contract/integration: geçti / başarısız / çalıştırılamadı (neden)
  - build/CI: geçti / başarısız / çalıştırılamadı (neden)
İlgili run ID / artifact hash / checkpoint (varsa):
Kaynak güvenliği, dataset/codec uyumluluğu ve performans etkisi:
Kapsam dışında bıraktıklarım:
Riskler ve geri alma yolu:
Sonraki en küçük doğrulanabilir adım:
```

**Raporlama davranışı:** GitHub'a push edilmediyse “pushlandı” yazma. Colab çalıştırılmadıysa “Colab doğrulandı” deme. Mock başarıysa gerçek model başarısı diye gösterme. Kullanıcı yalnızca Markdown istediğinde package/test değişikliği yapma. Eksik kalanları açık, somut ve kısa biçimde bildir.

### 20.1 Definition of Done — tek görev

Bir değişiklik ancak şu koşullarla tamam sayılır:

- [ ] Kullanıcı isteği karşılandı ve değişen repo sınırı doğru.
- [ ] Mevcut davranış ve hedef davranış doğrulanabilir biçimde ayrıldı.
- [ ] Yeni kod test edildi; başarısız veya çalıştırılamayan testler raporlandı.
- [ ] Kritik negatif/hata senaryosu için en az bir test var.
- [ ] Secret veya kullanıcı verisi log/artifact/repo'ya sızmadı.
- [ ] API, recipe, artifact ve release sözleşmeleri kırılmadı ya da geçiş belgelendi.
- [ ] Dosya/DB/worker kaynakları doğru kapatılıyor; retry/cancel güvenli.
- [ ] Gerekli belge ve example güncellendi.
- [ ] Değişiklik için geri dönüş yolu açıklanabilir.
- [ ] Gerçek çıktı/kanıt bağlantısı veya komut raporu mevcut.

### 20.2 Definition of Done — Forge V1

- [ ] Bir geliştirici temiz checkout'ta Forge paketini kurup testleri çalıştırabilir.
- [ ] Geçersiz recipe, bozuk dataset hash veya model/codec uyuşmazlığı eğitimi başlatamaz.
- [ ] Tek bir Speech smoke run, **Forge üzerinden**, gerçek backend ile başlatılıp izlenir.
- [ ] Run state, config digest, environment ve source SHA kaydedilir.
- [ ] Checkpoint/metrics/artifacts gerçekten kaydedilir ve checksum doğrulanır.
- [ ] İptal/crash/yeniden başlatma iş durumunu yanlış başarıya çevirmeden yönetilir.
- [ ] En az bir evaluation suite ve model candidate kaydı üretilebilir.
- [ ] Release/promotion politikası ve testleri uygulanmışsa üretimde hazır olarak ayrıca doğrulanır.
- [ ] CPU CI, temel integration ve security kontrolleri geçti.
- [ ] Eksik kalan Colab/Drive/GPU özellikleri `planned`/`blocked` olarak görünür.

---

## 21. Kesinlikle kaçınılacak anti-pattern'ler

1. **Model çoğaltma:** `ForgeTransformer`, `ForgeCodec`, ikinci loss/trainer kopyası.
2. **Sahte test:** Başarısız run'ı veya olmayan GPU deneyini başarı olarak raporlamak.
3. **Fazla mimari:** İlk smoke eğitiminden önce mikroservis, Kubernetes, çok kullanıcılı SaaS, dashboard inşa etmek.
4. **Sessiz sürüm kayması:** Codec token formatı değiştiği halde eski checkpoint'i uyumlu varsaymak.
5. **Güvensiz subprocess:** Recipe string'ini shell komutuna eklemek veya arbitrary plugin çalıştırmak.
6. **Eksik dosya kabulü:** `.partial` checkpoint, kesik Drive upload veya bozuk SHA ile `completed` işaretlemek.
7. **Başlıksız veri:** Kaynak/lisans/split/provenance olmadan ses datasetini eğitime sokmak.
8. **Yalancı metrik:** Farklı frame rate veya tokenizer üzerinde loss değerlerini aynı ölçü gibi sunmak.
9. **Yetkisiz işlem:** Açık onay olmadan ücretli uzak iş, public GitHub release, model promotion, veri temizliği.
10. **Repo adı yanılsaması:** `kaiwen-speech` GitHub adından `kaiwen_speech` Python importunu varsaymak.
11. **Güvensiz dosya yükleme:** Rastgele uzak `.pt` checkpoint'i değerlendirmeden deserialize etmek.
12. **Yanlış kalite iddiası:** Tiny smoke overfit'i doğal Türkçe konuşma başarısı veya 2 GB inference kanıtı saymak.
13. **Tek seferlik patch:** Test/CI ve migration olmadan yalnızca happy path'i düzeltmek.
14. **Aşırı bağımlılık:** Bir satırlık iş için büyük framework kurmak; platformu tek cloud'a kilitlemek.
15. **Kök belge kaybı:** `kaiwen-ai/` kendi Git repo'su değilken root dosyaları yanlışlıkla Forge'a taşımak/silmek.

---

## 22. Son talimat: doğrulanabilir ilerleme, bitmeyen tasarım değil

Kaiwen Forge'un ilk başarısı geniş vizyon metni, göz alıcı dashboard veya onlarca adapter değildir. **Basit bir CPU smoke eğitiminin deney tarifiyle güvenli başlatılması, takip edilmesi, doğrulanmış checkpoint olarak kaydedilmesi ve hata durumlarından dürüstçe kurtarılmasıdır.**

Kodlama ajanı yeni bir teknoloji, servis veya framework keşfettiğinde onu otomatik görev listesine taşımaz. Önce bugün çözülecek problemi, ölçülecek faydayı, maliyeti ve geri alma yolunu açıklar. Ancak gerçekten gerekeni inşa eder. Eğitim sonuçlarını bilimsel kanıt olarak kaydeder; **bilinmeyeni gizlemez, kanıt olmadan başarı ilan etmez.**

**Tek cümlelik karar filtresi:** *Bu değişiklik Kaiwen Forge'u model algoritmasını kopyalamadan daha güvenilir, daha tekrarlanabilir ve daha kolay doğrulanabilir bir araştırma yürütme sistemine dönüştürüyor mu?* Yanıt hayırsa işi küçült, yeniden konumlandır veya kapsam dışına çıkar.
