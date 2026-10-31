# Kaiwen Forge — Model Registry, Artifact Güvenliği ve Yayın Yaşam Döngüsü Şartnamesi

> **Belge türü:** Normatif mühendislik tasarımı, veri sözleşmesi ve uygulanabilir geliştirme planı  
> **Hedef dosya:** `kaiwen-ai/kaiwen-forge/docs/MODEL_REGISTRY.md`  
> **Durum:** `TARGET_V1` — tasarım belgesi; çalışan Forge registry henüz uygulanmış değildir.  
> **Sahip:** Kaiwen Forge. Model/codec yükleme ve checkpoint algoritması Kaiwen Speech'in; son kullanıcıya model seçme ve oturum yönetimi Kaiwen Live'ın sorumluluğundadır.  
> **Tarih:** 2026-10-30. Bu tarih, belgede açıklanan repo incelemesinin bağlamıdır; gelecekteki implementasyonun durumu ayrıca doğrulanmalıdır.  
> **Bağlantılar:** [Forge README](../README.md) · [Forge AGENTS](../AGENTS.md) · [Training Pipeline](TRAINING_PIPELINE.md) · [Experiments](EXPERIMENTS.md) · [Evaluation](EVALUATION.md) · [Speech API](../../kaiwen-speech/docs/API.md) · [Speech Architecture](../../kaiwen-speech/docs/ARCHITECTURE.md) · [Speech Codec](../../kaiwen-speech/docs/CODEC.md) · [Speech Performance](../../kaiwen-speech/docs/PERFORMANCE.md) · [Workspace Integration](../../docs/INTEGRATION.md) · [Engineering Standards](../../docs/ENGINEERING_STANDARDS.md) · [Roadmap](../../docs/ROADMAP.md)

---

## 0. Okuma kılavuzu, kapsam ve doğruluk bildirimi

Bu belge **bir model dosyasını bir klasöre kopyalayıp adını sürüm olarak yazmaktan çok daha fazlasını** tasarlar. Bir eğitim sonucunun hangi codec'le, hangi token sözleşmesiyle, hangi kod ve veriyle üretildiğini; hangi değerlendirmelerden geçtiğini; kim tarafından, hangi amaçla yayınlanmasına izin verildiğini; ve Kaiwen Live tarafından güvenle nasıl yüklenebileceğini tarif eder. Registry, araştırmada izlenebilirlik sağlayan kayıt defteri ve uygulamada güvenilir sürüm seçme aracıdır. Model eğitmez, ses üretmez ve kullanıcının mikrofonunu yönetmez.

**Güncel gerçek durum:** 30 Ekim 2026 incelemesinde `ImtheKaiwen/kaiwen-forge` GitHub reposu yalnızca kısa bir `README.md` içeriyordu. Registry veritabanı, Python paketi, CLI, yayın mekanizması ve doğrulayıcı henüz yoktur. Bu dokümandaki `forge ...` komutları, `kaiwen_forge` sınıfları, API örnekleri ve klasör ağaçları **HEDEF**tir. Bu belgede kod örneğinin bulunması, çalışan ürün kanıtı değildir.

**Mevcut referans davranış:** `kaiwen-speech` içindeki `src/kaiwen_voice/training/checkpoint.py`, `format_version=1` kullanan PyTorch checkpoint yazma/yükleme; `src/kaiwen_voice/codec/base.py`, codec metadata; `src/kaiwen_voice/codec/mimi_codec.py`, frozen Mimi adaptörü sağlar. Mevcut `scripts/infer_audio.py` turn-based çıkarımda checkpoint ve Mimi codec kullanır. Bunlar kullanılabilir bir araştırma temeli sunar, fakat tek başına yayınlanmış bağımsız model paketi veya model registry değildir. Kodun gerçekten çalıştığı koşullar CI veya yerel test raporuyla ayrıca doğrulanmalıdır.

Bu belge aşağıdaki statüleri kullanır:

| Statü | Anlam | Nasıl raporlanır? |
|---|---|---|
| `IMPLEMENTED` | İncelenen kaynakta mevcut | Dosya ve test referansı ile |
| `TARGET_V1` | İlk çalışır sürüme ait normatif gereksinim | Görev ve kabul testi ile |
| `EXPERIMENTAL` | Henüz kanıtlanmamış araştırma/ürün seçeneği | Hipotez ve benchmark ile |
| `DEFERRED` | V1 kapsamı dışında | Bağımlılık ve gerekçe ile |
| `BLOCKED` | Eksik izin, uyum veya kanıt nedeniyle durdurulmuş | Hata kodu, insan eylemi ile |

**Normatif kelimeler:** “ZORUNLU” güvenli ve geçerli V1 için şart; “ÖNERİLEN” ölçüm/uygulama sonucuyla değişebilir; “YASAK” güvenlik veya araştırma bütünlüğünü ihlal eder. Boş alanları başarılı sonuçla doldurmak, geçersiz hash'i geçerli göstermeye çalışmak veya uygulanmamış özelliğe `ready` etiketi vermek YASAKTIR.

### 0.1 Tek bir belge, tek bir sahiplik sınırı

Burada release manifestinin **Forge kayıt/dağıtım yönünü** belirleriz. Speech'in model checkpoint formatının semantiği Speech'te kalır. Live, manifest tüketimi ve local session sınırının sahibidir. `EXPERIMENTS.md` metrik tanımlarının ve bilimsel karşılaştırılabilirliğin otoritesidir; `EVALUATION.md` ise değerlendirme protokol ve rapor kanıtının. Registry bunları kopyalayıp ikinci tanım kaynağı üretmez; sürümlü referansları, hash'leri ve kararları saklar.

### 0.2 Belgenin dört temel sorusu

1. Bu artifact **gerçekten hangi byte'lardan** oluşuyor ve değişmiş mi?
2. Bu model, bu codec ve bu runtime **teknik olarak uyumlu** mu?
3. Modelin amaçlanan kullanımını destekleyen **gerçek kanıt ve gerekli hak/izinler** var mı?
4. Yeni sürüm yüklenirken bir hata olursa **son çalışan sürüme güvenle** nasıl döneriz?

Bu dört soruya ayrı ayrı yanıt veremeyen registry, yalnızca dosya listesi sayılır.

---

## 1. Temel amaç, ürün sınırı ve başarı tanımı

Kaiwen Forge Model Registry, bir `run` tarafından üretilen checkpoint'i doğrudan Live'a sunmaz. Eğitim çıktıları önce doğrulanır, gerekirse inference için paketlenir, uyumluluk ve değerlendirme kapılarından geçirilir, ardından sürümlü bir **release** oluşturulur. Registry'nin temel garantisi bir `release_id`'nin değişmez bir manifest ve değişmez artifact setine işaret etmesidir. `latest`, `stable` veya `recommended` gibi ifadeler ise **değişebilir yönlendirme kayıtlarıdır**; release kimliği değildir.

**V1 başarı tanımı:** En az bir Kaiwen Speech baseline checkpoint'i ve onunla uyumlu **gerçek** Mimi codec referansı, doğrulanmış model manifesti ve araştırma sınırlamalarıyla yerel registry'ye alınır. Bağımsız bir `inspect` komutu dosya bütünlüğünü ve metadata'yı okuyabilir. `promote` yalnız tüm gerekli kapılar geçildikten ve insan onayı kaydedildikten sonra çalışır. Live için üretilen paket (uygun runtime geliştirildiğinde) kötü hash'i reddeder, codec uyuşmazlığını reddeder ve sağlam önceki sürümden çalışmaya devam edebilir. Bu sırada yeni Kaiwen Codec'i veya native streaming'i *mevcutmuş gibi* göstermeyiz.

### 1.1 Birinci sürümde olmayacaklar

- Herhangi bir bulut sağlayıcısına bağlanmayı zorunlu kılan SaaS model hub'ı.
- Kullanıcı hesabı, ödeme, geniş rol yönetimi ve çok kiracılı (multi-tenant) altyapı.
- Sadece Forge için ayrı GPU eğitim motoru veya Speech'in checkpoint formatının yeniden yazılması.
- Model dosyasını yüklerken otomatik kod çalıştırma ya da manifestten rastgele komut yürütme.
- “En iyi model”i tek bir loss veya en yeni timestamp'e göre otomatik yayınlama.
- Açık oturumda inference ağırlıklarını sessiz değiştirme, otomatik arka plan indirmesi.
- Kanıtlanmamış 2 GB VRAM, sıfır gecikme veya tam Türkçe doğal konuşma vaadi.

Forge'un ilk çalışır registry'si **yerel SQLite + dosya sistemi + explicit CLI** olabilir. Uzak depolama ve diğer model aileleri gerçek ihtiyaç ortaya çıktıkça eklenir.

---

## 2. Ekosistem bileşenlerinin açık sorumlulukları

```text
Kaiwen Corpus -- sürümlü veri ve lisans kanıtları -----------┐
                                                           │
Kaiwen Forge -- recipe / run / attempt / evaluation --------┼──> Registry
       |                                                   │       |
       +-- Speech adapter --> Kaiwen Speech checkpoint -----┘       |
                                                                   v
                                                 doğrulanmış release paketi
                                                                   |
                                                                   v
                                                        Kaiwen Live loader
                                                                   |
                                                                   v
                                                      Kaiwen Speech runtime
```

| Bileşen | Ürettiği bilgi | Registry ilişkisi |
|---|---|---|
| **Corpus** | Dataset manifesti, lisans, içerik hash'leri, split, consent bilgisi | Değişmez veri/provenance referansı |
| **Speech** | Codec ve model mimarisi, checkpoint formatı, inference loader, capabilities | Model kimliğini teknik olarak doğrular |
| **Forge Training** | Run, attempt, job, artifact ve environment kaydı | Candidate kaynağı |
| **Forge Experiments** | Hipotez, metrik tanımı, karşılaştırılabilirlik | Model seçiminin bilimsel gerekçesi |
| **Forge Evaluation** | Test protokolü, rapor, örnek başarı/başarısızlık, ölçüm ortamı | Promotion için kanıt |
| **Forge Registry** | Candidate, release, channel, onay, digest, geri alma | Katalog ve yayın otoritesi |
| **Live** | Model seçimi, yerel yükleme, sesli oturum ve kullanıcı hatası | Sadece doğrulanmış release tüketir |
| **Design/Agent** | UI bileşenleri / opsiyonel araçlar | Registry çekirdeğine import edilmez |

Forge'un Speech'e ihtiyacı **model adapter sınırında** olabilir; Forge domain nesneleri yalnız `model_family`, `artifact_ref`, `compatibility_report` gibi tipli alanları bilir. Speech'in Forge'u import etmesi yasaktır. Model runtime'ın yükleyemediği bir release yalnız manifest testini geçerek “hazır” sayılamaz. Live'ın çalışma zamanı beklentileri, Speech API sözleşmesiyle contract testlerde doğrulanır.

---

## 3. Domain modeli ve karıştırılmaması gereken kimlikler

Birincil ilişkiler aşağıdaki kavramlardan oluşur:

| Nesne | Kimlik | Değişebilir mi? | Görev |
|---|---|---|---|
| `study` / `experiment` | `experiment_id` | Yeni revision ile | Hipotez ve deney grubu |
| `run` | `run_id` | Durum geçişi olur | Sabit resolved recipe ve eğitim hedefi |
| `attempt` | `attempt_id` | Durum geçişi olur | Tek proses/Colab yürütümü |
| `artifact` | `artifact_id` ve SHA-256 | **Byte içeriği değişmez** | Checkpoint, config, codec, rapor |
| `candidate` | `candidate_id` | İnceleme durumu değişebilir | Bir artifact grubunun aday gösterilmesi |
| `evaluation` | `evaluation_id` | Sonuç artifaktı değişmez | Belirli model ve protokolle ölçüm |
| `release` | `release_id` | **Manifest ve artifact seti değişmez** | Yayınlanan belirli paket |
| `channel pointer` | `family/channel/platform` | Kontrollü değişebilir | `stable` gibi aktif sürüme yönlendirme |
| `installation` | `install_id` | Yerel kurulum state'i değişir | Live'ın doğrulanmış model kopyası |

### 3.1 Model ID, release ID, artifact ID ve version farkı

`model_id` bir model ailesi/ürün hattını (`kaiwen-speech-tiny`) tanımlar. `model_version` insanın anlamlandırdığı sürümdür (`0.1.0` gibi). **Gerçek paket eşsizliği** `release_id` ve manifest digest'iyle sağlanır. `artifact_id` ise tek dosyaya veya içerik hash'ine aittir. Aynı `model_version` için farklı ağırlıkların sessiz değiştirilmesi yasaktır. Yeni derleme, quantization veya farklı codec bileşimi farklı bir release kimliği ve açık variant kaydı gerektirir.

`run_id` bir model sürümü değildir: eğitim tekrarlandığında farklı run'lar aynı model ürün ailesinde aday olabilir. Aynı checkpoint dosyası farklı evaluation protokollerine girebilir; sonuçlar `evaluation_id` ile ayrılır. `attempt_id` resume olduğunda değişir ancak üst `run_id` sabit kalır. Bu ayrım, tek bir eğitim denemesinin rastgele “release v1” olarak görünmesini önler.

### 3.2 Kimlik üretimi ve idempotence

V1 için ULID/UUIDv7 gibi sıralanabilir benzersiz ID kullanılabilir; eşsizliği yalnız tarih/saat veya dosya adından türetmeyiz. İstemci `request_id`/idempotency key gönderdiğinde aynı promotion/registration isteği aynı sonucu dönmeli; iki kez aynı artifact'leri yayınlamamalıdır. İstek aynı anahtarla farklı payload getirirse açık `IDEMPOTENCY_CONFLICT` hatası oluşur. Hash collision varsayımı pratikte düşük olsa da kayıt `sha256` + dosya boyutu + türü saklar.

---

## 4. Yaşam döngüsü: checkpoint → candidate → release → kurulum

```text
TRAINING RUN
   └─ checkpoint produced
       └─ artifact registered (raw, quarantined when needed)
           └─ integrity VERIFIED
               └─ candidate CREATED
                   ├─ compatibility BLOCKED/FAILED ─> düzelt / yeni candidate
                   ├─ evaluation PENDING/FAILED ──> yeni kanıt / yeniden eğitim
                   ├─ rights PENDING/DENIED ──────> araştırma içi kalır
                   └─ all required gates PASS
                       └─ human APPROVED
                           └─ release SEALED (immutable)
                               └─ channel pointer updated (optional)
                                   └─ Live staged install → verified → activated
```

**Checkpoint state'i**, **release durumu**, **channel durumu** ve **Live kurulum durumu** farklı state machine'lerdir. Bir checkpoint başarıyla kaydedildi diye release `approved` olmaz. Bir release kaydedildi diye `stable` kanalı otomatik güncellenmez. Kanal güncellendi diye Live'ın var olan oturumu yeni modele geçmez.

### 4.1 Candidate durumları

| Durum | Geçiş koşulu |
|---|---|
| `draft` | Artifact referansları toplanıyor, henüz kanıt eksik |
| `verification_pending` | Hash, şema, codec/model uyumu kontrol edilecek |
| `verification_failed` | Somut doğrulama hatası; eski sonuç saklanır |
| `review_ready` | Zorunlu otomatik kapılar geçti; onay bekliyor |
| `approved_for_experimental` | Deneysel kullanım için insan onayı ve sınırlar kaydedildi |
| `rejected` | Red nedeni, aktör, zaman ve kanıt referansı saklandı |

Bir `candidate` yayımlanmış immutable release değildir. Kaynak artifact referansları değişecekse aynı candidate üzerinde sessiz değiştirmek yerine yeni candidate revision/kimliği açılır. Eksik kanıt tespit edildiğinde review-ready durumu geri alınabilir ama geçmiş geçiş eventleri silinmez.

### 4.2 Release durumları ve değişmezlik

Release oluşturulduğunda manifest ve artifact digest'leri sealed olur. Release'in sonradan işletimsel görünürlüğü `available`, `deprecated`, `quarantined`, `revoked` gibi **ayrı status kaydıyla** değişebilir; içerik asla overwrite edilmez. `revoked` bile tarihi ve araştırma amaçlı audit'i silmez. Genel kullanıcıya sunulan kanalda bu sürüm seçilmez; dağıtımı ve yeniden yüklenmesi ilgili policy'ye göre engellenir.

`deprecated` eski fakat geçerli bir sürümü ifade eder; `revoked` ciddi güvenlik/lisans/bütünlük riski için engellemeyi ifade eder. `quarantined` belirsizlik çözülene kadar kullanımın durdurulmasıdır. Bunları aynı `failed` değeri altında toplamak YASAKTIR.

---

## 5. Artifact sınıflandırması ve fiziki paket sınırı

| Artifact türü | İçerik | Ürün paketi için davranış |
|---|---|---|
| `training_checkpoint` | Model + optimizer/scheduler/scaler/step/repro state | **Doğrudan kullanıcıya dağıtılmaz** |
| `inference_weights` | Yalnız gerekli model ağırlıkları | Zorunlu bileşen |
| `model_config` | Mimarinin kesin konfigürasyonu, token/heads vb. | Zorunlu bileşen |
| `codec_weights` | Kaiwen Codec veya güvenilen dış codec ağırlıkları | Modelle eşleşen zorunlu bileşen |
| `codec_config` | Sample rate, Q/K, frame rate, tokenizer identity | Zorunlu bileşen |
| `runtime_contract` | Minimum API, backend, precision, capabilities | Zorunlu metadata |
| `evaluation_report` | Protokol, sonuç, başarısızlık ve ortam | Promotion kanıtı |
| `model_card` | Kullanım, sınırlar, lisans ve bilinen hatalar | Release için zorunlu |
| `license_evidence` | Kaynak veri ve ağırlık kullanım hakları | Dağıtım için zorunlu |
| `example_audio` | İzinli dinleme örnekleri | Opsiyonel, varsayılan özel |
| `debug_logs` | Worker çıktıları | Release paketine dahil edilmez |

**Önemli:** Eski Mimi gibi dışarıdan yüklenen codec için yalnız `codec_id=kyutai/mimi` yazmak yeterli değildir. Tam model/revision/artifact hash'i, kullanım koşulları ve dağıtım stratejisi açıkça çözülmelidir. Codec'i paketlemeden network üzerinden indirmeyi seçmek, varsayılan local-first/offline davranışıyla çelişebilir ve ayrı izin/kurulum aşaması ister. Lisans gereği yeniden dağıtılamıyorsa release manifesti bunu belirtmeli; Live gerekli offline codec olmadan `READY` ilan etmemelidir.

### 5.1 İki ayrı paket: eğitim kurtarma ve inference release

Trainer checkpoint'i optimizer state'leri gibi büyük ve eğitim gizliliği açısından gereksiz bilgiler içerebilir. Inference release, Speech'in onayladığı `export_inference` dönüştürme sınırından geçmelidir. Model durumunun güvenle dönüştürülmesi, `state_dict` anahtar eşleşmesi, precision ve quantization doğrulaması, yükleyici sürüm kontrolü ve çıktı eşdeğerliği testleri içerir. Forge dönüşümü **koordine eder**; modele özel tensor dönüşümünü Speech'e ait export adapter uygular.

Mevcut Speech dosyaları `.pt` üzerinden çalışır. `safetensors` destekli bir inference export bugün yazılmış değildir. Dolayısıyla `.safetensors` kullanımı **hedef format tercihi**, gerçekleşmiş yetenek değildir. `torch.load(weights_only=True)` daha güvenli bir seçenek olabilir ancak bilinmeyen kaynaklar için tam izolasyon sağlamaz.

---

## 6. Tek kanonik release şeması: dokümanda hizalandı, uygulamada ADR gerekli

**Tarihsel çelişki ve son belge denetimi:** İlk taslaklarda Workspace [`INTEGRATION.md`](../../docs/INTEGRATION.md) `kaiwen.model.release`/`weights`, Speech [`API.md`](../../kaiwen-speech/docs/API.md) `kaiwen.speech.release`/`model.checkpoint_path` kullanıyordu. **31 belge sonrası denetimde bu iki örnek kanonik düzeyde tekleştirildi:** artık her iki belge `kaiwen.model.release` ve Forge sahipliğindeki `model.weights` alanlarına referans verir. **Kodda henüz doğrulanmış şema/fixture/loader olmadığı için bu değişiklik implementasyon tamamlandı anlamına gelmez.** Eski isimlerin sessiz alias kabulü yasaktır.

**Bu belgenin önerisi:** Çok aileli geleceğe uygun dış paket için **`kaiwen.model.release` v1** kanonik Forge envelope olsun; Speech'in algoritmik config ve checkpoint semantiği bu envelope içindeki `model`/`codec` alt nesnelerine taşınsın. Model ailesine özel alanlar sürümlü bir `family_contract` nesnesinde bulunsun. Bu, dokümantasyon seviyesinde hizalanmış **teknik hedef karar taslağıdır**; kod yazılmadan önce MR0 ADR, makinece doğrulanabilir schema ve Speech/Live consumer golden-fixture testleri zorunludur.

**Migration gate MR-00:** Şema seçimi, JSON Schema/Pydantic modeli, canonical serialization ve fixture seti ortak contract PR'ı olarak onaylanmadan gerçek release yazma fonksiyonu uygulanmayacak. Eski örnekler `legacy_example` olarak belgelenecek veya yeni şemaya taşınacak; “iki formatı da şimdilik kabul edelim” gibi sessiz fallback yapılmayacak.

### 6.1 Hedef kanonik manifest taslağı — JSON, gerçek yayın değildir

Aşağıdaki JSON sözdizimsel örnektir. `0` tekrarlı SHA-256 dizileri ve `example` kimlikleri **yalnız fixture'larda** kullanılır; üretim ortamındaki integrity veya provenance doğrulamasının kanıtı değildir. `channel=experimental`, değerlendirme kanıtı henüz yokken `release_approved=true` iddiası üretmez.

```json
{
  "schema_name": "kaiwen.model.release",
  "schema_version": "1.0.0",
  "release_id": "example-kaiwen-speech-tiny-0.1.0-exp1",
  "model_id": "kaiwen-speech-tiny",
  "model_version": "0.1.0-exp.1",
  "model_family": "kaiwen-speech",
  "release_channel": "experimental",
  "manifest_state": "example_only",
  "model": {
    "architecture_id": "kaiwen-temporal-parallel-v1",
    "weights": {
      "path": "model.pt",
      "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
      "size_bytes": 0,
      "format": "pytorch-state-dict"
    },
    "config": {
      "path": "model.yaml",
      "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
      "size_bytes": 0
    }
  },
  "codec": {
    "codec_id": "kaiwen-codec-rvq-v1",
    "codec_release_id": "example-codec-0001",
    "weights": {
      "path": "codec.pt",
      "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
      "size_bytes": 0
    },
    "sample_rate_hz": 24000,
    "frame_rate_hz": 50.0,
    "num_codebooks": 4,
    "codebook_size": 1024,
    "token_layout": "B,T,Q"
  },
  "token_protocol": {
    "protocol_version": "1.0.0",
    "special_tokens": ["bos", "user", "assistant", "eos", "pad"],
    "acoustic_delay_frames": 0
  },
  "runtime": {
    "api_min_version": "0.2.0",
    "required_capabilities": ["batch_turn"],
    "supported_precisions": ["fp32", "fp16"],
    "tested_platforms": []
  },
  "provenance": {
    "training_run_id": "example-run-001",
    "training_attempt_id": "example-attempt-001",
    "source_git_sha": "EXAMPLE_NOT_A_REAL_COMMIT",
    "dataset_manifest_sha256": "0000000000000000000000000000000000000000000000000000000000000000"
  },
  "evidence": {
    "evaluation_ids": [],
    "model_card_path": "MODEL_CARD.md",
    "license_review_status": "pending",
    "approval_record_id": null
  }
}
```

Bu örnekte `manifest_state=example_only`, `size_bytes=0`, sentetik hash'ler, boş evaluation listesi ve `license_review_status=pending` bulunuyor: **gerçek registry validatörü bu paketi yayına kesinlikle reddetmelidir**. Bu nedenle yalnızca şema alanlarını göstermek için kullanılabilir. Gerçek release'e geçmeden önce digest, boyut, haklar ve değerlendirme alanları gerçeğe göre üretilmelidir.

### 6.2 Şema kısıtları ve geriye uyumluluk

- `schema_name` birebir eşleşmeli; yakın yazım, bilinmeyen isim veya farklı aile şeması varsayılan reddedilir.
- `schema_version`: SemVer tarzı sürüm; major sürümü desteklenmiyorsa yükleme engellenir; minor'da bilinmeyen **güvenlik-kritik** alanlar sessizce yok sayılmaz.
- `release_id` tekil ve değişmezdir. Aynı ID farklı manifest içeriğiyle gelirse `RELEASE_ID_COLLISION`.
- `model_id`, `model_version`, `model_family`: boş olamaz; izinli karakter kümesi ve uzunluk limiti uygulanır.
- `weights.path`, `config.path`: göreli, normalize, paketin kökü altında ve symlink politikasıyla uyumlu olmak zorundadır; `../`, mutlak URL/drive path ve Windows drive kaçışı yasaktır.
- `sha256`: canonical olarak **64 küçük hexadecimal karakter**, `size_bytes>0`. `sha256:` prefix kullanılacaksa bütün sözleşme aynı biçimi seçer; V1 örneğinde prefixsiz 64 hex kullanılmaktadır.
- `codec`: Q, K, frame rate ve sample rate teknik metadata'dır; codec **artifact ve token mapping kimliğinin yerine geçmez**.
- `runtime.required_capabilities`: gerçek yüklenen backend'in ilan ettiği desteklerle karşılaştırılır. Streaming desteklenmiyorsa `streaming` etiketi seçilemez.
- `provenance`: dondurulmuş run/commit/dataset referanslarıdır; boş ve sentetik kimlikler gerçek release'te reddedilir.
- `evidence`: onay ve lisans durumu veriyle doğrulanır; kullanıcı girdisiyle `approved` yazmak tek başına yetmez.

**Kendi kendini hash'leme problemi:** `manifest.json` dosyasının SHA-256'sı aynı JSON'un içine yerleştirilmez; dış `release.lock` veya registry `manifest_digest` sütununda tutulur. Canonicalization kuralları (UTF-8, normalize sayı/anahtar sırası, tekrar eden JSON key reddi, NaN yasak) tasarlanır; hash'in üretildiği byte'lar kesinleşmeden içeriğin eşdeğer olduğu iddia edilmez.

### 6.3 Şema değişiklikleri

- **Patch:** Açıklama/uygulama düzeltmesi, semantik değişmezse.
- **Minor:** Backward-compatible optional alan, açık varsayılan ve contract test.
- **Major:** Yeni zorunlu alan, farklı doğrulama/loader davranışı ya da codec/token protocol semantiği.
- Geçişler `schema_migrations/` altında testli dönüştürücüyle yapılır; `unknown` alanları silerek sahte başarı yaratmak yasaktır.
- Eski sürüm `read_only` inceleme olarak tutulabilir; yeni release oluştururken kanonik hedef şemaya explicit migrate gerekir.

---

## 7. Codec–Transformer uyumluluk matrisi

Bir audio token tensorunun şekli `[B,T,Q]` olması veya iki codec'in `Q=4`, `K=1024` kullanması, sözlük ID'lerinin aynı akustik/semantik anlamı taşıdığı anlamına gelmez. **Mimi ve özel Kaiwen Codec aynı şekli kullansa bile doğrudan token cache/checkpoint değiştirme işlemi yasaktır.**

Doğrulamada en az şu alanlar karşılaştırılır:

| Alan | Kaynak | Uyuşmazlık sonucu |
|---|---|---|
| Codec release/artifact hash'i | Registry + Speech Codec metadata | `CODEC_IDENTITY_MISMATCH` |
| Modelin eğitildiği codec kimliği | Training run snapshot | `TRAINED_CODEC_MISMATCH` |
| Sample rate / kanal ve preprocessor ID | Codec manifest | `AUDIO_FORMAT_INCOMPATIBLE` |
| Frame rate ve hop size | Codec profile | `FRAME_ALIGNMENT_MISMATCH` |
| Q, K ve token layout | Model config + codec | `TOKEN_LAYOUT_MISMATCH` |
| Special token ID mapping | Speech config | `CONTROL_TOKEN_MISMATCH` |
| Depth/parallel mode | Model config / state_dict | `ARCHITECTURE_MISMATCH` |
| Acoustic delay + flush policy | Model config/token protocol | `ACOUSTIC_DELAY_MISMATCH` |
| Codec stateful streaming capability | Speech runtime | `CAPABILITY_UNAVAILABLE` |
| Model weight format + keys | Speech loader | `MODEL_FORMAT_UNSUPPORTED` |
| Precision/quantization backend | Export metadata + runtime | `PRECISION_UNSUPPORTED` |

### 7.1 Eski Mimi baseline ile yeni Kaiwen Codec ilişkisi

- Mevcut Mimi adapter tipik olarak 24 kHz, 12,5 Hz, 8×2048 token profiline sahiptir; kesin değer adapter'ın gerçek metadata'sından alınır.
- Yeni Kaiwen Codec başlangıç araştırma profili 24 kHz, yaklaşık 50 Hz, 4×1024 olabilir. Bu yalnız aday profildir.
- Mimi tokenlarıyla eğitilmiş Transformer'ın Kaiwen Codec tokenlarıyla doğrudan çalışabileceği varsayılamaz; **model yeniden eğitimi, adaptasyon veya açıkça test edilmiş mapping** gerektirir.
- Farklı `codec_id`/artifact revision aynı `model_id` altında sessiz hot-swap yapılmaz; yeni release oluşturulur.
- Speech'in mevcut `CodecMetadata` veri sınıfında sample rate, frame rate, codebook count/size bulunur; **tam artifact identity ve token protocol identity** ayrıca sürümlü release metadata'sına eklenmelidir.

### 7.2 Delayed acoustic codebook ve diyalog

Bugünkü `acoustic_delay_frames=1` transform'u continuation yolunda uygulanır; dialogue veri yoluna otomatik ve eşdeğer uygulanmış kabul edilmez. Delay özelliği olan bir diyalog release'i için özel eğitim–inference hizalama testi ve gerçek decoding örnekleri istenir. Başarı kanıtı yoksa `dialogue_turn` capability'sine bu seçenekle onay verilmez. `EOS` ve flush frame davranışı da onaylı protokole bağlanır.

---

## 8. Runtime capability negotiation ve platform profili

Release manifesti hangi özellikleri *gerektirdiğini*, Speech runtime hangi özellikleri *sağladığını* ve Live hangi özellikleri *kullanacağını* ayrı ilan eder. `batch_turn`, `incremental_decode`, `streaming_codec`, `streaming_transformer`, `barge_in`, `duplex`, `speaker_aware`, `agent_bridge` gibi yetenekler **bool tahminleriyle değil, sürümlü capability tanımlarıyla** yönetilir.

Örneğin `batch_turn` çalışırken `streaming_codec` henüz geliştirilmemiş olabilir. Böyle bir release kullanıcıya turn-based demo olarak sunulabilir; ancak gerçek streaming destekli diye listelenemez. Codec ve modelin farklı state mekanizmaları olduğu için ikisinin de incremental çalışması gerekir. Live'ın mikrofon callback'i çalışıyor diye modelin streaming capability'si var kabul edilmez.

### 8.1 Platform profili

Her teknik destek iddiası en az şu bilgilere bağlanır:

- İşletim sistemi ve mimari; Python/PyTorch/CUDA/backend sürümleri.
- GPU vendor, kart adı, raporlanan toplam VRAM, kullanılan precision ve quantization backend.
- Bağlam uzunluğu, chunk boyutu, codebook frame rate, oturum sayısı.
- Peak device memory **ölçüm yöntemi** (`allocated`, `reserved`, harici process toplamı ayrı).
- TTFA, RTF, en azından p50/p95 ve sample sayısı; ses çıkış sınırı açık.
- Testin batch mi real-time mı olduğu ve model/codec/transport bileşenleri.

**2 GB VRAM hedefi:** Sadece FP16 ağırlığın teorik küçük olması veya 6 GB GPU üzerinde `allocated<2 GB` ölçülmesi, gerçek 2 GB cihazda başarı kanıtı değildir. Registry örneği hangi donanımda test ettiğimizi açık tutar ve doğrulanmamış cihazları destek listesine kendiliğinden eklemez. `tested_platforms=[]` boşsa `2GB_VERIFIED` etiketi verilemez.

### 8.2 Capability semantiği sürümlü olmalı

Aynı `streaming` sözcüğü yalnız parçalı PCM gönderme ile tam-duplex model üretimini ifade edebilir; bu belirsizlik yasak. Her capability `id`, `version`, `required_inputs`, `guarantees`, `limitations` taşır. Örneğin `speech.batch_turn.v1` tam cümle sonrası yanıtı; `speech.incremental_generation.v1` token seviyesinde cache'li üretimi; `speech.full_duplex.v1` eşzamanlı dinleme/konuşma davranışını ayrı iddia eder. Live hangi capability'ye ihtiyacı varsa onu açık ister.

---

## 9. Model Card: bilimsel ve kullanıcıya dönük dürüstlük

Her yayınlanabilir release için bir `MODEL_CARD.md` (veya eşdeğer yapılandırılmış kart) zorunludur. Kartın içeriği release'le sürümlü olarak saklanır; bağımsız, değiştirilebilir reklam metni değildir.

**Asgari kart bölümleri:** model kimliği ve sürümü, amaç ve tasarım, kullanılan model/codec, eğitim veri kökeni ve izin sınırları, değerlendirme protokolleri ve örnek sayıları, kalite bulguları, bilinen başarısızlıklar, desteklenmeyen özellikler, hedef donanım ölçümleri, quantization ayrıntıları, kurulum ve offline davranışı, güvenlik/gizlilik, lisans/yeniden dağıtım şartları, kullanıcı beklentisi ve sürüm geçmişi.

### 9.1 Değerlendirmeyi reklam cümlesine dönüştürmeme

- Bir run'ın train loss'unun düşmesi **Türkçe doğal konuşma kalitesinin** kanıtı değildir.
- Codec roundtrip ses örneği tek başına konuşma modelinin diyalog kurabildiğini göstermez.
- Cümle tanıma için bir ASR'nin CER ölçümü doğal diyalog, prosodi ve turn-taking'in yerine geçmez.
- Başarılı bir tek örnek video, 100 örneklik benchmark başarısı sayılmaz.
- “Türkiye'nin ilk...” gibi tarihsel üstünlük iddiaları bağımsız literatür ve açık model araştırması olmadan yazılmaz.
- Üçüncü taraf codec/dataset lisansına dair belirsizlik varsa kart `research_only` diye işaretlenir ve paket varsayılan public kanala çıkmaz.

Kartın ölçümsüz alanlarında `not_measured` ve neden kullanılmalı; sıfır değeri uydurulmamalıdır. Demo kartı ile production kartı aynı görünse bile statüleri farklıdır.

---

## 10. Haklar, veri kökeni, consent ve dağıtım izni

Forge registry, modelleri yalnız “çalışıyor” diye yayımlayamaz. Eğitim veri kaynaklarının ve üçüncü taraf ağırlıkların lisansları ayrıca kontrol edilir. Araştırma için izin verilen kullanım, ticari dağıtım veya halka açık yeniden paylaşım hakkı vermeyebilir. YouTube vb. kaynaklardan teknik olarak ses indirilebilmesi, kullanım/yeniden dağıtım yetkisi anlamına gelmez.

Önerilen `license_review_status` değerleri: `pending`, `approved_for_private_research`, `approved_for_internal_demo`, `approved_for_public_distribution`, `denied`, `restricted`. Her kararda inceleyen, zaman, dayanak belge/artifact, lisans kimliği ve kısıtlar bulunur. `pending` ve `denied` durumundan `stable_public` kanala geçiş mümkün değildir. Otomatik lisans tahmini onayın yerine geçmez.

### 10.1 Veri sızıntısı ve gizlilik

- Corpus kaynak dosyalarında gerçek kişilerin sesleri bulunabilir; ham sesi ve transcriptleri istemeden public model paketine veya GitHub Release'e eklemeyiz.
- Örnek sesler ayrı izinli/anonimleştirilmişse küçük preview olarak sunulabilir; aksi hâlde özel kalır.
- Dataset URI ve yerel tam dosya yolları gizli kullanıcı adı/dizin yapısı içerebilir. Dış manifestte iç dizinler yerine içerik adresi ve soyut dataset referansı tercih edilir.
- Model memorization/risk incelemesi, veri sahipliği ve gerekirse özel veri talepleri için süreç kurulmadan “kişisel veri güvenli” ibaresi kullanılmaz.
- Evaluation holdout split'leri daha iyi test puanı almak için eğitim reçetesine gizlice sızdırılmaz.

### 10.2 Model dosyası ve lisans aynı şey değildir

Hash yalnız dosyanın değişmediğini söyler. Hash'i doğru olan bir modelin hukuken dağıtılabilir, güvenilir veya kötü amaçlı davranışlardan arınmış olduğu sonucu çıkmaz. Üç ayrı kapı tutulur: `integrity_gate`, `rights_gate`, `evaluation_gate`.

---

## 11. Promotion gate: otomatik koşullar ve insan onayı

Release için minimum kapılar, **amaçlanan kanal**a göre tanımlanır. `experimental` bile integrity/uyumluluk kontrolünü atlamaz. `stable` için daha geniş performans, bilimsel kanıt ve regresyon kontrolü gerekir; bu çıta önceden policy revision'ında kayıtlıdır.

| Gate | Experimental local | Demo / internal | Stable / public |
|---|---|---|---|
| Artifact SHA-256, boyut, tam paket | Zorunlu | Zorunlu | Zorunlu |
| Model–codec–token protokol uyumu | Zorunlu | Zorunlu | Zorunlu |
| Checkpoint/load smoke | Zorunlu | Zorunlu | Zorunlu |
| Dataset ve hak durumu | Araştırma kullanımına izin | Gösterim izni | Yeniden dağıtım izni |
| En az bir gerçek evaluation raporu | Zorunlu, sınırlar açık | Zorunlu, demo ile ilgili | Zorunlu, geniş kapsam |
| Türkçe dinleme/kalite kanıtı | Araştırma kısıtlarıyla | Demo iddiasını destekler | Önceden tanımlı kalite kapısı |
| Hedef cihaz performansı | Ölçülmediyse `unsupported` | Demo cihazında ölçülmüş | İlan edilen her profil için kanıt |
| Bilinen regresyonlar | Açıkça kayıtlı | Kritik regresyon yok | Policy eşiklerini geçer |
| Model card ve risk açıklaması | Zorunlu | Zorunlu | Zorunlu |
| İnsan onayı, audit | Zorunlu | Zorunlu | Zorunlu |

**Sayı uydurmama kuralı:** Eşikler (`TTFA<...`, `RTF<...`, MOS gibi) araştırma protokolü ve gerçek cihazda kalibre edilmeden kesin üretim limiti olarak belgelenmez. Policy somut metrikleri, minimum örnek sayısını, donanımı, aggregation yöntemini ve geçici istisnaları sürümlü biçimde saklar. Eksik ölçüm `not_measured` olarak kalır ve hedef kanala göre promotion'ı engeller.

### 11.1 Approval kaydı

`approval_id`, `candidate_id`, `candidate_manifest_digest`, `policy_id`, `policy_revision`, `reviewer`, `scope`, `decision`, `reason`, `evidence_refs`, `timestamp_utc` alanları zorunlu olsun. Yerel V1'de reviewer makine kullanıcısı olabilir; sistem güçlü kimlik doğrulama olmadan buna “kurumsal güvenli imza” dememelidir. Promotion, kullanıcı eylemi veya ayrıca yetkilendirilmiş otomasyon ile açıkça yapılır. Codex'in önce görev tamamlayıp sonra kendiliğinden tüm model sürümlerini dağıtması yasaktır.

### 11.2 Deneysel istisna yönetimi

İstisna kanıtı olmayan bir gate'i başarılı göstermez. Yalnız belirli policy ile, belirli kanal ve süre için, yetkili onayla `waiver` eklenebilir. Güvenlik/dosya bütünlüğü/codec uyumu için `waiver` yoktur. “Türkçe MOS henüz ölçülmedi” gibi araştırma eksikliği varsa yalnız düşük riskli araştırma kanalında açık sınırlamayla kalabilir; public stable'a geçmez.

---

## 12. Yayın kanalları ve destek politikası

Önerilen kanallar:

- `research`: yalnız araştırma için özel ve gerektiğinde erişim sınırlı.
- `experimental`: bütünlük doğrulanmış, sınırlamalar açık, kullanıcı tarafından seçilir.
- `demo`: belirli donanım ve senaryo üzerinde doğrulanmış gösterim sürümü.
- `stable`: genel kullanıma önerilmesi için ayrıca onaylanmış release.

Bu kanallar *model kalitesinin nesnel sıralaması değildir*. `experimental` üzerindeki sürüm bir ölçümde `stable`dan daha hızlı olabilir; ancak kanıt kapsamı veya riskleri farklıdır. Kanal işaretçisi (`channel_pointer`) değiştirilebilir ama yönettiği `release_id` değişmez.

`latest` tek başına kanal değildir ve güvenli seçme yöntemi değildir. En yeni sürüm, belirli cihazda en uyumlu sürüm olmayabilir. Live model listesi kullanıcıya kanal, doğrulanmış cihaz profili, disk ihtiyacı ve bilinen sınırlamaları göstermelidir.

### 12.1 Stable pointer güncelleme protokolü

1. Hedef release halen available mı ve revocation yok mu?
2. Pinned manifest digest doğrulanıyor mu?
3. Kanal policy ve insan onayı geçerli mi?
4. Önceki pointer ve hedef değer audit event'e yazıldı mı?
5. Tek bir transaction ile `stable` pointer'ı değiştir.
6. Kurulum sistemlerine yeni pointer'ı bildir; açık session'ları zorla değiştirme.
7. Yükleme/sağlık kontrolleri başarısızsa önceki pointer'a kontrollü rollback yap.

Kanal pointer güncellemesi **artifact'in yeniden yayımlanması** değildir. Release dosyaları aynı kalır. Audit olayı `previous_release_id`, `new_release_id`, `actor`, `reason`, `timestamp` içerir.

---

## 13. Artifact depolama: SQLite + içerik adresli dosyalar

V1 için tek makinede çalışan local-first tasarım yeterlidir. SQLite küçük metadata, state, onaylar ve artifact referansları tutar. Büyük `.pt`, `.safetensors`, codec ağırlığı, WAV, rapor ve benchmark JSONL dosyaları dosya sisteminde kalır. Aynı checkpoint'in iki adayda tekrar kullanılması durumunda içeriğe dayalı adresleme gereksiz kopyalamayı azaltır.

Önerilen fiziksel yapı:

```text
forge-data/
  registry/
    registry.sqlite3
    blobs/
      sha256/
        ab/
          abcd...                              # SHA-256 ile içerik adresli
    candidates/
      <candidate_id>/
        candidate.snapshot.json
        verification.json
    releases/
      <model_id>/
        <release_id>/
          release.json                         # immutable manifest
          release.lock                         # detached manifest digest
          MODEL_CARD.md
          artifacts/                           # hardlink/copy/snapshot policy
    channels/
      <model_id>/
        experimental.json
        stable.json
    events/
      registry-events.jsonl
    staging/
      <transfer_or_publish_id>/
    quarantine/
      <artifact_id>/
```

**Yolların görevi:** `releases` altında isimlendirilmiş klasör kullanımı insana okunabilirlik içindir; güven kimliği manifest digest'dir. Dosyaların bütünlüğü path adına veya dosyanın `last_modified` zamanına dayanmaz. Staging tamamlanmadan `releases/` altına “aktif” paket konmaz.

### 13.1 Dosya kopyalama, hardlink ve değişmezlik

Hardlink kullanılması storage tasarrufu sağlar ama yazılabilir hardlink üzerinden iki farklı release'in byte'larının değişmesi risktir. Blob saklama dizini yazmaya kapalı ve integrity check ile korunmuyorsa V1'de fiziksel kopya tercih edilebilir. Symlink/junction kullanımı paket kökü dışına kaçış oluşturabileceği için Windows dahil dikkatli policy ister. `chmod read-only` tek başına güçlü güvenlik sınırı değildir.

### 13.2 Garbage collection

Registry'de hiç referansı kalmamış artifact'ler hemen silinmez. Önce `candidate`, `release`, `evaluation`, `run`, `channel`, aktif Live kurulumları ve retention policy üzerinden bir **reachability graph** oluşturulur. Audit veya yasal saklama gereksinimi varsa silme engellenir. Dry-run raporu, kullanıcı onayı, backup ve doğrulanabilir transaction olmadan otomatik toplu silme yapılmaz. `revoked` bir sürümün kanıtlarını yok etme izni değildir.

---

## 14. SQLite şema taslağı ve transaction sınırları

Aşağıdaki SQL yalnız **TARGET_V1** taslağıdır; Forge repo içinde henüz bu tablo ve migration'lar yoktur. `runs`, `attempts`, `evaluations` ve ortak artifact katalogu önceki Forge belgeleriyle aynı veri kaynağını paylaşmalıdır. İkinci bir çakışan `experiments.sqlite` kopyası oluşturulmaz.

```sql
CREATE TABLE IF NOT EXISTS registry_candidates (
  candidate_id TEXT PRIMARY KEY,
  model_id TEXT NOT NULL,
  source_run_id TEXT NOT NULL,
  source_attempt_id TEXT,
  snapshot_digest TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at_utc TEXT NOT NULL,
  updated_at_utc TEXT NOT NULL,
  CHECK (status IN ('draft','verification_pending','verification_failed',
                   'review_ready','approved_for_experimental','rejected'))
);
CREATE TABLE IF NOT EXISTS registry_releases (
  release_id TEXT PRIMARY KEY,
  model_id TEXT NOT NULL,
  model_version TEXT NOT NULL,
  manifest_sha256 TEXT NOT NULL UNIQUE,
  candidate_id TEXT NOT NULL,
  release_channel_at_creation TEXT NOT NULL,
  created_at_utc TEXT NOT NULL,
  FOREIGN KEY(candidate_id) REFERENCES registry_candidates(candidate_id),
  UNIQUE(model_id, model_version)
);
CREATE TABLE IF NOT EXISTS registry_release_status (
  release_id TEXT PRIMARY KEY,
  visibility_status TEXT NOT NULL,
  reason TEXT,
  updated_at_utc TEXT NOT NULL,
  FOREIGN KEY(release_id) REFERENCES registry_releases(release_id),
  CHECK (visibility_status IN ('available','deprecated','quarantined','revoked'))
);
CREATE TABLE IF NOT EXISTS registry_channels (
  model_id TEXT NOT NULL,
  channel TEXT NOT NULL,
  release_id TEXT NOT NULL,
  revision INTEGER NOT NULL DEFAULT 1,
  updated_at_utc TEXT NOT NULL,
  PRIMARY KEY(model_id, channel),
  FOREIGN KEY(release_id) REFERENCES registry_releases(release_id)
);
CREATE TABLE IF NOT EXISTS registry_approvals (
  approval_id TEXT PRIMARY KEY,
  candidate_id TEXT NOT NULL,
  policy_revision TEXT NOT NULL,
  decision TEXT NOT NULL,
  actor TEXT NOT NULL,
  reason TEXT NOT NULL,
  evidence_digest TEXT NOT NULL,
  created_at_utc TEXT NOT NULL,
  FOREIGN KEY(candidate_id) REFERENCES registry_candidates(candidate_id)
);
CREATE TABLE IF NOT EXISTS registry_events (
  event_id TEXT PRIMARY KEY,
  entity_kind TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  event_kind TEXT NOT NULL,
  actor TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  occurred_at_utc TEXT NOT NULL
);
```

SQL pratikte migration sistemi, foreign key enforcement (`PRAGMA foreign_keys=ON`), WAL ve kilit retry politikasıyla uygulanmalıdır. Modelin farklı quantization/backend variant'larının aynı sürümde yayımlanmasına izin verilecekse `UNIQUE(model_id, model_version)` bilinçli olarak `(model_id, model_version, variant_id)` ile güncellenir; bu belgenin V1 politikası aynı version'a tek immutable release bağlar.

### 14.1 Dosya sistemi + DB iki aşamalı dayanıklılık

SQLite transaction ile `rename` işlemi tek bir atomik transaction oluşturmaz. Bu nedenle şu strateji uygulanır: (a) `staging` dosyaları yazılır, flush/fsync ve checksum yapılır; (b) doğrulanmış manifest ile `sealed` aday hazırlanır; (c) dosyalar aynı file system üzerinde atomik rename ile final konuma taşınır; (d) DB release insert + event tek transaction'a alınır; (e) crash recovery, orphan staging/final klasörleri yeniden tarar ve tutarlı duruma getirir. Windows AV/lock kaynaklı rename hataları test edilir; Drive mount üzerinde atomik rename garantisi varsayılmaz.

**Çift yayın yarışı:** Aynı `release_id` için iki process yarışırsa UNIQUE constraint, optimistic compare-and-swap ve idempotency key ile yalnız biri kazanır. Diğeri mevcut digest aynıysa aynı kayıt referansını döner; farklıysa conflict verir. `model_version` overwrite edilmez.

---

## 15. Integrity: SHA-256, kimlik ve güven kökü

Artifact kayıt anında dosya akışı parça parça okunarak SHA-256 hesaplanır; dev dosya tek seferde RAM'e alınmaz. `size_bytes` gerçek dosya boyutudur. Okuma sırasında dosya değişebilirse snapshot'a kopyalandıktan sonra hash alınır. İndirme veya Drive kopyalama sırasında farklı kaynaklardan gelen digest'ler karşılaştırılır; mismatch durumunda dosya quarantine edilir.

**SHA-256 sınırı:** Sağlam bir hash, verinin beklenen byte'larla aynı olduğunu kanıtlar; o byte'ların **güvenilir üreticiden geldiğini** veya Python pickle'ın güvenli olduğunu kanıtlamaz. Güvenilir release metadata kaynağı, mümkünse imza doğrulaması, izin verilen publisher listesi ve hash pinning ayrıca gerekir. V1 yerel geliştirici kullanımında güçlü kriptografik imza altyapısı kurulmadan “signed supply chain” iddiası yapılmaz.

### 15.1 Hash doğrulama sırası

1. Manifest boyut limiti ve parse doğrulaması.
2. Manifest digest'i detached `release.lock` veya kayıtlı `manifest_sha256` ile eşleştir.
3. Her artifact için güvenli path resolve, dosya tipi ve boyut kontrolü.
4. Stream SHA-256 hesapla; sabit hex karşılaştırması yap.
5. İç metadata'yı Speech adapter üzerinden doğrula; yalnız dosyanın varlığı yeterli değildir.
6. Yükleme gerekiyorsa trust policy ve sınırlı deserialization uygula.
7. Başarılı olunca signed/pinned release resolution record oluştur; işlem süresince dosyaların değiştirilmediğinden emin ol.

### 15.2 Örnek doğrulayıcı çekirdeği — HEDEF Python taslağı

```python
from __future__ import annotations

import hashlib
import re
from pathlib import Path

_HEX_64 = re.compile(r"[0-9a-f]{64}\Z")


def sha256_file(path: Path, chunk_bytes: int = 1024 * 1024) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as stream:
        for block in iter(lambda: stream.read(chunk_bytes), b""):
            digest.update(block)
    return digest.hexdigest()


def verify_artifact(path: Path, expected_hash: str, expected_size: int) -> None:
    if _HEX_64.fullmatch(expected_hash) is None or expected_size <= 0:
        raise ValueError("invalid artifact metadata")
    if not path.is_file() or path.stat().st_size != expected_size:
        raise ValueError("artifact missing or size mismatch")
    if sha256_file(path) != expected_hash:
        raise ValueError("artifact sha256 mismatch")
```

Bu kısa fonksiyon **yalnız bütünlük doğrulamasının çekirdeğidir**; package root containment, symlink/junction savunması, TOCTOU, trust/signature, izinler ve gerçek Speech model yükleme kontrollerini içermez. Onaylı production loader yerine tek başına kullanılamaz. Manifestteki `model.pt` gibi path'ler güvenli paket kökü çözümlemesiyle sağlanmalıdır.

---

## 16. Güvenlik tehdit modeli ve güvensiz kaynaklar

Registry'nin yüksek riskli girdileri internetten indirilen model dosyaları, başkasının ürettiği checkpoint'ler, kullanıcının seçtiği klasörler, dış manifest JSON'ları, SQLite importları ve Colab/Drive artifact paketleridir.

| Tehdit | Örnek | Savunma |
|---|---|---|
| Path traversal | `../../secret.txt` | Root containment, normalize path, allowlist |
| Windows path kaçışı | `C:\\Users\\...`, junction | Windows canonicalization ve güvenli açma |
| Symlink attack | Staging'i kök dışına bağlama | Symlink/junction policy, handle-based checks |
| Deserialization | Untrusted `.pt` pickle payload | Trust gate, sınırlı loader, isolation gerektiğinde |
| ZIP slip | Arşivde `../` veya absolute path | Önce entries validate, boyut ve derinlik sınırı |
| Zip bomb | Küçük arşivden dev unpack | Uncompressed size, dosya sayısı ve kota |
| Disk exhaustion | Sonsuz checkpoint/preview | Quota, retention, capacity preflight |
| Tamper | Dosya publish sonrası değişiyor | Immutable blobs, yeniden hash, manifest pinning |
| Supply chain | Yanlış yayıncı, sahte model | Trusted source, provenance/imza policy |
| Metadata injection | Manifestte shell komutu | **Asla shell olarak çalıştırma** |
| Secret leakage | Drive token log'a yazıldı | Secret redaction, erişim kontrolü |
| PII leakage | Gerçek kişinin ses kaydı release içine girdi | Rights gate, preview opt-in |
| Race condition | Aynı ID'ye iki release | UNIQUE, CAS, idempotency |

**Hak olmayan bir dosyayı teknik olarak indirebilmek dağıtım izni değildir.** “Güvenli pickle” gibi mutlak iddialardan kaçınılır. Speech loader `torch.load(weights_only=True)` kullansa bile dosya kökeni, boyut, hash, dtype ve cihaz kaynak limitleri kontrol edilir.

### 16.1 URI ve indirme davranışı

`file://`, `drive://`, `https://` ve blob URI'leri tek tip shell/file path olarak ele alınmaz. V1 local-first kurulumda sadece açıkça sağlanmış yerel dizin veya kullanıcı tarafından doğrulanmış paket kabul edilir. Uzak artifact indirme ayrı `fetch` işlemi ve kullanıcı izniyle, TLS, retry/backoff, kesilebilir transfer ve checksum doğrulamasıyla yapılır. Manifest içindeki URL, otomatik network yetkisi vermez; SSRF ve internal network erişimi riski nedeniyle izinli domain/path politikası gerekir.

---

## 17. Reproducibility ve lineage: modeli yeniden izlemek

Registry tek başına eğitimi deterministik yapmaz; yalnız doğrulanabilen üretim geçmişini izlenebilir kılar. Bir release'e geri dönüldüğünde aşağıdaki zincir kurulabilmelidir:

```text
release_id
  ├─ manifest_sha256
  ├─ model weights SHA-256 + config SHA-256
  ├─ codec release SHA-256 + token protocol version
  ├─ source run_id / attempt_id
  │    ├─ resolved recipe digest
  │    ├─ Speech Git commit SHA
  │    ├─ training dataset manifest SHA-256
  │    ├─ training environment / lockfile digest
  │    ├─ original checkpoint SHA-256
  │    └─ resume provenance / seed notes
  ├─ evaluation_id(s)
  │    ├─ protocol digest and holdout manifest digest
  │    ├─ evaluator version, device conditions
  │    └─ full results + failures
  └─ rights review / promotion policy / approval event
```

Bu zincirde eksik öğe varsa `provenance_level` açıkça düşürülür veya promotion bloke edilir. Hash'ler kopyalanmış dosyaları tanımlasa da aynı eğitimin farklı GPU'da bit bit aynı sonucu üreteceğini otomatik kanıtlamaz. Mevcut Speech checkpoint bazı eğitim state'lerini saklar; RNG, DataLoader cursor ve eksternal dataset sıralamasının eksiksiz devamı ayrıca geliştirme gerektirebilir.

### 17.1 Imported external Colab runs

Dış Colab oturumundan gelen checkpoint `origin=external_colab`, `imported_at_utc`, `source_manifest_digest`, `original_run_id`, `verification_level` ve varsa güvenilir log/revision bilgisiyle alınır. Forge'un kendisi başlatmadığı eğitim `managed_by_forge=true` olarak işaretlenmez. Eksik dataset license veya environment hash'i varsa `research_only` kalır. Yanlış bir tam deterministik resume hikâyesi oluşturmak yerine raporda `resume_reproducibility=partial` denir.

---

## 18. Evaluation kanıtının bağlanması ve seçim politikası

Release promotion kapısı `EVALUATION.md` içindeki sürümlü protokol ve raporu okur; kendi başına yeni metrik hesaplama kütüphanesi yazmaz. Belirli bir `evaluation_id` ancak **o release'in model weights, codec identity, preprocessing ve dataset/holdout** kombinasyonuna bağlıysa kanıt olarak kullanılır. Başka checkpoint'e veya daha sonra değişen codec'e ait sonuçların kopyalanması yasaktır.

### 18.1 Minimum değerlendirme kanıtı

- Protocol snapshot ve hash.
- Model/codec weights hash ve config version.
- Dataset manifest hash, split ve leakage audit sonucu.
- Örnek başına attempted/completed/failed kaydı.
- Primary metric + açık biçimde missing secondary metrics.
- Sampling/aggregation yöntemi, gerekiyorsa confidence interval.
- Device/backend/precision/quantization bilgileri.
- Bilinen hatalar ve sesli örneklerin kullanım izni.

**“Test geçti” başka bir kavramdır:** `pytest`/contract test şema ve fonksiyon doğruluğunu gösterir. İnsanların doğal Türkçe konuşmayı anladığını veya 2 GB VRAM hedefini karşılandığını göstermez.

### 18.2 Regression gate

Baseline ile candidate karşılaştırması `EXPERIMENTS.md` uyumluluk denetiminden geçmelidir. Mimi token CE ile Kaiwen Codec CE doğrudan tek sıralama metriği olarak kullanılamaz. Türkçe konuşma ve codec reconstruction, eş koşullarda kaydedilen ses çıktılarıyla karşılaştırılır. Kalite artarken gecikme artmışsa karar çok amaçlı kalite/latency/VRAM profiline göre verilir; hiçbir regresyon gizlenmez.

---

## 19. Model seçim algoritması: “en yeni” değil, “uyumlu ve kanıtlı”

Live model listesi kullanıcının donanımı ve istediği özelliklerle uyumlu release'leri göstermelidir. İyi bir model seçimi şu sıradadır:

1. Kullanıcının açık seçimi veya profil filtresi (`experimental` dahil).
2. Manifest şeması ve runtime sürümü uyumluluğu.
3. Artifact local availability ve verification.
4. Codec/model identity ve feature capability uyumu.
5. Hedef cihazda kanıtlanmış performans; yoksa `untested` uyarısı.
6. Lisans/dağıtım izni ve revocation/visibility durumu.
7. Önceden belirlenen preference (kalite/latency/disk) ve güvenli fallback.

Bu sıralamadan sonra `stable` kanalından bir uygun model önerilebilir. Ama arka planda daha büyük model indirmek, kullanıcı izin vermeden veri aktarmak veya özellik desteği uydurmak olmaz. `2 GB sınıfı` için doğrulanmamış model listede “2 GB destekli” etiketi alamaz.

### 19.1 Örnek seçim raporu — statik taslak

```json
{
  "request": {"platform": "windows-x86_64", "required": ["batch_turn"], "vram_limit_bytes": 2147483648},
  "selected_release_id": null,
  "decision": "no_verified_compatible_release",
  "rejections": [
    {"release_id": "example-exp-001", "reason": "platform_not_benchmarked"},
    {"release_id": "example-exp-002", "reason": "codec_artifact_missing"}
  ],
  "next_action": "show_manual_selection_with_limitations"
}
```

Bu rapor gerçek model araması sonucu değildir. Başarısız seçim, gizli otomatik indirme veya ölçümsüz modelin başarısızlığı saklanarak default seçilmesiyle çözülemez.

---

## 20. Kaiwen Live'a model aktarım protokolü

Forge bir release paketinin meta/artifact bütünlüğünü hazırlayabilir; **Live bir model inference runtime'ı değildir**. Live, Speech loader'a paket konumunu, istenen capabilities ve memory preference iletir. Modelin Python yorumlayıcısı veya yerel sidecar, UI render thread'ini bloke etmeden yükleme yapar.

```text
Forge registry: release sealed
       |
       v
Live: user selects release / explicit fetch
       |
       v
Live staging: capacity check, download/copy, size+sha256 verification
       |
       v
Speech runtime: codec + transformer + token protocol loading
       |
       v
Live: health check and capability negotiation
       |
       +── failed --> clean staging; previous release stays active
       |
       v
Live: activate for NEW session; previous session remains pinned
```

### 20.1 Install/activate state'leri

`not_installed → fetching → staged → verified → loading → ready → active`. Hata durumlarında `failed_integrity`, `failed_compatibility`, `failed_runtime`, `failed_oom`, `cancelled` gibi ayrı nedenler saklanır. `active` yalnız Speech runtime `READY` verdiğinde gerçekleşir. Staged dosya kısmi indirme sırasında çalıştırılamaz. Model değişimi açık oturumu sessizce dönüştürmez; `session_epoch`, playback queue ve GPU model kaynağı pinning'i Live/Speech sözleşmesindeki gibi yönetilir.

### 20.2 Atomic switch ve rollback

Live'ın `active_release_id` seçimi atomik olarak güncellenir. Yeni release'in yüklenmesi OOM veya codec mismatch ile başarısızsa `active_release_id` önceki doğrulanmış release'i göstermeyi sürdürür. Yükleme sırasında iki model birden GPU'da tutuluyorsa 2 GB bütçesi aşılabilir; bellek kısıtında sırayla unload/reload ve kontrollü yeniden başlatma politikası gerekebilir. “Sıfır kesinti” zorunlu hedef değildir; güvenli ve kullanıcıya açık geçiş önceliklidir.

### 20.3 Görünen kullanıcı bilgileri

Live kullanıcıya `model_id`, `model_version`, release channel, codec name, desteklenen batch/streaming durumları, indirme boyutu, depolama ihtiyacı, test edilmiş cihazlar, bilinen kalite sınırları ve model card bağlantısı göstermelidir. Arayüzün Kaiwen Design ile çizilmesi model uyumluluğu mantığını Design'a taşımaz.

---

## 21. Uzak depolama, Google Drive ve kesintili bağlantı

V1'in otoritesi **yerel kayıt ve doğrulanmış artifact setidir**. Google Drive 5 TB alanına sahip olmak Colab runtime diskinin hızlı/sınırsız olduğu anlamına gelmez. Colab ephemeral diskte oluşturulan checkpoint önce tamamen yazılır; boyut ve hash doğrulanır; sonra Drive'a aktarılır. Transfer bir `partial` dosya üzerinden devam ediyorsa final dosya olarak işaretlenmez. Uzak kaynak metadata'sı dosya sahipliği ve aktarım geçmişiyle birlikte kaydedilir.

### 21.1 Transfer state machine

`requested → downloading/uploading → transferred → hash_verified → imported`. Hata hâlinde `retryable_error`, `permanent_error`, `cancelled`, `unknown_remote_state`. Colab oturumu koptuğunda remote işin bittiğine dair veri yoksa `unknown_remote_state` kullanılır. Aynı dosya ikinci kez geldiğinde SHA-256 içerik adreslemesi ve kaynak kimliği ile idempotent import yapılır; bozuk veya eksik dosya quarantine'e taşınır.

### 21.2 Byte-size, quota ve hız

Transfer başlamadan önce hedef diskin boş alanı, indirilecek toplam byte, staging overhead ve beklenen geçici dosya tüketimi hesaplanır. Large model assetleri için chunked transfer ve artan backoff kullanılır; sınırsız retry yapılmaz. Drive klasör adını artifact identity sanmayız: aynı isim farklı içeriğe sahip olabilir. İndirme URL'sine token gömülürse log/redaction ve yetki politikası gerekir.

---

## 22. Release varyantları: FP16, INT8, INT4, CPU ve diğer runtime'lar

Aynı modelin quantization sonrası ağırlık byte'ları değişir; dolayısıyla **aynı artifact digest ile gösterilemez**. Quantization yeni `variant_id`, yeni model weights digest, dönüştürme aracı sürümü, target backend, precision ve ayrı evaluation/performance kanıtı gerektirir. Araştırma mantığında “aynı aile” olabilir; dağıtım kimliği farklıdır.

V1 basitliği için `(model_id, model_version)` başına tek release önerilir. İleride `variant_id` ve platform bağımlı paketler gerektiğinde şema major/minor uygunluğu ve UNIQUE indeksleri açık migration ile genişletilir. Önce tek bir gerçek FP32/FP16 release'in doğru yüklenmesi sağlanmadan çoklu backend ve otomatik quantization farm'ı yazılmayacak.

### 22.1 INT4 gerçeği

Parametre × 4 bit hesabı yalnız **teorik ağırlık alt sınırıdır**. Scales, zero points, packing, temporary buffers, KV cache, codec, CUDA context ve kernel padding toplam VRAM'i etkiler. Quantization sonrası ses doğallığı ve Türkçe telaffuz tekrar ölçülmelidir. Registry, ölçülmemiş kalite ve bellek iyileşmesini model kartında veri olarak sunmaz.

---

## 23. Güvenli yayın ve semantik sürümleme

İnsan dostu model sürümü önerisi SemVer'dir; model araştırmasında SemVer tek başına “eski sürüme API uyumu” garantisi sunmaz. Bu nedenle şu sürümler ayrı tutulur:

| Sürüm | Sahibi | Ne değişirse artar? |
|---|---|---|
| `registry_schema_version` | Forge | Manifest/kayıt formatı |
| `forge_package_version` | Forge | CLI/library davranışı |
| `speech_runtime_api_version` | Speech | Loader, Python SDK ve capability sözleşmesi |
| `model_version` | Model ailesi release | Ağırlık/kalite/konfigürasyonun açıklanması |
| `codec_release_id` | Speech Codec | Token mapping ve codec ağırlıkları |
| `token_protocol_version` | Speech | Control IDs, delay/layout semantiği |
| `evaluation_protocol_revision` | Forge Evaluation | Ölçüm yöntemi/veri/aggregation |
| `channel_pointer_revision` | Forge Registry | Stable/experimental yönlendirme geçmişi |

**Ayrım:** Yeni `model_version` aynı codec'i kullanabilir; yeni codec release'i ise eski modelin kullanılabilirliğini otomatik sağlamaz. Registry release her bağımlılığı **tam artifact ID/digest** ile pin'ler. `latest` etiketini bağımlılık olarak kullanmak YASAKTIR.

### 23.1 Changelog ve release notes

Her yeni release için kaynak model, önceki referans model, önemli iyileştirme/regresyonlar, değerlendirme kapsamı, donanım desteği, değişen codec/token protocol, migration gereksinimi ve rollback yolu açıklanmalıdır. Örneğin yalnız “daha hızlı” değil: hangi cihazda, hangi yükte, p50/p95 değerleriyle, hangi kalite etkisiyle hızlı olduğu yazılır.

---

## 24. Revocation, güvenlik olayı ve geri alma

Hatalı veya izinsiz dağıtılmış bir release'in neden durdurulduğu ve hangi kurulumların etkilenebileceği audit'te korunur. Revocation tipleri: `artifact_corruption`, `license_violation`, `security_issue`, `unsafe_behavior`, `metadata_fraud`, `runtime_incompatibility`. Registry dosyayı tarihten silemez; **uygun görünürlük ve erişim politikasıyla** yeni yüklemeyi durdurur, daha önce etkin olan paketlere uygulanacak uyarıyı Live'a iletir.

### 24.1 Rollback prosedürü

1. Sorunla ilgili `incident_id` ve gerekçe kaydet.
2. Etkilenen release'leri `quarantined` veya `revoked` olarak işaretle.
3. Channel pointer'ı son **halen doğrulanmış ve izinli** release'e al.
4. Live yeni oturumlarda sorunlu release'i seçmesin; açık oturumun güvenli kapatılma politikasını uygula.
5. Yerel cache'leri yeniden hash kontrolünden geçir; onaysız indirmeyi durdur.
6. Model/codec/rights etkisini ve kullanıcıya gösterilen mesajı raporla.
7. Sorun çözüldüyse orijinal release'i sessiz değiştirme; yeni release ve bağımsız değerlendirme oluştur.

Rollback “eskisine dön” butonu kadar basit olmayabilir: codec, runtime ve veri formatı birlikte değişmiş olabilir. Geri dönecek sürümün asgari API/codec uyumluluğu ayrıca doğrulanır.

---

## 25. Hata sözleşmesi ve fail-closed davranış

| Hata kodu | Tetikleyici | Sistem davranışı |
|---|---|---|
| `REGISTRY_SCHEMA_UNSUPPORTED` | Tanınmayan major/schema | Release reddedilir; migrate önerilir |
| `RELEASE_ID_COLLISION` | Aynı ID farklı digest | İşlem iptal, audit |
| `MODEL_INTEGRITY_ERROR` | Boyut veya hash mismatch | Quarantine, install yok |
| `UNTRUSTED_ARTIFACT` | Güven kaynağı doğrulanmadı | Deserialization engellenir |
| `CODEC_IDENTITY_MISMATCH` | Codec weights/token mapping farklı | Loader başlatılmaz |
| `MODEL_INCOMPATIBLE` | Model config ve runtime uyuşmaz | Kullanıcıya uyum bilgisi |
| `CAPABILITY_UNAVAILABLE` | Streaming gibi destek yok | Yalnız gerçekten var olan mod sunulur |
| `RIGHTS_REVIEW_REQUIRED` | Lisans/consent kanıtı eksik | Promotion bloke |
| `EVALUATION_EVIDENCE_MISSING` | Gerekli rapor yok | Promotion bloke |
| `DEVICE_PROFILE_UNVERIFIED` | VRAM/TTFA kanıtı yok | Uyum iddiası yapılmaz |
| `APPROVAL_REQUIRED` | İnsan onayı yok | Release oluşturulmaz |
| `NO_DISK_SPACE` | Staging alanı yetersiz | Transfer başlamaz veya güvenle kesilir |
| `REMOTE_STATE_UNKNOWN` | Colab/Drive bağlantısı kesildi | Başarı iddiası yok; reconcile gerekir |
| `REGISTRY_BUSY` | DB lock/yarış | Kontrollü retry, idempotent davranış |
| `RELEASE_REVOKED` | Güvenlik/lisans iptali | Yeni yüklemeyi engelle |

Hata kodları domain seviyesinde makine tarafından anlaşılır; kullanıcı mesajları kısa, eyleme dönük ve gizli path/token içermemelidir. Teknik hata ayrıntısı structured log'a gider. **Hata sonrası sistemi sıfırlamak veya kullanıcı verisini silmek varsayılan çözüm değildir.**

---

## 26. Python SDK hedefi ve katman tasarımı

V1, küçük ve test edilebilir modüllerden oluşur. Gereksiz plugin marketplace, mikroservis veya ağ protokolü V1 için gerekli değildir.

```text
kaiwen-forge/
  pyproject.toml
  src/kaiwen_forge/
    registry/
      models.py             # immutable domain DTO, typed enums
      schema.py             # manifest validation / schema versions
      identity.py           # canonical serialization, IDs, SHA-256
      store.py              # SQLite metadata + blob storage
      candidates.py         # candidate lifecycle and verification
      compatibility.py      # Speech adapter compatibility call
      policy.py             # promotion rules and evidence gates
      releases.py           # seal/publish/revoke
      channels.py           # pointer updates and rollback
      transfer.py           # optional explicit transfer
      audit.py              # append-only events
    integrations/
      speech/
        registry_adapter.py # model/codec/checkpoint format checks
    cli.py
  tests/
    registry/
      test_manifest.py
      test_integrity.py
      test_promotion.py
      test_rollback.py
      test_security.py
      test_idempotency.py
  docs/
    MODEL_REGISTRY.md
```

Bunlar **hedef modül yollarıdır**; güncel Forge reposunda henüz yoktur. İlk PR'da tüm klasörleri boş açmak yerine çalışan validation/store dilimiyle başlayın.

### 26.1 Hedef tipli arayüz taslağı — henüz uygulanmadı

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Literal, Protocol


@dataclass(frozen=True)
class ReleaseRef:
    release_id: str
    model_id: str
    manifest_sha256: str


@dataclass(frozen=True)
class VerificationResult:
    status: Literal["verified", "blocked", "failed"]
    reasons: tuple[str, ...]


class ModelRegistry(Protocol):
    def register_candidate(self, snapshot_path: Path, *, request_id: str) -> str: ...
    def verify_candidate(self, candidate_id: str) -> VerificationResult: ...
    def seal_release(self, candidate_id: str, *, approval_id: str) -> ReleaseRef: ...
    def get_release(self, release_id: str) -> ReleaseRef: ...
    def resolve_channel(self, model_id: str, channel: str) -> ReleaseRef: ...
    def revoke_release(self, release_id: str, *, reason: str, actor: str) -> None: ...
```

Bu bir `Protocol` taslağıdır; `...` kullanılan metotlar örnek uygulama değildir. Gerçek implementasyon typed error/result, trust policy, transaction ve concurrency semantiğiyle tamamlanacaktır. `release_id` döndüren fonksiyon artifact'lerin fiziksel olarak yüklendiğini ayrıca doğrulamadan çağırana kullanılabilir model sözü vermez.

### 26.2 Forge CLI hedefi — bugün çalışmaz

```bash
# HEDEF KOMUT SÖZLEŞMESİ — implementasyon yok
forge registry candidate add --run example-run-001 --checkpoint artifacts/latest.pt
forge registry candidate inspect example-candidate-001
forge registry candidate verify example-candidate-001
forge registry candidate approve example-candidate-001 --scope experimental
forge registry release create --candidate example-candidate-001
forge registry release inspect example-release-001
forge registry release verify example-release-001
forge registry channel set --model kaiwen-speech-tiny --channel experimental --release example-release-001
forge registry channel history --model kaiwen-speech-tiny --channel experimental
forge registry release revoke example-release-001 --reason integrity_incident
forge registry gc --dry-run
```

CLI isimleri `README.md` içindeki geniş taslakla çatışırsa tek public CLI sözleşmesi seçilip hem dokümanlar hem testler güncellenir. Bu liste harici servisin mevcut API'si değildir; komutları terminalde hemen çalıştırmak için değildir. Önce `forge registry inspect/verify` gibi tek doğrulanabilir alt kümeyle başlanır.

---

## 27. Sürüm sözleşmesi için JSON Schema ve fixture yaklaşımı

Release formatının alanları prose ile sınırlı kalmamalıdır. `registry/release.schema.json` gibi gerçek dosyada JSON Schema 2020-12 veya kontrollü Pydantic model seçilir; ikinci paralel ve tutarsız şema kaynağı tutulmaz. Dosya tipi, `additionalProperties`, `required`, numeric limits, SHA-256 regex, nested compatibility ve enum değerleri test edilir. Bazı çapraz alan koşulları (codec/model identity, approval digest) salt JSON Schema ile değil domain validation ile yapılır.

**Golden fixtures:**

- Geçerli ve bütünlük doğrulanmış küçük sahte tensor dosyalarıyla bir başarılı local release.
- Yanlış SHA-256, yanlış `size_bytes`, bozuk manifest.
- Eski major schema ve eksik zorunlu alan.
- Farklı codec ID ama aynı Q/K değerleri.
- Farklı token special ID veya acoustic delay.
- `release_channel=stable` ama evaluation/rights/approval yok.
- Revoked release'in yeniden seçilmeye çalışılması.
- Windows traversal/symlink/junction, UNC/network path edge cases.
- Tam dosya varken eksik registry DB ve tam DB varken eksik dosya.

Fixture'lar **gerçek model release iddiası taşımaz**; sentetik ve küçük tutulur. İntegrasyon testinde eski Speech smoke checkpoint kullanılabilir fakat mevcut testlerin doğrulayamadığı dinleme kalitesi sonuç olarak yazılmaz.

---

## 28. Test matrisi ve CI kalite kapıları

| Test ID | Durum | Beklenen davranış |
|---|---|---|
| MR-T01 | Manifest şeması doğru | Parse + normalize + deterministic digest |
| MR-T02 | Duplicate JSON key/NaN/oversize | Şema reddi |
| MR-T03 | Aynı release ID farklı bytes | Collision, overwrite yok |
| MR-T04 | Aynı idempotency key tekrar | Tek sonuç, tek event dizisi |
| MR-T05 | SHA-256 mismatch | Quarantine; release sealed değil |
| MR-T06 | Size mismatch / dosya yok | Doğrulama başarısız |
| MR-T07 | Relative path traversal | Paket kökü dışına erişim yok |
| MR-T08 | Windows junction/UNC path | Bypass yok; uygunsuzsa fail-closed |
| MR-T09 | Codec Q/K aynı, identity farklı | `CODEC_IDENTITY_MISMATCH` |
| MR-T10 | Special/control token ID farklı | `CONTROL_TOKEN_MISMATCH` |
| MR-T11 | Checkpoint state_dict keys/shape bozuk | Loader reddi |
| MR-T12 | Unknown runtime capability | Seçim/kurulum reddi |
| MR-T13 | Experimental rights pending | Public promotion yok |
| MR-T14 | Evaluation eksik/başka checkpoint | Gate bloke |
| MR-T15 | Başarılı otomatik gate ama onay yok | Approval required |
| MR-T16 | Crash: staging'de kesildi | Release görünür değil, cleanup mümkün |
| MR-T17 | Crash: rename sonrası DB öncesi | Recovery orphan reconcile |
| MR-T18 | Eşzamanlı channel promote | CAS/transaction doğru |
| MR-T19 | Live yeni release OOM | Önceki aktif release korunur |
| MR-T20 | Revoked release yüklenmeye çalışıldı | Yeni oturuma izin yok |
| MR-T21 | Quantization sonrası farklı bytes | Yeni digest/variant şart |
| MR-T22 | Drive partial transfer | `verified=false`, final ad kullanılmaz |
| MR-T23 | Trusted/unsigned policy | Sahte publisher kabul edilmez |
| MR-T24 | Disk quota doldu | Güvenli iptal, artifact kaybı yok |
| MR-T25 | GC referenced checkpoint | Silinmez; dry-run korunur |
| MR-T26 | Metric kanıtı farklı codec'e ait | Promotion gate reddi |
| MR-T27 | 2 GB etiketi ölçüm yok | `DEVICE_PROFILE_UNVERIFIED` |
| MR-T28 | Release manifest içeriği değiştirilmiş | Detached digest uyuşmazlığı |
| MR-T29 | Model card ve lisans ilişkisi | Eksik alan dağıtımı bloke |
| MR-T30 | Rollback öncesi release revoked | Geçerli başka release aranır, kör rollback yok |

### 28.1 CI katmanları

1. **Pull request:** Ruff, Pyright/Mypy (belirlenen ayar), pytest unit, JSON Schema, fixture validation, CLI `--help` smoke, format ve migration testleri.
2. **Integration:** Küçük sentetik artifact üret → register → verify → candidate → approve → sealed release → channel → local simulated Live loader; red senaryoları dahil.
3. **Windows özel:** Path escaping, file lock, `os.replace`, junction, File Explorer dosya izinleri; Linux testinin yerine geçmez.
4. **Speech contract:** Gerçek Speech checkpoint formatı ve codec identity validator'ı pin'lenmiş sürümle denenir.
5. **GPU/evaluation:** Donanıma bağlı performans koşuları ayrı pipeline'dır; GPU yoksa geçmediği hâlde başarılı gösterilmez. `skipped_due_to_no_gpu` açık raporlanır.
6. **Release workflow:** İmza/kanıt/approvals geçilmeden GitHub Release veya external model hub'a yükleme yok; public paylaşım ayrıca kullanıcı yetkisi ister.

**Önemli:** GitHub Actions'ın başarılı olması gerçek modelin iyi konuştuğunu kanıtlamaz. CPU CI için küçük sentetik model/codec fixture'ı, ses kalite araştırması için ayrı kanıt gerekir.

---

## 29. Observability, audit ve kullanıcıya raporlanan durum

Her registry eylemi structured event üretir: `event_id`, `correlation_id`, `entity_id`, `actor`, `action`, `previous_state`, `new_state`, `reason`, `timestamp_utc`, `artifact_digest`, `policy_revision` ve gerekiyorsa `error_code`. Eylemler idempotent olacak, tekrar olayları yeni başarı olarak sayılmayacak. Token, şifre ve kişisel ses verisi log'a yazılmayacak.

Metrikler örnekleri: candidate verification success/failure, hash throughput, transfer retry count, promotion block reason, release load failure count, rollback count, storage bytes by class, orphan staging count. Bu **operasyonel metrikler**, Speech modelinin bilimsel evaluation metrikleriyle aynı registry'de anlamsızca karıştırılmaz. `EXPERIMENTS.md` ölçüm kataloğunda tanımlanan metrik isimleriyle semantik çakışma varsa sürümlü mapping kurulur.

### 29.1 Hata raporunun örnek yapısı

```json
{
  "operation": "release.verify",
  "release_id": "example-release-001",
  "status": "blocked",
  "error_code": "CODEC_IDENTITY_MISMATCH",
  "message": "Modelin beklediği codec ile paketteki codec farklı.",
  "safe_action": "Uyumlu codec sürümünü seçin veya yeni model release oluşturun.",
  "correlation_id": "example-correlation-001"
}
```

Bu örnek hata yapısını gösterir; Forge'da bugün çalışan `release.verify` API'si değildir. Teknik log ayrı; kullanıcıya sunulan hata açıklaması sırları sızdırmaz.

---

## 30. Operasyon el kitabı: kayıt, doğrulama, onay, yayın, geri alma

### 30.1 Bir checkpoint'i candidate'a almak

1. Training pipeline'dan run/attempt ve checkpoint path referansını al.
2. Checkpoint yazımının tamamlandığını ve istikrarlı byte'lara sahip olduğunu doğrula.
3. SHA-256, size, source commit ve training recipe snapshot'ını kaydet.
4. Codec ID/artifact hash, token layout, model config ve test edilme iddialarını çözümle.
5. `model_card` ve lisans kanıtı için istenen dokümanları belirle.
6. Eksik alanları `pending` olarak kaydet; sahte varsayılanlar doldurma.
7. Candidate snapshot'ını dondur, `candidate_id` ver ve verification kuyruğuna al.

### 30.2 Verification işlem sırası

1. Manifest parse/şema/versiyon doğrulaması.
2. Tüm artifact'lerin root containment, size ve hash doğrulaması.
3. Training checkpoint'in güvenilen kaynaktan geldiği denetimi.
4. Speech adapter ile model config/codec/token protocol doğrulaması.
5. İnference export smoke ve varsa kısa codec roundtrip.
6. Evaluation raporlarının ilgili artifact digest ve protokolüyle eşleşmesi.
7. Haklar ve model card tamamlanma durumu.
8. Hedef release kanalı için eksik/kritik kapı raporu.

### 30.3 Seal/publish

Güvenli candidate ve insan onayı sağlandıktan sonra export paketinin değişmez artifact listesi hazırlanır. Staging'de tüm dosyalar kontrol edilir. Final `release.json` + detached `release.lock` oluşturulur; release kimliği/version collision kontrol edilir; dosya taşıma ve DB transaction tamamlama adımları audit'e yazılır. Kanala otomatik yönlendirme ayrı, isteğe bağlı açık adım olmalıdır. Public dosya yüklemesi ayrı izin eylemidir.

### 30.4 Geri alma ve bozuk sürüm

Bozuk release doğrulanamazsa onu tamir etmek için aynı `release_id`'nin içeriğini değiştirme. Kaynak kanıtları sakla, quarantine et, nedeni raporla ve son sağlam sürümü güvenli biçimde seç. Kayda yeni release oluştur; eski pointer, yeni pointer ve rollback gerekçesi audit'te korunsun.

---

## 31. Geliştirme planı: MR0–MR12

**Her milestone tek başına test edilebilir küçük PR olmalı.** Forge'da şu anda yalnız başlangıç README'si olduğu unutulmamalı; tüm registry bir defada yazılmayacak.

| Aşama | Somut çıktı | Kanıt / kabul kriteri |
|---|---|---|
| **MR0** | `kaiwen.model.release` kanonik şema kararı ve ADR | Workspace/Speech/Live örnekleri uyumlu; golden fixture test |
| **MR1** | Python paket iskeleti, typed models, config | `uv sync`, lint, typecheck, CPU CI çalışır |
| **MR2** | Manifest parser, canonical JSON ve hash | Geçerli/geçersiz fixture matrisi yeşil |
| **MR3** | Yerel blob store, SQLite migrations | Dosya integrity, crash/restart, cleanup test |
| **MR4** | Candidate kayıt ve idempotence | Aynı request tek kayıt; collision reddi |
| **MR5** | Speech adapter, codec/model compatibility | Mimi baseline fixture ve kasıtlı mismatch test |
| **MR6** | Evaluation/rights/model card gate | Eksik kanıt promotion'ı engeller |
| **MR7** | Human approval ve audit | Kaydedilmemiş onayla yayın yok |
| **MR8** | Immutable release seal + detached digest | Crash-safe publish; overwrite reddi |
| **MR9** | Channel pointer, deprecate/revoke/rollback | Transaction, CAS ve güvenli fallback test |
| **MR10** | Live test adapter ve staged load contract | Kötü hash, codec mismatch, OOM failure test |
| **MR11** | Colab/Drive artifact import (gerektiğinde) | Partial/resume/hash ve origin doğrulama |
| **MR12** | Release readiness, kullanıcı kılavuzu, CI | Bağımsız demo + audit raporu + açık sınırlamalar |

**Minimum kullanışlı V1 dilimi:** MR0–MR5 sonunda bir gerçek Speech araştırma checkpoint'i güvenilir biçimde kayıt/inspect/verify edilebilmeli. MR6–MR9 tamamlanmadan otomatik veya manuel `stable` yayın devreye alınmaz. Live kontratı MR10 aşamasında, Live UI hazır olduğunda entegre edilir; model kalitesi araştırması yayın altyapısını sessizce bloke etmemeli ama kalite başarısı uydurulmamalıdır.

### 31.1 MR0 için zorunlu karar listesi

- `INTEGRATION.md`: `kaiwen.model.release` envelope önermiştir.
- Speech `API.md`: tarihsel `kaiwen.speech.release` taslağını artık kanonik saymaz; `kaiwen.model.release` için Forge şemasına yönlendirir.
- Bu belge `kaiwen.model.release` kanonik envelope önerir; codec/model alt sözleşmelerinin sahibi Speech kalır.
- Hash ve canonical JSON kuralı, token protocol identity, external codec lisans/paketleme politikası, signature/trust kökü ve CLI public isimleri ADR/contract test olmadan kesinleşmiş kabul edilmez.
- **Farklı belgeler arasındaki çatışma çözülene kadar** “tam entegre registry v1” ilan edilmez.

---

## 32. Definition of Ready ve Definition of Done

**Definition of Ready (uygulamaya başlamadan):** Hangi model/artifact türü kaydedilecek, güven kaynağı, codec identity, şema sürümü, en az bir gerçek Speech checkpoint fixture'ı, haklar/kullanım kapsamı, hedef cihaz, başarısızlık senaryoları, test planı ve başka repoya dokunma gereksinimi belirlenmiş olmalı. Forge ile Speech arasında yeniden trainer yazılmasını gerektiren bir tasarım kabul edilmez.

**Definition of Done (özellik gerçekten tamamlandı):**

1. Kod doğru repoda, modüler ve tipli.
2. İlgili şema ve public API sürümü belgelendi, fixture/contract test var.
3. Başarılı senaryo kadar bozuk hash, yanlış codec ve eksik izin senaryoları test edildi.
4. Idempotence, crash recovery ve Windows/Linux path davranışı ölçüldü veya eksik açıkça belirtildi.
5. Gerçek CI komutları ve sonuçları raporlandı; çalıştırılmayan testler “geçti” denmedi.
6. Güvenlik/personal data/third-party license policy geçti; gereksiz veri yayını yok.
7. Performance/VRAM iddiaları gerçek ölçümle ilişkilendirildi; kanıt yoksa destek listesinde bulunmuyor.
8. Release ve channel pointer geri alma yolu test edildi; önceki artifact'ler korunuyor.
9. README, AGENTS, ilgili Forge/Speech/Live belgeleri ve varsa ADR aynı sürümle tutarlı.
10. Codex sonuç raporu değişen dosyaları, komutları, test kanıtlarını, riskleri, çözülmeyen eksikleri ve kullanıcı onayı gerektiren işleri içeriyor.

Belgenin yazılmış olması bu maddelerin **uygulandığı anlamına gelmez**. Bu dosya MR0–MR12 için mühendislik kabul sözleşmesidir.

---

## 33. Sık sorulan mimari kararlar

**Neden MLflow gibi mevcut bir experiment/registry ürünüyle başlamıyoruz?** Forge'un hedefi yerel, hafif, kendi Speech araştırmasıyla entegrasyonu güçlü bir temel oluşturmaktır. Mevcut açık kaynak çözümleri teknik olarak değerlendirebiliriz; ancak dış servis bağımlılığı, local-first kullanım, bakım ve lisans gereksinimleri doğrulanmadan yeni bir büyük sistemi zorunlu kılmayız. Tekrarlayan özellikler kanıtlanmış maliyet yaratırsa adaptör seçeneği açılır. Araç seçimi araştırma sonucu olmalıdır; marka tercihi değil.

**Model ağırlıkları GitHub reposuna konmalı mı?** Büyük ağırlık ve kişisel ses verilerini doğrudan Git geçmişine yazmak genellikle kötü fikirdir. Ayrı artifact depolama, release asset veya içerik adresli store kullanılabilir; kullanıcının paylaşım izni ve dosya/lisans sınırları önce doğrulanır. GitHub Actions ile public release otomatik yapılıyorsa ayrı onay gerekir.

**Mimi ile yeni Kaiwen Codec aynı registry'ye girebilir mi?** Evet, farklı codec kimliği ve ayrı release olarak; ancak birbirlerinin tokenları/weights'i doğrudan paylaşılabilir sanılmamalıdır. Aynı `model_id` altında yeni model varyantı teknik olarak mümkün olsa da kimlik ve sürüm açıklığı korunmalıdır.

**Yeni codec henüz yokken registry geliştirilebilir mi?** Evet. Mevcut Speech smoke checkpoint'i, frozen Mimi adaptörü ve sentetik küçük fixture'larla integrity ve compatibility sistemi geliştirilebilir. Tam konuşma kalitesi iddiaları ise gerçek model/evaluation çıkana kadar `not_measured` olur.

**Bir release neden `verified` ama `stable` değil?** Doğrulama dosya bütünlüğü ve teknik uyumu kanıtlayabilir. Türkçe konuşma kalitesi, telif hakkı, cihaz uygunluğu veya insan onayı eksik olabilir. Bu ayrım mimari gerekliliktir; hata değildir.

**Model kesintisiz değiştirilmek zorunda mı?** Hayır. 2 GB VRAM sınıfında aynı anda iki modeli bellekte tutmak mümkün olmayabilir. Güvenli stop–unload–load–verify–resume akışı, sessiz hot-swap'tan daha doğru başlangıçtır.

**Model release ile Forge paket release aynı mı?** Hayır. `kaiwen-forge` Python kütüphanesi `v0.x` olarak yayınlanabilir; onun yönettiği Speech model release'leri farklı cadence ve kimlik taşır. Birine otomatik semver artışı diğerini değiştirmez.

**Tek belge her şeyi kapsar mı?** Hayır. Registry, sahip olduğu release ve provenance semantiğini ayrıntılı tanımlar; `EVALUATION.md` değerlendirmeyi, `EXPERIMENTS.md` araştırma karşılaştırmasını, `TRAINING_PIPELINE.md` iş yürütümünü, Speech `API.md` runtime yüklemeyi ve Workspace `INTEGRATION.md` repo sınırlarını tanımlar. Bir sözleşmenin iki farklı kopyası oluştuğunda birini kanonik kabul edip diğerini referansa dönüştürmek gerekir.

---

## 34. Son denetim listesi ve proje yönünü koruma

Bir Forge registry geliştirmesi tamamlandıktan sonra şu sorular cevaplanabiliyor olmalı:

- Her model/codec dosyasının gerçek SHA-256 ve boyutu var mı?
- Candidate ve release kimlikleri çakışmadan ve overwrite etmeden yönetiliyor mu?
- Aynı release farklı codec veya special token mapping ile çalıştırılmaya kalkınca **reddediliyor mu**?
- Eğitim checkpoint'i ile inference paketinin farkı açık mı?
- Evaluation kanıtları doğru checkpoint ve holdout'a bağlı mı?
- Test edilmiş cihazlar ile varsayılan destek iddiaları ayrılmış mı?
- Dataset/lisans/kişisel ses hakları public dağıtımı gerçekten karşılıyor mu?
- Bir kişi onay vermeden candidate `stable` olabiliyor mu? **Olmamalı.**
- Drive aktarımı yarıda kesilirse eski sağlam artifact kayboluyor mu? **Kaybolmamalı.**
- Live bozuk release yüklerse önceki modelle devam edebiliyor mu?
- Revoked release yeni oturumlara sessizce sunuluyor mu? **Sunulmamalı.**
- Kullanılmayan dosyalar silinmeden önce reachability ve retention kontrolü var mı?
- `kaiwen.model.release` doküman hizalamasına ek olarak eski `kaiwen.speech.release` geçişi **kodda** ADR, fixture ve consumer testleriyle çözüldü mü?
- Gerçek test çıktılarıyla yazılan iddialar arasında fark var mı?
- Forge yalnız orkestrasyon/katalog rolünde mi, Speech model kodunu kopyalamıyor mu?

**Nihai ilke:** Kaiwen Forge Model Registry'nin değeri, depoladığı kaç model olduğu değil; *her sürümün nereden geldiğini, neyi kanıtladığını, hangi koşulda çalışabildiğini ve bir sorun olduğunda nasıl güvenle geri alınacağını* gösterebilmesidir. Bu güven zinciri kurulmadan “model yönetim sistemi hazır” denmeyecek.
