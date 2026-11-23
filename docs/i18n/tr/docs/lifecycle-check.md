<a name="repeatable-lifecycle-check"></a>

# Tekrarlanabilir yaşam döngüsü kontrolü

`tooling/run-lifecycle-check.py`, gerçek hedefi, kullanım ölçümünü ve alt ajan envanterini pin’li candidate üzerinden işler. Gözlemleri host’tan ana ajan alır. Oturum ve giriş dosyalarını Git dışında özel bir yerde tutun. Oturuma yalnız bir yazıcı atayın.

<a name="prepare-before-measuring"></a>

## Ölçümü almadan önce hazırlanın

Paketi üretip kabul edilen harici pin’i koruyun. `prepare` candidate veya izleyici başlatmadan paketi denetleyip yeni oturum kaydı yazar. İlk gerçek ölçümü hazırlık bittikten sonra alın.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256`, kabul edilen build kaydından gelir. Var olan bir proje ve yeni oturum dosyası kullanın. Paket değişirse eski oturumu yeni pin’e bağlamayın.

<a name="host-input-object"></a>

## Host girdi nesnesi

`run`, `observe`, `resume`, `close` ve `timed-test`, `--input` ile bir JSON nesnesi alır.

| Alan | İçerik |
| --- | --- |
| `activation` | Yalnız ilk `run`. `bind-active-goal` için tam tipli candidate girdisi. `native_binding`, eşleşen `app_goal_observation`, hedef, yorum, usul kontrolü, gözlenen model ve muhakeme düzeyleri, slot sınırı, gerçek envanter ve gözlem zamanı. |
| `app_goal_observation` | Gerçek `get_goal` ile hedef ve sohbet kimliği, özgün hedef, oluşturma zamanı ve durum. `source_kind: get_goal`, gerçek kaynak referansı ve `observed_at` gerekir. Varsa gerçek `updated_at` değerini de ekleyin. |
| `usage` | [Yerleşik kullanım kayıtlarındaki](../skills/agent-workflow-v1-1/references/native-usage.md) `source`, `sourceId`, `observedAt`, `result` alanları. Yalnız gereken pencereleri alın. |
| `inventory` | Gerçek güncel alt ajan kimlikleri ve durumları. Alt ajan yoksa `[]`. |
| `checkpoint_files` | Ana ajanın dosyaları dahil kurtarılacak eklenmiş, değişmiş ve silinmiş proje yolları. Çekirdek görevlerin sahiplik yollarını da dahil eder. |
| `writers_quiescent` | Tüm yazıcıların durduğunu gözledikten sonra `true`. Araç gözlemi kendi başına üretemez. |
| `next_action` | Kurtarma veya normal kapanış sonrasındaki anlamlı adım. |
| `weekly_exception` | İlk `run` için isteğe bağlı. Gerçek kullanıcı `exact_quote` değeri ve sınırlı Unix sona erme zamanı `valid_until`. Varsayılan istisna yoktur. |
| `resume` | İsteğe bağlı tam candidate devam girdisi. Yoksa araç yalnız kayıtlı eskime için `usage_recovery: true` dener. Kota ve kullanıcı duruşları kendi koşullarına tabidir. |

Yerleşik API ayrı hedef kimliği vermiyorsa `threadId:createdAt` gibi tutarlı bir türetme kullanın ve bunu türetilmiş olarak belirtin. Ayrıntılı tip sözleşmeleri için [işletim](../skills/agent-workflow-v1-1/references/operations.md) ve [komut rehberini](../skills/agent-workflow-v1-1/references/commands.md) okuyun.

<a name="run-refresh-and-close"></a>

## Çalıştırma, yenileme ve kapanış

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

Yerleşik çağrı döner dönmez gerçek gözlem zamanını kaydedin. Araç `status.at == usage.observedAt`, canlı sahipli süreç, epoch, kalp atışı, taze örneklem ve lease’i ayrı ayrı denetler. `observe` ayrıca lease’i yeniler. `refresh_by` erken yenileme önerisidir ve gerçek son tarihi değiştirmez.

`CHECKPOINT_SOON` geldiğinde bitmiş iş için checkpoint doğrulayın. `checkpoint_files` güncel kalsın. Normal `close` taze izleme, doğrulanmış snapshot, complete ve gözlenmiş süreç çıkışı ister. Normal akış yapay SAVE üretmez.

İlk `run` girdisinde `weekly_exception` varsa araç sınırlı izleyici politikasını kurup `NEED_POLICY_OBSERVATION` döndürür. İş akışı henüz etkin değildir. Gerçek hedefi ve kullanımı yeniden gözleyin. Ardından aynı oturumda tekrar çalıştırın. Önceki kayıt zamanlarını koruyun.

<a name="halt-and-resume-after-expiry"></a>

## Eskime sonrası durdurma ve devam

İzleyici SAVE isterse `observe` stop, snapshot ve halt uygulayıp `NEED_POST_HALT_OBSERVATION` döndürür. Halt sonrasında yeni hedef, kullanım ve envanter gözlemleri alın.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

Çekirdek gözlenen watcher nedenini kullanır. Teknik toparlanma için aynı yetkili etkin hedef, değişmemiş dosyalar ve checkpoint, halt sonrası taze kanıt ve sahipli süreç çıkışı gerekir. Kota, erken rezerv ve açık kullanıcı duruşları kendi devam koşullarına tabidir. Bu araç yerleşik hedefi duraklatmaz, engellemez veya etkinleştirmez.

İlk etkinleştirme başarısız olursa oturum, candidate başlangıç denemesinin hash’ini saklar. Desteklenen yeniden deneme koşullarında yeni kanıtla aynı `run` kullanılabilir. Değişmiş oturum dosyası geçerli kanıt değildir. Tamamlanmış ve belirsiz denemeler otomatik yeniden başlamaz.

<a name="run-a-timed-diagnosis-only-when-selected"></a>

## Süreli tanılama yalnız seçilirse

Olaydan önce sağlıklı izleyiciyi hazırlayın. `timed-test` yalnız açıkça istenen tanılama içindir. Gerçek yerleşik gözlemden gelen olay JSON alanları `event_id`, `goal_id`, `thread_id`, `source_id`, `event_at` ve `observed_at` değerleridir.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

Geç gelen olay `INVALID_TIMING` üretir. `SAVE_REQUEST_ON_TIME` yalnız SAVE isteminin zaman koşulunu geçtiğini gösterir. Yerleşik pause ayrı host kanıtı ister. Bu yüzden araç `native_pause_verified: false` ve `native_paused_at: null` bırakır. İki canlı turda önce etkin durum ve doğrulanmış duruş, sonra aynı hedef ve run üzerinde yetkili gerçek devam gözlenir. Sentetik saat testi yalnız mantığı sınar.
