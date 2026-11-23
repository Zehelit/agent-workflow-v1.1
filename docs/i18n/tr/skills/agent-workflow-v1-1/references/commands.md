<a name="command-contract-a02"></a>

# Komut sözleşmesi (A02)

Standart kütüphane yardımcısı `Workflow.execute` ve `Workflow.apply` üzerinden geçen komutları doğrular. Yerleşik aracı engellemez, araç makbuzunun veya kullanıcı mesajının kaynağını doğrulamaz, süreç durdurmaz veya fiilen kullanılan modeli gözlemez. Bunları ana ajan yapar ve kanıtı sağlar. A26 yetkisi, checkpoint, sahiplik ve kapasite kontrolleri geçerlidir.

Durumda tamsayı `schema_version: 2` kullanılır. Sürümsüz, eski, gelecekteki şema ve bozuk durum yazımdan önce reddedilir. Eski durum taşıma veya içe aktarma aracı yoktur. Güncel tipli yetkiyle yeni AGWF run’ı açın. Snapshot ve restore desteklenir. Eski kanıt yeni iş yetkisi vermez. Hata enjeksiyonu yalnız geçici test kopyalarında yapılır.

<a name="json-and-error-policy"></a>

## JSON ve hata sözleşmesi

Durum, `--input`, stdin, checkpoint metadatası ve installer sahiplik JSON’u aynı katı ayrıştırıcıyı kullanır. Aynı değer verilse de iç içe yinelenen anahtarlar reddedilir. `NaN`, `Infinity`, `-Infinity` ve `1e999` gibi sonsuza taşan sayılar geçersizdir. Girdi UTF-8 ve isteğe bağlı BOM kabul eder. Komut kökü JSON nesnesi olmalıdır. Boş stdin `{}` verir. Yalnız komut boş nesneye izin veriyorsa geçerlidir. `write_json` dizin veya geçici dosya oluşturmadan `allow_nan=False` ile doğrular ve serileştirir.

Girdi anahtarları metindir. Liste liste olarak kalır, tamsayı ve float boolean değildir. Bayrak tam Python `bool`, yani JSON `true`/`false` olmalıdır. Metin, 0/1, null, liste ve nesne bayrak olmaz. Zorunlu bayrak eksikse işlem reddedilir. İsteğe bağlı bayrak yoksa false, null ise geçersizdir. Sayılar sonlu olmalıdır. Adet tamsayı ≥1. Zaman [0, sys.float_info.max] arasında tamsayı/float, yüzde [0,100] aralığındadır. Zaman üst sınırı saat hesabının taşmasını engeller. Komut tazelik kuralları ayrıca geçerlidir. Yeni bir azami kapasite eklenmez. Gönderim, ana ajanı içeren slot hesabını ve mevcut alt ajan sınırını kullanır.

Girdi, dizin veya kilit oluşturmadan ve yürütmeden önce doğrulanır. `apply` özel JSON kopyasında çalışır ve denetimler geçince bu kopyayı geçirir. Aşağıdaki kalıcı INPUT_INVALIDATED ve contract_invalid dışında ret eski durumu ve SHA-256’yı korur. İlk reddedilen komut durum oluşturmaz. Başarısız snapshot, etkinleşmemiş sahipli checkpoint dizini bırakabilir. Geçerli biçimde gelen fakat ulaşılamayan kullanım ayrı bir durum geçişidir.

Yardımcı hatada 1 ile çıkar ve stderr’e şu nesneyi verir.

```json
{"error":"exact boolean required","code":"FIELD_TYPE","field":"$.accepted"}
```

| Kod | Anlamı | Alan |
| --- | --- | --- |
| `COMMAND_UNKNOWN` | Bilinmeyen çekirdek komutu | `$.command` |
| `FIELD_REQUIRED` | Gerekli yük alanı veya yardımcı seçenek eksik | Tam alan |
| `FIELD_UNKNOWN` | Bilinmeyen alan veya komutta desteklenmeyen CLI seçeneği | Tam alan |
| `FIELD_TYPE` | Yanlış girdi türü | Tam alan veya liste sırası |
| `FIELD_VALUE` | Geçersiz kimlik, zorunlu boş girdi, sayı aralığı veya envanter durumu | Tam alan veya liste sırası |
| `JSON_DUPLICATE_KEY` | Ayrıştırılan JSON’da yinelenen üye | İç içe alan dahil tam üye |
| `JSON_NONFINITE` | Ham metadata dahil sonlu olmayan sayı veya token | Tam alan veya liste sırası |
| `JSON_SYNTAX` | JSON sözdizimi veya kodlama geçersiz | `$` |
| `JSON_TYPE` | JSON dışı çalışma zamanı değeri veya metin olmayan anahtar | Tam değer veya nesne |
| `COMMAND_REFERENCE` | Görev ya da durum alanı yok | `$.id`, açıklamada eksik anahtar |
| `COMMAND_CONSTRAINT` | Yetki, durum, sahiplik, inceleme veya tazelik denetimi | Varsa ilgili alan. Genel durum için `$.command` |
| `STATE_VERSION` / `STATE_TYPE` | Desteklenmeyen sürüm ya da nesne olmayan durum kökü | `$.schema_version` / `$` |
| `IO_ERROR` / `INVALID_VALUE` | Dosya sistemi hatası veya başka geçersiz değer | `$` |

Basit anahtarlar nokta, diğerleri köşeli parantez, listeler `[index]` kullanır. Argparse’ın bilinmeyen seçenek, komut veya eksik seçenek değeri gibi hataları kendi stderr’iyle 2 döndürür. Tanınan ama o komutta geçersiz seçenek `FIELD_UNKNOWN` olur. Denetim `assert` kullanmaz, `python -O` altında çalışır.

<a name="payload-types"></a>

## Girdi türleri

`text` doğrulamada kenar boşlukları alındıktan sonra dolu metindir. Özgün değer saklanır. `string` boş olabilir. `task_id`, `[a-z][a-z0-9_]{0,63}` desenine uyar. `strings` boş olmayan metinler listesidir, liste boş kalabilir. `criteria` en az bir öğe ister. `task_ids` görev kimlikleridir. `timestamp`, `count` ve `percent` yukarıdaki sayı kurallarına uyar. Yalnız `?` türü açık null kabul eder.

Aşağıdaki her komut ve yapılandırılmış iç nesne, gerekli ve isteğe bağlı alan listesi dışındakileri reddeder. İki istisna ham sağlayıcı çıktısını korur. Makbuz içeriği ile ham `rateLimits` ve pencerelerin ölçüm dışı metadatası opak JSON’dur. Bunlarda da yinelenen anahtar, sonlu olmayan sayı veya JSON dışı değer reddedilir. Bilinmeyen metadata bayrak veya geçerli ölçüm kanıtı sayılmaz.

| İç tür | Zorunlu alanlar | İsteğe bağlı alanlar ve koşullar |
| --- | --- | --- |
| `authorization` | `event_id`, `exact_quote`, `kind`, `source_kind`, `source_id`, `goal_id`, `thread_id`, `run_id` metin. `scope` kapsam. `observed_at`, `source_created_at` zaman. `original_turn_id`, `observed_turn_id` metin. | Ek alan yoktur. A26 olay türü, doğrudan kaynak, kapsam eşitliği, tekrar ve yaş kontrollerini uygular. |
| `scope` | `kind`, `id`, `objective` metin | `kind` A26 altında objective veya report. |
| `source` | `source_kind`, `source_id`, `exact_quote`, `original_turn_id` metin. `source_created_at` zaman. | Yetki kaynağının izdüşümüyle eşit olmalı. |
| `native_binding` | `binding_id`, `goal_id`, `thread_id`, `run_id`, `objective` metin. `scope` kapsam. `goal_created_at`, `observed_at` zaman. | Yalnız ilk N02 bağı. Kapsam objective ve tam amaçla eşit. Hedef oluşturma gözlemden önce. İnsan kaynak alanı yok. |
| `app_goal` | `goal_id`, `thread_id`, `status`, `source_kind`, `source_id` metin. `observed_at` zaman. | `objective` metin ve `goal_created_at` zaman isteğe bağlı. Yerleşik bağda her yenilemede zorunlu ve eşit. A26 kaynak get_goal, aynı kimlik, ≤60 yaş ve uygun etkin durum ister. |
| `inventory` | Nesne listesi. `id` ve `status` metin. | Ek alan yok. Kimlikler benzersiz. Durum running/interrupting/unknown/idle/stopped/done, gözlem yaşı 0–60. |
| `models` | Dolu model adlarını `strings` muhakeme düzeyi listesine eşleyen nesne. | Boş nesne ve liste olabilir. Prepare için tam model ve düzey gözlenmiş olmalı. |
| `arguments` | Yapısal zorunlu alan yok. | Yalnız `task_name` görev kimliği, `fork_turns` metin, `message` string, `model` ve `reasoning_effort` metin. Kayıt hazır argümanla birebir eşleşme ister. Eksik alan gönderim yapamaz. |
| `receipt` | Dolu nesne, liste veya metin. | Gerçek araçtan gelen opak JSON. Kaynak kimliğini kanıtlamaz. |
| `rate_limits?` | Nesne veya null. | İsteğe bağlı primary/secondary pencere. Ek ham metadata opak. |
| `window?` | Nesne veya null. | `usedPercent` isteğe bağlı yüzde, `windowDurationMins` adet, `resetsAt` zaman. Diğer metadata opak. |

<a name="engine-commands"></a>

## Çekirdek komutları

Gerekli ve isteğe bağlı listeler tüm alanları gösterir. İsteğe bağlı alan belirtilmiyorsa yalnız gerekli alanları verin. Boş komutlar `{}` alır.

| Komut | Gerekli girdi alanları | İsteğe bağlı alanlar ve varsayılanlar |
| --- | --- | --- |
| `activate` | `objective` metin, `authorization` yetki, `interpretation` string, `source_message` kaynak, `app_goal_observation` app_goal, `models` model listesi, `slot_limit` adet, `inventory` envanter, `observed_at` zaman | `session_limit` isteğe bağlı adet. Eksik veya null olduğunda ek açık oturum sınırı yoktur. |
| `bind-active-goal` | `objective` metin, `native_binding` yerleşik bağ, `interpretation` ve `procedural_check` metin, `app_goal_observation` app_goal, `models` model listesi, `slot_limit` adet, `inventory` envanter, `observed_at` zaman | `session_limit` isteğe bağlı adet. Yalnız görev ve geçmiş alanı olmayan, başlangıçtaki etkin olmayan schema2 kabul edilir. |
| `observe-authorization` | `authorization` yetki, `interpretation` string, `source_message` kaynak | `app_goal_observation` app_goal kabul edilebilir. Kaydedilen hedef gözlemi yenilenmez. |
| `app-goal` | `app_goal_observation` app_goal | Yok. |
| `usage` | Yok. | `observed_at` zaman ve `rateLimits` isteğe bağlı ölçüm nesnesi. Eksik ölçüm kullanılamıyor demektir. |
| `usage-due` | Yok. | `substantial_work` boolean, varsayılan false. |
| `inventory` | `inventory` envanter ve `observed_at` zaman | Yok. |
| `task` | `id` görev kimliği, `role` metin, `objective` metin, `criteria` ölçütler | `file_inputs` dosya girdileri [], `context_refs` metinler [], `review_targets` inceleme hedefleri [], `revision` metin ve yoksa üretilen UUID, `writes` metinler [], `depends_on` görev kimlikleri [], `independent_review` boolean false. |
| `prepare` | `id` görev kimliği | `effort` yoksa rol varsayılanı, `current_context` N06 context_pin girdisi. |
| `record` | `id` görev kimliği, `submitted_arguments` argümanlar, `session_id` metin, `receipt` makbuz | `effective_model` ve `effective_effort` isteğe bağlı metin. Eksik veya null doğrulanmamış demektir. |
| `dispatch_failed` | `id` görev kimliği ve `evidence` metin | `certain_not_started` boolean false. Eksik veya false çağrı sonucu belirsizdir. |
| `reconcile` | `id` görev kimliği, `evidence` metin, `no_owned_work_running` boolean | true ve etkin alt ajan içermeyen taze envanter gerekir. |
| `finish` | `id` görev kimliği, `accepted` boolean, `evidence` metin | `review_task` görev kimliği. Bağımsız inceleme isteyen görevi kabul etmek için gerekir. |
| `cancel` | `id` görev kimliği ve `reason` metin | Yalnız henüz gönderilmemiş hazır görev. |
| `stop` | Yok. | Yok. |
| `snapshot` | `writers_quiescent` boolean ve `next_action` metin | `files` metinler []. Yazıcı bayrağı true olmalı. |
| `restore` | `destination` metin | Proje dışında yeni hedef gerekli. |
| `halt` | `inventory` envanter, `observed_at` zaman, `owned_processes_stopped` boolean, `stop_evidence` metin | `degraded` boolean false. True ise `reason` ve `next_action` metin zorunludur. Süreç bayrağı true olmalı. |
| `resume` | `authorization` yetki, `interpretation` string, `source_message` kaynak, `app_goal_observation` app_goal | Ek alan yok. Eşleşen halt sonrası devam, taze kullanım ve envanter, güncel doğrulanmış checkpoint gerekli. |
| `compact` | Yok. | Terminal görevler kabul veya ret durumunda, etkin alt ajan yok, checkpoint güncel ve doğrulanmış olmalı. |
| `complete` | Yok. | Compact kontrollerine ek olarak etkin run gerekir. |
| `status` | Yok. | Etkinleştirilmiş bir durum gerekir. |

Dosya atamaları mevcut Windows takma ad, sınır ve symlink denetimleriyle tam proje göreli dosyalardır. Rol ve muhakeme mevcut katalog ve destek denetimine tabidir. Boş yazma listesi salt okunur doğrulayıcı ve koordinatörü mümkün kılar. Bilinmeyen görev reddedilir ve kendiliğinden görev açılmaz. `accepted: false` yazmayı bitirmiş işi reddeder. `accepted: true` ancak ilgili inceleme kontrollerinden sonra kabul eder.

<a name="raw-usage-lifecycle"></a>

## Ham kullanım yaşam döngüsü

N06 `context_pin` tam `{path, sha256, manifest_path, manifest_sha256}` nesnesidir. Proje göreli yollar dolu, SHA-256 küçük harfli olmalıdır. Manifest mevcut değişmez görev girdisi olmalıdır. `prepare` taze version1 özeti güncel durum ve ayrı pin’li manifeste göre doğrular. Özetli hazırlanmış mesaj bu protokole bağlanır. `dispatch-start` `current_context` alabilir ve hazırlama sonrası durum ile özgün manifeste göre yeniden üretilmiş özet ister. Yerleşik kontroller geçerlidir. Özet üretimi, engeller, üretici teslim denetimi ve sınırlar için [bağlama](context.md) bakın.

N07 `status` eski görev durum tablosunu korur ve `task_observations: {contract_version: 1, tasks: ...}` ekler. Yerleşik yürütme ile çıktı kabulü ayrıdır. Done, finish kontrolleri geçene dek kabul bekler. Bağımsız finish mevcut isteğe bağlı `review` bağını kaydeder ve iki farklı, boş olmayan yerleşik oturum kimliği ister. Atanan ve kaydedilen inceleme hash’leri eşleşmelidir. Schema2 değişmez. İnceleme kendi pin’li üreticisinin kabulüne doğrudan veya dolaylı bağımlı olamaz. İnceleyici, üretici kimliği veya inceleme pin’i eksik tarihsel kabul `historical_unrecorded` görünür ve prepare, record veya dispatch-start aşamalarında yeni bağımlılığı karşılamaz. Özet engeli ve yardımcı aynı bağımsızlık koşulunu kullanır. Otomatik geçmiş taşıması yoktur.

Bozuk tür ve aralıklar durum baytlarına dokunmadan hata verir. Örnekler `usedPercent: "10"`, boolean adet veya zaman, yüzde 101, negatif reset, pencere olarak liste, sonlu olmayan metadata. Eksik/null `rateLimits`, eksik pencere/değer, doğru türde olup 300 veya 10080 dakikalık pencereden biri eksik olan ölçüm ve geçersiz yaşlı ya da gelecek gözlem kullanılamaz sayılır. Böyle geçerli biçimli `usage` eski ölçümü temizler ve `usage_errors` artırır. İlk `RETRY_USAGE_ONCE`, ikinci `SAVE_AND_STOP` olur. Halted veya complete değilse durum stopping olur. Duran run kendi kendine başlamaz. Yüzde 0, 100 ve sonlu kesir geçerli. 100 eşik kurtarmasını tetikler. Bozuk tür kullanılabilirlik sayacını artırmaz. Ham metadata korunur ama gerekli pencerelerin yerine geçmez.

<a name="supported-pinned-entry"></a>

## Desteklenen pin’li giriş

Tüm işlemler `scripts/candidate.py --manifest <manifest> --manifest-sha256 <accepted-pin> --version 1.1` kullanır. Run modları ayrıca `--project <absolute-project>` ve `--run-id <fresh-run>` ister.

| Mod | Girdi | Davranış |
| --- | --- | --- |
| `workflow validate` | Girdi yok | Kabul edilen paketi ve rol politikasını doğrular, etkin olmayan run alanını bağlar. |
| `workflow <typed-command>` | `--input <json-file>` veya stdin | Yukarıdaki tipli sözleşmeleri yürütür. |
| `coordination` | `--input <json-file>` | Sınırlı modelsiz batch yürütür. Yerleşik atama otomatik gönderilmez. |
| `watcher` | `start`, `renew`, `stop`, `status`, `once` | Pin’li özel örneklemciyi ve sahipli run durumunu kullanır. |
| `install` | Açık `--destination .../agent-workflow-v1-1`, isteğe bağlı `--update` | Paket baytlarını ve sahiplik koşullarını doğrular. |

Doğrudan `workflow.py` CLI reddedilir. Kurucu, dışa aktarma ve yerleşik rol üretme girişi içermez. Dağıtım dosyalarını `tooling/build-candidate.py` ile üretin. Codex’te yalnız pin’li kurucuyu kullanın. Eski şema dönüşümü candidate komutu değildir ve emekli kurulumu çalıştırmak için kullanılmaz.

<a name="input-and-state-boundary"></a>

## Girdi ve durum sınırı

Doğru boolean girdiler geçici tekrar testinde özgün kararlarını korur. İsteğe bağlı güvenli false varsayılanı desteklenir. Yanlış türde truthy değerler reddedilir. Katı durum ayrıştırması JSON sözdizimi ve sonlu değerleri korur. Metin yetki ve eski durum kabul edilmez. Yalnız geçici projelerle test edilir.

<a name="typed-assignment-and-source-chronology-a04--gi01"></a>

## Tipli atama ve kaynak sırası (A04 / GI01)

N02 `bind-active-goal` uzun hazırlık sonrası gerçek etkin yerleşik hedefi, insan kaynağı uydurmadan veya zamanını değiştirmeden bağlar. Durum yok ya da tam `{schema_version: 2, status: inactive, tasks: {}}` olmalıdır. Gerçek JSON’da metinler tırnaklıdır. Taşınmış, tamamlanmış, durmuş, halted, daha önce işlem görmüş veya başka şekilde değişmiş yardımcı bağlanamaz. İlk gerçek `get_goal` gözlemi 0–60 saniye olmalı, hedef, sohbet, amaç ve özgün oluşturma/gözlem zamanı bağla eşleşmelidir. Yerleşik oluşturma eski olabilir fakat gözlemden sonra olamaz. Komut amacı ve kapsam eşittir. Rapor veya geniş kapsam reddedilir. Ayrı dolu yorum ve usul kontrolü ana ajanın güncel bağlam ve yeni kısıtları incelediğini kaydeder. Yardımcı bunu kimlik kanıtı olarak doğrulayamaz.

Durumda `native_binding` geçerli dayanak, `native_binding_events` ve `native_binding_ids` ayrı yerleşik geçmiş ve tekrar rezervasyonudur. Her olay `binding`, `interpretation`, `procedural_check` ve özgün `app_goal_observation` içerir. Bağ kimliği sonraki insan olayları için ayrılır. Run ve kapsam kimliği önceki geçmişle ayrılır. Yerleşik dayanak iş için amaç ve oluşturma zamanıyla eşleşen taze hedef gözlemi ister. İnsan yetkisinde tam kaynak şeması ve 300 saniyelik özgün kaynak kuralı geçerlidir. İnsan devamı yerleşik dayanağın yerine geçer, özgün geçmişi korur. Sonraki etkinleştirme de bu geçmişi tutar. Bozuk, eksik, yinelenen ve çelişkili yerleşik kayıt ve rezervasyonlar, insan devamından sonra dahi reddedilir. Stop, snapshot ve halt geçerlidir. Taze etkin hedef gözlemi durmuş yardımcıyı kendiliğinden açmaz.

Schema2 eski görev `inputs` alanını reddeder. Geçmiş metinler açıkça `context_refs` olarak korunabilir. Bu metinler görev veya dosya yetkisi olarak yorumlanmaz. `file_inputs` tam `{path, revision}` nesnelerinden oluşur. Atama proje göreli dosyayı normalize eder. Büyük/küçük harfle eşleşen tekrar, sınır dışı yol, ayrılmış alan ve bağlantıyı reddeder. `sha256` ve `present` pin’lenir, null ve false yokluğu gösterir. Değişmez girdi görevin yazma dosyasıyla çakışamaz. Yazma dosyasının gönderim öncesi ayrı base hash’i vardır. Açık revizyon ve atama hash’i tüm görev sözleşmesini kapsar. Yeniden hash almak eski işi yeni girdilerle kabul etmez.

`review_targets` yalnız doğrulayıcılara izinli `{producer_id, revision}` listesidir. Her üretici tam revizyonda var olmalı ve atama ile inceleme kapılarında taze idle/done/stopped oturum gözlemi taşımalıdır. Atama mevcut üretici çıktılarının hash ve varlığını sabitler. Bağlam metni inceleme yetkisi değildir. Hazır argümanlar tüm tipli pin’leri içerir. Girdi ve üretici sözleşmeleri ile çıktı prepare, record, doğrulayıcı finish ve üretici kabulünde tekrar doğrulanır. Kabul edilmiş bağımlılıklar da denetlenir. A05 kabul/çalışıyor durum makinesi korunur. Bağımsız inceleme bekleyen üretici running kalabilir ve yazmayı bıraktığı gözlenebilir.

Eskimiş sözleşme `INPUT_INVALIDATED` ve `invalidation` makbuzu verir. Makbuz görev kimliği, revizyon, neden, ilgili tam yol, beklenen/gözlenen hash veya revizyon, zaman ve sıralı etkilenen görev kimliklerini içerir. Aynı yolu kullananlar yalnız kendi pin veya sözleşmesi gözlenen bayt ya da yoklukla çelişirse eklenir. Güncel bayta veya yokluğa bağlı yeni sağlam revizyon etkilenmez. Geçersizleşme gerçek bağımlılık ve inceleme ilişkisini izler. Bağlam metni veya yol eşitliği tek başına ilişki kurmaz. Görev çalışma durumları değişmez. `contract_invalid: true` ve kalıcı kayıt eski çıktı kabulünü durdurur. Eski baytın geri gelmesi eski üretici veya incelemeyi canlandırmaz. Denetimi tekrarlamak kayıt ve durum baytlarını korur.

Girdi, incelenen çıktı ve bağımlı çıktı gözlemi hash’ten önce symlink, junction, hardlink, dizinle değiştirme ve sınır dışı yolu reddeder. Güvensiz çıktı atama, prepare, record, finish veya üretici kabulünde hata dönmeden üretici ile ilgili inceleme ve tüketicileri geçersizleştirir. Makbuz beklenen hash veya null, tam göreli yol ve `actual` alanında `unsafe: ` önekiyle neden taşır. Eksik dosyada null kaydedilir. EIO gibi bağımsız G/Ç hatası sözleşme değişimi diye etiketlenmez. Denetim gözlenen yol içindir. Dışarıdan eşzamanlı değişikliğe karşı dosya sistemi genelinde atomik görüntü sunmaz.

İlgisiz kısmi değişiklikler atılır. Doğrudan `apply` yalnız geçersizleşme kayıtlarını verilen durumda tutar. Execute hata vermeden önce bunları atomik olarak yazar. Reset veya yeniden hash komutu yok. Makbuzu okuyun, yeni görev kimliği ve revizyonla taze pin’ler atayın. Diğer retler mevcut bayt koruma politikasına uyar.

GI01 yetkide ve birebir source_message izdüşümünde source_created_at ve original_turn_id ister. observed_at ve observed_turn_id gözlemi ayrı belirtir. Kaynak gözlemden önce olmalı, yeni kabulde yaşı en çok 300 saniyedir. Devam için kaynak oluşturma zamanı stop, halt veya complete sonrasıdır. Son kabul edilen kaynak işareti yeni devam ve etkinleştirmede korunur. Oluşturma zamanı geriye gidemez, eşit zaman aynı özgün turu gerektirir. Olay ve özgün kaynak ID’leri tür, gözlem zamanı, run, sohbet ve devam boyunca tekrar kullanılamaz. Gözlem güncel hedef, run ve sohbetle eşleşir. Açık duruş amaç kapsamıyla da eşleşir. Gelecek niyet, politika ve rapor gözlemi kanıt kaydeder, mevcut işi veya devamı yetkilendirmez. Kullanım eşiği yardımcıyı mekanik olarak durdurur. Hedef gözlemini değiştirmez ve kullanıcı pause yetkisi üretmez.

Bu alanlar çağıranın kanıtıdır. Sağlayıcı doğrulaması değildir. Yardımcı yerleşik hedef araçlarını engellemez ve update_goal çağırmaz. Eski duruş, yeni devam, aynı tur bağlamı tekrarı testi yardımcının reddini ve etkin uygulama gözleminin korunduğunu gösterir. N07 yukarıda açıklanan sürümlü yürütme ve kabul görünümünü ekler. Gerçek taze alt ajan bağlamı, cache ölçümü ve A06/GI02 pause entegrasyonu ayrı kanıt bekler.
