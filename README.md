# Uzinkass GPS Traccar

O'zbekiston bo'ylab transport vositalarini real vaqtda kuzatish uchun **Traccar** asosidagi GPS kuzatuv loyihasi.

## Struktura

```
Uzinkass_GPS_Traccar/
├── docs/        # loyiha hujjatlari (tahlil, versiya taqqoslash)
├── deploy/      # sozlash va deployment fayllari (config namunasi, h.k.)
├── custom/      # Uzinkass'ga xos qo'shimcha kod/kengaytmalar
└── traccar/     # Traccar manba kodi — git SUBMODULE (v6.16.0)
```

> **traccar/** — bu [traccar/traccar](https://github.com/traccar/traccar) rasmiy reposiga bog'langan **submodule** (`v6.16.0` tegida). Bu upstream yangilanishlarni oson olish imkonini beradi va o'z kodingizni Traccar kodidan ajratib turadi.

## Klon qilish (submodule bilan)

```bash
git clone --recursive https://github.com/azizbek-yunusov/Uzinkass_GPS_Traccar.git
# yoki clone qilib bo'lgach:
git submodule update --init --recursive
```

## Traccar'ni yangilash (upstream)

```bash
cd traccar
git fetch --tags
git checkout v6.17.0     # yangi teg chiqqanda
cd ..
git add traccar
git commit -m "Traccar v6.17.0 ga yangilandi"
```

## Build

```bash
cd traccar
./gradlew assemble        # natija: target/tracker-server.jar
```

## Hujjatlar

- [`docs/01-server-tahlili.md`](docs/01-server-tahlili.md) — ishlab turgan server, konfiguratsiya va DB tahlili
- [`docs/02-versiya-taqqoslash.md`](docs/02-versiya-taqqoslash.md) — Traccar 5.5 → 6.16 taqqoslash va afzalliklar

## Sozlash

`deploy/traccar.xml.example` ni `traccar.xml` deb nusxa qilib, DB parolini to'ldiring. Haqiqiy parolli fayl `.gitignore` orqali repoga tushmaydi.

## Litsenziya

Traccar — Apache License 2.0. Qarang: `traccar/LICENSE.txt`.
