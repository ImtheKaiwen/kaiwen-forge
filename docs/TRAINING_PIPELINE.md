# Kaiwen Forge — Training Pipeline, Execution Lifecycle ve Recovery Sözleşmesi

> **Belge:** `kaiwen-forge/docs/TRAINING_PIPELINE.md`  
> **Durum:** HEDEF TEKNİK TASARIM — henüz uygulanmış bir Forge training pipeline değildir.  
> **Kapsam:** V1 yerel tek-host orkestrasyon, çalışan Kaiwen Speech baseline'ına adapter, dayanıklı run/artifact kayıtları, Colab/Drive'dan doğrulanmış run import'u; genişleme noktaları.  
> **Kaynak gerçekliği:** 30 Ekim 2026 tarihinde GitHub `ImtheKaiwen/kaiwen-forge` reposu yalnızca kısa bir `README.md` içermektedir. Bu çalışma alanındaki ayrıntılı README ve AGENTS belgeleri hazırlanan taslaklardır, GitHub'daki uygulanmış kod değildir.  
> **Temel karar:** Önce **tek bir Speech CPU smoke eğitimini güvenle başlat, izle, iptal et, sakla ve yeniden üret**. Dağıtık platformu sonra değerlendir.

## 1. Belgenin amacı ve normatif dil

Bu belge Kaiwen Forge'un **bir training run'ının kontrol düzlemi** olarak ne yapacağını açıklar. Modelin nöral katmanlarını, loss hesaplamasını, optimizer algoritmasını veya codec eğitimini yeniden tasarlamaz. Bu işler [Kaiwen Speech](../../kaiwen-speech/README.md) uygulamasına aittir. Forge'un görevi; bir eğitim işinin girdilerini doğru tanımlamak, başlamadan doğrulamak, uygun çalıştırıcıya iletmek, çalışmasını gözlemek, kayıtları ve artifact'leri doğrulamak, başarısızlıkları dürüstçe bildirmek ve yeniden üretilebilirlik kanıtlarını korumaktır.

Bu dokümanda kullanılan etiketler:

- **MEVCUT / DOĞRULANMIŞ KOD:** Gerçek `kaiwen-speech` kaynak dosyalarında doğrulanan arayüz veya komut. Bu, komutun bu oturumda çalıştırıldığı anlamına gelmez.
- **HEDEF / TASARIM:** Henüz Forge içinde implementasyonu olmayan, sonraki iş paketlerini yönlendiren sözleşme.
- **ÖRNEK / ŞEMA TASLAĞI:** Gerçek parser veya import tarafından henüz kabul edilmeyen biçim. Örnek ID, SHA veya yol mutlaka gerçek çalışma anında çözülür.
- **DENEYSEL:** Karşılaştırmalı ölçüm yapılmadan varsayılan kabul edilmeyecek özellik.
- **ZORUNLU:** İlgili özellik hayata geçirildiğinde kabul testi olmadan tamamlandı sayılamaz.

Daha yüksek seviyedeki kararların kaynağı [Workspace Overview](../../docs/OVERVIEW.md), [Integration](../../docs/INTEGRATION.md), [Engineering Standards](../../docs/ENGINEERING_STANDARDS.md), [Roadmap](../../docs/ROADMAP.md), [Forge README](../README.md) ve [Forge AGENTS](../AGENTS.md) belgeleridir. Bu dosya yalnızca *training pipeline* konusunun ayrıntılarını sahiplenir; experiment analizleri, değerlendirme bilimsel protokolü ve registry promotion ayrı belgelerde ayrıntılandırılacaktır.

## 2. Sorumluluk ve sahiplik matrisi

| İşlev | Sahip | Forge'un rolü | Kesin sınır |
|---|---|---|---|
| Ham ses keşfi, indirme, lisans, etiketleme | Kaiwen Corpus | Export manifestini doğrular | İkinci downloader/annotation GUI yazılmaz |
| Waveform → codec tokenları | Kaiwen Speech | Uyumlu token cache'i seçer, kaydeder | Codec'i Forge içinde yeniden implementasyon yok |
| Codec encoder/RVQ/decoder eğitimi | Kaiwen Speech | Deneyi başlatır, çıktıları toplar | İkinci codec trainer yok |
| Temporal/Depth Transformer ve loss | Kaiwen Speech | Doğrulanmış backend çağrısı | Optimizer algoritması Forge'a kopyalanmaz |
| Recipe ve runtime snapshot | Kaiwen Forge | Tek yazarı ve schema sahibi | Eğitim başlarken snapshot değişmez |
| Job queue, worker, timeout, cancellation | Kaiwen Forge | Tam sahibi | İlk V1 tek host; Kubernetes zorunlu değil |
| Metrik toplama ve deney geçmişi | Kaiwen Forge | Tam sahibi | Model metriğinin hesaplamasını çarpıtmaz |
| Checkpoint içeriği ve formatı | Kaiwen Speech | Kopya, hash, metadata ve uyumluluk denetimi | Model state şemasını kendiliğinden değiştirmez |
| Artifact store/registry katalogu | Kaiwen Forge | Tam sahibi | Yetkisiz checkpoint'i Live'a yayınlamaz |
| Windows sesli kullanıcı arayüzü | Kaiwen Live | Release manifest tüketicisine veri sağlar | UI veya mikrofon Forge içinde yok |

**Bağımlılık yönü:** Forge, Speech'in sürümlü arayüzünü veya gerçek CLI'sını tüketir. Speech, Forge olmadan eğitim yapabilir. Bu ters yönlü bağımlılık konulmamalıdır. `kaiwen-forge` repo adı ile planlanan `kaiwen_forge` Python paketi farklı kavramlardır. Speech repo adının `kaiwen-speech` olması da bugün Python importunu `kaiwen_speech` yapmaz: mevcut `pyproject.toml` dağıtım adı `kaiwen-voice`, kaynak paket adı `kaiwen_voice` şeklindedir. Migration ancak testli, uyumluluk planlı ayrı PR ile yapılır.

## 3. Gerçek başlangıç ve ilk dikey dilim

30 Ekim 2026 GitHub incelemesine göre Forge'da henüz `pyproject.toml`, `src/kaiwen_forge`, çalışan CLI, worker veya SQLite kayıt motoru **yoktur**. Dolayısıyla `forge run`, `forge resume`, `RunManager` gibi isimler bu belgede yalnızca hedef arayüz olarak kullanılır.

Kaiwen Speech deposunda **mevcut dosyalar**:

- `scripts/train_continuation.py`: `--config`, `--synthetic` / `--token-cache`, `--overfit`, `--steps`, `--output-dir`, `--resume`, `--init-from`, `--init-depth-from` argümanları.
- `scripts/train_dialogue.py`: diyalog eğitimi, `--synthetic` / `--token-cache`, `--resume`.
- `configs/smoke.yaml`: iki codebook, düşük boyutlu 100-step CPU smoke yapılandırması.
- `src/kaiwen_voice/training/trainer.py`: optimizer, scheduler, validation, checkpoint, resume ve loglar.
- `src/kaiwen_voice/training/checkpoint.py`: `format_version=1`, model, optimizer, scheduler, scaler ve reproducibility metadata.
- `.github/workflows/test.yml`: Ruff, Pyright, pytest için CPU CI.

**MEVCUT Speech komutu** (Speech repo kökünde ve bağımlılıklar kurulu ortamda çalıştırılacak; bu belge hazırlanırken çalıştırılmadı):

```bash
uv sync --frozen --extra dev
uv run python scripts/train_continuation.py \
  --config configs/smoke.yaml \
  --synthetic \
  --steps 100 \
  --overfit \
  --output-dir /absolute/path/to/run/checkpoints
```

Windows PowerShell'de satır devamı `\` değil backtick (`) veya tek satır komut kullanılmalıdır; backend argüman dizisini kullanan Forge executor bu shell farkını kullanıcıya yansıtmaz.

**İlk Forge V1 vertical slice** şu kanıtları üretmedikçe tamamlanmış sayılmaz:

1. Reçete doğrulanır, uyumlu Speech kaynağına karşı çözülür.
2. Tek bir run immutable snapshot ile açılır ve SQLite'a kaydedilir.
3. `subprocess` sabit komut şablonuyla Speech eğitimini başlatır; stdout/stderr saklanır.
4. 100-step smoke sonucu gerçekten checkpoint üretir; yüklenebilirliği ve bütünlüğü denetlenir.
5. Süreç çıkış kodu, artifact sonuçları ve terminal durum **birlikte** kayda geçer.
6. Aynı işi tekrar gönderme, iptal, süreç çökmesi ve yeniden başlatma test edilir.
7. Kullanıcı run ID ile hangi kodun, konfigürasyonun ve ortamın kullanıldığını bulabilir.

Bu aşamayı geçmeden dağıtık job queue, web dashboard, GPU autoscaler veya genel amaçlı framework geliştirmek doğru değildir.

## 4. Kavramsal mimari ve veri akışları

```text
                     CLI / future UI
                           |
                     submit recipe
                           v
       +---------------------------+
       | Recipe validator/resolver |----> Corpus export + Speech config
       +---------------------------+              |
                           | immutable snapshot  | SHA, license, codec
                           v                     v
              +----------------------+   +---------------+
              | Run Store (SQLite)   |   | Input Validator|
              +----------------------+   +---------------+
                         |
                 transaction/queue
                         v
                +------------------+
                | Local Scheduler  |----> resource slot / lease
                +------------------+
                         |
                 process supervisor
                         v
                +------------------+
                | Speech Adapter   |----> real kaiwen_voice CLI
                +------------------+
                         |
                stdout/stderr, exit, checkpoints
                         v
        +--------------------------------------+
        | Metrics + Artifact Verification      |
        +--------------------------------------+
                 |                 |
                 v                 v
           Run database     Local artifact store
                 |
                 v
       evaluations / remote import / registry
```

**Control plane:** recipe çözümleme, doğrulama, state machine, kuyruk, audit, katalog ve yetki. **Execution plane:** gerçek Speech process ve eğitim GPU/CPU'su. **Data plane:** Corpus export, token cache, checkpoint, metrik ve değerlendirme dosyaları. Bu ayrım iş başlatma hatası ile model eğitimi hatasının ayrı raporlanmasını sağlar.

Başlangıçta aynı makinede tek Python process ve bir worker process yeterlidir. Modüler sınır, ayrı microservice açmayı zorunlu kılmaz. V1 için sıkı bağlantılı, test edilebilir küçük modüller tercih edilir.

## 5. Önerilen paket bileşenleri ve oluşturulma sırası

**HEDEF kaynak yapısı** (bugün repo içinde yoktur):

```text
src/kaiwen_forge/
  __init__.py
  cli.py
  config.py
  domain/
    recipes.py           # schema, frozen resolved recipe
    runs.py              # state machine, attempt identity
    artifacts.py         # artifact catalog types
  datasets/
    manifest.py          # Corpus export validation
  orchestration/
    scheduler.py         # bounded local queue + single worker
    runner.py            # local process supervisor
    lifecycle.py         # allowed state transitions
    cancellation.py      # graceful/forced termination
  integrations/
    speech/
      adapter.py         # Speech CLI allowlist mapping
      compatibility.py   # codec and checkpoint checks
  experiments/
    store.py             # SQLite WAL, transactional writes
    metrics.py           # structured metrics/event ingestion
  artifacts/
    store.py             # atomic staging, sha256, retention
  evaluation/
    runner.py            # post-run orchestration, no fake metrics
```

İlk PR yalnızca package skeleton + unit test + CI içerebilir. Sonraki küçük PR'larda recipe parser, run state persistence, fake executor, Speech adapter sırasıyla gelir. Başlangıçta tüm dosyaları boş interface'lerle doldurmak gerekmez. Her yeni modül gerçek bir ihtiyaç veya test fixture ile doğmalıdır.

## 6. Domain kimlikleri: recipe, experiment, run, attempt, artifact

- `recipe_id`: insan tarafından düzenlenebilir deney tarifi adı; tek başına immutable kimlik değildir.
- `recipe_version`: tarife ait semantik/schema sürümü; Git commit veya içerik özetiyle sabitlenir.
- `resolved_recipe`: tüm yolları, varsayılanları, environment etkileşimlerini çözülmüş, **değiştirilemez** çalışma girdisi.
- `experiment_id`: ortak amaçla gruplanan run'lar (ör. parallel vs depth karşılaştırması).
- `run_id`: bir deneydeki belirli veri/model/konfigürasyon kombinasyonunun kalıcı mantıksal kaydı.
- `attempt_id`: aynı mantıksal run'ın bir başlatma/kurtarma denemesi. Yeni process her zaman yeni attempt kaydına sahiptir.
- `job_id`: kuyruktaki yürütme isteği; V1'de run/attempt ile bire bir tutulabilir, yine de semantik fark kayda geçer.
- `artifact_id`: içerik hash'i ve üretici referansı olan nesne kaydı; run dosya adından bağımsızdır.
- `idempotency_key`: kullanıcı veya çağıranın aynı `submit` isteğini tekrarında duplicate iş açılmasını önleyen kapsamlı anahtar.

**Karar:** Tamamlanmış run'ın geçmişi silinmez. Resume aynı run altında yeni attempt kaydıyla veya açık bir `resumed_from` zinciriyle yürür; eski attempt metadata'sının üzerine yazılmaz. `run_id`, her zaman birden fazla denemenin sonuçlarından hangisinin otoritatif olduğuna işaret eden denetlenebilir rapor taşımalıdır.

Kimliklerde rastgele UUID/ULID, sortable alternatifler veya açık namespace değerlendirilebilir. Sadece zamana göre üretilmiş dosya adı çakışma ve tahmin edilebilirlik riski nedeniyle otoritatif run ID olamaz. Kimliklerin serileştirilmesi, dosya adlarına kaçış/normalizasyon ve güvenli path doğrulaması gerekir.

## 7. Recipe şeması ve resolved snapshot

**HEDEF şema taslağı** aşağıdadır. Forge parser/CLI henüz yoktur; örnek değerler çalışma anında gerçek sürüm, mutlak yol ve digest ile değiştirilir.

```yaml
schema_version: "1.0"
recipe_id: "speech-cpu-smoke"
family: "speech"
backend: "kaiwen-speech"
source:
  git_repository: "https://github.com/ImtheKaiwen/kaiwen-speech"
  git_commit: "<resolve-at-launch>"
input:
  mode: "synthetic"
  dataset_manifest: null
  dataset_digest: null
  split: null
  codec_id: null
  codec_revision: null
model:
  config: "../kaiwen-speech/configs/smoke.yaml"
  task: "continuation"
training:
  seed: 42
  max_steps: 100
  precision: "fp32"
  overfit: true
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

Bu örnekte `model.config` içindeki `training.device: auto` ile recipe'deki `runtime.device: cpu` farklı katman ayarlarıdır. V1 adapter **etkili cihaz kararını** backend'e nasıl uyguladığını açıkça kanıtlamadan `cpu` garantisi veremez. Speech CLI şu anda `--device` training override'ı sunmamaktadır; CPU seçimi doğrulanmış bir `smoke.yaml` kopyası/çözülmüş config üzerinden yapılmalı veya Speech'e küçük, testli bir CLI cihaz override'ı eklenmelidir. `CUDA_VISIBLE_DEVICES` ile ortamı sınırlamak da ayrı platform testleri gerektirir. Forge `runtime.device` alanını tek başına gerçek device pinning saymamalıdır.

### 7.1 Ayrıntılı doğrulama tablosu

| Alan | Kontrol | Başarısızlık davranışı |
|---|---|---|
| `schema_version` | Desteklenen sürüm ve required field'lar | `UNSUPPORTED_RECIPE_VERSION` |
| `source.git_commit` | Gerçek SHA, çalışma kaynağıyla eşleşme | `SOURCE_REVISION_MISMATCH` |
| `family/backend` | V1 allowlist: Speech | `UNSUPPORTED_BACKEND` |
| `model.config` | Mevcut dosya, parse edilebilir YAML, hash | `MODEL_CONFIG_INVALID` |
| `model.task` | `continuation` veya `dialogue`; CLI ile uyumlu | `UNSUPPORTED_TASK` |
| `input.mode` | Synthetic veya gerçek dataset; ikisi birden değil | `INVALID_INPUT_MODE` |
| `dataset_digest` | Corpus manifestinin gerçek byte hash'i | `DATASET_INTEGRITY_FAILED` |
| `codec_revision` | Kaynak token cache ve checkpoint uyumluluğu | `CODEC_MISMATCH` |
| `training.max_steps` | Pozitif; checkpoint step ile tutarlı | `INVALID_TRAINING_STEPS` |
| `training.precision` | Device/backend tarafından destekleniyor | `PRECISION_NOT_SUPPORTED` |
| `runtime.output_root` | Sandbox içerisinde, güvenli ve yazılabilir | `OUTPUT_PATH_INVALID` |
| `max_wall_time_minutes` | Pozitif, üst sınır kontrollü | `INVALID_TIMEOUT` |

`input.mode: synthetic` için dataset manifesti/codec identity'si boş bırakılabilir; bu yol sadece yapısal smoke testidir, gerçek Türkçe ses kalitesini kanıtlamaz. Gerçek token-cache eğitiminde codec identity ve veri kökeni zorunludur. Başarısız validation, eğitim başlatılmadan gerçekleşir; parse başarısızlığı bile audit/diagnostic olayı olarak kaydedilir, ama geçersiz kayda `validated` statüsü verilmez.

### 7.2 Snapshot ve override kuralları

1. Kullanıcı YAML dosyasının içeriği **orijinal kaynak** olarak kaydedilir.
2. Environment, working directory, relative path ve izinli CLI override'ları çözülür.
3. Typed canonical `resolved_recipe` UTF-8 biçiminde serileştirilir; credential değerleri redakte edilir.
4. Canonical bytes üzerinden SHA-256 `recipe_digest` hesaplanır; JSON canonicalization kuralı şemada sabitlenir.
5. Model config ayrı hash'e ve copy'ye sahiptir. Başlatıldıktan sonra kaynak config düzenlense bile run değişmez.
6. `git_commit`, paket sürümleri, dirty tree durumu, dependency lock ve codec/dataset digest kayda alınır.
7. Resolved snapshot ve ilk run kaydı aynı atomik oluşturma akışında kalıcı hâle gelir; biri başarılı diğeriyse kayıp olamaz.

İş başlamadan önce doğrulanmayan placeholder SHA, internetten sessiz çekilen "latest" model veya otomatik değişen veri bölmesi kabul edilmez. Reçeteye sonradan eklenen environment override'ı snapshot dışına kaçamaz.

## 8. Run state machine ve izinli geçişler

**HEDEF V1 durumları:**

```text
created --validate--> validated --enqueue--> queued --claim--> running
   |                    |                       |           |
   +--> rejected        +--> rejected          +--> cancelled
                                                            |
                        +-----------------------------------+-----------------+
                        |                   |               |                 |
                    completed             failed        interrupted       cancelling
                                                                            |
                                                                        cancelled
```

Uygulama için doğruluk, şeklin görünümünden değil **tek merkezî geçiş tablosundan** gelir:

| Önceki durum | İzinli sonraki durum | Zorunlu kanıt |
|---|---|---|
| `created` | `validated`, `rejected` | Recipe/contract doğrulama sonucu |
| `validated` | `queued`, `rejected` | Immutable snapshot ve queue kaydı |
| `queued` | `running`, `cancelled` | Atomik claim / persisted cancel |
| `running` | `completed`, `failed`, `interrupted`, `cancelling` | Process + artifact veya hata/timeout kanıtı |
| `cancelling` | `cancelled`, `failed`, `interrupted` | OS termination doğrulaması veya failure report |
| `completed` | Hiçbiri | Immutable terminal sonuç |
| `failed` | Hiçbiri | New attempt ile recovery gerek |
| `interrupted` | Hiçbiri | New attempt ile recovery gerek |
| `cancelled` | Hiçbiri | New attempt/explicit resubmit gerek |
| `rejected` | Hiçbiri | Geçerli yeni submission gerek |

**İnce ayrım:** Terminal run kaydı ile attempt yaşam döngüsü karışmamalıdır. Basit V1'de üstteki durumlar *attempt* statüsü olarak uygulanabilir; run'ın görüntülenen durumu son otoritatif attempt'ten türetilir. Yeni resume attempt'i `interrupted` attempt'inin üzerine geçmez. Aynı durumu hem run hem attempt tablosunda saklayıp çelişen iki otorite oluşturmayın.

`running -> completed` yalnızca şu üç kapının tamamından geçildiğinde olur: (1) alt sürecin doğrulanmış başarılı exit code'u; (2) zorunlu artifact'lerin varlığı, boyut/hash ve format doğrulaması; (3) recipe'nin gerektirdiği validation/evaluation kapılarının sonucu. Sadece `exit_code=0`, sadece `latest.pt` dosyasının varlığı veya GUI'nin yeşil görünmesi kanıt değildir.

### 8.1 Idempotency ve yarış koşulları

- Aynı `(submitter, idempotency_key)` aynı canonical isteğe karşı aynı run'ı döndürür.
- Aynı anahtar farklı payload ile gelirse `IDEMPOTENCY_CONFLICT` döner.
- Worker claim işlemi SQLite transaction içinde yapılır; iki worker aynı kuyruğu aynı anda alamaz.
- Claim transaction ile OS process başlangıcı arasındaki pencere ayrı bir `attempt` state ve lease ile kurtarılabilir olmalıdır.
- Geç event, iptal edilmiş attempt'i `completed` yapamaz. Her event `run_id + attempt_id + sequence_number` ile kontrol edilir.
- Yeni attempt'in checkpoint klasörü ayrı olmalı veya writer fencing token uygulanmalıdır.

### 8.2 State event journal

Her geçiş en az `timestamp_utc`, `run_id`, `attempt_id`, `previous_state`, `next_state`, `actor`, `reason_code`, `request_id` ve monoton `sequence` içerir. Event journal, SQLite state alanıyla aynı transaction içinde kalıcı olmalıdır. Eventlerin olay zamanı ile DB'ye yazılma zamanı farklıysa ikisi ayrı tutulur.

## 9. SQLite WAL ve dayanıklı run store

**HEDEF V1** için SQLite yeterlidir. V1'de tek makine/tek sahibi olan writable veritabanı; büyük binary checkpoint'ler veritabanında değil artifact store'da saklanır. Çok host üzerinden network filesystem'deki aynı SQLite dosyasına güvenilmez. Colab çıktıları uzaktan dosya halinde gelir, yerelde kontrollü import edilir.

Önerilen tablolar:

```text
experiments(experiment_id, name, created_at_utc, description)
runs(run_id, experiment_id, recipe_digest, created_at_utc,
     source_git_sha, dataset_digest, latest_attempt_id, run_result)
attempts(attempt_id, run_id, state, queued_at_utc, started_at_utc,
         ended_at_utc, executor_kind, owner_id, lease_epoch,
         pid, process_start_fingerprint, heartbeat_at_utc,
         exit_code, termination_reason, resumed_from_attempt_id)
run_events(event_id, run_id, attempt_id, sequence, at_utc,
           from_state, to_state, actor, reason_code, payload_json)
metrics(run_id, attempt_id, step, name, value, recorded_at_utc, source)
artifacts(artifact_id, run_id, attempt_id, kind, uri,
          sha256, size_bytes, state, verified_at_utc)
idempotency_keys(submitter, key, request_digest, run_id)
```

**Şema taslağıdır, hazır SQL migration değildir.** Gerçek DDL'de `FOREIGN KEY`, `UNIQUE`, `CHECK`, indeks, timestamp UTC ve schema migration versiyonu açıkça uygulanır. `PRAGMA foreign_keys=ON`, WAL ve makul `busy_timeout` bağlantı başına kurulmalıdır; `BEGIN IMMEDIATE` ile claim transaction stratejisi test edilmelidir.

### 9.1 Crash consistency

- `resolved_recipe` dosyası geçici dosyaya yazılır, flush/fsync ve aynı dosya sistemi üzerinde atomic replace ile tamamlanır.
- SQLite kayıtları transaction ile commit edilir; disk üzerindeki snapshot ile DB ilişkisi recovery sırasında reconcile edilir.
- DB'ye henüz kaydı eklenmemiş bir artifact temp dosyası garbage collection'a adaydır; kesinlikle sessizce resmi artifact sayılmaz.
- SQLite checkpoint/WAL retention, backup ve integrity check süreci bulunur. WAL dosyasını canlı veritabanının yanından elle silmek yasaktır.
- `PRAGMA integrity_check` ile uygulama-seviyesi artifact hash doğrulaması farklı şeylerdir; ikisi de önemlidir.
- Schema migration backup alır, eski schema sürümünü reddetme veya açık upgrade yolunu tanımlar; sessiz yıkıcı migration yoktur.

## 10. Kuyruk, scheduler ve kaynak slotları

İlk V1'de **tek aktif eğitim** ve sınırlı bekleyen queue varsayılanı yeterlidir. Scheduler'ın görevi, uygun kaynak kullanılabilir olduğunda doğrulanmış bir attempt'i local executor'a teslim etmektir; model batch içindeki GPU scheduling'e müdahale etmez.

Önerilen ilk kaynak profili:

```yaml
# HEDEF operator config; şu an Forge tarafından okunmaz.
scheduler:
  max_running_total: 1
  max_pending: 16
  queue_policy: fifo
  claim_lease_seconds: 60
  heartbeat_interval_seconds: 10
  startup_timeout_seconds: 120
  graceful_shutdown_seconds: 15
resources:
  allow_gpu_concurrency: false
  preflight_min_disk_gib: 2
  gpu_reservation: "exclusive"
```

Bu sayıların hepsi **başlangıç politikası önerisidir**, benchmark'la gerekçelendirilmiş cihaz garantisi değildir. Sıra boyutu ve disk eşiği kullanıcı workload'una göre değişebilir.

### 10.1 GPU kaynağı atama

- İş başlamadan önce cihazın mevcut olup olmadığı ve backend tarafından desteklenip desteklenmediği doğrulanır.
- GPU lease owner, device id, started_at, lease expiration ve fencing epoch ile kaydedilir.
- Aynı Forge instance içindeki istemeden çift GPU eğitimi engellenir. OS'deki başka programların VRAM tüketimini Forge bütünüyle kontrol edemez.
- `CUDA_VISIBLE_DEVICES`, backend'e gönderilen device config ve PyTorch'un gördüğü cihaz indeksleri tutarlı tutulur. Sadece değişken ayarlamak kesin GPU izolasyonu değildir.
- CPU smoke ve GPU gerçek eğitim işleri farklı kaynak class'ları olarak işaretlenebilir; V1'de fairness karmaşıklığı zorunlu değildir.
- Memory estimate ile ölçülen `torch.cuda.max_memory_allocated` / `max_memory_reserved` ayrıdır. OOM durumunda sessizce batch size azaltıp *aynı recipe*'yi devam ettirmek yasaktır.

### 10.2 Disk ve log sınırları

Çıktı dizini işten önce yazılabilirlik ve boş alan açısından kontrol edilir. Checkpoint yazımı sırasında disk dolması `failed`/`interrupted` olarak kaydedilir, son sağlam checkpoint korunur. Log rotasyonu ve disk üst sınırı uygulanır; ancak eğitim hata kanıtını silen döngüsel log saklama kabul edilmez. Retention politikasının zorunlu kanıtları ve yedekleri koruması gerekir.

## 11. Local process supervisor sözleşmesi

**HEDEF V1:** Python `subprocess.Popen` ile açık argüman dizisi kullan; string shell komutunu `shell=True` ile yürütme. Working directory, Python interpreter, env allowlist, timeout, stdout/stderr path ve child tree açıkça tanımlanır.

Örnek mantıksal arayüz (**pseudocode; çalışan Forge API'si değildir**):

```python
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True)
class ProcessSpec:
    argv: tuple[str, ...]
    cwd: Path
    environment: dict[str, str]
    stdout_path: Path
    stderr_path: Path
    wall_timeout_seconds: int

# Gerçek implementasyon:
# - subprocess.Popen(list(spec.argv), cwd=spec.cwd, env=...,
#                    stdin=DEVNULL, stdout=..., stderr=..., shell=False)
# - process ownership + heartbeat kalıcı kaydedilir
# - terminate/kill platforma uygun ve testli yapılır
# - process exit kodu artifact doğrulamasıyla birleştirilir
```

`argv` kullanıcıya açık arbitrary command değildir; Speech adapter, bilinen script ve sınırlı argümanlardan inşa eder. `uv`, interpreter, çalışma dizini ve repo SHA gerçek ortamdan çözülür. Repo checkout'u kullanıcı tarafından değiştirilebilir bir dizindeyse güven politikası uygulanır; otomatik remote script download edip çalıştırma yoktur.

### 11.1 İzinli komut eşlemesi

| Recipe task | Gerçek Speech entrypoint | V1 doğrulama |
|---|---|---|
| `continuation` | `scripts/train_continuation.py` | `--synthetic` XOR `--token-cache`; config hash |
| `dialogue` | `scripts/train_dialogue.py` | `--synthetic` XOR `--token-cache`; dataset kind |
| codec training | Henüz kendi Kaiwen Codec CLI'sı yok | **UNSUPPORTED**; sahte script uydurma |
| real-time streaming training | Henüz ayrı trainer yok | **UNSUPPORTED** |
| evaluation | Mevcut Speech değerlendirme scriptleri sınırlı | Sürümle destek matrisi şart |

Adapter hiçbir zaman CLI bayrağının varlığını varsaymamalıdır. `--resume` iki gerçek training script'inde bulunur, `--init-from` ve `--init-depth-from` ise mevcut continuation script'inde bulunmaktadır; dialogue script'ine aynı argümanlar otomatik eklenmez.

### 11.2 Process identity

Sadece PID saklamak yeniden açılan işletim sisteminde PID reuse yüzünden güvenli değildir. Attempt ownership için process başlangıç zamanı/OS fingerprint, parent worker id, lease epoch ve expected argv hash'i gibi ek kanıt tutulur. Forge restart olduğunda aynı PID'nin başka uygulamaya ait olduğu anlaşılırsa o process'e sinyal gönderilmez.

### 11.3 Log ve heartbeat

Backend standart çıktısı yüksek hacimde gelebilir. stdout/stderr ayrık dosyalara akıtılır veya bounded reader ile tüketilir; pipe dolması nedeniyle çocuk process'in kitlenmesine izin verilmez. Heartbeat, bir log satırının gelmesine bağımlı olmamalıdır: sessiz çalışan eğitim geçerli olabilir. `last_output_at` ile `worker_alive_at` ayrı tutulur. İnsan tarafından okunabilir log, yapılandırılmış olay protokolü yerine geçmez.

## 12. Speech adapter: çalışabilen ve hedef davranışlar

**MEVCUT:** `kaiwen_voice` paketinde `KaiwenSpeechTransformer`, `SpeechTransformerConfig`, `Trainer`, `TokenCacheReader`, `load_checkpoint`, `warm_start_depth_from_parallel`, `warm_start_depth_with_delay` kaynak kodu bulunur. Ancak Forge bunların hangisinin stabil public API olduğunu Speech ile birlikte kararlaştırmalıdır; en düşük entegrasyon riski gerçek CLI giriş noktalarını kullanmaktır.

**HEDEF adapter operasyonları:**

```text
capabilities()             -> backend version + supported tasks
validate(recipe, inputs)   -> typed diagnostic list
prepare(attempt)           -> validated process spec + artifacts root
launch(spec)               -> process ownership handle
observe(handle)            -> exit, heartbeat, log/metrics events
collect(attempt)           -> discovered checkpoint + manifest
check_resume(checkpoint)   -> compatible or explicit reject
```

İsimler API taslağıdır. V1'de tek adapter sınıfı ve doğrudan modüller yeterli olabilir. Farklı backend'ler gerçekten gerektiğinde plugin registry eklenir; ilk günden dinamik import ve karmaşık class factory altyapısı kurulmaz.

### 12.1 Mevcut eğitim logunun sınırları

Speech `Trainer._log()` insan-okur formda örneğin `step=10 train_loss=... lr=... grad_norm=...` çıktısı yazabilir. Forge regex veya key/value parser ile metrikleri alabilir; fakat bunun **sürüm sabitlenmiş geçici adapter** olduğu açıkça yazılmalıdır. Speech'e küçük bir `--metrics-jsonl` gibi opsiyonel yapılandırılmış çıktı eklenmesi uygun bir ayrı PR olabilir. Yapılandırılmış metrik olmadan `loss=...` tekstinden grafik üretmek kırılgan ve eksik logları başarı gösterebilir.

Metrik parser, bozuk satırı raw logdan silmez; parse hatasını ayrı event olarak raporlar. `train_loss`, `val_loss`, `grad_norm`, `lr` alanları gözlemlenen gerçek değerlerdir. Olmayan `wer`, `mos` veya `rtf` metriği üretilmiş gibi gösterilemez.

### 12.2 Exit code ile gerçek başarı ayrımı

Başarılı process çıkışı, bütün zorunlu dosyaları yazdığı anlamına gelmez. Örneğin `checkpoint_every` beklenmedik ayarlarda veya disk hatasında çıktı eksik kalabilir. V1 kontrolü:

1. Process gerçekten `0` ile çıktı mı?
2. Son checkpoint dosyası bulunuyor mu ve byte boyutu sıfırdan büyük mü?
3. Dosya `.tmp` değil, final yolunda mı?
4. SHA-256 hesaplandı mı, kayıttaki beklenen hash ile eşleşiyor mu?
5. Speech `load_checkpoint(..., device="cpu")` ile model/config/step doğrulanıyor mu? **Yalnızca güvenilir iç üretim checkpoint'i için.**
6. `step` recipe'de beklenen son adımla uyumlu mu?
7. Gerekli test/evaluation varsa geçti mi?

Bu kontrolleri geçen iş `completed` olabilir. Kontrol başarısızsa **exit 0 olsa da** `failed` veya açıklayıcı ayrı validation-failure sonucu kaydedilir.

## 13. Dataset handoff, token cache ve provenance

Corpus data ingest/annotation/export'un tek sahibidir. Forge yalnızca immutable export'u tüketir. Dataset manifesti, her parçanın stable ID'si, kaynak/lisans durumu, ses formatı, konuşmacı/split bilgisi, segment referansı, checksum ve üretim pipeline sürümüyle ilişkilendirilmelidir.

**İki farklı veri düzeyi** karıştırılmaz:

1. **Corpus export:** waveform/segment manifest, kayıt kökeni, haklar ve kalite etiketleri. Büyük ham ses her seferinde RAM'e yüklenmez.
2. **Speech token cache:** codec ile üretilmiş `.pt` shard'ları, `format_version: 1`, codec metadata ve token kayıtları. Bunlar belirli codec revision'a bağlı türetilmiş çıktılardır.

Mevcut Speech `TokenCacheWriter` bounded `real_*.pt` / `dialogue_*.pt` shard'ları üretir; `TokenCacheReader` metadata eşitliği ve token aralığı denetler. **Aynı Q/K/frame rate olsa bile farklı neural codec checkpoint'lerinin ID semantiği değişir.** Yeni codec yayınında eski Mimi token cache'lerini dönüştürülmüş saymak veya aynı eğitime karıştırmak yasaktır.

### 13.1 Devamlılık ve veri sızıntısı

Validation/test split kayıtları yalnızca shard sırasına göre ayrılmamalıdır. Speech'teki mevcut `split_token_records` sabit record ID hash'i kullanır; bu, aynı kişinin farklı kayıtlarda bulunmasını veya aynı konuşmanın bölümlerinin ayrılmasını otomatik engellemez. Corpus'taki kaynak/konuşmacı/grup bilgileri ile daha güçlü leakage kontrolü ayrıca gerekir.

Preflight kontrolleri: manifestte belirtilen kayıt sayıları ve duration toplamları tutarlı mı; silinmiş kaynak veya yanlış path var mı; ses dosyalarının digest'i beklenenle eşleşiyor mu; split'ler tekrar üretiliyor mu; güvenilmeyen/veri hakkı belirsiz kayıtlar dışlanıyor mu? Bunlardan biri belirsizse Forge eğitime sessizce başlamaz.

### 13.2 24 kHz ve 16 kHz ayrımı

Corpus'un eski işlem hattında 16 kHz PCM normalizasyon bulunur. Speech/Mimi araştırma yolu 24 kHz WAV ister. **16 kHz dosyayı 24 kHz'e resample etmek yüksek frekans bilgisini geri kazandırmaz.** Codec eğitimi için mümkünse orijinal yeterli örnekleme hızlı arşiv kaynağı kullanılmalı, dönüştürmenin provenance kaydı saklanmalıdır. `sample_rate` etiketi dönüştürülen dosyanın gerçek örnekleme hızıyla eşleşmelidir.

## 14. Reproducibility ve çevre snapshot'ı

Tekrarlanabilirlik `seed=42` yazmaktan ibaret değildir. Forge her run için şu kanıtları toplar:

- Speech ve Forge source Git SHA, dirty worktree durumu ve varsa diff digest (özel diff içeriği otomatik yayımlanmaz).
- Kullanılan `uv.lock` / dependency freeze, Python/PyTorch/CUDA/cuDNN sürümleri.
- CPU/GPU model adı, görünen CUDA device index, desteklenen precision, ortam platformu.
- Dataset version + manifest digest + split politikası, codec model id/revision + token cache digest.
- Orijinal ve resolved recipe, model config, tokenizer/control token sürümü.
- Başlangıç/bitiş zamanları, step, train/val metrikleri, checkpoint hash ve test suite sürümü.
- Deterministic algorithm setting, random seeds, sampler/split state ve nondeterminism istisnaları.
- Uygulanan `resume` veya `warm-start` kararı, kaynak artifact ve değişiklik raporu.

**ÖRNEK environment snapshot şeması** (doğrulama için, gerçek sonuç değil):

```json
{
  "schema_version": "1.0",
  "record_kind": "environment_snapshot",
  "run_id": "example-run-001",
  "attempt_id": "example-attempt-001",
  "source": {
    "forge_commit": "REPLACE_WITH_ACTUAL_SHA",
    "speech_commit": "REPLACE_WITH_ACTUAL_SHA",
    "speech_package": "kaiwen-voice"
  },
  "runtime": {
    "platform": "example-linux",
    "python": "REPLACE_WITH_ACTUAL_VERSION",
    "torch": "REPLACE_WITH_ACTUAL_VERSION",
    "device": "cpu",
    "precision": "fp32"
  },
  "data": {"mode": "synthetic", "dataset_digest": null},
  "recorded_at_utc": "2026-10-30T00:00:00Z"
}
```

Geliştirici bu JSON'daki örnek SHA/sürümü değiştirip gerçek metrikmiş gibi saklayamaz. Otomatik snapshot üretiminde ilgili bilgi alınamıyorsa `unknown`/`null` ve açık uyarı kullanılır. Makine hostname, kullanıcı yolu, private token gibi hassas alanlar redakte edilir.

**Determinism sınırı:** Farklı GPU kernel, CUDA/cuDNN, reduced precision, paralel CPU iş parçacığı ve Colab imajları bire bir bit-level sonuç üretmeyebilir. Forge başarıyı «her ortamda aynı byte checkpoint» gibi garantilerle tanımlamaz; hangi koşullarda tekrar üretim beklediğini yazar, makul toleranslar ve bilimsel karşılaştırma protokolü kullanır.

## 15. Checkpoint sözleşmesi ve yazma güvenliği

Speech'teki mevcut `save_checkpoint()` `format_version: 1` içeren `.pt` sözlüğü yazar; `step`, model config/state, optimizer, scheduler, scaler, metrics, training config ve reproducibility metadata içerebilir. `latest.pt`, `best.pt` ve `step_XXXXX.pt` isimleri mevcut Trainer tarafından kullanılır. Forge bu dosyaların yazarı değildir, doğrulayıcı ve kataloglayıcısıdır.

### 15.1 Artifact writer ve verifier ayrımı

- Speech training process checkpoint'i **kendi run sandbox'ına** geçici dosya ardından atomic replace ile yazar.
- Forge tamamlanmış dosyayı görür, yazma bitmiş mi kontrol eder, SHA-256/size/format metadata çıkarır.
- Artifact store'a transfer edildiğinde tekrar hash hesaplar; doğrulama olmadan `verified` durumuna geçmez.
- Bir dosyanın `latest.pt` adıyla yeniden yazılması, run tarihi boyunca hash'in sabit kaldığı anlamına gelmez. Forge her snapshot'ı unique ID veya content-addressed URI ile kataloglar.
- Temel metadata `checkpoint_step`, `model_config_digest`, `codec_id`, `source_run_id`, `origin_sha` içerir. Bunlar checkpoint içine gömülü değilse bağlı run manifestinden türetildiği açıkça işaretlenir.
- Checkpoint path'i otomatik model registry «released» statüsü vermez.

### 15.2 Güvenli checkpoint yükleme

Mevcut Speech loader'ları `torch.load(..., weights_only=True)` kullanır. Bu önemli ama tek başına her yabancı dosyanın güvenli olduğunu veya doğru model tarafından üretildiğini **kanıtlamaz**. Forge, dışarıdan gelen `.pt` dosyalarını körlemesine deserialize etmez. Güvenilir kaynak + SHA/manifest + format/size limit + izolasyon stratejisi gerekir. Yayına çıkacak Live inference paketi optimizer state içermez; ayrı ve daha dar bir inference artifact üretilir.

### 15.3 Resume ile warm-start teknik olarak farklıdır

| Özellik | Resume | Warm-start |
|---|---|---|
| Amaç | Kesilmiş aynı eğitime devam | Başka konfigürasyona ağırlık aktarımı |
| Run lineage | Aynı mantıksal run, yeni attempt | Yeni run ve `warm_started_from` ilişkisi |
| Optimizer | Uygunsa geri yüklenir | Çoğu durumda sıfırdan |
| Scheduler/scaler | Checkpoint uyumluysa geri yüklenir | Yeniden tanımlanır |
| Model config | Tam uyum aranır | Açık izinli kısmi farklar |
| Metrik kıyaslama | Step sürekliliği | Yeni deneme olarak rapor |

Speech'in `train_continuation.py` giriş noktası `--resume`, `--init-from` ve `--init-depth-from` seçeneklerini birbirini dışlayan biçimde işler. `--init-from` parallel→depth warm-start ve `--init-depth-from` depth konfigürasyonunda acoustic delay farkıyla geçiş gibi sınırlı senaryolar içindir. Bu kurallar doğrulanmış Speech kaynak kodundan türetilmiştir; Forge yeni bir genel checkpoint migrator yazmış gibi davranmaz.

### 15.4 Kesintide resume için zorunlu kontrol

Son sağlam checkpoint listesini **en yüksek dosya adı/mtime** üzerinden seçme. Verified manifestte `step`, hash, model config ve codec identity tutarlı olmalı. `training.steps` kaynak step'ten büyük olmalı. Speech Trainer mevcut global step'i yükler; yeni `steps` değeri zaten tamamlanmışsa hata verir. `validation_fraction`, split seed ve dataset revision bilinçsiz değişmemeli; optimizer/scheduler schedule değişiklikleri ayrı policy ile raporlanmalıdır.

**Önemli gerçekçi sınır:** Mevcut Speech checkpoint payload'ında RNG ve DataLoader iteration state'in tam geri yüklenmesini garanti eden ayrı, doğrulanmış bir kontrat yoktur. Bu nedenle mevcut resume'a **bit düzeyinde kesintisiz aynı eğitim izi** garantisi verilemez. Forge eksik state'i «restore edildi» diye raporlayamaz; çözüm için Speech'te kontrollü API/format geliştirmesi gerekir.

## 16. Metrics pipeline ve kayıpsız gözlem

**HEDEF metrik event şeması**:

```json
{
  "schema_version": "1.0",
  "kind": "training.metric",
  "run_id": "example-run-001",
  "attempt_id": "example-attempt-001",
  "sequence": 12,
  "step": 100,
  "name": "train_loss",
  "value": 1.234,
  "unit": null,
  "source": "speech-trainer",
  "timestamp_utc": "2026-10-30T00:00:00Z"
}
```

Metriklerin her biri gerçek kaynakları ve birimleriyle tutulur. `step`, modelin global optimizer step'i mi batch sayısı mı, schema ile tanımlanır. Evaluation ölçümleri farklı namespace'te (`eval/…`) raporlanır; eğitim loss'u inference konuşma kalitesiymiş gibi adlandırılmaz. Scalar sample, histogram veya ses örneği aynı olay gibi kaydedilmez.

### 16.1 Metrik doğrulama

- `nan`/`inf` gibi sonlu olmayan değerler başarılı metrik olarak kayıt edilmez; ayrı hata event'i oluşturulur.
- Aynı `(run_id,attempt_id,step,name,source)` için yinelenen aynı event idempotent; farklı değer çakışma raporudur.
- Zaman bilgisi kaybolursa `received_at` korunur, sahte `timestamp` uydurulmaz.
- Train/val metriklerinin veri split ve yöntem versiyonu aynı run'a bağlanır.
- Log parser'ın görmediği ara adımlar doldurulmuş sayılmaz; interpolation olsa bile görsel katmanda açıkça türetilmiş olarak etiketlenir.
- Artan `sequence` sırası geç event'leri saptar; network upload sonrası out-of-order kayıt gerekliyse ayrı `producer_sequence` ve dedup korunur.

### 16.2 Gözlemlenebilirlik

Forge'ın ilk sürümü için şu log alanları yeterlidir: `run_id`, `attempt_id`, `stage`, `step`, `reason_code`, `duration_ms`, `device`, `artifact_id`. Session/API key ve özel ses path'leri loga ham yazılmaz. Eğitim sırasında GPU memory, CPU/RAM/disk ve IO metrikleri alınabilir; bunların toplanma aralığı ve profiler overhead'i raporlanır. "Ağır tracing" ilk çalışan sürümü geciktirmemelidir.

## 17. İptal, timeout ve process tree yönetimi

İptal iki aşamalıdır: (1) kalıcı `cancel_requested` kaydı, (2) OS seviyesinde çalışmanın gerçekten durdurulması ve kaynakların bırakılması. UI'da «İptal edildi» yazmak, CUDA process'i hâlâ çalışıyorsa yanıltıcıdır.

### 17.1 Önerilen yürütme akışı

```text
user cancel
  -> persist cancel_requested(reason, actor)
  -> transition running -> cancelling
  -> signal owned child process/group
  -> wait grace period
  -> if still alive: force terminate owned process tree
  -> verify process exit + release lease
  -> transition cancelling -> cancelled
  -> inspect final checkpoint separately
```

- Linux/POSIX: process group/session yönetimi, SIGTERM→SIGKILL ve child ownership test edilmeli.
- Windows: process tree, Job Object veya doğrulanmış `taskkill`/platform adapter stratejisi gerekebilir. POSIX `killpg()` Windows'ta doğrudan çalışmaz.
- Worker'ın ölmesi, çocuğun da öldüğü anlamına gelmez; orphan tespit edilmelidir.
- Kullanıcının istemediği başka PID'ye yanlışlıkla kill göndermek ağır hatadır; PID fingerprint zorunludur.
- Cancel sonrası doğrulanmış checkpoint bulunabilir ama **iş yine cancelled** kalır. Checkpoint'ten yeni attempt ile devam edilebilir.
- `cancel` isteği tekrarlandığında aynı operasyonun durumu döner; iki eşzamanlı kill isteği yarışa sokulmaz.

### 17.2 Timeout sınıfları

`startup_timeout` (process spawn/hazırlık), `heartbeat_timeout` (worker/remote durum kaybı), `wall_timeout` (gerçek saat üst sınırı), `shutdown_timeout` (graceful kapanış) ayrı tanımlanır. Uzun ilk model download'u ile çalışan eğitimi aynı süreye sıkıştırma. Timeout nedeni ve başladığı zaman UTC + monotonic clock farklarıyla doğru raporlanmalıdır; sistem saati sıçraması süre ölçümünü bozmasın.

`timeout` sonucu otomatik `failed` mi `interrupted` mı sınıflanacaksa neden kodlarıyla birlikte deterministik politika olmalıdır. Aynı hatanın farklı çalıştırmalarda rastgele terminal duruma gitmesi test başarısızlığıdır.

## 18. Hata sınıfları ve recovery matrisi

| Hata | Preflight veya runtime | Nihai yaklaşım | Otomatik retry? |
|---|---|---|---|
| Bozuk YAML, eksik alan | Preflight | `rejected`; alan bazlı hata | Hayır |
| Dataset hash uyuşmazlığı | Preflight | Fail-closed, dosyayı karantinaya al | Hayır |
| Yanlış codec revision | Preflight | `CODEC_MISMATCH` | Hayır |
| Eksik bağımlılık / yanlış Python | Preflight | Env raporu, run reddi | Yeni env hazırlama sonrası |
| CUDA OOM | Runtime | `failed`, kaynakları temizle | Değişiklikli yeni run/attempt ve onay |
| NaN/Inf loss | Runtime | Backend hata, checkpoint ayır | Hayır, teşhis şart |
| Disk full | Runtime | `failed/interrupted`, son sağlam artifact | Alan açılınca kontrollü |
| Worker crash | Runtime | `interrupted`, orphan kontrol | Sahiplik uzlaşması sonrası |
| Backend exit 0, checkpoint yok | Verify | `failed`, yanlış başarı yok | Hata çözümünden sonra |
| Checkpoint kısmi/bozuk | Verify | Quarantine, önceki verified | Sağlam checkpoint varsa |
| Colab oturum kapanması | Remote | `interrupted` veya `unknown`, import taraması | Sadece açık resume kararı |
| Drive copy yarım | Transfer | Staging dosyası atılır/yeniden dener | Sınırlı exponential backoff |
| Kullanıcı iptali | Runtime | `cancelled` | Otomatik değil |
| Evaluator bozuk | Post-run | Eğitim sonucu ve evaluation başarısızlığı ayrı | Evaluation rerun olabilir |

**Retry sınıflandırması:** Ağ/transfer gibi geçici altyapı hataları küçük sayıda ve jitter'lı backoff ile tekrar denenebilir. Validasyon, lisans, codec, mantıksal model uyumsuzluğu veya OOM gibi durumlarda kör otomatik retry, gizli maliyet ve sahte başarı oluşturur. Retry sayısı, son hata ve neden kaydedilir.

### 18.1 OOM için doğru politika

CUDA OOM olduğunda Forge kullanıcıya `batch_size`, `max_frames`, gradient accumulation, precision, activation checkpointing ve model küçültmenin olası etkilerini gösterebilir. Ama bunlardan birini **aynı recipe altında sessizce değiştiremez**. Başka konfigürasyon başka bilimsel koşuldur; yeni run identity/digest gerektirir. OOM öncesi kısmi checkpoint'in sağlığı doğrulanmadan resume başlatılamaz.

## 19. Crash recovery ve yeniden başlatma algoritması

**HEDEF recovery sırası:**

1. SQLite transaction ile worker ownership lease tablosunu ve bekleyen run/attempt durumlarını oku.
2. Her `running/cancelling` attempt için process identity ve heartbeat kanıtını kontrol et.
3. Aynı process gerçekten çalışıyor ve aynı worker onu izleyebiliyorsa owner lease'i güvenle yenile; zayıf kanıt varsa sahipliği varsayma.
4. Process kaybolmuşsa attempt `interrupted` olur; terminal sonuç uydurma.
5. Orphan child varsa ayrı orphan policy ile kullanıcıya bildir veya doğrulanmış ownership altında kapat; başka uygulama process'ine müdahale etme.
6. Yarım kalan `.tmp` dosyaları final artifact sayılmaz; quarantined staging dizinine taşınır/retention ile temizlenir.
7. Son sağlam checkpoint'i metadata + hash + güvenli load kontrolünden geçir.
8. Kuyruktaki işlerin snapshot/dataset revizyonunu yeniden doğrula; artık geçersiz girdili job'u çalıştırma.
9. Kullanıcının açık resume politikası ve varsa onayıyla yeni attempt oluştur; eski attempt'i değiştirme.
10. Recovery raporu (`recovered`, `interrupted`, `orphaned`, `manual_action_required`) saklanır.

Recovery işlemi idempotent olmalıdır: aynı uzlaşma iki kez yürütüldüğünde duplicate attempt veya çift process oluşturmaz. Gerçek testte supervisor process'i zorla öldürmek, ayrı terminalde çalışan alt process'i bırakmak ve sonra Forge'u tekrar açmak gerekir. Yalnızca Python exception ile yapılan simülasyon yeterli değildir.

## 20. Hosted Google Colab ile uygulama sınırı

**Başlangıç politikası:** Hosted Colab runtime üzerinde model eğitimi yapılabilir, fakat Forge V1 doğrudan Colab oturumunu sürekli kontrol eden güvenilir uzak daemon varmış gibi davranmaz. Colab oturumları süre, GPU, RAM, disk, network, disconnect ve idle politikaları nedeniyle kesilebilir. Platform kaynak sınırları ve kullanım koşulları değişebilir; koşulsuz süre veya ücretsiz GPU garantisi verilemez.

İlk güvenli model:

```text
Local Forge
  -> validate + export immutable run bundle
  -> kullanıcı Colab ortamını açar / bundle'ı çalıştırır
Hosted Colab
  -> download dataset from permitted remote source
  -> train Speech backend using pinned recipe
  -> write temp checkpoints locally (/content)
  -> copy verified shards/checkpoints to Drive
  -> write completion manifest LAST
Local Forge
  -> import remote run manifests and artifacts
  -> verify hashes + source identities + metadata
  -> record terminal status only with evidence
```

**Önemli:** `local Forge -> Colab` otomatik başlatma, token ile API kontrolü veya uzaktan yönetim bir V1 gerekliliği değildir. Colab üzerinde çalışan notebook, Forge'un paketini kuran ve sürümlü çalışma reçetesini kullanan bir adapter olabilir. Başlangıçta manuel transfer/import daha güvenilir ve test edilebilir.

### 20.1 Colab run bundle içeriği

```text
run_bundle/
  resolved_recipe.yaml
  recipe_digest.txt
  speech_source_ref.json
  data_manifest_ref.json
  codec_ref.json
  environment_requirements.json
  launch_instructions.md
  expected_outputs.json
```

`launch_instructions.md`, Colab'da **hangi komutların gerçekten mevcut olduğunu** ve hangilerinin sadece hedef olduğunu ayırır. Bu bundle, özel git token'ı veya Drive credential içermez. Source pin'leri ve dataset lisansları doğrulanmadan remote execution başlatılmaz.

### 20.2 Colab runtime hazırlığı

- Drive mount çalıştırma anında açık kullanıcı yetkisi gerektirir; credential kaynak koduna yazılmaz.
- GPU varsa tam model adı, PyTorch/CUDA, compute capability ve görünür VRAM raporlanır.
- Disk alanı, indirilecek veri + eğitim cache + checkpoint peak ihtiyacına göre kontrol edilir.
- Paketlerin floating latest sürümüne sessiz upgrade yapılmaz; uyumlu lock/source SHA doğrulanır.
- `HF_HOME` gibi cache yolları konfigürasyona alınır; Drive'a küçük rastgele yazma işlemleri çoğaltılmaz.
- CPU ve GPU işleri ayrılabilir: eski Speech bootstrap akışında EMA Lightning CPU + Mimi CUDA seçimi belirli ortam için doğrulanmış tarihsel tercihtir; her Colab imajına kalıcı garanti değildir.

## 21. Google Drive depolama ve transfer sözleşmesi

Drive kalıcı artifact hedefi olabilir, ama yüksek sıklıklı random I/O'lu optimizer/checkpoint yazımı için güvenilir POSIX local disk gibi kullanılmamalıdır. Tercih: `/content` içindeki kısa ömürlü çalışma diski → staging → checksum doğrulama → Drive final artifact. Tekil büyük dosyalarda parçalara bölme ve kalıcı resume manifest tercih edilir.

**HEDEF önerilen yol** (mevcut Drive'daki kesin klasörleri temsil etmez):

```text
MyDrive/KaiwenAI/
  datasets/<dataset_id>/<version>/
  forge/runs/<run_id>/
    resolved_recipe.yaml
    environment.json
    metrics.jsonl
    checkpoints/
    artifacts/
    transfer_manifest.json
    completion_manifest.json
  forge/registry/<model_id>/<version>/
```

Eski `MyDrive/KaiwenVoice/` altındaki deneyleri silme veya otomatik taşıma yoktur. Yeni arşiv yolu için import/alias/migration stratejisi tanımlanır.

### 21.1 İki aşamalı transfer

1. Kaynak lokal artifact stabil hâle gelsin ve kaynak hash hesaplansın.
2. Transfer hedefi `staging/<id>.partial` gibi tamamlanmamış isimle açılır.
3. Boyut ve SHA-256 aktarım sonrası tekrar ölçülür.
4. Hedef beklenenle aynıysa `verified` manifest yazılır.
5. Final dosya yolu veya `completed` marker ancak verification bitince yayınlanır.
6. Drive semantics atomic rename garantisi vermiyorsa manifest son yazma + immutable content-addressed hedef kullanılır; atomic rename varsayılmaz.
7. Retry için shard resume cursor/sparse upload bilgisi tutulur; bozuk kısmi veri sessiz overwrite edilmez.
8. Transfer yavaş veya quota doluysa eğitim işinin kurtarılabilir kalması için local retention ve açık uyarı gerekir; Colab oturumu kapanmadan aktarılmayan veri kaybolabilir.

`completion_manifest.json` oluşmuş olsa bile Forge import sırasında tüm artifact hash'lerini **yeniden** doğrular. Metadata dosyası başarılı yazıldı diye bozuk checkpoint doğru sayılmaz.

### 21.2 Remote import güven modeli

Remote manifest bir *claim*'dir; gerçek başarı kanıtı değildir. Importer, beklenen run ID, source SHA, dataset digest, recipe digest, codec identity, artifact hashes, step ve completion reason alanlarını karşılaştırır. Çelişki varsa import `quarantined` olur, mevcut run'ın kaydı sessizce değiştirilmez. Origin trust/erişim hakkı teyidi olmayan `.pt` dosyaları güvenilmez kaynak sayılır. Cloud tarafında oluşan `interrupted` event'i tam başarısızlık kanıtı veya `completed` anlamına gelmez; durum ve eksik kanıt birlikte raporlanır.

## 22. Artifact manifest ve release'den ayrım

Bir run tamamlanınca en az şu artifact kayıtları beklenir:

```json
{
  "schema_version": "1.0",
  "run_id": "example-run-001",
  "attempt_id": "example-attempt-001",
  "artifacts": [
    {
      "kind": "checkpoint",
      "uri": "artifacts/runs/example-run-001/checkpoints/step_00100.pt",
      "sha256": "REPLACE_WITH_VERIFIED_SHA256",
      "size_bytes": 1234,
      "verification": "example_only"
    },
    {
      "kind": "metrics",
      "uri": "artifacts/runs/example-run-001/metrics.jsonl",
      "sha256": "REPLACE_WITH_VERIFIED_SHA256",
      "size_bytes": 1234,
      "verification": "example_only"
    }
  ]
}
```

Bu örnek gerçek artifact değildir. Placeholder SHA değerleri ve örnek boyutlar üretim validator'ından geçmemelidir. Bir run checkpoint'i training artifact'idir. **Model registry'de candidate/released statüsü başka bir doğrulama aşamasıdır**: doğru Speech+codec sürümü, lisans/provenance, ses kalite değerlendirmesi, performans, inference export ve Live compatibility geçilmeden model dağıtılamaz.

## 23. Evaluation ve training pipeline sınırı

Forge training pipeline metrikleri kaydeder ve istenen evaluator suite'ini **iş sonrası veya ayrı job** olarak çağırabilir. Ancak değerlendirme sonuçlarının bilimsel sahipliği ve ölçüm protokolü ileride hazırlanacak `EVALUATION.md` belgesinde detaylandırılacaktır. Training job başarılı olabilirken evaluation başarısız olabilir. Bu nedenle üç farklı alan tutulmalıdır:

- `training_execution_status`: alt süreç çıkışı, optimizer step ve checkpoint doğruluğu.
- `evaluation_status`: belirli suite'in gerçek değerlendirme sonucu.
- `promotion_eligibility`: registry'ye geçiş şartlarının birleşik sonucu.

Bunları tek bir `completed` Boolean'ına indirgeme. Örneğin 100-step synthetic smoke test yapısal doğruluğu kanıtlar ama Türkçe konuşma kalitesini veya canlı 2 GB VRAM performansını kanıtlamaz. Training loss grafiklerinde değerlendirme yapılmadan «Türkçe ses modeli iyi konuşuyor» etiketi gösterilmez.

### 23.1 Çoklu deney ve ablation

Sadece bir hiperparametre farkını değerlendirmek için her run ayrı `recipe_digest`, codec/dataset snapshot ve etiketlenmiş hipotez taşımalı. `parallel` vs `depth`, `12.5 Hz` vs `50 Hz`, synthetic vs real gibi karşılaştırmalar aynı bilimsel veri split/protokolü ve mümkün olduğunca eşdeğer donanım koşulları olmadan yorumlanmaz. Forge karşılaştırma kimliğini düzenler; model mimarisini kendiliğinden değiştirmez.

### 23.2 Evaluation başarısızsa recovery

Eğitim tamamlandıktan sonra evaluator process çökerse eğitim checkpoint'i değişmez; yalnızca değerlendirme attempt'i başarısız kalır. Aynı verified checkpoint + suite version ile evaluation tekrar koşturulabilir. Çıktıların hash'leri ve suite revision saklanır. Evaluator'ı tekrar çalıştırmak için eğitimi otomatik tekrarlamak maliyetli ve gereksizdir. Farklı suite revision ile üretilen sonuçlar aynı tabloda karşılaştırılırsa revision farkı görünür olmalıdır.

## 24. Bilgi güvenliği ve tehdit modeli

Pipeline yazılım çalıştırma, dosya okuma/yazma ve GPU tüketme yetkisine sahiptir; saldırı yüzeyi sadece web API ile sınırlı değildir.

| Risk | Önlem |
|---|---|
| Kullanıcı reçetesinden keyfi shell command | Sabit backend adapter, argv allowlist, `shell=False` |
| Path traversal / symlink escape | Resolve + sandbox/root policy, path ownership, race testleri |
| Zararlı PyTorch `.pt` | Trust boundary, `weights_only=True` + izolasyon, imza/digest politikası |
| Credential sızması | Secret manager/env izinleri, log redaction, bundle secret-free |
| Dataset yetki/lisans ihlali | Provenance doğrulama, otomatik yayını kapatma |
| Yanlış process öldürme | PID fingerprint, process tree ownership, fencing |
| Supply-chain / floating dependencies | Commit pin, lockfile, integrity verification, dependency review |
| Resource exhaustion | Bound queue, disk/CPU/GPU limitleri, timeout, rate limit |
| Sahte completion/metrics | Typed schema, gerçek process/verified artifact ve event journal |
| Corrupt SQLite/Drive transfer | Backup/WAL, manifest + hash, restore testleri |

Metrik/log dosyalarında ham konuşma transkriptleri, gerçek kişi bilgileri veya ses kaydının korunmasız yolları varsayılan bulunmaz. Eğitim verisi rıza/lisans koşullarına göre erişim kontrolüyle saklanır. GitHub Actions'a Google Drive kimlik bilgileri veya eğitim verisi konmaz; secret ancak ilgili işlem ihtiyaç duyuyorsa dar yetkiyle runtime'a aktarılır. Public Forge reposuna gerçek özel dataset manifestleri/checkpoint'leri commit edilmez.

### 24.1 Denetim ve erişim sınırları

V1 tek kullanıcı ortamı olması, uygulamanın varsayılan olarak tüm dosya sistemini silme yetkisiyle çalışmasını haklı çıkarmaz. Kullanıcı onayı gerektiren işler: başka run'a ait artifact silme, toplu dataset temizliği, Drive migration, uzak işlem başlatma ve release publish. Artifact temizleme retention kurallarıyla kendi sandbox'ı içinde ve dry-run raporuyla çalışmalıdır. Input dosyalarına yalnızca okunabilir erişim verilmeli, output dizini ayrı olmalıdır.

## 25. Test stratejisi ve minimum kalite kapıları

Her özelliğin testi hem başarılı hem de başarısız davranışı doğrulamalıdır.

| Test türü | Gereken örnekler | Çalıştırma ortamı |
|---|---|---|
| Unit: recipe | Geçerli YAML, unknown field, conflict, digest canonicalization | CPU CI |
| Unit: state | Bütün izinli/yasak geçişler, idempotency, stale event | CPU CI |
| Unit: path/security | Traversal, symlink, shell injection, secret redaction | CPU CI |
| Unit: resource | Queue limit, lease expiry, duplicate claim | CPU CI |
| Integration: SQLite | WAL restart, concurrent claim, migration, rollback | CPU CI |
| Integration: fake backend | Success, exit != 0, hang, timeout, stderr flood, cancellation | CPU CI |
| Integration: artifact | Partial write, hash mismatch, corrupt checkpoint, low disk | CPU CI |
| Contract: Speech | Gerçek CLI argümanları ve config parse uyumluluğu | CPU CI |
| Smoke: Speech | 100-step synthetic continuation, checkpoint yükle | CPU/dev |
| Integration: remote | Drive mock/fixture, duplicate transfer, manifest conflict | CPU CI |
| Fault injection | Supervisor kill, orphan process, restart recovery | OS matrix |
| GPU/e2e | OOM, precision, VRAM, warm-start/resume | İsteğe bağlı gerçek GPU |

### 25.1 Gerçek GPU olmadan güvenilir geliştirme

GPU gerektirmeyen CI testleri fake backend ile deterministik yürütülür. Fake backend **Forge orchestration'ın doğruluğunu** kanıtlar, gerçek Speech kalitesini değil. Speech import'u, basic model forward ve checkpoint yapısal smoke testleri CPU'da yapılabilir. 2 GB VRAM uyumu veya Colab GPU hızı, CI fake testleriyle doğrulanmış sayılamaz. GPU benchmark'ları cihaz adı, sürüm, precision, batch, step, warmup ve rapor hash'i ile ayrı koşturulur.

### 25.2 CI örnek hedefi

**HEDEF iş akışı** (şu anda Forge'da `.github/workflows/ci.yml` yok):

```yaml
name: ci
on:
  push:
  pull_request:
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v7
        with:
          python-version: "3.12"
          enable-cache: true
      - run: uv sync --frozen --extra dev
      - run: uv run ruff check .
      - run: uv run pyright
      - run: uv run pytest
```

Bu akış ancak Forge `pyproject.toml`, extras, test setup ve lock dosyası yazılıp doğrulandıktan sonra uygulanmalıdır. Burada kullanılan versiyonlar Speech'in mevcut CI politikasından örneklendirilmiştir; Forge için otomatik çalıştığı iddia edilmez. Üçüncü taraf action revizyon pin'i ve least-privilege token izinleri release hazırlığında ayrıca değerlendirilir.

### 25.3 Kabul kanıtı raporu

Her PR şu alanları içerir: değişen modüller, kapsam sınırı, çalıştırılan komutlar, beklenen/gerçek sonuçlar, hangi testlerin çalışmadığı, sonuç metrikleri, artifact path ve hash, güvenlik/uyumluluk etkisi, rollback planı. CI yeşil olsa bile ses kalitesi veya VRAM hedefi ölçülmediyse bu hedef için «tamamlandı» denmez.

## 26. Geliştirme aşamaları: F0–F9 uygulama sırası

| Aşama | Yapılacak somut iş | Zorunlu kabul kanıtı |
|---|---|---|
| **F0** | Proje iskeleti, Python package, uv, lint/type/test, CI | Temiz checkout'ta kurulabilir import ve CPU CI |
| **F1** | Recipe parser/resolver, snapshot, schema, digest | Fixture ile geçerli/geçersiz recipe ve immutable kayıt |
| **F2** | SQLite run store, events, attempt, migration | Restart, concurrent claim, idempotency negatif testleri |
| **F3** | Fake backend + local process supervisor | Timeout, cancel, crash, stdout flood, ownership testleri |
| **F4** | Mevcut Speech CLI adapter ve smoke entegrasyonu | 100-step CPU run, verified checkpoint, real metrics |
| **F5** | Corpus manifest + token cache validation | Kaynak/split/codec mismatch durumda fail-closed |
| **F6** | Post-run evaluation job kayıtları | Verified suite identity, training/eval durum ayrımı |
| **F7** | Registry candidate/release boundary | Uyumlu manifest, no silent promotion, rollback |
| **F8** | Colab bundle + Drive verified import | Kısmi transfer ve disconnect testleri; hash parity |
| **F9** | Kullanıcı rehberi, reliability ve release sertleştirme | End-to-end demo, Windows/Linux matris, dokümante eksikler |

Bunlar **yapılacak iş paketleridir, tamamlandı statüsü değildir**. F0–F4 tamamlandığında küçük ama gerçek bir Forge sistemi vardır. F8 ve F9 gelmeden «end-to-end cloud platform» ifadesini kullanmak yanıltıcıdır. V1 tüm model ailelerini aynı anda desteklemez; yalnızca Speech backend gerçek örnekle doğrulanır.

### 26.1 İlk beş PR önerisi

1. `foundation`: `pyproject.toml`, src, README tutarlılığı, unit tests ve CI.
2. `recipes`: typed schema, sample `speech-cpu-smoke.yaml`, canonical snapshot ve alan doğrulama.
3. `run-store`: SQLite schema, attempts/events, transaction, recovery başlangıcı.
4. `local-executor`: fake backend subprocess, cancellation, timeout ve platform testleri.
5. `speech-adapter`: gerçek `train_continuation.py` komutu, log parser ve checkpoint verifier.

Her PR küçük olmalı, test geçmeli, eski kanıtları korumalıdır. Farklı PR'lara göre birden fazla repo etkilenirse bağımlılık yönüne göre PR dizisi ve contract testleri oluşturulur.

## 27. Operasyon akışı: örnek yürütme senaryoları

### 27.1 Başarılı CPU smoke

```text
Operator submits speech-cpu-smoke.yaml
  -> recipe validation succeeds
  -> run R created, immutable snapshot saved
  -> attempt A queued then atomically claimed
  -> actual Speech CLI launched in known cwd
  -> logs persisted; events/steps recorded
  -> process exit=0
  -> checkpoint verified with expected step 100
  -> artifact SHA-256 stored
  -> attempt A completed; run result reflected
```

Gerçek `train_continuation.py` çıktısında `completed_steps=100`, `first_loss=...`, `final_loss=...` görülebilir; ancak Forge bunu değerleri gerçekten okuyup kaydetmeden kanıt sayamaz. Smoke eğitim loss'unun düşmesi, konuşan bir Türkçe model kalitesini kanıtlamaz.

### 27.2 İş ortasında elektrik veya terminal kapanması

```text
run R / attempt A running
  -> supervisor killed unexpectedly
  -> Forge restarted
  -> DB running state reconciled with process identity
  -> orphan found or loss of worker detected
  -> A marked interrupted with reason
  -> verified checkpoint discovered (if any)
  -> operator explicitly requests resume
  -> attempt B created with resumed_from A
  -> B runs only after compatibility check
```

Bu senaryoda yeni attempt ID'yi saklamadan eski terminal durumu yeniden `running` yazmak kabul edilmez.

### 27.3 Colab'da eksik transfer

```text
Remote output: step_00500.pt verified on /content
  -> upload to Drive partial path starts
  -> Colab session disconnects
  -> Drive has partial bytes, no final completion record
  -> Local import marks transfer incomplete, not completed
  -> resumed remote upload verifies hash and finalizes
  -> local importer re-checks bytes against trusted manifest
```

`Drive'a dosya adı geldi` ifadesi tamamlanma kanıtı değildir.

### 27.4 Değişen codec ile yanlış resume

```text
Old run: Mimi tokenizer Q8/K2048 @ 12.5Hz
New proposed run: Kaiwen Codec Q4/K1024 @ 50Hz
  -> validator detects codec/token mismatch
  -> resume rejected before GPU allocation
  -> user offered new research run or explicit conversion plan
```

Aynı konfigürasyon sayıları mevcut olsa bile codec revision/content hash değişiyorsa da fail-closed uygulanır.

## 28. Performans ve kaynak mühendisliği

Forge'un CPU orkestrasyon maliyeti, Speech modelinin eğitim süresinden ayrı ölçülmelidir. İlk aşamada şu metrikler yeterlidir:

- `submit_to_validated_ms`: recipe çözümleme gecikmesi.
- `queue_wait_ms`: resource slot bekleme.
- `launch_to_first_step_ms`: process spawn + model init + veri erişimi.
- `last_heartbeat_age_s`: worker canlılık kanıtı.
- `artifact_verify_ms` ve `artifact_bytes_per_s`: checkpoint transfer/doğrulama maliyeti.
- `sqlite_write_latency_ms`: metrik event ingestion maliyeti.
- `gpu_allocated_bytes`, `gpu_reserved_bytes`, `host_ram_peak_bytes`: ölçülen kaynak kullanımı.
- `training_steps_per_sec`: Speech backend metriklerinden kaynak bağlantılı değer.

Ölçümler her zaman platform, örnek boyutu, step, batch, precision, codec/model sürümü ve cold/warm cache koşullarıyla raporlanır. «Forge overhead yüzde birden az» gibi doğrulanmamış eşikler yazılmaz. İlk acceptance için supervisor'un eğitim performansını gereksiz etkileyen aşırı log/DB flush yapmadığı ölçülmelidir; metrik batch ve artifact stream yolları optimize edilebilir.

50 Hz codec ile 12.5 Hz Mimi baseline aynı ses süresinde eşit temporal frame sayısı üretmez; Transformer compute ve KV cache maliyeti farklı olabilir. Forge farklı varyantların eğitim süresini ölçüm koşullarıyla birlikte raporlar ama mimari performans iddiasını birleştirmez. 2 GB VRAM hedefi esas olarak **inference** içindir; eğitim için ayrı donanım ve memory bütçesi gerekir, bunlar farklı benchmark suite'leriyle kanıtlanır.

## 29. Codex uygulama protokolü ve değişiklik sınırları

Bir Codex görevi şu sıralamayı izlemelidir:

1. `kaiwen-forge/README.md`, `kaiwen-forge/AGENTS.md` ve bu belgeyi oku.
2. Workspace'teki [Engineering Standards](../../docs/ENGINEERING_STANDARDS.md) ile [Integration](../../docs/INTEGRATION.md) kurallarını kontrol et.
3. GitHub repo mevcut ağacını, değiştirilmiş dosyaları, gerçek Speech CLI argümanlarını, lock/config durumunu incele.
4. Görevin F0–F9 hangi aşamasına ait olduğunu ve non-goal'lerini yaz.
5. Yalnızca gerekli minimal modülleri ve contract fixture'larını oluştur.
6. Önce negatif testleri (bozuk config, duplicate, crash, hash mismatch), sonra başarı testlerini ekle.
7. CI kalitesini bozan iş bırakma; çalıştırılmayan testlerin nedenini raporla.
8. Başarıyı sadece log mesajıyla ilan etme; gerçek artifact/step/state kanıtı topla.
9. Dokümanlar güncel kodla çelişiyorsa durumu `Implemented/Planned/Experimental` ile düzelt; eski gerçeği silme.
10. Kullanıcı onayı olmadan mevcut checkpoint/dataset silme, Drive toplu migration yapma, gizli veriyi dışarı aktarma veya geçmişi bozacak force-push gerçekleştirme.

**Yasak anti-pattern'ler:** sırf yeni framework tanındı diye orkestratörü yeniden yazmak, model Trainer'ını Forge'a kopyalamak, her backend için henüz ihtiyacı olmayan yüzlerce interface eklemek, local CPU smoke olmadan Kubernetes'e geçmek, `except Exception: pass` ile iş kaybını gizlemek, bozuk checkpoint'i `completed` işaretlemek, metrik üretmek için sahte sayı doldurmak, tek config ile tüm hardware için hız garantisi vermek.

## 30. Definition of Done ve belge kabul kriterleri

`TRAINING_PIPELINE.md` teknik tasarımı **uygulanmış** sayılmadan önce aşağıdaki gerçek kanıtlar gereklidir:

- [ ] Forge package kurulabilir, CI ve tip/lint/test kapıları geçer.
- [ ] Recipe parser/validator sabit şema ve snapshot digest üretir.
- [ ] SQLite run/attempt/event kaydı restart sonrası korunur.
- [ ] Queue claim yarışında aynı attempt çift başlatılmaz.
- [ ] Fake backend'de success, exit nonzero, hang, timeout, cancel, crash testleri vardır.
- [ ] Gerçek Speech CLI ile 100-step synthetic continuation run başlatılır.
- [ ] Doğrulanmış `latest.pt` veya step checkpoint'i, gerçek step ve hash ile kayda alınır.
- [ ] Son checkpoint bozulduğunda iş sahte başarıya dönüşmez.
- [ ] Resume / warm-start ayrımı ve codec uyumluluk testleri bulunur.
- [ ] Hatalı Corpus manifest ve leakage kontrolleri net hata üretir.
- [ ] Colab/Drive import'unda yarım upload, yeniden deneme, hash ve conflict testleri geçer.
- [ ] Windows/Linux process cancellation senaryoları gerçek ortamda doğrulanır veya platform desteği açıkça kısıtlanır.
- [ ] README, AGENTS ve ilgili çapraz repo sözleşmeleri uygulanan kodla uyumludur.
- [ ] En az bir tam E2E demo kanıtı (recipe + run snapshot + process exit + metrics + verified checkpoint + recovery test raporu) saklanır.

Bu checklist **bugün tamamlanmış iş listesi değil**, kod geliştirmeye başladığımızda kullanılacak test sözleşmesidir. Herhangi bir adım uygulanamıyorsa nedenleri ve alternatifleri kayda geçer; üzerinde «tamamlandı» işareti konmaz.

## 31. Açık kararlar ve karar verme eşiği

| Açık karar | Şimdilik güvenli varsayım | Ne zaman yeniden bakılacak? |
|---|---|---|
| SQLite mı Postgres mi? | Tek host SQLite WAL | Gerçek çok yazarlı/multi-host ihtiyacı çıkarsa |
| CLI mı in-process Python adapter mı? | Gerçek Speech CLI | Çok sık çağrı ve profil ölçümü gerekçe sunarsa |
| Tek worker mı dağıtık queue mu? | Tek local worker | Eş zamanlı çok GPU/host kanıtlanırsa |
| Colab uzaktan yönetim mi import mu? | Bundle + verified import | Güvenilir API/oturum kontrolleri mümkünse |
| Metrics için log parser mı JSONL mı? | Mevcut log parser geçici | İlk Speech adapter PR'ında JSONL protokolü öner |
| Checkpoint content-addressed mı? | Verified immutable copy + manifest | Depolama maliyeti/geri çağırma ölçülürken |
| Artifact transfer için cloud SDK mı? | Yerel + Drive mount staging | İhtiyaç/kota/güvenlik ölçülünce |
| Genel model adapter framework mü? | Tek Speech adapter | İkinci gerçek model ailesi olduğunda |
| UI dashboard gerekli mi? | CLI + okunabilir rapor | Operatörün somut zaman kaybı ölçülünce |
| Tam bit-deterministic resume? | Mevcut sınır raporlanır | Speech state formatı ve loader iyileştirilince |

Yeni bir teknoloji ortaya çıktığında «bilmiyorduk, bütün tasarımı değiştir» yaklaşımı yerine [Engineering Standards — Discovery Gate](../../docs/ENGINEERING_STANDARDS.md) kullanılır: problemi tanımla, mevcut çözümle karşılaştır, küçük proof-of-concept ile ölç, ancak sonra kabul et. Gereksiz planlama geliştirmenin önüne geçmemelidir.

## 32. Kaynaklar, bağlantılar ve uyumluluk notu

**Bu dokümantasyon setinde mevcut dosyalar:**

- [Kaiwen Forge README](../README.md) — ürün sınırı ve hedef paket yapısı.
- [Kaiwen Forge AGENTS](../AGENTS.md) — Codex çalışma protokolü.
- [Workspace Overview](../../docs/OVERVIEW.md) — altı repo mimarisi.
- [Workspace Integration](../../docs/INTEGRATION.md) — veri, codec, model ve Live entegrasyon sözleşmeleri.
- [Workspace Engineering Standards](../../docs/ENGINEERING_STANDARDS.md) — mühendislik kalite kapıları.
- [Workspace Roadmap](../../docs/ROADMAP.md) — F0–F9'un üst seviye iş programı.
- [Kaiwen Speech Architecture](../../kaiwen-speech/docs/ARCHITECTURE.md) — codec/token/model sözleşmeleri.
- [Kaiwen Speech Speech Model](../../kaiwen-speech/docs/SPEECH_MODEL.md) — loss, masking, inference ve model eğitimi.
- [Kaiwen Speech API](../../kaiwen-speech/docs/API.md) — bugünkü API / hedef SDK ayrımı.
- [Kaiwen Speech Performance](../../kaiwen-speech/docs/PERFORMANCE.md) — ölçümlerin koşulları.

**Sonraki, henüz ayrıntılandırılmamış Forge belgeleri:** `docs/EXPERIMENTS.md`, `docs/EVALUATION.md`, `docs/MODEL_REGISTRY.md`. Bu belge onların yerini almaz. Her biri sonraki adımlarda birer dosya olarak oluşturulacaktır. Bu dosyanın varlığı kendi başına Forge kodunun hazır olduğu veya gerçek deneylerin çalıştığı anlamına gelmez.

---

**Son ilke:** Forge'un başarısı ne kadar çok altyapı soyutlaması içerdiğiyle değil; *gerçek bir Kaiwen Speech eğitimini başlatıp sonuçlarını, kaynaklarını, hatalarını ve yeniden üretilebilirlik kanıtlarını dürüst ve güvenilir biçimde yönetebilmesiyle* ölçülecektir.
