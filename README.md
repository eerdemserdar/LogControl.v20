# UDT Scale & Log (TIA Portal / SCL)

> Analog/real/int bir **giriş değeri**ni tanımlı **min–max aralıklarından** hedef aralığa **ölçekleyip** (scale) sonucu **100 kayıt**lık bir **ring buffer**'da periyodik olarak **loglayan** UDT tabanlı çözüm.

<p align="center">
  <!-- <img src="docs/cover.png" alt="Kapak" width="720"/> -->
</p>

<p align="center">
  <!-- <a href="#"><img src="https://img.shields.io/badge/platform-S7--1200%2F1500-informational"/></a>
  <a href="#"><img src="https://img.shields.io/badge/language-SCL-blue"/></a>
  <a href="#"><img src="https://img.shields.io/badge/TIA%20Portal-v17%2B-success"/></a>
  <a href="#katk%C4%B1da-bulunma"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen"/></a> -->
</p>

---

## Özet

Bu proje; **UDT yapısı** ile çalışan, **kaynak değeri** (INT/DINT/REAL olabilir) **min–max** giriş aralığına göre **normalize eden** ve istenen **hedef aralığa** (ör. 0–100, 4–20 mA → 0–10 bar vb.) **scala** eden bir **FB** içerir. Elde edilen **Out** değeri belirlenen **örnekleme periyoduyla** (örn. 100 ms, 1 s) **100 adet**lik bir **ring buffer**’a kaydedilir.

**Neden UDT?** UDT sayesinde, farklı istasyon/sinyaller için yalnızca **hedef DB’de UDT örneği** tanımlayıp aynı FB’yi **IN\_OUT** olarak bu UDT’ye bağlamak yeterlidir. Böylece her sinyal için tekrar tekrar farklı kod yazmadan **kolayca log** tutabilirsiniz.

## Öne Çıkanlar

* 🔧 **UDT merkezli tasarım**: Her sinyal için tek tip veri şeması; FB çağrısında yalnızca hedef UDT’yi bağlayın.
* 🧰 **Tip esnekliği**: Kaynak INT/DINT değerlerini REAL’e çevrilerek işleme imkânı.
* 📚 **Ring buffer log (100 kayıt)**: Kayıtlar dairesel olarak tutulur; fazla olduğunda en eski üstüne yazar.
* 🖥️ **HMI/SCADA dostu**: UDT içindeki buffer, trend/tabloda kolayca görselleştirilebilir.

## Ekran Görüntüleri / Demo

<img width="856" height="631" alt="Screenshot 2025-08-19 211743" src="https://github.com/user-attachments/assets/f84e938c-89a3-428f-ae1d-f132d0248e94" />

<img width="837" height="610" alt="Screenshot 2025-08-19 212338" src="https://github.com/user-attachments/assets/a6cc27e6-1fbf-4873-a1a5-e717e8026897" />

<img width="840" height="612" alt="Screenshot 2025-08-19 212428" src="https://github.com/user-attachments/assets/9855443f-4cc1-4f15-b15e-1733a13b0525" />

<img width="967" height="727" alt="Screenshot 2025-08-19 212514" src="https://github.com/user-attachments/assets/cf024f77-c25b-4bdb-8a19-190349476d70" />





> Görsellerdeki yapıya uyumlu: **Log Control DB** altında `LogData.Settings` (InMin, InMax, OutMin, OutMax, Log Repeat time), **Warning Control\[1..10]** (Text/MinValue/MaxValue), anlık **Value/Text** ve **Prescription\[1..100]** (Tarih/Saat, Value, Text).

## Teknoloji Yığını & Destek Matrisi

| Bileşen    |           Sürüm/Model | Not                |
| ---------- | --------------------: | ------------------ |
| PLC        |     S7‑1200 / S7‑1500 | OB1/OB35 destekli  |
| TIA Portal |                  v20  | SCL ile geliştirme |

## Gereksinimler

* TIA Portal v20+ (önerilir)
* (İsteğe bağlı) OB35 periyodik kesmesi
* S7‑1200 / S7‑1500
* HMI Panel

## Kurulum

1. UDT’yi ekleyin (örn. `LogControlUDT`).
2. FB’yi ekleyin (örn. `Log ControlFC`).
3. Her sinyal için hedef DB’de `LogControlUDT` alanı oluşturun (örn. `TankLevel.LogControlUDT`).
4. OB1/OB35 içinde FB’yi çağırıp **IN\_OUT**’a ilgili UDT alanını bağlayın.


Bu alan ne işe yarıyor?
<img width="856" height="631" alt="Screenshot 2025-08-19 211743" src="https://github.com/user-attachments/assets/a3a96182-829f-41c9-bd76-2747f854ce75" />

In Min / In Max: Giriş (raw) aralığı. Analog/REAL/INT değerin beklenen minimum–maksimum sınırları (ör. 0–27648 ya da 0–100).

Out Min / Out Max: Mühendislik birimi aralığı. Giriş aralığını buraya ölçekleyeceğiz (örn. 0–10 bar, 0–100 %).

Log Repeat time: Loglama örnekleme periyodu (görselde T#15s). Her tetikte 1 kayıt alınır.


> Notlar:
>
> * Görsellerde **Prescription\[1..100]** dolumu *kaydırma* ile yapılmış görünüyor. Performans için **ring buffer** alanlarını (`Head`, `Count`) etkin kullanmayı öneriyoruz.
> * **Visible in HMI** onaylarının tabloda açık olduğunu gördüm; README’ye HMI alan adları bu varsayımla yazıldı.

## FB Arayüzü (Güncel SCL Örneği)

```pascal
FUNCTION_BLOCK FB_ScaleLogger
  VAR_IN_OUT
    U : UDT_ScaleLog;  // Log Control DB içindeki alan
  END_VAR
  VAR
    ton        : TON;
    spanIn     : REAL;
    spanOut    : REAL;
    norm       : REAL;
    i          : INT;
    now        : DTL;   // RD_LOC_T ile doldurulacak
    nextHead   : INT;
  END_VAR
BEGIN
  // --- SCALE ---
  IF U.Settings.InMax > U.Settings.InMin THEN
    spanIn  := U.Settings.InMax - U.Settings.InMin;
    spanOut := U.Settings.OutMax - U.Settings.OutMin;
    norm := (U.Value - U.Settings.InMin) / spanIn; // U.Value burada InputRaw ise ona göre değiştirin
    IF norm < 0.0 THEN norm := 0.0; END_IF;
    IF norm > 1.0 THEN norm := 1.0; END_IF;
    U.Value := U.Settings.OutMin + norm * spanOut; // Ölçekli değer
  END_IF;

  // --- BAND / TEXT SEÇİMİ ---
  U.Text := '';
  FOR i := 1 TO 10 DO
    IF (U.Value >= U.WarningControl[i].MinValue) AND (U.Value <= U.WarningControl[i].MaxValue) THEN
      U.Text := U.WarningControl[i].Text;
      EXIT;
    END_IF;
  END_FOR;

  // --- LOG (RING BUFFER) ---
  ton(IN := TRUE, PT := U.Settings.LogRepeatTime);
  IF ton.Q THEN
    // Zaman damgası
    RD_LOC_T(RET_VAL := , PDTL := now); // S7-1500; eski CPU'larda READ_CLK kullanın

    // Sıradaki indeksi hesapla
    nextHead := U.Head;
    IF nextHead < 1 OR nextHead > 100 THEN nextHead := 1; END_IF;

    U.Prescription[nextHead].TarihSaat := now;
    U.Prescription[nextHead].Value     := U.Value;
    U.Prescription[nextHead].Text      := U.Text;

    // İleri sar
    nextHead := nextHead + 1;
    IF nextHead > 100 THEN nextHead := 1; END_IF;
    U.Head := nextHead;

    IF U.Count < 100 THEN
      U.Count := U.Count + 1;
    END_IF;

    // TON'u yeniden başlat
    ton(IN := FALSE);
    ton(IN := TRUE);
  END_IF;
END_FUNCTION_BLOCK
```

> Ölçümü **kaydırma** yöntemiyle yapmak istiyorsanız, mevcut diziyi 100→1 geriye kaydırıp `Prescription[1]`’e yeni kaydı yazabilirsiniz; ancak **O(N)** kopyalama maliyeti sebebiyle ring buffer daha verimlidir.

## OB Çağrısı (Örnek)

```pascal
// OB1 veya periyodik OB35 (kısa örnekleme süreleri için OB35 önerilir)
CALL FB_ScaleLogger, DB_ScaleLogger
  U := "Log Control DB".LogData; // Veya: DB_Process.Whatever (UDT_ScaleLog)
```

## Yapılandırma

* `InMin/InMax`: Kaynak sensör aralığı (örn. 0..27648, 4..20 mA eşleniği vb.)
* `OutMin/OutMax`: Mühendislik birimi aralığı (örn. 0..10 bar, 0..100 %)
* `SampleTime`: Log periyodu (örn. `T#100ms`, `T#1s`)
* `Enable`: TRUE olduğunda örnekleme başlar

## HMI/SCADA Entegrasyonu

**Tablo (görseldeki düzen):**

* **Log No**: Sayfalama/pencere başına satır sıra numarası (ör. 91..100).
* **Log Time**: `Prescription[i].TarihSaat` (DTL → tarih-saat formatlı).
* **Log Value**: `Prescription[i].Value`.
* **Log Text**: `Prescription[i].Text` (band adı).

**Sayfalama Önerisi (10 satır/ekran):**

* `Page` (INT), `RowsPerPage` = 10.
* `StartIdx = (Head - 1) - Page*RowsPerPage` (ring buffer için negatifse 100 ekleyin ve 1..100 aralığına mod alın).
* Her satır için `Idx = 1 + ((StartIdx - r) MOD 100 + 100) MOD 100)` formülüyle döngü kurup en yeni → en eski olacak şekilde gösterin.

**WinCC Ayarları:**

* Görsellerde **Visible in HMI** işaretli; aynı UDT yollarını WinCC tag’lerine bağlayın.
* DTL gösterimi için sütunda tarih/saat biçimi seçin (örn. `dd.MM.yyyy HH:mm:ss`).

## Kalite: Test, Lint, Format

* SCL statik analiz (TIA Portal Check & Compile)
* PLCSIM ile senaryo testleri (ör. farklı In/Out aralıkları, doygunluk testi)

## CI/CD

* (Opsiyonel) TIA Portal proje dosyası versiyon kontrolü (Git LFS)
* PLCSIM/Unit Test (3rd party) entegrasyonları mümkün

## Performans & İzlenebilirlik

* **Ring buffer** yaklaşımı: O(N) kopyalama/shift yok, **O(1)** yazım
* Örnekleme periyodu çok kısa ise OB35 kullanımı önerilir

## SSS / Sorun Giderme

**Out değeri doygunda** → `InMin/InMax` ile gerçek sensör aralığı (örn. 0..27648) ve mühendislik birimi `OutMin/OutMax` uyumunu kontrol edin.

**Log ilerlemiyor** → `Log Repeat time` (görselde T#15s) ve OB çağrı periyodunu doğrulayın; TON tetikleniyor mu bakın.

**Text yanlış** → WarningControl bantlarının **çakışmadığından** emin olun; \[i].MinValue ≤ \[i].MaxValue ve boşluk kalmamalı.

**Kaydırma yavaş** → Ring buffer’a geçin (`Head/Count`), O(1) yazım.

## Güvenlik

* Proses kritik eşikler için ayrıca **alarm** ve **limit** izleme önerilir (ayrı FB/FC).

## Katkıda Bulunma

* PR’larda test senaryosu ve UDT alan değişikliklerini açıkça belgeleyin.

## Sürümleme & Yayınlama

* SemVer, `CHANGELOG.md`

## Lisans

Bu proje **MIT** lisansı ile lisanslanmıştır.

## İletişim

* İsim: Ad Soyad
* E‑posta: email@örnek.com
* Proje: [https://github.com/](https://github.com/)\<kullanıcı>/<repo>

## Teşekkür

* İlham/kütüphaneler: ...
* Katkıda bulunanlar: ...
