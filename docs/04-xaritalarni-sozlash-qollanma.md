# Xaritalarni Sozlash Qo'llanmasi

> **Hujjat sanasi:** 2026-10-02
> **Asos:** Traccar web manba kodi (`traccar/traccar-web`, v6.16) tahlili
> **Bog'liq:** [`01-server-tahlili.md`](01-server-tahlili.md), [`02-versiya-taqqoslash.md`](02-versiya-taqqoslash.md)

Traccar **MapLibre GL** xarita dvigatelidan foydalanadi va ko'plab xarita provayderlarini qo'llab-quvvatlaydi. Xaritani almashtirish **server kodini o'zgartirmaydi** — bu faqat sozlama (attribute). 5.5 da ham, 6.16 da ham mexanizm bir xil (6.16 da provayderlar roʻyxati kengroq).

---

## 1. Mavjud xarita provayderlari

### Bepul (API kalit talab qilmaydi)
- **OpenStreetMap** (`osm`) — eng mashhur bepul xarita
- **OpenFreeMap** (`openFreeMap`) — bepul vector xarita
- **OpenTopoMap** (`openTopoMap`) — topografik (relyef)

### API kalit talab qiladiganlar
- **Google** — road / satellite / hybrid
- **Mapbox** — streets / outdoors / satellite / dark
- **MapTiler** — basic / hybrid
- **Bing** — road / aerial / hybrid
- **HERE** — basic / hybrid / satellite
- **TomTom** — basic
- **LocationIQ** — streets / dark *(hozir loyihada ishlatilyapti)*
- **Ordnance Survey** — (asosan Buyuk Britaniya)

### Mintaqaviy
- **Yandex** (`yandexMap`)
- **AutoNavi** (`autoNavi`) — Xitoy
- **Tencent** (`tencentMap`) — Xitoy

### 🟢 Custom — o'zingizning xaritangiz
- `custom` — istalgan tile yoki style URL (pastда batafsil)

### Overlay (ustki qatlamlar)
Asosiy xarita ustiga qo'shimcha qatlam:
- **Google Traffic** — real vaqt tirbandlik
- **OpenWeather** — bulutlar
- **OpenSeaMap** — dengiz
- **OpenRailwayMap** — temir yo'l
- **AutoNavi Labels** — yozuvlar

---

## 2. Xaritani almashtirish (eng oddiy)

Har bir foydalanuvchi o'z xaritasini tanlashi mumkin:

1. Web interfeysga kiring
2. **Settings (⚙️) → Preferences**
3. **Map** (yoki "Active map layer") ro'yxatidan xaritani tanlang

> Har bir viloyat foydalanuvchisi (Farg'ona, Namangan, ...) o'ziga alohida xarita tanlashi mumkin.

---

## 3. API kalit talab qiladigan xaritalarni ulash

Google, Mapbox, MapTiler kabi xaritalar uchun avval provayderdan **API kalit** olasiz, so'ng Traccar'ga atribut sifatida kiritasiz.

### Atribut nomlari (manba koddan)

| Xarita | Atribut nomi |
|---|---|
| Google | `googleKey` |
| Mapbox | `mapboxAccessToken` |
| MapTiler | `mapTilerKey` |
| Bing | `bingMapsKey` |
| HERE | `hereKey` |
| TomTom | `tomTomKey` |
| LocationIQ | `locationIqKey` |
| Ordnance Survey | `ordnanceSurveyKey` |

### Qayerga kiritiladi

**Hamma uchun (server darajasida):**
1. **Settings → Server** (admin huquqi kerak)
2. **Attributes** bo'limiga o'ting
3. **+ Add** → atribut nomi (masalan `mapboxAccessToken`) va qiymat (kalit) ni kiriting
4. Saqlang

**Faqat bitta foydalanuvchi uchun:**
1. **Settings → Users → [foydalanuvchi]**
2. **Attributes** → o'sha atributni qo'shing

> Foydalanuvchi atributi server atributidan ustun turadi.

---

## 4. 🟢 O'zingizning xaritangizni (custom) ulash

Eng moslashuvchan variant — o'z xaritangiz yoki tashqi tile serverini ulash. Bu `mapUrl` server atributi orqali amalga oshiriladi.

### Qadamlar

1. **Settings → Server → Attributes**
2. **+ Add** → nom: `mapUrl`
3. Qiymat sifatida ikkita formatdan birini kiriting:

**a) Tile URL (raster tiles):**
```
https://sizning-server.uz/tiles/{z}/{x}/{y}.png
```
- `{z}` — zoom, `{x}`/`{y}` — koordinata. Traccar avtomatik to'ldiradi.
- Bing uslubidagi `{quadkey}` ham qo'llanadi.

**b) MapLibre style JSON:**
```
https://sizning-server.uz/styles/custom-style.json
```
- To'liq vector xarita uslubi (ranglar, shriftlar bilan).

4. Saqlang → endi xarita roʻyxatida **"Custom"** paydo bo'ladi (3-bo'limdagi kabi tanlang).

### Qachon kerak
- **Ichki/offline tile server** — tashqi provayderga bog'liq bo'lmaslik uchun
- **Trafik ko'p** — tashqi API limitlariga tushmaslik uchun
- **Maxsus dizayn** — Uzinkass brendiga mos xarita

> O'z tile serveringizni qurish uchun: OpenMapTiles, TileServer GL yoki Mapnik kabi vositalar. Bu alohida loyiha.

---

## 5. O'zbekiston uchun amaliy tavsiyalar

| Xarita | Afzalligi | Kamchiligi |
|---|---|---|
| **OpenStreetMap / OpenFreeMap** | Bepul, O'zbekistonда yaxshi qoplangan | Sun'iy yo'ldosh yo'q |
| **Google Satellite/Hybrid** | Eng aniq, sun'iy yo'ldosh | API kalit + to'lov |
| **Yandex** | Mintaqa uchun ba'zan aniqroq | Litsenziya shartlariga e'tibor |
| **Custom (o'z tile server)** | To'liq nazorat, limitsiz | O'rnatish va saqlash mehnati |

**Tavsiya:**
- Kundalik ish uchun → **OpenStreetMap** (bepul, barqaror)
- Aniq joylashuv kerak bo'lганда → **Google Hybrid** (kalit bilan)
- Uzoq muddatli, mustaqillik uchun → **o'z tile serveringiz** (`mapUrl`)

---

## 6. Geokoder (manzilni aniqlash) — xaritadan alohida

Diqqat: **xarita qatlami** va **geokoder** (koordinatani manzilga aylantirish) — ikki alohida narsa.

Hozirgi serverда geokoder **LocationIQ**ga sozlangan (`01-server-tahlili.md` ga qarang):
```xml
<entry key='geocoder.type'>nominatim</entry>
<entry key='geocoder.url'>https://us1.locationiq.com/v1/reverse.php</entry>
<entry key='geocoder.key'>...</entry>
```
Buni ham almashtirish mumkin (masalan o'z Nominatim serveringizga) — bu `conf/traccar.xml` orqali, web Settings orqali emas.

---

## 7. Qisqa xulosa

- ✅ Boshqa xarita qo'yish **mumkin** — 20+ tayyor provayder + custom
- ✅ Almashtirish oddiy: **Settings → Preferences → Map**
- ✅ API kalitli xaritalar: **Settings → Server/User → Attributes** (`googleKey`, `mapboxAccessToken`, ...)
- ✅ O'z xaritangiz: `mapUrl` atributi (tile URL yoki style JSON)
- ✅ Server kodini o'zgartirish **shart emas** — hammasi sozlama

---

*Ushbu qo'llanma `traccar-web` v6.16 manba kodidagi `src/map/core/useMapStyles.js` tahlili asosida tuzildi. 5.5 da provayderlar roʻyxati biroz kamroq bo'lishi mumkin, lekin sozlash usuli bir xil.*
