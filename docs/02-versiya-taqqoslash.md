# Traccar Versiya Taqqoslash: 5.5 → 6.16

> **Hujjat sanasi:** 2026-10-02
> **Maqsad:** Joriy Traccar versiyasini aniqlash va bizning eski versiyamizdan (5.5) afzalliklarini o'rganish
> **Bog'liq hujjat:** [`01-server-tahlili.md`](01-server-tahlili.md)

---

## 1. Umumiy holat

| | Bizning server | Joriy versiya |
|---|---|---|
| Versiya | **5.5** (~2022-yil oxiri) | **6.16** (2026-yil sentyabr) |
| Farq | — | **5.x → 6.x — katta (major) sakrash** |
| Ortda qolish | — | 1 ta major + 16 ta minor reliz |

Biz butun bir **major versiya (5 → 6)** va ~4 yillik rivojlanishdan ortda qoldik.

---

## 2. Eng muhim afzalliklar (bizning holatimizga mos)

### 🔴 2.1 Ma'lumotlar bazasi ishlashi — ENG MUHIMI

> Bizda `tc_positions` jadvalida **~1.69 milliard qator (721 GB)** bor. Bu bo'lim bevosita biz uchun.

- **Traccar 6.14 — "batching database writes"**: yozuvlar endi **to'plam (batch)** bo'lib yoziladi. Bu Traccar jamoasining o'zi aytganicha *"tizimdagi asosiy tiqilinch (bottleneck)"* ni bartaraf etadi.
  - 230 ta qurilma doimiy pozitsiya yuborayotgan bizning tizimda disk I/O va DB yuki **sezilarli kamayadi**.
- **DB connection pool** standart hajmi oshirilgan — ko'proq bir vaqtdagi ulanishlar.
- **Thread-safety** tuzatishlari (race condition oldini olish).
- Eski/osilib qolgan ulanish sessiyalarini tozalash (resurs bo'shatish).

### 🔴 2.2 Xavfsizlik

- Traccar 6.0 va keyingi relizlarda **bir nechta jiddiy (critical) xavfsizlik tuzatishlari** kiritilgan.
- 5.5 da bu zaifliklar **ochiq** holatda — bu yangilanish uchun **eng kuchli sabab**.

### 🟠 2.3 Pozitsiyalarni to'g'ri qayta ishlash (Position Processing Pipeline)

- Qurilma ma'lumotni **noto'g'ri tartibda** (buferlab, kechikib) yuborganda ham to'g'ri qayta ishlanadi.
- Bitta qurilma pozitsiyalari **ketma-ket** qayta ishlanadi (geocoding kabi tashqi xizmat qatnashsa ham tartib buzilmaydi).
- Natijada **masofa hisoblashdagi xatolar** tuzatilgan — fleet hisobotlari aniqroq.

### 🟠 2.4 Veb-interfeys (biz `modern` React UI ishlatamiz)

- **6.14**: lazy loading — birinchi yuklanish hajmi **~90% kamaygan**, UI sezilarli tezlashgan.
- Yangilangan login dizayni va xatolarni to'g'ri ko'rsatish.
- Veb-ilova keshlash mexanizmi yaxshilangan.

### 🟡 2.5 Yangi imkoniyatlar — batafsil

#### 2.5.1 Video oqim (JT/T 1078 protokoli) — 6.13

- **Nima:** JT/T 1078 — zamonaviy Xitoy GPS/dashcam qurilmalarining standart video protokoli. Traccar endi qurilmadagi kameradan **jonli video** ko'rsata oladi.
- **Qanday ishlaydi:**
  1. Veb-interfeys qurilmaga `videoStart` / `videoStop` buyrug'ini yuboradi.
  2. Backend video oqimini HLS formatida beradi: `/api/stream/{deviceId}/live.m3u8` (`.m3u8` + `.ts` segmentlar).
  3. Frontend `hls.js` orqali oddiy HTML5 video pleyerda ko'rsatadi.
- **Ko'p kanal:** old va orqa kamera (bir nechta kanal) bir vaqtda qo'llab-quvvatlanadi.
- **Qurilmalar:** JT1078-mos dashcamlar (masalan JC181).
- **Biz uchun foyda:** avtomobillarga video-registrator (dashcam) o'rnatilsa, dispetcher real vaqtda kuzatishi mumkin. Mavjud infratuzilma ustiga qo'shimcha imkoniyat.

#### 2.5.2 MQTT forwarding — 6.0

- **Nima:** Traccar ma'lumotni **tashqi tizimga** (broker/boshqa ilovaga) uzatishi mumkin. Uzatish uchun uchta oqim bor: qayta ishlangan **pozitsiyalar**, **hodisalar (events)** va **xom protokol trafigi**.
- **Muhim cheklov:** hozirda **MQTT faqat hodisalar (events) uchun** ishlaydi, barcha pozitsiyalar uchun emas. Pozitsiyalarni MQTT orqali uzatish uchun oraliq xizmat kerak bo'ladi (yoki HTTP/Kafka/AMQP formatidan foydalanish).
- **Namuna konfiguratsiya (hodisa uzatish):**
  ```xml
  <entry key='event.forward.type'>mqtt</entry>
  <entry key='event.forward.url'>mqtt://user:pass@IP:1883</entry>
  <entry key='event.forward.topic'>traccar/events</entry>
  ```
- **Biz uchun foyda:** GPS hodisalarini (signalizatsiya, geozona, haddan tashqari tezlik va h.k.) boshqa ichki tizimga — masalan Uzinkass'ning tahlil/hisobot tizimiga — real vaqtda ulash imkonini beradi.

#### 2.5.3 MCP / AI qo'llab-quvvatlash — 6.16

- **Nima:** Model Context Protocol (MCP) orqali Traccar API'larining ko'p qismi **AI yordamchilarga** ochiladi.
- **Biz uchun foyda:** kelajakda AI orqali "falon avtomobil bugun qayerda bo'ldi?", "qaysi mashinalar offline?" kabi so'rovlarni tabiiy tilda bajarish imkoniyati.

#### 2.5.4 Hisobot va boshqaruv imkoniyatlari

| Imkoniyat | Reliz | Batafsil |
|---|---|---|
| **Excel eksport** | 6.0 | Qurilmalar ro'yxatini `.xlsx` ko'rinishida yuklab olish — hisobotlar uchun qulay |
| **Yoqilg'i hisoboti aniqlashtirilgan** | 6.16 | Iste'mol hisobida yoqilg'i to'ldirish/to'kish alohida ajratildi — sarf aniqroq hisoblanadi |
| **Audit hisobotida foydalanuvchi nomi** | 6.16 | Har bir amalni kim bajargani ko'rinadi — 19 ta viloyat foydalanuvchimiz uchun nazorat osonlashadi |
| **Vaqt bo'yicha texxizmat (maintenance)** | 6.0 | Masofaga emas, **vaqt** (muddat) bo'yicha texxizmat eslatmasi sozlash |
| **Geozona "crossed" hodisasi** | 6.16 | Geozonani kesib o'tishni alohida hodisa sifatida qayd etish |
| **Qurilma ulashish muddati** | 6.0 | Qurilmaga vaqtinchalik kirish berib, avtomatik tugash muddati belgilash |

#### 2.5.5 Yangi protokollar — 6.0

- **FleetGuide** (rus GPS kompaniyasi protokoli) va **Valtrack** (Valtrack-V4 uskunalari).
- **Biz uchun foyda:** yangi qurilma turlarini xarid qilsak, qo'llab-quvvatlash doirasi kengaygan.

---

## 3. ⚠️ Yangilashda e'tibor beriladigan jihatlar

| № | Jihat | Izoh |
|---|---|---|
| 1 | **Legacy veb-ilova 6.x dan o'chirilgan** | Bizda `/opt/traccar/legacy` bor, lekin biz `modern` ishlatamiz — muammo yo'q |
| 2 | **DB migratsiyasi** | 5.5 → 6.16 sxema o'zgarishlari Liquibase orqali avtomatik qo'llanadi |
| 3 | **Majburiy zaxira kerak** | 724 GB baza, hozircha zaxira yo'q — bu **birinchi** hal qilinadigan masala |
| 4 | **Katta jadval migratsiyasi** | 1.69 mlrd qatorli jadvalda migratsiya vaqt olishi mumkin — test muhitida sinash tavsiya etiladi |
| 5 | **Sakrash kattaligi** | Bir necha major versiya o'tilgani uchun avval test serverida to'liq sinab ko'rish shart |

---

## 4. Tavsiya

**Yangilash kuchli tavsiya etiladi**, chunki:
1. Xavfsizlik tuzatishlari (critical) — 5.5 zaif.
2. DB batch-yozish (6.14) — bizning eng katta muammomiz (1.69 mlrd qator) uchun to'g'ridan-to'g'ri yechim.
3. UI va umumiy barqarorlik sezilarli yaxshilangan.

**Tartib (tavsiya):**
1. Avval **to'liq DB zaxira** olish (`pg_dump` yoki fizik zaxira).
2. **Test serverida** 5.5 → 6.16 migratsiyasini sinash (nusxa baza bilan).
3. Migratsiya vaqtini o'lchash (1.69 mlrd qator uchun).
4. Ishlab chiqarish serverida reja bo'yicha yangilash (texnik tanaffus bilan).

---

## 5. Manbalar

- [Traccar 6.0 relizi (5.x dan asosiy o'zgarishlar)](https://www.traccar.org/blog/traccar-6-0/)
- [Traccar 6.14 (performans yangilanishi — batch DB)](https://www.traccar.org/blog/traccar-6-14/)
- [Traccar 6.13 (JT/T 1078 video oqim)](https://www.traccar.org/blog/traccar-6-13/)
- [Traccar 6.16 (joriy versiya)](https://www.traccar.org/blog/traccar-6-16/)
- [Barcha relizlar ro'yxati](https://www.traccar.org/blog/category/releases/)
- [Eski versiyalar](https://www.traccar.org/old-versions/)
- [Forwarding (ma'lumot uzatish) hujjati](https://www.traccar.org/forward/)
- [JT/T 1078 video — forum muhokamasi](https://www.traccar.org/forums/topic/traccar-v-6133-jtt-1078-streaming-support-protocol/)

---

*Ushbu hujjat ochiq manbalardagi Traccar rasmiy reliz eslatmalari asosida tuzildi.*
