<a name="changelog"></a>

# Değişiklik geçmişi

Bu, mevcut sürümlerin geçmişidir. Paketin tam kimliği kaynak çalışma kopyası ve kabul edilen build kaydıyla belirlenir.

<a name="11--2026-11-10"></a>

## 1.1 — 10 Kasım 2026

<a name="added"></a>

### Eklenenler

- İlk activate veya bind-active-goal denemesi kalıcı olarak kaydedilir. Güvenli tekrar için hata kaydının dışarıda saklanan SHA-256 değeri kullanılır.
- Candidate yaşam döngüsü işlemleri sahipli tek yazıcı kilidi kullanır.
- tooling/run-lifecycle-check.py hazırlık, başlangıç, gözlem yenileme, halt sonrası devam ve normal kapanış için eklendi.
- Açık timed-test modu SAVE, checkpoint ve halt zamanlarını ayrı kaydeder ve geç olayı sınıflandırır. Yerleşik pause kanıtı üretmez.
- Durum özeti gerçek ölçüm zamanını, son tarihi, aralığı ve erken yenileme önerisini gösterir.
- Amaç, mimari, workflow ve orkestrasyon şemaları, insan ve ajan kurulumu ve yaşam döngüsü rehberi eklendi.

<a name="fixed"></a>

### Düzeltmeler

- Eskimiş kullanım kaydı, aynı yetkili etkin hedefte kontrollü toparlanabilir. Kota ve kullanıcı duruşu bu yolu kullanmaz.
- İlk başlatma yeniden denenirken kapsam, geçmiş SAVE, paket bağı ve doğrulanmış süreç yokluğu korunur.
- Süreli haftalık istisna hem çekirdekte hem izleyicide aynı karara yol açar. Gözlenen ölçümler değiştirilmez.
- Windows’ta belirlenmiş geçici paylaşım ve okuma kilidi hataları sınırlı sayıda yeniden denenir. Kalıcı ve sınıflandırılamayan hatalar görünür kalır.
- Ölçüm ve lease yenilemesi ile halt sonrası tazelik koşulları açıklandı.
- Eski test yardımcılarının çağrı imzaları ve süreç/dosya beklemeleri güncellendi.

<a name="compatibility-and-evidence"></a>

### Uyumluluk ve kanıt kapsamı

- Çağrı, kurulum dizini ve yeni çalışma alanı birlikte 1.1’e geçti. `$agent-workflow-v1-1`, `agent-workflow-v1-1/` ve `.agent-workflow-v1-1/runs/`.
- Yeni manifestler önceki kimliği reddeder. Eski çalışmalar tarihsel kanıt olarak kalır ve yeni pakete otomatik bağlanmaz. Eski çalışma alanları sahiplik ve geri yükleme işlemlerinde ayrılmıştır.
- Güncelleme eski run’ın değişmez bağını yeniden yazmaz. Yeni paketi kabul ettikten sonra yeni run açın.
- Python 3.11+ ve standart kütüphane yeterlidir. İzleyici yerleşik kullanım aracına kendi başına erişmez ve ikinci Codex sunucusu başlatmaz.
- İzole testlerde sentetik host gözlemleri ve gerçek Python süreçleri kullanılır. Canlı Remote, Continue ve arayüz zamanlaması ayrı kanıt ister.
- Sürüm kaynak commit’leriyle kaydedildi. GitHub Release ve tag önceki yayın kapsamının dışındaydı.

<a name="10--previous-source-version"></a>

## 1.0 — önceki kaynak sürümü

- Manifest pin’li candidate girişi, tipli çalışma durumu, rol belgeleri, dosya sahipliği ve kapasite denetimleri.
- Modelsiz entegre izleyici, checkpoint ve geri yükleme, kalıcı koordinasyon günlüğü ve yeniden üretilebilir paketleme.
- Bu geçmiş kaydı, karşılık gelen GitHub Release veya tag’in varlığını iddia etmez.
