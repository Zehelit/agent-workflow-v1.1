<a name="lifecycle-startup-and-recovery-implementation-plan"></a>

# Başlangıç ve toparlanma uygulama planı

> 10 Kasım 2026 tarihli plan. Aşağıdaki işler, yetki ve özel kanıtlar geçmiş çalışmaya aittir. Güncel davranış için [işletim sözleşmesi](../../../skills/agent-workflow-v1-1/references/operations.md).

<a name="goal-and-constraints"></a>

## Hedef ve kısıtlar

Plan ilk başlatma hatasının güvenli tekrarı, zamanında ölçüm yenileme, doğrulanmış toparlanma ve normal kapanışı kapsıyordu. Pin’li candidate, tipli çekirdek ve modelsiz izleyici korunacaktı. Yaşam döngüsü aracı hazırlık, gerçek host gözlemi, çalışma ve kapanışı ayıracaktı. Süreli test açıkça seçilecek bir mod olarak kalacaktı. [Tarihsel şartname](../specs/2026-11-10-agwf-lifecycle-reliability.md).

Önceki plan v1.1 kaynak commit ve push’una izin veriyordu. GitHub Release ve tag kapsam dışındaydı. Python 3.11+, standart kütüphane, unittest, mevcut Codex hedef ve kullanım araçları ile Windows dosya/süreç davranışı korunacaktı. Kullanıcı çalışması, eski kanıt ve bağlar korunacaktı. Paket, hedef, ölçüm, kullanıcı sözü ve süreç kimliği gerçekten gözlenecekti. Sentetik veri yalnız izole testlerde kullanılacaktı. Ölçüm eskimesi, kota, erken rezerv ve kullanıcı duruşu ayrı yetki gerektiriyordu. Normal akış yapay SAVE üretmeyecekti. Başka oturumların işi ve model ayarı kapsam dışındaydı.

<a name="review-focus"></a>

## İnceleme odağı

1. İlk başlatma gerçekten eskime yüzünden SAVE verdiyse, yardımcı etkinleşmeden önce uygun tekrar mümkün olmalı.
2. Kota veya kullanıcı duruşu, hedef değişimi ve belirsiz süreç teknik kurtarmadan geçmemeli.
3. Yarım veya eşzamanlı başlangıç, desteklenen kanıt olmadan başarı sayılmamalı.
4. Yeni dosya yazımı eski gözlemi tazelemez. Taze örneklem varken lease bitmiş olabilir.
5. Geç yerleşik olay zamanlama testini geçemez. Windows paylaşım hatası süreç çıkışı değildir.

<a name="files-and-responsibilities"></a>

## Dosya ve sorumluluklar

| Yol | Planlanan sorumluluk |
| --- | --- |
| `skills/agent-workflow-v1-1/scripts/candidate.py` | Başlangıç denemesi, tekrar kararı, süreç sahipliği ve yaşam döngüsü |
| `skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py` | SAVE, epoch ve receipt sözleşmesine yönelik dar değişiklikler |
| `skills/agent-workflow-v1-1/scripts/workflow.py` | Mevcut active, halted, complete ve kullanıcı duruşu kuralları |
| `tooling/run-lifecycle-check.py` | Ana ajanın yerleşik kanıtla çalıştırdığı hazırlıklı araç |
| `tooling/tests/test_lifecycle_check.py` | Hazırlık, geç olay ve sonuç sınıflandırması |
| `skills/agent-workflow-v1-1/tests/test_candidate_coexistence.py` | Kaynak ve kurulu paket girişinden gerçek sahipli worker entegrasyonu |
| `test_native_usage_source.py`, `test_goal_binding.py`, `test_watcher_engine.py`, `test_monitor.py` ve `test_watcher_atomic_write.py` | Ölçüm, hedef, yetki, zamanlama ve Windows regresyonları |
| `SKILL.md`, `references/operations.md`, `references/native-usage.md` ve `references/coexistence.md` | Yenileme, tekrar, kurulum ve ayrı host kabulü yönergeleri |

<a name="task-1-retry-a-failed-first-activation"></a>

## Görev 1. Başarısız ilk etkinleştirmeyi tekrarla

Arayüzler `activation_identity(command: str, payload: dict) -> dict`, `record_activation_attempt(runtime, record: dict) -> None` ve `startup_retry_options(runtime, engine, command: str, payload: dict) -> tuple[str, ...]`. Run başına `activation-attempt.json` kaynak kimliği, `schema_version`, `command`, `pre_state_sha256`, `phase` ve gözlenen süreç/hata ayrıntısını kaydeder. `prepared`, `failed` ve `committed` ayrı aşamalardır. Yarım `prepared` uzlaştırma ister.

- İlk eski ölçüm reddini göster, sonra aynı hedefte taze `bind-active-goal` ve `activate` tekrarı. İlk SAVE hash’i ve yetki korunmalı. Sağlıklı worker ve active doğrulansın.
- Başlangıç niyetini `preflight_activation` sonrasında yaz. Komut, run, hedef, sohbet, kapsam, amaç ve yetki kimliğini bağla. Yeni gözlem tek başına yeni kapsam oluşturmaz.
- Candidate ve çekirdek arasına sahipli tek yazıcı kilidi koy. Sahip bilinmiyorsa reddet. Yerleşik arayüzle ortak atomik işlem yoktur.
- Teknik epoch yalnız doğrulanmış aynı run `usage_receipt_expired`, yeni etkin hedef ve gerçek receipt ile. Retry işareti yetki değildir. Başarıyı yalnız izlemeyi ve çekirdek kaydını tamamlayınca yaz.
- Kota ve rezerv, başka hata, yeni SAVE, kullanıcı duruşu, değişmiş/pasif hedef, canlı/belirsiz süreç, değiştirilmiş veya yarım günlük, yanlış paket ve eşzamanlı başlangıcı reddet. Bozuk ölçümden sonra geçerli tekrarı sına. Geri alma yalnız doğrulanmış önceki kontrolü yükler, yeni SAVE’i silmez.
- Coexistence ve native receipt testlerinde active, canlı kimlik ve tazelik, eski SAVE ve gerçek süreç temizliği gözle.

<a name="task-2-prepare-before-measuring-and-refresh-early"></a>

## Görev 2. Önce hazırlan, son tarihten önce yenile

Araç `prepare`, `run --input PATH`, `observe --input PATH` ve `close` sunacaktı. Hazırlık paket, pin, rol ve komutları çalıştırmadan denetler. `refresh_by(at: float, interval: float) -> float`, `at + interval - min(15, interval / 4)` önerir. 60 saniyelik aralıkta 45 saniye. Gerçek son tarih aynı kalır.

- Prepare ve help canlı girdi istememeli. Eski veya eksik ölçüm active durum üretmemeli. Replay zamanı yenilememeli.
- Hazırlıktan sonra gerçek hedef ve ölçümü al. Yalnız iki pencere ve reset dahil gerekli alanları koru. Hesap, kimlik ve kredi alanı saklama.
- `sample_at`, `next_check_at`, `check_interval_seconds` ve `refresh_by` farklıdır. Dosya yazımı ve kalp atışı taze okuma kanıtı değildir.
- `status.at == usage.observedAt`, canlı kimlik, epoch, `sampleFresh` ve `leaseValid` aranır. Ingest sonrası lease doğrula. Kesilemeyen çağrıda tazelik garantisi verme.
- Normal akışta yapay zamanlama/hata kapalı. Gerekince snapshot doğrula. Doğal complete ve süreç çıkışı ile yapay SAVE olmadığını sına.
- Lifecycle ve receipt testlerini farklı gözlem ve yayın zamanlarıyla çalıştır.

<a name="task-3-diagnose-timing-without-confusing-late-events-with-success"></a>

## Görev 3. Geç olayın zamanlamayı geçmesini engelle

- `timed-test` yalnız açık tanılama isteğinde, sağlıklı hazırlanmış süreç ve gerçek olayla çalışır. `--event-input PATH --delay-seconds 10 --tolerance-seconds 1` kullanılır. Son tarih `event_at + 10`, bekleme monotonic.
- 45 saniye geç olay `INVALID_TIMING` verir. 98,74 ve 45,05 saniyelik önceki sonuçlar başarısız kalır.
- Mevcut resmî goal olayı kullanılır. Desteklenen olay kaynağı yoksa geç veya eksik olarak sınıflandır. Özel taşıma veya kimlik dosyasıyla UI ölçme.
- `native_event_at`, `observed_at`, `worker_healthy_at`, `save_requested_at`, `save_returned_at`, `snapshot_verified_at`, `helper_halted_at` ve `native_paused_at` ayrı kaydedilir. 9–11 saniye SAVE istemi içindir ve worker önceden sağlıklı olmalı. Yerleşik pause ayrı kanıt ister.
- İki canlı tur aynı hedef ve run’da. Önce active, doğrulanmış checkpoint ve halted, sonra gerçek yetkili devam. İlk bağlama devam sayılmaz.
- Erken/geç/tekrarlı/değişmiş sentetik olayları, eksik pause, kullanıcı duruşu ve iki turu sınayın. Canlı süreli test ayrı koşuldur.

<a name="task-4-verify-recovery-and-regressions"></a>

## Görev 4. Kurtarma ve regresyon

- Temiz ilk başlangıç, ilk eskime ve taze tekrar, etkin run’da expiry ve stop/snapshot/halt/resume, bozuk sonra doğru tekrar, pasif→etkin Continue ve doğal kapanış kaynak ve pakette sınansın.
- Devam öncesi `halted` ve doğrulanmış checkpoint. Sonra aynı hedef/run, yeni epoch, korunmuş SAVE hash’i, yetki ve gerçek sağlık.
- Kota/rezerv, yeni kota SAVE, kullanıcı duruşu, farklı hedef, eskimiş/replay ölçüm, bozuk checkpoint, canlı/belirsiz alt ajan reddedilsin. Yerleşik `blocked` kendi koşullarına bağlı.
- Windows testlerinde gerekli sahiplik dosyalarını kopyala ve bilinen kimliği terminal sağlıkla kontrol et. Geçici paylaşım hatası çıkış değildir. Kalıcı izin hatası sınırlı sürede açığa çıkmalı.
- Odak testlerinden sonra iki paket testi. Sayıyı keşiften al. Canlı normal akış gerçek hedef ve hazır araç ister. Ayrı yetkisi olmayan expiry ve zaman testi `not_tested` kalabilir.

<a name="task-5-document-the-contract-and-verify-distribution"></a>

## Görev 5. Sözleşme ve dağıtım

- İlk tekrar, durmuş resume ve kota veya kullanıcı duruşu yollarını ayrı yaz. Yenileme ve lease sırasını ve geç olayın anlamını açıkla.
- Paket ve kullanıcı çalışmasını envanterle. Yerel değişiklikleri ve çalışan kurulumu koru. Eski pin’den etkinlik çıkarma. Aktif run’ı taşıma.
- İki aynı build’in manifest/ZIP hash’lerini karşılaştır. Değişen payload’ı sına. Kurulu girişi geçici pasif run, pin ve Windows erişimiyle doğrula. Pasif kapsamı doğru raporla.
- `startup_retry`, `receipt_refresh`, `halted_resume`, `native_continue`, `natural_complete`, `timed_test`, `worker_exit` ve `package_match` ayrı kanıta bağlansın. Önceki olumsuz kayıtları koru.
- O zamanki kaynak yayını öncesi yetki, eski pakete geri dönüş ve yeni SAVE korumasını bağımsız incelet. Özel rapor ve runtime kaydını Git dışında tut.

<a name="task-6-align-the-existing-v11-identity-and-presentation"></a>

## Görev 6. Mevcut v1.1 kimliği ve sunumu

- Görünen ad, teknik çağrı, kurulum ve yeni çalışma alanını birlikte taşı. Eski kimlik ve pin’leri koru.
- `CHANGELOG.md` v1.0 ve v1.1 geçmişini belgelemeli. README amaç, sınırlar, mimari, çalışma akışı ve orkestrasyonu anlatmalı.
- İnsan ve ajan kurulumu temiz checkout, harici pin, doğrulama ve açık çağrıyı içermeli. Kurulum hedef veya model ayarını değiştirmez.
- O zamanki yetkili yayında About ve Topics davranışa göre seçilsin. Erişim ve dal koruması korunsun. README komutları ve diyagramlar doğrulansın.
- Yalnız yetkili dosyaları yayımla, uzak commit/tree, sürüm ve belgelere tekrar bak. Release ve tag’in oluşturulmadığını doğrula.

<a name="acceptance-boundary"></a>

## Kabul kapsamı

Özgün sıra 1–5 çekirdek görevleri ve sonrasında yayın işiydi. İlk tekrar ve gerçekten halted durumundan devam matrisi, regresyon, paket ve doğru kapsamlı gözlemler istendi. Yerleşik arayüz iddiası gerçek host kanıtı gerektirir. Canlı on saniye sonucu gerçek olay ve ayrıca ölçülen zaman kanıtına dayanır.
