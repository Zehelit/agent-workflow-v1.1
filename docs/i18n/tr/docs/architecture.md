<a name="architecture-and-orchestration"></a>

# Mimari ve orkestrasyon

AGWF’de yetkinin kaynağı kullanıcıdır. Yerleşik Codex araçları hedefi ve gerçekleşen işlemleri gösterir. Python candidate, kendisine iletilen kanıtı denetler ve yerel durum geçişlerini yönetir. Bir JSON kaydının geçerli olması, kaynağının bağımsız olarak doğrulandığı anlamına gelmez.

<a name="components-and-ownership"></a>

## Bileşenler ve sorumluluklar

```mermaid
flowchart TD
    User["Kullanıcı"] -->|"kapsam ve açık çağrı"| Main["Ana Codex ajanı"]
    Main -->|"komutlar ve kanıtlar"| Candidate["candidate.py"]
    Candidate -->|"tipli durum geçişi"| Core["workflow.py"]
    Core -->|"durum ve snapshot"| Run["Proje çalışma dizini"]
```

Candidate, paketi projeye ve çalışma kimliğine bağlamadan önce harici manifest pin’ini ve desteklenen giriş noktasını denetler. Çekirdek görevleri, sahipliği, tekrar kullanım korumalarını, yetki durumunu ve snapshot bilgilerini saklar. Bir koordinasyon günlüğü, sonucu belirsiz veya bekleyen işlemleri kaydeder. İzleyicinin kanıt akışı ayrıdır. Paket bütünlüğü komut ve ölçüm sınırlarında denetlenir. Kalp atışında daha küçük sahiplik ve kontrol kaydı doğrulanır.

```mermaid
flowchart TD
    Host["Yerleşik Codex araçları"] -->|"kullanım ölçümü"| Main["Ana ajan"]
    Main -->|"ölçüm kaydı"| Watcher["Python izleyicisi"]
    Watcher -->|"örneklem ve sağlık"| Candidate["candidate.py"]
```

İzleyiciyi candidate başlatır. Ana ajan, hedefi ve alt ajan envanterini de host’tan alır. İzleyici çalışma ve epoch lease’ini denetler. Codex kullanımını doğrudan okuyamaz.

| Sınır | Kaynak |
| --- | --- |
| Harici hash, bağımlılıklar, etkinleştirme ve izleyici yaşam döngüsü | [Candidate](../../../../skills/agent-workflow-v1-1/scripts/candidate.py), [paket bağı](../skills/agent-workflow-v1-1/references/coexistence.md) |
| Yetki, dosya sahipliği, atama hazırlığı, çıktı kontrolü ve checkpoint | [İş akışı çekirdeği](../../../../skills/agent-workflow-v1-1/scripts/workflow.py), [komut sözleşmesi](../skills/agent-workflow-v1-1/references/commands.md) |
| Kullanım ölçümü, replay, kalp atışı, lease ve duruş kararı | [İzleyici](../../../../skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py), [yerleşik kullanım kayıtları](../skills/agent-workflow-v1-1/references/native-usage.md) |
| Oturum hazırlığı ve yaşam döngüsü gözlemlerini işleme | [Yaşam döngüsü aracı](../../../../tooling/run-lifecycle-check.py), [kullanım kılavuzu](lifecycle-check.md) |

<a name="who-hands-what-to-whom"></a>

## Kim kime ne iletir?

```mermaid
sequenceDiagram
    participant Main as Ana ajan
    participant Core as Sabitlenmiş yardımcı
    Main->>Core: Yetkili görev<br/>dosya sahipliği ve envanter
    Core-->>Main: Hazır atama
    Main->>Core: Gerçek atama makbuzu
    Main->>Core: Bitiş envanteri<br/>ve kabul kanıtı
```

Atama, hazırlık ile atama makbuzunun kaydı arasında Codex üzerinden gönderilir. Çıktı ve kanıt ana ajana döner.

```mermaid
sequenceDiagram
    participant Main as Ana ajan
    participant Child as Alt ajan
    Main->>Child: Codex aracıyla görevlendir
    Child-->>Main: Dosyalar ve kanıt
```

Ana ajan küçük bir görevi kendisi yürütebilir. Koordinatör, kapsamlı işler için sınır önerebilir. Mimar, uygulayıcı ve araştırmacı sınırları belirlenmiş görevlere atanır. Doğrulayıcı, kontrol edilen çıktıyı bağımsız inceleyebilir. Pakette [rol tanımları](../../../../skills/agent-workflow-v1-1/roles) ve yönergeleri bulunur. Ana ajan kullanıcının seçtiği model ve muhakeme düzeyiyle çalışır. Hazırlanan ayar ile sağlayıcıdan gözlenen ayar ayrı kanıtlardır.

Sonucu belirsiz bir atamayı tekrar göndermeden önce ana ajan hazırlanan argümanları ve gerçek oturumu taze envanterle uzlaştırır. Alt ajanın bitiş durumu, yürütme belirsizliğini giderir. Çıktının doğruluğu ayrı denetlenir. Bağımsız doğrulayıcı belirli çıktı sürümünü ve hash’i esas alır. Dosya sonradan düzenlenirse bu inceleme hedefi geçersizleşir. [Kabul ve iş devri](../skills/agent-workflow-v1-1/references/context.md).

<a name="what-survives-a-stop"></a>

## Duruşta hangi kayıtlar korunur?

```mermaid
flowchart TD
    Save["SAVE_AND_STOP"] -->|"sahipli yazıcıları durdur"| Snapshot["Snapshot ve geri yükleme"]
    Snapshot -->|"dosya ve hash doğrulandı"| Halt["Gözlenen nedenle durdur"]
    Halt -->|"ölçüm eskidi<br/>aynı etkin hedef"| Check["Taze kanıt ve süreç çıkışı"]
    Check -->|"geçerli checkpoint"| Resume["Doğrulanan devam<br/>yeni epoch"]
    Halt -->|"başka neden"| Wait["Nedene uygun açık devam"]
```

Snapshot belirtilen değişiklikleri, silinen yol kayıtlarını, görev sahipliği kayıtlarını ve yerel durumu içerir. Araç yeni bir dizine geri yükler ve hash’leri karşılaştırır. Değişmiş veya eksik checkpoint uzlaştırılmadan kullanılamaz. Yardımcı yerleşik hedefin durumunu değiştirmez. Görev buna yetki veriyorsa ana ajan durumu Codex araçlarıyla okur veya günceller.

Bu teknik toparlanma yalnız kayıtlı ölçüm eskimesi olan ve koşulları karşılayan bağlı iş akışına uygulanır. Aynı yetkili hedef hâlâ etkin olmalı ve duruştan sonra gerçek gözlemler toplanmalıdır. Yalnız izleyici modunda açık devam veya doğrulanmış yerleşik geçiş gerekir. Karar vermeden önce [işletim ve toparlanma kurallarını](../skills/agent-workflow-v1-1/references/operations.md) okuyun.
