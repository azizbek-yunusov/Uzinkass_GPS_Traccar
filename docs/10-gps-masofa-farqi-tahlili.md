# GPS Masofa Farqi Tahlili (GPS km < qog'oz hisobot km)

> **Hujjat sanasi:** 2026-11-02
> **Holat:** 🟡 Tahlil qilindi, gipotezalar tuzildi — **SQL tekshiruv natijalari kutilmoqda** (6-bo'lim)
> **Tegishli tizim:** production `10.0.57.102` (Traccar 5.5), yangi server `10.0.57.64` (Traccar 6.16)
> **Bog'liq:** [`01-server-tahlili.md`](01-server-tahlili.md) (konfiguratsiya, qurilmalar)

---

## 1. Muammo

Traccar'dagi mashinaning **bir kunlik yurgan masofasi** qog'oz hisobotdagi (yo'l varaqasi / путевой лист) masofadan **30–40% kam** chiqyapti.

**Kuzatilgan faktlar:**

| Fakt | Ahamiyati |
|---|---|
| GPS qurilma serverga **har ~2 soniyada** ma'lumot yuboradi | Nuqtalar zich — siyraklik muammosi yo'q |
| Xaritada kunlik trek **to'g'ri** ko'rinadi, uzun to'g'ri chiziqlar (uzilishlar) deyarli yo'q | Ma'lumot serverga to'liq yetib kelyapti |
| GPS km **kam**, qog'oz km **ko'p** | Farq bir tomonlama (tasodifiy emas) |

**Asosiy xulosa:** muammo ma'lumot **yig'ishda** emas, km **raqamini hisoblashda** (yoki qog'oz tomonda).

> 2 soniyalik zich trekdan hisoblangan masofa odatda real masofadan **biroz ko'p** chiqadi (GPS "titrashi" qo'shiladi). Zich, to'g'ri trek 30–40% **kam** masofa berishi deyarli mumkin emas — demak hisobotdagi raqam trekdan olinmayapti.

---

## 2. Traccar masofani qanday hisoblaydi

> Kod havolalari `traccar/` submodule'dan (**v6.16.0**). Production'dagi 5.5 da ham mantiq shu tamoyilda (odometr ustun) — kerak bo'lsa 5.5 manbasida tasdiqlash.

### 2.1 Har bir pozitsiyada — `DistanceHandler`

`traccar/src/main/java/org/traccar/handler/DistanceHandler.java`

1. Joriy nuqta bilan **oldingi nuqta** orasidagi **to'g'ri chiziq** masofasi hisoblanadi (`:54`) → `distance` atributi.
2. `totalDistance = oldingi totalDistance + distance` (`:70`) — qurilmaning jamlangan "GPS odometri".
3. Agar `coordinates.filter` yoqilgan bo'lsa va siljish `coordinates.minError` dan kichik / `coordinates.maxError` dan katta bo'lsa — koordinata oldingisi bilan almashtiriladi va `distance = 0` (`:56-64`).

### 2.2 Hisobotda — odometr ustun turadi

`traccar/src/main/java/org/traccar/helper/model/PositionUtil.java:38-49`, `reports/SummaryReportProvider.java:99`

**Summary** hisoboti masofani nuqtalarni qo'shib **emas**, faqat davrning **birinchi va oxirgi** pozitsiyasidan oladi:

```
agar (ignoreOdometer = false) va odometer(birinchi) != 0 va odometer(oxirgi) != 0:
    masofa = odometer(oxirgi) − odometer(birinchi)        ← QURILMA hisoblagichi
aks holda:
    masofa = totalDistance(oxirgi) − totalDistance(birinchi)  ← trekdan hisoblangan
```

**Trips** hisoboti ham xuddi shunday (`reports/common/ReportUtils.java:241-249`).

### 2.3 Teltonika odometri

`traccar/src/main/java/org/traccar/protocol/TeltonikaProtocolDecoder.java:204`

```java
register(16, any, (p, b) -> p.set(Position.KEY_ODOMETER, b.readUnsignedInt()));
```

Teltonika **IO 16 (Total Odometer)** ni yuborsa, u `odometer` atributiga yoziladi va **hisobotdagi km shu qurilma hisoblagichidan olinadi** — xaritadagi trekdan emas.

> 💡 Shuning uchun "trek to'g'ri, lekin km noto'g'ri" holati to'liq mumkin: trek koordinatalardan chiziladi, km esa qurilmaning ichki odometridan.

---

## 3. Gipotezalar

| № | Sabab | Qayerda | Ehtimollik | Holat |
|---|---|---|---|---|
| 1 | **Teltonika ichki odometri (IO 16) kam sanayapti** — Traccar hisobotda shuni ishlatadi | Qurilma sozlamasi (serverda chetlab o'tish mumkin) | 🔴 Yuqori | Tekshirilmoqda |
| 2 | **Trips hisoboti** ishlatilmoqda — `report.trip.minimalTripDuration = 300` s va `minimalTripDistance = 500` m dan qisqa qatnovlar hisobga olinmaydi (inkassatsiyada qisqa qatnovlar ko'p) | Server / hisobot tanlovi | 🟠 O'rta | Qaysi hisobot ishlatilishini aniqlash kerak |
| 3 | **Vaqt zonasi / davr** — foydalanuvchi sozlamasi UTC bo'lsa, "kun" Toshkent bo'yicha 05:00 → 05:00 olinadi | Server (foydalanuvchi sozlamasi) | 🟡 Past–o'rta | Tekshirish kerak |
| 4 | **`coordinates.minError` filtri** — kichik siljishlarni 0 qiladi | Server (`traccar.xml`) | 🟡 Past | `traccar.xml` ni ko'rish kerak |
| 5 | **Qog'oz / spidometr ko'p ko'rsatadi** — spidometr zavoddan +3–10%, nostandart shina; yo'l varaqasida yoqilg'i normasini oqlash uchun km oshirib yozilishi | Inson / mashina | 🟠 O'rta | Spidometr surati bilan tekshirish |
| 6 | ~~GPS nuqtalari siyrak (60–300 s) — burilishlar "kesiladi"~~ | Qurilma | ⚪ | **Chiqarib tashlandi** — 2 s interval |
| 7 | ~~Ma'lumot uzilishlari (aloqa yo'q, xotira to'lgan, uyqu rejimi)~~ | Qurilma / aloqa | ⚪ | **Chiqarib tashlandi** — trekda uzilish yo'q |

---

## 4. Diagnostika

### 4.1 Asosiy SQL — bitta mashina, bitta kun, 4 xil masofa

Bitta so'rov bilan trekning **haqiqiy uzunligi** va Traccar'ning har bir hisob manbasi yonma-yon chiqadi.

```sql
-- deviceid ni almashtiring.
-- fixtime bazada UTC'da saqlanadi: Toshkent (UTC+5) bo'yicha 30-oktabr kuni =
-- '2026-10-29 19:00' dan '2026-10-30 19:00' gacha (UTC).
WITH p AS (
  SELECT latitude, longitude, attributes::json AS a,
         lag(latitude)  OVER w AS lat0,
         lag(longitude) OVER w AS lon0
  FROM tc_positions
  WHERE deviceid = 123
    AND fixtime >= '2026-10-29 19:00' AND fixtime < '2026-10-30 19:00'
  WINDOW w AS (ORDER BY fixtime)
)
SELECT
  count(*) AS nuqtalar,
  round((sum(2*6371000*asin(sqrt(
      power(sin(radians(latitude-lat0)/2),2) +
      cos(radians(lat0))*cos(radians(latitude))*power(sin(radians(longitude-lon0)/2),2)
  )))/1000)::numeric, 1)                                                          AS trek_km,
  round(sum((a->>'distance')::numeric)/1000, 1)                                   AS traccar_distance_km,
  round((max((a->>'totalDistance')::numeric)-min((a->>'totalDistance')::numeric))/1000, 1) AS totaldistance_km,
  round((max((a->>'odometer')::numeric)-min((a->>'odometer')::numeric))/1000, 1)  AS odometr_km
FROM p;
```

| Ustun | Ma'nosi |
|---|---|
| `trek_km` | Koordinatalardan mustaqil hisoblangan trek uzunligi (haqiqatga eng yaqin) |
| `traccar_distance_km` | Traccar `DistanceHandler` qo'shgan masofalar yig'indisi |
| `totaldistance_km` | Traccar GPS odometri farqi (odometr bo'lmasa hisobot shuni oladi) |
| `odometr_km` | Teltonika IO 16 farqi (bor bo'lsa hisobot **shuni** oladi) |

> `tc_positions` da `(deviceid, fixtime)` indeksi bor — so'rov tez ishlaydi, butun jadvalni skanerlamaydi.

### 4.2 Qo'shimcha SQL — nuqtalar zichligi va uzilishlar

```sql
SELECT count(*)                                AS nuqtalar,
       round(avg(gap))                         AS ortacha_oraliq_s,
       max(gap)                                AS eng_katta_uzilish_s,
       count(*) FILTER (WHERE gap > 120)       AS uzilish_2min_dan_kop,
       count(*) FILTER (WHERE NOT valid)       AS yaroqsiz_fix
FROM (
  SELECT *, extract(epoch FROM fixtime - lag(fixtime) OVER (ORDER BY fixtime)) AS gap
  FROM tc_positions
  WHERE deviceid = 123
    AND fixtime >= '2026-10-29 19:00' AND fixtime < '2026-10-30 19:00'
) t
WHERE speed > 0 OR gap IS NULL;
```

### 4.3 Natijani talqin qilish

| Natija | Xulosa | Muammo qayerda |
|---|---|---|
| `trek_km` ≈ qog'oz, `odometr_km` kam | Teltonika odometri kam sanayapti | Qurilma (1) |
| `trek_km` ≈ qog'oz, `totaldistance_km` kam | Traccar filtri masofani kesyapti | Server (4) |
| Barcha ustunlar ≈ qog'oz, faqat UI hisobot kam | Trips hisoboti yoki vaqt zonasi | Server / hisobot (2, 3) |
| Barcha ustunlar ≈ UI hisobot, qog'oz ko'p | Trek va hisob to'g'ri | Qog'oz / spidometr (5) |

### 4.4 Mustaqil tekshiruv — spidometr

2–3 mashinada **kun boshida va oxirida spidometrni suratga olish**:

- spidometr ≈ qog'oz, GPS kam → muammo GPS hisobida (1–4)
- spidometr ≈ GPS, qog'oz ko'p → muammo qog'ozda (5)

### 4.5 Konfiguratsiyani tekshirish

```bash
# Production serverda (faqat o'qish)
grep -E "filter\.|coordinates\.|report\." /opt/traccar/conf/traccar.xml
```

Teltonika Configurator'da: **Trip/Odometer** bo'limi — odometr manbasi (GNSS / CAN / OBD), boshlang'ich qiymat, masofa hisoblash rejimi; **Data Acquisition** — yozish davri.

---

## 5. Yechimlar (tasdiqlangan sababga qarab)

| Sabab | Yechim | Qayerda |
|---|---|---|
| 1 — Teltonika odometri | **Tezkor:** qurilma (yoki guruh/global) atributiga `report.ignoreOdometer = true` — hisobot trekdan hisoblangan `totalDistance` ni oladi. **To'g'ri:** Teltonika Trip/Odometer sozlamasini tuzatish | Server (tezkor), qurilma (doimiy) |
| 2 — Trips hisoboti | Kunlik masofa uchun **Summary** hisobotidan foydalanish yoki `report.trip.minimalTripDuration` / `minimalTripDistance` ni kamaytirish | Server / foydalanuvchilar |
| 3 — Vaqt zonasi | Foydalanuvchi/server sozlamasida `Asia/Tashkent` | Server |
| 4 — Filtr | `coordinates.minError` ni kamaytirish yoki o'chirish | `traccar.xml` |
| 5 — Qog'oz | Hisobotni GPS bo'yicha yuritish; spidometr kalibrovkasi | Tashkiliy |

> ⚠️ `report.ignoreOdometer` kaliti 6.16 manbasida bor (`Keys.java:1684`, standart `false`). Production 5.5 da mavjudligini qo'llashdan oldin tekshirish kerak. Yangi serverga (6.16) o'tishda ham shu sozlama hisobga olinsin.

---

## 6. Natijalar (to'ldiriladi)

| Sana | Mashina (deviceid) | Qog'oz km | UI hisobot km (turi) | `trek_km` | `totaldistance_km` | `odometr_km` | Spidometr km | Xulosa |
|---|---|---|---|---|---|---|---|---|
| | | | | | | | | |
| | | | | | | | | |

---

## 7. Keyingi qadamlar

- [ ] Foydalanuvchilar kunlik km uchun **qaysi hisobotni** (Summary / Trips) ishlatishini aniqlash
- [ ] 1–2 mashina × 1 kun uchun 4.1 SQL ni ishga tushirish, 6-bo'limni to'ldirish
- [ ] Shu kunlar uchun spidometr suratlari
- [ ] Production `traccar.xml` dagi `filter.*`, `coordinates.*`, `report.*` sozlamalarini yozib olish
- [ ] Teltonika Configurator'da Trip/Odometer sozlamasini ko'rish
- [ ] Sabab tasdiqlangach — 5-bo'limdagi yechimni avval **1 qurilmada** sinash, keyin barchasiga
