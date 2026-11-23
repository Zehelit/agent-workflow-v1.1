<a name="operation"></a>

# İşlem

Python 3.11+ ve standart kütüphane gerekir. `python -X utf8 -B scripts/candidate.py --help` ile başlayın. Başlatmadan önce [paketin çalışma bağını](coexistence.md) okuyun ve dışarıda saklanan build kaydındaki manifest SHA değerini koruyun. `python` bulunamazsa host’un paketlediği Python’u kullanın. Komutlar JSON’u `--input path` ya da stdin üzerinden alır. Durum `<project>/.agent-workflow-v1-1/runs/<run-id>/state.json` dosyasındadır. Checkpoint, kilit, geçici veri, izleyici sahipliği ve koordinasyon günlükleri aynı izole alandadır. Duruma yalnız ana koordinatör yazar. Dosyaları, test çıktılarını ve snapshot’ları normal prompt girdisi olarak kullanmayın.

<a name="start-and-dispatch"></a>

## Başlatma ve gönderme

Açık çağrı, hazırlık veya küçük görevler dahil esas işe geçmeden önce aynı turda başlatmayı gerektirir. Sadece yönergeyi okumak başlangıç değildir. Yerleşik hedef oluşturma yetki dışındaysa manuel izlemeyi başlatın. Hedef veya bağlama kanıtı uydurmayın. Skill’i tartışmak veya bakımını yapmak durmuş görevi devam ettirmez.

Etkin iş akışı olmadan izlemek için yeni bir run kimliği seçin ve pin’li candidate ile `watcher start --usage-input USAGE.json` çalıştırın. Yalnız izleme başlar. İş akışı etkinleşmez ve ajan yetkisi verilmez. `watcher status` ile doğru run ve epoch için sağlıklı taze örneklem ve canlı sahipli süreci, ya da reddi görün. Sonraki gerçek ölçümler için `watcher ingest`, iş sınırlarında `watcher renew` ve bitişte `watcher stop` kullanıp süreç çıkışını doğrulayın. Eşleşen sağlıklı izleyici yeniden kullanılabilir. Durmuş çalışma kabul edilen devam ve yeni epoch yoluyla, kurtarma kontrolleri uygulanarak açılır. Duruşu aşmak için ikinci izleyici başlatmayın. İşe geçmeden sonucu bildirin. Başlangıç reddedilir veya `SAVE_AND_STOP` istenirse nedeni bildirin ve esas işi bekletin.

Etkinleştirme veya devamdan önce [yerleşik kullanım kayıtlarını](native-usage.md) okuyun. Ana ajan mevcut `mcp__codex_app__get_usage_limits` aracından güncel ölçüm alır ve `--usage-input` ile verir. Python kaydı okur, ikinci Codex app-server açmaz, yerleşik aracı çağıramaz. Uyarlanan son tarihten önce gerçekten yeni kayıt alın. Aynı kaydın tekrar kullanılması yaşını sıfırlamaz. Eksik, süresi dolmuş, tekrarlanmış veya başka run’a ait kayıt işi durdurur. Manuel izleyici de aynı yenileme kuralına tabidir. `watcher ingest --usage-input PATH` yalnız veri yayımlar. İzleyici başlatmaz veya AGWF’yi açmaz.

1. `validate` tanımları denetler. `activate`, `objective`, aşağıdaki tipli yetki, `models` (gözlenen model ve muhakeme listeleri), `slot_limit` (ana ajan dahil etkin slotlar), `inventory` (gerçek alt ajan kimliği ve durumu), `observed_at` (Unix) ve isteğe bağlı `session_limit` alır. Yetenek kanıtı uydurmayın. Etkinleştirme ana sohbetin modelini seçmez veya raporlamaz.
2. Yetkili `activate`/`bind-active-goal`, özel izleyiciyi candidate üzerinden başlatır. Taze ve sağlıklı ilk örneklemden sonra etkin durumu kaydeder. Geçerli `resume` yeni epoch açar. Etkin iş akışı/koordinasyon komutları 600 saniyelik lease’i yeniler. Uzun işler arasında `workflow status` ile sağlık ve yenilemeyi kontrol edin. Lease biter veya ölçüm sağlıksızsa yeni iş kapalı, kurtarma komutları açıktır. `complete`/`halt` duruş ister ve süreç çıkışını doğrular. Manuel `watcher start/status/renew/stop` etkin workflow dışındaki işler ve tanılama için yeni run kimliğiyle kullanılır. Codex’te otomatik başlatma kancası yoktur. Ana ajan uygun yolu başlatıp doğrular. Eskime ve kurtarma için pin’li özel watcher politikasını kullanın. Ölçümcü model çağırmaz ve sohbeti yönetmez. Rutin limit kontrolü ajan başlatmaz. `usage-due`, `REUSE_MEASUREMENT` veya `READ_USAGE` döndürür. Kapsamlı veya paralel işten önce `substantial_work: true` yeni ölçüm ister. `usage`, `observed_at` ve gerçek aracın ham `rateLimits` alanını alır. İki pencere de gerekli. Bilinmeyen ölçüm bir kez daha denenir, sonra kurtarmaya geçilir. Kalan kullanım >%40, >%25, >%20 ve ≤%20 düzeylerinde üst aralık sırasıyla 300, 120, 60 ve 30 saniyedir. Her pencerenin gözlenen tüketimiyle %20’ye tahmini sürenin dörtte birine kısalabilir. Sıfırlanan ve eski örnekler tüketim hızı kanıtı değildir. ≤%25 veya tahmin ≤10 dakika iken `CHECKPOINT_SOON` bitmiş iş için doğrulanmış checkpoint ister. Her mesajda değil, biten yazma adımlarında güncelleyin. ≤%20 ya da bu eşiğe ≤120 saniye tahmininde `SAVE_AND_STOP` gönderimi dondurup erken toparlanma ister. `prepare` uyarlanmış süresi dolan ölçümü reddeder. Boşta sorgu yoktur. Kesilemeyen çağrı veya başka sohbetin ani tüketiminde tahmin koruma garantisi vermez.
3. `task` alanları `id`, `role`, `objective`, `file_inputs`, `context_refs`, `review_targets`, `criteria`, `writes` (silinenler dahil tam proje göreli yollar), `depends_on` ve boolean `independent_review`. Salt okunur işte yazma listesi boş kalır. Dizin dosya sahipliği değildir. `inventory` her gönderimden önce gerçek alt ajan durumunu yeniler.
4. `prepare` görev `id`, isteğe bağlı `effort` ve pin’li isteğe bağlı `current_context` alır. Model, kullanım, slot, bağımlılık ve dosya hash’lerini denetler, kapasiteyi ayırır ve `fork_turns: none` içeren gerçek `collaboration.spawn_agent` argümanlarını döndürür. Atanmış rolün `SKILL.md` ve `MEMORY.md` metni eklenir. Eksik ya da boş belge reddedilir ve her belge manifest kapsamındadır. Argümanları olduğu gibi çağırın. Sonra `record`, `id`, `session_id`, `submitted_arguments`, gerçek `receipt` ve gözlendiyse `effective_model`/`effective_effort` alır. Başarısız veya belirsiz çağrıyı `dispatch_failed` ve `certain_not_started` ile kaydedin. Sonucu belirsiz oturum gerçek envanterle uzlaşmadan gönderimi kapatır. Spawn’ı körlemesine tekrarlamayın.
5. `finish`, `id`, `accepted`, `evidence` ve bağımsız inceleme varsa `review_task` alır. Önce `inventory` ile oturumu boş, durmuş veya bitmiş görün. Yazıcı çalışırken sahipliği bırakmayın. İnceleyicinin `review_targets` alanında hedef görev kimliği ve revizyonu olsun. Üreten ajan yazmayı bitirmiş, çıktı kabulü henüz açık olmalıdır. Hash’ler inceleme boyunca sabit kalmalıdır. Reddedilen iş reddedilmiş kalır. Neden düzeltildikten sonra yeni görev açılır. `cancel` henüz gönderilmemiş hazır işi `id` ve `reason` ile bırakır. Ana ajan entegrasyon öncesi kullanıcı değişikliklerini kontrol eder. Sonucu belirsiz dış işlem tekrardan önce uzlaştırılır.

Görev rollerinde yalnız `gpt-6.1-sol`, `gpt-6-astra` ve `gpt-6.1-terra`, en az `medium` kullanılır. Model ve varsayılan muhakeme düzeyi rol dosyasında sabittir. Otomatik değişim yoktur. Ana sohbeti kullanıcı ayarlar. Paket çalışan modeli değiştirmez. Hazırlanan argümanlar tam rol ayarını korur. Kaldırılan `astra_coordinator` kullanılamaz.

Limit-watcher tam olarak `gpt-6.1-terra`/`medium` kullanır. Alternatif model yoktur. Etkinleştirmede fiilen gözlenen model ve muhakeme desteğini kaydedin. Kesin rol ayarı yoksa hazırlık reddedilir.

<a name="recovery-and-stop"></a>

## Kurtarma ve durdurma

<a name="initial-activation-retry-v11"></a>

### İlk etkinleştirmeyi yeniden deneme (v1.1)

Candidate ilk etkinleştirmeden önce `activation-attempt.json` yazar. İzleyici sağlıklı başlar ve çekirdek durum yazımı tamamlanırsa denemeyi tamamlandı olarak işaretler. Hata yanıtı `startup_retry.attempt_sha256` içerebilir. Hata kaydını değiştirilebilir run dizininin dışında koruyun. Taze gerçek kullanım, aynı hedefin taze gözlemi, özgün yetki ve `--startup-attempt-sha256 PRESERVED_FAILURE_PIN` ile aynı `activate` veya `bind-active-goal` işlemini yeniden deneyin. Düzenlenmiş günlüğün hash’ini onay olarak kullanmayın.

| Durum | Desteklenen işlem |
| --- | --- |
| İlk başlatmada kayıtlı ölçüm eskimesi, değişmemiş kimlik ve durum, izleyicinin yokluğu doğrulandı | Harici hata pin’i ve yeni hata sonrası ölçümle aynı ilk komut |
| Mevcut durmuş workflow, kayıtlı ölçüm eskimesi | `usage_recovery: true` ile doğrulamalı `resume`, tüm halt sonrası kontrollerle |
| Kota, erken rezerv, kullanıcı duruşu, başka hata, değişmiş/yarım günlük veya belirsiz süreç | Teknik geçiş yok. Kanıtı koruyun ve geçerli devam ya da uzlaştırma koşulunu karşılayın. |

Başlatma, kayıt, geri alma, duruş ve yetki geçişleri aynı sahipli candidate kilidini kullanır. Nedeni bilinmeyen `lifecycle.lock` dosyasını kaldırmadan önce sahibini ve yarım kalan işlemi inceleyin. Yerleşik Codex çağrılarıyla tek atomik işlem yoktur. Koordinasyon batch’leri sıralı, günlüklenen ve atomik olmayan işlemlerdir.

Yerleşik ölçümden önce hazırlanın. Uygunsa `sample_at + interval - min(15, interval / 4)` zamanına kadar yenileyin. Bu öneri gerçek son tarihi uzatmaz. Yeni örneklemin zamanını, canlı kimliği ve epoch’u, kalp atışını, örneklem tazeliğini ve lease’i ayrı doğrulayın. Ingest tek başına lease’i yenilemez. Haftalık istisna gerçek alıntı ve sınırlı sona erme zamanı ister. Çekirdek ve izleyici aynı pencerelerle karar verirken iki ham ölçümü de saklar.

<a name="initial-native-goal-binding-n02"></a>

### İlk yerleşik hedef bağı (N02)

Yerleşik hedef zaten etkinse ve hazırlık insan kaynağının tazelik süresini aştıysa `bind-active-goal` kullanın. Yalnızca gerçekten ilk, etkin olmayan schema2 yardımcı durumu açılır. Görev, eski işlem alanı, stop/halt/tamamlanma, taşıma veya replay kaydı olamaz. Mevcut run’lar güncel yetkiyi ve taze uygulama gözlemlerini kullanır. Yerleşik bağlı ve durmuş run, halt sonrasında `paused`/`blocked`, ardından aynı hedef için daha yeni `active` gözlemi kullanabilir. Yalnız `updated_at` farkı kullanıcı durum geçişini kanıtlamaz. Aynı yetkili etkin hedefin ölçüm eskimesinden doğan kesintisinde aşağıdaki teknik yol uygulanır.

`native_binding` içine `binding_id`, hedef/sohbet/run kimlikleri, yeni kapsam kimliğiyle hedef kapsamı, yerleşik `objective`, özgün `goal_created_at` ve güncel `observed_at` verin. Komutun `objective` alanı yerleşik hedefle birebir eşleşsin. Model, kapasite ve envanter kanıtlarını ekleyin. Aynı hedef, sohbet, amaç, oluşturma ve gözlem zamanını, `status: active`, `source_kind: get_goal` ve gerçek makbuz kimliğini ayrı `app_goal_observation` içinde verin. Gözlem en çok 60 saniye yaşında ve gelecek dışı olmalıdır. Özgün oluşturma zamanı daha eski kalabilir. Hedef metnine açıklama ön eki eklemeyin.

`interpretation` ve `procedural_check` boş olmayan, ayrı, ana ajan tarafından yazılmış kanıtlardır. Bağlamadan önce yetkili kapsamı ve daha yeni kullanıcı kısıtlarını okuyun. Eski duruş, rapor veya politika mesajından bağ oluşturmayın. `exact_quote`, `source_message`, `original_turn_id` veya kaynak oluşturma zamanı uydurmayın. Canlı veya belirsiz alt ajan kalmamalıdır. Yardımcı sağlanan biçim ve tutarlılığı denetler. Gerçek kaynağı ana ajan gözler.

Yerleşik bağ ve replay geçmişi sonraki kullanıcı etkinleştirmelerinde ve devamında korunur. Yerleşik yetki için her `app-goal` yenilemesi hedef ve sohbet kimliğiyle birlikte özgün oluşturma zamanı ve tam hedefi içerir. Taze kullanım, envanter, kapasite, belirsizlik ve atama kontrolleri gerekir. Yerleşik hedef etkin görünse bile yardımcı stop/halt durumu geçerlidir. Durdurma, snapshot ve doğrulanmış kurtarma normal çalışır. Telemetri dışı `resume` için halt sonrasında doğrudan `explicit_goal_continuation` veya `native_ui_resume: true` gerekir. İkinci yol halt sonrası `app-goal` ile `paused`/`blocked`, ardından aynı hedef için taze `active` `get_goal` makbuzu ve stop ile halt’tan kesinlikle daha yeni Unix `updated_at` gerektirir. Aynı saniyedeki belirsizlik reddedilir. İki yol da checkpoint, kullanım ve envanteri doğrular. Yerleşik makbuz tek kullanımlıktır ve işlemi kimin yaptığına dair kanıt vermez.

<a name="expired-telemetry-recovery-under-existing-authority"></a>

### Mevcut yetki altında sona ermiş telemetri kurtarımı

`usage_receipt_expired` ana ajanın uyarlanan kayıt süresini kaçırdığını gösterir. Bu gözlem, kullanım hakkının tükendiğini kanıtlamaz. Yeni işi durdurun, kaydedin ve halt uygulayın. Pin’li candidate aynı run ve epoch için gerçek izleyici nedenini `halt` sırasında kaydeder. Candidate’a `usage_failure` alanını kendiniz vermeyin. Yalnız kurtarma amaçlı koordinasyon batch’i durdurma ve snapshot yapabilir. Yaşam döngüsündeki `halt` ve `resume` doğrudan candidate komutlarıdır.

Bu kayıtlı neden için duruştan sonra gerçek kullanım ve aynı etkin hedefin taze `get_goal` gözlemini alın. Sahipli alt ajan envanterini yenileyin. Checkpoint, canlı dosya hash’leri ve süreç çıkışını doğrulayın. `workflow usage` ve `inventory` kaydedin. Sonra `workflow resume --input RECOVERY.json --usage-input USAGE.json` çalıştırın. `RECOVERY.json` yalnız `usage_recovery: true` ve `app_goal_observation` içermelidir. Yerleşik bağda özgün objective ve hedef oluşturma zamanı korunur. Değişmiş `updated_at`, pasif durum gözlemi, yeni kullanıcı sözü veya uydurma tıklama makbuzu gerekmez. Geçerli mevcut yetki kullanılır. Candidate yeni epoch açmadan kayıtlı neden ile gerçek watcher latch’ini karşılaştırır ve sağlıklı izlemeyi bekler. Sınırlı haftalık istisna ve eski SAVE kanıtı korunur. Eski görevler reddedilir ve kendiliğinden yeniden başlamaz.

Gerçek kota, erken rezerv, eksik, geçersiz ya da başka telemetri hatası, kayıtlı neden yokluğu, başka hedef, eskimiş ölçüm, etkin olmayan hedef, canlı veya belirsiz alt ajan, değişmiş checkpoint ve devamla geçersiz kılınmamış açık kullanıcı duruşu bu yoldan geçmez. Yerleşik hedef durumu değiştirilmez. Kullanıcı duraklatılmış veya engellenmiş hedefi Devam Et ile etkinleştirebilir. Gerçek etkin hedefi gözleyip aynı teknik kontrolleri uygulayın. Aynı hedef etkin kalıyorsa otomatik goal turları da teknik toparlanmayı yapabilir. Yalnız telemetri eskidi diye yeni yazılı devam istemeyin veya `blocked` yazmayın. `update_goal` aracının kendi koşullarına uyun. Etkin durumun zaman damgası, kimin tıkladığını kanıtlamaz. Açık kullanıcı duruşu yalnız kabul edilmiş doğrudan devamla veya güncel olarak yerleşik bağlı run’da doğrulanmış pasif→etkin geçişle aşılabilir. `usage_recovery` bunun yerine geçmez.

Yerleşik hedefe bağlı olmayan manuel izleyici çalışmaları workflow `usage_recovery` kullanmaz. Açık devam ve yeni epoch kuralları geçerlidir. Önceki paketten gelen ve nedeni kayda geçmemiş halted run otomatik yükseltilmez veya yeniden etiketlenmez. Pin’li kanıtı koruyun ve paket güncellemesi ile çalışma bağı kurallarına uyun. Bu yol, nedenin mevcut paket tarafından kaydedilmesini gerektirir. Kaynak bilgisi eksikse uydurmayın ve eksikliği bildirin.

<a name="authorization-lifecycle-a26--auth01"></a>

### Yetki yaşam döngüsü (A26 / AUTH01)

`authorization` açık bir olay nesnesidir. Eski ve dolu bir `authorization` metni de dahil olmak üzere düz metin reddedilir. Eski durum içe aktarılamaz. Yeni iş, güncel paketle ve güncel yetkiyle yeni run ister. `interpretation` ayrı bir üst seviye metindir. Kullanıcının sözünü noktalamasına kadar koruyun. Ana ajanın yorumunu alıntıya eklemeyin. Ayrı `source_message` kaydı, olayın alıntısı ve kaynak bilgisiyle birebir eşleşmelidir.

Devam girdisi örneği. Kimlik ve zamanlar sentetiktir. Gerçek gözlemleri kullanın.

```json
{
  "authorization": {
    "event_id": "user-event-42",
    "kind": "explicit_goal_continuation",
    "exact_quote": "Continue this goal now.",
    "source_kind": "user_goal_continuation",
    "source_id": "user-message-42",
    "goal_id": "goal-7",
    "thread_id": "thread-3",
    "run_id": "run-5",
    "scope": {"kind": "objective", "id": "scope-5", "objective": "Deliver the authorized project change"},
    "observed_at": 1791230401,
    "source_created_at": 1791230401, "original_turn_id": "turn-42", "observed_turn_id": "turn-42"
  },
  "interpretation": "Main classified this source as immediate continuation of the matching objective.",
  "source_message": {
    "source_kind": "user_goal_continuation",
    "source_id": "user-message-42",
    "exact_quote": "Continue this goal now.",
    "source_created_at": 1791230401, "original_turn_id": "turn-42"
  },
  "app_goal_observation": {
    "goal_id": "goal-7", "thread_id": "thread-3", "status": "active",
    "source_kind": "get_goal", "source_id": "actual-goal-tool-receipt",
    "observed_at": 1791230402
  }
}
```

Kaynak türü `user_message` veya `user_goal_continuation` olabilir. Kimlikler ve alıntı boş olmayan metin olmalıdır. Olay gözlemi 300 saniye sonra eskir ve gelecek zaman kabul edilmez. Olay kimliği ve kaynak mesajı, tamamlanmış çalışmalar arasında bile iki kez kullanılamaz. Kaynak mesajı, iki tür için de sohbet kimliği ve kaynak kimliği çiftiyle tanınır. Olay kimliğini veya türü değiştirerek aynı kullanıcı mesajını yeniden kullanmayın. Hedef gözleminin hedef ve sohbet kimliği eşleşmeli, kaynak `get_goal`, makbuz kimliği dolu ve zaman damgası sonlu olmalıdır. Olaydan önce olamaz ve 60 saniyeyi aşamaz. Esas işten önce gerçek ve taze gözlemi `app-goal` ile kaydedin. `task` ve `prepare` gözlemi güncel saate ve yetki kimliğine göre yeniden doğrular. 60 saniye kabul edilir. 61 saniye, gelecek tarih ve eşleşmeyen kimlik durum baytlarına dokunmadan reddedilir. Kullanım ve envanter yenilemesi hedef kanıtını yenilemez. Bu alanlar çağıranın sağladığı kanıttır. Python biçimi, eşitliği, tazeliği ve geçiş kurallarını doğrular. Sağlayıcıya ulaşmaz, transkript veya araç kaydının kaynağını doğrulamaz, doğal dili sınıflandırmaz. Ana ajan gerçek kaynak ve araç kayıtlarını koruyup olay türünü doğru sınıflandırır. İki uydurma alıntı eşleşebilir. Bu eşitlik kaynak doğrulaması değildir.

| Olay türü | Yeni etkinleştirme | Durmuş eski hedef | Görev gönderimi |
| --- | --- | --- | --- |
| `explicit_task_authorization` | Ayrı run ve kapsam, eşleşen hedef ve gözlenen etkin uygulama hedefi | Devam etmez | Yalnız yeni yetkili run |
| `explicit_goal_continuation` | Etkin hedef kanıtıyla ayrı run ve kapsam mümkündür | Aşağıdaki kontrollerden sonra devam | Başarılı devamdan sonra yeni görevler |
| `future_goal_intent` | Reddedilir | Reddedilir | Eski işi yetkilendirmez |
| `policy_change` | Reddedilir | Reddedilir | Eski işi yetkilendirmez |
| `independent_report_request` | Yalnız ayrı `kind: report` kapsamı ve o raporun tam hedefi | Eski durdurulmuş run üzerinde devam veya etkinleştirme reddedilir | Ayrı rapor run’ı, eski iş durur |
| `explicit_stop` | Reddedilir | Eşleşen kimlik ve kapsam için `observe-authorization` ile kaydet | Yeni görev ve hazırlığı hemen dondur |

`observe-authorization` gelecek niyeti, politika değişikliği, ayrı rapor talebi ve kapsamlı duruşu kaydeder. Hedefi etkinleştirmez veya devam ettirmez ve mevcut yetkiyi değiştirmez. Duran hedef için rapor ayrı yardımcı proje ve durumla yürütülür. Eski run’ı görevlerine ulaşmak için değiştiremez. `app-goal` taze gözlemi ayrı kaydeder. Duraklatılmış, engellenmiş, durdurulmuş veya tamamlanmış hedef gözlemi, etkin yardımcıyı dondurur. Sonraki etkin gözlem tek başına devamı kanıtlamaz. Arayüz yolunda halt sonrası önce etkin olmayan durumu gözleyin. Kayıtlı ölçüm eskimesinde mevcut yetkiyle `usage_recovery` kullanılır. `status` yardımcı `status` ve `app_goal_observation` alanlarını ayrı döndürür. Gönderim talimatı yetkili kapsamı, hedefi, sohbeti ve run kimliğini içerir.

Doğrudan kullanıcı mesajıyla devam etmek için stop ve halt’tan kesinlikle sonra gözlenmiş yeni `explicit_goal_continuation` olayı gerekir. Kimlikler ve kapsam birebir eşleşmelidir. Önce duruştan sonra gerçek kullanım ve alt ajan envanterini okuyun. Kullanım taze, gerekli payın üzerinde ve uyarlanan süresi içinde olmalıdır. Envanter güncel olmalı ve canlı veya belirsiz sahipli alt ajan içermemelidir. Geri yükleme ve güncel dosya hash’lerini yeniden doğrulayın. Eşleşen hedefin halt’tan sonra ve devam gözleminden erken olmayacak şekilde etkin olduğunu görün. Eksik durumlu kurtarmada checkpoint önce onarılır. Başarılı devam bitmemiş eski görevleri reddeder ve yeni atamaya izin verir. Eski ajanı kendiliğinden yeniden başlatmaz. Eksik, eski, tekrar kullanılmış, bozuk veya devam niteliği taşımayan yetki durum dosyasını değiştirmeden reddedilir.

Açık duruşta ana ajan esas işi ve yeni görev gönderimini hemen bırakır. Yalnız kurtarma için gereken kısa sahipli yazıcı durdurma, onay alma, koruma ve halt adımlarını yapın. Yeni plan, geniş iş devri veya kurtarma altyapısı kurmayın. Uygulama gözlemi ile yardımcı geçişini ayrı kaydedin. Yardımcı yerleşik araçları engellemez. Kaynağı sınıflandırma ve yerleşik duruş ana ajanın usul sorumluluğudur. A26’nın gerçek devam/duruş kayıtları ve yetkisiz araç işlemlerinin bağımsız denetimi geçici Python testlerinin dışında ayrı kanıt gerektirir.

`stop` gönderimi dondurur ve çalışan yazıcılar için SAVE_ACK ister. Her yazıcıdan kısa onay alın. En geç 30 saniye sonra gerektiğinde yanıt vermeyen sahipli yazıcıyı kesin ve bellekte kaydedilmemiş veri kalabileceğini bildirin. Snapshot öncesi diskte yazma durmalıdır. Yardımcı kendi başına mesaj göndermez, süreç kesmez veya sürekli envanter sorgulamaz.

İzleme sağlıksız veya `SAVE_AND_STOP` kayıtlı olsa da pin’li candidate üzerinden `stop` mümkündür. Kurtarma için önce sağlıklı ölçüm gerekmez. Doğrudan `stop` → `snapshot` → `halt` ve yalnız kurtarma batch’i aynı izinli komutları kullanır. Bunlar yeni görev, SAVE kaydını kaldırma, sağlıksız lease’i yenileme veya devam yetkisi vermez. `halt` yine durmuş envanteri, checkpoint bütünlüğünü ve sahipli worker çıkışını doğrular.

Yazıcıların durduğu görüldükten sonra `snapshot`, değişen, eklenen ve silinen proje göreli `files`, `next_action` ve `writers_quiescent: true` alır. Görevlerin sahipli yolları otomatik eklenir. Ana ajanın yazdığı ve Git’in izlemediği dosyaları açıkça ekleyin. Kopyalama sırasında hash’ler sabit olmalıdır. Manifest silinen ve bulunmaması gereken yollarla durum kopyasını kaydeder. Ayrı yeni iç dizinde geri yükleme denenip hash’ler doğrulanır. Kimlik bilgisi yüklenmez. Dış araçla elde edilen ve kurtarılması gereken içerikleri önce o araçtan yerel olarak koruyun.

Sahipli ajanları ve süreçleri host araçlarıyla durdurun. `halt` gerçek gözleme dayanan taze `inventory`, `observed_at`, `owned_processes_stopped: true` ve `stop_evidence` alır. Checkpoint ve güncel dosya hash’lerini tekrar denetler. Snapshot veya geri yükleme başarısızsa `degraded: true`, `reason`, `next_action` ve gerçek duruş kanıtı kullanın. Doğrulanmış kurtarma iddia etmeyin. `resume` tanımlanan ölçüm eskimesi, doğrudan devam veya daha yeni yerleşik arayüz geçişlerinden biriyle ve bütün kurtarma kontrolleriyle yapılır. Eksik durumlu halt’ta önce checkpoint’i onarın. Otomatik reset kullanmayın, yerleşik hedef durumunu değiştirmeyin. Telemetri kurtarması zaten yetkili etkin hedefin yardımcısını geri açar.

`restore` doğrulanmış checkpoint’i proje dışındaki YENİ hedefe kopyalar. Proje içeriği `files/` altında, kurtarma metadatası bu ağacın dışında kalır. Manifest bulunmaması gereken yolları kaydeder. Canlı dosyalar silinmez ve üzerlerine yazılmaz. Kalıcı snapshot kaydından sonra güncel ve bir önceki sağlam checkpoint’i saklayın. Yalnız sahipli eski dizinleri budayın. Bağlantıları reddedin, bilinmeyen içeriği koruyun. `compact` için etkin alt ajan veya bitmemiş görev olmamalı ve snapshot doğrulanmış olmalıdır. Komut bitmiş görev kayıtlarını arşivler. `complete` teslim edilmiş run’ı kapatır. Gelecek iş yeni etkinleştirme ister.

<a name="mechanical-coordination-and-durable-dispatch-n05"></a>

## Mekanik koordinasyon ve kalıcı görev gönderimi (N05)

`scripts/candidate.py <global pins/options> coordination --input <request.json>`, standart kütüphaneyle çalışan modelsiz bir adaptördür. İstek biçimi `{ "batch_id": "unique_id", "steps": [{ "key": "unique_step", "command": "task", "payload": { ... } }], "seen": [] }`. En çok 64 adım sırayla çalışır. Her workflow komutunun denetimi, durum kilidi ve kaydı ayrıdır. Batch ortak bir atomik işlem değildir. Bağımsız okuma için `read` komutunda `payload.paths` tam proje göreli dosyaları içerir. Günlük okunan metni, bayt sayısını ve SHA-256’yı kaydeder. Olmayan dosyanın pin’i null olur. Sonuç, engeli, gözlenen oturum kimliklerini, workflow kaynak pin’ini, günlük yolunu, adım sonuçlarını ve bir sonuç token’ını içeren kısa karardır. Ham istek, kayıt ve sonuç `.agent-workflow-v1-1/runs/<run-id>/coordination/<batch_id>.json` içinde kalır. Daha önce tüketilen token’ları `seen` ile vererek değişmemiş tamamlanan sonucu tekrar göndermeyin. Yerleşik araç çıktısı bu adaptörün dışındadır.

Her adım yürütmeden önce önceki durum hash’iyle `pending` olarak yazılır. Daha sonra sonuç ve yeni durum hash’iyle `applied` ya da hata ve yeni hash ile `failed` olarak kaydedilir. Durum yazımı ile sonuç kaydı arasında arıza olursa adım `pending` kalır. Batch tekrar çağrılınca engeli döndürür ve başka adım uygulamaz. Uygulanan adımlar tekrarlanmaz. Hatalı adım önceki sonuçları koruyarak batch’i durdurur. Aynı ID’yi farklı istekle kullanmayın. Niyet ve sonuç kaydını silmeyin. Durum baytları aynı diye tekrar güvenli değildir, snapshot veya restore diskte oluşmuş olabilir. Günlük ve güncel çekirdeği inceleyip gerçek dış kayıtları koruyun. Bilinmeyen yan etki çözüldüğünde ayrı ve sınırlı uzlaştırma batch’i kullanın. Çökme başarıya dönüştürülmez. Günlük ve çekirdek kilitleri kendi yazımlarını korur. Yalnız ana ajan yazar. Diğer araçlarla genel bir işlem kilidi yoktur. Eski kilidi kaldırmadan sahibini inceleyin.

Günlüklenen yerleşik işlem:

1. Gönderim öncesi gerçek yerleşik envanteri alın ve `inventory` kaydedin. `prepare` olağan kesin argümanlara ek olarak çekirdek görev kaydında ayrılmış `intent_id` döndürür.
2. Gerçek aracı çağırmadan hemen önce `id` ve `intent_id` ile `dispatch-start` çalıştırın. Yetki ve hedef tazeliği, kullanım ve SAVE, envanter, görev/girdi geçerliliği ve yazma hash’lerini yeniden denetler. Tam `collaboration.spawn_agent` argümanlarını vermeden önce `uncertain` yazar. Zaman aşımı, kayıp sonuç veya yeniden başlatma ikinci gönderimi engeller. Araç çağrısından önceki çökme de belirsizdir. Çözmeden önce gerçek envantere bakın. Eski sıralı `prepare` / yerleşik çağrı / `record` yolu kullanılabilir ve çağrı öncesi belirsizlik işareti içermez.
3. Ana ajan gerçek `collaboration.spawn_agent` aracını desteklenen yoldan doğrudan çağırır. Python veya `functions.exec` kullanmaz. Gerçek sonucu koruyun. Taze envanteri `observation: { "source_kind": "collaboration.list_agents", "source_id": "caller-capture-correlation-id", "evidence_path": "operations/raw-native-inventory.json", "sha256": "<actual-file-hash>" }` ile kaydedin. Tam ham çıktı proje göreli dosyada bir kez tutulur, durum ve günlük yalnız yol ve pin taşır. Alternatif `receipt: <actual-native-output>` ham çıktıyı durum ve günlükte çoğaltır. İki biçimden yalnız birini kullanın. Yerleşik envanter kanıtı `dispatch-start` ve geç başarıyı uzlaştırmak için gerekir. 60 saniyelik süreyi ve kimlik/durum kontrollerini değiştirmez. Çekirdek her kullanımda kanıtı yeniden okuyup pin’i doğrular. `native_inventory(raw)` desteklenen gerçek `{agents:[{agent_name,agent_status}]}` verisini normalize eder. `running` ve `idle` korunur. `interrupted`, `stopped` olur. Tam `{completed: <final text string>}` nesnesi `done` olur. Bilinmeyen, bozuk, belirsiz ve yinelenen girdiler reddedilir. Çağıranın normalize envanteri sıralama dışında ham izdüşümle aynı olmalıdır. Bitmiş metin kanıtta kalır.
4. Belirsiz görevi `id`, `evidence`, `no_owned_work_running: false`, tam `submitted_arguments`, gerçek `session_id` ve `receipt` ile uzlaştırın. Gözlenen model ve muhakeme düzeyi isteğe bağlıdır. Makbuz nesnesinin `id`, `agent_id`, `session_id` veya `task_name` alanı envanter kimliğiyle eşleşmelidir. Oturum, niyet zamanından önce olmayan taze envanterde bilinen durumda olmalıdır. Hazır argümanlar ve değişmez girdiler aynı kalmalıdır. Tekrar kullanılan oturum reddedilir. `Coordination.receipt(...)` geçişten önce ham makbuzu günlüğe yazar. CLI’da aynı `reconcile` adımı kullanılabilir. Eksik, eskimiş, yanlış kaynaktan, bozuk veya kimliği tutarsız kanıt görevi çalışır duruma getiremez. Duruş sırasında geç görülen oturum, yardımcı `stopping` durumunda kurtarma için kaydedilir. Yetki veya yeni gönderim açılmaz. Girdi geçersizleşmişse makbuzu ve günlüğü koruyup alt ajanı gerçek araçla durdurun.
5. Kabul kapısında yeni gerçek terminal envanteriyle bitirin. Bağımsız inceleme ve çıktı pin kontrolleri geçerlidir. Kapanış ve kurtarma kapıları gerektirdiğinde envanteri tekrar alın. Canlı veya belirsiz sahipli iş kalmadığı gerçek envanterle görülürse `reconcile` ve `no_owned_work_running: true` belirsiz görevi reddeder. Görev yeniden gönderilmez ve makbuz yokken başarılı sayılmaz.

Yardımcı biçimi, kaynak etiketini, tazeliği ve tutarlılığı denetler. Çağıran, yerleşik makbuz veya transkriptin gerçekliğini doğrulayamaz. Ana ajan gerçek araç çıktısını elde edip korur. Sağlayıcı çağrı kimliği vermiyorsa `observation.source_id` saklanan ham sonucu işaret eden ve ana ajan tarafından üretilen korelasyon kimliğidir. Sağlayıcı tarafından verilmiş ya da doğrulanmış sayılmaz. Gönderimin gerçek ajan/oturum/görev kimliği yine eşleşmelidir. Kimlik eşleşmesi sağlayıcı metadatasını doğrulamaz. Prepare/spawn/record atomik değildir. Adaptör yerleşik aracı engellemez ve çıktısını bastırmaz. Günlükte `source_pin` workflow.py ve coordination.py SHA-256 değerlerinin standart eşlemesini hash’ler.

`collaboration.list_agents` bilinen canonical görev öneki için `path_prefix` destekler. İlgili sahipli gruba gereken tek kapı gözlemini yapın. Önek bilinmiyorsa önce envanteri bulun. Önek sorgusu tamamlanan ajanların tam son metnini döndürebilir. Yerleşik `compact`, `exclude_completed` veya çıktı filtresi yoktur. Yerel sonuç token’ları yalnız tekrar teslim edilen yardımcı sonucunu bastırır. Sağlayıcı baytlarının filtrelendiğini kanıtlamaz.

Depo kökünden `python -X utf8 -B -m unittest discover -s skills/agent-workflow-v1-1/tests -p test_candidate_coexistence.py -v` çalıştırın. Kaynak ve geçici paket girişleri, korumalı batch ve yaşam döngüsü sınanır. Eski doğrudan çekirdek denemesi candidate hazır olduğunu kanıtlamaz. İki giriş de görev ve girdi pin’i, hazırlık, sentetik makbuz ve terminal gözlemi, kabul, doğrulanmış snapshot ve kapanış yapar. Test, alt süreç çağrı sayıları ve çıkışlarıyla stdout/stderr baytlarını raporlar. Bunlar modellenen karar sınırlarıdır, gerçek ana model çağrı sayısı değildir. Sentetik makbuz ve çağrılar yerleşik gönderim kanıtı olmaz. Ana ajan kendi sıralı ya da batch çağrılarını gerçek araç kayıtlarıyla ayrıca ölçer. Zamanlama, geçici yol uzunluğu ve biçim bayt sayısını etkileyebilir.

<a name="install-and-update"></a>

## Kurulum ve güncelleme

`tooling/build-candidate.py` ile yeniden üretilebilir paket, ZIP ve harici build kaydı oluşturun. Candidate `install --destination <explicit-directory>/agent-workflow-v1-1` ile her paket baytını doğrular. Aynı sahipli paket değişmez. `--update`, yalnız sahipliği doğrulanan ve değiştirilmemiş kurulumu değiştirir. Düzenlenmiş veya sahipsiz içerik reddedilir. Yerine alma başarısızsa önceki dizin kurtarılır. Başarılı güncelleme eski paket ve kalıcı geri alma arşivi bırakmaz. Kesin komutları ve pin’i [paket bağlama rehberinde](coexistence.md) görün.

Dolaylı skill seçimi kapalıdır. Pin’li kaynak, geçici veya kurulu mapping run’ın çalışma kimliğidir. Aynı run içinde eşlemeyi değiştirmeyin. İzleyici motoru, rolü ve yönergeleri `private/limit-watcher` içinde paketlenir ve yalnız pin’li candidate kontrol eder. Güncelleme yeni onaylı run kullanır. Eski kanıtı taşımaz ve yeniden bağlamaz. Paketi üretmek veya kurmak AGWF’yi etkinleştirmez ve ajan kaydetmez. Proje hazırlığı için açık çağrı alındığında başlangıç kuralı aynı kapsamda geçerlidir. Ana istemcinin modeli kullanıcının seçimine bağlıdır.
