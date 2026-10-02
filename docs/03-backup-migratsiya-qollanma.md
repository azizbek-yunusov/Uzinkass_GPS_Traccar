# Backup va Migratsiya Qo'llanmasi (Eski → Yangi Server)

> **Hujjat sanasi:** 2026-10-02
> **Maqsad:** Eski serverdan (Traccar 5.5, PostgreSQL 14, 724 GB baza) zaxira olish, ko'chirish va yangi serverga joylash
> **Bog'liq:** [`01-server-tahlili.md`](01-server-tahlili.md), [`02-versiya-taqqoslash.md`](02-versiya-taqqoslash.md)

> 🔑 **Parollar haqida:** Quyidagi buyruqlarda `<DB_PAROL>` deb yozilgan joyga **haqiqiy DB parolini** qo'ying (u eski serverдagi `/opt/traccar/conf/traccar.xml` faylida bor). Haqiqiy parolni **hech qachon** git repoga yoki hujjatга yozmang.

---

## ⚠️ Avval o'qing — muhim ogohlantirishlar

1. **Baza juda katta — 724 GB** (`tc_positions` ~1.69 mlrd qator). Backup va restore **soatlab** davom etishi mumkin.
2. **Eski serverда disk 85% to'lgan** (atigi ~139 GB bo'sh). Shu sababli **724 GB zaxirani eski serverning o'ziga saqlab bo'lmaydi** — zaxirani to'g'ridan-to'g'ri yangi serverga yoki tashqi diskka oqizish kerak.
3. **Hech narsani o'chirmang** — migratsiya to'liq tekshirilib, yangi server ishlaganiga ishonch hosil qilmaguningizcha eski server tegilmasin.
4. Ishni **kam yuklamali vaqtda** (tunda) qiling — downtime bo'ladi.

---

## 1. Strategiyani tanlang

Ikki yo'l bor:

| | A: Bir xil ko'chirish (5.5 → 5.5) | B: Ko'chirish + yangilash (5.5 → 6.16) |
|---|---|---|
| Murakkablik | Oson | O'rtacha |
| Xavf | Past | O'rtacha (sxema migratsiyasi) |
| Natija | Xuddi eskidek | Yangi imkoniyatlar + xavfsizlik |

> **Tavsiya:** Avval **A** bilan yangi serverni ishga tushiring (toza ko'chirish). Keyin yangi serverда **B** (5.5 → 6.16 yangilash) ni alohida qadam sifatida bajaring. Shunda muammo bo'lsa, qaysi bosqichda ekani aniq bo'ladi.

Bu qo'llanma: **A (ko'chirish)** + oxirida **B (yangilash)** varianti.

---

## 2. Yangi serverni tayyorlash

Yangi serverда (Ubuntu 22.04 tavsiya etiladi) quyidagilar bo'lishi kerak:

- **Disk:** kamida **1 TB bo'sh** (baza 724 GB + o'sish zaxirasi). Ko'proq bo'lgani ma'qul.
- **RAM:** kamida 16 GB (eski serverдek), katta baza uchun ko'proq yaxshi.
- PostgreSQL o'rnatish (eski bilan **bir xil major versiya — 14**):

```bash
sudo apt update
sudo apt install -y postgresql-14 postgresql-client-14
sudo systemctl enable --now postgresql
```

> ⚠️ **Muhim:** Agar yangi serverда boshqa PostgreSQL versiyasi (masalan 16) bo'lsa, fizik zaxira (file-level) **ishlamaydi** — faqat `pg_dump` (logik) ishlaydi. Bu qo'llanma `pg_dump` dan foydalanadi, shuning uchun versiya farqi muammo emas, lekin bir xil versiya baribir xavfsizroq.

---

## 3. Eski serverdan zaxira olish (`pg_dump`)

`pg_dump` — baza faol ishlab turganda ham xavfsiz (ichki holatni muzlatadi). Traccar'ni to'xtatish **shart emas**, lekin eng toza nusxa uchun tavsiya etiladi.

> Parolni buyruqqa yozmaslik uchun `~/.pgpass` faylidan foydalaning yoki buyruqdan oldin `read -s PGPASSWORD; export PGPASSWORD` orqali interaktiv kiriting. Quyida soddalik uchun `PGPASSWORD` muhit o'zgaruvchisi ko'rsatilgan — undan oldin uni o'rnating.

### 3.1 Eng yaxshi usul — to'g'ridan-to'g'ri yangi serverga oqizish (disk tejaydi)

Eski serverда disk kam bo'lgani uchun, zaxirani **oraliq faylsiz** to'g'ridan-to'g'ri yangi serverga quvur (pipe) orqali yuboramiz.

**Eski serverда** (SSH orqali kiring):

```bash
# Parolni avval o'rnating (terminal tarixiga tushmasligi uchun -s bilan):
read -s PGPASSWORD; export PGPASSWORD

# Eski serverdan -> yangi serverga to'g'ridan-to'g'ri (siqilgan holda)
pg_dump -h 127.0.0.1 -U postgres -d gps -Fc -Z6 \
  | ssh -p <YANGI_PORT> root@<YANGI_SERVER_IP> 'cat > /var/lib/postgresql/gps_backup.dump'
```

- `-Fc` — custom format (siqilgan, parallel restore imkoniyati bilan)
- `-Z6` — siqish darajasi (GPS raqamli ma'lumotlar yaxshi siqiladi — 724 GB ~100-200 GB gacha kichrayishi mumkin)
- `| ssh ... 'cat > ...'` — oraliq faylsiz to'g'ridan-to'g'ri yangi serverga yozadi

### 3.2 Muqobil — tashqi diskka saqlash

Agar yangi serverga to'g'ridan-to'g'ri oqizish qiyin bo'lsa, tashqi USB/tarmoq diskka:

```bash
read -s PGPASSWORD; export PGPASSWORD
pg_dump -h 127.0.0.1 -U postgres -d gps -Fc -Z6 -f /mnt/external/gps_backup.dump
```

> `/mnt/external` — tashqi disk ulangan joy. Eski serverning ichki diskiga (139 GB bo'sh) 724 GB sig'maydi!

### 3.3 Tezroq variant — parallel dump (katta baza uchun)

Katalog formati (`-Fd`) bir nechta ipda (parallel) ishlaydi, tezroq:

```bash
read -s PGPASSWORD; export PGPASSWORD
pg_dump -h 127.0.0.1 -U postgres -d gps -Fd -j 4 -Z6 -f /mnt/external/gps_dump_dir
```

- `-Fd` — directory format, `-j 4` — 4 ta parallel ip (CPU 16 yadroli, ko'paytirsa bo'ladi)
- Keyin bu papkani `rsync` bilan yangi serverga ko'chirasiz

### 3.4 Progress (jarayonni kuzatish)

Katta dump uzoq davom etadi. Alohida terminalда kuzating:

```bash
# Dump hajmi o'syaptimi?
watch -n 10 'ls -lh /mnt/external/gps_backup.dump'
```

### 3.5 O'z kompyuteringizga olish (pull / scp) — arxiv nusxa uchun tavsiya

Zaxirani **o'z kompyuteringizga** olish mumkin — ayniqsa **xavfsizlik (arxiv) nusxasi** sifatida juda foydali (hozirda hech qanday backup yo'q!). Hatto migratsiyadan oldin **hoziroq** bitta nusxa olib qo'yish tavsiya etiladi.

**Amaliy jihatlar:**

| Jihat | Izoh |
|---|---|
| Disk joyi | Kompyuterда ~100–200 GB bo'sh kerak (siqilgan dump) |
| Vaqt | Tarmoq tezligiga qarab soatlab |
| Ikki marta uzatish | Server → kompyuter → yangi server = sekinroq (server↔server to'g'ridan-to'g'ri tezroq) |
| Uzilish | `scp` uzilsa noldan boshlanadi; `rsync` esa davom ettiradi |

> ⚠️ Eski serverда atigi ~139 GB bo'sh. Shuning uchun serverда katta dump fayl yaratmay, uni **to'g'ridan-to'g'ri kompyuteringizga oqizish** (pull) eng maqbul.

**A) To'g'ridan-to'g'ri oqizish (serverда fayl yaratmaydi) — tavsiya etiladi:**

```bash
# O'z kompyuteringizdan ishga tushiring:
ssh -p 49001 root@10.0.57.102 \
  "PGPASSWORD='<DB_PAROL>' pg_dump -h 127.0.0.1 -U postgres -d gps -Fc -Z6" \
  > gps_backup.dump
```

**B) Oddiy scp (avval serverда dump fayl bo'lishi kerak — tashqi diskда):**

```bash
scp -P 49001 root@10.0.57.102:/mnt/external/gps_backup.dump .
```

**C) rsync (uzilsa davom ettiradi — katta fayl uchun eng ishonchli):**

```bash
rsync -avP -e 'ssh -p 49001' root@10.0.57.102:/mnt/external/gps_backup.dump .
```

> **Qaysi birini qachon:**
> - **Arxiv/xavfsizlik nusxasi** → kompyuterga olish (A yoki C) — a'lo
> - **Yangi serverga ko'chirish** → agar ikkala server bir-biriga ulana olsa, **server→server to'g'ridan-to'g'ri** (3.1) tezroq va ishonchliroq
> - **Uzilishdan xavfsirsangiz** → `scp` o'rniga `rsync -avP` (resume)

---

## 4. Traccar sozlama va fayllarini ko'chirish

Baza bilan birga quyidagi fayllarni ham yangi serverga ko'chiring (eski serverда):

```bash
# Konfiguratsiya, media, shablonlar
rsync -avz -e 'ssh -p <YANGI_PORT>' \
  /opt/traccar/conf \
  /opt/traccar/media \
  /opt/traccar/templates \
  root@<YANGI_SERVER_IP>:/tmp/traccar_backup/
```

> **Diqqat:** `/opt/traccar/conf/traccar.xml` ichida **DB parol** bor. Uni ko'chirgandan keyin yangi serverда ham o'sha fayl ishlatiladi (yoki parolni yangisiga almashtirasiz — quyida).

Eng muhim fayl — **`/opt/traccar/conf/traccar.xml`** (barcha sozlamalaringiz). Agar faqat shuni ko'chirsangiz ham yetarli, chunki boshqa narsalar Traccar o'rnatuvchisidan keladi.

---

## 5. Yangi serverga joylash (restore)

### 5.1 Bazani yaratish

**Yangi serverда:**

```bash
sudo -u postgres psql
```
```sql
-- Baza va parol (eski bilan bir xil parol ishlatsangiz, Traccar config o'zgarmaydi)
CREATE DATABASE gps;
ALTER USER postgres WITH PASSWORD '<DB_PAROL>';
\q
```

### 5.2 Restore qilish

```bash
read -s PGPASSWORD; export PGPASSWORD

# custom format (3.1 / 3.2) uchun:
pg_restore -h 127.0.0.1 -U postgres -d gps -j 4 --no-owner /var/lib/postgresql/gps_backup.dump

# directory format (3.3) uchun:
pg_restore -h 127.0.0.1 -U postgres -d gps -j 4 --no-owner /path/to/gps_dump_dir
```

- `-j 4` — parallel restore (katta baza uchun ancha tez)
- `--no-owner` — egalik muammolarini oldini oladi
- Bu ham **soatlab** davom etishi mumkin (1.69 mlrd qator + indekslarni qayta qurish)

### 5.3 Traccar o'rnatish

```bash
# Rasmiy o'rnatuvchini yuklab oling (5.5 uchun — eski versiyalardan,
# yoki to'g'ridan 6.16 uchun — traccar.org dan)
cd /tmp
wget https://www.traccar.org/download/traccar-linux-64-latest.zip
unzip traccar-linux-64-latest.zip
sudo ./traccar.run
```

> 5.5 ni aynan o'rnatish uchun: https://www.traccar.org/old-versions/

### 5.4 Konfiguratsiyani qo'yish

```bash
# Ko'chirilgan traccar.xml ni joyiga qo'ying
sudo cp /tmp/traccar_backup/conf/traccar.xml /opt/traccar/conf/traccar.xml

# DB URL ichida localhost to'g'riligini tekshiring:
#   jdbc:postgresql://127.0.0.1:5432/gps
sudo nano /opt/traccar/conf/traccar.xml
```

### 5.5 Ishga tushirish

```bash
sudo systemctl enable --now traccar
sudo systemctl status traccar
```

---

## 6. Tekshirish (verification)

Yangi server to'g'ri ishlayotganini tasdiqlang:

```bash
# 1. Traccar xizmati ishlayaptimi?
systemctl status traccar

# 2. Web ochiladimi? (brauzerда)
#    http://<YANGI_SERVER_IP>:49002

# 3. Baza qatorlari soni eski bilan mosmi?
read -s PGPASSWORD; export PGPASSWORD
psql -h 127.0.0.1 -U postgres -d gps -c \
  "SELECT (SELECT count(*) FROM tc_devices) devices, (SELECT count(*) FROM tc_users) users;"
#    -> devices: 230, users: 19 bo'lishi kerak

# 4. Eng yangi pozitsiya vaqtini tekshiring (oxirgi ma'lumot ko'chganmi?)
psql -h 127.0.0.1 -U postgres -d gps -t -c \
  "SELECT max(servertime) FROM tc_positions;"
```

- Web interfeysga **admin** foydalanuvchi bilan kiring, qurilmalar xaritada ko'rinishини tekshiring.

---

## 7. GPS qurilmalarni yangi serverga yo'naltirish

Baza ko'chgach, **qurilmalar hali eski serverga ma'lumot yuboradi**. Ularni yangi serverga o'tkazish kerak:

- **Variant 1 — IP o'zgartirish:** Qurilmalar SMS buyrug'i orqali yangi server IP/portga sozlanadi (har bir qurilma modeli bo'yicha).
- **Variant 2 — Eski IP'ni yangi serverga ko'chirish:** Agar imkon bo'lsa, eski server IP manzilini yangi serverga bersangiz, qurilmalar o'zgarishsiz ishlaydi (eng oson).
- **Variant 3 — DNS:** Agar qurilmalar domen orqali ulansa, DNS'ni yangi serverga yo'naltiring.

> Bu bosqichda qisqa "ikkilik" davr bo'lishi mumkin — ba'zi qurilmalar eski, ba'zisi yangi serverga yuboradi. Shuning uchun **o'tishni tez bajaring** yoki o'tish lahzasidagi ma'lumotlarni keyin birlashtirishni rejalashtiring.

---

## 8. Versiyani yangilash (5.5 → 6.16) — ixtiyoriy, keyingi bosqich

Yangi server 5.5 da barqaror ishlagach:

1. **Avval yangi serverда DB ni yana zaxiralang** (6.16 migratsiyasi sxemani o'zgartiradi).
2. 6.16 o'rnatuvchisini yuklab, ustiga o'rnating — Liquibase sxemani avtomatik yangilaydi.
3. Birinchi ishga tushishda migratsiya **uzoq davom etishi mumkin** (1.69 mlrd qator) — sabr qiling, to'xtatmang.
4. Tekshiring (6-bo'lim).

Batafsil afzalliklar: [`02-versiya-taqqoslash.md`](02-versiya-taqqoslash.md).

---

## 9. Xavfsizlik va keyingi yaxshilashlar (migratsiyadan keyin)

Yangi serverga o'tganда [`01-server-tahlili.md`](01-server-tahlili.md) dagi xavflarni ham hal qiling:

- ✅ **Avtomatik kunlik backup** sozlang (`cron` + `pg_dump`) — eng muhim, eski serverда yo'q edi
- ✅ PostgreSQL 5432-portni `127.0.0.1` ga cheklang (firewall)
- ✅ DB parolini kuchliroqqa almashtiring
- ✅ Traccar'ni alohida (root bo'lmagan) foydalanuvchi ostida ishga tushiring
- ✅ PostgreSQL sozlamalarini katta baza uchun tunning qiling (`shared_buffers` va h.k.)

### Namuna: kunlik avtomatik backup (yangi serverда)

Parolni cron faylga **ochiq yozmang** — `postgres` foydalanuvchining `~/.pgpass` faylidan foydalaning (`chmod 600`):

```bash
# ~postgres/.pgpass  (format: host:port:db:user:parol)
127.0.0.1:5432:gps:postgres:<DB_PAROL>
```

```bash
# /etc/cron.d/traccar-backup
0 2 * * * postgres pg_dump -h 127.0.0.1 -U postgres -d gps -Fc -Z6 -f /backup/gps_$(date +\%Y\%m\%d).dump && find /backup -name 'gps_*.dump' -mtime +7 -delete
```
> Har kuni soat 02:00 da zaxira oladi, 7 kundan eski nusxalarni o'chiradi. `/backup` alohida diskда bo'lgani ma'qul.

---

## 10. Qisqa yo'l-xarita (checklist)

- [ ] Yangi serverда disk (≥1 TB) va PostgreSQL 14 tayyor
- [ ] Eski serverdan `pg_dump` (to'g'ridan-to'g'ri yangi serverga yoki tashqi diskka)
- [ ] `traccar.xml` va `media` ko'chirildi
- [ ] Yangi serverда `gps` bazasi yaratildi, parol qo'yildi
- [ ] `pg_restore` bajarildi
- [ ] Traccar o'rnatildi, `traccar.xml` qo'yildi, ishga tushdi
- [ ] Tekshirildi (web, qurilmalar soni, oxirgi pozitsiya)
- [ ] Qurilmalar yangi serverga yo'naltirildi
- [ ] (ixtiyoriy) 6.16 ga yangilandi
- [ ] Avtomatik backup sozlandi
- [ ] Eski server bir muddat zaxira sifatida saqlanadi (darhol o'chirilmaydi)

---

*Ushbu qo'llanma `01-server-tahlili.md` dagi real server tahlili asosida tuzildi. Buyruqlardagi `<YANGI_SERVER_IP>`, `<YANGI_PORT>` va `<DB_PAROL>` ni o'z qiymatlaringiz bilan almashtiring.*
