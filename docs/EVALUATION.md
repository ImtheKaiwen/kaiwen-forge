# Kaiwen Forge — Evaluation Engine, Speech Benchmarks ve Bilimsel Doğrulama Şartnamesi

> **Belge türü:** Mimari karar, değerlendirme protokolü ve uygulama sözleşmesi  
> **Hedef konum:** `kaiwen-ai/kaiwen-forge/docs/EVALUATION.md`  
> **Durum:** Tasarım / uygulama şartnamesi; Forge Evaluation Engine henüz uygulanmış değildir.  
> **Sahip:** Kaiwen Forge; modele özgü metrik hesabı ve adapter sınırları için Kaiwen Speech ile sözleşmeli.  
> **Temel referanslar:** [Forge README](../README.md), [Training Pipeline](TRAINING_PIPELINE.md), [Experiments](EXPERIMENTS.md), [Speech Performance](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/PERFORMANCE.md), [Speech Codec](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/CODEC.md), [Speech Model](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/SPEECH_MODEL.md), [Integration](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/INTEGRATION.md), [Engineering Standards](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ENGINEERING_STANDARDS.md).

---

## 1. Amaç: bir modeli değil, bir iddiayı sınamak

Kaiwen Forge Evaluation Engine'in görevi, bir model çalıştırıldığında ortaya çıkan sayıları toplamakla sınırlı değildir. Görevi; **hangi araştırma sorusunun hangi veri, yöntem ve ölçümle sınandığını**, ölçümün hangi koşullarda geçerli olduğunu, sonucunun tekrarlanabilirliğini ve sonucun bir ürün iddiasını destekleyip desteklemediğini kanıt zinciriyle göstermektir.

Kaiwen Speech'in ana hedefi Türkçe konuşmayı doğrudan ses temsilleri üzerinde işleyen, düşük gecikmeli ve düşük kaynak tüketimli bir sistem geliştirmektir. Bu nedenle tek bir `loss`, `accuracy`, `WER`, `MOS` veya `forward_ms` puanını projenin toplam başarısı olarak kabul etmeyiz. Bir codec sesin anlaşılabilirliğini koruyabilir ama prozodiyi bozabilir. Bir Transformer token tahmininde iyi olabilir ama kendi ürettiği konuşmayı devam ettiremeyebilir. Bir model düşük RTF elde edebilir ancak ilk sesi çok geç verebilir. Başarılı bir çalışma bu ayrımları görünür kılar.

Belge üç katmanlıdır:

1. **Mevcut durum:** GitHub'da gerçekten bulunan kod, CLI, test ve veri tipleri.
2. **V1 tasarım sözleşmesi:** İlk kullanılabilir Evaluation Engine'in davranışları, API'leri, dosya biçimleri ve testleri.
3. **Araştırma uzantıları:** Gelişmiş insan değerlendirmesi, gerçek zamanlı konuşma, speaker/overlap/agent iş akışları ve istatistiksel karşılaştırmalar.

V1 tamamlanana kadar örnek kodlar **taslak** olarak okunur. Bu belgede bir CLI örneğinin yer alması, ilgili CLI'ın bugün mevcut olduğu anlamına gelmez.

### 1.1 Normatif kavramlar

| Etiket | Anlamı |
|---|---|
| `IMPLEMENTED` | İncelenen kaynak kodda bulunan davranış; canlı koşulda ayrıca doğrulanmalıdır. |
| `TARGET_V1` | İlk mühendislik iterasyonunda uygulanması istenen sözleşme. |
| `EXPERIMENTAL` | Başarısı kanıtlanmamış araştırma seçeneği. |
| `DEFERRED` | Şimdilik geliştirme kapsamı dışında. |
| `BLOCKED` | Eksik veri, yetki veya uyumluluk nedeniyle sonuç üretilemez. |
| `INVALID` | Ölçüm oluşturulmuş olsa bile geçersiz/savunulamaz. |

**Zorunlu kural:** Forge, üretmediği metriği `0` olarak yazmayacak; üretemediği sonuç için `not_measured` durumunu ve nedenini kaydedecektir. Bilinmeyen sonuçlar başarılı kabul edilmez.

## 2. Kapsam ve repository sorumlulukları

### 2.1 Forge'un sahibi olduğu işler

- Değerlendirme protokolünü çözümlemek, doğrulamak, kilitlemek ve revizyonlamak.
- Veri seti, referans ses, checkpoint, codec ve evaluator sürümlerini doğrulamak.
- Görevi uygun worker'a ve modele özgü adapter'a yönlendirmek.
- Örnek düzeyinde çıktı, metrik, hata ve kaynak kullanımını kalıcı kaydetmek.
- Metrik tanımlarını, birimleri, aggregation ve karşılaştırılabilirlik kurallarını yönetmek.
- İnsan dinleme çalışmasının körleştirme, randomizasyon ve sonuç içe aktarımını yürütmek.
- Confidence interval, paired comparison ve istatistiksel uyarıları hesaplamak.
- Model release kararına veri sağlayacak raporu üretmek; otomatik model release yetkisi kullanmamak.

### 2.2 Speech'in sahibi olduğu işler

- Codec encode/decode ve gerçek model inference fonksiyonları.
- `KaiwenSpeechTransformer`, eski `MimiCodec`, yeni `KaiwenCodec` ve ses token mantığı.
- Teacher-forced model skorları, codebook masking ve gecikmeli akışın doğru hizalanması.
- Speech modeline özgü continuation/dialogue inference ve codec özel test fixture'ları.
- Modelin streaming destekleyip desteklemediğini belirten gerçek capability kontratı.

### 2.3 Corpus'un sahibi olduğu işler

- Verinin kaynağı, lisansı, izinleri, konuşmacı kimlikleri ve onay durumu.
- Ham/işlenmiş ses, segment, referans transcript ve değerlendirici anotasyonları.
- Kaynak/konuşmacı bazlı train/validation/test ayrımları ve veri sızıntısı kontrolleri.
- Holdout değerlendirme setinin sürümlü ve değişmez export'u.

### 2.4 Live ve Agent sınırı

- **Live:** Mikrofon, hoparlör, gerçek cihaz IPC, audible TTFA, playback underrun, barge-in ve oturum UX telemetrisi.
- **Agent:** Tool çağrıları, izinler, yanıt doğruluğu, dış kaynağın güvenilirliği; Speech çekirdeğinin içine gömülmez.
- **Design:** Evaluation raporunun veya gelecekteki Forge UI'sının yeniden kullanılabilir görsel bileşenleri; model metriği hesaplamaz.

Forge çekirdeği; PyTorch model katmanını, ses codec implementasyonunu, Electron UI kodunu veya Agent business logic'ini kendisine kopyalamaz. Model adapter'ı, Speech'in sağladığı mevcut veya sürümlenmiş public API'yi çağırır.

## 3. Bugünkü gerçek kod envanteri

`ImtheKaiwen/kaiwen-forge` başlangıçta yalnızca kısa bir `README.md` içeriyor. Python paketi, Evaluation Engine, model registry, job scheduler veya CLI'ın var olduğu varsayılmayacaktır. Bu dosya yeni bir hedef sözleşmedir.

`ImtheKaiwen/kaiwen-speech` içinde doğrulanan mevcut öğeler:

| Dosya / komut | Gerçekte yaptığı | Başlıca sınırlaması |
|---|---|---|
| `scripts/evaluate_codebooks.py` | Token cache ve checkpoint üzerinden codebook başına CE/PPL/Top-1 ve tam frame doğruluğu | Temel olarak continuation/teacher-forcing analizi; konuşma doğallığı veya diyalog başarısı değil |
| `scripts/evaluate_continuation_audio.py` | Belirli continuation örneği için ground-truth ve çeşitli tahmin WAV dosyalarını üretir | İnsan dinleme skorunu veya kalıcı eval manifestini hesaplamaz |
| `scripts/benchmark.py` | Bir modelin parametre sayısını ve tek forward çağrısının süresini ölçer | TTFA, RTF, bütün ses hattı veya uzun oturum ölçümü değil |
| `scripts/infer_audio.py` | Bir checkpoint + Mimi ile turn-based sesli çıktı üretir | Checkpoint gerekir; gerçek streaming ve kalite garantisi yok |
| `src/kaiwen_voice/training/checkpoint.py` | Model config, step, optimizer ve belirli metadata'yı kaydeder/yükler | Evaluator sürümü, holdout protokolü ve tam veri lineage'i otomatik kurulmaz |
| `tests/test_*` | Kodun belirli davranışlarına yönelik testler | Bilimsel değerlendirme sonucu veya gerçek Türkçe kalite kanıtı değildir |

Mevcut Python dağıtımı `kaiwen-voice`, import kökü `kaiwen_voice` olarak kalmıştır. Repo adı `kaiwen-speech` olsa da bu import adı kendiliğinden değişmez. Önerilen `kaiwen_forge` API'leri ise gelecekte uygulanacak arayüzlerdir.

### 3.1 İlk uygulanabilir dikey dilim

1. Forge bağımsız bir **değerlendirme planı** kabul eder.
2. Bu plan mevcut Speech CLI'larından birini sürümlenmiş adapter ile başlatır.
3. Exit code ve beklenen çıktı dosyalarını doğrular.
4. Ham stdout/stderr ve üretilen metrikleri, yorum katmanını ayırarak saklar.
5. Tek bir model için `evaluation.json`, `samples.jsonl`, `metrics.jsonl` ve insan-okur `report.md` üretir.
6. Aynı fixture ile iki kez çalıştığında kararlı kimlikler ve doğru hash'lerle karşılaştırılabilir kayıtlar üretir.

Bu küçük dilim başarıyla tamamlanmadan genel amaçlı GPU scheduler, web dashboard, dağıtık evaluator kümesi veya otomatik leaderboard geliştirilmez.

## 4. Değerlendirme türleri ve kanıt seviyeleri

| Tür | Temel soru | Referans | Kanıt seviyesinin sınırı |
|---|---|---|---|
| `unit_fixture` | Metrik hesaplayıcı doğru mu? | Bilinen sentetik sayılar | Sadece evaluator doğruluğu |
| `token_teacher_forced` | Gerçek tokenın olasılığı nedir? | Etiketli token cache | İstatistiksel tahmin; konuşma kalitesi değil |
| `codec_roundtrip` | Codec sesi ne kadar koruyor? | Aynı referans PCM | Codec rekonstrüksiyon kalitesi |
| `continuation_free_running` | Model bağlamı akıcı sürdürüyor mu? | Bağımsız held-out ses | Konuşma devamlılığına dair kanıt |
| `dialogue_response` | Kullanıcı sesine uygun yanıt veriyor mu? | Türkçe konuşma senaryoları | Görev ve insan değerlendirmesi gerekir |
| `streaming_runtime` | Ses parçaları zamanında üretiliyor mu? | Kontrollü ses kaydı | Runtime; hoparlör etkisi dışarıda |
| `live_e2e` | İnsan ilk sesi ne zaman duyuyor? | Gerçek/simüle mikrofon–hoparlör | Ürün düzeyi uçtan uca kanıt |
| `human_blind` | Dinleyiciler hangisini tercih ediyor? | Randomize kör ses çiftleri | Seçilen katılımcılar ve sorularla sınırlı |
| `agent_handoff` | Yardımcı araca güvenli geçiliyor mu? | Onaylı senaryolar | Speech başarısına otomatik eşit değil |

Her evaluation çalışması **tek bir birincil amaç** belirtir. Örneğin codec roundtrip ve dialogue response ayrı protokollerdir; dosyalar aynı raporda yer alabilir ama tek bir toplama puan zorla birleştirilemez.

### 4.1 Aşamalı kanıt

- `E0`: Metrik algoritmaları ve sahte model fixture'ları doğrulandı.
- `E1`: Mevcut gerçek Speech checkpoint'i veya codec ile tekrarlanabilir örnekler üretildi.
- `E2`: Bağımsız holdout veri, ön kayıtlı metric policy ve kıyaslama mevcut.
- `E3`: İnsan dinleme veya gerçek diyalog senaryolarıyla tamamlayıcı kanıt var.
- `E4`: Seçilmiş gerçek donanımda uçtan uca performans, uzun oturum ve bellek doğrulandı.

E0 başarıyla tamamlanmış bir sistem E4 ürün hedefini gerçekleştirmiş sayılmaz.

## 5. Kavramsal mimari

```text
Corpus versioned eval export
            |
            v
     Dataset Resolver -----> Rights / leakage / consent gate
            |
            v
   Evaluation Protocol
            |
            v
     Preflight Validator
          /       \
         v         v
   Artifact       Capability & Environment Check
   Verifier                  |
         \                   /
          v                 v
          Evaluation Scheduler (Forge)
                       |
                Speech Adapter
                       |
         +-------------+-------------+
         |             |             |
     Token Eval    Codec Eval    Speech / Streaming Eval
         |             |             |
         +-------------+-------------+
                       |
              Sample-Level Results
                       |
             Metric Definitions
                       |
              Aggregation / CI
                       |
       Evaluation Manifest + Report
                       |
         Experiments / Registry Gate
```

`Evaluation Scheduler` ilk sürümde bir subprocess ve SQLite kaydından ibaret olabilir. Görsel şema dağıtık servis zorunluluğu getirmez.

### 5.1 Temel bileşenler — hedef

- `protocols`: plan parse, immutable snapshot, şema sürümü, kapı kuralları.
- `datasets`: güvenilir manifest resolver, split ve source-group doğrulama.
- `adapters/speech`: mevcut CLI entegrasyonu ve daha sonra public Speech SDK adaptörü.
- `evaluators`: codec, token, dialogue, streaming gibi görev-specific evaluator'lar.
- `metrics`: isimli tanımlar, doğrulama, örnek başına hesaplama, agregasyon.
- `human`: blind trial manifesti, response import, tercihler.
- `stats`: paired bootstrap, uncertainty, effect size, cluster aggregation.
- `storage`: SQLite metadata, immutable JSONL, checksum, yerel artifact.
- `reporting`: Markdown, JSON, karşılaştırma tablosu ve ham veri referansı.
- `cli`: preflight, run, verify, compare, report komut yüzeyi.

Bu modüller yalnızca ihtiyaç ortaya çıktıkça eklenir; hepsi baştan ayrı plugin framework'ü olmaz.

## 6. Evaluator arayüzü: küçük, test edilebilir, sürümlü

`TARGET_V1` temel sözleşme: evaluator girdi olarak sabit bağlam, bir örnek ve gerekli model adapter'ını alır; metrik kayıtları ve artifact referanslarını döndürür. Orkestratör dosyaya yazma, zaman aşımı ve süreci yönetir. Evaluator; SQLite bağlantısı açıp kalıcı durum mutasyonu yapmaz.

```python
# HEDEF ÖRNEK / ŞU ANDA IMPORT EDİLEBİLEN KOD DEĞİLDİR.
from dataclasses import dataclass
from typing import Protocol, Mapping, Any

@dataclass(frozen=True)
class EvalSample:
    sample_id: str
    audio_uri: str | None
    reference_text: str | None
    source_group_id: str
    metadata: Mapping[str, Any]

@dataclass(frozen=True)
class MetricValue:
    name: str
    value: float | None
    unit: str
    status: str
    reason: str | None = None

@dataclass(frozen=True)
class SampleResult:
    sample_id: str
    status: str
    metrics: tuple[MetricValue, ...]
    artifact_uris: tuple[str, ...]
    error_code: str | None = None

class Evaluator(Protocol):
    evaluator_id: str
    evaluator_version: str

    def evaluate_sample(self, sample: EvalSample) -> SampleResult:
        ...
```

**API kuralı:** Bu, ilk hedef yüzeyin anlaşılması için hazırlanmış örnektir. Gerçek implementation sırasında asyncio, batch API veya resource context ihtiyacı varsa ADR ile değiştirilebilir; dokümansız paralel iki public API oluşturulamaz. `source_group_id` boş olan örnekler kör bilimsel karşılaştırmaya sokulmaz.

### 6.1 Evaluator kimliği

Her evaluator aşağıdakileri bildirir:

- `evaluator_id`, `version`, `implementation_commit`, `implementation_digest`.
- Desteklediği `task`, `input_schema_version`, `output_schema_version`.
- Gerekli model capability'leri: `codec_decode`, `token_logits`, `turn_inference`, `streaming_session`.
- Gerekli veri alanları: `reference_audio`, `reference_transcript`, `speaker_group`, `turn_annotations`.
- Minimum/maximum örnek uzunluğu, beklenen sample rate, channels.
- Determinism garantisi ve zorunlu seed policy.
- İzin verilen audio preprocessing adımları.
- Dış modele gereksinim: `ASR_MODEL`, `SPEECH_EMBEDDER`, vb.; erişilemiyorsa `not_measured`.

Evaluator kimliği değişmeden metric formülü değiştirilemez. Formül revizyonu evaluator versiyonu veya metric definition version yükseltir.

### 6.2 Resource lifecycle

- `prepare()` / `teardown()` benzeri kaynak yönetimi tek worker tarafından sahiplenilir.
- Her değerlendirme örneğinde yeniden model yüklenmez; load maliyeti ayrı benchmark profilidir.
- Hata sonrası GPU belleğinin/decoder state'in temizliği testlenir.
- Bir örnekteki hata bütün örnekleri otomatik geçersiz kılmaz; `fail_fast` politikası protokolde açıktır.
- Async görevler ve subprocess'ler iptal edildiğinde yetim worker, kilit veya dosya bırakmamalıdır.
- Değerlendirme sampling'i deterministikse seed örnek ID ve run ID'den türetilebilir; formül sürümlenir.

## 7. Evaluation protocol: veri ve yöntem sabitliği

Protokol değerlendirme başlamadan kaydedilir ve kilitlenir. Model, dataset veya birincil metrik değişirse aynı koşunun üstüne yazılmaz; yeni evaluation revision açılır.

```yaml
# HEDEF evaluation recipe: mevcut Forge CLI bu dosyayı henüz okumaz.
schema_version: 1
evaluation_id: example-eval-001
kind: codec_roundtrip
protocol_revision: 1
status: preregistered
model_ref:
  run_id: example-run-001
  checkpoint_digest: REPLACE_WITH_ACTUAL_SHA256
codec_ref:
  codec_id: kaiwen-codec-candidate
  weights_digest: REPLACE_WITH_ACTUAL_SHA256
dataset:
  manifest_uri: file:///REPLACE_WITH_VERSIONED_DATASET/dataset.lock
  manifest_sha256: REPLACE_WITH_ACTUAL_SHA256
  split: test
  source_group_policy: strict
sampling:
  seed: 42
  max_samples: 100
  selection: stable_hash
preprocessing:
  expected_sample_rate: 24000
  channels: 1
  normalizer_id: none
metrics:
  primary: eval.codec.stft_distance
  additional:
    - eval.codec.si_sdr_db
resources:
  device: cpu
  timeout_per_sample_seconds: 120
failure_policy:
  max_failed_fraction: 0.05
  missing_primary_metric: fail
report:
  output_dir: artifacts/evaluations/example-eval-001
```

Örnekteki `REPLACE_WITH_ACTUAL_SHA256` yer tutucudur; gerçek preflight işleminde reddedilir. `max_samples`, örnek seçimini açıklar fakat küçük subset'i genel corpus sonucuyla karıştırmaya izin vermez. Protocol sha256 normalize edilmiş canonical snapshot üzerinden hesaplanır.

### 7.1 Protokolün asgari alanları

| Alan | Gerekçe | Eksikse davranış |
|---|---|---|
| `evaluation_id` | Artifakt ve sonuç kimliği | Hata |
| `protocol_revision` | Bilimsel yöntem izlenebilirliği | Hata |
| `kind` | Metrik ve evaluator seçimi | Hata |
| `model_ref` / `codec_ref` | Tam ağırlık kimliği | İlgili türde hata |
| `dataset.manifest_sha256` | Veri değişmezliği | Hata |
| `split` | Holdout ve leakage kapısı | Hata |
| `sampling` | Alt örneklem seçimi | Hata |
| `preprocessing` | Ses ve metin karşılaştırma adaleti | Hata |
| `metric definitions` | Metrik anlamı | Hata |
| `environment` | Performance karşılaştırması | Gerektiğinde hata |
| `failure_policy` | Kaybolan örnekler üzerinden yanlılık | Hata |

### 7.2 Seçim ve tekrar

`stable_hash(seed, sample_id)` kullanarak seçmek, manifest sırası değiştiğinde örneklemin oynamasını önler. Örnek kimlikleri benzersiz olmak zorundadır. Aynı sesin farklı segment ID'leriyle çoğaltılması veri sızıntısı ve sahte örnek sayısı üretebilir; `audio_sha256`, `recording_id` ve `source_group_id` kontrol edilir.

Birincil test seti, eğitim boyunca sürekli seçme/ayar yapma için kullanılmaz. Geliştirme sırasında ölçülen validation veya development split, nihai holdout'tan ayrı tutulur.

## 8. Dataset sözleşmesi ve provenance kapısı

Corpus export dosyası üzerinde en az şu denetimler yapılır:

1. Manifest schema/version ve digest geçerli.
2. İstenen split mevcut; örnek sayısı ve toplam süre sıfır değil.
3. Girdi örneklerinin referans dosyaları bulunuyor ve beklenen hash'e sahip.
4. Her örneğin lisansı ve kullanım amacı araştırma/dağıtım politikasına uygun.
5. Onay gerektiren insan sesi için izin ve erişim koşulları saklanmış.
6. Split'ler arasında aynı `recording_id`, `speaker_group`, `source_group`, yakın kopya/fingerprint sızıntısı yok.
7. Referans transcript varsa dili, normalizer protokolü ve doğrulama durumu bilinmeli.
8. Açık çevrimde kullanılacak sentezlenmiş sesler `synthetic` diye etiketlenmiş.
9. Audio sample rate, kanal sayısı ve segment offset aralığı tutarlı.
10. Test setine doğrudan model tuning veya eğitim verisi olarak erişilmez.

### 8.1 Türkçe değerlendirme seti bölümleri

Başlangıçtaki önerilen stratifikasyon:

| Alt grup | Ne ölçer? | Zorunlu metadata |
|---|---|---|
| Günlük tek konuşmacı | Kısa ve temiz konuşma | Speaker/source ID, yaşanmayan varsayımlar olmadan kayıt türü |
| Doğal diyalog | Yanıt ve sıra alma | Turn sınırları, roller |
| Sayılar ve özel adlar | Telaffuz/anlam korunması | Doğrulanmış transcript, rubrik |
| Çeşitli aksanlar | Dil çeşitliliği | Kaynak onayı; demografik tahmin icat edilmez |
| Gürültülü ortam | Robustness | Gürültü koşulu ve SNR ölçülmüşse yöntemi |
| Uzun bağlam | Tutarlılık, bellek, sessizlik | Context uzunluğu ve referans konuşma |
| Kesme ve overlap | Barge-in, konuşmacı ayrımı | Overlap/boundary onayları |
| Sentetik diyalog | Bootstrap regresyonu | `synthetic=true`, TTS ve lisans |
| Sessizlik / boş giriş | Hata ve yanlış aktivasyon | Gerçek sessizlik süreleri |

Bu kategorilerin hepsinin bugün elde bulunduğu varsayılmaz. Gold set oluşturulmadan `speaker_accuracy` veya `turn_taking` puanı yayımlanmaz.

### 8.2 16 kHz / 24 kHz kritik sınır

Corpus'un bugünkü bazı indirilen kayıtları ve ASR hattı 16 kHz PCM üretmektedir. Kaiwen Codec araştırmasının başlangıç standardı 24 kHz'dir. **16 kHz kayıt 24 kHz'e yeniden örneklendi diye 24 kHz kaynak kalitesine ulaşmaz.** Değerlendirme referansı mümkün olduğunca gerçek yüksek kaliteli kaynak ses olur. Resampling gereken yerde giriş kaynak frekansı, algoritma, filtre, kanal dönüşümü ve işlem digest'i kaydedilir; iki sistemin aynı işlemden geçmesi sağlanır.

### 8.3 Veri hakları ve gizlilik

Holdout seti herkese açık artifact olarak otomatik paketlenmez. Eğitim verisi lisansı, evaluation seslerini yeniden dağıtma hakkını zorunlu olarak vermez. Varsayılan rapor; kimlikleri pseudonymize edilmiş örnek kimliği, agregasyon ve hak sahibi tarafından onaylanan dinleme kliplerini içerir. Ham ses kişisel veri içerebileceğinden erişim kayıtları ve saklama süreleri tanımlanır.

## 9. Model, codec ve checkpoint uyumluluk kapısı

Değerlendirmeden önce aşağıdakiler kontrol edilir:

- Checkpoint var ve digest eşleşiyor.
- Model config ve layer/head/codebook konfigürasyonu geçerli.
- Codec `sample_rate`, `frame_rate`, `num_codebooks`, `codebook_size` ve ağırlık kimliği biliniyor.
- Codec token ID eşlemesi, cache metadata'sı ve model vocab'u aynı **gerçek codec sürümüne** ait.
- `acoustic_delay_frames`, `codebook_modeling`, special token listesi ve masking formülü uyumlu.
- İstenen evaluator capability'si modelde gerçekten var.
- Model yalnız batch çalışıyorsa streaming sonucu üretilmez.
- Model checkpoint'i optimizer içerebilir; değerlendirme için yalnız güvenilir gerekli ağırlıklar yüklenir.

**Kritik kural:** Mimi 8×2048 token tensoruyla aynı şekli üreten yeni Kaiwen Codec tokenlarının semantiği aynı değildir. Uyumlu boyut, uyumlu embedding veya ses üretimi garantisi vermez. Değerlendirme `codec_weights_digest` ve token schema sürümüne göre eşleştirme yapar.

### 9.1 Checkpoint seçimi

- `latest` eğitimdeki en son durumu belirtir; otomatik olarak en iyi kalite değildir.
- `best` hangi validation metriğine göre seçildiyse o politika korunur.
- Nihai test sonucuna göre farklı checkpoint'leri deneyip en iyisini seçmek, holdout'u model seçimi için kullanmak olur; bilimsel raporda açıkça belirtilmeden yapılamaz.
- Checkpoint `step`, optimizer durumuyla resume, ağırlık değişimi ve token cache provenance'i ayrı kavramlardır.

### 9.2 Güvenli yükleme

Yerel `.pt` dosyası bulunması, dosyanın güvenilir olduğu anlamına gelmez. `torch.load(weights_only=True)` gibi sınırlayıcı seçenekler tek başına sandbox garantisi vermez. Forge hash, dosya kökeni, beklenecek şema, boyut limiti ve yetki sınırı doğrulamasını uygular. Bilinmeyen harici artifact'lar sistemde otomatik kod yürütmeye zorlanmaz.

## 10. Metric definition registry

`EXPERIMENTS.md` içerisinde tanımlanan metric registry burada **tek ortak otorite** kabul edilir. Evaluation ayrı isimlerle ikinci metric kataloğu oluşturmaz; gerekirse metrik tanımı registry'ye sürümlü eklenir.

Her metrik şu alanları taşır: `name`, `version`, `task`, `unit`, `direction`, `range`, `aggregation`, `sampling_unit`, `requires`, `preprocessing_id`, `reference_policy`, `evaluator_id`, `missing_policy`.

### 10.1 Önerilen başlangıç matrisi

| Metric adı | Yön | Birim | Yorum / sınır |
|---|---|---|---|
| `eval.token.cross_entropy` | min | nats/token | Aynı codec/mask/vocab koşulu gerekli |
| `eval.token.top1` | max | fraction | Teacher-forced; serbest konuşma kanıtı değil |
| `eval.token.exact_frame` | max | fraction | Bütün codebooklar doğru; model/frame alignment sabit |
| `eval.codec.stft_distance` | min | tanımlı STFT uzaklığı | FFT, hop, log/linear ve alignment protokolü sabit |
| `eval.codec.si_sdr_db` | max | dB | Referans/çıktı hizalaması kritik; tek başına algısal kalite değil |
| `eval.asr.cer` | min | fraction | ASR ve TR normalizer kimliği şart |
| `eval.asr.wer` | min | fraction | Tokenization/morphology/normalizer politikası şart |
| `eval.speech.pairwise_preference` | max | fraction | Kör ve eşlenmiş insan değerlendirmesi |
| `eval.speech.intelligibility` | max | protokole göre | İnsan transkripsiyonu veya doğrulanmış ölçüm |
| `eval.speech.task_success` | max | fraction | Önceden tanımlı senaryo ve rubrik |
| `eval.turn_taking.success_rate` | max | fraction | Annotated turn boundary gerektirir |
| `eval.speaker.correctness` | max | fraction | Gerçek gold speaker labels şart |
| `perf.ttfa_ms` | min | ms | Runtime ve audible ayrı metric definition |
| `perf.rtf` | min | ratio | Süre/hesaplama sınırı belirlenir |
| `perf.peak_vram_bytes` | min | bytes | Allocator vs physical device ölçümü ayrılır |
| `perf.interruption_stop_ms` | min | ms | Live hoparlör/loopback metrik sınırı |

Ad çakışması veya metrik tanımı değişimi varsa registry migration ile yönetilir. Özellikle `eval.asr.cer` ile `eval.speech.intelligibility` birbiri yerine kullanılamaz.

### 10.2 Metrik değer sözleşmesi

```json
{
  "schema_version": 1,
  "metric_definition": "eval.codec.stft_distance@1",
  "evaluation_id": "example-eval-001",
  "sample_id": "example-record-001",
  "value": 0.187,
  "unit": "spectral_distance",
  "status": "measured",
  "evaluator_id": "codec_stft_v1",
  "reference_artifact_digest": "EXAMPLE_REPLACE_WITH_REAL_SHA256",
  "output_artifact_digest": "EXAMPLE_REPLACE_WITH_REAL_SHA256",
  "preprocessing_digest": "EXAMPLE_REPLACE_WITH_REAL_SHA256"
}
```

Bu değer **tamamen örnektir; gerçek ölçüm değildir**. `value` finite number olmalı; `NaN`, `Infinity`, tırnaklı sayı, sahte `0` ve tanımsız birim reddedilir. `status=not_measured` olduğunda `value=null` ve `reason` gereklidir. Event idempotent yazılır.

## 11. Codec reconstruction değerlendirmesi

Codec testinin temel sorusu, **aynı kaynak sesten encode→decode sonrası ne kadar ve hangi açıdan bilgi kaybedildiği**dir. Bu değerlendirmede modelin dil anlama başarısı değil codec'in ses temsili incelenir.

### 11.1 Kalite boyutları

- Sesin anlaşılabilirliği ve doğal artikülasyon.
- Spektral yapı ve enstrümantal/çevresel ses kaybı.
- Konuşmacı kimliğinin korunması.
- Prosodi: vurgu, ton, ritim, duraklama.
- Sessizlik ve zayıf seslerin korunması.
- Arka plan gürültüsünün hangi yönde değiştiği.
- Başlangıç/son chunk ve araya eklenen yapay sesler.

### 11.2 STFT tabanlı metrikler

Birden fazla pencere/hop kombinasyonu kullanılır. Protokol; sample rate, FFT length, window, hop, magnitude epsilon, normalization, spectral convergence ve log magnitude ağırlıklarını açıkça sabitler.

Örnek iki bileşen:

\[
SC = \frac{\| |STFT(x)|-|STFT(\hat x)| \|_F}{\| |STFT(x)| \|_F + \varepsilon}
\]

\[
L_{\log} = \frac{1}{N}\sum_i |\log(|S_i(x)|+\varepsilon)-\log(|S_i(\hat x)|+\varepsilon)|
\]

Bu formüllerin hangilerinin hangi ağırlıkla tek skora dönüştürüleceği evaluator sürümünün parçasıdır. Eğitimin kullandığı STFT loss ile değerlendirme metriği aynı olabilir; **ancak evaluator bağımsız uygulama ve testlerle doğrulanır.** STFT uzaklığı tek başına doğal insan sesi kanıtı değildir.

### 11.3 SI-SDR ve hizalama

SI-SDR, sinyal yeniden oluşturma için yardımcı göstergedir. Codec'in filtre gecikmesi veya chunk boundary örneği değiştirmesi metrikte yapay cezaya yol açabilir. İzin verilen time-alignment yöntemi önceden sabitlenir; değerlendirme çıkışını referans gibi gösterecek sınırsız hizalama yapılmaz. Sessizlik, çok kısa kayıt ve sıfıra yakın enerji ayrı edge-case olarak ele alınır.

### 11.4 PESQ / STOI türü metrikler

PESQ gibi bazı geleneksel ölçümler belirli dar bant/geniş bant örnekleme hızlarına, konuşma bağlamına ve ek lisans/dağıtım koşullarına sahiptir. Bunlar **24 kHz sonuçlara doğrudan, rastgele resampling ile geçerli sayılar olarak uygulanmaz**. Kullanılacaksa standartların desteklediği örnekleme rejimi, lisansı, preprocessing ve hesaplama sınırlılığı protokolde açıkça belirtilir. STOI da anlaşılabilirliğin sınırlı vekilidir; doğal etkileşimi ölçmez.

### 11.5 Öğrenilmiş algısal metrikler

Önceden eğitilmiş bir ses kalitesi modelinden puan almak ek bir değerlendirme olabilir. Fakat değerlendirici modelin Türkçede, ilgili aksanlarda ve codec artefaktlarında ne kadar güvenilir olduğu ayrıca gösterilmelidir. ASR, speaker embedding veya öğrenilmiş kalite modelinin kimliği ve weight digest'i kayıt edilir; bir modelin skoru objektif ve tartışmasız gerçek diye sunulmaz.

### 11.6 Bitrate ile ortak rapor

RVQ tabanlı sabit frame/codebook yapısı için teorik bitrate:

\[
R_{bits/s} = f_{frames/s} \times Q \times \log_2 K
\]

Örnek: 50 Hz, Q=4, K=1024 için `50 × 4 × 10 = 2000 bit/s` yani teorik **2 kbps**. Bu sayı container, başlık, entropy coding ve ses dosyası overhead'lerini içermez. Codec karşılaştırmasında bitrate, kalite ve decoder hızı birlikte raporlanır; tek parametreye göre kazanan seçilmez.

## 12. Codebook ve token-prediction değerlendirmesi

Mevcut Speech `scripts/evaluate_codebooks.py`, codebook başına cross-entropy, perplexity, top-1 ve exact complete frame doğruluğu hesaplayabilir. Forge bu sonucun **hangi dataset, token formatı, checkpoint ve evaluator kodu ile** üretildiğini kaydeder.

### 12.1 Öğretmen zorlamalı ve serbest üretim ayrımı

- **Teacher-forced:** Geçmişte doğru token verilir; sonraki token tahmin olasılığı ölçülür.
- **Free-running:** Model önceki kendi üretimini girdi olarak kullanır; hata birikimi, kesilme ve tekrarlar görünür olur.

Teacher-forced accuracy iyi iken serbest üretim anlamsız veya kopuk olabilir. Birincil konuşma kalitesi puanı teacher-forced loss olarak tanımlanamaz.

### 12.2 Parallel ve Depth'in karşılaştırılması

Depth modunda q0 semantik/control iddiasını ifade eden akış ile q1..qN akustik akışlarının loss mask'i farklıdır. Acoustics loss, kontrol/EOS/pad karelerinde yanlış hesaplanmamalıdır. Cross-entropy agregasyonu her codebook'a eşit ağırlık veriyor mu, token sayısına göre mi ağırlıklandırılıyor; evaluator bunu açıkça yazar.

### 12.3 Acoustic delay uyumluluğu

`acoustic_delay_frames=1` continuation tarafında uygulanan dönüşüm; q0 güncel, q1..q7 bir frame gecikmeli temsil kullanır. `apply_acoustic_delay` / `remove_acoustic_delay` hizası değerlendirmede korunur. Mevcut dialogue yolunda aynı dönüşümün genel olarak uygulandığı varsayılmayacaktır. **Gecikmeli diyalog metriği için önce Speech tarafında kontrat/test düzeltmesi gerekir.**

### 12.4 Token doğruluk testleri

- q0 yalnız izinli control vocabulary'sini kullanır; acoustic stream kontrol tokenlarını yanlışlıkla yutmaz.
- EOS frame'i ses gibi decode edilmez.
- Acoustic token `0` gerçek değer olabilir; pad sayılmaz.
- `labels=-100`, loss masking içindir; gerçek token değildir.
- `Q`, `K`, frame rate ve tokenizer/model identity eşleşmeyen denemeler bloklanır.
- Boş audio üretimi ve çok erken EOS başarısız örnek olarak saklanır.
- Üretim `max_frames` sınırına çarparsa `truncated` durumuyla kaydedilir.

## 13. Türkçe konuşma değerlendirmesi

Kaiwen modelinin Türkçeye özgü başarısı; yalnız ASR'den alınan yazıya veya öznel “güzel ses” izlenimine indirgenmeyecek. Değerlendirme en az dört ayrı boyutta yapılır:

1. **İçerik doğruluğu:** Kullanıcının söylediği şeyi doğru anlaması ve uygun cevabı vermesi.
2. **Duyulabilirlik:** Türkçe telaffuzun, hecelerin, sözcük sınırlarının ve sayılar/özel adların anlaşılması.
3. **Etkileşim davranışı:** Uygun sıra alma, cümle sonu, duraksama, kısa tepki ve araya girme.
4. **Doğallık:** Ritim, vurgu, prosodi, ses tutarlılığı, anormal tekrar ve robotik artefaktlar.

### 13.1 Türkçe test sorusu tasarımı

Her test durumu `scenario_id`, `input_audio`, `reference_transcript` (varsa), `expected_intent`, `accepted_response_patterns`, `rubric`, `speaker_group`, `source_group` ve `difficulty` alanlarına sahip olabilir. Beklenen tek doğru cümle yerine **anlamsal kabul rubriği** hazırlanır. Örneğin “Günaydın” sorusunun cevabının birebir aynı waveform olması beklenmez; uygun Türkçe karşılık yeterlidir.

Test grupları: gündelik selamlaşma, kısa bilgi geri çağırma, olumsuzlama, sayı/tarih, özel ad, emir/soru ayrımı, belirsiz ifade, nazik reddetme, kısa bağlamsal devam, sessizlik/arka plan sesleri, araya girme, doğal backchannel.

### 13.2 Morfoloji ve kelime hataları

Türkçe eklemeli bir dil olduğu için WER tek başına yeterli değildir. `CER` ve `WER` birlikte raporlanabilir; ama ASR normalizer'ının noktalama, büyük/küçük harf, sayılar, kesme işaretleri, kısaltma, apostrof, Türkçe `İ/ı` ve dolgu seslerine nasıl davrandığı sabitlenmelidir. *Yanlış bir ASR transkripti, yanlış ses modeli sonucu sanılmamalıdır.*

### 13.3 ASR ile değerlendirmenin sınırı

Çıktıyı bağımsız bir ASR modeline yazdırıp transkripti ölçmek telaffuz/anlaşılabilirlik hakkında yardımcı olabilir. Ancak bir konuşma yanıtı birden fazla doğru ifadeyle verilebilir. Yanıtın ASR transkriptiyle tek bir referans cümle arasındaki WER, **diyalog anlam doğruluğunun güvenilir ana metriği değildir**. Uygun olduğu yerde insan anotasyonu, rubrik veya ayrı task evaluator gerekir.

### 13.4 Ses kalite hata taksonomisi

İnsan değerlendirmesinde şu etiketler kullanılabilir: `unintelligible`, `wrong_language`, `semantic_mismatch`, `repetition`, `truncated`, `hallucinated_content`, `unnatural_pause`, `prosody_flat`, `voice_drift`, `codec_artifact`, `speaker_confusion`, `interrupt_failure`. Bunlar gold label değildir; her etiketin açıklaması, örnekleri ve annotator agreement ölçülmelidir.

## 14. Doğal diyalog değerlendirmesi ve turn-taking

Diyalog değerlendirmesi yalnız bir dosyanın dinlenip puanlanması değildir. Girdi sesinin zaman sınırları ve üretilen cevap arasındaki etkileşimin zamana bağlı doğası ölçülmelidir.

### 14.1 Önerilen senaryo ailesi

- Kısa selamlaşma ve karşılık verme.
- İki turlu bağlam takibi.
- Kullanıcının kendi cümlesini düzeltmesi.
- Yarım bırakılmış cümle ve sessizliğin ayırt edilmesi.
- Uzun duraksama ve uygunsuz erken cevap.
- Backchannel için uygun/uygunsuz örnekler.
- Kullanıcının asistan konuşurken araya girmesi.
- Mikrofon gürültüsü ve üçüncü kişinin konuşması.

### 14.2 Turn-taking ölçüm prensipleri

Gold anotasyonda kullanıcı başlangıç/bitiş zamanları, response trigger anı ve kabul edilen cevap aralığı tanımlanır. `turn_taking_success` tek başına fazla geniştir; şu alt ölçümler ayrılır:

- `premature_response_rate`: kullanıcı sözünü bitirmeden uygunsuz konuşma.
- `missed_response_rate`: cevap verilmesi gerekirken yanıt yok.
- `response_start_delay_ms`: kabul edilen trigger anı → ilk uygun asistan konuşması.
- `backchannel_appropriateness`: kısa tepkilerin bağlama uygunluğu; insan rubriği.
- `overlap_handling`: konuşma çakışmasını yönetme; gerçek gold labels gerekir.

Bu metriklerin hiçbiri mevcut turn-based baseline'da otomatik “destekli” sayılmaz. Ölçümler ancak uygun audio event/state ve gold anotasyon sağlandığında açılır.

### 14.3 Full-duplex ve speaker ayrımı

Full-duplex uzun vadeli araştırma hedefidir. Kullanıcı/assistant rollerinin bulunması, birden çok konuşmacının güvenilir diyarlizasyonunu sağladığı anlamına gelmez. Diarization error rate gibi metrikler kullanılacaksa gerçeğe uygun konuşmacı segmentleri, overlap politikası ve scorer konfigürasyonu şarttır.

## 15. Codec dışındaki anlam ve konuşma kalitesi için insan değerlendirmesi

Otomatik ses metrikleri tek başına güçlü bir speech-native iddia için yeterli değildir. İnsan değerlendirmesi özellikle konuşma uygunluğu, doğal zamanlama, dinlenebilirlik ve bağlamsal doğrulukta önemlidir.

### 15.1 Kör A/B protokolü

- Aynı input ve aynı genel koşullarda iki model çıktısı hazırlanır.
- Klipler A/B olarak isimlendirilir; model adı ve deney hipotezi gizlenir.
- Sunum sırası, model etiketi ve A/B tarafı kayıt bazında randomize edilir.
- Ses dosyaları kırpılırken iki kolun aynı içerik sınırları ve loudness policy'si kullanılır; aşırı düzeltmeyle performans farkı maskelenmez.
- Dinleyiciye net ve tek boyutlu soru yöneltilir: “Hangisi daha anlaşılır?” veya “Hangisi daha doğal?” gibi.
- `tie` / `cannot_decide` seçeneği tanımlanır; zorla tercih alınmaz.
- Aynı konuşmacı/kayıt için tekrar puanlama sayısı kaydedilir.
- Anket ekranı model ismi veya önceden iddia edilen iyileşmeyi göstermemelidir.

### 15.2 Örnek blind trial manifesti

```json
{
  "schema_version": 1,
  "study_id": "blind-study-example-01",
  "trial_id": "example-trial-002",
  "sample_id": "example-record-011",
  "question_id": "naturalness-v1",
  "presentation_order_seed": 7421,
  "audio_a": "asset://example-variant-opaque-a",
  "audio_b": "asset://example-variant-opaque-b",
  "identity_mapping_ref": "restricted://example-private-mapping",
  "reference_audio_ref": "asset://example-reference",
  "allowed_answers": ["a", "b", "tie", "cannot_decide"]
}
```

Harici katılımcıya verilen trial manifestinde **gerçek model kimliği veya kimliğe geri çözülebilecek dosya yolu bulunmaz**. `identity_mapping_ref` yalnız erişim kontrollü analiz katmanında çözülür. Bu JSON bir örnektir; servis bugün çalışmıyor.

### 15.3 İnsan değerlendirme rubriği

Her soru ayrı ve anlaşılır değerlendirilir. Örnek rubrikler:

| Boyut | Örnek soru | Yanlılık kaynağı |
|---|---|---|
| Anlaşılırlık | Söylenen ifadeyi anlayabildin mi? | Kulaklık, ortam gürültüsü |
| Doğallık | Konuşma doğal bir insan temposunda mı? | Aksan tercihleri |
| Anlamsal uygunluk | Kullanıcıya uygun karşılık mı? | Girdi bilgisinin eksik verilmesi |
| Zamanlama | Çok erken/geç mi konuştu? | Playback UI'nin farklı yükleme süreleri |
| Ses tutarlılığı | Başından sonuna ses aynı mı? | Kaynak ses kalitesi |
| Codec artefaktı | Tıkırtı, metaliklik, kopma var mı? | Sıkıştırma ve sample rate |

### 15.4 MOS kullanımı

Mean Opinion Score kullanılacaksa ölçek açıklaması, soru metni, katılımcı seçim kriteri, ortam, görev, kontrol klipleri ve örnek sayısı kaydedilir. 1–5 puanın otomatik ortalamasını almak tek başına güvenilir çalışma yapmaz. Özellikle katılımcıların bağımsızlığı, örneklerin temsiliyeti ve ölçeğin ordinal doğası göz önüne alınmalıdır.

### 15.5 Etik koşullar

Katılımcılar dinletilen kayıtların araştırma amacıyla kullanılacağı konusunda bilgilendirilir. Gönüllü katılım, erişim kısıtı ve sonuç saklama politikası uygulanır. Kayıtlarda kişisel bilgi, iletişim içeriği veya gerçek kişileri tanımlayıcı unsurlar varsa anonimleştirme ve izin kontrolü zorunludur.

## 16. İstatistiksel değerlendirme ve belirsizlik

### 16.1 Temel analiz birimi

Her token, birbirinden bağımsız bir insan örneği değildir. Aynı konuşmacıya ait 100 ses segmentini 100 bağımsız konuşmacı gibi saymak güven aralığını yapay daraltabilir. Analiz birimi çalışmanın sorusuna göre `speaker`, `recording`, `conversation` veya `participant` olabilir; protokol bunu önceden tanımlar.

### 16.2 Eşlenmiş kıyaslama

İki model aynı input örnekleri üzerinde değerlendiriliyorsa, her `sample_id` için A−B farkı hesaplanır. **Örnekleri eşlemeden** bağımsız iki ortalamayı karşılaştırmak, veri değişkenliğini gereksiz artırır. Başarısız veya eksik eşler hangi nedenle dışlandıysa raporlanır; yalnızca iyi cevap verenler seçilmez.

### 16.3 Bootstrap güven aralığı

Aşağıdaki kod **algoritma taslağıdır**; istatistik kütüphanesi veya çalışan Forge API değildir.

```python
# HEDEF PSEUDOCODE: gerçek sampling_unit / cluster yöntemi ayrı doğrulanmalı.
# pairs = [(speaker_id, score_a, score_b), ...]
# delta = score_b - score_a
# 1) unique speaker_id gruplarını oluştur
# 2) her tekrar için speaker gruplarını geri koymalı örnekle
# 3) seçilen grupların içindeki paired delta'ların istatistiğini hesapla
# 4) 2.5 ve 97.5 persentillerini CI olarak raporla
# 5) seed, replikasyon sayısı ve estimator adını sakla
```

Bootstrap tekrarı, sampling unit ve confidence level sürümlenmelidir. Confidence interval gerçek belirsizliğin tek kaynağı değildir; dataset/domain drift ve değerlendirici yanlılığı ayrıca açıklanır.

### 16.4 Çoklu karşılaştırma

Birden fazla codec, frame rate, Q sayısı, model boyutu ve on metrik deneniyorsa tesadüfen iyi sonuç çıkma olasılığı yükselir. **Önceden seçilmiş primary endpoint** ayrı; post-hoc keşifsel analiz ayrı sunulur. Birden çok hypothesis test için gerekirse Holm/FDR gibi düzeltmeler uygulanır; kullanılan yöntem protokolde yer alır.

### 16.5 Etki büyüklüğü

`p < 0.05` sonucunu ürün açısından anlamlı kazanım saymayız. Örneğin 2 ms hızlanma, ses kalitesini ciddi düşürüyorsa birincil başarı değildir. Mutlak fark, göreli fark, güven aralığı, donanım maliyeti ve ses kalitesi değişimini birlikte sunarız.

### 16.6 Tekrarlı seed ve environment belirsizliği

Aynı model konfigürasyonunun farklı seed'lerle eğitimi ayrı run'lardır. Test setinin aynı olması eğitimin deterministik olduğu anlamına gelmez. Bazı GPU kernel'leri deterministik değilse “bitwise reproducible” iddiası verilmez. İstatistiksel analiz `between_run` ve `within_dataset` varyansını gerektiğinde ayrı raporlar.

## 17. Performance evaluation ve ölçüm sınırları

Bu bölüm [Speech Performance](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/PERFORMANCE.md) sözlüğünü tekrar tanımlamaz; Forge aynı sürümlü metrik tanımlarını tüketir. Runtime benchmark ve Live E2E arasındaki ölçüm farkı korunur.

### 17.1 Başlangıç koşulları

- **Cold:** Process yeni ve model yeniden yükleniyor; disk/cache etkisi kaydedilir.
- **Warm:** Model yüklü, ilk kernel/cache etkileri ölçülüyor.
- **Hot:** Kararlı aşamada üretim ve ses aktarımı ölçülüyor.

Üç sonuç aynı grafikte tek “latency” değeri olarak gösterilmez.

### 17.2 TTFA

- `ttfa_runtime_ms`: konuşma için yeterli girdinin runtime'a verildiği andan, ilk geçerli PCM ses chunk'ına.
- `ttfa_audible_ms`: kullanıcı açısından yanıt tetiklendikten hoparlörden ilk duyulabilir sesin çıkışına.

İlk token, ilk PCM, ilk duyulabilir ses ve model yükleme süresi farklı ölçümlerdir. Forge bu sınırları tek `ttfa_ms` değeri altında birleştirmeyecektir.

### 17.3 RTF

\[
RTF = \frac{\text{duvar saati işlem süresi}}{\text{üretilen/işlenen ses süresi}}
\]

RTF<1 ortalama hesaplama hızının gerçek zamanın önünde olduğuna işaret edebilir; kısa gecikme sıçramalarını, playback underrun'ı veya erken EOS'u tek başına dışlamaz. Codec encode, Transformer generation, codec decode ve tamamı için ayrı RTF gerekir.

### 17.4 GPU bellek

- `peak_gpu_allocated_bytes`: framework tarafından aktif kullanılan GPU tensörleri.
- `peak_gpu_reserved_bytes`: allocator'ın rezerve ettiği GPU bellek.
- `peak_device_used_bytes`: harici telemetriyle gerçek cihaz/process kullanımına yaklaşan ölçüm.
- `peak_rss_bytes`: ana sistem RAM'i.

Özellikle 2 GB VRAM hedefinde yalnız model ağırlığı veya allocated CUDA memory yeterli kanıt değildir. Driver, context, codec, KV cache, audio queue ve uzun session state de ölçülmelidir. 6 GB RTX 4050'de 2 GB tüketim görüldü diye bağımsız 2 GB GPU'da çalıştığı iddia edilmez.

### 17.5 Benchmark güvenilirliği

Ölçüm ortamı: CPU/GPU modeli, sürücü, CUDA, Python/Torch sürümü, power mode, precision/quantization, codec, model, context size, warmup, tekrar sayısı, eşzamanlı yükler, ses uzunluğu ve session sayısı ile kaydedilir. CPU/GPU eşzamanlı işlemlerin asenkron doğası nedeniyle doğru senkronizasyon sınırları gerekir. Mikrobenchmark ile doğal akış benchmark'ı ayrı tutulur.

### 17.6 Performans regresyonu

- Uyarı eşikleri CI için önce **tekrarlanabilir baseline dağılımı** ile seçilir.
- Sabit mutlak sınırlar ile göreli yüzde regresyonları farklı kategoridir.
- Donanım değiştiyse karşılaştırma cihaz/driver uyumluluk kapısını geçmelidir.
- CI CPU smoke testi ağır gerçek GPU performansının yerine geçmez.
- Kalite düşüşünü gizleyerek hız kazanmak kabul edilmez; quality–latency Pareto birlikte değerlendirilir.

## 18. Streaming, kesme ve zamanlama değerlendirmesi

Gerçek streaming ancak modelin `streaming_session` capability'si varsa test edilir. Fake runtime session testleri yalnız protokolün doğru çalıştığını kanıtlar; gerçek modelin streaming yeteneğini değil.

### 18.1 Streaming metrikleri

- Input PCM chunk geliş aralığı ve jitter.
- `prefill_ms`, temporal frame başına `decode_step_ms`.
- Codec encode/decode parça gecikmesi.
- Queue depth, drop count, backpressure event'leri.
- İlk PCM chunk ve ilk duyulabilir ses.
- Playback underrun sayısı ve ses kopma süresi.
- `interrupt_audio_stop_ms` ve stale-generation sesinin sızıp sızmadığı.
- Oturum kapatıldıktan sonra bellek ve worker temizliği.

### 18.2 İptal ve stale output

Bir kullanıcının konuşmayı kestiği `generation_id` sonrasında eski neslin PCM paketleri hoparlöre ulaşmamalıdır. `session_epoch` değişince eski event'ler kimlik doğrulama filtresinden geçemez. Başarısızlıkları yalnız ölçülen gecikme sayısı değil `stale_audio_after_cancel` güvenlik/UX ihlali olarak raporlarız.

### 18.3 Offline/online parity

Codec chunk encoder/decoder çıktıları ile offline çıktılar aynı koşullarda kabul edilen toleransta eşleşmelidir. Transformer cached vs uncached deterministik generation karşılaştırılır. Baştan itibaren bütün akışa latency azaltmak adına farklı semantik kararlar eklenirse bunun etkisi ayrı deney olarak adlandırılır.

### 18.4 Uzun oturum

Dakikalarca süren tekrarlı konuşmalar, session reset, yarım input chunk, cihaz ayrılması, OOM ve uygulama yeniden açılışı test edilir. Belirli donanım profilinde onaylanan soak süresi sabitlenir. Bir kez çalışan 5 saniyelik demo “stable full-duplex” iddiasını desteklemez.

## 19. Hata taksonomisi ve eksik örnek politikası

Evaluation engine hiçbir hatayı sessizce dışarı atamaz. Örnek başına sonuç durumu şu sınıflardan biri olmalıdır:

| Durum | Örnek | Nasıl ele alınır? |
|---|---|---|
| `completed` | Geçerli ses/metrik üretildi | Agregasyona dahil |
| `completed_with_caveat` | Ses üretildi ama süre kırpıldı | Protokole göre etiketli dahil veya dışarı |
| `not_measured` | Gerekli ASR modeli yok | `null + reason`, başarı sayılmaz |
| `blocked` | Codec digest uyuşmuyor | Eval fail-fast |
| `invalid_input` | Bozuk / çok kısa PCM | Hata nedeni, örnek ID kaydedilir |
| `timeout` | Süre limitini aştı | Başarısızlık/coverage metriğine dahil |
| `oom` | GPU bellek yetersiz | Başarısızlık, donanım kaydı |
| `generation_failed` | Model boş cevap verdi | Sıfır ses kalitesi olarak uydurulmaz |
| `artifact_missing` | Output kayıp | Tamamlanmadı, rapor engellenebilir |
| `cancelled` | İş kullanıcı tarafından kesildi | Ayrı terminal status |

### 19.1 Coverage ve başarısızlık yanlılığı

`success_rate = completed / attempted` raporlanır. Bir kalite metriği yalnız 60/100 örnekte hesaplanabiliyorsa “quality=0.9” yanına **n=60/100 ve 40 başarısız örneğin nedenleri** yazılır. Protokolde izin verilen maksimum kayıp oranı aşılırsa sonuç otomatik olarak `inconclusive` veya `invalid` olur.

### 19.2 Fail-fast / continue

Hash uyumsuzluğu, codec identity uyuşmazlığı, veri seti lisans engeli veya schema bozukluğu gibi sistemik hatalar **fail-fast** davranır. Bireysel bozuk klip veya sample timeout belirlenmiş limite kadar `continue_and_record` olabilir. Bu karar değerlendirme başlamadan protokolde kilitlenir.

### 19.3 Retry ve idempotency

Tekrar edilen sample sonucu yeni `attempt_id` ile kayıt edilir. Aynı `evaluation_id/sample_id/attempt_id` çiftine iki farklı payload yazılmaz. Retry sonucunu eski başarısız kaydın üstüne yazmak yasaktır. Hangi girişin son rapora girdiğini aggregation policy tanımlar.

## 20. Evaluation execution lifecycle

`TARGET_V1` state machine:

```text
created
   |
   v
preflight -----failed_preflight
   |
   v
queued ------ cancelled
   |
   v
running -----> cancelled
   |   \
   |    \------> interrupted / failed
   v
verifying ----> failed_verification
   |
   v
aggregating -> inconclusive / failed_aggregation
   |
   v
completed
```

`completed` demek modelin kaliteli olduğu değil, belirlenen değerlendirme işinin geçerli metrik ve artifaktlarla tamamlandığı anlamına gelir.

### 20.1 Worker yönetimi

- Run kimliği, kaynak ayırma, worker PID ve heartbeat Forge job altyapısında saklanır.
- Timeout hard/soft ayrılır; GPU işlemleri iptal sonrasında temizlenir.
- Aynı GPU üzerinde rekabet eden eğitim koşuları performans ölçümünü bozabileceğinden benchmark scheduler exclusive slot desteklemelidir.
- Eval result yazımı geçici dosya → fsync/durable policy → atomik rename → digest doğrulama zincirini izlemelidir; dosya sistemi sınırlarında başarısızlık testleri olmalıdır.
- Colab veya remote worker sonuçları yalnız signed/verified manifest ve sınırlı güven modeliyle içe alınır.

### 20.2 Process ve log güvenliği

Speech CLI çağrıları `shell=False` ve izin verilen argüman listesiyle yapılır. Serbest kullanıcı metni komuta dönüştürülmez. Stdout başarının tek kaynağı değildir; process exit, beklenen artifact, schema, hash ve coverage doğrulaması birlikte gerekir. Loglara kişisel ses içeriği, erişim tokenı veya secret yazılmaz.

## 21. Artifact sözleşmesi ve dosya düzeni

Hedef bir evaluation çalışma dizini:

```text
artifacts/evaluations/<evaluation_id>/
├── protocol.snapshot.json
├── protocol.sha256
├── inputs.manifest.json
├── environment.json
├── evaluation.json
├── metrics.jsonl
├── samples.jsonl
├── events.jsonl
├── statistics.json
├── report.md
├── checksums.sha256
├── previews/                 # hak/izin uygunsa, varsayılan private
└── attempts/
    └── <attempt_id>/
        ├── worker.log
        ├── stdout.log
        ├── stderr.log
        └── tool-output/      # Speech CLI ham artifact, doğrulama sonrası
```

Dizin isimleri path traversal'a kapalı, normalize edilmiş güvenli ID'lerden üretilir. Artifact URI'ları kendi başına kamuya açık URL olarak verilmez. Her dosya hash, boyut ve schema version taşır.

### 21.1 Evaluation manifest örneği

```json
{
  "schema_version": 1,
  "evaluation_id": "example-eval-001",
  "experiment_id": "example-experiment-001",
  "run_id": "example-run-001",
  "attempt_id": "example-attempt-001",
  "status": "completed",
  "task": "codec_roundtrip",
  "protocol_revision": 1,
  "protocol_digest": "EXAMPLE_REPLACE_WITH_REAL_SHA256",
  "dataset_manifest_digest": "EXAMPLE_REPLACE_WITH_REAL_SHA256",
  "model_artifact_digest": "EXAMPLE_REPLACE_WITH_REAL_SHA256",
  "codec_artifact_digest": "EXAMPLE_REPLACE_WITH_REAL_SHA256",
  "evaluator": {
    "id": "codec_stft_v1",
    "version": "1.0.0",
    "implementation_commit": "EXAMPLE_REPLACE_WITH_REAL_COMMIT"
  },
  "coverage": {
    "attempted": 100,
    "completed": 100,
    "failed": 0
  },
  "metrics": {
    "eval.codec.stft_distance@1": {
      "value": 0.187,
      "unit": "spectral_distance",
      "n": 100,
      "status": "measured"
    }
  },
  "report_uri": "artifact://example-eval-001/report.md",
  "verified": false,
  "verification_note": "Illustrative manifest only; hashes and scores are placeholders."
}
```

`status=completed` üst seviyede örnek amaçlıdır; `verified=false` olduğu için böyle bir manifest gerçek registry gate'ten geçemez. Sınır çok önemlidir: **yapısal geçerlilik bilimsel doğrulama değildir**.

### 21.2 JSONL event kayıtları

`sample_id`, `attempt_id`, `metric_name@version`, `status`, `value`, `unit`, `timestamp_utc`, `evaluator_digest`, `input_digest`, `output_digest` ile kaydedilir. Tekrarlanan sample satırı sessizce son değerle değiştirilmez. Log ve metrik olaylarının kişisel veri içermemesi varsayılan kuraldır.

### 21.3 Güvenlik ve saklama

Kamuya açıklanan rapor ile hassas ses/yanıt kayıtları ayrı depolanır. Örnek çıktılar kamuya açık depoya push edilmez. Eğitim datasetindeki erişim kuralı evaluation export'unda da korunur. Dış kaynaktan indirilen dosyalara süresiz güven verilmez; hash ve hak bilgisi tekrar kontrol edilir.

## 22. SQLite kayıt modeli ve transaction sınırı

İlk sürüm için SQLite WAL yeterlidir. Önerilen mantıksal tablolar:

- `evaluation_protocols(id, revision, digest, created_at, locked_at, snapshot_uri)`
- `evaluations(id, protocol_id, run_id, status, created_at, completed_at)`
- `evaluation_attempts(id, evaluation_id, worker_id, status, started_at, ended_at, error_code)`
- `evaluation_samples(evaluation_id, sample_id, status, source_group_id, selected_attempt_id)`
- `metric_definitions(name, version, unit, aggregation, direction, evaluator_digest)`
- `metric_observations(evaluation_id, attempt_id, sample_id, metric_id, value, status, reason)`
- `evaluation_artifacts(id, evaluation_id, digest, path, size_bytes, media_type)`
- `evaluation_events(id, evaluation_id, attempt_id, sequence, kind, payload_uri)`

Unique constraints ve foreign key zorunludur. Örnek başına metrik upsert için idempotency kuralı açıkça test edilir. Tabloların gerçek adı mevcut `TRAINING_PIPELINE.md` ve `EXPERIMENTS.md` data model'i ile uyarlanır: **üç ayrı belge üç bağımsız run store anlamına gelmez.**

### 22.1 Transaction ilkeleri

- Bir işin `completed` durumu, artifact doğrulama ve summary kaydı tamamlanmadan yazılamaz.
- SQLite commit ile harici dosya yazımı atomik tek işlem değildir. `staged → verify → committed` iki aşamalı uygulama gerekir.
- Crash recovery, eksik artifact/DB uyumsuzluklarını tarar ve tamamlanmış gibi göstermeden `recoverable` veya `failed_verification` statüsüne taşır.
- Schema migration öncesi yedek ve rollback prosedürü bulunur.

## 23. Değerlendirme raporunun yapısı

Her rapor hem insan-okur Markdown hem machine-readable JSON üretir. Minimum başlıklar:

1. Araştırma sorusu, protokol kimliği ve tarih.
2. Model/codec/checkpoint/dataset sürümü, hashes ve evaluator commit'i.
3. Test verisinin içeriği, split/fingerprint, lisans özeti ve dışarıda bırakılan kayıtlar.
4. Önceden seçilmiş birincil metrik ve yöntem.
5. Örnek başına geçerlilik, `n attempted / n completed` ve başarısızlık nedenleri.
6. Agregasyon, confidence interval, seed ve sampling unit.
7. Baseline–candidate kıyaslaması, compatibility gate sonucu.
8. Donanım ve hız/bellek koşulları (ölçüldüyse).
9. İnsan dinleme protokolü, körleme ve katılımcı sayısı (uygulandıysa).
10. Başarısız örneklerden seçilmiş etik/izinli hata analizi.
11. Sonuçların sınırları; genelleştirilmemesi gereken koşullar.
12. Tam veri ve kod provenance'i; tekrar çalıştırma talimatları.

Rapor başlığında `VERIFIED`, `INCONCLUSIVE`, `INVALID` veya `EXPLORATORY` sınıflarından biri yer alır. **Görsel olarak güzel rapor, eksik bilimsel kanıtı ortadan kaldırmaz.**

### 23.1 Karar özeti

Bir aday model “tercih edilir” sonucuna ancak protokolde tanımlı metric/constraint setini sağlıyorsa ulaşır. Örneğin kalite primary, TTFA ve peak VRAM hard constraint olabilir; ya da 2 GB hedefi için kalite–bellek Pareto karşılaştırması yapılabilir. Bu tercih deney başlamadan yazılır. Belge belirli sabit kalite sayısı uydurmaz; önerilen threshold başlangıç olarak konur ve pilot deneyden sonra gerekçelendirilerek sürümlenir.

## 24. Speech'teki mevcut komutlarla ilk entegrasyon

Şu komutlar **mevcut Speech reposunda vardır**, fakat bu belge hazırlanırken çalıştırılmamıştır. `uv` ortamı ve gerçek dosya yolları gerekir.

```bash
# kaiwen-speech repository kökünde
uv sync --frozen --extra dev
uv run pytest
uv run ruff check .
uv run pyright

# Sentetik model forward maliyeti; gerçek zamanlı ses ölçümü değildir:
uv run python scripts/benchmark.py --config configs/baseline_30m.yaml --frames 128

# Var olan gerçek checkpoint ve Mimi token cache sağlandıktan sonra:
uv run python scripts/evaluate_codebooks.py \
  --checkpoint /path/to/checkpoint.pt \
  --token-cache /path/to/mimi_tokens \
  --device cpu

# Gerçek checkpoint ve token cache ile continuation WAV örnekleri:
uv run python scripts/evaluate_continuation_audio.py \
  --checkpoint /path/to/checkpoint.pt \
  --token-cache /path/to/mimi_tokens \
  --output-dir /path/to/eval_audio
```

`/path/to/...` gerçek giriş değerleriyle değiştirilmeden komutlar çalışmaz. `evaluate_codebooks.py` token cache pattern `real_*.pt` bekler; `evaluate_continuation_audio.py` da continuation kayıtları bekler. Her iki komutun iç `validation_records or all_records` fallback'i olabilir: **val split boşsa tüm kayıtlara dönüldüğü için çıkan skor bağımsız holdout sonucu olarak ilan edilemez**. Forge adapter'ı gerçek split provenance ve leakage gate'i kontrol etmeden onu `test` diye etiketlemez.

### 24.1 CLI adapter aşamaları

1. Speech commit/paket sürümü sabitlenir.
2. İzinli komut ve argüman biçimleri listelenir.
3. CLI stdout biçimi için fixture-backed parser eklenir.
4. Dosya üretimi ve çıkış kodu doğrulanır.
5. Artefaktlar Forge evaluation manifestine bağlanır.
6. Hatalı parse veya sürüm değişiminde sessiz yanlış sayı üretmek yerine fail edilir.
7. Speech'e structured evaluation sink eklenince CLI adapter kontrollü biçimde güncellenir.

### 24.2 Hedef Forge CLI — uygulanmış değildir

```bash
# HEDEF KOMUTLAR — bugün kaiwen-forge reposunda çalışmaz.
kaiwen-forge eval validate evaluations/codec_v1.yaml
kaiwen-forge eval run evaluations/codec_v1.yaml
kaiwen-forge eval verify example-eval-001
kaiwen-forge eval compare example-eval-001 example-eval-002
kaiwen-forge eval report example-eval-001 --format markdown
```

Çalışan CLI uygulamasının gerçek isimleri ve parametreleri ayrı PR ile kesinleşir. README belgeleriyle eş zamanlı güncellenmeden kullanıcıya “çalıştır” komutu verilmez.

## 25. Karşılaştırılabilirlik kapısı ve deney sonuçları

Aday model ile baseline karşılaştırılmadan önce `EXPERIMENTS.md` içindeki `compatible`, `compatible_with_caveats`, `incomparable`, `insufficient_provenance` kuralları uygulanır. İki farklı görevden gelen `loss` aynı grafiğe konabilir ancak tek eksenli ranking metriği yapılamaz.

### 25.1 İzinli ve izinli olmayan karşılaştırmalar

| Senaryo | İlke |
|---|---|
| Aynı codec/dataset, Parallel vs Depth | Teacher-forced ve free-running birlikte ölçülerek kıyaslanabilir |
| Mimi vs Kaiwen Codec reconstruction | Aynı kaynak PCM, tutarlı preprocessing ve bitrate/kalite koşullarıyla karşılaştırılabilir |
| Mimi token CE vs Kaiwen token CE | Farklı vocabulary/token rate yüzünden doğrudan kalite sırası kurulamaz |
| 6 GB GPU'daki 2 GB allocated vs gerçek 2 GB GPU | Doğrudan cihaz uyumluluğu iddiası yapılamaz |
| Teacher-forced continuation vs canlı dialogue | Aynı metrik/başarı anlamına gelmez |
| İnsan A/B testi, aynı örnekler ve kör protokol | Katılımcı ve örneklem sınırı belirtilerek karşılaştırılabilir |
| Farklı input türleri ama aynı RTF | Süre, pipeline ve cihaz eşit değilse iddia kurulamaz |

### 25.2 Pareto ve karar

Model seçimi birden fazla amaç taşır: speech quality, TTFA, RTF, VRAM, model/codec size ve bakım maliyeti. Pareto frontier raporu yapılabilir; fakat örnek kalitesi ve metrikler farklıysa Pareto hesabı bilimsel olarak geçersizdir. Bir kararın gerekçesi hangi kısıtın sert, hangisinin tercih olduğu açıkça yazılmalıdır.

## 26. Google Colab ve Google Drive değerlendirme handoff'u

Forge'un evaluation worker'ı ileride Colab hosted runtime'da çalışabilir. Ancak Colab runtime'ının disk/kota/oturum süreleri değişken olduğu için sonuç paketleri küçük, doğrulanabilir ve resume edilebilir biçimde hazırlanır.

Hedef akış:

1. Forge değerlendirme protokolünü ve gereken model/dataset artifact referanslarını çözümleyip hash'ler.
2. Colab ortamı doğrulanan versiyonları indirir; mümkünse büyük dosyaları runtime'ın yerel hızlı diskinde işler.
3. Değerlendirme `samples.jsonl`, `metrics.jsonl` ve artifact manifestini kademeli yazar.
4. Tamamlanan shard/part dosyaları checksum ile kalıcı Drive dizinine aktarılır.
5. Colab kesilirse sadece sağlam parçalar yeni attempt ile devam eder.
6. Yerel Forge import aşaması hash, schema ve rights gate'i doğrular.
7. `verified` olmayan sonuç registry'ye kanıt olarak eklenmez.

**Ağ/kapasite gerçeği:** Google Drive'ın kullanıcının hesabındaki alanı Colab runtime diskini sınırsız yapmaz; Drive mount kalıcılığı ve veri aktarım hızı garanti değildir. Kaynak erişim izinlerini ve Hugging Face lisans kısıtlarını bypass etmek için otomasyon kurulmaz.

## 27. Model Registry ve yayınlama sınırı

Evaluation sonuçları model release kararı için **girdi** sağlar; Forge Evaluation Engine kendiliğinden model yayımlamaz. Registry adayının en az aşağıdakileri taşıması beklenir:

- Model, codec, tokenizer ve config kimliği/digest.
- Hedef runtime ve desteklenen input/output sözleşmeleri.
- Dataset/license lineage ve dağıtım izni.
- Primary metric, evaluator ve dataset manifest sürümleri.
- En az bir gerçek konuşma kalite kanıtı, gerekiyorsa insan değerlendirmesi.
- Hedef donanımda performans kanıtı veya açıkça belirtilmiş destek sınırı.
- Known failures, security review, rollback ve önceki release'e göre regressions.

“Candidate”, “Research Verified”, “Demo Ready”, “Release Ready” farklı durumlar olabilir. Hiçbiri yalnız `train_loss` veya `checkpoint exists` nedeniyle otomatik yükseltilmez.

## 28. Güvenlik, tehdit modeli ve etik

### 28.1 Korunacak varlıklar

- Kişisel veya lisanslı ses kayıtları.
- Privileged evaluation seti ve kilitli holdout cevapları.
- Model ağırlıkları ve ticari/kısıtlı artifact'lar.
- Deney sonuçlarının bilimsel bütünlüğü.
- Colab/Drive/GitHub tokenları ve bulut erişimleri.

### 28.2 Saldırı yüzeyleri

- User-provided evaluation YAML üzerinden shell injection veya path traversal.
- Güvenilmeyen `torch.load` veya pickle tabanlı yükleme.
- Manipüle edilmiş dataset manifesti/digest.
- Kötü niyetli medya dosyası, çok uzun/bozuk PCM, arşiv bombası.
- Yanıltıcı metrik event'leri, tekrar eden sample ID ve sıralama yarışı.
- Trial mapping sızıntısı sonucu kör değerlendirmenin bozulması.
- Yanlışlıkla Git'e commit edilen ham ses, kişisel transcript veya API anahtarı.

### 28.3 Asgari önlemler

- Dosya yolu allowlist ve şema validasyonu; hash ve boyut sınırları.
- `shell=False` ve kontrollü executable/argüman listesi.
- Artifact yüklemede veri tipini/formatını izinli hale getirme.
- OS application data / çalışma dizini sınırları ve en az yetki.
- Gizli dosyaların `.gitignore` ve CI secret scanning politikaları.
- Holdout setine erişen rollerin ve kullanım kayıtlarının izlenmesi.
- Evaluator çıktılarının immutable audit log ile saklanması.

## 29. Test stratejisi: üç ayrı şey test edilir

**A. Hesaplama doğruluğu:** Evaluator matematiği ve aggregation doğru mu?  
**B. Yazılım güvenilirliği:** İşler, event'ler ve artifact'lar kaybolmadan yönetiliyor mu?  
**C. Bilimsel geçerlilik:** Veri, metrik, sampling ve karşılaştırma soruya uygun mu?

Bu üç kategori birbirinin yerine geçmez.

### 29.1 Unit testleri — TARGET_V1

- [ ] CER/WER normalizer fixture'ları (`İ/i`, `ı/I`, sayılar, kesme, Unicode NFC).
- [ ] STFT metric için bilinen dalga/zero/silence/phase/noise fixture'ları.
- [ ] SI-SDR'da sıfır enerji ve hizalama hata durumları.
- [ ] Metric registry `name@version` çözümleme.
- [ ] Finite value ve unit validation; `NaN`/`Inf` reddi.
- [ ] Stable hash sampling; manifest satır sırası bağımsızlığı.
- [ ] Aynı sample'ın kopyalarının tespiti.
- [ ] Paired delta ve deterministic bootstrap fixture'ları.
- [ ] Cluster sampling ile sahte bağımsızlık testleri.
- [ ] Başarısız örneklerin coverage hesaplamasına dahil edilmesi.
- [ ] `not_measured` metriklerine sıfır uydurulmaması.
- [ ] Protocol snapshot kilidi ve revision enforcement.
- [ ] Audio sample rate/channel mismatch engeli.

### 29.2 Adapter / integration testleri

- [ ] Mevcut Speech `evaluate_codebooks.py` stdout fixture parser.
- [ ] Speech CLI sürümü değişince explicit adapter incompatibility.
- [ ] Synthetic Speech fixture üzerinden `evaluation.json` üretimi.
- [ ] Wrong codec digest nedeniyle fail-fast.
- [ ] Missing checkpoint ve bozuk `.pt` için güvenli hata.
- [ ] Dataset split fingerprint uyuşmazlığı engeli.
- [ ] Output artifact yazma/rename sırasında crash recovery.
- [ ] İş iptali sonrası yetim process bulunmaması.
- [ ] Tekrar başlatılan eval attempt kayıtlarının korunması.
- [ ] Colab import'ta yanlış SHA-256 reddi.
- [ ] `completed` duruma geçiş için artifact verification şartı.

### 29.3 İnsan değerlendirme testleri

- [ ] Trial A/B tarafının seed ile yeniden üretilebilir randomizasyonu.
- [ ] Değerlendiriciye model kimliği sızıntısı olmaması.
- [ ] Körleme anahtarının yalnız analysis side'da çözülmesi.
- [ ] Aynı participant'ın birden fazla oyu için cluster farkındalığı.
- [ ] `tie`, `skip`, `cannot_decide` ayrı ve geçerli yanıtlar.
- [ ] Hak/izin kontrolü olmadan audio export yapılmaması.

### 29.4 Performance testleri

- [ ] CPU fixture benchmark deterministik metrik şeması üretir; “GPU verified” demez.
- [ ] CUDA mevcut olmayan ortamda `peak_vram` metrikleri `not_measured` olur.
- [ ] Timings wall-clock metodolojisi ve warmup protocol'ü kaydedilir.
- [ ] Gerekli `cuda.synchronize` olmadan GPU hız iddiası yapılmaz.
- [ ] Cancel sonrası stale audio event kabul edilmez.
- [ ] Rolling 30–60 dakikalık soak test planı ve leak ölçüm protokolü.

### 29.5 CI/CD standardı

V1 PR kalite kapısı: `ruff`, type checking, `pytest` CPU fixture testleri, JSON/YAML şema örnekleri, package build ve gizli bilgi taraması. Ağ gerektiren ASR/codec download'ları varsayılan CI testinden ayrılır. GPU ve büyük ses datasetleri nightly/manual opt-in matrix ile çalıştırılır; sonuçlar donanım/protokol kimlikleriyle saklanır.

`kaiwen-forge` reposunda bugün bu CI/workflow dosyaları yoktur. Bu liste **implementation requirement** olup mevcut test başarısı iddiası değildir.

## 30. Geliştirme planı: EV0–EV12

Her aşama bağımsız doğrulanabilir olmalıdır. Tek PR'da tüm evaluation ekosistemini kurmak reddedilir.

### EV0 — Repo gerçekliği ve sözleşme teyidi

Forge repo içeriği, Speech CLI sürümleri, mevcut dokümantasyon, riskler ve bağımlılık yönleri raporlanır. Çıktı: kısa implementation ADR ve görev listesi.

### EV1 — Domain modelleri ve schema

`EvaluationProtocol`, `EvaluationAttempt`, `SampleResult`, `MetricDefinition`, `ArtifactRef` tipleri ve JSON schema yazılır. Test: geçersiz kimlik ve eksik provenance reddi.

### EV2 — Registry ve metric ingestion

Tek kaynaktan metrik tanımı çözümlenir, idempotent örnek kaydı ve finite value guard uygulanır. Test: tekrar event ve yanlış birim hatası.

### EV3 — Dataset resolver ve rights gate

Corpus manifesti okunur, split/hash/lisans/çift kayıt doğrulanır. Test: sahte ve gerçek fixture. Veri dışa kopyalanmadan referans üzerinde çalışabilir.

### EV4 — Speech CLI adapter

`evaluate_codebooks.py` için kontrollü subprocess wrapper, version parser, stdout capture ve artifact doğrulama geliştirilir. Test: küçük token cache ve sahte CLI fixture.

### EV5 — İlk gerçek token evaluation

Mevcut Speech checkpoint'i varsa gerçek CPU eval çalıştırılır; yoksa synthetic fixture açık `fixture_only` durumuyla raporlanır. Tek run evaluation artifact paketi üretilir.

### EV6 — Codec reconstruction evaluator

Reference PCM→codec→PCM metrikleri uygulanır; sessizlik, sample-rate ve hizalama fixture'ları test edilir. Üretilen skorlar referans seslerle dinlenerek spot check yapılır.

### EV7 — İlk otomatik Markdown raporu

`evaluation.json`, `samples.jsonl`, `metrics.jsonl`, `report.md` üretimi ve checksum doğrulaması tamamlanır. `not_measured` ve failed cases raporda eksiksiz görünür.

### EV8 — Compatibility gate ve pairwise comparison

İki evaluation için dataset, evaluator, input/sample ID ve metric tanımları eşleşir; uygun değilse `incomparable` dönülür. Test: uyumlu/uyumsuz fixture matrisi.

### EV9 — İstatistik ve cluster analizi

Paired bootstrap CI, sampling unit, seed repeats ve missingness raporu eklenir. Rastgele örnek seçimi sabitlenir; basit sayısal regression fixture'ları yazılır.

### EV10 — Blind listening import

Kör trial hazırlanır, cevaplar şemaya göre okunur, model kimlikleri güvenli tarafta çözülür. İlk sürümde ayrı büyük bir React paneli zorunlu değildir; CLI ile protokol yürütülebilir.

### EV11 — Runtime ve Live performance adapter

Speech/Live sağlayabildiği gerçek event'ler üzerinden TTFA, RTF, VRAM ve interrupt raporu alınır. Desteklenmeyen capability açıkça `not_measured` kalır.

### EV12 — Akademik araştırma kanıt paketi

Codec ve Speech modellerinden en az bir kontrollü karşılaştırma, doğru veri split'i, rapor, örnek çıktılar, metrik belirsizliği ve tezde kullanılabilir grafik tablolar üretilir.

## 31. Definition of Done: evaluator tamamlandı diyebilmek için

Bir evaluation özelliği **ancak** aşağıdaki koşulları sağlıyorsa tamamlanmış kabul edilir:

- [ ] Girdisi ve çıktısı şema ve versiyonla tanımlı.
- [ ] Model/codec/dataset/evaluator hash ve identity doğrulanıyor.
- [ ] Referans ses/metadata hakları korunuyor.
- [ ] Kod path'i doğru repoda; Speech model implementasyonu Forge'a kopyalanmamış.
- [ ] Ham örnek skorları ve başarısızlık nedenleri saklanıyor.
- [ ] `not_measured` ile gerçek `0` ayırt ediliyor.
- [ ] Agregasyon ve sampling unit açık.
- [ ] Karşılaştırılabilirlik gate'i uygulandı.
- [ ] Testler ve gerçek komutlar raporlandı; çalışmayanlar açıkça işaretlendi.
- [ ] CI kalite kapıları yeşil ya da eksik işaretleri dürüstçe raporlandı.
- [ ] Output artifact güvenli, atomik ve doğrulanabilir.
- [ ] Kullanılan donanım ve runtime sürümü kayıtta.
- [ ] Kullanıcıya sunulan rapor veri sızıntısı/PII içermiyor.
- [ ] Sonuçlar yalnız destekledikleri iddiayı ifade ediyor.

## 32. Codex çalışma protokolü

Forge Evaluation işini alan Codex şu sırayı izler:

1. Yerel workspace ve altı repo sınırını okur; tek repo erişimi varsa bunu bildirir.
2. `kaiwen-forge/AGENTS.md`, `README.md`, `TRAINING_PIPELINE.md`, `EXPERIMENTS.md`, bu belge ve workspace `ENGINEERING_STANDARDS.md` dosyalarını okur.
3. Git status ve mevcut implementation'ı tarar. Kullanıcının çalışmasını silmez.
4. Bir feature için en küçük çalışır dikey dilimi seçer, gerekirse ADR yazar.
5. Giriş/çıkış sözleşmesini ve en az bir başarılı/bir başarısız test örneğini kurar.
6. Speech tarafına değişiklik gerekiyorsa **ayrı repo/ayrı PR** ve consumer compatibility testi yapar.
7. Kod, test, doküman ve gerekliyse schema migration'ı aynı kapsamlı görevde günceller.
8. Testlerin gerçek çıktısını, çalıştırılmamış olanları ve ölçülmemiş performansı ayırır.
9. Veri ve model dosyalarını rastgele Git'e eklemez; kaynak/hash/izin korunur.
10. PR/commit kapsamını, doğrulama kanıtını, geri dönüş yolunu ve açık riskleri raporlar.

**Yasak örnekler:** Sessiz `torch.load` kolaycılığı, sahte metrik üretimi, disabled testleri başarı gibi gösterme, random split'i source-independent diye etiketleme, `loss` grafiğini tüm konuşma kalitesine eşitleme, model importunu GitHub repo rename'iyle aynı anda düşüncesizce değiştirme, Forge içine yeni Transformer yazma.

## 33. Karar verilmemiş konular ve seçme yöntemi

Aşağıdakiler ilk gün bağlayıcı teknoloji kararları değildir:

- MLflow veya başka üçüncü taraf experiment/evaluation backend'i.
- Büyük ölçekli dağıtık evaluator scheduler.
- PostgreSQL, Redis, Kafka, Kubernetes.
- Kalıcı değerlendirme dashboard'u veya hosted leaderboard.
- Ek insan değerlendirme sistemi sağlayıcıları.
- Harici öğrenilmiş kalite metriği ve ASR backend'i.
- Model tabanlı konuşma rubrik puanlama veya LLM-as-judge.
- Differential privacy, formal verification gibi ileri seçenekler.

Her aday önce **hangi gerçek problemi çözdüğü**, mevcut SQLite+JSONL ile neden çözülemediği, bakım ve lisans maliyeti, güvenlik, doğruluk, tek kullanıcılı akademik ihtiyaç ve benchmark etkisiyle değerlendirilir. Küçük bir proof-of-concept somut fayda göstermiyorsa yeni altyapı yalnız daha profesyonel göründüğü için eklenmez.

### 33.1 LLM-as-judge için özel uyarı

Bir LLM'nin konuşma değerlendirme rubriği açıklaması, otomatik ön eleme veya insanlara yardımcı anotasyon için deneysel kullanılabilir. **Temel araştırma sonucunun tek kanıtı olamaz.** Model sürümü, prompt, sampling, bias, Türkçe anlama kalitesi, gerçek ses yerine transcript kullanma kaybı ve değerlendirme sızıntısı kaydedilmelidir. Speech-native ses deneyimini yalnız yazıya çevirip başka bir LLM'ye puanlatmak aynı görevi ölçmez.

## 34. Kaynak haritası ve belgeler arası ilişkiler

| Belge | Kanonik sorumluluk |
|---|---|
| [Forge README](../README.md) | Proje tanımı, mevcut durum ve kullanım girişi |
| [Forge AGENTS](../AGENTS.md) | Codex için repo-özel çalışma kuralları |
| [Training Pipeline](TRAINING_PIPELINE.md) | Eğitim işinin güvenli yürütülmesi |
| [Experiments](EXPERIMENTS.md) | Deney tasarımı, metric registry, comparison, istatistiksel protokol |
| **Bu belge** | Evaluation engine, evaluator API, test setleri, kalite ölçümü ve kanıt üretimi |
| [Model Registry](MODEL_REGISTRY.md) | Doğrulanmış artifact'ın yaşam döngüsü ve release gate için mevcut tasarım sözleşmesi; uygulama henüz planlanmaktadır |
| [Speech Model](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/SPEECH_MODEL.md) | Token modelinin gerçek algoritmaları |
| [Speech Codec](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/CODEC.md) | Codec, RVQ, ses eğitim ve doğruluk sözleşmesi |
| [Speech Performance](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/PERFORMANCE.md) | TTFA, RTF, VRAM ve benchmark ölçüm semantiği |
| [Integration](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/INTEGRATION.md) | Corpus/Forge/Speech/Live/Agent sınırları |
| [Engineering Standards](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ENGINEERING_STANDARDS.md) | Test, kalite, güvenlik ve CI/CD ortak standardı |

**Belge kapanış ilkesi:** Kaiwen Forge yalnızca puan üreten bir araç değildir. Hangi puanın neyi ölçtüğünü, neyi ölçmediğini, hangi veriyle üretildiğini, neden karşılaştırılabilir olduğunu ve hangi koşullarda güvenilemeyeceğini açıklayan **kanıta dayalı araştırma altyapısı** olmalıdır. Başarı, modelin iyi görünmesinden değil; iyi, kötü ve belirsiz sonuçların aynı dürüstlükle ölçülüp saklanmasından doğar.
