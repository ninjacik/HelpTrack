# HELPTRACK — PROMPT PENGEMBANGAN SISTEM LENGKAP
### Untuk: Claude Code / Cursor / Windsurf / Vibe Coder

---

## KONTEKS SISTEM

**HelpTrack** adalah platform manajemen inventaris dan bantuan bencana nasional berbasis web. Sistem ini digunakan oleh koordinator lapangan, relawan, dan lembaga seperti BNPB dan PMI untuk memantau, mendistribusikan, dan melaporkan bantuan ke lokasi bencana secara real-time.

Sistem saat ini sudah memiliki **UI/HTML statis** yang berfungsi sebagai desain referensi. Tugasmu adalah **mengembangkan sistem penuh** (frontend fungsional + backend + database) berdasarkan desain tersebut, dengan **mempertahankan identitas visual yang sudah ada** namun menghapus semua emoji dan menggantinya dengan elemen teks atau SVG yang tepat agar tampilan terlihat profesional dan tidak seperti hasil generate AI.

---

## DESAIN & VISUAL YANG HARUS DIPERTAHANKAN

Gunakan design system berikut **persis** dari file HTML referensi:

**Tipografi**
- Heading/display: `DM Serif Display` (Google Fonts)
- Body/UI: `DM Sans` (Google Fonts)

**Palet Warna (CSS Variables)**
```css
--teal: #00796B;    --teal-l: #E0F2F1;   --teal-d: #005B54;
--coral: #D84315;   --coral-l: #FBE9E7;
--amber: #F57C00;   --amber-l: #FFF3E0;
--blue: #1565C0;    --blue-l: #E3F2FD;
--surface: #FFFFFF; --bg: #F5F6F8;       --bg2: #ECEEF2;
--text1: #1C1E26;   --text2: #444957;    --text3: #7B8194;
--border: #E2E5EE;  --border2: #CDD1DC;
--danger: #B71C1C;  --success: #2E7D32;  --warning: #E65100;
```

**Komponen Utama yang Sudah Ada**
- Top navigation bar dengan breadcrumb halaman
- Sidebar dashboard dengan navigasi section
- Stat cards dengan progress bar
- Tabel data dengan badge status (DARURAT / SIAGA / PEMULIHAN)
- Modal detail dengan timeline
- Filter chip bar + search field
- Toast notification system
- Activity feed realtime

**Aturan Visual Wajib**
- Hapus semua emoji dari kode (termasuk yang ada di HTML referensi). Ganti dengan teks label atau SVG inline sederhana.
- Jangan gunakan ikon dari library eksternal (FontAwesome, Lucide, dll). Gunakan SVG inline minimalis jika diperlukan ikon.
- Pertahankan border-radius, spacing, shadow, dan transisi yang sudah ada di CSS referensi.
- Jangan ubah font family, warna, dan layout grid.

---

## ARSITEKTUR YANG HARUS DIBANGUN

### Tech Stack Rekomendasi

**Frontend**
- Framework: **Next.js 14** (App Router) + TypeScript
- Styling: **Tailwind CSS** dengan custom CSS variables sesuai design system di atas
- State management: **Zustand** untuk client state, React Query untuk server state
- Grafik/Chart: **Recharts** (ringan, tidak perlu CDN eksternal)
- Peta: **Leaflet.js** dengan tile OpenStreetMap (gratis, tanpa API key)
- Real-time: **Socket.io** client

**Backend**
- Runtime: **Node.js** dengan **Express.js** atau **Fastify**
- Database: **PostgreSQL** dengan **Prisma ORM**
- Real-time: **Socket.io** server
- Auth: **JWT** + bcrypt, dengan middleware role-based access
- File upload: **Multer** + lokal storage (atau S3 jika tersedia)
- Export: **PDFKit** untuk generate laporan PDF

**Database**
- PostgreSQL (bisa menggunakan Supabase atau Railway untuk hosted)

---

## STRUKTUR DATABASE (Prisma Schema)

Buat schema Prisma dengan model-model berikut:

```prisma
model User {
  id        String   @id @default(cuid())
  name      String
  email     String   @unique
  password  String
  role      Role     @default(RELAWAN)
  unit      String?
  phone     String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  deliveries Delivery[]
  activities Activity[]
  reports    Report[]
}

enum Role {
  ADMIN
  KOORDINATOR
  RELAWAN
  VIEWER
}

model Disaster {
  id          String        @id @default(cuid())
  name        String
  location    String
  latitude    Float
  longitude   Float
  type        DisasterType
  status      DisasterStatus @default(DARURAT)
  affectedCount Int          @default(0)
  description String?
  verifiedBy  String?
  reportedAt  DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  needs       Need[]
  deliveries  Delivery[]
  timeline    Timeline[]
  activities  Activity[]
}

enum DisasterType {
  GEMPA
  BANJIR
  LONGSOR
  ERUPSI
  KEKERINGAN
  LAINNYA
}

enum DisasterStatus {
  DARURAT
  SIAGA
  PEMULIHAN
  SELESAI
}

model Need {
  id          String     @id @default(cuid())
  disasterId  String
  disaster    Disaster   @relation(fields: [disasterId], references: [id])
  category    NeedCategory
  description String
  quantity    Int
  fulfilled   Int        @default(0)
  unit        String
  urgent      Boolean    @default(false)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  deliveryItems DeliveryItem[]
}

enum NeedCategory {
  MAKANAN
  AIR
  MEDIS
  SHELTER
  PAKAIAN
  EVAKUASI
  LAINNYA
}

model Inventory {
  id          String   @id @default(cuid())
  name        String
  category    NeedCategory
  quantity    Int
  unit        String
  warehouseId String
  warehouse   Warehouse @relation(fields: [warehouseId], references: [id])
  minStock    Int      @default(10)
  updatedAt   DateTime @updatedAt

  deliveryItems DeliveryItem[]
}

model Warehouse {
  id        String      @id @default(cuid())
  name      String
  location  String
  latitude  Float
  longitude Float
  isActive  Boolean     @default(true)

  inventory Inventory[]
  deliveries Delivery[] @relation("OriginWarehouse")
}

model Delivery {
  id          String         @id @default(cuid())
  code        String         @unique
  disasterId  String
  disaster    Disaster       @relation(fields: [disasterId], references: [id])
  warehouseId String
  warehouse   Warehouse      @relation("OriginWarehouse", fields: [warehouseId], references: [id])
  driverId    String?
  driver      User?          @relation(fields: [driverId], references: [id])
  status      DeliveryStatus @default(DISIAPKAN)
  vehicleId   String?
  vehicle     Vehicle?       @relation(fields: [vehicleId], references: [id])
  departedAt  DateTime?
  arrivedAt   DateTime?
  eta         DateTime?
  notes       String?
  createdAt   DateTime       @default(now())

  items       DeliveryItem[]
}

enum DeliveryStatus {
  DISIAPKAN
  BERANGKAT
  DALAM_PERJALANAN
  TIBA
  SELESAI
  DIBATALKAN
}

model DeliveryItem {
  id          String    @id @default(cuid())
  deliveryId  String
  delivery    Delivery  @relation(fields: [deliveryId], references: [id])
  inventoryId String
  inventory   Inventory @relation(fields: [inventoryId], references: [id])
  needId      String?
  need        Need?     @relation(fields: [needId], references: [id])
  quantity    Int
}

model Vehicle {
  id       String     @id @default(cuid())
  code     String     @unique
  type     String
  plate    String
  capacity Int
  isActive Boolean    @default(true)

  deliveries Delivery[]
}

model Volunteer {
  id          String          @id @default(cuid())
  userId      String          @unique
  specialization String
  skills      String[]
  location    String?
  status      VolunteerStatus @default(STANDBY)
  joinedAt    DateTime        @default(now())
}

enum VolunteerStatus {
  AKTIF
  STANDBY
  TIDAK_AKTIF
}

model Timeline {
  id         String   @id @default(cuid())
  disasterId String
  disaster   Disaster @relation(fields: [disasterId], references: [id])
  event      String
  detail     String?
  createdAt  DateTime @default(now())
  createdBy  String?
}

model Activity {
  id         String   @id @default(cuid())
  type       String
  message    String
  disasterId String?
  disaster   Disaster? @relation(fields: [disasterId], references: [id])
  userId     String?
  user       User?    @relation(fields: [userId], references: [id])
  createdAt  DateTime @default(now())
}

model Notification {
  id        String   @id @default(cuid())
  userId    String
  type      String
  message   String
  isRead    Boolean  @default(false)
  link      String?
  createdAt DateTime @default(now())
}

model Report {
  id          String   @id @default(cuid())
  type        String
  period      String
  generatedBy String
  user        User     @relation(fields: [generatedBy], references: [id])
  data        Json
  pdfUrl      String?
  createdAt   DateTime @default(now())
}
```

---

## FITUR YANG HARUS DIBANGUN (Lengkap)

### 1. AUTENTIKASI & MANAJEMEN USER

**Halaman Login**
- Form email + password dengan validasi
- JWT access token (15 menit) + refresh token (7 hari) disimpan di httpOnly cookie
- Halaman pendaftaran relawan (role default: RELAWAN, perlu approval KOORDINATOR)
- Halaman lupa password dengan OTP email sederhana (bisa mock tanpa SMTP dulu)

**Role & Permission**
```
ADMIN        → akses penuh: CRUD semua data, manajemen user, pengaturan sistem
KOORDINATOR  → bisa tambah/edit bencana, kebutuhan, pengiriman, approve relawan
RELAWAN      → bisa update status pengiriman yang ditugaskan, lihat data
VIEWER       → hanya baca (publik terautentikasi)
```

Middleware guard di setiap route API berdasarkan role di atas.

---

### 2. DASHBOARD — OVERVIEW (Halaman Utama Setelah Login)

Ganti data statis dengan data real dari API:

**Stat Cards (4 kartu)**
- Jiwa terdampak total (sum dari semua disaster aktif)
- Total logistik tersalurkan hari ini (sum delivery items yang selesai hari ini)
- Pengiriman aktif saat ini (count delivery dengan status BERANGKAT / DALAM_PERJALANAN)
- Kebutuhan belum terpenuhi (count need dengan fulfilled < quantity)

**Peta Mini (di Overview)**
- Gunakan Leaflet.js dengan tile OpenStreetMap
- Tampilkan marker per disaster dengan warna sesuai status
- Tampilkan garis rute dari gudang pusat ke tiap disaster aktif
- Marker bisa diklik → buka modal detail disaster

**Aktivitas Terbaru**
- Feed dari tabel `Activity`, real-time via Socket.io
- Setiap perubahan status pengiriman, penambahan kebutuhan, atau pendaftaran relawan → emit event → update feed tanpa reload

**Tabel Status Bencana Aktif**
- Data dari API `/api/disasters?status=active`
- Kolom: Lokasi, Jenis, Jiwa, Status, Progress Logistik (%), Relawan Aktif, Terakhir Diperbarui, Aksi
- Tombol "Respons" → buka modal detail / navigasi ke halaman detail bencana
- Tombol "+ Tambah Laporan" → modal form tambah bencana baru

---

### 3. PETA BENCANA LIVE (Tab Peta)

Halaman peta penuh menggunakan Leaflet.js:

- Tile layer: OpenStreetMap (gratis)
- Marker cluster untuk area dengan banyak pin
- Filter panel di sidebar peta: by status (Darurat/Siaga/Pemulihan), by jenis bencana
- Setiap marker: popup dengan info ringkas (nama, jiwa, status, progress) + tombol "Lihat Detail"
- Layer kendaraan: posisi truck/armada yang sedang dalam pengiriman (update tiap 30 detik dari API)
- Layer gudang: marker gudang dengan info stok ringkas
- Legend di pojok bawah kiri (tanpa emoji, gunakan titik berwarna + label teks)

---

### 4. KEBUTUHAN AKTIF (Tab Needs)

**Tabel + Filter (di Dashboard)**
- Filter by: Semua, Darurat, Siaga, Pemulihan, Pangan, Medis, Shelter
- Search by: nama lokasi, jenis kebutuhan
- Kolom: Lokasi, Deskripsi Kebutuhan, Status, Progress, Donatur/Relawan, Terakhir Update, Aksi
- Tombol "Detail" → modal detail
- Tombol "Kirim" → buka modal form pengiriman baru yang terhubung ke kebutuhan ini

**Modal Detail Bencana/Kebutuhan**
- Info lengkap: deskripsi, jumlah jiwa, verifikasi, koordinat
- Progress bar pemenuhan per kategori kebutuhan
- Timeline respons (dari tabel `Timeline`)
- Daftar kebutuhan per kategori dengan quantity yang dibutuhkan vs terpenuhi
- Tombol: "Buat Pengiriman", "Salin Koordinat", "Unduh Laporan"

**Form Tambah/Edit Kebutuhan**
- Input: kategori, deskripsi, quantity, unit, tingkat urgensi
- Validasi wajib semua field
- Setelah submit → update tabel + emit Socket.io event ke semua client

---

### 5. MANAJEMEN PENGIRIMAN (Tab Logistics)

**Tabel Pengiriman**
- Filter: Semua, Disiapkan, Berangkat, Dalam Perjalanan, Tiba, Selesai
- Kolom: Kode (HT-XX), Muatan (ringkasan item), Tujuan, Status, Driver, Kendaraan, ETA, Aksi
- Badge status berwarna sesuai design system
- Tombol "Update Status" → dropdown inline untuk ubah status pengiriman
- Tombol "Detail" → slide panel atau modal dengan detail lengkap

**Form Buat Pengiriman Baru**
- Pilih disaster tujuan (dropdown dari data aktif)
- Pilih gudang asal
- Tambah item: pilih inventaris → set quantity (validasi tidak melebihi stok)
- Pilih driver (dari daftar relawan dengan skill transportasi)
- Pilih kendaraan (dari daftar kendaraan aktif)
- Set ETA
- Setelah submit: stok inventaris otomatis berkurang, activity log dibuat

**Update Status Pengiriman**
- DISIAPKAN → BERANGKAT: catat `departedAt`
- BERANGKAT → DALAM_PERJALANAN: update posisi (input koordinat atau GPS)
- DALAM_PERJALANAN → TIBA: catat `arrivedAt`, trigger update `Need.fulfilled`
- TIBA → SELESAI: konfirmasi penerimaan

---

### 6. INVENTARIS GUDANG (Tab Inventory)

**Tabel Stok per Gudang**
- Tab atau dropdown pilih gudang (Gudang A, Gudang B, Gudang C, dll.)
- Kolom: Nama Barang, Kategori, Stok Tersedia, Unit, Stok Minimum, Status (Normal / Menipis / Kritis)
- Row dengan stok di bawah minimum → highlight warna kuning/merah
- Search & filter by kategori

**Form Tambah/Edit Stok**
- Input: nama barang, kategori, quantity, unit, stok minimum, gudang
- Validasi tidak boleh negatif

**Penerimaan Donasi/Masuk Barang**
- Form: pilih barang (atau tambah baru), quantity masuk, sumber donasi, tanggal
- Setelah submit → stok bertambah + activity log

**Alert Stok Kritis**
- Background job: cek setiap 30 menit, jika ada item dengan stok < minimum → buat Notification untuk KOORDINATOR dan ADMIN
- Tampilkan daftar "Item Kritis" di bagian atas halaman inventaris

---

### 7. MANAJEMEN RELAWAN (Tab Volunteers)

**Stat Ringkasan**
- Total relawan aktif, di lapangan, tenaga medis, standby

**Tabel Relawan**
- Kolom: Nama, Spesialisasi, Lokasi Penugasan, Status, Bergabung, Aksi
- Filter by status, by spesialisasi
- Tombol "Profil" → modal detail relawan
- Tombol "Tugaskan" → assign ke disaster tertentu

**Form Pendaftaran/Edit Relawan**
- Input: nama, email, spesialisasi, skills (multi-select: Medis, Logistik, Transportasi, Konstruksi, Komunikasi), nomor HP, lokasi asal

**Approval Relawan Baru**
- KOORDINATOR/ADMIN bisa approve atau tolak pendaftaran baru
- Status pending ditampilkan dengan badge khusus

---

### 8. LAPORAN TRANSPARANSI (Tab Transparency)

**Laporan Harian (Auto-generate)**
- Endpoint: `GET /api/reports/daily?date=YYYY-MM-DD`
- Data: total jiwa terdampak, item tersalurkan, nilai estimasi bantuan, jumlah relawan aktif
- Tampilan: card dengan metrik + ringkasan aktivitas hari itu

**Laporan Inventaris Mingguan**
- Data: total barang masuk, total barang keluar, item kritis, jumlah gudang aktif
- Grafik bar: perbandingan masuk vs keluar per hari (gunakan Recharts)

**Laporan Distribusi**
- Data: jumlah pengiriman selesai, aktif, total km, success rate
- Grafik line: tren pengiriman per hari (Recharts)

**Download PDF**
- Tombol "Unduh PDF" → call `POST /api/reports/generate` → backend generate PDF menggunakan PDFKit → return file stream
- PDF berisi header HelpTrack, metrik, tabel ringkasan, footer dengan timestamp dan nama generator

---

### 9. NOTIFIKASI REAL-TIME

**Panel Notifikasi (di navbar kanan)**
- Data dari `GET /api/notifications?userId=...`
- Badge count untuk yang belum dibaca
- Klik notif → mark as read + navigasi ke halaman terkait
- "Tandai semua dibaca" button
- Real-time: Socket.io push notification saat ada event baru

**Trigger Notifikasi Otomatis**
- Stok item < minimum threshold
- Bencana baru dilaporkan
- Status pengiriman berubah (khusus KOORDINATOR yang menugaskan)
- Relawan baru mendaftar (khusus KOORDINATOR/ADMIN)
- Pengiriman terlambat (ETA terlewat > 1 jam)

---

### 10. HALAMAN PUBLIK (Tanpa Login)

**Landing Page** (sudah ada di HTML referensi, buat versi fungsional)
- Hero section dengan counter live (jiwa terdampak, logistik tersalurkan, relawan aktif) → data dari API publik
- Alert banner bencana aktif teratas (bencana dengan status DARURAT paling baru)
- Peta mini dengan pin lokasi bencana aktif → Leaflet.js
- Grid 3 kebutuhan paling mendesak (darurat dengan progress terendah)
- Section "Cara Kerja" (statis)
- Footer dengan kontak darurat

**Halaman Kebutuhan Aktif (Publik)**
- Grid semua bencana aktif dengan filter dan search
- Publik hanya bisa lihat detail, tidak bisa buat pengiriman
- Tombol "Kirim Bantuan" → redirect ke halaman login

---

## ENDPOINT API YANG HARUS DIBUAT

```
AUTH
POST   /api/auth/login
POST   /api/auth/register
POST   /api/auth/logout
POST   /api/auth/refresh
GET    /api/auth/me

DISASTERS
GET    /api/disasters                    (public, filter: status, type)
GET    /api/disasters/:id
POST   /api/disasters                    (KOORDINATOR+)
PUT    /api/disasters/:id                (KOORDINATOR+)
DELETE /api/disasters/:id                (ADMIN)
GET    /api/disasters/:id/timeline
POST   /api/disasters/:id/timeline       (KOORDINATOR+)

NEEDS
GET    /api/needs                        (filter: disasterId, category, status)
GET    /api/needs/:id
POST   /api/needs                        (KOORDINATOR+)
PUT    /api/needs/:id                    (KOORDINATOR+)
DELETE /api/needs/:id                    (KOORDINATOR+)

INVENTORY
GET    /api/inventory                    (filter: warehouseId, category)
GET    /api/inventory/:id
POST   /api/inventory                    (KOORDINATOR+)
PUT    /api/inventory/:id                (KOORDINATOR+)
POST   /api/inventory/:id/restock        (quantity masuk)

WAREHOUSES
GET    /api/warehouses
POST   /api/warehouses                   (ADMIN)
PUT    /api/warehouses/:id               (ADMIN)

DELIVERIES
GET    /api/deliveries                   (filter: status, disasterId)
GET    /api/deliveries/:id
POST   /api/deliveries                   (KOORDINATOR+)
PUT    /api/deliveries/:id/status        (update status + timestamp)
DELETE /api/deliveries/:id               (ADMIN)

VOLUNTEERS
GET    /api/volunteers                   (filter: status, specialization)
GET    /api/volunteers/:id
POST   /api/volunteers                   (RELAWAN register)
PUT    /api/volunteers/:id               (KOORDINATOR+)
PUT    /api/volunteers/:id/approve       (KOORDINATOR+)

VEHICLES
GET    /api/vehicles
POST   /api/vehicles                     (ADMIN)
PUT    /api/vehicles/:id                 (ADMIN)

ACTIVITIES
GET    /api/activities                   (limit: 50, filter: disasterId)

NOTIFICATIONS
GET    /api/notifications                (userId dari token)
PUT    /api/notifications/:id/read
PUT    /api/notifications/read-all

REPORTS
GET    /api/reports/daily?date=
GET    /api/reports/weekly-inventory?week=
GET    /api/reports/weekly-delivery?week=
POST   /api/reports/generate             (generate + return PDF)

STATS (PUBLIC)
GET    /api/stats/summary                (jiwa, logistik, relawan — untuk landing page)
```

---

## SOCKET.IO EVENTS

**Server Emit (backend → client)**
```
disaster:new          { disaster }
disaster:updated      { disasterId, changes }
delivery:status       { deliveryId, status, updatedAt }
inventory:low         { inventoryId, name, quantity }
activity:new          { activity }
notification:new      { notification, userId }
volunteer:new         { volunteer }
```

**Client Emit (client → server)**
```
subscribe:dashboard   (join room 'dashboard')
subscribe:disaster    { disasterId }
delivery:location     { deliveryId, lat, lng }
```

---

## FITUR YANG BELUM ADA DI HTML REFERENSI (HARUS DITAMBAHKAN)

Berikut fitur yang **sama sekali tidak ada** di file HTML statis dan harus dibangun dari nol:

1. **Sistem login/autentikasi** — tidak ada sama sekali, semua halaman terbuka
2. **Form input fungsional** — semua tombol hanya menampilkan toast, tidak ada CRUD nyata
3. **Peta interaktif real** — peta di HTML hanya simulasi CSS gradient, bukan peta sungguhan
4. **Grafik/chart analitik** — tidak ada visualisasi data sama sekali di laporan
5. **Manajemen stok inventaris** — hanya ada tabel statis tanpa CRUD
6. **Form pengiriman terhubung ke stok** — tombol "Kirim" tidak mengurangi stok
7. **Approval workflow relawan** — tidak ada mekanisme approve/reject
8. **Sistem alert otomatis** — tidak ada background job untuk cek threshold stok
9. **Generate PDF laporan** — tombol unduh hanya toast
10. **Pencarian global** — tidak ada search yang benar-benar memfilter data API
11. **Manajemen kendaraan** — tidak ada halaman/CRUD untuk armada
12. **Manajemen gudang** — tidak ada CRUD untuk lokasi gudang
13. **Penugasan relawan ke disaster** — tidak ada mekanisme assignment
14. **History pengiriman per disaster** — tidak ada audit trail pengiriman
15. **Export data** — tidak ada export ke CSV/Excel

---

## STRUKTUR FOLDER PROYEK

```
helptrack/
├── apps/
│   ├── web/                    (Next.js frontend)
│   │   ├── app/
│   │   │   ├── (public)/       (landing, campaigns — tanpa auth)
│   │   │   │   ├── page.tsx    (landing)
│   │   │   │   └── needs/
│   │   │   │       └── page.tsx
│   │   │   ├── (auth)/
│   │   │   │   ├── login/
│   │   │   │   └── register/
│   │   │   └── dashboard/      (protected routes)
│   │   │       ├── layout.tsx  (sidebar + topbar)
│   │   │       ├── page.tsx    (overview)
│   │   │       ├── map/
│   │   │       ├── needs/
│   │   │       ├── logistics/
│   │   │       ├── inventory/
│   │   │       ├── volunteers/
│   │   │       └── reports/
│   │   ├── components/
│   │   │   ├── ui/             (Button, Badge, Modal, Toast, etc.)
│   │   │   ├── dashboard/      (StatCard, ActivityFeed, MiniMap, etc.)
│   │   │   ├── forms/          (DisasterForm, DeliveryForm, NeedForm, etc.)
│   │   │   └── charts/         (BarChart, LineChart wrapper)
│   │   ├── hooks/              (useSocket, useNotifications, useAuth, etc.)
│   │   ├── lib/                (api client, socket client, utils)
│   │   └── store/              (Zustand stores)
│   │
│   └── api/                    (Express/Fastify backend)
│       ├── src/
│       │   ├── routes/         (satu file per resource)
│       │   ├── controllers/
│       │   ├── middleware/     (auth, roles, validate)
│       │   ├── services/       (business logic)
│       │   ├── jobs/           (cron: cek stok, generate laporan)
│       │   ├── socket/         (socket event handlers)
│       │   └── lib/            (prisma client, pdf generator)
│       └── prisma/
│           ├── schema.prisma
│           └── seed.ts         (data awal untuk development)
│
├── packages/
│   └── shared/                 (types, constants yang dipakai keduanya)
│
├── docker-compose.yml          (PostgreSQL, Redis jika pakai queue)
└── README.md
```

---

## SEED DATA UNTUK DEVELOPMENT

Buat file `prisma/seed.ts` dengan data awal:

- 3 warehouse: Gudang Jakarta, Gudang Surabaya, Gudang Denpasar
- 5 disaster aktif: Gempa Karangasem, Longsor Cianjur, Erupsi Merapi, Gempa Lombok, Banjir Demak
- 20+ kebutuhan tersebar di disaster di atas
- 15+ item inventaris per gudang (makanan, air, obat, tenda, selimut, dll.)
- 5 kendaraan: HT-01 s/d HT-05
- 10 relawan dengan spesialisasi berbeda
- 3 pengiriman aktif dengan status berbeda
- 20 aktivitas terbaru
- User default:
  - admin@helptrack.id / admin123 (ADMIN)
  - koordinator@helptrack.id / koordinator123 (KOORDINATOR)
  - relawan@helptrack.id / relawan123 (RELAWAN)

---

## ATURAN CODING

1. **TypeScript strict mode** — tidak boleh ada `any` yang tidak perlu
2. **Error handling** — semua endpoint API harus punya try-catch dan return error yang konsisten:
   ```json
   { "success": false, "message": "Pesan error", "code": "ERROR_CODE" }
   ```
3. **Validasi input** — gunakan Zod untuk validasi request body di semua endpoint
4. **Pagination** — semua endpoint yang return list harus support `?page=&limit=`
5. **Response format konsisten**:
   ```json
   { "success": true, "data": {...}, "meta": { "page": 1, "total": 50 } }
   ```
6. **Environment variables** — semua config sensitif di `.env`, jangan hardcode
7. **Seed data** — pastikan `npm run seed` bekerja untuk development

---

## URUTAN PENGERJAAN YANG DISARANKAN

Kerjakan dalam urutan ini supaya sistem bisa dicoba bertahap:

1. Setup project monorepo + Docker Compose (PostgreSQL)
2. Buat Prisma schema + migration + seed
3. Backend: auth routes (login, register, me)
4. Backend: CRUD disasters + needs + inventory
5. Backend: CRUD deliveries + vehicles + warehouses
6. Backend: volunteers + notifications
7. Backend: Socket.io setup + emit events
8. Backend: report generation + PDF export
9. Frontend: setup Next.js + design system CSS variables + komponen UI dasar
10. Frontend: halaman login/register
11. Frontend: dashboard layout (sidebar + topbar) dengan routing
12. Frontend: Overview tab (stat cards + mini map + activity feed)
13. Frontend: integrasi Socket.io client
14. Frontend: tab Peta Penuh (Leaflet)
15. Frontend: tab Kebutuhan + modal detail + form
16. Frontend: tab Logistik + form pengiriman
17. Frontend: tab Inventaris + alert stok
18. Frontend: tab Relawan + approval
19. Frontend: tab Laporan + chart + download PDF
20. Frontend: landing page publik + halaman needs publik
21. Testing end-to-end + polish UI

---

## CATATAN PENTING TAMBAHAN

- Semua label status gunakan **Bahasa Indonesia** sesuai referensi (DARURAT, SIAGA, PEMULIHAN, BERANGKAT, dll.)
- Semua angka format Indonesia: `toLocaleString('id-ID')` (12.847, bukan 12,847)
- Tanggal format Indonesia: "5 Jun 2025, 03:24"
- Tidak perlu payment gateway — sistem ini adalah platform koordinasi, bukan donasi uang
- Mobile responsiveness: sidebar collapse di layar < 768px, navigasi pindah ke bottom bar
- Dark mode: tidak diperlukan untuk versi pertama
- Accessibility: pastikan semua input punya label, semua button punya text yang jelas
- Loading state: setiap data fetch harus ada skeleton loader, bukan spinner kosong
- Empty state: setiap tabel/grid harus punya tampilan "Tidak ada data" yang informatif

---

*Prompt ini dibuat berdasarkan analisis file `helptrack_redesign__2_.html` — desain referensi UI yang sudah ada.*
*Versi prompt: 1.0 — HelpTrack Full-Stack Development*
