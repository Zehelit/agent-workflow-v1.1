---
name: agent-workflow-v1-1
description: Coordinate substantial project work across coding, research, writing and analysis; delegate bounded tasks when useful.
---


# agent-workflow-v1.1

Bu skill’i kullanıcı bu çalışma için agent-workflow-v1.1’i açıkça seçtiğinde kullanın. Kendiliğinden çağrılmaz. Güncel açık çağrı, hazırlık ve küçük işlerde de ana ajanın yetkili kapsamda aynı tur içinde AGWF’yi başlatmasını ister. Önceden kapatılan çalışma bu kapsamda yeniden etkinleştirilebilir. Geçmiş bir seçim veya başka bir oturum bunu yapmaz. Skill’i tartışmak, alıntılamak, incelemek veya güncellemek eski çalışmayı yeniden başlatmaz. Ana sohbet teslimattan, durum yazımlarından ve gerçek ajan çağrılarından sorumludur. Modeli ve muhakeme düzeyini kullanıcı Codex’te seçer. Başlatma sonrası küçük işler ana ajanda yürüyebilir. Ekip isteğe bağlıdır, başlangıç ve izleme zorunludur.

<a name="required-startup-on-explicit-invocation"></a>

## Açık çağrıda başlangıç

Araştırmaya, düzenlemeye, teste veya görev atamaya geçmeden önce gerçek ve taze bir yerleşik kullanım kaydı alın. Başlangıcı pin’li candidate üzerinden yapın. Yetki ve gerekli kanıt varsa `workflow activate`/`bind-active-goal` kullanın. Kullanıcı kapsamı hazırlıkla sınırlamışsa, yerleşik hedef oluşturmayı yasaklamışsa veya etkin hedef vermemişse [işletim](references/operations.md) rehberindeki manuel izleyiciyi başlatın. Açık izin olmadan bir yerleşik hedef oluşturmayın. Skill kataloğunu yüklemek, seçiciyi açmak veya kullanım hakkına bir kez bakmak başlangıç doğrulaması değildir.

Seçilen run ve epoch için canlı sahipli izleyici ile taze ve sağlıklı örneklemi doğrulayın. Tam iş akışında ayrıca etkin workflow state’i kontrol edin. Sonucu işten önce bildirin. Yetkili kapsamla eşleşen sağlıklı çalışma varsa onu kullanın. Bir duruştan sonra yeni bir açık çağrı mevcut devam veya yeni epoch yolunu yetkilendirebilir. Gerçek kullanıcı olayını ve kurtarma kontrollerini kaydedin.

Başlatma reddedilir, kanıt eksik kalır veya izleme `SAVE_AND_STOP` isterse somut nedeni hemen bildirin ve yalnız gerekli koruma ve kurtarma adımlarını yapın. Ana ajanda sessizce yeni işe devam etmeyin. Yeni duruş talimatı önceki başlatmaya üstün gelir. Duruş sırasında skill’i incelemek veya güncellemek, önceki görevi canlandırmaz.

Komutlar, koordinasyon ve limit izleyicisi için pin’li `scripts/candidate.py` girişini kullanın. Giriş, kabul edilmiş manifest pin’iyle terminalden de çalışabilir. Bunun için skill veya watcher çağrısı önkoşul değildir. Doğrudan core CLI çağrıları reddedilir. Harici güven dayanağı ve run yalıtımı için [paket ve çalışma bağını](references/coexistence.md), başlangıç için [işletim rehberini](references/operations.md) okuyun. Model ayarları rol TOML dosyaları ve `private/limit-watcher/role.json` ile belirlenir. İzleyici motoru, yönergesi ve rolü bu paket içinde doğrulanır.

<a name="role-skills-and-memory"></a>

## Rol skill’leri ve bellek

Her alt ajanın kendi `SKILL.md` ve `MEMORY.md` dosyası vardır. Koordinatör, mimar, uygulayıcı, araştırmacı ve doğrulayıcı `roles/<role>/`, limit_watcher ise `private/limit-watcher/` altındadır. Rol tanımları kendi belgelerini belirtir. Hazırlanan atamaya yalnız ilgili rolün belgeleri eklenir. Eksik veya boş belge atamayı reddettirir. Geçici uzman, temel rol belgelerine ek olarak görev alanına özgü talimatları alır.

Rol skill’lerinin her biri açık çağrı politikasına sahiptir. Hazır atama ve gerçek model/muhakeme kanıtı olmadan bir rol doğrudan seçilirse eksik kanıtı bildirerek durun. Bu roller AGWF’yi veya izlemeyi etkinleştirmez. Devre dışı bırakmak için yedi skill kaydını kapatıp yalnız sahipli izleyicileri durdurun. Paket güncellemesi bunları açmaz.

Rol belleği kalıcı başvuru malzemesidir. Yetki veya güncel proje kanıtı vermez. Geçmişi uydurmayın. Bellek ekleme önerilerini ana ajana gönderin. Ana ajan değişiklikleri gözden geçirip çalışmalar arasında belgeyi günceller. Rol dosyaları manifest kapsamındadır. Bellek güncellemesinden sonra yeni paket ve yeni run bağı kurulur. Canlı iş sırasında kurulu skill’e dokunmayın.

<a name="integrated-limit-watcher"></a>

## Entegre limit izleyicisi

Yetkili `workflow activate` veya `bind-active-goal` çalıştığında candidate model kullanmayan izleyiciyi başlatır. Önce mevcut Codex bağlantısından gerçek bir `mcp__codex_app__get_usage_limits` kaydı alın ve `--usage-input` ile verin. [Yerleşik kullanım sözleşmesini](references/native-usage.md) izleyin. İzleyici ayrı app-server başlatmaz, PATH’ten CLI seçmez ve başka taşıma kanalına bağlanmaz. Candidate etkin workflow durumunu yazmadan önce sağlıklı ve taze bir örneklem arar. Eksik ve süresi dolmuş kayıt başlangıcı reddettirir. `resume` taze ölçüm ve yeni epoch ister. Kayıtlı `usage_receipt_expired` duruşunda aşağıdaki teknik yolu kullanın. Diğer nedenler doğrudan devam veya kabul edilmiş hedef geçişi gerektirir.

Ana ajan, yetkili iş sırasında ölçümü 300/120/60/30 saniyelik uyarlanan son tarihe göre yeniler. Python süreci kendisine verilen kayıtları denetler. Taze veri yoksa yeni iş kapanır ve süreç host aracını kendisi çağıramaz. Aynı dosyayı tekrar okumak ölçüm zamanını değiştirmez. Etkin komutlar 600 saniyelik lease’i yeniler. Sağlıksız izlemeye rağmen yeni iş kabul edilmez. Kurtarma işlemleri açık kalır. `complete` ve `halt` sahipli süreci durdurup çıkışı doğrular. Kalp atışı yalnız run ve epoch sahipliğini kontrol eder. Tam bağımlılık hash kontrolü komut ve ölçüm sınırında yapılır. İş sınırlarında `workflow status` kullanın. Sohbetin sessiz kalması ölçüm veya lease’i yenilemez. Codex otomatik başlatma kancası kurmaz. Manuel `watcher start/ingest/status/renew/stop` aynı ölçüm ve yetki kurallarına tabidir.

Kullanım hakkı %25’e düştüğünde tamamlanmış iş için doğrulanmış checkpoint hazırlayın. %20 veya erken durma sinyalinde yeni işi durdurun, etkin yazıcılardan kısa durum bilgisi alın, gerekli dosyaları doğrulayın ve yalnız sahipli süreçleri durdurun. Ölçüm yoksa kalan hak bilinmiyor. Bir kez tekrar deneyin, sonra kaydedip durun. `SAVE_AND_STOP` geldiğinde ana ajan bir alt ajanı beklemeden koruma adımlarına geçer. Eskimiş ölçüm teknik kesintidir. Kalan hakkın bittiği veya yetkinin geri alındığı sonucunu vermez. Kaydedip durdurduktan sonra `usage_receipt_expired`, aynı yetkili ve etkin yerleşik hedefle `resume` ve `usage_recovery: true` kullanarak toparlanabilir. Taze gerçek ölçüm, aynı hedefin taze gözlemi ve envanteri, değişmemiş doğrulanmış checkpoint ve durmuş sahipli süreçler gerekir. Candidate duruş nedenini `halt` sırasında gerçek izleyici kaydından alır. Kota, erken rezerv, başka hata, açık kullanıcı duruşu ve etkin olmayan hedef bu yoldan geçmez. Kanıt varsa bu kurtarmayı uygulayın. Sırf telemetri eskidi diye kullanıcıdan yeni bir devam metni veya duraklatma gözlemi istemeyin. Yerleşik hedefi bu nedenle `blocked` yapmayın. Durum aracının kendi ölçütlerini uygulayın. Kurtarma yardımcıyı ve izlemeyi açar. Hedef durumunu değiştirmez, kullanıcı sözü veya arayüz olayı üretmez. Diğer duruş nedenleri için ilgili devam kanıtı gerekir. Otomatik kredi, reset veya hedef durumu değişikliği yapmayın. Haftalık pencere istisnası gerçek kullanıcı sözü ve sınırlı sona erme zamanı gerektirir. İki pencerenin ölçümü korunur. Yalnız karar belirsizken `limit_watcher` rolünü tam `gpt-6.1-terra`/`medium` ile kullanın. Rol kullanılamıyorsa bunu bildirin. [İşletim](references/operations.md) ve [izleyici rolü](private/limit-watcher/limit_watcher.md).

Sınırlı güncel durum devri için [bağlam ve kabul sözleşmesini](references/context.md) kullanın. Özgün yetki zamanlarını, pin’li kanıtı ve bağımsız incelemenin karşı kanıtını koruyun. Özet, görev gönderme yetkisi veya cache kazancı vermez. Yerleşik yürütme bitişi ve bağımsız çıktı kabulü ayrı sürümlü gözlemlerdir. Önemli bağımsız incelemeden önce model kullanmayan üretici kontrolünü çalıştırın.

Alt ajana hedefi, gerekli girdileri, izinli yazma yollarını ve kabul ölçütlerini verin. Ana sohbeti kopyalamadan yeni bağlamla başlayın. Uygulayıcı oturumunu frontend, backend, editor veya data_analysis gibi gerçek işine göre adlandırın. Uygun mevcut proje skill’lerini kullanın. Özel işte ilgili temel rolden geçici uzman oluşturun. Adını, alan yönergesini, kaynaklarını, araçlarını ve kabul ölçütlerini belirtin. Bunun için önceden tanımlanmış ayrı rol veya ayrı kullanıcı çağrısı gerekmez.

Araştırmacı kanıtı toplar ve birleştirir. Uygulayıcı tasarlar, geliştirir ve düzeltir. Doğrulayıcı önemli çıktıları bağımsız sınar. Mimar yalnız maliyetli karar veya teknik uyuşmazlık için gerekir. Tekrarlanan mekanik işi betiğe verin. Uygun betik varsa yeniden kullanın, yoksa tekrar veya güvenilirlik ihtiyacı gerekçelendiriyorsa yerel betiği hazırlayıp sınayın. Karar göreve bağlıdır. Boşta çalışan zamanlayıcı veya sürekli Admin ve Executor katmanı yoktur.

Her dosyanın tek yazıcısı olsun. Ortak arayüzler değiştirilmeden önce uzmanlar anlaşsın. Güvenlik, kimlik doğrulama, olası veri kaybı, ortak API ve geniş davranış değişikliklerini bağımsız doğrulayın. Ana ajan dar ve geri alınabilir işi kendisi doğrulayabilir. Ham log yerine sonucu, değişen yolları ve kanıtı verin.

Kullanıcının güncel yetkisine ve proje sınırlamalarına uyun. Duruş halinde açık devam gelene kadar yeni görev göndermeyin. Güncel iş ve kalıcı kararları koruyun. Kendi geçici dosyalarınızı kurtarma bağımlılıkları kapandıktan sonra kaldırın. Keyfî mesaj uzunluğu veya toplantı koşulu koymayın.

Yardımcı kendi içinden geçen işlemleri denetler. Yerleşik araç çağrılarını kesmez ve süreçleri kendiliğinden durdurmaz. Ana ajan gerçek çağrıyı yürütür, sonucu kaydeder ve canlı envanteri inceler. Gözlem metadatası olmadan istenen modeli fiilen kullanılan model olarak bildirmeyin.

<a name="coordination"></a>

## Koordinasyon

Birbirine bağlı uzmanlıklar gerektiren büyük işleri göndermeden önce `coordinator` rolünden bölümlendirme, bağımlılıklar, rol, dosya sahipliği ve kabul önerisi alın. Kapsam belirgin değişirse, uzman sonuçları anlaşmazsa veya tekrarlı hata yeni plan gerektirirse yine danışın. Küçük veya bağımsız işler ve rutin takip ana ajanda kalır.

Hedefi, güncel sınırlamaları, bulguları ve ilgili kaynakları yeni bağlamla verin. Koordinatör salt okunur plan döndürür. Ajan başlatmaz veya workflow state’e yazmaz. Ana ajan kullanıcı kapsamıyla uygunluğunu ve yapılabilirliği kontrol eder, kabul edilen görevleri kaydedip planı uygular. Plana bağlı görevleri kabul edilen koordinasyon görevine `depends_on` ile bağlayın. Küçük kapasite ve dosya düzenlemeleri ana ajanda kalır. Yalnız önemli strateji değişikliğinde yeniden danışın. Teknik tasarım mimarın, iş düzeni koordinatörün alanıdır. Biten oturumu beklemeyin. Gerekiyorsa yeni ve sınırlı görev açın. Tam model veya muhakeme düzeyi yoksa eksikliği bildirip plan bağımlı işi durdurun. Bağımsız yetkili iş sürebilir. Kurtarmayı planlama ajanına bağlamayın.

<a name="user-model-policy-27-october-2026"></a>

## Kullanıcının model politikası, 27 Ekim 2026

Görev rollerinde yalnız `gpt-6.1-sol`, `gpt-6-astra` veya `gpt-6.1-terra` kullanılabilir. Muhakeme düzeyi en az `medium` olmalıdır. `none`, `minimal` ve `low` kullanılmaz. Rol dosyasındaki pin’li model ve düzeyi başka ayarla değiştirmeyin. Emekli `astra_coordinator` rolünü geri getirmeyin. Entegre `limit_watcher` yalnız `gpt-6.1-terra`/`medium` kullanır. Ana istemci modeli ve muhakeme düzeyi kullanıcının tercihidir.
