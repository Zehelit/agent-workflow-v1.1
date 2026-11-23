<a name="agwf-maintenance-on-28-october-2026"></a>

# AGWF bakımı, 28 Ekim 2026

> Tarihsel bakım kaydı. Aşağıdaki yetki ve gözlemler o çalışmaya aittir. Güncel davranış için [README](../README.md).

Altı rolün her biri için ayrı `agents/openai.yaml` ve `allow_implicit_invocation: false` eklendi. Rol açıklamaları hazırlanmış görevlere bağlı. Atama ve fiilî model/muhakeme kanıtı yoksa rol durur. Rol skill’leri workflow veya watcher başlatmaz. Ana skill’den eski proje seçiminin hâlâ geçerli olduğunu ileri süren metin kaldırıldı.

Windows durum dosyalarında paylaşımlı okuma, izinleri koruyan dosya değiştirme ve sınırlı tekrar kaynak ve kurulumda eşitlendi. `candidate.py` değişen durum için aynı okuyucuyu kullanır. Dış okuyucu dosyayı kilitlerse eski dosya korunur. Kalıcı kilit hatadır.

Bağlam ve kullanım araçları, kurulum çalışma kopyasına bağlı kalmasın diye paketlenmiş `tools/` dizinine taşındı. O kontrolde kaynak, paket ve kurulu 41 dosya eşleşti. Bağlam aracının eski kök kopyası kaldırıldı. Test yolları güncellendi.

Kayıtta 63 paket/araç ve dokuz Windows dosya erişim testi geçmiştir. Dokuz Windows testi `-O` ile ve kurulu dosyalar üzerinden tekrar sınandı. Yedi skill tanımı ve dosya hash’i denetlendi. Altı eksik rol politikasının paket testi düzeltme öncesi başarısız, sonrasında başarılıydı.

AGWF bakım sırasında devre dışıydı. Yedi skill ayarı kapalı kaldı. Testler geçici dosyalar üzerinden çekirdek erişimini sınadı. Canlı Remote uyumu kapsamda değildi.

Her ölçümde ikinci Codex sunucusu açmak, PATH’ten başka CLI seçmek, alt sunucunun stderr verisini atmak ve tekrarlı paket taraması o sırada kalan risklerdi. Bu aşama Remote sorununun nedenini doğrulamadı. [Sonraki bakım](agwf-tum-duzeltmeler-20261028.md) sonraki düzeltmeleri kaydeder. Güncel izleyici ikinci sunucu açmaz.
