
# GODBRAIN QUANTUM  
### Adaptive Research & Trading Operating System (Public Overview)

> “If it thinks, it must learn. If it learns, it must survive.”

GODBRAIN QUANTUM;  
laboratuvar (research) ve canlı piyasa (execution) tarafını birbirinden **fiziksel olarak ayıran**,  
aralarında sadece küçük bir **“Neural Link”** bırakan,  
kendi kendini izleyen, toparlayan ve evrimleştiren bir **research & trading operating system**’dir.

Bu repo, GODBRAIN ekosisteminin **genel mimarisini ve tasarım prensiplerini** anlatır.  
Ticari sırlar, hassas parametreler ve strateji detayları **bilinçli olarak dışarıda bırakılmıştır.**

---

## 1. Tasarım Hedefleri

GODBRAIN şu sorulara cevap vermek için tasarlandı:

- Canlı borsada çalışan bir botu, **laboratuvarda evrimleşen bir beyinle** nasıl güvenli bağlarız?
- Sistemi öyle kurabilir miyiz ki:
  - Bir modül çökse bile **diğerleri ayakta kalsın**,  
  - Hataları mümkün olduğunca **kendi kendine toparlasın**,  
  - Kararlarını **bilimsel olarak kayıt altına alıp** sonradan analiz edebilelim?

Bu nedenle sistem şu üç ana hedefe göre inşa edildi:

1. **İzole Mimari (Decoupled Architecture)**  
   – Lab ve Live tamamen ayrı process’ler ve klasörler.
2. **Gözlemlenebilir Evrim (Observable Evolution)**  
   – Her cycle, her DNA değişimi, her piyasa modu veri seti olarak kaydediliyor.
3. **Ölümsüzlük + Öz-koruma (Self-Healing + Circuit Breaker)**  
   – Sistem kapanmıyor; ama “intihar döngüsü”ne girerse kendini durdurup insan çağırıyor.

---

## 2. Yüksek Seviyeli Mimari

Sistem 4 ana terminal metaforu ile çalışır:

### 2.1 Terminal 1 – Live Bot (KAS / Sağ Lob)

- Dosya: `agg.py`  
- Görev: Borsa (OKX vb.) ile gerçek zamanlı bağlantı, emir açma/kapama.  
- Özellik: Sadece **`live_dna.json`** dosyasını okur, asla yazmaz.

### 2.2 Terminal 2 – GODCOSMIC_CORE Lab (BEYİN / Sol Lob)

- Modül: `godcosmic_core.infinite_runner`  
- Görev: Simüle evrenler üzerinde sürekli deney koşturmak, yeni strateji DNA’ları üretmek.  
- Çıktı: `live_dna.json` + `lab_status.json`

### 2.3 Terminal 3 – System Monitor (GÖZLEM KULESİ)

- Dosya: `system_monitor.py`  
- Görev: Lab’ın nabzını tutmak.  
- Gösterge: `🟢 ONLINE / 🟠 STALE / 🔴 OFFLINE` ve “Last Pulse: X minutes ago”.

### 2.4 Terminal 4 – GODBRAIN RESEARCH CONSOLE (ANALİTİK PANOSU)

- Framework: **Streamlit**  
- Görev: `research_data/` altındaki CSV dataset’leri okuyup:

  - **Cycle Count**  
  - **World Mood (Psych Score)**  
  - **Champion Profit**  
  - **Current Stop Loss %**  
  - **Fitness Evolution (Profit over Cycles)**  
  - **DNA Strategy Shift (RSI / Stop Loss değişimi)**  

  üzerinden yapay zekânın evrimini görselleştirmek.

### 2.5 Shared State Katmanı

Tüm bu terminaller, ortak bir **Shared State** klasörü üzerinden konuşur:

```text
godbrain-shared/
  ├── live_dna.json      # Lab yazar, Bot okur
  ├── lab_status.json    # Lab yazar, Monitor & Dashboard okur
  └── (opsiyonel) bot_status.json
````

---

## 3. Neural Link Protokolü

Lab ve Live arasındaki tek bağ, **JSON tabanlı bir “Neural Link”**tir.

### 3.1 Lab → Live

* Lab, her cycle sonunda yeni bir strateji üretir ve bunu `live_dna.json` içine yazar.
* Bu JSON; stop loss, take profit, risk iştahı, RSI seviyeleri gibi **soyut strateji parametrelerini** içerir.
* Format intentionally generic tutulmuştur; ticari sır niteliğindeki metrikler paylaşılmaz.

### 3.2 Live → Lab

* Live bot, bu dosyayı periyodik olarak okur.
* Dosya değişmediyse eski strateji ile devam eder.
* Dosya bozuk veya eksik ise **güncellemeyi reddeder** ve son sağlam DNA ile devam eder.

### 3.3 Dayanıklılık

Bu sayede:

* Lab çökerse → son yazdığı DNA ile bot çalışmaya devam eder.
* Bot çökerse → Lab, simülasyon ve yeni DNA üretimine devam eder.

Hiçbir kritik bileşen diğerine **bağımlı** değildir; yalnızca ondan **beslenir.**

---

## 4. Atomic I/O & Data Integrity

GODBRAIN, dosya tabanlı iletişimde klasik “race condition / partial write” problemini çözmek için **atomic I/O pattern’i** kullanır.

### 4.1 Yazma Tarafı (Lab)

1. JSON veri önce `*.tmp` dosyasına yazılır.
2. Dosya başarıyla flush edildikten sonra `rename()` ile gerçek dosya adının üzerine atomik olarak taşınır.

```text
live_dna.json.tmp  →  live_dna.json  (atomic rename)
```

Sonuç:

* Sistem hiçbir zaman “yarım yazılmış JSON” görmez.
* Disk yazımı sırasında crash olsa bile, eski sağlam dosya korunur.

### 4.2 Okuma Tarafı (Live Bot)

* Dosyanın **mtime** değeri değiştiyse okur, aksi halde cache’den devam eder.
* JSON parse edildikten sonra schema kontrolü yapılır:

  * Gerekli alanlar (`stop_loss_pct`, `take_profit_pct`, `rsi_buy_level`, `rsi_sell_level`, vb.) yoksa güncelleme reddedilir.
  * Hata durumunda eski strateji ile devam edilir.

Bu iki katman birleştiğinde, **veri bütünlüğü üretim seviyesinde** garanti altına alınmış olur.

---

## 5. Scientific Logger & Research Dataset

Sistem sadece “log” basmaz; aynı zamanda **bilimsel veri toplar.**

Her lab cycle’ında özetle şu bilgiler **`research_data/`** altında CSV formatında saklanır:

* `Timestamp`
* `Cycle` ID
* `Market_Mood` (Trend / Range / Crash / Mixed)
* `Fear_Greed_Index / Psych Score`
* `Best_Fitness_PnL` (kâr metriği)
* DNA strateji parametreleri (özelleştirilmiş ve soyutlanmış biçimde)

Bu dataset:

* Paper / rapor yazımı,
* Backtest analizi,
* Evrimsel davranışların istatistiksel testleri

için kullanılmak üzere tasarlanmıştır.

**Özel strateji fonksiyonları veya domain-spesifik formüller bu dataset’lerde yer almaz; sadece yüksek seviyeli özetler tutulur.**

---

## 6. GODBRAIN RESEARCH CONSOLE

`GODBRAIN RESEARCH CONSOLE`, yukarıdaki dataset’i **interaktif bir araştırma laboratuvarına** dönüştürür.

### 6.1 Ana Özellikler

* **KPI Şeridi**

  * `Cycle Count`
  * `World Mood` + `Psych Score`
  * `Champion Profit`
  * `Current Stop Loss %`

* **Fitness Evolution (Profit over Cycles)**
  – Zaman içinde kâr eğrisi → sistem gerçekten öğreniyor mu?

* **DNA Strategy Shift**
  – RSI / Stop Loss gibi parametrelerin cycle bazlı değişimi →
  *“Scalper’dan ‘Survivor’a, oradan Hyper-Defensive moda evrim”* gibi emergent davranışlar burada görünür hale gelir.

* **Raw Data Explorer**
  – `research_data/` altındaki satırların tablo halinde incelenmesi, filtreleme ve slice alma imkânı.

### 6.2 Emergent Behavior Örneği

Dashboard üzerinden gözlemlediğimiz tipik bir senaryo:

* Cycle #2:

  * Göreli olarak yüksek kâr
  * Daha agresif RSI / Stop Loss ayarları
* Cycle #3:

  * Piyasa rejimi değiştiğinde, sistem yüksek kârdan ziyade
    **risk-adjusted survival** moduna geçiyor
  * Daha derin RSI giriş eşiği, daha defansif davranış, daha düşük PnL fakat daha güvenli profil

Bu, sistemin **sadece matematiksel optimizasyon değil, stratejik derinlik** kazandığını gösteren önemli bir işaret.

---

## 7. Context-Aware Evolution (WorldSensor)

Lab tarafında çalışan **WorldSensor** modülü;

* Simüle edilen **piyasa modunu** (trend / range / crash / mixed),
* Psikoloji / sentiment skorlarını,

kullanarak **evren parametrelerini** değiştirmektedir.

Örneğin:

* “Fear” ortamında:

  * Daha volatil, crash ağırlıklı evrenler koşturulur.
  * Bot DNA’ları daha defansif hale gelir.
* “Greed” ortamında:

  * Trend-following senaryolar öne çıkar.
  * Risk iştahı (örneğin pozisyon çarpanı) daha yüksek denenir.
* “Mixed / Range” modunda:

  * Sistem scalper / swing hibrit stratejiler dener, sabır eşiğini ve stop mesafesini ayarlar.

Bu sayede GODBRAIN, “sabit bir stratejiyi optimize eden bot” olmaktan çıkıp,
**dünyanın haline göre karakter değiştiren adaptif bir ajan** gibi davranır.

---

## 8. Self-Healing & Circuit Breaker (GODBRAIN SENTINEL v2.0)

Üretim sistemleri için kritik konulardan biri;
sistemin hem **kendi kendini toparlayabilmesi**, hem de **sonsuz crash döngüsüne girmemesi**dir.

GODBRAIN bunu iki katmanda çözer:

### 8.1 İç Katman – Yazılım Seviyesi

* `try/except` blokları
* Atomic I/O
* Validation

sayesinde lokal hatalar çoğu zaman **sistemi durdurmadan** absorbe edilir.

### 8.2 Dış Katman – Process Seviyesi (Sentinel v2.0)

PowerShell tabanlı **GODBRAIN SENTINEL v2.0** wrapper’ları:

* İlgili script’i (`agg.py` veya `godcosmic_core.infinite_runner`) sonsuz loop içinde çalıştırır.

* Process ölürse:

  * 3 saniye bekler ve yeniden başlatır (**self-healing**).

* 60 saniye içinde 5’ten fazla crash olursa:

  * **Circuit Breaker devreye girer.**
  * “🛑 HARD FAIL” moduna geçer, restart etmeyi bırakır.
  * Operatöre:
    “System is crashing too fast. Check logs and fix the bug.” mesajı verilir.

Bu kombinasyon, **Kubernetes / systemd** tarzı modern production pattern’lerinin
hafif, script tabanlı bir karşılığıdır.

---

## 9. Güvenlik ve Sırlar

Bu repo:

* **Hiçbir borsa API anahtarını, hesap detayını veya gerçek trade geçmişini içermez.**
* DNA formatları ve evrimsel algoritmaların **çekirdek matematiği** gizli tutulur.
* Sadece mimari tasarım, pattern’ler ve araştırma metodolojisi paylaşılır.

Canlı sistemde kullanılan:

* Parametre tuning’leri
* Özel feature set’leri
* Proprietary scoring / selection fonksiyonları

**açık kaynakta bulunmaz**; yalnızca yüksek seviyeli özetlere yer verilir.

---

## 10. Yol Haritası (Public Roadmap)

* [ ] Reproducible experiment script’leri (`experiments/exp_*.py`)
* [ ] Otomatik test & CI pipeline’ı
* [ ] Özet teknik rapor / whitepaper (PDF)
* [ ] Plug-in architecture ile farklı execution motorları (farklı borsalar / piyasa türleri)
* [ ] Anonimleştirilmiş “case study” sonuçlarının paylaşımı

---

## 11. Lisans

Bu repo, **öğrenme ve ilham amacıyla** paylaşılan bir mimari örnektir.
Ticari kullanım, kopyalama ve türev projeler için lisans koşulları henüz netleştirilmemiştir.

> **Not:** Bu sistem, finansal tavsiye vermez.
> Herhangi bir gerçek para ile kullanım, tamamen kullanıcının kendi sorumluluğundadır.

---

## 12. İmza

GODBRAIN QUANTUM & GODCOSMIC_CORE
**Design & Architecture**

**Ünal Tüzün / Azun'el Skywolf**

```
```
