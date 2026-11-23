<a name="native-usage-receipts"></a>

# Yerleşik kullanım kayıtları

Bu yolu yalnız açıkça yetkilendirilmiş ve etkin bir AGWF çalışmasında kullanın. Güncel ve açık skill çağrısı, o kapsam için başlangıç gerektirir. Yerleşik hedef oluşturmaya izin yoksa [işletim rehberindeki](operations.md) manuel izlemeyi kullanın. Paket güncellemek, skill’i tartışmak veya kullanım verisini yayımlamak devre dışı iş akışını yeniden açmaz.

Ana ajan mevcut Codex bağlantısından `mcp__codex_app__get_usage_limits` aracını çağırır. Python ikinci bir app-server açmaz, farklı CLI seçmez, kimlik doğrulama dosyalarını okumaz, özel veri yoluna bağlanmaz ve model çağrısı yapmaz. Arka plan süreci verilen kayıtların tazeliğini izler. İş sınırlarında host aracını yine ana ajan çağırır.

Gerçek araç yanıtından özel bir JSON girdi dosyası oluşturun.

```json
{
  "source": "mcp__codex_app__get_usage_limits",
  "sourceId": "actual-tool-call-receipt-id",
  "observedAt": 1800000000,
  "result": {
    "rateLimits": {
      "limitId": "codex",
      "primary": {"usedPercent": 25, "windowDurationMins": 300, "resetsAt": 1800010000},
      "secondary": {"usedPercent": 30, "windowDurationMins": 10080, "resetsAt": 1800600000}
    }
  }
}
```

Sayılar örnektir. Gerçek gözlem zamanını, kayıt kimliğini ve değerleri koruyun. Bunları üretmeyin ve gözlem zamanını dosyanın yazıldığı zamanla değiştirmeyin. Gerçek araç sonucu birden fazla grup içeriyorsa `rateLimitsByLimitId.codex` kullanın. Birden çok grup olup Codex yoksa işlem reddedilir. İki pencerenin süreleri ve reset zamanları gerekir. Girdiye yalnız gereken kullanım alanlarını alın. Hesap, profil ve kredi alanları gerekmez. Yayımlanan yerel `watcher/usage-source.json` yalnız izinli pencere alanlarını, gözlem ve kaynak kimliğini, bağlı run ve epoch’u içerir.

`workflow activate --input AUTH.json --usage-input USAGE.json` ve `bind-active-goal`, izleyici başlamadan önce yetkiyi doğrular. `resume` da `--usage-input` ile yeni kayıt ister. Eski epoch kaydı yeniden kullanılamaz. Manuel `watcher start --usage-input USAGE.json` aynı giriş kontrolünden geçer. Eksik veya eskimiş veri varsa süreç başlatılmaz.

İş sırasında taze ölçümü `watcher ingest --usage-input USAGE.json` veya workflow ya da koordinasyon komutundaki `--usage-input` ile yayımlayın. `ingest` iş akışını etkinleştirmez, süreç başlatmaz, yetkiyi yenilemez ve SAVE kaydını kaldırmaz. Yeni kaynak dosyası, önceki süre sonunu beklemeden değerlendirmeyi tetikler. İzleyici dosyayı işlerken status kısa süre önceki değerlendirmeyi gösterebilir. Önemli bir görev göndermeden önce taze ve sağlıklı durum bekleyin.

Ana ajan ölçümü uyarlanan son tarihten önce yeniler. Aralık en çok 300 saniyedir, eşiğe yakınken veya hızla tüketilirken kısalır. Aynı dosyayı okumak yeni ölçüm değildir. Taze ölçüm gelmezse izleyici bir kez daha doğrular ve SAVE_AND_STOP kaydeder. Kayıtlı `usage_receipt_expired` aynı yetkili etkin hedefte `workflow resume` ve `usage_recovery: true` ile toparlanabilir. [Teknik kurtarma](operations.md#expired-telemetry-recovery-under-existing-authority) koşullarını uygulayın. Taze veri tek başına SAVE kaydını kaldırmaz. Bu teknik yol yeni yazılı devam gerektirmez. Gerçek kota ve açık kullanıcı duruşları ayrı devam koşullarına tabidir. İş sınırında, sonraki araç çağrısına zaman kalacak şekilde yenileyin. Son tarih bekleme önerisi değildir. Uzun ve kesilemeyen çağrılar yenilemeye engel olabilir. Arka plan süreci yerleşik aracı yalnız başına çağıramaz.

Bozuk girdi, başka run veya epoch, gelecekteki ya da eskimiş ölçüm, geriye giden zaman veya içeriği değişmiş tekrar reddedilir. Aynı kayıt tekrar yayımlanırsa durum değişmez. Zaman ve kaynak bilgisi çağıranın sağladığı kanıttır. JSON biçimi araç sağlayıcısının kimliğini doğrulamaz. Ham kanıtı özel saklayın. Bir kaynak etiketi tek başına bağımsız kanıt değildir.

Ölçüm hatalarında sample ve status, izinli `diagnosticCode` ve hata sınıfını verir. `watcher/diagnostics.json` en çok 20 yapılandırılmış kayıt tutar. İstisna metni, kimlik doğrulama verisi ve tam araç içeriği dışarıda kalır. Beklenmeyen Python süreç hataları için yerel `worker-stderr.log` vardır. İkinci sunucu açılmadığı için onun stderr çıktısı oluşmaz.

Paket bütünlüğü her candidate komutunda ve yeni veriler dahil her izleyici değerlendirmesinde doğrulanır. Kalp atışı yalnız run ve epoch sahipliğini denetler ve sağlık durumu yazar. Paketin bütün dosyalarını tekrar taramaz. Değişen bağımlılık sonraki ölçüm veya komut sınırında reddedilir, her kalp atışında tespit sözü verilmez.

Birim testleri gerçek dosyada kayıt yazmayı ve okumayı, katı JSON’u, eskimeyi, tekrarı, kimliği, eşikleri, sınırlı tanı verisini ve ölçüm sırasında yeni süreç oluşturulmadığını sınar. Canlı Remote kabulü ayrı kanıt ister. Kullanıcı açıkça yeniden etkinleştirene dek kurulu workflow kapalı kalır. Bakım işi, kabulü göstermek amacıyla canlı AGWF süreci başlatmaz.
