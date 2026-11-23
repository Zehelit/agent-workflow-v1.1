<a name="state-compatibility-and-recovery-boundaries"></a>

# Durum uyumluluğu ve kurtarma sınırı

v1.1 çalışma zamanı, kabul edilmiş değişmez çalışma alanında güncel schema2 durumu kullanır. Bilinmeyen, bozuk ve eski durumlar mevcut durumu değiştirmeden reddedilir. Eski yetki metinleri, arşiv görev kayıtları ve eskimiş hedef gözlemleri yeni yürütme yetkisi vermez.

Pin’li `scripts/candidate.py` taşıma komutu sunmaz. Çekirdek eski durum dönüştürücüsü veya eski yetki doğrulayıcısı içermez. Başka kurulumdan otomatik içe aktarma, bootstrap ve paket rollback’i desteklenmez. Kabul edilen paket güncellemesinden sonra yeni run açın. Eski run’ın manifesti ve yardımcı eşlemesi korunur.

Kurtarma, kabul edilmiş run’ın doğrulanmış checkpoint’ini ve yeni geçerli kullanıcı yetkisini kullanır. Kaynak zamanını, replay rezervasyonlarını, dosya bütünlüğünü ve yazıcıların durduğuna dair gözlemi koruyun. Eski duruş ve devam metni geçmişe aittir. Yardımcı kurtarılan hedefi kendiliğinden açmaz.

Yeni checkpoint’in sahibi `agent-workflow-v1-1`. Çekirdeğin ayrı varsayılan dizini `.agent-workflow-v1-1`. Desteklenen pin’li giriş `.agent-workflow-v1-1/runs/<run-id>` kullanır. Eski `.agent-workflow` ve `.agent-workflow-v1-0` dizinleri içe aktarılmaz. Ayrılmış alanlardır, görev yolu veya geri yükleme hedefi yapılamaz. Güncel snapshot temizliği eski sahipliği silmez. Doğrudan çekirdek CLI reddedilir.

<a name="move-the-user-installation-to-the-11-technical-identity"></a>

## Kurulumu 1.1 teknik kimliğine taşıma

1. Mevcut worker ve yazıcıları envanterle. Yalnız sana ait çalışmayı durdur. Eski kurulumu taşımadan sahipli duruşu ve süreç çıkışını doğrula.
2. Eski paketi sahiplik kaydıyla karşılaştır. Keşfedilen skill dizinlerinin dışında yedek al ve hash’lerini doğrula. Önce kullanıcı değişikliklerini koru.
3. Yeni manifest ve harici pin ile `skills/agent-workflow-v1-1` yoluna temiz kur. Önceki kimliğe `--update` uygulama.
4. Keşif yollarını değiştirmeden yeni kurulu girişten benzersiz doğrulama run kimliğiyle `workflow validate` çalıştır. Kimlik, sürüm, çalışma alanı ve manifest pin’ini karşılaştır. `run-binding.json` oluşur. Hedef, workflow durumu ve izleyici başlamaz.
5. Yeni çağrı `$agent-workflow-v1-1`. Doğrudan doğrulama geçince ana ve rol skill keşif yollarını taşı. Etkinlik ve açık çağrı ayarlarını koru. Eşzamanlı config değişikliklerini koru. Sahipli süreçleri yeniden incele ve başka oturumun kullandığı paketi taşıma. Güvenli geçişten sonra eski kurulumu keşif dışına arşivle, config ve yedeği doğrula.
6. Eski checkpoint, run bağı ve makbuzların kimlik, yol ve hash’lerine dokunma. Yeni paketle yeni run aç. Bitmemiş iş varsa doğrulanmış önceki çıktıları ve geçerli güncel yetkiyi ayrı gir.
