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


HMI Tasarımı
<img width="967" height="727" alt="Screenshot 2025-08-19 212514" src="https://github.com/user-attachments/assets/91432258-91bc-4048-b946-d4de02e4e2d9" />



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

# Bu alan ne işe yarıyor?

<img width="856" height="631" alt="Screenshot 2025-08-19 211743" src="https://github.com/user-attachments/assets/a3a96182-829f-41c9-bd76-2747f854ce75" />

In Min / In Max: Giriş (raw) aralığı. Analog/REAL/INT değerin beklenen minimum–maksimum sınırları (ör. 0–27648 ya da 0–100).
Out Min / Out Max: Mühendislik birimi aralığı. Giriş aralığını buraya ölçekleyeceğiz (örn. 0–10 bar, 0–100 %).
Log Repeat time: Loglama örnekleme periyodu (görselde T#1s). Her tetikte 1 kayıt alınır.

# Bu alan ne işe yarıyor?
<img width="837" height="610" alt="Screenshot 2025-08-19 212338" src="https://github.com/user-attachments/assets/0eab11e7-3023-4f95-a50d-4fd0fc1997cf" />

Text: Banda verilecek isim. (Şu an “Value1, Value2…”; istersen “Düşük, Normal, Yüksek…” yap.)
MinValue / MaxValue: O bandın kapsadığı alt–üst sınırlar.
Örn: Value1: 0–10, Value2: 10–20, Value3: 20–30 …

# Bu alan ne işe yarıyor?
<img width="840" height="612" alt="Screenshot 2025-08-19 212428" src="https://github.com/user-attachments/assets/9d618d6f-138a-4a58-8b30-bc1c39cf3dee" />

Burası Log Control DB içindeki anlık çıktı ve tarihçenin tutulduğu bölüm:
Value: O anda ölçeklenmiş (scaled) sayısal değer.
Text: O değerin düştüğü band/etiket (Warning Control’deki aralıklarla eşleşen metin).
Altındaki diziler:
Prescription[1..100]: Her biri tek bir log kaydıdır.
Tarih/Saat (DTL): Kaydın alındığı zaman damgası (S7-1500’de RD_LOC_T ile doldurulur).
Value: O anda hesaplanan ölçekli değer.
Text: O anda seçilen bandın metni (örn. “Value5”, “Normal”, “Alarm” vb.).
Bu alanlar WinCC’de gördüğün tabloyla birebir eşleşir:

Log Time ⇢ Prescription[i].TarihSaat
Log Value ⇢ Prescription[i].Value
Log Text ⇢ Prescription[i].Text

