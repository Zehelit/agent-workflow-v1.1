<a name="install-verify-and-maintain-agwf"></a>

# AGWF kurulumu, doğrulaması ve bakımı

Python 3.11 veya üstü ve Git gerekir. Paket üretimi ve candidate yalnızca Python standart kütüphanesini kullanır. Kurulumu ilgili kullanıcının yazabildiği skill dizinine yapın. Komutlar PowerShell içindir. Linux veya macOS’ta aynı Codex kökü altındaki karşılık gelen dizini seçip kabuk sözdizimini uyarlayın. Windows ACL kontrolleri bu platforma özgüdür.

<a name="build-from-the-selected-checkout"></a>

## Seçtiğiniz çalışma kopyasından paket hazırlayın

AGWF deposunun kökünde çalıştırın. Paketi kabul etmeden önce kaynak commit’ini kaydedin ve yerel değişiklikleri inceleyin. Yeni bir çıktı dizini kullanın.

```powershell
python --version
git rev-parse HEAD
git status --short
$build = Join-Path $env:TEMP ('agwf-build-' + [guid]::NewGuid().ToString('N'))
python -X utf8 -B tooling/build-candidate.py --output $build
if ($LASTEXITCODE -ne 0) { throw 'AGWF build failed.' }
$receipt = Get-Content -LiteralPath (Join-Path $build 'build-receipt.json') -Raw | ConvertFrom-Json
$pin = $receipt.manifest_sha256
$package = Join-Path $build 'agent-workflow-v1-1'
$entry = Join-Path $package 'scripts/candidate.py'
$manifest = Join-Path $package 'manifest.json'
```

Paketleyici bir paket dizini, aynı girdiden yeniden üretilebilen bir ZIP ve build kaydı oluşturur. Kabul edilen kaydı kaynak commit’iyle birlikte, değiştirilebilen paketin dışında saklayın. Kurucu kabul edilen `manifest_sha256` değerini bekler. Beklenmeyen bir düzenleme sonrası yeniden hesaplanan hash yeni bir gözlemdir. [Paket bağı ve kurulum](../skills/agent-workflow-v1-1/references/coexistence.md).

<a name="choose-a-human-or-agent-path"></a>

## Manuel veya ajanla kurulum

Mevcut kullanıcının Codex skill dizini için aşağıdaki komutu kullanın. Geçici deneme için hedefi `agent-workflow-v1-1` ile biten izole bir dizine ayarlayın. İlk kurulum ile aynı paketin değişiklik yapmadan doğrulanması aynı komutu kullanır.

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false`, işlemin yalnızca paket dosyalarını kurduğu veya doğruladığı, iş akışının başlamadığı anlamına gelir. Windows’ta hazırlık dizini hedefin üst dizininden erişim haklarını devralır. Kurulu dosyaların okunabildiğini ve çalıştırılabildiğini ilgili normal kullanıcı oturumunda doğrulayın.

Kurulumu bir ajan yapacaksa:

1. Hedefi ve işlemin ilk kurulum, güncelleme veya izole deneme olup olmadığını belirleyin. Mevcut bir hedefi değiştirmeden önce kurulu paketi, kullanıcı değişikliklerini ve sahipli süreçleri inceleyin.
2. Çalışma kopyasının kimliğini ve Python sürümünü kontrol edin. Paketi yeni bir dizine üretip kabul edilen harici manifest pin’ini saklayın.
3. Belirtilen hedefe pin’li candidate ile kurun. Güncellemeden önce yalnız ilgili sahipli eski süreçleri durdurup çıkışlarını doğrulayın. Kurucu işlemi reddederse mevcut hedefi incelemek için koruyun.
4. Kurulu manifesti ve paket içeriğini doğrulayın. Aşağıdaki pasif kontrolü kurulu girişten çalıştırın. Kaynak commit’ini, manifest hash’ini, hedefi, sonucu ve işlemin pasif kontrol mü yoksa açıkça başlatılmış canlı iş akışı mı olduğunu bildirin.

<a name="verify-the-installed-entrypoint"></a>

## Kurulu girişi doğrulayın

Yeni bir geçici proje ve çalışma kimliği kullanın. Bu komut geçici projede bir çalışma bağı yazar. İzleyici başlatmaz ve hedefi etkinleştirmez.

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

Yanıttaki `identity`, `version`, `run_id`, `manifest_sha256` ve `missing_dependencies` alanlarını kontrol edin. Bu belge için yapılan izole kontrolde kurulum `activated:false` döndürdü. Doğrulama `run-binding.json` yazdı. İş akışı durum dosyası veya izleyici klasörü oluşmadı. Yanlış manifest hash’i 2 çıkış koduyla reddedildi.

Gerçek Codex çağrısı ayrı bir işlemdir. Açık oturumun skill kataloğunu yenilemek gerekebilir. `$agent-workflow-v1-1` çağrısını açıkça yapın ve işin kapsamını belirtin. Yetkili başlangıç ve izleyici sağlığı için [işletim rehberine](../skills/agent-workflow-v1-1/references/operations.md) bakın.

<a name="update-recover-or-remove"></a>

## Güncelleme, kurtarma ve kaldırma

- Sahipliği doğrulanan kurulumu güncellemeden önce ilgili süreci mevcut pin’li girişten durdurup çıkışı doğrulayın. Yeni paketi ayrı üretip kabul edin. Yalnızca sahipliği doğrulanmış ve değişmemiş kurulum için komuta `--update` ekleyin. Kullanıcı değişikliklerini önce uzlaştırın.
- Kurucu, dizini değiştirmeden önce hazırlanan paketi doğrular. Yerine alma başarısız olursa önceki dizini geri getirmeyi dener. Geri yükleme de başarısız olursa işlem dizinini koruyun. Başarılı güncellemeden sonra kalıcı geri alma arşivi tutulmaz. [Kurucu ayrıntıları](../skills/agent-workflow-v1-1/references/coexistence.md).
- Teknik kimlik değiştiğinde yeni dizine temiz kurulum yapın. Eski çalışma bağlarını özgün manifestleriyle saklayın ve Codex skill keşif ayarını ayrı denetleyin. Kabul edilen paket güncellemesiyle yeni çalışma kimliği kullanın. [Geçiş sınırları](../skills/agent-workflow-v1-1/references/migration.md).
- Kaldırma komutu yoktur. Kaldırmadan önce tam skill yolunu doğrulayın, sahipli süreçleri durdurun ve keşif dışında doğrulanmış yedek tutun. Kullanıcı projelerini, çalışma durumunu, checkpoint dosyalarını ve harici kayıtları koruyun. İşlemden sonra skill keşfini tekrar kontrol edin.
