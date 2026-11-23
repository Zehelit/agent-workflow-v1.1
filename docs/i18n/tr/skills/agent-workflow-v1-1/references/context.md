<a name="selective-current-context-n06"></a>

# Seçili güncel bağlam (N06)

`tools/render-active-run.py`, kaynakta veya kurulu skill’de geçici ya da açıkça seçilen projeyi okur. İş akışını etkinleştirmez, yenilemez, taşımaz ve durum yazmaz. Gerçek gözlem zamanını `--as-of` ile verin. Bu değer özgün kaynak oluşturma ya da tur zamanını değiştirmez. Sabit JSON sözleşmesi `context_version: 1`. Aynı durum, kaynak, referans ve gözlem zamanı aynı sonucu verir.

Çıktı hedefi, güncel kimliği, geçerli yetkiyi veya yerleşik bağı, özgün kaynak ve hedef zamanlarını, güncel hedef gözlemini, açık görev sözleşmelerini, bağımlılık ve inceleme pin’lerini, çalışma ve kabul gözlemlerini, engel ve karşı kanıtı, değişen çıktı hash’lerini, kanıt referanslarını ve sonraki adımı içerir. Makbuz içeriği, hazırlanan prompt, tam rapor, bitmiş metin, transkript ve araç logu dışarıda kalır. Kabul edilen görevlerin sözleşmeleri özgün pin’li durumda saklanır. Özet kabulü ve tam kanıt referansını gösterir.

Referanslar isteğe bağlı `task_id` içeren `{path, sha256, kind}` nesnelerinin katı JSON listesidir. `kind`, `evidence`, `decision` veya `counterevidence` olabilir. Yol tam ve güvenli proje göreli dosyadır. SHA-256 küçük harfle tam yazılır. Rapor gövdesi kaynağında kalır. Ana ajan yeni referans manifesti veya revizyonu açıkça düzenleyene kadar karşı kanıt engel olarak kalır. Özet kısaltmak engeli kaldıramaz. Eksik veya değişmiş hash, desteklenmeyen pin’siz referans, terminal sonuç kanıtı yokluğu, bekleyen veya geçersizleşmiş bağımlılık, reddedilmiş/belirsiz iş ve eskimiş hedef/envanter/kullanım görünür engeldir. Bozuk kaynak durumu, eksik bağımlılık veya geçersiz pin reddedilir.

Korumalı özet kullanımında manifest JSON’unu atama sırasında değişmez `file_inputs` girdisi olarak pin’leyin. `prepare.current_context` tam `{path, sha256, manifest_path, manifest_sha256}` alanlarından oluşur. Yardımcı manifesti tipli girdi pin’iyle karşılaştırıp ayrı okur. Özgün kanıttan ve güncel durumdan özeti yeniden üretip tamamını karşılaştırır. Kısaltılmış özeti yeniden pin’lemek, engeli kaldırmaz veya eski alıntıyı yeni yapmaz. Özet gözlemi 0–60 saniye yaşında olmalıdır. Hazırlama durumu değiştirdiği için aynı manifest pin’iyle `dispatch-start` öncesi taze özet üretin. İkinci özet eksik veya manifest farklıysa işlem reddedilir. Yetki, kullanım, kimlik, kapasite, hazır argüman ve dosya/revizyon kontrolleri geçerlidir. Özetin `dispatch_authorized` alanı her zaman false.

`current_context` kullanmayan komutlar doğrudan kaynak okumaya devam eder. Özet yerleşik araca yetki vermez. Ana ajan gerçek makbuz ve kullanıcı kaynağını doğrular. Hash bayt eşitliğidir, sağlayıcının kimliğini kanıtlamaz. Dosya gözlemi dış yazarlara karşı atomik snapshot değildir.

`cache_status` `unknown`, `token_measurement` `pending_N08` kalır. Özet sözleşmesi tek başına taze alt ajan bağlamı, önbellek yararı, token tasarrufu veya kurulu benimseme kanıtı değildir. Yapılabilir işi kaynak referanslarından çıkarın ve doğrulayıcının karşı kanıtını koruyun.

<a name="execution-and-acceptance-n07"></a>

# Yürütme ve kabul (N07)

`status.task_observations`, schema2 üzerinde salt okunur `contract_version: 1` görünümüdür. Görevde revizyon, gözlenen yürütme durumu, oturum, zaman, tazelik, bağımsız kabul, gerekçe, inceleyici ve yazma sahipliği bulunur. Eski veya eksik gözlem bilinmiyor olarak kalır. Yerleşik done/idle/stopped güncel envanterle etkin yürütme kapasitesini serbest bırakır. Çıktıyı kabul etmez. Bağımsız kabul beklerken üreten ajanın yazma sahipliği kalır. İsteğe bağlı açık oturum sınırı açık oturumları ayrı sayar.

Doğrulayıcı yazmayı bırakmış üreticinin tam revizyon ve çıktı hash’lerini sabitler. İnceleyici doğrudan ya da dolaylı olarak üreticinin kabulüne bağımlı olamaz. Kabul bağımlılığı ve inceleme hedefi farklı ilişkidir. İnceleme sürerken üretici kabulü bekler. Kabul, bağımsız kabul edilen doğrulayıcıyı mevcut görev `review` alanına kaydeder, ayrı oturumu, hedef revizyonu, girdi sözleşmesini ve tam çıktı pin’lerini denetler. Bayt veya revizyon değişirse eski inceleme kullanılamaz. Geçersizleşen inceleme eski baytlar geri getirilse de geri gelmez. İnceleyici bağlantısı olmayan eski schema2 kabulü `historical_unrecorded` olarak görünür.

Bağımsızlık için dolu ve farklı üretici/inceleyici oturum kimlikleri, atanan ve kaydedilen inceleme hash’lerinin eşleşmesi gerekir. Bu ölçüt hem görünüm hem bağımlılık kapısında aynıdır. Eksik tarihsel kabul `historical_unrecorded` olarak okunur, prepare, record veya dispatch-start sırasında yeni bağımlı göreve izin vermez. Özet `DEPENDENCY_NOT_ACCEPTED` içerir. Hazırlıktan sonra inceleyici bağlantısı ya da yerleşik kimlik kaybolursa gerçek gönderim reddedilir. Eski kimlikler tahmin edilmez ve taşınmaz.

Bu görünüm, mevcut isteğe bağlı alanı kullanır. schema2 değişmez. Eski şemalar reddedilir, içe aktarma yoktur. Geçerli yetki ve onaylı runtime bağı belirleyicidir. Özet bunları başlatmaz veya değiştirmez.

<a name="producer-handoff-gate"></a>

# Üretici teslim kontrolü

`tooling/check-producer.py` kaynak geliştirme kontrolüdür. Checkout’tan paketli Python ile `-X utf8 -B`, önce normal ve sonra `-O` ile çalıştırın. Kaynak testlerini gerektirir ve kurulu çalışma komutu değildir. Değişen yolları ve ortak API’yi bildirin. Sabit test seçicileri çekirdek, girdi, yetki, hedef ve koordinasyon regresyonlarını, bağlam ve kabul olumsuz testlerini ve bilinen A04/A09/A11 karşı örneklerini kullanır. A11’in eski çalıştırıcısı izole kaynak kopyasında candidate workflow pin’iyle bellekte doğrulanır. Özgün baytlar korunur, iki bağımlılık pin’i raporlanır. Bu komut geniş bir monitor veya tarayıcı test paketi çalıştırmaz.

Makine sonucu sabit candidate girdileriyle bildirilen değişmiş dosyaların birleşimini pin’ler. Sabit liste dışındaki Markdown da kapsamdadır. Yollar mevcut, normalize, proje göreli dosya olmalıdır. Sınır dışı yollar, mutlak/drive yolları, takma adlar ve bağlantılar reddedilir. Tüm birleşim seçili testlerden önce ve sonra hash’lenir. Bildirilen dosya değişirse PASS engellenir. `--expected-pins` tüm birleşimle eşleşmelidir. Seçiciler, sayılar, hatalar ve kaynak kararlılığı raporlanır. `independent_acceptance: false` alanı her zaman korunur. Başarısız test, eksik seçici, değişmiş pin veya bozuk ret metadatası üretici PASS sonucunu engeller. Üretici işi bitirdikten sonra önemli sonuç bir kez bağımsız incelenir.

İsteğe bağlı ret metadatası yalnız `rejections` ve `rework_seconds` alanlarıdır. Veri yoksa gözlenen ret listesi boş ve maliyet bilinmiyor kalır. Her ret hata kimliğini ve sınıfını, yeniden üretim referansını, tek düzeltici kimliği ve odaklı inceleyici kimliğini, revizyonu ve tam kaynak pin’lerini içerir. Düzeltici kendi işini inceleyemez. Aynı sınıf ikinci kez reddedilirse kısa `root_plan_correction` kaydedilir. Kararı ana ajan alır. Bir düzeltici ve yeni bağımsız inceleyici atar. Otomatik mimar veya ekip katmanı yoktur.
