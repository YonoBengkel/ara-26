# ARA ITS 2026 — segmentasi kerusakan jalan

Kode dan catatan kerjaku untuk babak data competition ARA ITS 2026: memberi mask piksel
pada bagian jalan yang rusak (lubang, retak, amblas) dari foto permukaan jalan. Metriknya
Dice Score.

Repo ini menyimpan notebook eksperimen dan catatan keputusannya apa adanya, bukan pipeline
yang sudah dirapikan. Yang menarik di sini justru jejak kenapa satu pendekatan dipilih dan
yang lain dibuang.

## Peranku

Lomba ini dikerjakan tim. Bagianku: eksplorasi data dan audit anomali gambar secara manual,
perumusan strategi preprocessing/augmentasi/arsitektur yang ada di `REVISI.txt`, serta
notebook eksperimen di repo ini. Training dan submission dijalankan bersama tim.

## Yang membentuk keputusannya

Audit manual terhadap data train dan test memunculkan beberapa hal yang tidak kelihatan dari
statistik saja, dan tiap temuan langsung mengubah pipeline:

- Resolusi gambar tidak seragam, sehingga resize kotak biasa merusak proporsi lubang.
  Solusinya `LongestMaxSize` + `PadIfNeeded` (letterbox), bukan resize paksa.
- Sebagian jalan adalah paving block, bukan aspal. Celah antar paving terbaca sebagai
  kerusakan oleh model. Ini yang melahirkan dua perlakuan: `CoarseDropout` saat augmentasi
  supaya model belajar tekstur retakan alih-alih menghafal pola kotak paving, dan
  penghapusan blob kecil (< 50 piksel) saat post-processing.
- Banyak jalan rusak yang tergenang air atau kontrasnya rendah terhadap aspal sehat, jadi
  augmentasi warna (CLAHE, HueSaturationValue) dipakai agar model tidak bersandar pada warna.
- Mask pada data train terbukti akurat saat di-overlay ke gambar aslinya, jadi label dipercaya
  apa adanya dan tidak ada pembersihan label.

## Pipeline akhir

| Tahap | Pilihan |
|---|---|
| Preprocessing | Letterbox ke 384x384 (`LongestMaxSize` + `PadIfNeeded`) |
| Augmentasi | Flip/Rotate90, GridDistortion, ShiftScaleRotate, CLAHE, HueSaturationValue, CoarseDropout |
| Loss | Combo 0.5 BCE + 0.5 Dice |
| Validasi | 5-Fold CV |
| Inference | TTA 8 arah (D4), crop padding, resize balik ke ukuran asli |
| Post-processing | Hapus blob < 50 piksel |

## Hasil tiap model

Skor Kaggle di papan publik, dan rata-rata Dice tiap fold saat validasi:

| Model | Kaggle | Fold 0 | Fold 1 | Fold 2 | Fold 3 | Fold 4 |
|---|---|---|---|---|---|---|
| U-Net + ResNet34 | 0.68858 | 0.7676 | 0.8041 | 0.7374 | 0.7660 | 0.7741 |
| U-Net++ + EfficientNet-B4 | 0.70773 | 0.7898 | 0.7952 | 0.7487 | 0.7719 | 0.7934 |
| DeepLabV3+ + ResNeSt50d | 0.68609 | 0.7841 | 0.7771 | 0.7479 | 0.7639 | 0.7757 |

Ketiganya diensemble dengan merata-ratakan probabilitas sebelum di-threshold. Jarak sekitar
0.07 antara skor fold dan skor Kaggle konsisten di ketiga model, yang menunjukkan selisih itu
berasal dari distribusi data test, bukan dari overfitting satu arsitektur.

## Isi repo

| Berkas | Isi |
|---|---|
| `REVISI.txt` | Catatan audit data, daftar anomali per gambar, dan alasan tiap keputusan pipeline |
| `njajal1`–`njajal4.ipynb` | Eksperimen arsitektur dan backbone |
| `njajalT1`–`njajalT3.ipynb` | Eksperimen TTA dan post-processing |
| `nyoba_real.ipynb` | Jalur inference ke submission |
| `submission.csv`, `submission_v2_fix.csv` | Submission yang dikirim |

Folder `dataset/` sengaja tidak ikut: citranya milik penyelenggara lomba.
