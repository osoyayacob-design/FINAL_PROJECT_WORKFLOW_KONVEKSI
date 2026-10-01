# FINAL_PROJECT_WORKFLOW_KONVEKSI

# Otomasi Pengecekan Stok Bahan Konveksi dari Purchase Order

Workflow n8n yang membaca Purchase Order (PDF) dari Gmail, mencatatnya ke Google Sheets, menghitung kebutuhan bahan, lalu mengirim notifikasi ke tim produksi atau tim pembelian sesuai kecukupan stok.

**Dibuat oleh:** [Nama Kamu] | **Bootcamp:** [Nama Bootcamp]

---

## Latar Belakang Masalah

Konveksi seragam menerima PO dalam bentuk PDF. Pencatatan manual membuat:
- stok bahan tidak tercatat rapi dan tidak ter-update,
- kekurangan bahan baru diketahui saat proses penjahitan sudah berjalan,
- PO yang formatnya berbeda-beda sulit diproses dengan aturan baku.

## Mengapa Perlu Automation dan AI

- **Automation:** pencatatan terintegrasi ke spreadsheet, selalu ter-update, dan produksi bisa dikendalikan sebelum penjahitan dimulai.
- **AI (Gemini):** layout PDF PO bisa berbeda-beda, sehingga ekstraksi dengan aturan tetap (regex/template) rapuh. Gemini membaca dokumen secara kontekstual dan mengubahnya menjadi JSON terstruktur.

## Alur Workflow

1. **Gmail Trigger**: menerima email masuk ke alamat khusus PO.
2. **If**: lanjut hanya jika email memiliki lampiran PDF. Tanpa PDF, workflow berhenti.
3. **Analyze document (Gemini)**: membaca PDF dan mengekstrak `no_po`, `model`, `jumlah` dalam bentuk JSON.
4. **Edit Fields**: memecah JSON menjadi field terpisah agar bisa dipetakan ke kolom sheet.
5. **Append row in sheet**: mencatat PO ke Google Sheets dengan status "MASUK".
6. **Switch**: memisahkan jalur berdasarkan model (Model 1 atau Model 2).
7. **Read stok + If**: membaca stok bahan dari sheet dan membandingkannya dengan kebutuhan (jumlah PO x kebutuhan per pcs).
8. **Gmail (notifikasi)**:
   - Bahan lengkap: email ke tim produksi.
   - Bahan kurang: email ke tim pembelian.

## Tools yang Digunakan

| Tool | Fungsi |
|---|---|
| n8n | Platform automation |
| Gemini API | Membaca PDF PO dan mengekstrak data |
| Gmail | Trigger penerimaan PO dan pengiriman notifikasi |
| Google Sheets | Pencatatan PO dan data stok bahan |

## Conditional Logic

- **If (awal):** memeriksa ada tidaknya lampiran PDF.
- **Switch:** percabangan berdasarkan model.
- **If (per model):** menentukan bahan lengkap atau kurang.

## Prompting (Gemini)

Prompt memuat:
- **Role:** asisten gudang konveksi seragam.
- **Tugas:** ambil hanya `no_po`, `model`, `jumlah`; abaikan informasi lain.
- **Format output:** JSON murni tanpa teks pembuka dan tanpa markdown, dengan skema baku.
- **Aturan:** `model` dinormalisasi menjadi "Model 1" atau "Model 2"; `jumlah` berupa angka; nilai tidak terbaca diisi `null` (tidak menebak); dokumen bukan PO diberi `keterangan`; PO multi-model mengambil jumlah terbesar.
- **Contoh output** sebagai acuan struktur.

## Struktur Google Sheets

**Sheet stok:** kolom `bahan`, `stok`, `Model1`, `Model2` (kebutuhan bahan per pcs).

| bahan | stok | Model1 | Model2 |
|---|---|---|---|
| kancing | 1000 | 10 | 14 |
| kain_keras | 500 | 30 | 37 |
| benang | 100 | 1 | 1.5 |

**Sheet PO:** `No_PO`, `Model`, `Jumlah`, `Status`, `butuh_kancing`, `butuh_kain`, `butuh_benang`, `Cek_Stok`.
Kolom `butuh_*` dan `Cek_Stok` dihitung dengan formula Google Sheets (jumlah PO x kebutuhan per pcs sesuai model, lalu dibandingkan dengan stok).

## Hasil Pengujian

| Skenario | Hasil |
|---|---|
| PO Model 2, 12 pcs | Cek_Stok "Cukup", email "Bahan Lengkap" ke produksi |
| PO Model 1, 5 pcs | "Cukup", email ke produksi |
| PO Model 1, 50 pcs | "Kurang" (kain_keras), email ke pembelian |
| PO Model 2, 80 pcs | "Kurang", email ke pembelian |
| Email tanpa lampiran PDF | Berhenti di cabang false If pertama |

## Screenshot

- `screenshots/workflow.png`: canvas workflow n8n
- `screenshots/sheet-po.png`: sheet PO beserta kolom kebutuhan dan Cek_Stok
- `screenshots/email-lengkap.png`: notifikasi bahan lengkap
- `screenshots/email-kurang.png`: notifikasi bahan kurang
- `screenshots/output-gemini.png`: output JSON dari Gemini

## Cara Menjalankan

1. Import workflow ke n8n.
2. Hubungkan credential: Gmail (OAuth), Google Sheets (OAuth), Gemini API key.
3. Sesuaikan ID spreadsheet dan nama sheet pada node Google Sheets.
4. Pastikan header sheet sesuai struktur di atas.
5. Aktifkan (publish) workflow, lalu kirim email berisi PDF PO ke alamat PO.

## Keterbatasan

- Stok di sheet belum otomatis dikurangi setelah PO diproses; pengecekan dilakukan per PO.
- PO yang memuat lebih dari satu model hanya memproses model dengan jumlah terbesar.
- Dokumen non-PO atau data yang tidak terbaca (`null`) belum punya jalur penanganan khusus.
- Email tanpa PDF diabaikan tanpa balasan ke pengirim.

## Pengembangan Lanjutan

- Pengurangan stok otomatis dan pencatatan riwayat.
- Dukungan PO multi-model.
- Balasan otomatis untuk PO tidak valid.
