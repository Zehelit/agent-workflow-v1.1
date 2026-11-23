<a name="agwf-monitoring-follow-up-on-28-october-2026"></a>

# AGWF izleme bakımı, 28 Ekim 2026

> Tarihsel kayıt. [İlk bakımın](agwf-duzeltmeler-20261028.md) devamını anlatır. Güncel etkinleştirme yetkisi vermez.

| Bulgu | O sırada kaydedilen değişiklik ve kanıt |
| --- | --- |
| Altı rol otomatik seçilebiliyordu | Her role `allow_implicit_invocation: false` ve hazırlanmış görev sınırı eklendi. Paket testi yedi skill’i denetledi. |
| Kaynak, uzak depo ve kurulum farklıydı | Gerekli araçlar pakete taşındı. Kurulum temiz commit’ten üretilen manifest ve ZIP ile karşılaştırıldı. |
| Windows okuma/yazma çakışıyordu | Paylaşımlı okuyucu ve değiştirme düzeltmesi ilgili bileşenlere yayıldı. Geçici dosya testleri kilidi, tekrarı ve eski verinin korunmasını sınadı. |
| Her ölçümde ikinci Codex sunucusu açılıyordu | Bu kod kaldırıldı. Ana ajan mevcut kullanım aracını çağırır, Python ölçüm kaydını okur. Ölçüm testleri süreç açmayı engeller. |
| PATH başka Codex CLI seçebiliyordu | Ölçüm akışından çalıştırılabilir dosya seçimi çıkarıldı. |
| Hata çıktısı kayboluyordu | Alt sunucu kaldırıldı. Ölçüm hataları kod, hata sınıfı ve aşamayla en çok 20 kayıt tutar. Beklenmeyen Python hatalarında yerel stderr korunur. |
| İki döngü paketi her saniye tarıyordu | Kalp atışı run ve epoch denetler. Tam tarama komut ve ölçüm sınırındadır. |
| Testler eski sahte sunucuyu kullanıyordu | Ölçüm kaydı sözleşmesine geçildi. Tarihsel kayıtta gerçek Codex kullanım yanıtı da geçici dosyada sınandı. |
| Rol politikası yanlış eşiği içeriyordu | `%10` ifadesi fiilen kullanılan `%20` eşiğine getirildi. Eksik ölçümle reddedilen start etkin işaretini kaldırır. |

Yeni veri akışı `--usage-input` ile özgün gözlem zamanını ve kaynak kimliğini kullanır. İzleyici kaydı run ve epoch’a bağlar. Dosyayı yeniden okumak yeni ölçüm değildir. Eksik, gelecek, süresi dolmuş, geri giden veya başka run’a ait kayıt reddedilir. Aynı kaydı yazmak zamanı ilerletmez. Yenileme olmazsa bir kez daha doğrulanır ve SAVE_AND_STOP kaydı kalır.

Yeni ölçümü süre dolmadan ana ajan alır. Python host aracını çağıramaz. `ingest` yalnız veri yayımlar. Etkinleştirme, süreç başlatma, yetki, hedef gözlemi ve SAVE kaydını kaldırma ayrı işlemlerdir.

Kayıt, normal ve `-O` modlarında 111 hedefli test bildirir. Testler gerçek AGWF ve Codex başlatmasını engelledi. Paketleyici ve salt okunur veri araçları izin verilen alt süreçleri kullandı. Kaynak testi, kurulu hash, kapalı ayar ve temiz kaynak build ayrı gözlendi.

Gerçek host ölçümü yeni süreç açmadan ve zamanı değiştirilmeden kayıt işlemlerinden geçirildi. Hesap veya kimlik bilgisi pakete alınmadı. Geçmiş Remote sorununun kesin nedeni doğrulanmadı. Canlı AGWF/Remote denemesi kapsam dışındaydı. [Yerleşik ölçüm sözleşmesi](../skills/agent-workflow-v1-1/references/native-usage.md).
