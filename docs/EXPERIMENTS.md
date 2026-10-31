# Kaiwen Forge — Experiments, Reproducibility ve Bilimsel Karşılaştırma Şartnamesi

> **Belge:** `kaiwen-ai/kaiwen-forge/docs/EXPERIMENTS.md`  
> **Durum:** **HEDEF TEKNİK TASARIM / IMPLEMENTED DEĞİL.** Bu belge çalışan bir deney platformunun var olduğunu iddia etmez.  
> **Tarihsel başlangıç:** 30 Ekim 2026 itibarıyla `ImtheKaiwen/kaiwen-forge` GitHub reposu yalnızca kısa bir README içeriyordu. Bu çalışma alanındaki ayrıntılı Forge README, AGENTS ve TRAINING_PIPELINE belgeleri Codex için hazırlanmış taslaklardır.  
> **V1 hedefi:** Kaiwen Speech ile yapılan bir deneyi kimliği, kodu, verisi, konfigürasyonu, cihazı, metrikleri, checkpoint'leri ve değerlendirme kanıtlarıyla kaydetmek; başka bir koşulla **adil biçimde karşılaştırmak**.  
> **Öncelik:** Önce tek model ailesi (`speech`), tek host, yerel SQLite + dosya tabanlı artifact store ve tekrar üretilebilir temel rapor. Dağıtık arama, hosted SaaS, büyük dashboard ve otomatik hiperparametre optimizasyonu sonraki aşamalardır.

---

## 1. Belgenin amacı, kapsamı ve bağlayıcılığı

Bu belgenin işi, model eğitiminin nasıl çalıştırıldığını yeniden tanımlamak değildir. Çalıştırma, işlerin durum makinesi, worker, checkpoint transferi ve iptal mekanizması için [Training Pipeline](TRAINING_PIPELINE.md) tek ayrıntılı kaynaktır. Bu belge ise eğitimlerden çıkan kanıtları bir araya getirir: **hangi soruyu sorduk, hangi değişkeni değiştirdik, hangi veriyi kullandık, gözlenen fark ne kadar güvenilir, hangi iddiayı destekliyor?**

Bu ayrım korunmadığında bir ekip aynı `loss` eğrisini farklı koşullarda çizip modeli iyileştirdiğini sanabilir. Özellikle Türkçe speech-native araştırmada ses kalitesi, semantik yanıt, konuşma zamanlaması ve donanım gecikmesi birbirinden farklı yeteneklerdir. Bunlar tek bir rakamda toplanıp bilimsel gerçekmiş gibi sunulamaz.

### 1.1 Kapsam içi

- Araştırma sorusu, hipotez ve karşılaştırma tasarımı.
- `experiment`, `study`, `run`, `attempt`, `evaluation`, `comparison`, `artifact` yaşam döngüleri.
- Run provenance: kaynak kod, veri, model/codec, config, donanım, bağımlılık, rastgelelik durumu.
- Structured metrik günlükleri, normalizasyon, eksik veri ve görselleştirme kuralları.
- Baseline ile candidate eşleştirme, birden fazla seed, eşlenmiş örnekler, varyans ve güven aralıkları.
- Hiperparametre taraması, ablation, kalite–gecikme–bellek Pareto analizleri.
- İnsan değerlendirmesi, transcript yardımcı sinyalleri, dinleme testleri için metodoloji.
- Hata, iptal, yeniden deneme ve kısmi sonuçların doğru raporlanması.
- Dataset leakage, test setine aşırı uyum, ölçüm metodolojisi değişikliği gibi araştırma riskleri.
- Forge V1'in SQLite + yerel artifact'lerle uygulayabileceği somut veri sözleşmeleri ve testler.

### 1.2 Kapsam dışı

- Neural codec katmanları, RVQ ve ses kayıpları: [Speech Codec](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/CODEC.md).
- Transformer mimarisi, attention, loss hesabı: [Speech Model](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/SPEECH_MODEL.md).
- Benchmark motorunun modele özgü uygulaması ve ölçüm yöntemleri: [Speech Performance](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/PERFORMANCE.md).
- Dataset toplama, diarization, etiketleme: Kaiwen Corpus.
- Bir checkpoint'in dağıtılabilir sürüm olarak onaylanması: gelecekteki `MODEL_REGISTRY.md`.
- Çalışan bir web panel, uzaktan dağıtık çalışma sistemi veya MLflow sunucusu zorunluluğu.

### 1.3 Normatif durum etiketleri

| Etiket | Kesin anlamı |
|---|---|
| **MEVCUT** | Kaynak kodda karşılığı incelenmiş, bugün mevcut arayüz. Bu oturumda çalıştırıldığı anlamına gelmez. |
| **HEDEF V1** | Forge içinde henüz yazılmamış ama ilk sürümün kabul koşulu. |
| **SONRAKİ** | V1 kanıtlandıktan sonra ihtiyaçla değerlendirilecek. |
| **DENEYSEL** | Araştırma adayı; doğrulanana kadar varsayılan çözüm değil. |
| **ÖRNEK** | Şema ya da protokolü açıklayan sahte değer; gerçek hash, dosya veya run sanılmamalı. |

Repo ismi `kaiwen-forge` ile hedef Python import adı `kaiwen_forge` farklıdır. Speech repo ismi `kaiwen-speech` iken **mevcut dağıtım adı** `kaiwen-voice`, **mevcut import** `kaiwen_voice` şeklindedir. Yeni isimlerle örnek kod yazılması o paketlerin bugün kurulu olduğunu göstermez.

## 2. Altı repository içinde sorumluluk ve veri akışı

```text
Corpus                               Speech
  |                                    |
  | versioned dataset export            | codec/model implementation
  v                                    v
+----------------------------------------------------------+
|                      Kaiwen Forge                        |
| Study -> Experiment -> Run -> Attempt -> Metrics          |
|                              |                           |
|                              v                           |
|                       Evaluations -> Comparisons          |
|                              |                           |
|                              v                           |
|                         Reports / Registry candidates     |
+----------------------------------------------------------+
                               |
                    verified model candidate
                               v
                             Live
```

Forge, Corpus'un lisans kararını veya segment düzeltmelerini kendi başına üretmez. Corpus manifestini hash ve immutable kimlikle tüketir. Forge, Speech'in çıktısını doğrular ama `torch.nn.Module` mimarisini yeniden yazmaz. Speech'in ürettiği checkpoint'in iç formatına keyfi alan ekleyerek geriye dönük uyumluluğu bozmaz. Live'a “en iyi model” diye doğrulanmamış bir checkpoint göndermek Forge'un işi değildir; registry onay kapısından geçilmelidir.

[Workspace Overview](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/OVERVIEW.md), [Integration](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/INTEGRATION.md), [Engineering Standards](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ENGINEERING_STANDARDS.md), [Roadmap](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ROADMAP.md), [Forge README](../README.md) ve [Forge AGENTS](../AGENTS.md) üst düzey kararları tanımlar. Bu belge yalnızca **deney ve araştırma kayıtlarının semantiği** konusunda ayrıntılı otoritedir.

## 3. Bilimsel araştırma felsefesi

Forge'un görevi güzel grafik üretmek değil, yanlış sonuçlara varmayı zorlaştırmaktır. Bir deney önce araştırma sorusuyla başlar, sonra başarı ölçüsü, kontrol koşulu, değişkenler, veri bölmesi ve analizi dondurulur. Sonuçlar görüldükten sonra araştırma hipotezini veya birincil metrikleri geriye dönük değiştirmek kabul edilmez; yeni hipotez ayrı revizyon olur.

Ana ilkeler:

1. **Tek hipotez, açık karşılaştırma:** “Depth kaliteyi artırır mı?” sorusu, “yeni model daha iyi mi?” sorusundan daha test edilebilir.
2. **Adil kaynak bütçesi:** Parametre, token, veri miktarı, eğitim süresi, GPU saati ve hesaplama farkları raporlanır. Yalnızca nominal training step eşitlemek her zaman eşit compute değildir.
3. **Önceden belirlenen metrik:** Sonuç ortaya çıktıktan sonra en iyi görünen metriği tek başarı ölçüsü ilan etmeyiz.
4. **Aynı test örnekleri:** Kalite karşılaştırmalarında mümkün olduğunda aynı kayıtlar ve aynı koşullar kullanılır.
5. **Tekrarlı ölçüm:** Tek seed veya tek ses örneğiyle genelleme yapılmaz.
6. **Negatif sonuçlar korunur:** Başarısız denemeler, OOM veya kalitesiz sesler saklanır; sonradan silinmez.
7. **Eksik veri görünür:** Yok olan, hesaplanmayan veya hatalı bir metrik `0` sayılamaz.
8. **Öznel–nesnel ayrımı:** Loss, CER, STOI, MOS ve kullanıcı deneyimi tek kavram değildir.
9. **Provenance olmadan karşılaştırma yok:** Kaynak kod/veri/codec kimliği eksikse sonuç “doğrulanmamış” işaretlenir.
10. **Küçük çalışır dikey dilim:** En başta büyük deney takip platformu yerine, tek bir gerçek Speech smoke run'ını kaydetmek ve aynı koşullarda tekrar çalıştırmak gerekir.

## 4. Kavramlar ve kalıcı kimlikler

Bir **study**, bir araştırma sorusunun genel çerçevesidir: örneğin “Türkçe modellerde 25 Hz ile 50 Hz codec temsilinin etkisi”. Bir **experiment**, belirli protokole bağlı kıyaslamadır: örneğin “aynı dataset, aynı Transformer ve aynı eğitim bütçesi altında codec A/B”. Bir **run**, tek bir donmuş recipe + seed + veri/model kombinasyonudur. Bir **attempt**, aynı run'ın belli anda ve belli çalıştırıcıda başlamış yürütme denemesidir. Bir **evaluation**, belirli artifact'e ve değerlendirme seti sürümüne uygulanan ölçüm prosedürüdür. Bir **comparison**, immutable run/evaluation kayıtlarını bir rapor protokolüyle karşılaştırır. Bir **report** ise bu karşılaştırmanın tekrar oluşturulabilir çıktısıdır.

| Kimlik | Format önerisi | Değişmezlik | Kritik kural |
|---|---|---|---|
| `study_id` | UUIDv7/ULID veya çakışmasız eşdeğer | Kalıcı | Başlık değişebilir, kimlik değişmez |
| `experiment_id` | UUIDv7/ULID | Kalıcı | Protokol revizyonu yeni versiyon oluşturur |
| `run_id` | UUIDv7/ULID | Kalıcı | Resolved recipe/seed/snapshot değişmez |
| `attempt_id` | UUIDv7/ULID | Her yürütme ayrı | Resume eski attempt'i ezmez |
| `evaluation_id` | UUIDv7/ULID | Her değerlendirme ayrı | Aynı checkpoint farklı eval protokolüyle ayrı kayıt |
| `comparison_id` | UUIDv7/ULID | Her rapor protokolü için | Sorgu sonucu değişirse sürüm/fingerprint değişir |
| `artifact_id` | İçeriğe dayalı SHA-256 veya UUID + digest | İçerik hash'i sabit | Dosya adı kimlik değildir |
| `protocol_id` | İsim + semver + SHA-256 | Donmuş | Farklı protokoller habersiz karıştırılmaz |

**Run / attempt ayrımı:** Eğitim kesilirse `run_id` korunur; yeni `attempt_id` açılır. Resume biçimi ve RNG/DataLoader devamlılığı ayrıca işaretlenir. İdeal deterministik resume sağlanana kadar kesintisiz run ile kesintiden dönmüş run istatistiksel olarak birebir eşdeğer kabul edilmez.

**Yeniden oluşturma:** Aynı recipe ve seed ile oluşturulan iki ayrı run, iki ayrı gözlemdir. Otomatik deduplication bunları tek run altında birleştirmez. Yalnızca kullanıcıdaki ağ tekrarıyla aynı submit isteğinin iki kez gelmesi `idempotency_key` yoluyla aynı run'a bağlanabilir.

## 5. Araştırma sorusundan deney planına

Deney başlamadan `research_plan` veya eşdeğer immutable plan kaydedilir. Minimum alanlar: problem, ana hipotez, null hipotezi, baz model, candidate, değişen bağımsız değişkenler, sabit kontrol değişkenleri, veri bölmeleri, primary metric, secondary metrics, etki büyüklüğü, tekrar sayısı, hesaplama bütçesi, erken durdurma, dışlama ve istatistiksel analiz.

### 5.1 Örnek araştırma planı — Parallel vs Depth

```yaml
# ÖRNEK / HEDEF ŞEMA; mevcut bir Forge CLI bunu henüz okumuyor.
schema_version: 1
study:
  slug: turkish-speech-codebook-study
  question: "Aynı eğitim bütçesinde Depth yaklaşımı Parallel'e göre ölçülebilir fayda sağlıyor mu?"
experiment:
  slug: parallel-vs-depth-v1
  protocol_version: "1.0.0"
  preregistered: true
  hypothesis:
    alternative: "Depth, önceden seçilen dinleme kalitesi metriğini iyileştirir."
    null: "Depth ile Parallel arasında ölçülebilir fark yoktur."
  factors:
    varied: [model.codebook_modeling]
    fixed: [dataset_manifest_digest, codec_identity, seed_set, eval_suite]
  arms:
    - name: parallel
      overrides: {model: {codebook_modeling: parallel}}
    - name: depth
      overrides: {model: {codebook_modeling: depth}}
  seeds: [11, 23, 37]
  primary_outcome:
    name: blinded_preference_rate
    direction: maximize
    unit: fraction
    aggregation: paired_by_utterance
  secondary_outcomes:
    - name: validation_loss
      direction: minimize
      unit: nats
    - name: p95_time_to_first_audio_ms
      direction: minimize
      unit: ms
    - name: peak_vram_bytes
      direction: minimize
      unit: bytes
  resources:
    gpu_hours_per_run_max: 4
  analysis:
    confidence_level: 0.95
    resampling_unit: source_group
    bootstrap_replicates: 2000
    familywise_adjustment: holm
  exclusion_policy:
    exclude_only_predefined_corruption: true
  notes: "Örnek değerler bilimsel güç analizi yerine geçmez."
```

Bu taslakta birincil sonucun nasıl ölçüleceği, kör değerlendirici protokolü ve yeterli örneklem ayrı tanımlanmalıdır. Üç seed veya 2.000 bootstrap örneği **otomatik olarak yeterli istatistiksel güç sağlar** demek değildir. Deney başlamadan pilot veriden yaklaşık varyans ve minimum anlamlı etki tahmini yapılmalıdır. Forge, planı ve uygulanan gerçek `resolved_plan`ı ayrı saklar.

### 5.2 Tasarım aşamasında engellenecek kusurlar

- Codec değişirken hem veri ön işleme hem model mimarisi hem de training steps değişiyorsa, tek neden iddiası kurulamaz.
- Bir model farklı veri lisansı veya farklı kaynak bölmesine sahipse aynı deney koluna konulmamalıdır.
- Training loss'a göre seçilen checkpoint, test setindeki skora göre tekrar tekrar seçilemez.
- İnsan değerlendiricisine hangi örneğin “Kaiwen yeni modeli” olduğu söylenerek kör test yapılamaz.
- Aynı kullanıcının/kaynağın kayıtları train ve test'e dağıldığında konuşmacı veya kayıt kaynaklı leakage göz ardı edilemez.
- Geliştirme süresince sürekli bakılan “final test” artık tarafsız final test değildir; yeni bağımsız holdout gerekir.
- Henüz üretilmemiş bir codec v1 eğitimliymiş gibi tanımlanamaz.

## 6. Experimental protocol: değişmez plan ve revizyonlar

Bir deney için `protocol_version`, `protocol_digest`, `created_at_utc`, `author`, `research_question`, `preregistration_timestamp`, `amendments`, `metric_definitions` ve `comparison_policy` saklanır. Plan ilk run başladığında kilitlenir. Birincil metriği değiştirmek, veri bölmesini güncellemek veya dışlama kriterini değiştirmek **yeni protokol revizyonu** gerektirir; eski plan silinmez.

İki ayrı değişiklik türü vardır:

1. **Teknik düzeltme:** Bozuk yazım, yol veya yorum düzenlemesi; eğer deney sonucu değişmiyorsa metadata revision ile belgelenir.
2. **Bilimsel değişiklik:** Metrik formülü, eval seti, seed, eğitim süresi, loss ağırlığı, checkpoint seçimi, örnek dışlama; yeni revizyon ve ayrı comparison gerektirir.

Sonuç görüldükten sonra yapılan ek analizler yasak değildir; **exploratory/post-hoc** diye etiketlenir. Bunlar önceden kayıtlı primary result ile aynı güven seviyesinde gösterilmez. Raporlarda her metrik `predefined` veya `exploratory` durumunu taşımalıdır.

## 7. Run provenance: yeniden üretmek için zorunlu alanlar

Reproducibility yalnızca `seed: 42` yazmak değildir. Aynı girdinin ses tokenları, training loader sırası, derleyici/backend sürümü, nondeterministic kernel ve GPU davranışı farklı sonuç verebilir. Forge, olabilecek en iyi kanıtı kaydeder; sağlanamayan garantileri açıklıkla yazar.

### 7.1 Örnek run snapshot

```json
{
  "schema_version": 1,
  "run_id": "example-run-001",
  "experiment_id": "example-experiment-001",
  "attempt_id": "example-attempt-001",
  "backend": "kaiwen_voice_cli",
  "task": "continuation",
  "source": {
    "repository": "ImtheKaiwen/kaiwen-speech",
    "git_commit": "EXAMPLE_COMMIT_SHA_REPLACE_AT_RUNTIME",
    "dirty_worktree": false,
    "python_distribution": "kaiwen-voice",
    "python_import": "kaiwen_voice"
  },
  "dataset": {
    "manifest_id": "example-dataset-v1",
    "manifest_sha256": "EXAMPLE_HASH_REPLACE_AT_RUNTIME",
    "split_fingerprint": "EXAMPLE_SPLIT_HASH",
    "license_review_status": "approved_for_research"
  },
  "codec": {
    "codec_id": "mimi-example",
    "sample_rate": 24000,
    "frame_rate": 12.5,
    "num_codebooks": 8,
    "codebook_size": 2048,
    "weights_digest": "EXAMPLE_CODEC_HASH"
  },
  "model": {
    "architecture": "temporal_parallel",
    "resolved_config_digest": "EXAMPLE_CONFIG_HASH",
    "checkpoint_id": null
  },
  "training": {
    "seed": 42,
    "max_steps": 100,
    "precision": "fp32",
    "deterministic_requested": true
  },
  "environment": {
    "python": "3.12.x",
    "pytorch": "EXAMPLE_VERSION",
    "cuda": null,
    "os": "EXAMPLE_OS",
    "gpu": null,
    "dependencies_lock_sha256": "EXAMPLE_LOCK_HASH"
  },
  "reproducibility": {
    "level": "configuration_reproducible",
    "bitwise_replay_verified": false,
    "reason": "RNG ve data-sampler recovery henüz bitwise test edilmedi."
  }
}
```

`git_commit` ve config checksum **çalışma anında** hesaplanır. Sahte örnek SHA değerleriyle run güvenilir kabul edilmez; örnek değerler gerçek importer tarafından reddedilir. Yukarıdaki JSON, Forge'un henüz mevcut olmayan hedef manifest biçimidir.

### 7.2 Korunacak kimlik alanları

- Repo URL ve commit SHA; dirty çalışma ağacı varsa diff hash ve gerekirse kaydı. Dirty ortamla yapılan run reddedilebilir veya `nonreproducible` etiketlenir.
- `uv.lock`/lockfile digest, Python, PyTorch, CUDA, cuDNN sürümleri; sürümler gerçekten bilinmiyorsa `null`.
- Dataset snapshot, filtre reçetesi, train/val/test split hash, kayıt/speaker/source-group ID kuralı.
- Codec ağırlık kimliği, checksum, token format sürümü, frame rate, num codebooks ve sample rate.
- Transformer config, prefill/max context, parameter count, output head düzeni.
- Seed bilgisi ve determinism bayrakları; DataLoader worker sayısı ve sampler stratejisi.
- Optimizer/scheduler/precision/gradient clipping, checkpoint seçme politikası.
- Cihaz tipi, CPU modeli, GPU VRAM, kaynak limiti, performans backend ayarları.
- Run'ın yerel veya Colab olduğunu, artifact aktarımını ve doğrulamasını gösteren metadata.
- Evaluation seti ve metrik hesaplama kodunun commit/digest bilgisi.

**Çekirdek kural:** Yalnızca aynı model boyutu veya aynı token tensor şekli, iki modeli karşılaştırılabilir kılmaz. Özellikle Mimi ile eğitilen token embedding'leri yeni Kaiwen Codec tokenlarıyla uyumlu değildir; codec kimliği eşleşmeden checkpoint ve cache birleştirilemez.

## 8. Karşılaştırılabilirlik ve compatibility gate

Forge iki run'ı aynı grafikte gösterebilir; fakat “A, B'den daha iyi” karşılaştırması yapabilmek için bir **compatibility gate** gerekir. Gate, her farkı yasaklamaz; nedenini ve karşılaştırma türünü açıkça belirler.

| Kontrol | Strict paired comparison | Exploratory comparison |
|---|---|---|
| Aynı değerlendirme örnekleri ve split fingerprint | Zorunlu | Farklıysa uyarı |
| Aynı eval pipeline sürümü | Zorunlu | Dönüşüm açıkça yazılır |
| Aynı task/çıktı türü | Zorunlu | Farklı task doğrudan kıyaslanamaz |
| Aynı hesaplama/çalıştırma cihazı (latency) | Zorunlu veya normalize edilmiş ayrı deney | Uyarı ve cihaz etiketi |
| Codec örnekleme standardı | Kıyas amacına bağlı | Aynı içerik/örnek süre şartı |
| Eğitilen adım/token/compute bütçesi | Kontrol edilen faktör | Açık confound raporu |
| Aynı seed kümesi | Tercih edilir | Fark uyarısı |
| Aynı kontrol değişkenleri | Zorunlu | Bilimsel sınırlama beyanı |
| Aynı kalite metriği yönü ve birimi | Zorunlu | Karşılaştırma reddi |

Örneğin **codec araştırması** için codec kimliğinin değişmesi deneyin amacı olabilir, ancak referans PCM kayıtları, bitrate bütçesi, değerlendirme örnekleri ve kalite ölçümü aynı tutulmalıdır. **Transformer Depth ablation** için codec ve dataset sabitlenir. **VRAM testi** için GPU ve backend sürümleri kritik kontrol değişkenidir. Benzerlik, karşılaştırma amacına göre tanımlanmalıdır.

Karar durumları:

- `compatible`: önceden belirlenmiş tüm gereklilikler karşılandı.
- `compatible_with_caveats`: analiz yapılabilir; uyarılar ve koşullar raporda görünür.
- `incomparable`: karşılaştırma bilimsel sonuç olarak üretilemez; yalnızca ham tablo gösterilebilir.
- `insufficient_provenance`: veri/codec/protokol kimliklerinden biri eksik; sonuç otomatik yayımlanmaz.

Yalnızca başarı veren kolları filtreleyip başarısızları saklamamak selection bias oluşturur. OOM ve timeout koşulları da araştırmanın maliyet ve kararlılık boyutunda yer alır.

## 9. Metric registry ve semantik tipler

Metrik adlarının rastgele string'ler olarak saklanması, aynı isimle farklı anlamların raporlanmasına yol açar. V1'de küçük bir **metric definition registry** tasarlanmalıdır. Her tanım `name`, `version`, `description`, `unit`, `direction`, `aggregation`, `sampling_unit`, `valid_range`, `provenance` ve `source` alanlarına sahip olmalıdır.

| Metrik | Birim | Yön | Bağlam ve sınırlama |
|---|---|---|---|
| `train.loss` | nats/token veya tanımlı loss | ↓ | Aynı architecture/loss masking olmadan doğrudan kıyaslanamaz |
| `validation.loss` | tanımlı loss | ↓ | Dataset/split/loss formülüyle birlikte anlamlı |
| `train.tokens_per_second` | token/s | ↑ | Token tanımı (frame mi, codebook token mı?) belirtilmeli |
| `eval.codec.stft_distance` | protokole göre | ↓ | Parametreleri ve audio formatı sabit |
| `eval.codec.si_sdr_db` | dB | ↑ | Kısıtlı anlam; algısal kalite yerine geçmez |
| `eval.asr.cer` | fraction | ↓ | ASR referansı, metin normalizer ve dil belirtilmeli |
| `eval.speech.pairwise_preference` | fraction | ↑ | Kör, eşlenmiş değerlendirici protokolü şart |
| `perf.ttfa_ms` | ms | ↓ | Başlangıç ve bitiş timestamp tanımı sabit |
| `perf.rtf` | ratio | ↓ | Gerçek PCM süresi ve cihaz belirtilmeli |
| `perf.peak_vram_bytes` | byte | ↓ | CUDA allocator ile gerçek device peak ayrımı kaydedilir |
| `perf.interruption_stop_ms` | ms | ↓ | Kullanıcı kesmesi → duyulan sesin durması |
| `eval.turn_taking.success_rate` | fraction | ↑ | Kurallı anotasyon, gerçek konuşma corpus'u gerek |
| `eval.agent_handoff.success_rate` | fraction | ↑ | İleriki aşama; sadece routing başarısı değil uçtan uç doğrulama |

`accuracy`, `quality` ve `latency` gibi bağlamsız isimler registry'ye alınmaz. `loss` adını ortak grafikte çizmek mümkündür ama q0 ve q1..q7 ağırlıkları farklıysa sayısal karşılaştırma **incomparable** olur. Codec reconstruction loss ile speech dialogue loss aynı ölçekte değildir.

**Yön:** `minimize`, `maximize`, `target_range` değerleri kabul edilir. **Ölçülmedi:** `null` ve açık `reason` saklanır; `NaN`, `Infinity` veya `0` olarak doldurulmaz. **Geçersiz ölçüm:** metrik kaydı ayrı `invalid` bayrağı taşır ve ham kayıt silinmez. **P95:** en az örnek sayısı, hesaplama yöntemi, percentile estimator ve gözlem penceresi kaydedilir.

## 10. Structured metric event sözleşmesi

MEVCUT Kaiwen Speech `Trainer._log()` stdout'a `step=... train_loss=... lr=...` benzeri insan-okur satırlar yazabilir. Forge V1 bu stdout'u sürümüne sabitlenmiş küçük bir adapter ile geçici olarak ayrıştırabilir. Uzun vadede Speech'e eklenebilecek **opsiyonel** JSONL metric sink'i daha güvenilir olacaktır; bunun geliştirilmesi Speech tarafında ayrı PR gerektirir.

### 10.1 Örnek metric event

```json
{
  "schema_version": 1,
  "event_id": "example-event-000013",
  "run_id": "example-run-001",
  "attempt_id": "example-attempt-001",
  "sequence": 13,
  "event_time_utc": "2026-10-30T12:00:00Z",
  "ingested_at_utc": "2026-10-30T12:00:01Z",
  "metric": {
    "name": "train.loss",
    "definition_version": "1.0.0",
    "value": 2.7812,
    "unit": "nats/token",
    "direction": "minimize",
    "step": 100,
    "samples_seen": null,
    "source": "speech_trainer_stdout_adapter",
    "aggregation": "logged_interval_mean"
  }
}
```

Sayı örnektir, ölçülmüş Kaiwen sonucu değildir. Kayıt anahtarı `(run_id, attempt_id, sequence)` tekrar eklendiğinde idempotent çalışır. Aynı anahtar farklı payload ile gelirse silent overwrite yerine conflict üretilir. Kaybolan event sequence aralığı raporlanır. Event zamanının worker'dan geldiği, veritabanına giriş zamanından farklı olabileceği unutulmaz. UI grafiği event sırasını `step` ve `attempt_id` ile doğru gösterir.

### 10.2 Event veri doğrulaması

- `metric.value` finite sayı veya tanımlanmış `missing` durumu olmalıdır; JSON NaN kabul edilmez.
- Birim ve yön metric registry ile uyumlu olmalıdır.
- `step` negatif olamaz; daha düşük step için kayıt ancak yeni attempt/resume bağlamında anlamlıdır.
- Tek bir run altında farklı `dataset_manifest_sha256` veya codec kimliği eventlere sonradan eklenemez.
- Parametre sayısı gibi `run.summary` metrikleri için `step` boş olabilir; training curve eventleri için step zorunludur.
- Worker'ın saati yanlışsa ingress zamanı ayrı korunur; sırayı yalnızca wall clock belirlemez.
- Kayıp stdout metrikleri başarısızlık olarak raporlanır, “smooth graph” üretmek için uydurulmaz.
- Metrik value çok büyük/küçük veya out-of-range ise tanımlı validation error oluşur.

## 11. Aggregation: grafiklerin anlamını koruma

Her grafik şu bilgileri açıkça taşımalıdır: x-ekseni (step, seen tokens, wall time, GPU hours), y-ekseni (metrik, birim), renk/kol (run), smoothing politikası, örnek sayısı, eksik aralıklar, attempt sınırları ve veri kaynağı. Smoothing yalnızca görüntüleme katmanında yapılır; ham seri değiştirilmez.

- **Moving average:** Pencere boyutu raporlanır ve tek başına sonuca kaynak olamaz.
- **Exponential moving average:** Alfa değeri ve initialization politikası açık olmalıdır.
- **Downsampling:** Min/max veya diğer seçilen yöntem belirtilir; spike veya OOM öncesi anomali kaybolmamalı.
- **Epoch/step:** İki koşulun dataset boyutları farklıysa step numarasını “eşit eğitim” kanıtı olarak gösterme.
- **Token count:** Codec frame rate farklıysa token birimi ayrıştırılır: `audio_frames`, `codebook_tokens`, `raw_audio_seconds`.
- **Resource efficiency:** GPU-saati ve eğitimde kullanılan toplam ses saati bağımsız metriklerdir.
- **Outlier:** Kaynağı bozuk ölçümden değilse istatistiksel aykırılık diye sessizce atılmaz.

**Yanlış grafik örneği:** 12.5 Hz Mimi ve 50 Hz Kaiwen tokenlarının 10k step'te ulaştığı loss değerini doğrudan aynı ölçekte kıyaslamak. Farklı codec vocab ve frame semantics nedeniyle bu eşit kalite veya hesaplama demek değildir.

## 12. Run sonucu, failure taxonomy ve censored observations

Run'ın `completed` olması modelin iyi olduğu anlamına gelmez. `completed`, execution pipeline'ın sözleşmesini tamamladığı anlamına gelir. `scientifically_valid`, `comparable`, `candidate` ve `release_approved` birbirinden ayrı alanlardır. Aşağıdaki durumlar birbirine karıştırılmamalıdır:

| Durum | Deney kaydı | Birincil analiz | Not |
|---|---|---|---|
| Başarıyla bitti, artifact doğrulandı | Tam | Uygun olabilir | Eval protokolü geçmeli |
| OOM | Tam failure kaydı | Belirlenmiş resource outcome analizine dahil | Kalite metriği uydurulmaz |
| Timeout | Tam failure kaydı | Censored veri olarak işaretlenir | Kullanılan bütçe ve timeout sınırı görünür |
| Kullanıcı iptal etti | Tam | Bilimsel sonuç için genelde dışarıda; neden açık | Yanlış başarı iddiası yok |
| Worker çöktü | Tam | Attempt-level rapor | Kısmi metriği final sonuç sanma |
| Yarım checkpoint | Quarantine | Hayır | SHA doğrulama geçmeden kullanma |
| Eval seti eksik | Run var, eval başarısız | Hayır | Yeniden değerlendirme gerekir |
| İzin/lisans belgesi yok | Run karantinada | Hayır | Yayın ve eğitim politikası kontrolü |

`censored` teknik terimi, istenen gözlemin tamamı değil yalnızca bir sınırının veya kısmının gözlenmiş olması demektir. Örneğin timeout ile kesilen run'ın son loss değeri, normal bitmiş run'ın final loss'u olarak aynı grafiğin son satırına konamaz. Gerekirse ayrı “training progress at fixed budget” analizi yapılır.

## 13. SQLite kayıt modeli ve migration ilkeleri

Training Pipeline dokümanında tanımlanan `experiments`, `runs`, `attempts`, `run_events`, `metrics`, `artifacts`, `idempotency_keys` tabloları **temel kanonik çekirdek** olarak kalır. Bu belge bunların yerine ikinci bir DB sistemi tasarlamaz. Araştırma katmanı küçük ek tablolarla genişletilir.

```sql
-- HEDEF şema taslağı; mevcut Forge migration'ı değildir.
CREATE TABLE IF NOT EXISTS studies (
  study_id TEXT PRIMARY KEY,
  slug TEXT NOT NULL UNIQUE,
  research_question TEXT NOT NULL,
  created_at_utc TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS experiment_protocols (
  protocol_id TEXT PRIMARY KEY,
  experiment_id TEXT NOT NULL,
  version TEXT NOT NULL,
  protocol_sha256 TEXT NOT NULL,
  protocol_json TEXT NOT NULL,
  frozen_at_utc TEXT NOT NULL,
  UNIQUE(experiment_id, version),
  UNIQUE(experiment_id, protocol_sha256)
);
CREATE TABLE IF NOT EXISTS evaluations (
  evaluation_id TEXT PRIMARY KEY,
  run_id TEXT NOT NULL,
  attempt_id TEXT,
  checkpoint_artifact_id TEXT NOT NULL,
  suite_id TEXT NOT NULL,
  suite_digest TEXT NOT NULL,
  status TEXT NOT NULL,
  metrics_artifact_id TEXT,
  created_at_utc TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS comparisons (
  comparison_id TEXT PRIMARY KEY,
  experiment_id TEXT NOT NULL,
  protocol_id TEXT NOT NULL,
  comparison_sha256 TEXT NOT NULL UNIQUE,
  report_artifact_id TEXT,
  created_at_utc TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS metric_definitions (
  name TEXT NOT NULL,
  version TEXT NOT NULL,
  definition_json TEXT NOT NULL,
  PRIMARY KEY(name, version)
);
```

Gerçek uygulamada foreign key, index, FK referanslarının canonical tablolarıyla uyuşması, `PRAGMA foreign_keys=ON`, SQLite WAL, migration versiyonu, transactional event yazımı ve `CHECK` kısıtları eklenmelidir. `metrics` tablosunun kolonları Training Pipeline ile tutarlı olmalıdır; `metric_definitions` geriye dönük semantik katmandır, ayrı bir metric event store değildir.

Migration sırası: migration öncesi yedek → yeni tablo ekle → veri backfill → salt-okunur doğrulama → kanonik read switch → eski API'yi kontrollü kapat. İçe aktarma sırasındaki hata transaction rollback üretmeli; kısmen oluşmuş karşılaştırma `completed` işaretlenmemelidir. Değiştirilemez audit kayıtları ve logical archive yaklaşımı tercih edilir.

## 14. İstatistiksel analiz: fark ile kanıt arasındaki sınır

Araştırmanın ilk görevi “yeni modelin skoru daha yüksek” cümlesini üretmek değil; gözlenen farkın örnekleme değişkenliğine ne ölçüde dayanıklı olduğunu anlamaktır. Forge V1 ham örnek başına metrikleri ve karşılaştırma planını korur; basit özet, eşlenmiş fark, güven aralığı ve grafik üretir. İleri istatistiksel metotlar ihtiyaçla eklenir, ama metodun varsayımları rapordan saklanmaz.

### 14.1 Deney birimini doğru seçme

Bir video kaydının 500 segmenti, 500 bağımsız konuşmacı değildir. Tek bir insanın 50 ses kaydı da 50 bağımsız örnek değildir. Bootstrap, train/test bölme ve güven aralığı hesabında bağımlılık yapısına dikkat edilmelidir.

- Codec reconstruction değerlendirmesinde aynı kayıt içinden segmentler bağımlıysa yeniden örnekleme birimi `recording_group_id` olabilir.
- Speaker robustness deneyinde aynı speaker'ın farklı kayıtları bağımlıdır; birim `speaker_id` veya speaker+source grup olabilir.
- Diyalog çalışmasında aynı konuşmanın çoklu turn'leri bağımsız kabul edilmez; `conversation_id` ile kümeleme gerekir.
- Donanım latency tekrarlarında aynı cihaz, aynı oturum ve arka plan sıcaklık durumu bağımlılık yaratabilir; `session_id`/device trial birimi kaydedilir.
- İnsan dinleme testlerinde aynı değerlendiricinin tüm oyları bağımsız değildir; değerlendirici ve örnek etkisi ayrı ele alınmalıdır.

**Pseudoreplication** yapılmaz: aynı eğitim run'ından yüzlerce ses dosyası üretmek, yüzlerce farklı training seed denemesi yapılmış gibi raporlanmaz. Belirsizlik hem model-run düzeyinde hem örnek düzeyinde ele alınabilir; hangisinin hedeflendiği söylenir.

### 14.2 Eşlenmiş fark ve bootstrap

Aynı kayıt `i` üzerinde candidate ve baseline skoru olsun. Eşlenmiş fark:

\[
\Delta_i = m_i^{(candidate)} - m_i^{(baseline)}
\]

Yönü `maximize` olan metrik için pozitif fark iyileşmedir; `minimize` olan metrik için negatif fark iyileşmedir. Forge karar tablosunda `improvement_direction` ve değişim işareti birlikte bulunur; eksenleri otomatik ters çevirip kullanıcıdan saklamaz.

Basit paired bootstrap prosedürü:

1. Sabit, kayıtlı PRNG seed ile bağımsız yeniden örnekleme gruplarını belirle.
2. Her tekrar için aynı grup indekslerini baseline ve candidate'a birlikte uygula.
3. Grup içindeki tüm örnekleri ilgili gruplarla birlikte taşı.
4. Önceden tanımlanmış agregasyonu uygula; `delta` hesapla.
5. Önceden belirlenmiş güven aralığı yöntemini (örn. percentile veya BCa) açıkça etiketle.
6. Fark dağılımını ve örnek/grup sayılarını raporla.

Bootstrap tekrarı sayısı hesaplama kararlılığı içindir; **asıl örneklem sayısını artırmaz**. Önceden yalnızca beş bağımsız konuşmacı varsa 10.000 bootstrap tekrarından güvenilir evrensel sonuç çıkarılmaz. Ayrıca sıfırın güven aralığı içinde olması “iki yöntem eşdeğer” demek değildir; eşdeğerlik için ayrı marj ve test gerekir.

### 14.3 Eğitim seed belirsizliği

Farklı model initialization, veri sırası ve sampling seed'leri eğitim sonucunu değiştirebilir. İdeal kıyaslama: aynı seed kümesiyle eşlenmiş run'lar + aynı evaluation seti. Seed başına metrik değerleri ve basit ortalama/medyan raporlanır; az sayıda seed için belirsizlik sınırlaması görünür. Özellikle küçük GPU bütçesinde “üç seed kullandık, bu yüzden kanıt kesin” iddiası yapılmaz.

Bir seed'in bozulmuş olması önceden belirlenmiş dışlama kuralı dışında keyfi dışlama nedeni olamaz. OOM başarısızlığı, kaynak maliyeti ve algoritma kararlılığı açısından raporun parçasıdır. Bir kol 3/3, diğeri 1/3 tamamlandıysa kalan tek kalite sonucunu eşit güvenle gösteremeyiz.

### 14.4 Çoklu hipotez testi ve keşif

On farklı metrikte veya onlarca model varyantında en iyi görünen sonucu seçmek false positive riskini yükseltir. Birincil hipotez ailesi önceden belirtilir; uygun durumlarda Holm gibi çoklu karşılaştırma düzeltmesi kullanılabilir. Bonferroni, Holm, FDR veya Bayesian model karşılaştırması seçenekleri sihirli varsayılan yapılmaz; analiz türü ve varsayımlar araştırma planıyla kilitlenir. Post-hoc keşif ile confirmatory analiz ayrı raporlanır.

### 14.5 Etki büyüklüğü ve pratik önem

Bir istatistiksel fark, kullanıcı için önemli bir fark olmayabilir. Forge karşılaştırma çıktısında `absolute_delta`, `relative_delta`, `confidence_interval`, `predefined_minimum_meaningful_delta` ve `sample_count` alanlarını ayırmalıdır. Örneğin p95 ses gecikmesinde çok küçük iyileşme, duyulabilir kaliteyi bozuyorsa otomatik “kazandı” sayılmaz. Birincil metrik eşiği geçse bile güvenlik veya performans guardrail'i başarısızsa deney `tradeoff` sonucunu verir.

## 15. İnsan dinleme değerlendirmesi ve tarafsız karşılaştırma

Speech-native ses kalitesi otomatik metrikle tam ölçülemez. Forge, insan değerlendirmesi verisini toplayan özel UI yazmak zorunda değildir; fakat kör ve kontrollü bir dinleme protokolünü, kaydedilmiş oyları ve veriye ilişkin izinleri saklayabilmelidir. Kaiwen Corpus'un anotasyon UI'sı varsa değerlendirme için ayrı bir modül tasarlanabilir; sahiplik önceden netleştirilmeden iki ayrı ürün yapılmaz.

### 15.1 A/B tercih testleri

- Kaynak kayıtlar aynı olmalı; iki model de aynı girişle test edilmelidir.
- Karşılaştırma sırası A/B rastgeleleştirilir ve yeniden üretilebilir randomization seed'i saklanır.
- Model adı ve konfigürasyonu değerlendiriciye gösterilmez.
- Ses yüksekliği güvenli ve mümkün olduğunca eşit algısal seviyede ayarlanır; normalize işlemi içeriği bozuyorsa kayıt tutulur.
- Örnek süreleri, kesilen sesler ve sessizlik politikası belirlenir.
- “A daha iyi”, “B daha iyi”, “eşit/kararsız” seçenekleri analizde ayrı kodlanır; zorla iki seçenek sunmak yanlılık getirebilir.
- Her değerlendiriciye aynı modelin tüm örnekleri blok halinde dinletilmez; sıra etkisi dengelenir.
- Dinleme cihazı ve ortamı sorulabilir, ancak kişisel veri minimizasyonu uygulanır.
- Referans ses, gerçek/sentetik kaynak, Türkçe aksan ve örnek türü analize dahil edilir.

### 15.2 İnsan değerlendirici kayıt şeması

```json
{
  "schema_version": 1,
  "evaluation_id": "example-listening-eval-001",
  "trial_id": "example-trial-014",
  "rater_pseudonym": "rater-07",
  "source_group_id": "example-conversation-04",
  "presentation_seed": 9191,
  "shown_order": ["candidate_b", "baseline_a"],
  "response": "left_preferred",
  "tie_allowed": true,
  "quality_dimension": "naturalness",
  "duration_ms": 18000,
  "recorded_at_utc": "2026-10-30T12:34:00Z"
}
```

Bu JSON model adlarını normalde rater UI'ya vermemeli; `shown_order` sahası araştırma backend'inde değerlendirme kapandıktan sonra çözülmelidir. Üstteki örnek öğreticidir; gerçek veri toplama sisteminde körlüğün korunması için ayrı `presentation_id -> artifact_id` eşlemesi ve erişim kontrolü gerekir. IP adresi veya ad-soyad varsayılan olarak tutulmaz.

### 15.3 MOS ve diğer öznel ölçekler

Mean Opinion Score (MOS) kullanılacaksa ölçek ankrajları, dil, soru ifadesi, örnek rastgeleleştirmesi, minimum değerlendirme süresi, değerlendirici eğitim örnekleri ve sonuç agregasyonu sürümlenmelidir. Bir 1–5 doğal konuşma puanı ile codec rekonstrüksiyon puanı eşdeğer değildir. Rapor, güven aralığını ve değerlendirmeci sayısını açıklamalı; 4.2 MOS gibi bir ortalama kaynak koşulları belirtilmeden pazarlama iddiası yapılmamalıdır.

### 15.4 Etik ve mahremiyet

Orijinal insan konuşmalarının lisans ve izinleri ayrı kontrol edilir. Kişisel/özel sesler varsayılan olarak public report'a eklenmez; raporda anonimleştirilmiş, yayın izni olan örnekler kullanılır. Değerlendiricilerin kimlikleri pseudonymous, erişim kontrollü tutulur. Hassas ses örnekleri eğitim/rapor depolarında gereksiz kopyalanmaz. İnsana ait ses üretimi ile sentetik ses birbirinden açıkça ayrılır.

## 16. Speech-native araştırma metrik matrisi

Farklı araştırma sorularında tek bir ana skoru zorlamak yerine metriklerin amacı tanımlanır.

| Araştırma hattı | Birincil metrik adayı | Guardrail / yardımcı metrik | Kritik koşul |
|---|---|---|---|
| Codec waveform kalitesi | Kör A/B tercih veya önceden seçilen akustik kalite | bitrate, RTF, frame latency | Aynı kaynak PCM / sample rate |
| Codec token verimliliği | Kalite–bitrate eğrisi | codebook kullanım dağılımı | Q/K/frame rate kontrolü |
| Speech continuation | Held-out sample quality ve uygun valid loss | free-running kalite, EOS | Aynı codec/valid split |
| Diyalog yanıtı | Kör uygunluk/doğallık değerlendirmesi | CER, yanıt süresi, duraksama | Sadece transcript kalitesi yeterli değil |
| Parallel vs Depth | Ön kayıtlı kalite hedefi | compute, frame latency, EOS | Parametre ve eğitim bütçesi belirtilir |
| GQA / KV cache | Decode latency, bellek | doğru token parity | Aynı checkpoint, aynı prompt |
| Streaming codec | Chunk/offline parity, canlı RTF | boundary artifacts, jitter | Son flush testi |
| 2 GB VRAM | Peak gerçek device memory | kalite ve TTFA | Tam sistem yüklü, tek cihaz |
| Barge-in | İşitilir ses durma süresi | stale frame oranı, race | Gerçek playback queue testi |
| Turn-taking | Doğru sıra alma başarısı | false interruption, backchannel | Etiketli doğal diyalog |
| Agent handoff | Doğru routing + güvenli tamamlanma | konuşma akışı, latency | Live/Agent ortak entegrasyonu |

**Türkçe içerik:** günlük konuşma, ekler, özel adlar, sayılar, tarih/saat ifadeleri, farklı aksan ve konuşma hızları, code-switching örnekleri, duraksamalar ve üst üste konuşma koşulları ayrı kategoriler olarak raporlanabilir. Ancak herhangi bir kategori yoksa metrik oluşturmak için sahte örnek eklenmez; veri kapsamı yetersiz diye raporlanır.

## 17. Kaynak/veri sızıntısı ve test seti yönetimi

Corpus'un train/validation/test split'leri stable manifest kimliğiyle gelir. Ancak stable split tek başına leakage olmadığını kanıtlamaz. Aynı konuşmanın farklı parçası, aynı sesli kitap bölümü, aynı videonun yeniden yüklenmiş varyantı veya sentetik bootstrap'ın kaynak metni farklı split'lerde bulunabilir.

Zorunlu kontroller:

1. Train/test arasında birebir SHA-256 eşleşmesi; near-duplicate tespiti için ayrı algoritma ve güven eşiği.
2. Source ID, recording ID, speaker ID, conversation ID bazlı grup ayrımı.
3. Aynı uzun kayıttan kırpılan segmentlerde overlap ve komşu bağlam kontrolü.
4. Sentetik örneklerde prompt/metin şablonlarından kaynaklanan yakınlık uyarısı.
5. Human evaluation örneklerinin daha önce eğitim verisine sızmadığına ilişkin provenance.
6. Test setinin dondurulması; sürekli aday seçimi için yalnızca validation kullanılması.
7. Kaç örneğin neden dışlandığı ve sonuçların kaç örnek üzerinden hesaplandığının kaydı.
8. Corpus export yeniden üretildiyse manifest digest'i değişir; eski run yeni split'e otomatik bağlanmaz.

Test setine dair bir sızıntı tespit edilirse `evaluation_status=invalidated` ve gerekçe kaydedilir. Tarihsel raporların üzerine sessizce yeni puan yazılmaz. Eski rapor **superseded** durumuna alınır, yeni değerlendirme yeni ID ile çalışır. Bitirme projesi sunumunda yalnızca düzeltilmiş sonuç gösterilirken önceki sonuçların neden geçersizleştiği de metodoloji notunda korunur.

## 18. Hiperparametre aramaları: küçük ve kontrollü başla

Forge V1 için grid/random search motoru zorunlu değildir; **deney kollarının ve seed tekrarlarının elle/recipe üzerinden tanımlanması** yeterlidir. Fakat arama tasarımının veri modeli ilk günden uyumlu olmalıdır.

### 18.1 Arama türleri

- **Grid search:** Küçük, açık arama alanlarında bütün kombinasyonların kontrolü kolaydır; toplam maliyet hızla artar.
- **Random search:** Az sayıda etkili parametre olduğunda daha geniş alan keşfi sağlayabilir; seed ve sampling distribution kaydedilir.
- **Successive halving/ASHA:** Erken durdurulan deneyler compute tasarrufu sağlayabilir; ancak erken performans ile son kalite ilişkisi doğrulanmalıdır. Özellikle ses üretiminde düşük ilk loss'un nihai konuşma kalitesini garanti etmediği unutulmaz.
- **Bayesian optimization:** Daha sonraki aşama; veri hacmi ve hesaplama maliyeti haklı çıkarmadan servis/karmaşıklık eklenmez.

### 18.2 Maliyet hesabı

Arama başlatılmadan önce maksimum kombinasyon sayısı, seed sayısı, her run'ın GPU saat bütçesi, veri ön işleme maliyeti ve saklanacak checkpoint boyutu hesaplanır. Arama şu bileşenleri de hesaba katar:

\[
C_{max} = N_{config} \times N_{seed} \times H_{per\_run}
\]

Bu, GPU saatleri için basit **üst sınır tahmini**dir. Codec ön işleme, evaluation, retry ve veri transferi ek maliyettir. Kullanıcı bütçe sınırı belirlediyse Forge bütçeyi aşacak koşulları önceden uyarır veya bloklar. Otomatik sonsuz retry mekanizması kurulmaz.

### 18.3 Arama kaydının değişmezliği

Bir arama ortasında search space değişirse yeni `search_revision` açılır. Kullanıcının yeni aday eklemesi mümkündür; fakat eski adaylarla aynı örneklem planını uyguladığı iddia edilemez. Budget-based early stopping ile biten run'lar tam eğitilmiş run'lardan ayrı kategoride raporlanır. “En iyi epoch” seçimi validation'a göre yapılır; final test'e göre yapılmaz.

## 19. Ablation çalışmaları: hangi bileşen fayda sağlıyor?

Ablation, bir değişkenin katkısını izole etmek için aynı başlangıç koşullarında sistematik kaldırma/değiştirme çalışmasıdır. Çok değişken aynı anda değişirse gözlenen farkın nedeni belirsiz kalır. Kod yapısında `ablation_group`, `baseline_run_id`, `factor_name`, `factor_level`, `controlled_variables_digest` ve `notes` tutulmalıdır.

### 19.1 Kaiwen Speech için önerilen ilk ablation ailesi

| Deney | Baseline | Aday | Sabit tutulması gerekenler | Özel risk |
|---|---|---|---|---|
| S1 | Parallel q0..q7 | Depth q0→q7 | Mimi codec, split, seed, budget | Parametre sayısı artabilir |
| S2 | 0 frame delay | 1 frame acoustic delay | Aynı model, codec ve prompt | Diyalog yolunda hizalama henüz doğrulanmalı |
| S3 | Full-prefix generation | KV cache incremental | Aynı checkpoint, sampling | Token parity ve RoPE offset |
| S4 | MHA | GQA | Benzer kapasite ve eğitim budget | KV head sayısı kaliteyi etkiler |
| S5 | Mimi Q8 / 12.5 Hz | Kaiwen Codec Q4 / 50 Hz | Aynı referans ses, sabit kalite ölçümü | Token loss doğrudan kıyaslanamaz |
| S6 | 50 Hz codec | 25 Hz codec | Bitrate, aynı reconstruction görevi | Latent/frame tradeoff |
| S7 | Real-only subset | Real+synthetic | Benzer source composition, compute | Sentetik ses artefaktları |
| S8 | Standard attention | Experimental architecture | Aynı input ve maliyet raporu | Mamba/hybrid henüz mevcut değil |

S2 için **mevcut kod gerçeği** özellikle önemlidir: `acoustic_delay_frames=1` işlemi continuation dataset yolunda mevcut, dialogue yolu aynı dönüşümü uygulamıyor. Bu, dialogue delay deneyinde önceden çözülmesi gereken bir uygulama/test açığıdır; Forge sonuçları ölçülmeden başarı beklememelidir.

S5 codec değişimi sadece architecture ablation değildir; iki ayrı tokenizer temsilinin **aynı reconstruction hedefinde** araştırılmasıdır. Yeni codec tokenlarını Mimi ile eğitilmiş Transformer'a doğrudan vermek geçersiz bir karşılaştırmadır. Codec özel deneylerinde önce codec-only kalite, sonra her codec için ayrı eğitilmiş Speech modeli değerlendirilir.

### 19.2 Ablation raporunun biçimi

Her deneyde şu alanlar zorunlu: faktör, seviyeler, değişmeyen parametreler, eğitim bütçesi, dataset/split kimliği, artifact kimliği, ön kayıtlı metrikler, sonuç tablosu, belirsizlik, örnek sayısı, confound uyarıları. “İyi oldu” veya “daha doğal hissettirdi” gibi izlenimler nitel gözlem olarak saklanabilir; ölçülmüş veri olmadan nihai sonuç olmaz.

## 20. Multi-objective karar ve Pareto frontier

Kaiwen Speech'in ürünü tek bir doğruluk sayısıyla seçilemez. En az üç ana eksen vardır: **ses/doğal konuşma kalitesi**, **ilk duyulan ses gecikmesi/RTF**, **toplam VRAM**. Buna compute maliyeti, codec bitrate ve stabilite eklenebilir. Bir modelin daha iyi olması, araştırma senaryosuna bağlıdır.

Bir model A, tüm amaçlarda B'den kötü değil ve en az birinde daha iyiyse, B **dominated** olur. Dominated olmayan adaylar Pareto frontier'ı oluşturur. Forge grafiği karar desteği verir; kullanıcı için tek bir “best model” sayısını zorlamaz. Ürün hedefi olarak 2 GB VRAM sınıfı varsa bu, bilimsel çalışma için katı bir kısıt olabilir; fakat henüz bu sınırı sağlayan model bulunduğunu iddia etmeyiz.

Üç örnek karar:

- Kalite üstün fakat VRAM 2 GB'ı aşıyorsa: araştırma için değerli, düşük VRAM ürün adayı değil.
- VRAM hedefini geçiyor fakat yanıtlar anlamsızsa: performans demosu olabilir, konuşma modelinin başarı kanıtı değil.
- Çok düşük latency fakat sık stale audio/underrun varsa: kullanıcı deneyimi açısından başarısız.

Kısıtları geçmeyen adaylarla başarı raporu üretmek yerine `constraint_failed` durumu ve hangi ölçümün sınırı aştığı belirtilir.

## 21. Evaluation artifact ve bağımsız yeniden değerlendirme

Eğitim run'ı kapandıktan sonra aynı checkpoint için birden çok değerlendirme yapılabilmelidir. Bu, eval protokolünün değiştiğinde checkpoint'i tekrar eğitmeden yeniden ölçmemizi sağlar. Ancak farklı eval suite'lerden gelen sonuçlar sessizce aynı seri içine alınmaz.

Bir değerlendirme kaydı en az:

- `evaluation_id`, `source_run_id`, `checkpoint_artifact_id` ve doğrulanmış checkpoint hash'i.
- Eval suite kimliği, sürümü, hash'i, task ve örnek sayısı.
- Metric definitions sürümleri ve hesaplama kodu commit'i.
- Girdi ses formatı, output decoder, sampling temperature/seed/max frames.
- Ölçüm cihazı; gerekiyorsa CPU, GPU, sürücü, senkronizasyon politikası.
- Sonuç dosyaları, örnek-başına skorlar, dışlanan örnekler ve gerekçeler.
- Değerlendirme süresi, hata logları ve kısmi başarısızlık durumu.

**MEVCUT kod**: `kaiwen-speech/scripts/evaluate_continuation_audio.py` gerçek continuation çıktıları için çeşitli ses örnekleri üretebiliyor; `scripts/benchmark.py` tek bir tam forward ölçüyor. Bu scriptler **Forge'un değerlendirme platformu değildir** ve kendiliğinden uçtan uca TTFA/RTF, kör MOS veya tam latency benchmark sağlamaz. Forge'un hedef adapter'ı bu kaynakları uygun metadata ile tüketebilir; yeni eval motoru gerekiyorsa önce Speech tarafında modele özgü ölçüm yazılır.

### 21.1 Örnek değerlendirme manifesti

```json
{
  "schema_version": 1,
  "evaluation_id": "example-eval-001",
  "source_run_id": "example-run-001",
  "checkpoint_artifact_id": "example-checkpoint-001",
  "suite": {
    "name": "turkish_continuation_holdout",
    "version": "1.0.0",
    "digest": "EXAMPLE_SUITE_DIGEST"
  },
  "settings": {"temperature": 0.0, "seed": 42, "max_new_frames": 64},
  "device": {"type": "cpu", "id": "example-device"},
  "metrics": [{"name": "validation.loss", "value": 2.5, "unit": "nats/token"}],
  "sample_count": 20,
  "excluded_count": 0,
  "status": "example_not_executed"
}
```

Bu manifest örneğinde verilen değerler öğreticidir ve gerçek ölçüm değildir. `status=example_not_executed` özellikle rapor üreticisinin bunu çalışma sonucu olarak kullanmaması için bulunur.

## 22. Rapor üretimi: tekrarlanabilir kanıt paketi

Her comparison bir **machine-readable** `comparison.json` ve insan-okur `report.md` oluşturmalıdır. İlk V1 için statik Markdown + CSV/JSON yeterlidir; interaktif dashboard zorunlu değildir. Rapor, grafik üretim koduna ve veri hash'ine referans verir. Rapor üretiminde kullanılan analiz kodu sürümü kaydedilir; aynı girdilerle aynı raporun yeniden oluşturulması deterministik olmalıdır (timestamp gibi oynak alanlar sabit veya ayrı metadata olarak ele alınır).

Zorunlu rapor bölümleri:

1. Soru/hipotez, yayınlandığı protokol ve araştırma türü.
2. Baseline ile candidate kimlikleri ve training config farkları.
3. Dataset/codec/split uyumluluk kontrolü.
4. Primary metric, örnek sayısı, eşlenmiş fark ve güven aralığı.
5. Secondary metrics, kalite ve kaynak guardrail'leri.
6. Başarısız run'lar, dışlamalar, eksik ölçümler.
7. İstatistiksel yöntem, varsayımlar ve çoklu test düzeltmeleri.
8. En az bir nitel örnek seçimi için önceden belirlenmiş politika; cherry-picking yok.
9. Threats to validity: neden bu sonuç gerçek dünyada değişebilir?
10. Reproduction recipe: kaynak commit, veri kimliği, komut ve artifact bağlantıları.
11. Sonuç: `supported`, `not_supported`, `inconclusive`, `tradeoff`, `invalidated` gibi açık durum.
12. Sonraki deney önerisi ve açıklanan kapsam sınırı.

### 22.1 Report şeması taslağı

```json
{
  "schema_version": 1,
  "comparison_id": "example-comparison-001",
  "protocol_id": "example-protocol-1.0.0",
  "baseline_run_ids": ["example-run-a"],
  "candidate_run_ids": ["example-run-b"],
  "compatibility": {
    "status": "compatible_with_caveats",
    "caveats": ["Pilot örneklem; yeterli güç analizi yapılmadı."]
  },
  "primary_result": {
    "metric": "eval.speech.pairwise_preference",
    "unit": "fraction",
    "estimate": null,
    "confidence_interval": null,
    "sample_count": 0,
    "status": "not_measured"
  },
  "outcome": "inconclusive",
  "report_artifact_id": null
}
```

**Başarılı rapor üretimi = başarılı deney sonucu değildir.** Yukarıdaki örnekte ölçüm bulunmadığı için değerler null ve sonuç inconclusive'dur. CLI `report generated` yazarken bu durum açıkça görünür olmalıdır.

## 23. Karşılaştırma motoru: önerilen saf fonksiyonlar

Karşılaştırma fonksiyonları veri tabanına veya GUI state'ine gömülmemelidir. Mümkün olduğunca saf, unit test edilebilir bir hesaplama katmanı tercih edilir. Önerilen sözleşmeler:

```python
# ÖRNEK / HEDEF, bugün kaiwen_forge paketi mevcut değil.
from dataclasses import dataclass
from typing import Literal

@dataclass(frozen=True)
class ComparisonPolicy:
    primary_metric: str
    protocol_digest: str
    analysis_kind: Literal["paired", "independent", "descriptive"]
    confidence_level: float = 0.95
    bootstrap_replicates: int = 2000
    sampling_unit: str = "source_group"

@dataclass(frozen=True)
class CompatibilityDecision:
    status: Literal[
        "compatible",
        "compatible_with_caveats",
        "incomparable",
        "insufficient_provenance",
    ]
    reasons: tuple[str, ...]

# Planlanan giriş/çıkış protokolü: veri ve modeller henüz uygulanmış değil.
def check_comparability(baseline: object, candidate: object, policy: ComparisonPolicy) -> CompatibilityDecision:
    raise NotImplementedError
```

`object` burada örnek bir tasarım şablonudur; gerçek implementation Pydantic veya tipli dataclass/domain model kullanmalıdır. Karşılaştırma fonksiyonu kaynağı okumadan hesaplanmış ve doğrulanmış girdileri alır; çalışırken yeni eğitim başlatmaz, Internet'ten veri indirmez ve kullanıcıya görünmeden eval suite değiştirmez. Bootstrap PRNG state'i global state yerine fonksiyon girdisiyle yönetilir.

### 23.1 Önerilen katmanlar

- `experiments/models.py`: tipli domain objeleri.
- `experiments/protocols.py`: protokol dondurma ve revision.
- `experiments/metrics.py`: metric registry ve normalizasyon.
- `experiments/comparability.py`: uyumluluk kontrolü.
- `experiments/statistics.py`: deterministic aggregation ve confidence interval.
- `experiments/ablation.py`: faktör/kol ilişkileri.
- `experiments/reporting.py`: machine-readable ve Markdown raporları.
- `storage/`: mevcut kanonik run/artifact DB erişimi; ikinci DB değil.
- `adapters/speech/`: kaynak metrik/CLI formatlarını normalleştiren ince adapter.

Modül yolları **hedef düzen**dir. Önce çalışan küçük modül seti kurulur; her fikre ayrı soyutlama ve servis oluşturulmaz. Yeni analiz kütüphaneleri test ihtiyacı doğmadan eklenmez.

## 24. CLI hedefleri ve gerçek çalışan komutların ayrımı

Forge için önerilen **gelecekteki komutlar** aşağıdadır. Bunlar **bugün mevcut değildir**, `pip install kaiwen-forge` yapıldığında çalışacağı garanti edilmez:

```bash
# HEDEF ARAYÜZ / IMPLEMENTED DEĞİL
forge experiment create --plan studies/parallel-vs-depth.yaml
forge run submit --experiment example-experiment-001 --recipe recipes/parallel.yaml
forge run list --experiment example-experiment-001
forge run show example-run-001
forge metrics export example-run-001 --format jsonl
forge evaluation run --run example-run-001 --suite turkish-continuation-v1
forge compare --experiment example-experiment-001 --protocol 1.0.0
forge report export example-comparison-001 --format markdown
```

Buna karşılık bugünkü **Speech kaynak dosyasında bulunan gerçek CLI seçenekleri** `scripts/train_continuation.py`, `scripts/train_dialogue.py`, `scripts/evaluate_continuation_audio.py` ve `scripts/benchmark.py` içinde bulunur. Örneğin, gerekli bağımlılıklar kurulu ve Speech repo kökündeyken çalıştırılabilecek mevcut komut:

```bash
# MEVCUT SPEECH CLI; burada çalıştırılmadı.
uv sync --frozen --extra dev
uv run python scripts/train_continuation.py \
  --config configs/smoke.yaml \
  --synthetic \
  --steps 100 \
  --overfit
```

Forge'un ilk adapter'ı bu komutu fixed argv listesiyle yönetip artifact/log kaydına bağlar. `--experiment` veya `--metrics-jsonl` gibi mevcut Speech CLI'da bulunmayan argümanlar sessizce eklenmez. Bunlar için önce Speech tarafında değişiklik ve CLI contract test gerekir.

## 25. Colab, Google Drive ve dışarıdan içe alınan deneyler

Colab'da yürütülen deneyler Forge yerel süreçlerine “biz başlattık” diye kaydedilmez. V1'de iki net durum olabilir:

1. **Managed local run:** Forge kendi host üzerinde worker başlattı, heartbeat/exit kaydına sahip.
2. **Imported external run:** Hosted Colab veya başka ortamda yürütülen bir işin dış manifesti ve artifact'leri doğrulanarak içe alındı.

External import durumunda `origin=external_colab`, `verification_level`, `imported_at_utc`, `original_run_id`, source manifest hash'i ve eksik alanlar saklanır. Uzak işin nasıl başlatıldığına dair doğrulanmamış bilgileri Forge kendisi üretmiş gibi raporlamaz.

Colab ve Drive tarafında:

- CPU runtime üzerinden internet indirme ile GPU training farklı bütçeler/oturumlar olabilir.
- Drive yüksek hacimli saklama için uygundur; yüksek rastgele I/O çalışma klasörü olarak otomatik ideal varsayılmaz.
- Lokal geçici çalışma alanında checkpoint oluştur, dosya yazımı bittiğini doğrula, sonra Drive'a aktar.
- Her shard/artifact için boyut, SHA-256, aktarma durumu ve retry kaydı sakla.
- Colab bağlantısı düştüğünde `unknown_remote_state` gibi açık durum kullan; otomatik başarı veya başarısızlık uydurma.
- Aynı external run iki kere import edilirse content digest ve origin run ID ile idempotent davran.
- İzin belgeleri eksik, bozuk veya şüpheli PyTorch checkpoint dosyaları otomatik yüklenmez.

**Veri güvenliği notu:** `torch.load(weights_only=True)` bazı riskleri azaltır ama her kaynakta tam güvenlik garantisi vermez. Kaynağı doğrulanmamış artifact'ler uygun izolasyon ve dosya boyutu sınırı olmadan açılmamalıdır. Forge model eğitimi kurallarını veya script'leri dış manifestten keyfi komut olarak yürütmez.

## 26. Artifact storage, hash ve lineage

Her run'ın `run.json`, `recipe.resolved.yaml`, `environment.json`, `metrics.jsonl`, `events.jsonl`, `artifacts.manifest.json` ve varsa `checkpoints/`, `evaluations/`, `reports/` bölümleri olabilir. V1 fiziksel dizin yapısı gereğinden fazla zorunlu hâle getirilmez; asıl garanti metadata kaydının doğrulanabilir olmasıdır.

Önerilen mantıksal görünüm:

```text
forge-data/
  experiments/
    <experiment-id>/
      protocol-v1.json
  runs/
    <run-id>/
      run.json
      resolved-recipe.yaml
      attempts/
        <attempt-id>/
          events.jsonl
          metrics.jsonl
          logs/
      artifacts.manifest.json
  evaluations/
    <evaluation-id>/
      evaluation.json
      per-sample.jsonl
  comparisons/
    <comparison-id>/
      comparison.json
      report.md
```

Path içerisindeki `<...>` işaretleri yer tutucudur, gerçek dizin adı değildir. Windows path uzunlukları ve yasak karakterler için gerçek implementation güvenli isimlendirme yapar. Artifact manifestinde `artifact_id`, `kind`, `sha256`, `size_bytes`, `created_at_utc`, `source_run_id`, `origin_uri`, `verification_status`, `retention_policy` ve gerekiyorsa `access_classification` bulunur.

Atomic write kullanılır: geçici dosya → flush/fsync imkan dahilinde → hash → atomik rename → SQLite transaction. Yarım artifact `verified` olarak kaydedilmez. Hash'i değişen bir dosya farklı artifact kimliği alır; eski kaydı sessizce güncellemez. Büyük ses dosyaları ve eğitim veri shard'ları otomatik Git'e eklenmez. Drive'a aktarılan içerik de checksum doğrulanmadan “durable backup” sayılmaz.

## 27. Kaynak ve performans raporlarının adaleti

Model benchmark'ında sürücü, GPU, precision, batch size, audio seconds, input frame count, codec ve warmup sayısı korunmalıdır. Bir cihazın peak VRAM değeriyle başka cihazın toplam sistem RAM'i karşılaştırılmaz. CPU ve GPU wall time farklı kuyruk/gecikme kaynaklarına sahiptir.

### 27.1 TTFA tanımı

**Time to first audible audio (TTFA)**, kullanıcı etkileşimindeki açık bir başlangıç olayından (`end_of_user_turn` veya ilgili protokolde `first_input_audio`) hoparlörde gerçekten duyulabilir ilk örneğe kadar geçen süredir. Hangi başlangıç seçildiyse metrik adında/metadata'sında tanımlanmalıdır. Yalnızca modelin ilk tokenı üretmesi, TTFA değildir. Codec decode, IPC, audio buffering ve playback gecikmesi toplamı etkiler.

### 27.2 RTF ve hız

\[
RTF = \frac{\text{audio generation wall-clock seconds}}{\text{generated audio duration seconds}}
\]

RTF < 1 teorik olarak gerçek zamanlıdan hızlı üretimi gösterir; ama stream oynatma ve ilk chunk gecikmesi hâlâ kötü olabilir. RTF ölçümü GPU synchronization, warmup ve aynı değerlendirme örneğiyle yapılmalıdır. Codec-only RTF ile full pipeline RTF ayrı tutulur.

### 27.3 Peak VRAM ve düşük kaynak hedefi

2 GB sınıfı hedefinin ölçümü, **model+codec+runtime+KV cache+buffers** dahil tam uygulama altında yapılmalıdır. Parametre sayısı × dtype byte hesabı yalnızca ağırlık alt sınırıdır. CUDA allocator reservation ile gerçek device free/used ölçümlerinin farkı belirtilir. Quantization doğruluğu ve CPU offload kaynaklı gecikme regresyonu ayrı raporlanır.

### 27.4 Performans regresyonu kararı

Eşikler donanım profili ve ölçüm gürültüsüne göre config'te tutulur; bu dokümandaki rastgele yüzde sayıları otomatik ürün SLA'sı olmaz. Örnek: “aynı makinede p95 TTFA önceki release'e göre belirlenmiş tolerans üstünde artarsa review gerektirir.” CI ortamında GPU yoksa GPU benchmark `not_run` kaydedilir, “passed” denmez.

## 28. Güvenlik, erişim ve etik sorumluluk

- Artifact indirme/yayınlama için lisans ve source provenance zorunludur.
- Başka kullanıcının özel sesleri/kişisel kayıtları raporda veya otomatik örnek galerisinde sunulmaz.
- GitHub Actions secret'ları, Drive token'ları ve model sağlayıcı API anahtarları experiment snapshot'a yazılmaz.
- Run içindeki URL veya `command` string'i shell'e doğrudan aktarılmaz; allowlisted adapter ve argv kullanılır.
- Protokol ve metric registry değişiklikleri inceleme/PR ile kayıt altına alınır.
- Yalnızca doğrulanmış artifact'ler candidate olarak işaretlenebilir; registry promotion ayrı onay gerektirir.
- JSONL/SQLite'ye gelen açıklama alanları kod veya SQL gibi çalıştırılmaz; prepared statement, path normalization ve output escaping gerekir.
- Downloaded checkpoint içeriği güvenlik politikası olmadan `torch.load` ile otomatik açılmaz.
- Data subject silme veya lisans iptalinde araştırma kanıtlarının korunması ile ses içeriğinin silinmesi dengelenir; yasal zorunluluklar varsa ses artifact'i silinir, mümkünse metadata ve silme kaydı tutulur.

## 29. Test stratejisi: yanlış bilimsel sonucu engelleyen testler

Unit testler yalnızca endpoint'in 200 dönmesini değil **istatistik ve karşılaştırmanın doğruluğunu** kontrol etmelidir.

| Test kodu | Senaryo | Beklenen |
|---|---|---|
| EX-01 | Aynı metric event iki kez geldi | Tek kayıt; idempotent |
| EX-02 | Aynı event ID farklı payload | Conflict; eski event korunur |
| EX-03 | Eksik step aralığı | Gap işareti; lineer doldurma yok |
| EX-04 | NaN/Infinity metrik | Validation error |
| EX-05 | Aynı isim, farklı unit/version | Karşılaştırma reddedilir |
| EX-06 | Farklı eval split hash | Strict comparison reddedilir |
| EX-07 | Codec farklı, transformer comparison | Uyumsuzluk uyarısı veya red |
| EX-08 | Codec reconstruction araştırmasında iki codec | Aynı PCM/metric ile izinli tasarım |
| EX-09 | Validation ile test seti karışmış | Leak flag, rapor invalidated |
| EX-10 | Baseline/candidate örnek ID eşleşmiyor | Paired test reddedilir |
| EX-11 | Eşlenmiş bootstrap sabit seed | Aynı girdiyle aynı CI sonucu |
| EX-12 | Kaynak grupları bağımlı | Group bootstrap birimlerine uyum |
| EX-13 | Üç seed'den birinde OOM | Failure kayıtlı, sahte kalite puanı yok |
| EX-14 | Post-hoc primary metric değişimi | Protocol revision gerekir |
| EX-15 | Rapor yeniden üretimi | Aynı dataset/analysis digest, eşdeğer çıktı |
| EX-16 | Rater'a model etiketi sızıyor | Blind evaluation güvenlik testi başarısız |
| EX-17 | Yanlış sampling unit | Explicit validation warning |
| EX-18 | Checkpoint bozuk | Evaluation çalıştırılmaz |
| EX-19 | Aynı Colab import iki kere | Aynı origin, idempotent |
| EX-20 | Run completed ama eval başarısız | Run durumu korunur, eval failed |
| EX-21 | RTF paydası sıfır | Geçersiz metrik, divide by zero yok |
| EX-22 | Null metric sıfıra çevrilmiş | Schema veya analiz reddi |
| EX-23 | İki aday farklı cihazda benchmark | Strict latency comparison reddi |
| EX-24 | Run kimliği aynı ama recipe digest farklı | Immutable snapshot conflict |
| EX-25 | Evaluator kod commit'i değişti | Yeni eval protocol/evaluation ID |
| EX-26 | 2 GB VRAM yalnız weight hesaplandı | Full-system compliance iddiası reddi |
| EX-27 | Grafik smoothing aç/kapat | Ham veri değişmez |
| EX-28 | SQL migration yarıda kesildi | Transaction rollback, önceki veri kullanılabilir |
| EX-29 | İptal edilmiş run geç completed event alıyor | Run state değişmez |
| EX-30 | Rapor hash'i ve satır kaynağı uyuşmuyor | Rapor doğrulama başarısız |

**Integration fixture:** Speech `configs/smoke.yaml` üzerinden 100-step sentetik continuation run'ı mümkün olduğunca CPU ortamında gerçek adapter ile çalıştırılır. Forge çıktıyı parse eder, checkpoint'i kayıt altına alır ve aynı run'ı iki ayrı attempt olarak yeniden işletebilir. Gerekirse 3–10 adımlık kısa test fixture da kullanılır; bu hiçbir zaman model kalitesi kanıtı değildir.

**Property/metamorphic test:** İki kolun adını swap edince paired delta işareti ters dönmeli; aynı run iki kere eklendiğinde örnek sayısı değişmemeli; metric unit dönüşümü birimce tutarlı olmalı; raw metric smoothing'den etkilenmemeli; bootstrap aynı seed ile deterministic olmalıdır.

**Kabul yöntemi:** CI'da CPU-only hızlı test paketi, isteğe bağlı daha uzun integration testi, gerçek GPU varsa ayrı donanım benchmark'ı. Eksik GPU testleri “skip/not verified” olarak raporlanır. CI badge'i bilimsel olarak bir modelin daha iyi olduğunu göstermez.

## 30. CI/CD, kalite kapıları ve repo uygulama sınırları

Forge Python paketi kurulduktan sonra asgari kalite kapıları:

1. Formatting/lint; Python 3.11+ hedefiyle ruff veya repo standardı.
2. Python typing; seçilen strictness ve istisnalar belgelenir.
3. SQLite migration/recovery testleri.
4. Metric registry ve schema testleri.
5. Stats/paired comparison testleri ve reproducibility fixtures.
6. Speech adapter kontrat testi; gerçek CLI değişikliği algılanır.
7. Artifact digest/atomik kayıt güvenlik testleri.
8. Docs link + JSON/YAML örnek doğrulama.
9. Unit tests: herhangi bir sahte örneğin gerçek measurement gibi raporlanmadığını denetleme.
10. Paketleme/dependency lock ve izin kontrolü.

Merge öncesi PR raporu hangi testlerin gerçekten çalıştığını, hangilerinin atlandığını belirtmelidir. Ağ, GPU, indirme veya kullanıcı credential gerektiren testler koşullu gruplarda olabilir; “testler çalışmadı” durumunu başarı olarak gizlememeliyiz. Repo içindeki `AGENTS.md` bu genel standartları Codex'in günlük çalışma protokolüne çevirir.

## 31. Uygulama aşamaları: EX0–EX10

Aşağıdaki sıra `TRAINING_PIPELINE.md` dosyasındaki F0–F9 ile birlikte kullanılır. **EX** kodları ayrı bir plan sıralamasıdır, Forge run lifecycle'a yeni ikinci state machine eklemez.

### EX0 — Kaynak gerçekliği ve tasarım incelemesi

**İş:** Forge README/AGENTS, Speech Trainer, metrics stdout formatı, checkpoint ve mevcut dataset manifest yapısını denetle.  
**Çıktı:** İzlenebilir kaynak envanteri, bağımlılık/yeni API listesi.  
**Kabul:** Gerçekte bulunmayan CLI veya sınıf “mevcut” diye işaretlenmemiş.

### EX1 — Domain modelleri ve immutable protocol

**İş:** Study/experiment/run ilişkilerini Training Pipeline tablolarını kullanarak tipli birleştir; protocol revision ve checksum doğrulaması.  
**Kabul:** Aynı run snapshot değiştirilemiyor; yeni revizyon önceki planı silmiyor; migration testi yeşil.

### EX2 — Metric registry ve event ingestion

**İş:** Sınırlı anlamlı metrik kümesi, birim doğrulaması, step/attempt idempotence ve missing davranışı.  
**Kabul:** EX-01..EX-05 ve EX-22 testleri geçiyor; ham metrik kaybı raporlanabiliyor.

### EX3 — Speech CPU smoke experiment

**İş:** Gerçek Speech CLI için deney aç, run başlat, metric ve artifact al.  
**Kabul:** Çalıştırılan komut, stdout/exit, checkpoint hash'i ve config digest aynı run üzerinden bulunabiliyor. Fake loss yok.

### EX4 — Baseline/candidate compatibility gate

**İş:** Snapshot ve eval suite doğrulama, strict/exploratory politika.  
**Kabul:** Uyumsuz karşılaştırmalar sessizce “winner” raporu üretmiyor.

### EX5 — Değerlendirme kayıtları ve sample-level evidence

**İş:** Speech evaluator çıktısı ile evaluation manifest ve sample scoring; bozuk kayıtlar ayrı statü.  
**Kabul:** Aynı checkpoint yeni suite ile ayrı evaluation oluşturabiliyor; dataset leakage uyarısı saklanıyor.

### EX6 — Basit karşılaştırma ve CSV/Markdown rapor

**İş:** Predefined metric delta, group aggregation, sample count, caveat ve rapor artifact'i.  
**Kabul:** İki run'dan yalnızca gerçek data kullanarak deterministik rapor oluşturuluyor.

### EX7 — İstatistiksel güven ve dinleme testi import'u

**İş:** Paired/group bootstrap ve kör dinleme sonuç import'u.  
**Kabul:** Randomization ve sampling unit kayıtlı; aynı seed ile aynı çıktı; örnek sayısı ve rater etkisi raporlanıyor.

### EX8 — Ablation ve küçük arama matrisi

**İş:** Faktör/kol/seed planı ve bütçe üst sınırı; otomatik scheduler varsa mevcut worker'ı kullan.  
**Kabul:** Çoklu deney grubu confound uyarılarını koruyor; başarısız run'lar filtreyle kaybolmuyor.

### EX9 — Colab/Drive'dan doğrulanmış import

**İş:** External run manifest doğrulama ve origin provenance.  
**Kabul:** SHA doğrulanmadan artifact trusted olmuyor; local/remote kökenler ayırt ediliyor.

### EX10 — Bitirme projesi karşılaştırma paketi

**İş:** Ön kayıtlı en az bir karşılaştırmanın dataset/protokol/metric/artifact zinciriyle raporlanması.  
**Kabul:** Üçüncü bir kişi, erişim izni verilen veriyle protokolü ve analiz kodunu takip ederek sonucu yeniden hesaplayabiliyor. Sonuç başarısız veya inconclusive da olabilir; bu dokümantasyon başarısızlığı değildir.

**Geliştirme kuralı:** EX1–EX3 bitmeden fancy dashboard, distributed sweep veya üçüncü parti tracking server kurma. EX4–EX6 olmadan “hangi model daha iyi?” kararını otomatik verme. EX7 için gerçek değerlendirici/veri bulunmadan örnek sonuç uydurma.

## 32. Kabul kriterleri ve Definition of Done

Bir EXPERIMENTS V1 uygulaması ancak aşağıdaki koşullarla hazır sayılır:

- [ ] Study ve experiment için kimlik/protokol değişmezliği uygulanıyor.
- [ ] Gerçek Speech smoke run'ı Forge üzerinden yaratılıp bulunabiliyor.
- [ ] Source commit, resolved recipe, seed, dataset ve codec kimliği kayıtlı.
- [ ] Metric event'ler tipli, idempotent ve birim doğrulamalı.
- [ ] Ham metriklerle sunum grafikleri birbirinden ayrılmış.
- [ ] Eval kayıtları checkpoint ve suite digest'ine bağlı.
- [ ] Aynı test örneği olmayan iki run paired comparison olarak raporlanmıyor.
- [ ] En az bir gerçek karşılaştırma raporu, uyumluluk gerekçesi ve örnek sayısıyla yeniden üretiliyor.
- [ ] Eksik metrikler `null`/`not_measured`; başarı puanı uydurulmuyor.
- [ ] Bozuk/iptal/timeout run'lar ve negative result geçmişi korunuyor.
- [ ] Bootstrap/istatistik kullanılıyorsa PRNG seed, sampling unit ve varsayımlar kayıtlı.
- [ ] Test seti leakage şüphesi ilgili report/evaluation'ı invalidated yapabiliyor.
- [ ] Secrets, özel sesler ve tokenlar rapor/artifact loglarına sızmıyor.
- [ ] CPU testleri ve kontrat testleri CI'da çalışıyor; GPU yokluğu açık.
- [ ] Documentation examples implemented/planned etiketiyle tutarlı.
- [ ] Bitirme raporu için en az bir provenance zinciri uçtan uca takip edilebiliyor.

## 33. Başarısızlık, açık kararlar ve ileride değerlendirilecek teknolojiler

Bilinmeyen bir araç veya framework yüzünden çalışmayı durdurmamak için her yeni teknoloji şu soruyla değerlendirilir: **Şu an ölçülmüş, çözülmemiş hangi problemi çözüyor?** Örnekler:

- **MLflow:** Deney verilerinin başka ekip/cihazlar arasında paylaşımı ve entegre model registry ihtiyacı büyürse değerlendir. V1'de SQLite + artifact store yeterliyse sunucu ekleme.
- **Weights & Biases / hosted trackers:** Gelişmiş görselleştirme sağlayabilir; gizlilik, maliyet, veri çıkışı ve servis bağımlılığı karşılaştırılmadan zorunlu yapma.
- **Optuna:** Çok sayıda hiperparametre araması gerçekten gerekli olursa değerlendir; birkaç deney kolu için yeni veritabanı/worker katmanı kurma.
- **DVC / Git LFS / object store:** Büyük veri sürümlerinin ve ekip paylaşımının bugünkü manifest/hash yaklaşımını zorladığı noktada kullan; 5 TB Drive tek başına veri versiyonlama standardı değildir.
- **OpenTelemetry:** Multi-process/remote execution büyüdüğünde ortak trace için değerlendir; her basit metric için yeni tracing altyapısı zorunlu değil.
- **DuckDB/Parquet:** Milyonlarca metric veya analiz sorgusu SQLite sınırını somut ölçümlerle aştığında ekle.
- **Ray/Kubernetes:** Dağıtık scheduling ihtiyacı doğrulanmadan başlama; V1 tek host + hosted Colab import yeterli.

Seçilecek teknoloji için kısa ADR: mevcut problem ve kanıtı, alternatifler, beklenen fayda, operasyon maliyeti, güvenlik/mahremiyet, bağımlılık etkisi, küçük benchmark, rollback. Yeni araç “profesyonel görünmek” için eklenmez.

## 34. Codex için pratik çalışma protokolü

Codex bu dosyayı okuduğunda aşağıdaki adımları izlemelidir:

1. `kaiwen-forge` reposunun gerçek `HEAD` ve `git status` bilgisini incele; çalışma alanındaki taslakları uygulanmış kod sanma.
2. Root `AGENTS.md`, Forge `AGENTS.md`, Forge README ve `TRAINING_PIPELINE.md` ile kararları karşılaştır.
3. Speech repo Python package/CLI güncel durumunu ve testlerini keşfet; başka repo dosyalarını izinsiz değiştirme.
4. Küçük bir iş dilimi seç (`EX1`, `EX2` veya `EX3`); ayrı tasarım ve uygulama PR'larını gerektiğinde ayır.
5. Önce failing test yaz; event idempotence, immutable config veya provenance gibi kavramsal invariantları testle.
6. Metrik yorumlarını kaynak script ve tanım sürümüyle eşleştir; keyfi parser ile yanlış grafik üretme.
7. Kalite ve istatistik hesaplarını fixture/bağımsız küçük referans hesaplarıyla karşılaştır.
8. JSON/YAML/SQLite örneklerini gerçek şemaya dönüştürürken geriye dönük migration belgelerini koru.
9. Farklı repo değişikliklerinde ayrı PR ve entegrasyon test matrisi kullan.
10. Gerçek çalışma yapılmadıysa “uygulandı/test edildi” yazma. Sadece tasarım için hazırlanan dosyaları açıkça etiketle.
11. İş bittiğinde değişen dosyaları, çalıştırılan testleri, başarısız/atlanan testleri, kaynak/donanım etkisini ve rollback yolunu raporla.
12. Yeni framework önerisini Discovery Gate'ten geçir, kabul edilmezse backlog'a not et ve ana işi sürdür.

## 35. Kaynak haritası, bağlantılar ve son sınır

Bu dosya yalnızca deney tasarımını, metriklerin anlamını ve araştırma karşılaştırma kayıtlarını sahiplenir. Diğer sözleşmeler ilgili kanonik belgelerde tutulmalıdır:

- [Forge README](../README.md): ürün kapsamı, mevcut durum ve önerilen modüller.
- [Forge AGENTS](../AGENTS.md): Codex'in günlük çalışma ve onay kuralları.
- [Training Pipeline](TRAINING_PIPELINE.md): worker, scheduler, checkpoint, retry, Colab/Drive ve run durum makinesi.
- **Planlanan `EVALUATION.md`:** evaluator suite uygulamaları, ölçekler, örnek seçme ve benchmark motorları. Bu belge henüz yazılmadığından aşağıda çalışan dosya bağlantısı verilmemiştir.
- **Planlanan `MODEL_REGISTRY.md`:** candidate inceleme, onay, release ve rollback. Bu belge henüz yazılmadığından çalışan dosya bağlantısı verilmemiştir.
- [Workspace Integration](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/INTEGRATION.md): repo arası dataset ve artifact sözleşmeleri.
- [Engineering Standards](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ENGINEERING_STANDARDS.md): ortak mühendislik kalite kapıları.
- [Workspace Roadmap](https://github.com/ImtheKaiwen/kaiwen-ai/blob/a2476114c4da048090aeb23a2b03a2ec930b113a/docs/ROADMAP.md): kilometre taşları ve bağımlılıklar.
- [Speech Model](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/SPEECH_MODEL.md): training label/loss ve model mimarisinin gerçek sahibi.
- [Speech Codec](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/CODEC.md): bitrate, RVQ ve reconstruction araştırması.
- [Speech Performance](https://github.com/ImtheKaiwen/kaiwen-speech/blob/2dd8ae4ff4970fec13d761afbb5587b887957547/docs/PERFORMANCE.md): gerçek zamanlı performans metriklerinin teknik ölçümü.

**Son ilke:** Forge araştırma kararlarını süslemek için değil, sonuçları dürüst biçimde incelemek için vardır. İyi görünmeyen, anlamlı fark üretmeyen veya beklenenden kötü sonuçlanan deneyler de aynı ciddiyetle korunur. Model araştırmasında güvenilir bir “henüz bilmiyoruz” cevabı, kaynağı belirsiz bir “başardık” cümlesinden daha değerlidir.
