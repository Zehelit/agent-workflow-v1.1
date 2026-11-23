<a name="lifecycle-reliability-specification"></a>

# Yaşam döngüsü güvenilirliği gereksinimleri

> 10 Kasım 2026 tarihli şartname. Kanıt ve yetki geçmiş çalışmaya aittir. Güncel davranış için [işletim sözleşmesi](../../../skills/agent-workflow-v1-1/references/operations.md).

<a name="intended-result"></a>

## Beklenen sonuç

İlk etkinleştirme, kullanım yenileme, toparlanma ve doğal kapanış güvenilir olmalı. Hedef başlamadan test hazırlansın. İlk başlangıç, durmuş workflow’dan devam ve zamanlama ayrı gözlemle raporlansın.

<a name="evidence-available-at-the-time"></a>

## O sırada bulunan kanıt

- Özel zamanlama raporu, etkinleştirme öncesi eskimiş ölçüm kaydetti. 98,74 ve 45,05 saniyelik on saniye denemeleri başarısızdı. İkinci turda taze epoch ve ilk bağlama, durmuş run’dan devamı göstermedi.
- Özel normal akış raporunda sağlıklı başlangıç, 43,423 saniye sonra taze gerçek ölçüm, doğrulanmış checkpoint, normal tamamlanma ve süreç çıkışı gözlendi. SAVE latch sınanmadı.
- O aşamada `scripts/candidate.py`, başarısız `resume` için yeni teknik epoch açıp kontrolleri geri koyabiliyordu. İlk `activate` ve `bind-active-goal` aynı tekrar kaydına sahip değildi. Ayrı regresyon kanıtı gerekiyordu.
- Daha önceki bakım yetki ve yönerge çelişkilerini, dört test yardımcısı ve bekleme hatasını düzeltmişti. Bu davranışlar regresyonla korunacaktı.

<a name="requirements"></a>

## Gereksinimler

1. İlk startup yalnız aynı hedef/run içinde kayda geçmiş `usage_receipt_expired`, yeni gerçek ölçüm, etkin hedefin gerçek gözlemi ve worker’ın bulunmadığı doğrulandığında mevcut yetkiyle yeniden denenebilsin. Yeni alıntı veya run gerekmemeli, ölçüm değiştirilememeli.
2. Kota, erken rezerv, kullanıcı duruşu, etkin olmayan hedef, belirsiz sahiplik, değişen paket veya kanıt ve başka hedef reddedilsin. Python biçim ve tutarlılığı denetler. Çağıranın gösterdiği kişi veya aracı bağımsız doğrulamaz.
3. Başarısız denemeler görünür olsun. Yarım hazırlık etkin görünmesin ve yeni SAVE kanıtı korunsun.
4. Paket ve test gerçek ölçümden önce hazırlansın. Gözlem zamanı değişmesin. Dosya yazımı ölçümü yenilemez. `ingest` ile lease yenileme ayrı kalsın.
5. Normal akış gerçek `CHECKPOINT_SOON` ve `SAVE_AND_STOP` kararlarını uygulasın. Yapay duruş üretmesin.
6. Gerçekten etkin ve ardından durmuş workflow’dan Continue ve toparlanma, ilk bağlamadan ayrı sınansın.
7. Süreli test yalnız seçilmişse çalışsın. Başlatıcı gerçek olaydan önce hazır olsun. Geç gözlem veya son tarihi kaçıran hazırlık geçerli sayılmasın. SAVE istemi için önerilen aralık 10 ± 1 saniyeydi. Gerçek süre yazılsın.
8. Yerleşik hedef aracını host ana ajanı çağırsın. Arka plan süreci için belgelenmemiş host erişimi, özel Codex taşıması veya kimlik bilgisi okuması eklenmesin. Tıklamadan başlayan kesin süre gerçek olay kaynağı gerektirir.
9. Checkpoint, süreç çıkışı, yardımcı complete ve yerleşik hedef complete ayrı doğrulansın. Yalnız kullanıcı yetkisiyle pause ve yerleşik aracın koşullarıyla `blocked` uygulansın.
10. Python 3.11+ ve standart kütüphane korunsun. Windows paylaşım hatası çıkış veya başarı değildir. Yerel değişiklikler, eski çalışma kayıtları ve bağları korunarak güncelleme yapılsın.

<a name="original-authorization"></a>

## Önceki yetki

İlk iş planlamaydı. Kullanıcı daha sonra önceki v1.1 uygulamasını, kaynak commit/push ve belge düzenini yetkilendirdi. GitHub Release ve tag kapsam dışındaydı. Süreli canlı duruş ayrı tanılama seçimi olarak kaldı. Eski run bağları korundu.
