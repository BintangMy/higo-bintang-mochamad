## 01 · Monolith vs Microservices — Kapan Digunakan?

> **Analogi:** Bayangkan sebuah restoran. **Monolith** = satu dapur besar yang masak semua menu. **Microservices** = food-court, tiap stan punya dapur sendiri. Food-court lebih fleksibel tapi butuh koordinasi lebih besar.

### Monolith
- Satu codebase, satu aplikasi, deploy sekali jalan semua
- Mudah dibangun di awal, debugging lebih mudah
- Makin besar makin lambat dan sulit diubah
- Scale = harus scale semua bagian sekaligus

### Microservices
- Banyak layanan kecil, masing-masing mandiri & bisa deploy sendiri
- Tiap service bisa pakai bahasa/stack berbeda
- Scale hanya bagian yang butuh
- Lebih kompleks: network, config, monitoring, observability

### Kapan Pakai Yang Mana?

| Kondisi | Pilihan |
|---|---|
| Startup baru, tim kecil (<10 orang) | **Monolith** |
| Produk masih validasi pasar | **Monolith** |
| Timeline ketat, tidak ada budget DevOps | **Monolith** |
| Skala pengguna besar, tim banyak & independen | **Microservices** |
| Bagian sistem butuh scale berbeda | **Microservices** |
| Butuh resiliency & deployment independen | **Microservices** |

> **Best Practice Modern:** Mulai dari **modular monolith**, lalu pecah menjadi microservices saat ada pain point nyata. Jangan premature optimization!

**TL;DR** — Monolith = sederhana, cepat mulai. Microservices = fleksibel tapi mahal kompleksitasnya. Mulai monolith, pisah saat butuh.

---

## 02 · Cara Optimasi Query MongoDB yang Lambat + Contoh Index

> **Analogi:** Query tanpa index = mencari kata di buku tebal tanpa daftar isi. Index = daftar isi — langsung lompat ke halaman yang benar.

### Langkah Diagnosis

1. **Gunakan `.explain("executionStats")`** — lihat `totalDocsExamined`. Jika jauh lebih besar dari `totalDocsReturned`, artinya MongoDB scan terlalu banyak dokumen (COLLSCAN). Ini tanda perlu index.
2. **Aktifkan MongoDB Profiler** — query lebih dari 100ms akan tercatat di `system.profile`. Cara menemukan query lambat di production.
3. **Perhatikan field yang sering dipakai** di `find()`, `sort()`, dan `match()` dalam aggregation — itulah kandidat index.

### Contoh Penggunaan Index

```js
// Single field index — cari user by email
db.users.createIndex({ email: 1 })

// Compound index — cari order by userId + status
// Urutan field PENTING: field paling "selective" dulu
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 })

// Text index — full-text search pada nama film
db.movies.createIndex({ title: "text", description: "text" })

// Partial index — index hanya order yang "active"
// Lebih hemat storage daripada index semua dokumen
db.orders.createIndex(
  { createdAt: 1 },
  { partialFilterExpression: { status: "active" } }
)

// 🔍 Cek apakah query pakai index (lihat stage: IXSCAN vs COLLSCAN)
db.orders.find({ userId: "abc", status: "active" })
         .explain("executionStats")
```

### Tips Tambahan
- **Projection:** `find({}, {field1: 1})` — ambil hanya field yang dibutuhkan, kurangi data transfer.
- **Pagination:** Pakai cursor-based pagination (lastId) bukan `.skip()` yang makin lambat di halaman besar.
- **Hindari `$where` & `$regex`** tanpa anchor `^` — menyebabkan full scan meski ada index.

**TL;DR** — Diagnosis dengan `.explain()`, buat compound index sesuai urutan query, gunakan partial index jika hanya sebagian data yang aktif.

---

## 03 · Rancang API Pemesanan Tiket Bioskop — Endpoint & Payload

> **Analogi:** API ini seperti kasir bioskop digital. Ada loket jadwal film, loket pilih kursi, loket pembayaran, dan loket cetak tiket — semua punya "endpoint" tersendiri.

### Daftar Endpoint

| Method | Endpoint | Fungsi |
|---|---|---|
| `GET` | `/movies` | Daftar semua film yang sedang tayang |
| `GET` | `/movies/:id/showtimes` | Jadwal tayang film tertentu |
| `GET` | `/showtimes/:id/seats` | Denah & status kursi (available/taken) |
| `POST` | `/bookings/hold` | Hold kursi (reserved 10 menit) |
| `POST` | `/bookings/confirm` | Konfirmasi & bayar pemesanan |
| `GET` | `/bookings/:id` | Detail pemesanan & e-ticket |
| `DELETE` | `/bookings/:id` | Batalkan pemesanan |
| `POST` | `/auth/login` | Login pengguna |
| `POST` | `/auth/refresh` | Refresh access token |

### Contoh Payload

**POST /bookings/hold — Request Body**
```json
{
  "showtimeId": "show_abc123",
  "seats": ["D4", "D5"],
  "userId": "usr_789"
}
```

**POST /bookings/hold — Response**
```json
{
  "holdId": "hold_xyz",
  "expiresAt": "2024-06-01T14:10:00Z",
  "totalPrice": 75000,
  "seats": ["D4", "D5"],
  "status": "held"
}
```

**POST /bookings/confirm — Request Body**
```json
{
  "holdId": "hold_xyz",
  "paymentMethod": "gopay",
  "paymentToken": "tok_abc..."
}
```

> **Pola "Hold → Confirm"** sangat penting untuk mencegah *double booking*. Implementasikan dengan **Redis TTL** untuk auto-release kursi jika user tidak jadi bayar dalam 10 menit.

**TL;DR** — Gunakan pola hold-then-confirm, beri TTL pada hold agar kursi auto-release, semua endpoint sensitif wajib pakai JWT auth.

---

## 04 · Cara Mengamankan API Publik — Rate Limiting & Auth

> **Analogi:** API publik = toko yang terbuka untuk umum. **Auth** = satpam yang cek kartu ID di pintu masuk. **Rate Limiting** = aturan "maksimal 100 pelanggan per jam" agar toko tidak overload.

### Layer Keamanan API (wajib semua ada)

1. **Authentication (JWT / API Key)** — Setiap request wajib bawa token. Verifikasi signature di server sebelum proses request.
2. **Rate Limiting** — Batasi jumlah request per IP atau per user per satuan waktu. Response HTTP 429 jika melewati batas.
3. **HTTPS Wajib** — Semua data dienkripsi dalam transit. Gunakan HSTS header agar browser selalu pakai HTTPS.
4. **Input Validation** — Validasi semua input menggunakan library seperti `zod` atau `joi`. Cegah SQL/NoSQL injection dan XSS.
5. **CORS yang ketat** — Whitelist domain yang boleh akses. Jangan pakai `*` di production.

### Contoh Implementasi

```js
const rateLimit = require('express-rate-limit')
const jwt       = require('jsonwebtoken')

// Rate limiter: maks 100 request per 15 menit per IP
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,  // kirim header RateLimit-*
  legacyHeaders: false,
  keyGenerator: (req) => req.user?.id || req.ip, // by user jika login
})

// JWT auth middleware
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1]
  if (!token) return res.status(401).json({ error: 'Unauthorized' })
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET)
    next()
  } catch {
    res.status(403).json({ error: 'Invalid token' })
  }
}

app.use('/api', limiter, authenticate)
```

> Untuk API publik skala besar, pindahkan rate limiting ke layer infrastructure (Nginx, Cloudflare, API Gateway) — lebih efisien daripada di level aplikasi.

**TL;DR** — Auth = verifikasi identitas, Rate Limit = batasi frekuensi, HTTPS = enkripsi data, Validasi = cegah input jahat. Keempat layer harus ada semua.

---

## 05 · Risiko JWT Auth & Cara Mengatasinya

> **Analogi:** JWT seperti tiket masuk konser. Setelah diterima, panitia tidak bisa langsung membatalkan tiket yang sudah beredar — berbeda dengan sistem kartu ID yang bisa di-deaktivasi kapan saja.

### Risiko & Solusi

| Risiko | Solusi |
|---|---|
| ❌ Token tidak bisa di-revoke | ✅ Access token singkat (15 menit) + refresh token. Simpan daftar refresh token yang di-revoke di Redis. |
| ❌ Token bocor = akses penuh | ✅ Simpan refresh token di `httpOnly cookie`. Access token di memory (bukan localStorage). Wajib HTTPS. |
| ❌ Payload bisa dibaca siapa saja | ✅ Jangan simpan data sensitif di payload. Hanya simpan `userId`, `role`, `exp`. |
| ❌ Algorithm confusion attack | ✅ Selalu spesifikasikan algoritma: `jwt.verify(token, secret, { algorithms: ['HS256'] })`. Tolak `alg: none`. |

**TL;DR** — Access token singkat (15m) + refresh token di httpOnly cookie = pola paling aman. Jangan simpan data sensitif di payload. Validasi algoritma eksplisit.

---

## 06 · Strategi Zero Downtime Deployment Node.js

> **Analogi:** Zero downtime deployment seperti ganti ban mobil yang sedang melaju. Tidak boleh berhenti — bergantian, satu ban diganti sementara tiga ban lain tetap jalan.

### Strategi

1. **Rolling Update (paling umum)** — Update instance satu per satu. Load balancer tetap mengarahkan traffic ke instance yang masih running. Saat instance baru siap (health check pass), instance lama dimatikan.

2. **Blue-Green Deployment** — Dua environment identik (Blue = live, Green = baru). Deploy ke Green, test, lalu switch traffic sekaligus. Rollback instan cukup switch balik ke Blue.

3. **PM2 Cluster + Graceful Reload** — Untuk server single VPS. PM2 mematikan dan menghidupkan worker satu per satu agar selalu ada worker aktif.

4. **Graceful Shutdown (wajib untuk semua strategi)** — Saat process mau dimatikan, selesaikan semua request yang sedang berjalan, tutup koneksi database, baru exit.

### Contoh Graceful Shutdown

```js
const server = app.listen(3000)

process.on('SIGTERM', async () => {
  console.log('Menerima SIGTERM, mulai graceful shutdown...')

  // Stop menerima request baru
  server.close(async () => {
    // Selesaikan semua koneksi yang masih berjalan
    await db.disconnect()
    await redis.quit()
    console.log('Shutdown selesai dengan bersih')
    process.exit(0)
  })

  // Timeout 30 detik: force exit jika ada request yang hang
  setTimeout(() => process.exit(1), 30000)
})
```

> Untuk **Kubernetes**: gunakan `readinessProbe` — pod baru tidak akan menerima traffic sampai health check berhasil. Built-in zero downtime.

**TL;DR** — Blue-Green untuk rollback instan, Rolling Update untuk efisiensi resource, PM2 untuk VPS sederhana. Semua butuh graceful shutdown.

---

## 07 · Perbedaan CSR, SSR, SSG di React / Next.js

> **Analogi restoran:** **CSR** = self-service, bahan mentah dikirim ke meja, kamu masak sendiri. **SSR** = masak setiap order di dapur saat pesanan masuk. **SSG** = masak banyak-banyak di pagi hari, langsung saji saat dipesan.

### Perbandingan

| | CSR | SSR | SSG |
|---|---|---|---|
| **Render di** | Browser (client) | Server (per request) | Build time |
| **Loading awal** | Lambat | Cepat | Paling cepat |
| **SEO** | Jelek | Baik | Baik |
| **Data** | Selalu fresh | Selalu fresh | Statis (bisa ISR) |
| **Server load** | Rendah | Tinggi | Sangat rendah |
| **Cocok untuk** | Dashboard, app auth | Feed sosial, produk | Blog, docs, landing page |

### Contoh di Next.js

```jsx
// SSG: getStaticProps — build time, cocok blog/docs
export async function getStaticProps() {
  const posts = await fetchPosts()
  return { props: { posts }, revalidate: 60 } // ISR: rebuild tiap 60 detik
}

// SSR: getServerSideProps — per request, cocok data realtime
export async function getServerSideProps(context) {
  const user = await getUser(context.req.cookies.token)
  return { props: { user } }
}

// CSR: fetch di useEffect — data di-load setelah halaman muncul
function Dashboard() {
  const [data, setData] = useState(null)
  useEffect(() => { fetchDashboardData().then(setData) }, [])
  return data ? <Chart data={data} /> : <Spinner />
}
```

> **ISR (Incremental Static Regeneration)** = hybrid SSG: halaman di-rebuild otomatis setiap N detik di background. Dapat performa SSG dengan data yang relatif fresh.

**TL;DR** — SSG untuk konten statis (cepat, murah), SSR untuk konten personal/realtime, CSR untuk app yang butuh auth. Next.js bisa gabungkan ketiganya dalam satu project.

---

## 08 · Desain Fitur Chat Realtime — Stack & Alasan

> **Analogi:** Chat realtime seperti telepon — koneksi harus tetap terbuka terus-menerus agar bisa bicara dua arah. Berbeda dengan HTTP biasa yang seperti surat — kirim, tutup, tunggu balasan.

### Stack Yang Dipilih

| Layer | Teknologi | Alasan |
|---|---|---|
| Transport | **WebSocket** via Socket.io | Koneksi dua arah persisten, auto-fallback ke long-polling |
| Backend | **Node.js** | Event-driven, cocok untuk ribuan koneksi concurrent tanpa blocking |
| Message Broker | **Redis Pub/Sub** | Sinkronisasi pesan antar server jika pakai multiple instance |
| Database | **PostgreSQL** | Simpan riwayat chat secara permanen |
| Cache/Presence | **Redis** | Online status & typing indicator (TTL-based) |
| Frontend | **React** + Socket.io-client | Optimistic UI, pesan tampil sebelum server konfirmasi |
| Media Files | **S3 + CDN** | Gambar/video tidak lewat WebSocket, upload terpisah lalu kirim URL |

### Contoh Implementasi

```js
// Server: terima pesan, broadcast ke room
io.on('connection', (socket) => {

  socket.on('join_room', (roomId) => {
    socket.join(roomId)
  })

  socket.on('send_message', async (data) => {
    // 1. Simpan ke database dulu
    const msg = await Message.create(data)

    // 2. Broadcast ke semua user di room yang sama
    io.to(data.roomId).emit('new_message', msg)
  })

  socket.on('typing', (data) => {
    // Broadcast "si X sedang mengetik..." ke room, kecuali pengirim
    socket.to(data.roomId).emit('user_typing', data.userId)
  })
})
```

> **Scaling:** Gunakan `socket.io-redis-adapter` agar event dari satu server bisa diterima koneksi di server lain. Tanpa ini, user di server berbeda tidak bisa saling chat.

**TL;DR** — WebSocket untuk koneksi realtime, Redis Pub/Sub untuk multi-server sync, PostgreSQL untuk persistensi. Jangan kirim file besar lewat WebSocket.

---

## 09 · Cara Diagnosis & Perbaikan Memory Leak pada Aplikasi Web

> **Analogi:** Memory leak seperti ember bocor yang terus diisi. Air (memori) terus ditambah tapi tidak pernah keluar, lama-lama penuh dan meluap (crash). Tugas kita: cari lubang bocornya dan tambal.

### Diagnosis di Node.js

1. **Monitor penggunaan memori** — Pantau `process.memoryUsage().heapUsed` secara periodik. Jika terus naik tanpa turun, ada memory leak.
2. **Buat heap snapshot** — Gunakan Chrome DevTools (connect via `--inspect`) atau library `heapdump`. Bandingkan dua snapshot: objek apa yang terus bertambah?
3. **Gunakan Clinic.js atau 0x** — Untuk profiling otomatis. Clinic Doctor bisa detect pola memory leak dan memberikan rekomendasi.

### Penyebab Umum & Fix

| Penyebab | Fix |
|---|---|
| ❌ Event listener tidak di-remove | ✅ Di React: return cleanup di `useEffect`. Di Node.js: simpan referensi listener dan hapus saat tidak dibutuhkan. |
| ❌ Closure memegang referensi besar | ✅ Gunakan `WeakMap` atau `WeakRef` untuk cache. Set TTL pada cache manual. |
| ❌ setInterval tidak di-clear | ✅ `useEffect(() => { const id = setInterval(...); return () => clearInterval(id); }, [])` |
| ❌ Cache tidak punya batas ukuran | ✅ Gunakan LRU cache (library `lru-cache`) agar cache tidak tumbuh tak terbatas. |

### Monitor Memori Sederhana

```js
setInterval(() => {
  const mem = process.memoryUsage()
  const mb = (b) => (Math.round(b / 1024 / 1024 * 100) / 100) + ' MB'
  console.log({
    heapUsed:  mb(mem.heapUsed),   // memori JS yang dipakai
    heapTotal: mb(mem.heapTotal),  // total heap yang dialokasikan
    rss:       mb(mem.rss),        // total memori process (termasuk C++)
  })
}, 5000) // cek tiap 5 detik
```

**TL;DR** — Monitor heapUsed, ambil heap snapshot untuk identifikasi, cari event listener yang tidak di-remove, closure yang pegang data besar, dan interval yang tidak di-clear.

---

## 10 · Desain Upload File 1GB+ yang Efisien & Resumable

> **Analogi:** Upload 1GB sekaligus = mengirim 1000 halaman dalam satu amplop besar yang mudah rusak. **Chunked upload** = kirim 10 halaman per amplop. Kalau amplop ke-7 hilang, cukup kirim ulang amplop ke-7 saja — bukan semua dari awal.

### Arsitektur Solusi

1. **Chunking di Client** — File dipecah menjadi potongan kecil (5–10MB per chunk) menggunakan `File.slice()` di browser. Tiap chunk punya nomor urut.
2. **Multipart Upload ke S3** — Tiap chunk di-upload paralel dan independen. Jika chunk gagal, hanya chunk itu yang di-retry.
3. **Presigned URL** — Server generate presigned URL per chunk. Client upload langsung ke S3 tanpa melewati server kita — menghemat bandwidth server drastis.
4. **Resume tracking di Redis** — Simpan progress upload (chunk mana yang sudah selesai). Jika koneksi putus, client bisa query chunk yang belum terupload dan lanjut dari sana.
5. **Complete Upload** — Setelah semua chunk terupload, server panggil S3 `CompleteMultipartUpload` untuk menggabungkan semua chunk menjadi satu file final.

### Contoh Implementasi Client

```js
async function uploadLargeFile(file) {
  const CHUNK_SIZE = 5 * 1024 * 1024  // 5MB per chunk
  const totalChunks = Math.ceil(file.size / CHUNK_SIZE)

  // 1. Inisiasi upload, dapat uploadId dari server
  const { uploadId } = await initUpload({ fileName: file.name, totalChunks })

  // 2. Cek resume: chunk mana yang sudah selesai?
  const { completedChunks } = await getProgress(uploadId)

  // 3. Upload tiap chunk
  for (let i = 0; i < totalChunks; i++) {
    if (completedChunks.includes(i)) continue  // skip jika sudah selesai

    const chunk = file.slice(i * CHUNK_SIZE, (i + 1) * CHUNK_SIZE)

    // Dapatkan presigned URL dari server, lalu upload langsung ke S3
    const { presignedUrl } = await getPresignedUrl({ uploadId, chunkIndex: i })
    await fetch(presignedUrl, { method: 'PUT', body: chunk })

    // Tandai chunk selesai di server (untuk resume)
    await markChunkDone({ uploadId, chunkIndex: i })

    updateProgressBar((i + 1) / totalChunks * 100)
  }

  // 4. Gabungkan semua chunk menjadi file final di S3
  await completeUpload(uploadId)
}
```

> Untuk implementasi yang lebih cepat, gunakan protokol open-source **tus.io** yang sudah menyediakan client (browser, mobile) dan server resumable upload out-of-the-box.

**TL;DR** — Pecah file jadi chunk 5MB, upload paralel via presigned URL langsung ke S3, simpan progress di Redis untuk resume, server hanya koordinasi bukan lewatkan data.

