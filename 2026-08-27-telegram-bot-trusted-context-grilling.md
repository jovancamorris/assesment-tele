# Pertanyaan buat Tim Hermes: Telegram Bot vs Sistem Kita

**Tanggal:** 27 Agustus 2026
**Buat:** Tim Hermes (yang pegang Telegram Bot)
**Dari:** Tim Backend Software Factory
**Sifatnya:** nanya-nanya buat klarifikasi, bukan nolak desainnya ya. Kita belum lihat kode
bot-nya, jadi semua yang di bawah ini masih dugaan berdasarkan cara kerja sistem sekarang — bisa
aja udah kalian antisipasi, tinggal dikonfirmasi doang.

**Cara jawabnya:** tiap pertanyaan udah kita kasih beberapa pilihan. Kalau setuju sama pilihan
yang ditandain **[SARAN KAMI]**, tinggal balas nomornya aja, contoh: "1B, 6A, 14A". Kalau gak ada
yang cocok, tulis jawaban sendiri.

---

## Istilah-istilah di dokumen ini

Biar gak perlu buka Google tiap baca paragraf:

- **JWT (JSON Web Token)** — kayak KTP digital yang dikasih pas abis login. Isinya siapa user itu
  (disebut "subject"). Tiap request ke sistem kita bawa JWT ini biar sistem tahu siapa yang minta.
- **Trusted context** — izin sementara yang diterbitin sistem, isinya kira-kira "user X boleh
  ngapain di project Z, sampai jam segini". Ternyata ada dua jenis di sistem kita, dijelasin di
  bawah.
- **RBAC (Role-Based Access Control)** — aturan siapa boleh akses apa, berdasarkan role dan
  keanggotaan project. Kalau RBAC-nya kelewat, artinya sistem gak sempat ngecek "ini orang emang
  berhak apa enggak".
- **Redis** — database yang super cepet, dipakai buat nyimpen data yang umurnya pendek, kayak
  trusted context yang cuma berlaku 2 jam.
- **Webhook** — cara Telegram ngasih tahu bot kita "ada pesan baru masuk nih", dengan ngirim
  request ke URL publik punya bot.
- **Token statis (shared secret)** — kata sandi tetap yang ditanam di config, dipakai antar-service
  biar saling percaya. Bedanya sama JWT: token statis gak tahu siapa user-nya, cuma tahu "yang
  manggil ini emang salah satu service kita".
- **Device lock** — sistem kita ngebatesin satu akun cuma boleh aktif di satu perangkat/aplikasi
  dalam satu waktu.

---

## Fakta dari kode (udah kita cek langsung, bukan nebak)

Tabel ini isinya apa yang sistem lakuin **sekarang**, sebelum ada Telegram Bot. Kolom terakhir
nunjuk lokasi persis di kode, kalau mau dicek ulang.

| # | Yang kejadian sekarang | Di mana |
|---|---|---|
| F1 | Semua fitur chat (bikin sesi, kirim pesan) wajib bawa JWT dan header `X-Device-Id` | `ChatSessionController.java:42-52` |
| F2 | Satu akun cuma boleh punya satu perangkat aktif. Perangkat kedua yang coba masuk dalam 30 menit, ditolak | `DeviceSessionGuard.java:44-60` |
| F3 | Pas bikin sesi chat baru, `tenantId` sama `projectId` diterima apa adanya dari yang dikirim, gak dicek ulang | `CreateSessionRequest.java:8-13` |
| F4 | Ada kolom `channel` buat nandain sesi ini dari Portal, Telegram, dll. Udah ada, tapi belum ada validasi nilainya | `ChatServiceImpl.java:139`, `ChatSession.java:44-46` |
| F5 | Artifact (dokumen hasil kerja agent) dicari pakai kombinasi ID + siapa yang bikin. Kalau dua user "dianggap" orang yang sama sama sistem, mereka bisa saling liat dokumen | `ArtifactService.java:216-217` |
| F6 | "Siapa yang bikin" tadi diisi dari JWT si user | `ArtifactService.java:203-205` |
| F7 | Fitur publish dokumen sekarang cuma jalan buat role BA (bikin FSD) sama SA (bikin TSD). Role lain belum didukung | `ArtifactTypePolicy.java:13-16` |
| F8 | Kalau role-nya gak didukung, sistem artifact bakal nolak dari awal | `ArtifactContextService.java:76-79` |
| F9 | Izin sementara buat dokumen cuma dicek dua hal: ada apa enggak, sama udah kedaluwarsa apa belum. Gak dicek siapa yang minta | `ArtifactContextService.java:45-63` |
| F10 | Sistem penyimpanan dokumen percaya ke siapa aja yang tahu token statisnya, gak peduli itu bener Portal apa bukan | `InternalServiceAuthFilter.java:26-33`, `PublicApiAuthFilter.java:24-32` |
| F11 | Izin sementara buat dokumen minimal berlaku 30 hari, walau yang diminta cuma 30 menit | `HttpArtifactContextClient.java:27`, `ArtifactContextService.java:16, 33` |
| F12 | Gak ada tombol/endpoint buat "batalin izin ini sekarang juga" | `ArtifactContextResource.java` |

---

## A. Soal Identitas User

### 1. Pas user ketik `/start` di Telegram, dia "login sebagai" siapa di mata sistem kita?

Ini pertanyaan paling penting di seluruh dokumen. Jawabannya nentuin jawaban banyak pertanyaan
lain di bawah.

- **A.** Semua user Telegram numpang satu akun bareng-bareng (satu "KTP" buat semua orang)
- **B.** Tiap user Telegram disambungin ke akun Keycloak-nya masing-masing, lewat proses "hubungin akun" **[SARAN KAMI]**
- **C.** Bot bikin KTP sendiri, bukan dari sistem login kita
- **D.** Belum diputusin, pas demo pakai token yang di-hardcode

Kenapa kita saranin B: liat F5 sama F6 di atas. Sistem bedain dokumen punya user A sama user B
**cuma** dari "siapa yang bikin" hasil baca JWT. Kalau semua user Telegram numpang satu akun (A),
otomatis semua orang bisa liat dokumen orang lain lewat Telegram. Bukan karena ada bug — sistemnya
emang didesain gitu, cuma asumsi "satu akun = satu orang" jadi gak berlaku lagi.

Kalau jawabannya C, tolong dijelasin siapa yang nerbitin "KTP" itu sama seberapa aman
penyimpanannya.

### 2. Kalau jawabannya B (hubungin akun), gimana caranya?

- **A.** Kirim link lewat email (udah ada sistemnya di `eksad-notification-serivce`), user klik link itu dari Telegram, terus sistem simpen "chat ID Telegram ini = akun email ini" **[SARAN KAMI]**
- **B.** User ketik email + password langsung di chat Telegram
- **C.** Kode OTP dibikin di Portal, user salin-tempel ke bot
- **D.** Belum ada caranya, siapa aja yang tahu bot-nya bisa langsung pakai

Kenapa bukan B: apa pun yang diketik di chat Telegram bakal kesimpen permanen di riwayat chat, di
server Telegram, sama di HP user. Password gak bisa "ditarik balik" lagi setelah diketik di situ.

### 3. Header `X-Device-Id` diisi apa sama bot?

Inget F2: satu akun cuma boleh punya satu perangkat aktif dalam 30 menit terakhir.

- **A.** Tiap user Telegram punya `deviceId` sendiri (misal dari chat ID-nya), jadi gak saling kunci **[SARAN KAMI, kalau jawaban 1 = B]**
- **B.** Satu `deviceId` yang sama dipakai buat seluruh bot
- **C.** Belum kepikiran / diisi asal-asalan

Kalau B digabung sama jawaban 1A (satu akun bareng): user Telegram kedua yang chat dalam 30 menit
yang sama bakal ketolak, munculnya error "Perangkat aktif berbeda terdeteksi" — padahal dari sisi
user, dia cuma buka chat biasa aja. Error ini bakal bikin bingung karena gak nyambung sama apa
yang dia lakuin.

Kalau B digabung sama jawaban 1B (akun sendiri-sendiri): masalah lain muncul — orang yang lagi
buka Portal di browser, terus kirim pesan lewat Telegram, bakal saling nendang satu sama lain,
padahal itu orang yang sama, cuma pakai dua aplikasi.

### 4. Boleh gak, satu orang aktif di Portal (website) sama Telegram bareng-bareng?

- **A.** Boleh, kuncinya diatur per aplikasi, bukan per orang **[SARAN KAMI]**
- **B.** Gak boleh, aplikasi yang terakhir dipakai yang menang, yang lama otomatis logout
- **C.** Gak boleh, aplikasi pertama yang menang, yang kedua ketolak (ini perilaku sistem sekarang)
- **D.** Belum kepikiran

Kalau jawabannya A, kita perlu ubah dikit logika kunci perangkatnya, soalnya sekarang tabelnya
cuma nyimpen satu perangkat per akun (`AgentOperatorDevice.java:9-15`), gak ada tempat buat nyimpen
dua kunci sekaligus buat satu orang. Perubahannya kecil kok, tapi perlu disepakatin dulu.

---

## B. Soal Tenant, Project, sama Izin Akses

### 5. Dari mana bot tahu tenant sama project mana yang harus dipakai pas `/start`?

Liat F3 — sistem kita nerima `tenantId` sama `projectId` apa adanya, gak dicek ulang siapa yang
emang berhak atas project itu.

- **A.** Bot nanya ke Portal "project apa aja yang boleh diakses user ini", terus dikasih pilihan ke user **[SARAN KAMI]**
- **B.** Bot punya tabel sendiri yang nyimpen mapping user-ke-project
- **C.** Di-hardcode satu project aja, khusus buat demo
- **D.** User ketik sendiri kode project-nya di chat

Risiko kalau B: kalau user udah dikeluarin dari project di Portal, tapi tabel di bot belum
diupdate, user itu masih bisa kerja lewat Telegram sampai ada yang inget hapus barisnya manual.

### 6. Siapa yang ngecek user itu emang berhak di project itu?

Ini penting: fitur chat di sistem chat kita **gak** ngecek keanggotaan project sama sekali. Yang
ngecek itu ada di Portal, sebelum request diterusin.

- **A.** Bot tetep lewat Portal dulu buat validasi, baru diterusin ke sistem chat **[SARAN KAMI, jangka pendek]**
- **B.** Pengecekan izinnya dipindah biar semua aplikasi (Portal, Telegram, dst) dapet perlindungan yang sama **[SARAN KAMI, jangka menengah]**
- **C.** Belum ada pengecekan, asumsinya siapa yang bisa chat pasti udah berhak
- **D.** Bot punya pengecekan sendiri, pakai datanya sendiri

Ini pertanyaan paling penting soal keamanan di dokumen ini. Kalau jawabannya C, artinya siapa aja
yang berhasil ngobrol sama bot otomatis dapet akses penuh ke project apa pun yang dia sebutin
UUID-nya — soalnya emang gak ada yang ngecek dia berhak apa enggak.

### 7. Role agent mana aja yang mau dibuka di Telegram?

Inget F7 sama F8. Ini bukan cuma soal kebijakan — ini bakal langsung ketauan errornya. Kalau role
selain BA/SA dipakai, sistem publish dokumen bakal nolak, tapi baru ketauan **belakangan** (pas mau
publish, bukan pas mulai sesi), jadi errornya bikin bingung.

- **A.** Cuma BA sama SA dulu, sama kayak di Portal sekarang **[SARAN KAMI]**
- **B.** Semua role, tapi kita perluas dulu daftar role yang didukung sebelum rilis
- **C.** Semua role, tapi Telegram emang gak dipakai buat publish dokumen (liat pertanyaan 9)

---

## C. Soal Izin Sementara (Trusted Context) sama Dokumen (Artifact)

### 8. Apa bot pernah langsung ngomong ke sistem penyimpanan dokumen?

- **A.** Gak pernah, semua lewat sistem chat dulu **[SARAN KAMI]**
- **B.** Ya, buat ambil file yang mau dikirim ke user
- **C.** Ya, buat upload dokumen baru

Kenapa kita tanya (liat F9, F10): sistem penyimpanan dokumen gak bisa bedain siapa yang manggil
dia. Dia cuma cek dua hal — kata sandi tetapnya bener, sama izinnya belum kedaluwarsa. Gak ada
catetan "ini diterbitin buat siapa, sama siapa yang nerbitin".

Artinya: **siapa aja yang pegang kata sandi tetap itu bisa nerbitin izin atas nama siapa aja.**
Selama ini aman karena cuma sistem chat yang pegang kata sandi itu. Kalau bot Telegram ikutan
pegang, berarti ada satu pintu tambahan yang nyambung ke internet publik (soalnya webhook Telegram
emang harus publik) yang juga nyimpen kata sandi sensitif itu.

### 9. Apa ID dokumen atau ID izin sementara pernah dikirim lewat chat Telegram?

- **A.** Gak pernah, disimpen di server bot, user cuma liat tombol/link **[SARAN KAMI]**
- **B.** Ya, dikirim sebagai teks biar user bisa lanjut dari perangkat lain
- **C.** Belum tahu

Pesan Telegram kesimpen di server Telegram, bisa di-forward ke orang lain, dan bisa ikut ke-backup
ke cloud HP user. ID izin yang bocor itu berlaku minimal 30 hari (F11) dan **gak bisa dicabut**
(F12) — jadi kalau bocor, ya jalan terus sampai kedaluwarsa sendiri.

### 10. Apa dokumen hasil kerja agent dikirim sebagai file Telegram?

- **A.** Gak, bot kirim link berbatas waktu ke domain kita sendiri **[SARAN KAMI]**
- **B.** Ya, filenya di-upload langsung sebagai dokumen di Telegram
- **C.** Belum diputusin, tapi rencananya iya

Kalau B: dokumen kayak FSD/TSD (yang biasanya isinya info project klien) bakal kesimpen permanen
di server Telegram, di luar kendali kita. Bukan berarti gak boleh, tapi ini keputusan yang perlu
disetujuin sadar-sadar sama yang berwenang, bukan efek samping dari cara paling gampang buat
implementasi.

### 11. Izin sementara buat dokumen minimal 30 hari — ini masalah buat Telegram gak?

Sistem chat sebenernya minta izin 30 menit doang, tapi sistem penyimpanan dokumen "dibulatin ke
atas" jadi 30 hari (F11). Kalau user Telegram cuma chat sekali terus gak pernah balik lagi, izin
itu tetep hidup sebulan penuh, dan gak ada cara buat nyabutnya (F12).

- **A.** Ini masalah. Perlu ditambah fitur "cabut izin sekarang", dipanggil pas user ketik `/stop` atau putus koneksi akun **[SARAN KAMI]**
- **B.** Gak masalah, izin itu gak berguna tanpa kata sandi tetap yang juga harus dicuri
- **C.** Baru tahu soal ini, perlu didiskusiin dulu

---

## D. Soal Operasional

### 12. Kolom `channel` diisi apa buat sesi dari Telegram?

Liat F4 — kolomnya udah ada, udah bisa dikirim, tapi belum ada aturan nilai apa aja yang boleh.
Maksimal 20 huruf, lebih dari itu bakal error dari database.

- **A.** `"TELEGRAM"`, dan kita tambahin aturan validasi biar nilai sembarangan gak bisa masuk **[SARAN KAMI]**
- **B.** `"TELEGRAM"`, tanpa validasi tambahan
- **C.** Dibiarin kosong / pakai default `"WEB_PORTAL"`

Jangan pilih C. Tanpa penanda channel yang bener, kita gak bisa bedain mana sesi dari Portal sama
mana dari Telegram — padahal itu justru yang paling dibutuhin kalau nanti ada masalah dan perlu
ditelusurin asalnya dari mana.

### 13. Kalau semua user Telegram numpang satu akun (jawaban 1A), gimana caranya tahu siapa ngapain?

Catatan aktivitas (audit log) bakal nyatet nama akun yang sama buat semua user Telegram. Log-nya
tetep ada, tapi udah gak bisa lagi jawab "siapa sebenernya yang ngelakuin ini".

- **A.** Gak relevan, soalnya kita pilih jawaban 1B (akun sendiri-sendiri) **[SARAN KAMI]**
- **B.** Tambah kolom khusus yang nyimpen chat ID Telegram-nya
- **C.** Diterima apa adanya buat demo, dibenerin nanti sebelum dipakai serius

### 14. Apa webhook Telegram-nya udah diverifikasi?

- **A.** Udah, pakai token rahasia yang dicek di tiap request masuk **[SARAN KAMI]**
- **B.** Udah, pakai daftar IP resmi Telegram
- **C.** Belum

URL webhook itu emang harus bisa diakses siapa aja dari internet (itu cara kerjanya webhook).
Tanpa verifikasi, siapa aja yang nemuin URL-nya bisa kirim pesan palsu ke bot kita — termasuk
pura-pura jadi user lain pas `/start`, yang bikin semua jawaban di pertanyaan 1 sama 2 jadi gak
ada artinya.

---

## E. Dampak ke Tiap Bagian Sistem

Bagian ini jawab "kalau semua di atas dikerjain, bagian sistem mana aja yang perlu diubah?".
Jawabannya: bukan cuma 2 tempat. Ada 9 bagian sistem yang kesenggol, dan yang paling gede
dampaknya justru bukan yang tadinya kita duga (sistem penyimpanan dokumen), tapi **Portal**.

### Sebelum lanjut: ternyata ada DUA jenis "izin sementara", bukan satu

Istilah "trusted context" udah kita sebut berkali-kali di atas. Ternyata itu dipakai buat dua hal
beda di sistem kita, dan ini sumber banyak dampak di bawah.

| | **Izin buat Dokumen** (Artifact Context) | **Izin buat Percakapan** (Agent Context) |
|---|---|---|
| Diterbitin sama | sistem chat (Agent Adapter) | **cuma Portal, gak ada yang lain** |
| Buktinya di kode | `HttpArtifactContextClient.java:43-54` | `AgentContextStore.java:26`, plus komentar di `package-info.java:4` yang bilang persis "Portal owns conversation ownership; factory-agent-adapter never touches Redis" |
| Dipakai buat | ngizinin publish dokumen FSD/TSD | ngizinin **semua** operasi ke GitLab: baca file, bikin branch, commit, bikin merge request |
| Umurnya | minimal 30 hari | 2 jam |

Yang kedua ini yang bikin "bot langsung ke sistem chat, lewatin Portal" bukan cuma soal keamanan,
tapi bikin fitur GitLab-nya **mati total**. Detailnya di E.6.

Satu catatan tambahan: sistem chat kita sama sekali gak pernah baca atau nyentuh izin percakapan
itu — udah kita cari di seluruh kodenya dan nihil. Jadi kalaupun mau, sistem chat **gak bisa** jadi
pengganti Portal buat nerbitin izin ini. Cuma Portal yang bisa.

---

### E.1 Dampak ke User Management (sistem akun sama login)

**Yang kesenggol:** `usermanagement-svc`, `eksad-notification-serivce`

Semua sistem lain — chat, dokumen, keanggotaan project — sebenernya cuma ngandelin satu hal: siapa
yang kecatet di JWT. Jadi semua pertanyaan soal identitas di atas (1-4) muaranya ke sini.

Yang udah ada dan bisa langsung dipakai:
- Fitur login yang udah jadi, tinggal panggil (`AuthController.java:42-46`)
- Fitur perpanjang sesi login, tapi caranya sekarang lewat cookie browser (`AuthController.java:48-49`)
  — bot gak bisa pakai cookie per user kayak browser, jadi ini perlu jalan lain
- Sistem kirim email verifikasi yang udah ada di `eksad-notification-serivce`, bisa dipakai ulang
  buat proses "hubungin akun Telegram"

Yang belum ada sama sekali: cara nyambungin satu akun ke "identitas luar" kayak Telegram. Udah
kita cari di seluruh kode `usermanagement-svc` dan gak nemu apa-apa soal ini. Jadi ini kerjaan
baru, di mana pun nanti ditaruh.

#### 15. Disimpen dimana data "chat ID Telegram ini = akun siapa"?

- **A.** Di `usermanagement-svc`, soalnya semua data identitas lain juga di situ **[SARAN KAMI]**
- **B.** Di database punya bot sendiri
- **C.** Disimpen sebagai data tambahan di Keycloak

Kenapa bukan B: kalau datanya kesimpen terpisah dari sumber identitas asli, gak ada yang otomatis
ngehapus pas akun itu dinonaktifin. User yang udah resign tetep bisa pakai Telegram sampai ada yang
inget hapus datanya manual.

#### 16. Kalau sesi login user kedaluwarsa, gimana caranya bot memperpanjangnya?

- **A.** Bot simpen token perpanjangan per user di servernya, terus kirim lewat header (bukan cookie) — perlu perubahan kecil di sistem login **[SARAN KAMI]**
- **B.** User diminta login ulang lewat link email tiap kali sesinya abis
- **C.** Pakai satu token khusus yang umurnya panjang banget, khusus buat bot
- **D.** Belum kepikiran

Soal C: token yang umurnya panjang banget, kesimpen di server yang nerima request dari internet
publik (soalnya webhook Telegram), itu kombinasi yang perlu hati-hati.

---

### E.2 Dampak ke Web Portal

**Yang kesenggol:** `factory-portal-service`

Ini bagian yang paling banyak berubah, dan sebelumnya gak masuk dugaan awal kita.

Portal itu bukan cuma tampilan website doang. Portal pegang tiga tanggung jawab yang gak dipegang
siapa pun lain di sistem kita:

1. **Nerbitin izin percakapan.** Cuma Portal yang boleh nulis izin ini ke Redis
   (`AgentContextStore.java:26`). Ini bukan kebetulan, emang didesain gitu — komentar di kodenya
   bilang jelas: "Portal owns conversation ownership; factory-agent-adapter never touches Redis"
   (`package-info.java:4`).

2. **Ngecek kepemilikan percakapan.** Tiap ada pesan baru, Portal ngecek ulang "ini emang
   percakapan punya orang yang lagi request apa bukan". Kalau Redis mati, Portal milih nolak semua
   request (bukan ngizinin) — pilihan yang aman (`AgentConversationService.java:101-122`).

3. **Milih agent mana yang boleh dipakai.** Browser (atau bot) boleh *minta* agent tertentu, tapi
   gak bisa *maksa*. Portal yang ngitung ulang agent mana yang bener-bener boleh dipakai,
   berdasarkan role user sama pengaturan project (`AgentConversationService.java:150-166`).

Data keanggotaan project juga disimpen di Portal, bukan di sistem login
(`UserProjectMapping.java:12`).

#### 17. Kalau bot langsung ngomong ke sistem chat, ketiga tanggung jawab Portal di atas dikerjain siapa?

- **A.** Bot manggil Portal dulu, Portal yang neruskan ke sistem chat — bot dianggap kayak browser biasa **[SARAN KAMI]**
- **B.** Bot langsung ke sistem chat, tapi minta Portal buka jalur khusus buat nerbitin izin percakapan
- **C.** Ketiga tanggung jawab itu dibikin ulang di dalem bot
- **D.** Gak dikerjain sama sekali

A ini pilihan yang paling dikit ubah kodenya — nol perubahan di Portal, sistem chat, sama sistem
GitLab. Bot cukup dianggap kayak browser biasa. Kalau ada alesan (misal soal kecepatan respons)
kenapa A gak bisa dipakai, tolong kasih tahu ya, soalnya kita belum tahu alesannya.

C secara teknis bisa dikerjain, tapi artinya aturan "siapa boleh apa" ada di dua tempat beda yang
harus selalu dijaga sama persis. Kalau suatu saat keduanya beda dikit aja, itu baru ketauan pas
udah jadi masalah beneran.

#### 18. Apa percakapan dari Telegram muncul juga di Portal (website)?

- **A.** Ya, satu daftar percakapan, dibedain lewat kolom `channel` **[SARAN KAMI]**
- **B.** Gak, dua dunia yang bener-bener kepisah
- **C.** Belum kepikiran

Kalau B: user yang mulai diskusi requirement lewat Telegram gak bisa lanjutin di website, begitu
juga sebaliknya. Perlu dipastiin itu emang maunya tim produk, bukan sekadar kebetulan teknis
karena belum sempet kepikiran.

---

### E.3 Dampak ke Chat

**Yang kesenggol:** `factory-agent-adapter`

Pertanyaan 3, 4, sama 12 di atas semuanya soal bagian ini. Tambahan yang belum disebut:

- **Riwayat percakapan** — sekarang cara ambil pesan lama itu pakai sistem "scroll ke atas", yang
  cocok buat aplikasi chat biasa. Telegram gak punya cara yang sama persis, jadi perlu dipikirin
  gimana user bisa liat riwayat lamanya.
- **Kirim file** — sekarang kalau user kirim file lewat chat, filenya diproses langsung di memori
  server, gak pernah disimpen permanen (`ChatSessionController.java:83-85`). Bot Telegram perlu
  download dulu filenya dari server Telegram, baru diterusin ke sistem kita. Ukuran file yang
  boleh dikirim dibatesin Telegram sendiri (20 MB), bukan sama sistem kita.

#### 19. Gimana caranya user Telegram liat percakapan lamanya?

- **A.** Perintah khusus kayak `/history` yang nampilin beberapa pesan terakhir **[SARAN KAMI]**
- **B.** Gak ada, cukup scroll ke atas di aplikasi Telegram
- **C.** Kasih link ke Portal

Soal B, ada hal yang gampang kelewat: kalau user hapus chat-nya di Telegram, dari sisi dia
riwayatnya ilang, padahal datanya masih kesimpen di server kita. Perlu jelas mana yang jadi
"sumber kebenaran" data itu.

---

### E.4 Dampak ke Pembuatan Dokumen (Artifact Generation)

**Yang kesenggol:** `factory-agent-adapter`, sistem penyimpanan dokumen sama tool MCP-nya

Cara agent bikin sama nyimpen dokumen gak berubah cuma karena channel-nya beda — agent tetep
manggil fitur yang sama kayak biasa. Yang berubah itu soal siapa bisa liat hasilnya, sama gimana
caranya hasil itu sampai ke user.

Yang perlu diperhatiin:

- **Batas siapa liat dokumen siapa** sepenuhnya ngandelin "siapa yang bikin" (udah dibahas di
  pertanyaan 1).
- **Role yang didukung** cuma BA sama SA (udah dibahas di pertanyaan 7).
- **Batas ukuran file**: dokumen teks maksimal 1 MB, file biner maksimal 10 MB. Telegram sendiri
  ngebolehin sampai 50 MB, jadi batas kita yang duluan kena — bukan masalah, tapi pesan errornya
  perlu diterjemahin jadi bahasa yang masuk akal buat user di chat, bukan pesan error teknis.

#### 20. Gimana caranya dokumen yang udah jadi sampai ke user Telegram?

- **A.** Bot kirim link berbatas waktu ke website kita, user download lewat browser **[SARAN KAMI]**
- **B.** Bot download filenya, terus kirim sebagai file langsung di Telegram
- **C.** Bot cuma bilang "dokumennya udah jadi", user ambil sendiri lewat Portal

Ini pertanyaan 10 dilihat dari sisi teknisnya. Kalau B dipilih, dokumen kayak FSD/TSD (yang biasa
isinya info project klien) bakal kesimpen permanen di server Telegram. Bukan hal yang mustahil,
tapi harus jadi keputusan yang disadarin, bukan kebetulan.

---

### E.5 Dampak ke Trusted Context (dan yang nyambung ke dia)

**Yang kesenggol:** sistem penyimpanan dokumen, Portal

Inget, ada dua jenis izin sementara. Masalahnya beda-beda:

**Izin buat Dokumen.** Minimal berlaku 30 hari, walau yang diminta cuma 30 menit (udah dibahas di
pertanyaan 11). Gak ada cara nyabutnya lebih awal. Dan sistem gak ngecek siapa yang pakai izin
itu — jadi kalau bocor, siapa aja yang juga tahu kata sandi tetapnya bisa makainya.

**Izin buat Percakapan.** Berlaku 2 jam, disimpen di Redis. Kalau Redis lagi mati, sistem milih
nolak semua request daripada ngizinin tanpa cek (pilihan yang aman) — tapi ini juga berarti Redis
jadi satu titik yang kalau rusak, semua channel (termasuk Telegram nanti) ikutan keganggu, bukan
cuma Portal.

#### 21. Apa yang kejadian ke kedua izin itu pas user ketik `/stop` atau mutusin koneksi akun?

- **A.** Izin percakapan langsung dihapus dari Redis, izin dokumen dicabut lewat fitur baru yang perlu dibikin **[SARAN KAMI]**
- **B.** Izin percakapan dihapus, izin dokumen dibiarin kedaluwarsa sendiri (30 hari)
- **C.** Keduanya dibiarin aja
- **D.** Belum ada perintah `/stop`

#### 22. Apa bot nyimpen salinan ID izin di tempat lain selain Redis?

- **A.** Gak, selalu diambil ulang dari Portal pas dibutuhin **[SARAN KAMI]**
- **B.** Ya, di database bot, biar gak perlu bolak-balik minta
- **C.** Ya, disimpen sementara di memori proses bot

Kalau B: salinan itu gak ikut ilang pas izin aslinya dibatalin di Redis. Bot bakal terus ngirim ID
izin yang udah mati, dan sistem GitLab bakal nolaknya dengan pesan error yang bener secara teknis,
tapi user liatnya kegagalan yang gak bisa dia benerin sendiri.

---

### E.6 Dampak ke GitLab SCM (sistem ngelola kode/repo)

**Yang kesenggol:** sistem SCM sama tool MCP-nya buat GitLab

Ini bagian paling penting buat dibaca, soalnya dampaknya bukan cuma soal keamanan — ini soal fitur
yang bisa **mati total** kalau gak diantisipasi.

Gini alurnya, langkah demi langkah:

1. Portal nerbitin izin percakapan, ditulis ke Redis.
2. Izin itu dikirim ke sistem chat lewat sebuah header khusus di tiap request.
3. Sistem chat neruskan header itu ke agent yang lagi jalan.
4. Kalau agent mau ngelakuin sesuatu di GitLab (baca file, commit, bikin merge request), dia
   manggil tool GitLab, dan header itu ikut dikirim.
5. Sistem GitLab kita baca header itu, cek ke Redis: izinnya ada dan belum kedaluwarsa?
6. Kalau gak ada / udah kedaluwarsa → ditolak.

**Karena Portal itu satu-satunya yang boleh nulis izin percakapan ke Redis, kalau sesi Telegram
gak lewat Portal, izin itu gak akan pernah ada. Akibatnya: semua operasi GitLab pasti ditolak.**
Agent masih bisa ngobrol biasa, masih bisa bikin dokumen, tapi gak bisa commit satu baris kode pun
ke GitLab.

Kalau demo kemarin cuma nguji ngobrol biasa (belum coba minta agent commit sesuatu), masalah ini
belum bakal keliatan.

#### 23. Apa agent yang jalan lewat Telegram perlu bisa commit ke GitLab?

- **A.** Ya, perlu. Kalau gitu, pertanyaan 17 wajib dijawab A atau B — gak ada jalan pintas lain **[SARAN KAMI]**
- **B.** Gak buat sekarang. Telegram cuma buat tanya jawab sama bikin dokumen dulu
- **C.** Belum kepikiran

Kalau B: pastiin fitur GitLab-nya emang bener-bener dimatiin buat sesi Telegram, jangan dibiarin
gagal sendiri. Kalau dibiarin, agent bakal terus nyoba dan gagal berulang-ulang, dan dari sisi user
keliatannya kayak bot yang rusak, tanpa penjelasan yang jelas kenapa.

Catatan: gak ada satu baris kode pun di sistem GitLab yang perlu diubah buat skenario mana pun di
atas. Sistem itu ikut "rusak" bukan karena salah kodenya, tapi karena keputusan yang diambil di
tempat lain (Portal dilewatin apa enggak).

---

### Ringkasan: bagian sistem mana aja yang kesenggol

| Bagian sistem | Status | Kenapa |
|---|---|---|
| Sistem Chat (`factory-agent-adapter`) | **Perlu diubah** | kunci perangkat, validasi kolom channel, cara kenali identitas |
| Portal (`factory-portal-service`) | **Perlu diubah** | satu-satunya yang boleh nerbitin izin percakapan; pengecekan kepemilikan & pemilihan agent |
| Sistem Login (`usermanagement-svc`) | **Perlu diubah** | data "chat ID = akun siapa"; cara perpanjang sesi login tanpa cookie |
| Sistem Penyimpanan Dokumen | Tergantung jawaban | fitur cabut izin (kalau jawaban 21 = A) |
| Sistem Kirim Email (`eksad-notification-serivce`) | Tergantung jawaban | dipakai buat proses hubungin akun (kalau jawaban 2 = A) |
| Dokumen aturan role/tool | Tergantung jawaban | belum punya aturan khusus per channel |
| Sistem GitLab (SCM) | **Ikut rusak, tanpa perlu diubah** | nolak semua request kalau gak ada izin percakapan dari Portal |
| Tool MCP GitLab | **Ikut rusak, tanpa perlu diubah** | cuma neruskan data, ikut kena dampak sistem GitLab di atas |
| Tool MCP Dokumen | Gak berubah | batas ukuran & aturan yang ada sekarang tetep berlaku |

---

## Ringkasan: mana yang wajib dijawab dulu sebelum rilis

**Wajib dijawab — jangan dipakai di luar demo internal sebelum ini jelas:**

- **Pertanyaan 1** — kalau identitasnya numpang bareng, dokumen antar user Telegram bisa saling keliatan
- **Pertanyaan 6 sama 17** — kalau Portal dilewatin, pengecekan "siapa boleh akses apa" ikut kelewat
- **Pertanyaan 14** — webhook tanpa verifikasi bikin semua jawaban soal identitas di atas jadi gak ada artinya

**Wajib dijawab — soal fitur jalan apa enggak, bukan soal keamanan:**

- **Pertanyaan 23** — tanpa izin dari Portal, semua fitur GitLab bakal gagal terus
- **Pertanyaan 7** — role selain BA/SA bakal gagal belakangan, bukan di awal, dan errornya bikin bingung

**Bisa dikerjain sambil jalan:** sisanya.

---

Kalau demo kemarin emang cuma buat satu user dengan satu project yang di-hardcode, tolong bilang
aja — sebagian besar pertanyaan di atas otomatis gak berlaku, dan kita bisa langsung bahas versi
yang bakal dipakai serius. Dokumen ini dibikin bukan buat ngehambat, tapi biar perubahan yang
perlu dikerjain di Portal, sistem chat, sistem login, sama sistem penyimpanan dokumen bisa
direncanain bareng dari awal, bukan ketauan pas udah proses integrasi.
