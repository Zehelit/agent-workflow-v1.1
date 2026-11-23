<a name="pinned-runtime-binding-and-installation"></a>

# Sabitlenmiş çalışma bağı ve kurulum

Teknik kimlik `agent-workflow-v1-1`, görünen ad `agent-workflow-v1.1`. Dolaylı çağrı kapalıdır. Candidate, yalnız güncel açık seçimle kullanılır. Ana istemcinin modeli kullanıcıya aittir. Görev rollerinde yalnız `gpt-6.1-sol`, `gpt-6-astra` veya `gpt-6.1-terra` ve en az `medium` muhakeme kullanılır. Özel izleyici rolü Terra/medium’e sabittir. Model ve ayar kendiliğinden değişmez.

Güncel açık seçimde ana ajan, seçilen kapsamı aynı tur içinde başlatıp doğrular. Bu, daha önce kapatılmış run’ı o kapsam için yeniden açabilir. Eski seçim veya paket bakımı başlangıç sayılmaz. Tam etkinleştirme kanıtı yoksa veya yerleşik hedef oluşturmak yasaksa [işletim rehberindeki](operations.md) manuel izleyiciyi başlatın. Başlangıç doğrulanmadan yeni işe geçmeyin. Sonraki duruş talimatı geçerliliğini korur.

Python 3.11+ ve standart kütüphaneyle paket üretin.

```powershell
python -X utf8 -B tooling/build-candidate.py --output C:/staging/build-one
```

Çıktıda `agent-workflow-v1-1/`, aynı girdiden yeniden üretilebilen ZIP ve `build-receipt.json` vardır. Kayıttaki `manifest_sha256` değerini değiştirilebilir paketin dışında, kabul edilen iş devri kanıtında saklayın. Çağıranın verdiği bu pin güven dayanağıdır. Run bağlama, önemli iş akışı yazımları, koordinasyon adımları ve her ölçüm turundan önce manifest ile bağımlılıklar doğrulanır. Kalp atışı küçük run/epoch kontrolünü doğrular ve her saniye paketi yeniden hash’lemez. Değişen manifestin hash’ini yeniden hesaplamak harici kabul kanıtı değildir. Python bayt bütünlüğü ve kimlik tutarlılığını denetler. Kullanıcının onayı, sağlayıcı model metadatası veya çağıranın kontrol ettiği dosya sistemi bağımsız olarak doğrulanmaz. Değiştirilmiş yorumlayıcı veya hem bağı hem de onaylı harici pin’i değiştirebilen bir çağıran bu sözleşmenin dışındadır.

Küresel seçenekleri moddan önce yazın. Kaynak çalıştırma için `skills/agent-workflow-v1-1/scripts/candidate.py` ve build manifesti kullanılır. Kurulu çalıştırmada kurulu `scripts/candidate.py`, manifest ve aynı korunmuş harici pin kullanılır. Kaynak ve kurulu eşlemelerine ayrı run kimliği verilir. Değişmez bağ çözümlenen bağımlılık ve manifest yollarını, projeyi, run’ı, sürümü ve hash’leri içerir. Başka yardımcı eşlemesi, değiştirilmiş paket veya bağımlılık ya da farklı pin’le tekrar kullanım reddedilir. Onaylı güncellemeden sonra yeni run açın.

```powershell
$pin = '<preserved manifest_sha256 from accepted build receipt>'
$entry = 'C:/staging/build-one/agent-workflow-v1-1/scripts/candidate.py'
$manifest = 'C:/staging/build-one/agent-workflow-v1-1/manifest.json'
$common = @('-X','utf8','-B',$entry,'--manifest',$manifest,'--manifest-sha256',$pin,'--version','1.1','--project','C:/project','--run-id','candidate-run-1')
python @common workflow validate
python @common workflow status
python @common coordination --input C:/project/batch.json
```

Yetkili `workflow activate` veya `bind-active-goal` izleyiciyi başlatır ve `--usage-input` ile verilen ilk taze örneklemi sağlıklı olarak doğrular. `resume` geçerli devam ya da mevcut etkin hedef yetkisiyle doğrulanmış ölçüm eskimesi kurtarmasından sonra taze yerleşik kullanım kaydıyla yeni epoch açar. [İşletim](operations.md) ve [yerleşik kullanım rehberine](native-usage.md) bakın. Ölçüm için ayrı Codex sunucusu veya CLI çalışmaz. Etkin komutlar lease’i yeniler. `complete`/`halt` yalnız sahipli Python sürecini durdurur ve çıkışını doğrular. Açık `watcher` komutları aynı kayıt sözleşmesine tabidir.

`workflow` tipli çekirdek durum komutlarını ve `validate` komutunu kabul eder. JSON `--input` veya stdin’den okunur. Eski `native/install/export/migrate` çekirdek arayüzleri bu candidate’ta desteklenmez. `activate` yetkisi ve `bind-active-goal` kanıtı seçilen run kimliğini taşır. `validate` rol ve bağımlılıkları denetler ve run alanını bağlar, hedefi etkinleştirmez. Batch örneği `{ "batch_id": "inspect", "steps": [{ "key": "status", "command": "status", "payload": {} }] }` biçimindedir. En fazla 64 adım, tek ana yazıcı ve kalıcı bekleyen sonuç kaydı vardır. Otomatik tekrar veya yerleşik ajan gönderimi yoktur. Önemli komutlar güncel tipli yetki ve kabul kanıtı ister. Terminal envanter çıktı kabulü değildir. Yazıcısı durmuş eskimiş iş `finish accepted:false` ile bırakılabilir. Canlı yazıcı varsa bırakılamaz. Sıkıştırma doğrulanmış checkpoint gerektirir ve görev ile geçersizleşme geçmişini arşivler.

İzole kök `<project>/.agent-workflow-v1-1/runs/<run-id>/` yoludur. Çalışma durumu dizinleri görev yazma alanı olamaz. Entegre izleyici burada `watcher/` dizinini, pin’li zamanlama yardımcısını, değişmez sahipliği ve candidate yeniden başlatma argümanlarını kullanır. Başka run kontrolü veya doğrudan özel CLI başka bir sürecin lease’ini yenileyemez veya onu durduramaz. Sessiz sohbet lease’i yenilemez. İş sınırında etkin candidate komutu çalıştırın. Paket güncellemesi çalışan süreçleri taşımaz. Ayrı watcher skill’i veya başlatıcısı yoktur.

İlk kurulum, aynı paketin değişikliksiz doğrulanması ve sahipli güncelleme için hedef `agent-workflow-v1-1` ile bitmelidir.

```powershell
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1 --update
```

Güncelleme sahipliği doğrulanmış, değişmemiş dosyalar ister. Kurucu hazırlanan paketi dizini değiştirmeden doğrular. Yerine alma başarısız olursa önceki dizini geri yükler, başarıda işlem dosyalarını kaldırır. Kalıcı geri alma arşivi ve geri alma komutu yoktur. Geri yükleme de başarısızsa işlem dizini önceki tek kopyayı saklar. Kurtarma doğrulanana kadar bu dizini koruyun. Bu komutlar sadece dosyaları hazırlar ve `activated:false` döndürür. Katalog, açık yerleşik çağrı, sağlayıcıdan gözlenen fiilî model ve gerçek çalışma kabulü ayrı kurulum sonrası kontrollerdir.

Windows’ta kurucu, paket dosyalarını kopyalamadan geçici hazırlık kökünün ACL’sini hedef üst dizininden izin devralacak şekilde sıfırlar. Belirli kullanıcı adı içermez. Yeni veya adı değişmiş hesap, kendi `skills` üst dizinindeki izinleri alır. Üst dizin ilgili kullanıcıya okuma ve çalıştırma izni vermelidir. Kurulumdan sonra keşfi ve doğrudan candidate çalıştırmayı normal, yükseltilmemiş Codex oturumundan doğrulayın.
