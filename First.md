# SMS PTMO — Projek dari A hingga Z

**Versi:** v1.0 — 7 Oktober 2026
**Untuk:** Boss Ucop (Yusuf Azmi), Minda Optima
**Fungsi fail ni:** satu dokumen yang terangkan keseluruhan projek School Management System PTMO 2026 — apa, kenapa, status sebenar, plan, vendor yang dikaji, risiko, dan apa langkah seterusnya. Baca sekali dari atas ke bawah, kau faham semua.
**Status:** Rujukan. **Bukan keputusan.** Tiada vendor dipilih. Tiada checkpoint `CLAUDE.md` §7 dianggap lulus (SaaS vs custom, vendor akhir, kontrak, bayaran milestone, pilot go-live, rollout).

> Fail ni ringkasan yang dikonsolidasi. Sumber asal: `SMS-Whole-System-Validation-Map.md` (17 Sep 2026), `Vendor-Evaluation-Log-BM.md` v0.3, `Prompt-Grill-Vendor-Teardown-BM.md` v1.1, `SMS-Grill-Questions.md`. Kalau ada beza, sumber asal menang. Fail ni tak gantikan mana-mana.

---

## Isi kandungan

1. Satu perenggan: projek ni apa
2. Masalah yang nak diselesaikan
3. Skop: apa masuk, apa keluar
4. Siapa guna (5 role)
5. Gambaran sistem (architecture)
6. Flow operasi utama (A–L, ringkas)
7. Fasa pembinaan F1–F6
8. Duit, keselamatan, PDPA
9. Status sebenar hari ni (jujur)
10. Blocker dan keputusan yang masih terbuka
11. Kajian vendor: kenapa buat, dan 4 vendor
12. Matriks perbandingan (versi ringkas)
13. Curi / buat lebih baik / elak
14. Hipotesis grill (H1–H6) — bahagian paling penting
15. Kill-criteria (cadangan)
16. Cara guna prompt Claude Code
17. Peta dokumen dan konflik antara dokumen
18. Apa kena buat seterusnya
19. Glosari

---

## 1. Satu perenggan: projek ni apa

Minda Optima jalankan program bacaan kanak-kanak (Pantas Baca) di kira-kira **15–17 cawangan** dengan **2,300–2,600 murid aktif**. Hari ni semua benda hidup dalam **Google Sheets + WhatsApp**: yuran, jadual, kehadiran, progress, komunikasi ibu bapa. Boss nak satu sistem dalaman (**SMS PTMO**) sebagai *system of record*: murid, yuran, jadual, kehadiran, progress, dan portal ibu bapa. Dibina sendiri (in-house), billing dulu, dengan permukaan `/admin`, `/guru`, `/ibubapa`.

Sebelum mula coding, Boss nak **grill idea ni sampai yakin**, dan staf dah kumpul bahan 4 vendor sistem sedia ada sebagai rujukan. Vendor tu bukan untuk dibeli (setakat ni), tapi untuk **dicuri idea, dibuat lebih baik, dan dielak kesalahannya**. Tapi hasil kajian tu juga akan uji satu soalan keras: *betul ke bina sendiri lebih baik daripada beli?*

---

## 2. Masalah yang nak diselesaikan

- Tiada satu sumber kebenaran untuk murid, yuran, jadual, attendance, progress.
- Data tersebar dalam Sheets (contoh: `AA2-YURAN`, `CC1-BOOKING`, `AA5-GURU`) dan thread WhatsApp.
- Ibu bapa tanya benda sama berulang (jadual, bayaran, progress) kepada Branch Admin.
- Kutipan yuran, reconciliation bank dan outstanding dibuat manual.
- Founder tak ada gambaran satu muka untuk 15–17 cawangan.

**Soalan pertama dalam grill (jangan skip):** *apa satu kerosakan operasi paling besar hari ini (WhatsApp + Sheets) yang mesti hilang dalam 90 hari pertama sistem?* Kalau jawapan tak tajam, semua keputusan lain goyang.

---

## 3. Skop: apa masuk, apa keluar

### Dalam skop
- Cawangan, guardian, murid, guru, enrolment, pakej (BASIC / FAST / SUPER / SUPER DUPER), buku (1A dan seterusnya)
- Billing: invoice, prorate, diskaun, bayaran separa, resit, outstanding, refund/waive
- Reconciliation penyata bank (dengan confidence score)
- FPX pay-now dan direct debit, WhatsApp template
- Jadual, slot, sesi kelas, attendance, progress harian
- Portal ibu bapa (PWA), support ticket, kelas ganti, announcement, FAQ
- Payroll guru (rate per kelas)
- Laporan HQ, Founder pulse
- MCP read-only untuk AI dalaman (kemudian)

### Luar skop (sengaja)
| Benda | Duduk di mana |
|---|---|
| Lead dan trial | `minda-optima-leads` (SMS tarik masuk **sehala** bila status `ENROL`) |
| Staff Portal / HR task / SOP board | Projek berasingan, ditangguh |
| KPI semasa | `report-ptmo-phase2` |
| Doraemon/Hermes v1 | SOP-only, tiada akses DB SMS |
| Meal, nap, mood, MC upload, photo drop-off | Konsep childcare, dibuang (brief Julai sudah lapuk) |
| SaaS untuk sekolah luar | Bukan sekarang (franchise-ready, bukan external SaaS) |

---

## 4. Siapa guna (5 role)

| Role | Nampak apa | Kerja utama |
|---|---|---|
| **Founder** | Semua cawangan | Ringkasan bisnes, exception kritikal |
| **Admin HQ** | Semua cawangan | Master data, verify bayaran, laporan |
| **Branch Admin** | Cawangan sendiri | Murid, enrolment, invois, support |
| **Teacher** | Cawangan + kelas sendiri | Jadual, check-in, attendance, progress |
| **Parent** | Anak sendiri sahaja | Jadual, attendance, progress, bil, support, kelas ganti |

Permission khas dirancang: `centre_duty`, `finance_verify`, `master_data_write`. Staf boleh ditugaskan ke lebih satu cawangan.

**Akses yang mesti ditolak (ujian keselamatan):**
- Parent A buka anak Parent B (tukar URL/ID) → ditolak.
- Branch Admin X baca cawangan Y → ditolak.
- Teacher buka laporan bayaran → ditolak.
- `internal_note` tak pernah sampai parent.
- Admin HQ, Founder, Branch Admin wajib 2FA.

---

## 5. Gambaran sistem (architecture)

**Stack dirancang:** Laravel + PHP 8.4, Inertia + React + TypeScript, **PostgreSQL 17 dengan native RLS**, Redis (queue/scheduler), Cloudflare R2 (fail), Tailwind + shadcn/ui, Sentry. Hosting Malaysia (PDPA).

**Tiga permukaan:** `/admin` (desktop/tablet), `/guru` (mobile-first), `/ibubapa` (PWA BM/EN).

**Model isolation dua lapis:**
1. Laravel: `BranchContext` + global scope + model policy.
2. PostgreSQL: RLS ikut `branch_id`. Ujian keselamatan sengaja matikan scope aplikasi untuk buktikan DB masih menghalang.

Semua jadual milik cawangan ada `branch_id`. Semua tulis melalui policy dan masuk audit log. Duit disimpan sebagai **integer sen**, bukan float.

```
Lead / Book Trial  (minda-optima-leads)
      | sehala, bila ENROL sahaja
      v
SMS PTMO ── /admin  (Founder, Admin HQ, Branch Admin)
         ├─ /guru    (Teacher)
         ├─ /ibubapa (Parent PWA)
         ├─ Billing + bayaran + resit + bank match
         ├─ Kelas, attendance, progress
         └─ Support, kelas ganti, announcement, FAQ
                |
                └─> MCP read-only untuk agent dalaman (lepas F2)
```

**Yang diwarisi (INHERITED) dari fork "Jadual Kelas Student Auto":** login, Google sign-in, 2FA, passkey, RBAC, activity log, import Excel, export PDF/CSV, integration credentials, CI/Docker. Ini platform, **bukan bukti SMS dah siap**.

---

## 6. Flow operasi utama (A–L, ringkas)

- **A. Lead jadi murid:** lead `ENROL` ditarik sehala ke SMS, guardian + student dicipta/dipadankan, Branch Admin pilih cawangan, programme, pakej, mode. Tiada write-back ke Leads.
- **B. Bayaran pertama + direct debit:** invois pertama = pendaftaran + buku + yuran bulan pertama (item berasingan). Admin semak prorate. Parent bayar FPX. Serentak dapat jemputan mandate direct debit (ambil beberapa hari). Bulan kedua auto-debit. Gagal → retry + reminder, **tiada penalti**.
- **C. Billing bulanan:** job baca enrolment aktif, snapshot harga, satu invois seorang sebulan, unique key halang duplicate. Hantar WhatsApp. Webhook idempotent. Resit PDF lepas bayaran disahkan.
- **D. Reconciliation bank:** upload CSV, padan amount + rujukan + kod murid, confidence score, baris tak padan kekal dalam queue (tiada transaksi hilang senyap), finance verify.
- **E. Tangguh dan kembali:** invois berhenti dijana, slot dilepas, bila kembali cari slot kosong, billing ikut polisi prorate.
- **F. Operasi harian guru:** login, lihat jadual, check-in selfie + GPS, tanda attendance, isi progress, nota internal kekal private, sesi complete jadi input payroll.
- **G. Jadual + kapasiti:** admin set bilik/slot/cuti, sistem jana sesi rolling 8 minggu, enjin cadang placement tapi **admin yang confirm**, Postgres halang clash guru/bilik.
- **H. Parent semak anak:** pilih anak, nampak kelas, attendance, progress, invois, resit, sijil, announcement, FAQ.
- **I. Support:** 9 kategori → L1 Branch Admin → L2 Admin HQ → L3 Founder. Status lembut untuk parent: Diterima → Dalam semakan → Sedang diproses → Selesai → Ditutup.
- **J. Kelas ganti:** parent baca polisi, request, admin approve/reject, tarikh disahkan, masuk jadual guru. Tak hadir = no-show, **bukan open credit**.
- **K. Payroll guru:** sesi complete → rate per kelas + elaun − potongan → admin semak exception → export ke accounting. Kadar rasmi belum ada.
- **L. AI/Doraemon:** v1 SOP-only. Selepas F2, MCP v1 baca ringkasan billing role-scoped. Selepas F4, MCP v2 (OAuth) tambah attendance/progress. Semua read-only dan diaudit.

---

## 7. Fasa pembinaan F1–F6

| Fasa | Anggaran lama | Hasil |
|---|---:|---|
| **Pre-code gate** | — | System of Record sign-off, data audit, pilot dipilih |
| **F1** | ~4 minggu | Backbone: cawangan, master data, murid/guardian/guru/enrolment, migrasi data, staging, Sentry |
| **F2** | ~5 minggu | Billing, invois, resit, bank matching, outstanding, sijil |
| **F2.4** | ~4 hari | Founder one-page pulse |
| **F2.5** | ~4 minggu | WhatsApp, direct debit, FPX pay-now, MCP v1 |
| **F3** | ~5 minggu | Teacher app, jadual, attendance, progress, waiting list |
| **F4** | ~8 minggu | Parent PWA, support, kelas ganti, announcement, FAQ, lead bridge |
| **F5** | ~3 minggu | Payroll guru |
| **F6** | belum anggar | HQ analytics penuh |

Jumlah F1–F5 anggaran lama: ~29 minggu (~7 bulan) solo part-time. **Ini anggaran lama, bukan komitmen.**

**Rollout:** satu pilot branch yang *paling kotor* (ada BASIC + FAST, hybrid/online, buku resit kertas, murid suspended/stopped, adik-beradik), jalan selari dengan Sheets **dua kitaran bulanan**, angka mesti match tepat dua bulan berturut-turut, baru tiga cawangan serentak, baru semua.

**Definition of done** (contoh F2): generator tak duplicate bila dijalankan dua kali; jumlah invois pertama match resit lama sen-per-sen; laporan outstanding pilot match sheet manual; penyata bank sebenar import dengan semua baris tak padan nampak.

---

## 8. Duit, keselamatan, PDPA

- **Bayaran:** FPX pay-now untuk bayaran pertama; direct debit untuk berulang; fallback FPX. Provider dalam plan: **Bayarcash dulu**, tapi semua kadar dan terma mesti disahkan semula sebelum tandatangan. Konvensyen projek: Billplz / ToyyibPay / Bayarcash (bukan Stripe).
- **Notifikasi:** WhatsApp template jadi saluran utama (nombor Cloud API baru, bukan nombor operasi sedia ada), email sekunder.
- **Data kanak-kanak = data paling sensitif.** Encrypt IC dan credential integrasi, jangan log kandungan medan sensitif, retention sebagai job berjadual, aliran subject-access export + delete, runbook breach, secret scanning, mask nombor akaun bank.
- **Tiada faedah/penalti kompaun** pada lewat bayar.
- **Audit keselamatan wajib** sebelum 2,600 akaun parent didedahkan.
- **Syarat kontrak (jika ada developer/vendor):** IP 100% milik Minda Optima, deposit maksimum 20–30%, waranti 90 hari.
- **Governance masih terbuka:** siapa pemilik data protection/insiden, tempoh retention per kategori, frekuensi backup + siapa uji restore, ayat consent parent untuk gambar/GPS/progress, SLA tutup akaun staf.

---

## 9. Status sebenar hari ni (jujur)

**Yang confirmed ada:** repo `sms-ptmo` (branch `main` bersih semasa audit), platform Laravel yang kuat, branding/UI token Minda Optima, dokumen plan, decision log, status file.

**Yang belum boleh dianggap siap:**
- 16 domain migration yang nampak "lengkap" ialah domain **Jadual Auto untuk satu sekolah**, bukan SMS.
- Role warisan masih Super Admin/Admin/Guru, bukan lima role SMS.
- Tiada model `branch_id` multi-cawangan yang final.
- Domain SMS F1–F6 belum dibina.
- `SYSTEM-OF-RECORD.md` masih **DRAF, menunggu sign-off**.

**Kesimpulan:** ini **bukan** School Management System production-ready. Ia platform kuat + plan domain terperinci + feature warisan. Coding domain SMS **secara rasmi masih blocked** sehingga source-of-truth dan audit data selesai.

> Peraturan audit: ujian lulus warisan, skrin Jadual Auto, atau UI cantik **bukan bukti** domain SMS dah dibina.

---

## 10. Blocker dan keputusan yang masih terbuka

**Sebelum F1 (gate):**
- Sign-off `SYSTEM-OF-RECORD.md`; catat Jadual Auto sebagai pemilik sementara kelas/enrolment.
- Sahkan lokasi dan kelengkapan data 2,300–2,600 murid (`AA2-YURAN`, `CC1-BOOKING`, `AA5-GURU`).
- Senarai rasmi cawangan + kod + alias. Pilih pilot branch.

**Sebelum F2 (duit):** yuran rasmi BASIC/FAST/SUPER/SUPER DUPER, yuran pendaftaran, peraturan buku, status kenaikan yuran, peraturan diskaun adik-beradik, polisi refund, julat resit kertas aktif per cawangan.

**Sebelum F3:** polisi absent berulang, empat status attendance rasmi, radius GPS dan privasi, retention selfie, flow batal kelas.

**Sebelum F4:** syarat kelayakan kelas ganti (notis, had bulanan, expiry), SLA L1→L2→L3, senarai skrin parent, **PWA vs Capacitor** (putuskan awal, mahal kalau retrofit).

**Sebelum F5:** rate sheet guru, elaun/potongan, flow kelulusan, format export.

**Di luar kod:** semak semula LHDN e-Invoice dengan akauntan, EPF/SOCSO guru part-time, yuran/kontrak/DPA provider.

**Fitur brief Julai yang Boss perlu putuskan (kekal / buang / kemudian):** QR/PIN drop-off, notifikasi check-in fizikal, photo drop-off, MC upload, laporan meal/nap/mood (cadangan: buang), chat bebas parent↔guru (plan pilih support berstruktur), app native iOS/Android.

---

## 11. Kajian vendor

### 11.1 Kenapa buat

Staf Boss (Aznir dan lain-lain) kumpul bahan di Drive "Kajian Sistem Pengurusan Pusat". Tujuan: rujukan feature, bukan beli. Tapi hasilnya mesti dibaca dengan jujur:

- **❓ bukan ❌.** Banyak "kita menang" dalam jadual sebenarnya sebab *vendor senyap*, bukan sebab vendor tak ada.
- **Bukti tak setara:** AOne ada quote + proposal + website; SimTrain website sahaja; ClassFlow 5 screenshot; Remmu 40 daripada 50 screenshot dari demo Zoom.
- **Staf cari kategori yang salah sikit:** ClassFlow = daycare/tadika, Remmu = kelab sukan. AOne dan SimTrain paling dekat dengan learning-centre/tuisyen.
- **Semua dakwaan belum diuji hands-on.**

### 11.2 Vendor 1 — AOne (My AOne Learning Sdn Bhd)

**Bukti:** quote AONEQ20390 (1 Okt 2026), proposal PDF, website. Video Zoom (89MB) **belum dikaji**.

**Rujukan kos (bukan kriteria):** RM32,290.92 untuk 2 tahun termasuk SST. Termasuk 20 cawangan, cap 2,000 murid (**kurang daripada 2,300–2,600 sebenar**), onboarding RM1,000. Sah sampai **31 Okt 2026**. Bayar penuh depan, tiada refund. Harga selepas tahun ke-2 tak dinyatakan. Lebih kurang RM16k setahun, jadi hujah "custom lagi murah" **tak boleh dipakai**.

**Kelebihan:** 5 model billing; progress report draft → approve → bulk send; kelas ganti dengan credit ditolak hanya selepas murid ditanda hadir; multi-branch dengan override permission; payroll guru; Billplz + Payex; e-invoice LHDN.

**Kelemahan:** diskaun adik-beradik manual; progress tiada rubric/level/sijil; attendance manual; murid di-archive → parent hilang akses; 3 tier role sahaja; tiada sebutan direct debit, reconciliation, WhatsApp, PDPA, lokasi server, API.

### 11.3 Vendor 2 — SimTrain (Simit Group)

**Bukti:** website public sahaja. **Folder staf kosong.**

**Rujukan kos:** Professional USD 39/bulan atau USD 500/tahun, 3,000 murid, 20 centre. Ada pelan Free (100 murid, 1 centre) yang Boss boleh test terus.

**Kelebihan:** check-in RFID/QR/barcode; sibling discount; e-invoice LHDN dengan tracking status; credit note/refund; 5 role; parent app native.

**Kelemahan:** **tiada progress tracking langsung**; audit trail dan laporan rentas cawangan hanya Enterprise; komunikasi setakat announcement; tiada sebutan PDPA, export, direct debit.

### 11.4 Vendor 3 — ClassFlow (syarikat belum dikenal pasti)

**Bukti:** 5 screenshot akaun demo "Minda Optima Admin" (6 Okt). Carian web hanya jumpa "ClassFlow" Promethean (produk lain, tak berkaitan). Identiti syarikat dan harga tak diketahui.

**Kapasiti nampak:** 0/500 murid, 1/10 cawangan, kuota murid per cawangan.

**Kelebihan:** onboarding checklist + mod data contoh paling kemas; import Excel dalam onboarding; modul "Pembetulan Kehadiran" tersendiri; carian Ctrl+K; borang maklum balas dalam app; kuota per cawangan.

**Kelemahan:** orientasi **daycare** (hidangan, insiden, log aktiviti), yang Validation Map sudah keluarkan dari skop; had 500/10 (kalau itu had sebenar); billing, bayaran, WhatsApp, PDPA semuanya ❓. "Wajah belum didaftarkan" kemungkinan face recognition (inferens), isu PDPA data biometrik kanak-kanak.

### 11.5 Vendor 4 — Remmu (hubungi `info@remmu.com`)

**Bukti:** 40/50 screenshot demo Zoom ±53 minit (Aznir). Video `Remmu App.mp4` (±300MB) belum dikaji. Tenant demo = akademi sukan "UniqueRays" (taekwondo, renang, dewasa, matematik).

**Amaran kematangan:** page builder tunjuk `http://localhost:3000/...` (dev/staging), banyak rekod "Test", ada label "Coming soon" dan "Beta".

**⚠ Screenshot mengandungi PII pelanggan vendor (nama, e-mel, IC). Jangan edar.**

**Kelebihan (kaya, terutama bahagian duit):**
- Billing: bulanan, per sesi, trial, pakej/kredit, tahunan, deposit boleh refund, event; invois per enrolment atau per murid; nombor boleh konfigurasi; bulk generate sebagai draft; invois syarikat; bayaran separa; credit note.
- Bayaran: **semakan bukti bayaran dengan AI** (skor, auto-lulus ≥85%, had RM, mod monitor), gateway CommercePay (FPX/kad/e-wallet), guest pay link (Beta), halaman bayar QR + upload resit.
- Finance: aging report, "student payment health", collection by location, paparan Families (kutip banyak invois anak sekali).
- Attendance: geofenced check-in, peringatan guru (10 minit sebelum, 10 malam kalau tak lengkap), push ke penjaga, insights (<70% hadir), kelas ganti (tetingkap 1–30 hari), guru ganti.
- Admin: 5 role, permission per modul **lalai OFF**, penugasan lokasi, payroll lengkap, waiver e-sign, chat dalaman dengan pengawasan admin, WhatsApp berkredit (RM0.08–0.12/mesej).

**Kelemahan:**
- **Menu "Progress" wujud tapi TIADA satu pun screenshot.** Ini faktor paling penting untuk PTMO dan status = ❓ sepenuhnya.
- Orientasi sukan/kelab; banyak modul tak relevan (store, event, affiliate, landing builder).
- Gateway CommercePay, bukan Bayarcash/Billplz/ToyyibPay.
- Tiada bukti: direct debit, sibling discount, prorate, e-invoice LHDN, import penyata bank, audit log penuh, API, export penuh, PDPA/lokasi server, jenis parent app, akses parent selepas archive.
- WhatsApp kos tambahan per mesej.

---

## 12. Matriks perbandingan (versi ringkas)

✅ ada · ❌ tiada · ❓ tak dinyatakan (BUKAN tiada) · ➖ separa. Versi penuh (~41 baris) dalam `Vendor-Evaluation-Log-BM.md` §5.

| Feature | AOne | SimTrain | ClassFlow | Remmu | Plan SMS PTMO |
|---|---|---|---|---|---|
| Model billing | ✅ 5 model | ❓ | ❓ | ✅ paling kaya | Pakej BASIC→SUPER DUPER |
| Sibling discount auto | ❌ manual | ✅ | ❓ | ❓ | Rule OPEN |
| Prorate invois pertama | ➖ | ❓ | ❓ | ❓ | ✅ editable |
| Bayaran separa | ✅ | ❓ | ❓ | ✅ | ✅ |
| Gateway | Billplz, Payex | ❓ | ❓ | CommercePay | Bayarcash (FPX) |
| Direct debit | ❓ | ❓ | ❓ | ❓ | ✅ F2.5 |
| Bank reconciliation | ❓ | ❓ | ❓ | ➖ AI semak bukti | ✅ + confidence score |
| e-Invoice LHDN | ✅ | ✅ + tracking | ❓ | ❓ | Semak akauntan |
| Attendance | Manual | RFID/QR | Guru telefon; ➖ wajah? | ✅ geofence | Guru selfie + GPS |
| Kelas ganti | ✅ | ➖ | ❓ | ✅ | ✅ + expiry + no-show |
| Progress report | ✅ draft→approve | ❌ | ➖ Portfolio | ❓ tiada screenshot | Growth Story |
| **Progress berstruktur (buku/level)** | ❌ | ❌ | ❓ | ❓ | **✅ target utama** |
| Sijil | ❌ | ❌ | ❓ | ❓ | ✅ |
| Isolation multi-cawangan | ✅ (mekanisme ❓) | ➖ | ➖ | ➖ | ✅ RLS Postgres |
| Role | 3 tier + override | 5 | ❓ | 5 + permission modul | 5 + khas |
| Payroll guru | ✅ | ➖ | ❓ | ✅ | F5 |
| WhatsApp | ❓ | ➖ butang | ❓ | ✅ berkredit | ✅ utama |
| Parent app | ✅ | ✅ native | ✅ (jenis ❓) | ➖ | PWA |
| Archive → akses parent | ❌ hilang | ❓ | ❓ | ❓ | Parent kekal akses |
| Audit log | ❓ | Enterprise | ❓ | ➖ | ✅ semua role |
| Export penuh | XLSX | ❓ | ❓ | ➖ CSV | ✅ |
| PDPA / lokasi server | ❓ | ❓ | ❓ | ❓ | Wajib MY |
| Had pelan | 2,000 / 20 | 3,000 / 20 (Pro) | 500 / 10 | ❓ | 2,300–2,600 / 15–17 |

**Cara baca betul:** ❓ yang banyak di ClassFlow/Remmu/SimTrain bermaksud *kita belum lihat*, bukan *vendor tak ada*. Jangan keluar dengan kesimpulan "SMS kita menang" berdasarkan lajur ❓.

---

## 13. Curi / buat lebih baik / elak

### Patut dicuri
1. Draft → approve → bulk send untuk Growth Story (AOne).
2. Status bayaran papar masa attendance (AOne, SimTrain).
3. Status tracking e-invoice, dan credit note/refund sebagai flow tersendiri (SimTrain).
4. Override permission per individu di atas 5 role (AOne), tetap bawah RLS.
5. Credit kelas ganti hanya ditolak selepas murid ditanda hadir (AOne).
6. Onboarding checklist + mod data contoh (ClassFlow) untuk Branch Admin dan guru baru.
7. Pembetulan attendance sebagai flow dengan jejak (ClassFlow).
8. Kuota murid per cawangan (ClassFlow) untuk kapasiti slot.
9. Carian Ctrl+K (ClassFlow) untuk admin HQ.
10. **Semakan bukti bayaran dengan skor + had auto-lulus + mod monitor** (Remmu): corak sedia ada untuk confidence score kita; mod monitor cara selamat untuk pilot.
11. Guest pay link, aging report + payment health, deposit refundable, invois per enrolment vs per murid, nombor invois boleh konfigurasi (Remmu).
12. Peringatan guru 10 malam kalau attendance tak lengkap; **permission lalai OFF** (Remmu).
13. Kos WhatsApp telus: kos BSP mesti masuk model kos kita.

### Peluang menang (hanya jika terbukti, ingat ❓ ≠ ❌)
1. Progress berstruktur ikut buku/level, dengan evidence dan "belum cukup data".
2. Isolation di level DB (RLS) yang boleh dibuktikan.
3. Direct debit + reconciliation dengan confidence score.
4. Kelas ganti dengan eligibility, expiry, no-show (bukan open credit).
5. Parent kekal akses report selepas archive/graduate.
6. Audit log lalai untuk semua role.

### Elak
- Modul daycare (hidangan, insiden), jangan masuk sebab vendor ada.
- Face recognition kanak-kanak, kecuali PDPA + consent dijawab dulu.
- Skop kelab/komersial (store, event, affiliate, landing builder).
- Anggap demo dev/staging sebagai produk matang.
- Chat bebas parent↔guru sebagai default.
- Progress sebagai "surat bebas" tanpa struktur.
- Archive yang putuskan akses parent.

---

## 14. Hipotesis grill (H1–H6) — bahagian paling penting

Ni bukan kesimpulan. Ni tempat plan Boss paling mungkin salah. Tugas grill ialah cuba pecahkan satu-satu.

**H1 — Build vs buy tak boleh bergantung pada kos.** AOne ≈ RM16k setahun. Kos bina sendiri (masa, developer, penyelenggaraan 7+ bulan) hampir pasti lebih tinggi. Kalau Boss nak bina, alasannya mesti sesuatu yang vendor *tak boleh beri*: progress berstruktur, isolation DB, kawalan data, rumah untuk Balik Cerita. Kalau alasan tu tak terbukti, "bina sendiri" tinggal keinginan.

**H2 — Billing-first mungkin terbalik.** Plan bina billing dulu (commodity, semua vendor ada), manakala pembeza sebenar (progress berstruktur ikut buku/level, Growth Story, Balik Cerita) sampai paling lewat (F3/F4). Maknanya, bila-bila masa dalam 7–10 bulan pertama, sistem Boss lebih kurang sama dengan vendor yang boleh dibeli semalam. Pertimbangkan **hibrid**: beli/guna vendor untuk billing, bina progress sendiri.

**H3 — Pembeza palsu.** Banyak "kita menang" bergantung pada senyap vendor. Sebelum kata "tiada vendor ada X", sahkan dalam demo.

**H4 — Kapasiti solo builder dan kebergantungan luar.** Anggaran 29 minggu andaikan satu orang tanpa halangan. Hakikatnya ada lead-time luaran yang bukan dalam kawalan: onboarding Bayarcash, template WhatsApp BSP/Meta, DPA/PDPA, LHDN e-invoice, kelulusan mandate bank. Siapa developer, berapa jam seminggu?

**H5 — Migrasi dan cutover.** Data 2,300–2,600 murid dalam Sheets yang kotor. Dua kitaran selari tanpa beza sen-per-sen ialah syarat yang sangat keras. Cadangan: guna trial vendor sebagai **latihan pembersihan data** (import 30–100 baris sebenar, tengok apa pecah).

**H6 — Kill-criteria sebelum demo.** Tetapkan awal apa keputusan demo yang akan **menukar** "bina sendiri" kepada "beli" atau "hibrid". Kalau tak, demo cuma mengesahkan apa yang Boss dah percaya (confirmation bias).

---

## 15. Kill-criteria (cadangan draf, belum diluluskan Boss)

Vendor dikira **gugur** jika satu daripada ini gagal. Sebaliknya, jika satu vendor lulus semua *dan* Boss tak boleh sebut satu benda yang vendor tak boleh beri, soalan "kenapa bina sendiri?" mesti dijawab semula.

1. Tak boleh konfigurasi tahap buku 1A→1B dan naik level tanpa custom dev.
2. Tak boleh buat sibling discount + prorate pada invois pertama.
3. Gagal ujian isolation (cawangan X → Y, parent A → anak parent B).
4. Export penuh data tak lengkap (bukan XLSX sahaja).
5. Server bukan di Malaysia atau tiada DPA (PDPA).
6. Tiada reference client dengan 10+ cawangan.

**6 ujian wajib semasa demo/trial** (sama dengan log §7):
1. Konfigurasi buku 1A → 1B tanpa custom dev.
2. Sibling + prorate pada invois pertama.
3. User cawangan X cuba akses data cawangan Y melalui URL/ID.
4. Parent cuba akses anak parent lain.
5. Murid di-archive: apa yang parent masih nampak?
6. Export penuh: lengkap atau tidak?

Plus: import 30 baris Excel sebenar (murid, penjaga, pakej), dan ukur **minit** yang diambil Branch Admin sebenar serta seorang parent sebenar untuk setiap tugasan.

---

## 16. Cara guna prompt Claude Code

Fail: `Prompt-Grill-Vendor-Teardown-BM.md` (v1.1). Fungsi: tampal ke Claude Code, mod **grill sahaja, tiada coding**.

**Cara kerja prompt:**
- Satu soalan sekali, dengan cadangan jawapan Claude, lepas tu ditolak balik kalau lemah.
- Baca fail dulu (Fasa 0): vendor log v0.3, Validation Map, `SMS-Grill-Questions.md`, `CLAUDE.md`, Budget doc, SMS Brief, Soalan-untuk-Developer. **Jangan sentuh** Project-OS, fail Doraemon, dan PNG.
- Label bukti: ✅ ❌ ❓ ➖. Tak anggap checkpoint §7 lulus.
- Rekod keputusan dalam `Grill-Vendor-Decision-Log-BM.md` dengan status LOCKED / OPEN / Perlu data.
- Tunjuk konflik dokumen (lihat §17) dan minta Boss putuskan.
- Susunan grill 8 langkah, bermula dengan soalan "kerosakan operasi terbesar dalam 90 hari".
- Penutup: bila Boss rasa yakin, tanya *"Kalau kau kena bet duit dan 6 bulan masa sekarang, apa satu perkara yang kalau salah akan jatuhkan keseluruhan keputusan ni?"*
- Output sesi: Decision Log, kemas kini vendor log, satu nota kesediaan (readiness note) satu muka surat, **bukan green light**.

**Nota penting:** sesi Claude Code perlu pegang folder projek supaya boleh baca fail. Folder Drive vendor perlu dibuka melalui Chrome kerana carian Drive tak index isi subfolder.

---

## 17. Peta dokumen dan konflik antara dokumen

**Folder projek (Mac):** `.../School Management System Project 2026/School Management System PTMO 2026/`

| Fail | Apa | Status |
|---|---|---|
| `SMS-Whole-System-Validation-Map.md` | Peta menyeluruh skop/flow/fasa (17 Sep) | Sumber kebenaran skop |
| `Vendor-Evaluation-Log-BM.md` | Log vendor v0.3 | Hidup, update bila ada data baru |
| `Prompt-Grill-Vendor-Teardown-BM.md` | Prompt Claude Code v1.1 | Siap |
| `SMS-Grill-Questions.md` | Soalan produk A–K (18 Sep) | Jangan duplicate |
| `SMS-Brief-untuk-Developer.md` | Brief untuk developer | Semak kandungan (mungkin lapuk) |
| `Soalan-untuk-Developer-Sebelum-Bayar.md` | Soalan sebelum bayar developer | Rujukan |
| `Budget-Timeline-Reality-Check.md` | Anggaran bajet/masa | **Lapuk** (SaaS RM100k–400k vs quote AOne) |
| `CLAUDE.md` | Peraturan projek | **Perlu dibetulkan** |
| `SMS-Project-OS-Onboarding.md`, `Project-OS-*.png`, `Doraemon-*` | Project OS / Doraemon | Jangan fikir lagi buat masa ni |
| `Alam-Aksara-Pulau-Bunyi-Pilot-v0.1.md` | Pilot (pengajaran) | Berasingan |

**Konflik yang mesti Boss selesaikan:**
1. `CLAUDE.md` masih tulis "15 childcare centres" (lapuk; Validation Map betul: program bacaan). Dan peraturannya "jangan tulis kod SMS" bercanggah dengan hasrat mula coding lepas grill. Boss perlu setuju: update atau tidak.
2. `Budget-Timeline-Reality-Check.md` lapuk berbanding quote AOne.
3. Staf kumpul alat **childcare** (ClassFlow) dan **kelab sukan** (Remmu) untuk program bacaan; kategori carian perlu dibetulkan.
4. Gateway: Remmu guna CommercePay, plan guna Bayarcash/Billplz/ToyyibPay.

---

## 18. Apa kena buat seterusnya

**Untuk staf (Boss relay):**
- [ ] Tangkap skrin modul **Progress** Remmu, paparan parent/murid, dan halaman harga/Plan.
- [ ] Isi folder SimTrain; minta quote MYR + demo khusus Minda Optima (atau Boss test pelan Free).
- [ ] Dapatkan identiti syarikat + harga + had pelan untuk ClassFlow dan Remmu; sahkan Remmu demo live (bukan localhost).
- [ ] Transkrip atau nota dua video (AOne dan Remmu).
- [ ] Cari **vendor dalam kategori yang betul** (learning-centre / tuisyen / enrichment) selain AOne dan SimTrain.

**Untuk Boss:**
- [ ] Putuskan kill-criteria (§15) **sebelum** demo.
- [ ] Setuju/tolak update `CLAUDE.md`.
- [ ] Jawab soalan "kerosakan operasi terbesar 90 hari".
- [ ] Ingat tarikh akhir quote AOne: **31 Okt 2026** (bukan tekanan beli; sekadar tahu).
- [ ] Jalankan sesi grill dalam Claude Code dengan prompt v1.1.

**Pending dari log §9:** hantar soalan §3/§4b/§4c/§7 kepada vendor (draft e-mel boleh disediakan bila Boss minta), lihat 10 screenshot Remmu yang belum dibaca, ujian hands-on §7 untuk semua vendor, compare akhir, Boss beri go-ahead.

---

## 19. Glosari

- **SMS PTMO:** School Management System, Pantas Baca / Minda Optima, 2026.
- **System of Record (SoR):** satu tempat rasmi untuk setiap jenis data.
- **RLS:** Row-Level Security PostgreSQL; DB sendiri halang baris yang tak dibenarkan.
- **BranchContext:** lapisan Laravel yang tahu cawangan mana pengguna dibenarkan.
- **FPX:** pembayaran bank online Malaysia. **Direct debit:** potongan auto berulang melalui mandate.
- **Bayarcash / Billplz / ToyyibPay:** penyedia gateway Malaysia.
- **BSP:** WhatsApp Business Solution Provider.
- **MCP:** protokol untuk bagi AI akses data secara terkawal; di sini read-only dan role-scoped.
- **PWA:** web app yang boleh dipasang di telefon. **Capacitor:** pembungkus untuk hantar ke app store.
- **Idempotent:** buat dua kali, kesan sekali sahaja (elak invois/bayaran duplicate).
- **Prorate:** kira yuran separa bulan untuk murid yang mula tengah bulan.
- **Kill-criteria:** syarat yang, kalau gagal, terus gugurkan satu pilihan.
- **Pantas Baca (PB), TVR:** programme dalam plan.
- **BASIC / FAST / SUPER / SUPER DUPER:** empat pakej yuran.
- **Balik Cerita / Growth Story:** laporan progress naratif untuk ibu bapa (fasa kemudian).
- **✅ ❌ ❓ ➖:** ada / tiada / tak diketahui (bukan tiada) / separa.

---

## Changelog

- **v1.0 — 7 Okt 2026:** Versi pertama. Konsolidasi Validation Map (17 Sep), Vendor Log v0.3, Prompt Grill v1.1 dan Grill Questions menjadi satu penerangan A–Z.
