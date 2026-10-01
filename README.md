# SmartPark Residence — Digital Parking Ecosystem

Turar-joy majmualari uchun avtoturargohlarni adolatli taqsimlash, IoT
(kameralar, shlagbaumlar, sensorlar) bilan integratsiya, mehmon
ruxsatnomalari, to'lovlar va nizolarni hal qilish tizimi.

## Arxitektura

- **Backend**: Django 5 + DRF, Modular Monolith (`apps/users`, `complexes`,
  `parking`, `iot_integration`, `guest_access`, `billing`, `disputes`,
  `notifications`), Service Layer pattern (`services.py` har bir appda).
- **Real vaqt**: Django Channels + Redis (`ws/parking/<complex_id>/`,
  `ws/guard-feed/<complex_id>/`).
- **Fon vazifalari**: Celery + Celery Beat (mehmon ruxsatnomalarini
  muddati tugashi bilan bekor qilish, oylik rotatsiya va h.k.).
- **Ma'lumotlar bazasi**: PostgreSQL 16 (yoki lokal demo uchun SQLite, `USE_SQLITE=1`).
- **IoT**: ESP32 firmware (`firmware/`) + Mosquitto MQTT broker.
- **Reverse proxy**: Nginx (REST API + WebSocket upgrade).

```
smartpark_backend/
├── apps/                  # 8 ta domen app (models/services/views/urls)
├── core/utils/mqtt.py     # MQTT publish helper (barrier ochish)
├── smartpark_backend/     # settings, urls, asgi (Channels), celery
├── firmware/              # ESP32 .ino kodlari
├── nginx/nginx.conf
├── mosquitto/config/mosquitto.conf
├── docker-compose.yml
├── Dockerfile
└── requirements.txt
```

## Eng tez yo'l — Docker'siz, SQLite bilan (Windows/PyCharm uchun tavsiya etiladi)

Docker o'rnatish yoki eski Windows versiyasida muammo chiqsa, bu yo'l orqali
5 daqiqada ishga tushirasiz — PostgreSQL, Redis, PostGIS, Docker kerak emas:

```bash
cd smartpark_backend
python -m venv venv
venv\Scripts\activate          # Windows; macOS/Linux: source venv/bin/activate

pip install -r requirements.txt

set USE_SQLITE=1               # Windows cmd; PowerShell: $env:USE_SQLITE="1"; bash: export USE_SQLITE=1

python manage.py migrate
python manage.py seed_demo_data
python manage.py runserver
```

Brauzerda: **http://127.0.0.1:8000/admin/** va **http://127.0.0.1:8000/api/docs/**

`frontend/index.html`ni ko'rish uchun (alohida terminalda):
```bash
cd frontend
python -m http.server 5500
```
`http://localhost:5500` oching, `resident`/`resident123` bilan kiring.

**Eslatma**: bu rejimda jonli WebSocket yangilanishlari (`ws/parking/...`)
ishlamaydi, chunki `runserver` oddiy HTTP server, Daphne emas — lekin barcha
REST API (login, joylar ro'yxati, joy olish, mehmon ruxsatnomasi) to'liq
ishlaydi. Himoya uchun bu yetarli. To'liq (real-time) tizimni ko'rsatish
kerak bo'lsa, pastdagi Docker yo'lidan foydalaning.

## Local host'da ishga tushirish — variant A: Docker (tavsiya etiladi)

Talab qilinadi: Docker Desktop.

```bash
cd smartpark_backend
cp .env.example .env          # kerak bo'lsa qiymatlarni tahrirlang

docker compose up --build     # postgres, redis, mosquitto, backend, celery, nginx

# Boshqa terminalda — migratsiya va superadmin:
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py createsuperuser
```

Ishga tushgach:
- API: **http://localhost:80/api/v1/...** (Nginx orqali) yoki to'g'ridan-to'g'ri **http://localhost:8000**
- Admin panel: **http://localhost/admin/**
- Swagger hujjatlari: **http://localhost/api/docs/**
- WebSocket: **ws://localhost/ws/parking/1/**

### Demo ma'lumot bilan to'ldirish (qo'lda admin panelga yozuv kiritmasdan)

```bash
docker compose exec backend python manage.py seed_demo_data
```

Bu buyruq bitta to'liq ishlaydigan majmua yaratadi: 1 bino, 6 xonadon, 12 ta
turli holatdagi parkovka joyi, va 3 ta tayyor foydalanuvchi:

| Login | Parol | Rol |
|---|---|---|
| `superadmin` | `admin123` | Super Administrator (Django admin) |
| `tsj_admin` | `admin123` | TSJ Administratori |
| `resident` | `resident123` | Rezident (3-xonadon, avtomobil `01A777BB`, faol mehmon ruxsatnomasi bilan) |

### Hammasi ishlayaptimi? — avtomatik tekshiruv

Yuqoridagi 2 ta buyruqdan (migrate + seed_demo_data) keyin:

```bash
chmod +x scripts/smoke_test.sh
./scripts/smoke_test.sh
```

Bu skript: server javob berayaptimi → `resident` bilan login → joylar
ro'yxati → bo'sh joy olish (Allocation Service) → mehmon ruxsatnomasi
ro'yxati — degan zanjirni real HTTP so'rovlar bilan sinaydi va har bir
qadamni PASS/FAIL deb chiqaradi. Birinchi FAIL qaysi qatlamda (auth/DB/API)
muammo borligini ko'rsatadi — himoyadan oldin shuni yashil (hammasi PASS)
holatga keltiring.

### Vizual ko'rish — `frontend/index.html`

Backend o'zi faqat API (admin panel va Swagger'dan boshqa vizual interfeysi
yo'q). `frontend/index.html` — sizning haqiqiy local backendingizga
ulanadigan, login qilib, jonli joylar xaritasini va mehmon ruxsatnomalarini
ko'rsatadigan mustaqil sahifa (qorong'i + oltin dizayn).

```bash
# alohida terminalda:
cd frontend
python3 -m http.server 5500
# brauzerda oching: http://localhost:5500
```

`file://` orqali to'g'ridan-to'g'ri ochsangiz, brauzer CORS xatosi berishi
mumkin — shuning uchun yuqoridagi kabi kichik lokal serverdan oching.
`.env`dagi `CORS_ALLOWED_ORIGINS` allaqachon `http://localhost:5500`ni
o'z ichiga oladi. Sahifa ochilgach: login (`resident`/`resident123`) →
real joylar xaritasi va "Joy olish" tugmasi ko'rinadi.



## Local host'da ishga tushirish — variant B: qo'lda (venv)

Talab qilinadi: Python 3.12, PostgreSQL 16, Redis, Mosquitto
(yoki yuqoridagi "Docker'siz, SQLite bilan" bo'limidagi yengilroq yo'l).

```bash
cd smartpark_backend
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt

cp .env.example .env
# .env faylida POSTGRES_HOST=localhost va REDIS_HOST=localhost qoldiring

# Postgresda bazani yarating:
psql -U postgres -c "CREATE DATABASE smartpark;"

python manage.py migrate
python manage.py createsuperuser

# 3 ta alohida terminalda ishga tushiring:
daphne -b 0.0.0.0 -p 8000 smartpark_backend.asgi:application   # HTTP + WebSocket
celery -A smartpark_backend worker -l info                     # fon vazifalari
celery -A smartpark_backend beat -l info                       # rejalashtirilgan vazifalar

# Mosquitto broker (agar tizimda o'rnatilmagan bo'lsa):
mosquitto -c mosquitto/config/mosquitto.conf
```

Keyin (venv muhitida): `python manage.py seed_demo_data` — yuqoridagi Docker
bo'limida tasvirlangan demo majmua va foydalanuvchilarni yaratadi, so'ng
`bash scripts/smoke_test.sh` (yoki `BASE_URL=http://localhost:8000 bash scripts/smoke_test.sh`,
chunki bu holatda Nginx yo'q) bilan tekshiring.

## MQTT subscriber (sensor -> Django ko'prigi)

`apps/iot_integration/management/commands/mqtt_subscriber.py` — Mosquitto'ga
uzluksiz ulanib turadigan alohida jarayon. `smartpark/<complex_id>/slot/<code>/occupancy`
va `.../barrier/status` topic'larini tinglaydi, `ParkingSlot.status`ni
yangilaydi va `ws/parking/<complex_id>/` orqali jonli xaritaga signal beradi.
`docker-compose.yml`da alohida `mqtt_subscriber` xizmati sifatida qo'shilgan;
qo'lda ishga tushirish uchun:

```bash
python manage.py mqtt_subscriber
```

## Mobil ilova (`mobile_app/`) — joriy holat

- `theme/` — qorong'i + oltin dizayn tizimi, `SeniorMode` (katta shrift/tugma rejimi)
- `services/` — `ApiClient` (JWT saqlash), `ParkingService` (REST + jonli WebSocket oqimi), `GuestPassService`
- `screens/login_screen.dart` — `/api/v1/auth/login/`ga ulangan
- `screens/home_shell.dart` — pastki navigatsiya: Xarita / Mehmon / Sozlama
- `screens/parking_map_screen.dart` — joylarni REST orqali yuklaydi, keyin WebSocket orqali jonli yangilaydi, "Joy olish" tugmasi allocate() service'ni chaqiradi
- `screens/guest_pass_screen.dart` — mehmon ruxsatnomasi so'raydi va QR-kod ko'rsatadi (`qr_flutter`)
- `screens/settings_screen.dart` — Senior Mode almashtirgichi va chiqish

Hali qo'shilmagan (kelgusi qadam): biometrik autentifikatsiya (FaceID/Fingerprint),
o'zbekcha ovozli buyruqlar (speech_to_text/flutter_tts paketlari
`pubspec.yaml`da bor, lekin ekranga ulanmagan), fotofiksatsiya bilan nizo
xabar qilish ekrani. Bu — Flutter/Dart muhiti ushbu konteynerda mavjud
emasligi sabab (`flutter pub get` / `flutter run` sinovdan o'tkazilmadi) va
vaqt/hajm sabablariga ko'ra qoldirilgan asos, to'liq mahsulot emas.

## ESP32 IoT qurilmalarini ulash

`firmware/relay_controller.ino` va `firmware/parking_sensor.ino` — Arduino
IDE'da oching, fayl boshidagi `WIFI_SSID`, `WIFI_PASSWORD`, `MQTT_HOST`
qiymatlarini kompyuteringiz/serveringiz IP manziliga moslang, ESP32 platasini
tanlab yuklang. Ular Mosquitto brokerga ulanib, `smartpark/...` topic'lari
orqali backend bilan gaplashadi (real production'da Django tarafida MQTT
subscriber — Celery worker ichida — yozish kerak bo'ladi; hozircha
`iot_integration/services.py` HTTP webhook orqali ANPR hodisalarini qabul
qiladi, MQTT esa shlagbaum/sensor tomonga bir yo'nalishli signal beradi).

## Muhim eslatmalar

- `requirements.txt` hozircha faqat ro'yxat sifatida tayyorlangan — bu
  muhitda tarmoq yopiq bo'lgani uchun paketlar sinovdan o'tkazilmadi;
  o'z mashinangizda `pip install -r requirements.txt` ishga tushiring.
- Barcha `.py` fayllar sintaksis bo'yicha tekshirilgan (`py_compile`),
  lekin haqiqiy Postgres/Redis/Mosquitto'ga ulanmasdan to'liq ishga
  tushirish tekshirilmagan — birinchi marta ishga tushirganda kichik
  sozlash talab qilinishi mumkin.
- `mobile_app/` papkasida Flutter uchun boshlang'ich skelet (tema +
  login ekrani) bor — bu to'liq mobil ilova emas, davom ettirish uchun
  asos.
- OpenAPI/Swagger fayli qo'lda yozilmagan — u drf-spectacular tomonidan
  `/api/schema/` manzilida avtomatik generatsiya qilinadi (backend
  ishga tushgach yuklab olishingiz mumkin: `curl localhost/api/schema/ -o openapi.yaml`).
