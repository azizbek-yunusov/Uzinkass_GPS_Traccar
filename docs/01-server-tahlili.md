# GPS Kuzatuv Serveri — Texnik Hujjat

> **Hujjat sanasi:** 2026-10-02
> **Tahlil turi:** Faqat o'qish (read-only) — serverda hech narsa o'zgartirilmadi
> **Server manzili:** `root@10.0.57.102:49001` (SSH)

---

## 1. Umumiy xulosa

Bu server **Traccar** ochiq kodli GPS kuzatuv (GPS tracking) platformasini ishga tushiradi. Tizim O'zbekiston bo'ylab transport vositalarini (davlat raqamli avtomobillar) real vaqt rejimida kuzatish uchun ishlatiladi. Foydalanuvchilar viloyatlar bo'yicha ajratilgan (Farg'ona, Namangan, Buxoro, Navoiy, Jizzax va boshqalar).

Tizim **jonli va faol** — tahlil paytida ham GPS qurilmalaridan real vaqtda ma'lumot qabul qilinmoqda (oxirgi pozitsiya: `2026-10-02 09:33:54`).

| Ko'rsatkich | Qiymat |
|---|---|
| Platforma | Traccar **5.5** |
| Qurilmalar soni | **230** ta (63 online, 60 offline, 107 noma'lum) |
| Foydalanuvchilar | **19** ta (1 administrator) |
| Pozitsiya yozuvlari | **~1.69 milliard** qator (tc_positions) |
| Baza hajmi | **724 GB** (asosan GPS treklar) |
| Server ishlash muddati | 303 kun (uzluksiz) |

---

## 2. Server infratuzilmasi

### 2.1 Apparat va OS

| Parametr | Qiymat |
|---|---|
| Hostname | `gps` |
| OS | Ubuntu 22.04 LTS (Jammy), yadro `5.15.0-161-generic` |
| CPU | 16 yadro |
| RAM | 15 GiB (hozirda ~1.5 GiB band, 13 GiB cache) |
| Swap | 4 GiB |
| Arxitektura | x86_64 |

### 2.2 Disk

| Fayl tizimi | Tur | Hajm | Band | Bo'sh | Foiz | Ulanish nuqtasi |
|---|---|---|---|---|---|---|
| `/dev/mapper/ubuntu--vg-ubuntu--lv` | ext4 | 914 GB | 737 GB | 139 GB | **85%** | `/` |
| `/dev/sda2` | ext4 | 2.0 GB | 256 MB | 1.6 GB | 14% | `/boot` |
| `/dev/sda1` | vfat | 1.1 GB | 6.1 MB | 1.1 GB | 1% | `/boot/efi` |

> ⚠️ Asosiy disk **85% to'lgan**, bo'sh joy ~139 GB. Baza doimiy o'sib boryapti (724 GB), shuning uchun disk to'lishi real xavf.

### 2.3 Foydalanuvchi akkauntlari (OS darajasida)

- `root` — asosiy administrator
- `gps` (uid 1000) — `/home/gps`
- `postgres` (uid 113) — ma'lumotlar bazasi xizmati

### 2.4 Oxirgi kirishlar (SSH)

Barcha oxirgi kirishlar `root` orqali, asosan `10.0.57.111` IP manzilidan amalga oshirilgan.

---

## 3. Ishlab turgan xizmatlar

| Xizmat | Holat | Tavsif |
|---|---|---|
| `traccar.service` | ✅ running | Asosiy GPS platforma (Java) |
| `postgresql@14-main` | ✅ running | PostgreSQL 14 ma'lumotlar bazasi |
| `nginx.service` | ✅ running | Web-server (lekin faol konfiguratsiyasiz, quyida) |
| `ModemManager`, `udisks2` | ✅ running | Tizim xizmatlari |

### 3.1 Ochiq portlar (tinglayotgan)

| Port | Xizmat | Izoh |
|---|---|---|
| **49001** | SSH (sshd) | Boshqaruv uchun kirish porti |
| **49002** | Traccar Web UI (Java) | Veb-interfeys (`web.port`) |
| **5432** | PostgreSQL | DB (barcha interfeyslarda `0.0.0.0` — e'tibor bering) |
| **49027** | Teltonika protokoli (Java) | Qurilmalarning asosiy ma'lumot porti |
| **5001–5200+** | Turli GPS protokollari | Traccar har bir qurilma protokoli uchun alohida port ochadi |

---

## 4. Traccar ilovasi

### 4.1 Joylashuv va ishga tushirish

- **O'rnatilgan katalog:** `/opt/traccar`
- **Asosiy fayl:** `/opt/traccar/tracker-server.jar` (versiya 5.5)
- **O'rnatilgan JRE:** `/opt/traccar/jre` (tizimdan alohida)
- **systemd xizmati:** `/etc/systemd/system/traccar.service`

```ini
[Service]
Type=simple
WorkingDirectory=/opt/traccar
ExecStart=/opt/traccar/jre/bin/java -jar tracker-server.jar conf/traccar.xml
WatchdogSec=600
Restart=on-failure
RestartSec=10
```

Jarayon `root` ostida ishlaydi (Aug17 dan beri, ~1 kun 02 soat CPU vaqti).

### 4.2 Katalog tuzilmasi (`/opt/traccar`)

| Katalog | Vazifasi |
|---|---|
| `conf/` | Konfiguratsiya fayllari (`traccar.xml`, `default.xml`) |
| `data/` | H2 ma'lumotlar (hozir ishlatilmaydi — PostgreSQL faol) |
| `jre/` | Birga keluvchi Java muhiti |
| `lib/` | Java kutubxonalari |
| `logs/` | Loglar (`tracker-server.log`) |
| `media/` | Media fayllar (`media.path`) |
| `modern/` | Yangi (React) veb-interfeys — `web.path` |
| `legacy/` | Eski veb-interfeys |
| `schema/` | DB sxema o'zgartirishlari (Liquibase changelog) |
| `templates/` | Bildirishnoma shablonlari |

### 4.3 Konfiguratsiya

**Asosiy konfiguratsiya:** `/opt/traccar/conf/traccar.xml` (foydalanuvchi sozlamalari)
**Standart konfiguratsiya:** `/opt/traccar/conf/default.xml` (o'zgartirmaslik tavsiya etiladi)

Muhim sozlamalar (`default.xml`):

| Kalit | Qiymat | Izoh |
|---|---|---|
| `web.port` | `49002` | Veb-interfeys porti |
| `web.path` | `./modern` | Yangi React interfeysi ishlatiladi |
| `web.sanitize` | `false` | HTML tozalash o'chirilgan |
| `geocoder.enable` | `true` | Manzilni aniqlash yoqilgan |
| `geocoder.type` | `nominatim` | LocationIQ orqali |
| `geocoder.url` | `https://us1.locationiq.com/v1/reverse.php` | Tashqi geokoder xizmati |
| `geocoder.key` | *(API kalit sozlangan)* | LocationIQ API kaliti |
| `logger.level` | `off` | ⚠️ Loglar **o'chirilgan** |
| `filter.enable` | `true` | Pozitsiya filtri yoqilgan |
| `filter.future` | `86400` | Kelajakdagi (1 kun) yozuvlarni rad etish |
| `notificator.types` | `web,mail` | Bildirishnomalar: veb va email |
| `status.timeout` | `60` | 60s javobsiz = offline |

> **Eslatma:** `logger.level=off` bo'lgani uchun `tracker-server.log` bo'sh. Nosozliklarni tekshirish qiyinlashadi.

---

## 5. Ma'lumotlar bazasi (PostgreSQL)

### 5.1 Ulanish sozlamalari

Traccar PostgreSQL'ga quyidagicha ulanadi (`conf/traccar.xml`):

| Parametr | Qiymat |
|---|---|
| Driver | `org.postgresql.Driver` |
| URL | `jdbc:postgresql://127.0.0.1:5432/gps` |
| Baza nomi | `gps` |
| Foydalanuvchi | `postgres` |
| Parol | *(traccar.xml ichida ochiq matnda saqlangan — xavfsizlik bo'limiga qarang)* |

> H2 ma'lumotlar bazasi konfiguratsiyada bor, lekin izohga olingan (ishlatilmaydi).

### 5.2 PostgreSQL versiyasi va sozlamalari

- **Versiya:** PostgreSQL 14.24 (Ubuntu)
- **Klaster:** `14-main`

| Sozlama | Qiymat | Baho |
|---|---|---|
| `shared_buffers` | 128 MB | ⚠️ Standart — 724 GB baza uchun juda past |
| `effective_cache_size` | 4 GB | Pastroq (15 GB RAM uchun ~10 GB tavsiya) |
| `work_mem` | 4 MB | Standart |
| `maintenance_work_mem` | 64 MB | Past |
| `max_connections` | 100 | Standart |

> PostgreSQL sozlamalari **standart holatda** — bunday katta baza uchun maxsus sozlanmagan. Bu so'rovlar tezligiga ta'sir qiladi.

### 5.3 Bazalar

| Baza | Egasi | Hajm |
|---|---|---|
| **gps** | postgres | **724 GB** |
| postgres | postgres | 8.5 MB |
| template0/template1 | postgres | 8.5 MB |

### 5.4 Jadvallar (`gps` bazasi, 44 ta jadval)

Traccar standart sxemasi — barcha jadvallar `tc_` prefiksi bilan. Eng katta jadvallar:

| Jadval | Hajm | Taxminiy qatorlar | Vazifasi |
|---|---|---|---|
| **tc_positions** | **721 GB** | **~1,688,456,966** (1.69 mlrd) | GPS pozitsiya treklari |
| tc_events | 2998 MB | ~23,836,751 | Hodisalar (signal, geozona va h.k.) |
| tc_devices | 17 MB | 230 | Kuzatiladigan qurilmalar |
| tc_statistics | 232 kB | 1308 | Statistika |
| tc_users | 112 kB | 19 | Foydalanuvchilar |
| tc_servers | 64 kB | 1 | Server sozlamalari |

Qolgan jadvallar: `tc_geofences` (geozonalar), `tc_drivers`, `tc_commands`, `tc_notifications`, `tc_calendars`, `tc_maintenances`, `tc_orders`, hamda bog'lovchi (`tc_user_device`, `tc_group_*`, `tc_device_*`) jadvallar.

**`tc_positions` indekslari:**
- `tc_positions_pkey` — `id` bo'yicha (unikal)
- `position_deviceid_fixtime` — `(deviceid, fixtime)` bo'yicha
- Eng katta `id`: `1,726,803,434`

### 5.5 Foydalanuvchilar (ilova darajasida, `tc_users`)

19 ta foydalanuvchi mavjud. 1-chisi — **Administrator** (admin huquqli). Qolganlari viloyatlar/bo'linmalar nomidan:

`Djizak`, `A-Rasulkulov`, `Andijan`, `Fergona`, `Navoiy`, `Xorezm`, `Nukus`, `Buxara`, `Karshi`, `Namangan`, `Samarkand`, `Syrxandaryo`, `Sirdaryo`, `AB-Yusupov`, `Dejurka-RIX`, `SH-Turgunov`, `A.Sobirov`, `A.Saidkarimov` va boshqalar. Barchasi faol (disabled = false).

### 5.6 Qurilmalar (`tc_devices`)

230 ta qurilma — har biri transport vositasi. Nomlar davlat raqami + viloyat formatida, masalan:
- `40 836 GBA / 2052 ФЕРГАНА`
- `50 047 FBA / 2285 НАМАНГАН`
- `80 146 PBА / 3459 БУХАРА`
- `85 142 ТАА / 3656 НАВОИЙ`

Har bir qurilmada `uniqueid` (IMEI, masalan `356173065281706`) mavjud. Ko'pchiligi **Teltonika** uskunalari (port 49027 orqali ulanadi).

---

## 6. Web-server (Nginx)

- Nginx **ishlab turibdi**, lekin `/etc/nginx/sites-enabled/traccar` fayli **bo'sh** — faol `server {}` bloki yo'q.
- `nginx -T` natijasida hech qanday virtual host topilmadi.
- `/var/www/` ichida: `html`, `html1` (standart Debian sahifalari), `traccar` (eski statik React build, Jan 2023), `cache`.

> **Xulosa:** Nginx hozir reverse-proxy sifatida ishlatilmayapti. Foydalanuvchilar Traccar veb-interfeysiga **to'g'ridan-to'g'ri 49002-port** orqali kirishadi. `/var/www/traccar` eski/foydalanilmaydigan nusxa ko'rinadi.

---

## 7. Zaxira nusxalash va monitoring

- **DB zaxira nusxasi topilmadi.** Butun tizim bo'ylab (`.sql`, `.dump`) qidiruvda PostgreSQL tizim fayllaridan boshqa foydalanuvchi zaxira nusxasi aniqlanmadi.
- `pgAgent` o'rnatilgan (`pgagent.pga_job`), lekin **sozlangan vazifa yo'q** (0 ta job).
- `crontab` (root) bo'sh. `/etc/cron.d` va `/etc/cron.daily` da faqat tizim vazifalari (logrotate, apt, sysstat).
- `/var/backups` da faqat tizim fayllari (`alternatives.tar`), DB zaxirasi emas.

> ⚠️ **724 GB jonli ma'lumotlar bazasi uchun avtomatik zaxira nusxalash sozlanmagan — bu eng jiddiy xavf.**

---

## 8. Aniqlangan xavflar va tavsiyalar

> Quyidagilar faqat kuzatuv va tavsiyalar — serverda hech qanday o'zgartirish kiritilmagan.

| № | Xavf | Darajasi | Tavsiya |
|---|---|---|---|
| 1 | DB zaxira nusxasi yo'q (724 GB) | 🔴 Yuqori | `pg_dump` yoki PITR (WAL arxivlash) orqali muntazam zaxira sozlash; tashqi diskka/serverga nusxalash |
| 2 | Disk 85% to'lgan, baza o'smoqda | 🔴 Yuqori | Eski pozitsiyalarni arxivlash/tozalash siyosati; disk hajmini kengaytirish; `tc_positions` partitsiyalash |
| 3 | DB paroli `traccar.xml` da ochiq matnda | 🟠 O'rta | Fayl ruxsatlarini cheklash; parolni almashtirishni ko'rib chiqish |
| 4 | PostgreSQL 5432-portda `0.0.0.0` da tinglaydi | 🟠 O'rta | Tashqi kirish kerak bo'lmasa, `127.0.0.1` ga cheklash yoki firewall qo'yish |
| 5 | Traccar `root` ostida ishlaydi | 🟠 O'rta | Alohida kam huquqli foydalanuvchi ostida ishga tushirish |
| 6 | PostgreSQL standart sozlamalar (shared_buffers 128MB) | 🟡 Past | Katta baza uchun `shared_buffers`, `effective_cache_size` ni tunning qilish |
| 7 | `logger.level=off` | 🟡 Past | Nosozliklarni tekshirish uchun kamida `warning`/`info` yoqish foydali |
| 8 | Traccar 5.5 — eski versiya | 🟡 Past | Yangilashni rejalashtirish (xavfsizlik yangilanishlari uchun) |

---

## 9. Tezkor ma'lumotnoma

```
SSH:            root@10.0.57.102:49001
Platforma:      Traccar 5.5 (/opt/traccar)
Veb UI:         http://10.0.57.102:49002 (modern/React)
Asosiy config:  /opt/traccar/conf/traccar.xml
Standart config:/opt/traccar/conf/default.xml
Xizmat:         systemctl status traccar
Loglar:         /opt/traccar/logs/tracker-server.log (hozir o'chirilgan)

DB:             PostgreSQL 14.24, baza "gps" (724 GB)
DB ulanish:     psql -h 127.0.0.1 -U postgres -d gps
Asosiy jadval:  tc_positions (~1.69 mlrd qator)
Qurilma porti:  49027 (Teltonika)
```

---

*Ushbu hujjat serverni faqat o'qish rejimida tahlil qilish natijasida tuzildi. Hech qanday fayl, konfiguratsiya yoki ma'lumot o'zgartirilmagan.*
