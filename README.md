# Mini Project PCD — Deteksi Keberadaan Tanda Tangan Dekan

## 📖 Deskripsi Proyek

Proyek ini merupakan **Mini Project Pengolahan Citra Digital (PCD)** yang bertujuan untuk mendeteksi **keberadaan tanda tangan dekan** pada dokumen resmi (ijazah, surat keputusan, sertifikat) menggunakan teknik pengolahan citra digital dengan **Python** dan **OpenCV**.

Sistem menganalisis **area tanda tangan dekan** dengan tahapan:

1. Menghubungkan Google Drive
2. Pemeriksaan citra (load, cek ukuran & kontras)
3. Cropping area tanda tangan dekan
4. Konversi ke grayscale
5. Thresholding (Global & Otsu)
6. Morphological Operation (Opening & Closing)
7. Pengukuran piksel foreground
8. Klasifikasi **SIGNATURE PRESENT** / **SIGNATURE ABSENT**

---

## 🗂️ Struktur Dataset

Dataset disimpan di Google Drive dengan struktur:

```
MyDrive/
└── PCD6/
    ├── Present/          → gambar DENGAN tanda tangan dekan
    │   ├── Copy of Copy of 01_HighQuality_Enhanced.jpg
    │   ├── Copy of Copy of 02_LowContrast.jpg
    │   ├── Copy of Copy of 03_Blurred.jpg
    │   ├── Copy of Copy of 04_HighNoise.jpg
    │   ├── Copy of Copy of 05_LowResolution_Upsampled.jpg
    │   ├── Copy of Copy of 06_Faded_Underexposed.jpg
    │   ├── Copy of Copy of 07_ColorShift_WarmTint.jpg
    │   ├── Copy of Copy of 08_JPEGCompression_Artifacts.jpg
    │   └── Copy of Copy of 09_CombinedDegradation.jpg
    │
    ├── Absent/           → gambar TANPA tanda tangan dekan
    │   ├── Copy of gambar 1.jpg
    │   ├── Copy of gambar 2.jpg
    │   ├── Copy of gambar 3.jpg
    │   ├── Copy of gambar 4.jpg
    │   ├── Copy of gambar 5.jpg
    │   ├── Copy of gambar 6.jpg
    │   ├── Copy of gambar 7.jpg
    │   ├── Copy of gambar 8.jpg
    │   └── Copy of gambar 9.jpg
    │
    └── output/           → folder output (otomatis dibuat oleh program)
```

### Ringkasan Dataset

| Kelas   | Jumlah | Resolusi        | Keterangan                     |
|---------|--------|-----------------|--------------------------------|
| Present | 9      | 2481 × 3506 px  | Citra dengan TTD dekan         |
| Absent  | 9      | 864 × 1221 px   | Citra tanpa TTD dekan          |
| Total   | 18     | —               | —                              |

---

## ⚙️ Prasyarat

### 1. Akun
- **Google Account** (untuk akses Google Colab & Google Drive)

### 2. Library Python

Semua library berikut sudah tersedia default di Google Colab. Jika belum:

```python
!pip install opencv-python-headless matplotlib numpy pandas
```

| Library | Versi Minimum | Fungsi                        |
|---------|---------------|-------------------------------|
| OpenCV  | 4.x           | Pengolahan citra              |
| NumPy   | 1.20+         | Operasi array                 |
| Pandas  | 1.3+          | Manipulasi tabel              |
| Matplotlib | 3.4+       | Visualisasi                   |

### 3. Dataset
- Folder `PCD6/` sudah diunggah ke Google Drive (di My Drive, **bukan** Shared Drive)
- Berisi subfolder `Present/` dan `Absent/`

---

## 🚀 How to Run Program

### **Langkah 1 — Buka Google Colab**

1. Buka [https://colab.research.google.com](https://colab.research.google.com)
2. Login dengan akun Google Anda
3. Pilih **File → Upload notebook** lalu unggah file `.ipynb` proyek ini
   - Atau: **File → New notebook** lalu salin kode dari repo ini

### **Langkah 2 — Mount Google Drive & Akses Dataset**

Jalankan **Cell Tahap 1**:

```python
# TAHAP 1: Hubungkan Google Drive dan akses folder PCD6
from google.colab import drive
drive.mount('/content/drive')

import os, cv2, numpy as np, pandas as pd
import matplotlib.pyplot as plt

BASE_DIR    = "/content/drive/MyDrive/PCD6"
PRESENT_DIR = os.path.join(BASE_DIR, "Present")
ABSENT_DIR  = os.path.join(BASE_DIR, "Absent")
OUTPUT_DIR  = os.path.join(BASE_DIR, "output")
os.makedirs(OUTPUT_DIR, exist_ok=True)

present_files = sorted(os.listdir(PRESENT_DIR))
absent_files  = sorted(os.listdir(ABSENT_DIR))

print("Present :", present_files)
print("Absent  :", absent_files)
print("Total   :", len(present_files) + len(absent_files), "gambar")
```

**Prompt otorisasi akan muncul:**
1. Klik link yang diberikan
2. Pilih akun Google Anda
3. Klik **Allow / Izinkan**
4. Copy kode otorisasi → paste ke kolom di Colab → Enter

**Output yang diharapkan:**

```
Mounted at /content/drive
Present : ['Copy of Copy of 01_HighQuality_Enhanced.jpg', ...]
Absent  : ['Copy of gambar 1.jpg', ...]
Total   : 18 gambar
```

### **Langkah 3 — Pemeriksaan Citra (Tahap 2)**

Jalankan Cell Tahap 2 untuk memuat, memeriksa ukuran & kontras, dan preview:

```python
# TAHAP 2: Baca & periksa gambar
def load_images(folder, label):
    data = []
    for fname in sorted(os.listdir(folder)):
        if not fname.lower().endswith((".jpg", ".jpeg", ".png")):
            continue
        path = os.path.join(folder, fname)
        img  = cv2.imread(path)
        if img is None:
            print("[SKIP]", path); continue
        h, w = img.shape[:2]
        std  = float(np.std(cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)))
        data.append({"path": path, "fname": fname, "label": label,
                     "img": img, "h": h, "w": w, "std": round(std, 2)})
    return data

present_imgs = load_images(PRESENT_DIR, "present")
absent_imgs  = load_images(ABSENT_DIR,  "absent")
all_imgs     = present_imgs + absent_imgs

df_check = pd.DataFrame([{
    "File": d["fname"], "Label": d["label"],
    "Width": d["w"], "Height": d["h"],
    "Std (kontras)": d["std"]
} for d in all_imgs])
display(df_check)
```

**Output:** tabel 18 baris + preview grid gambar.

### **Langkah 4 — Cropping Area TTD Dekan (Tahap 3)**

```python
# TAHAP 3: Crop area tanda tangan dekan (kanan bawah)
ROI_RATIO = (0.55, 0.70, 1.00, 1.00)   # (x1, y1, x2, y2)

def crop_dean_signature(image, roi_ratio=ROI_RATIO):
    h, w = image.shape[:2]
    x1, y1 = int(w*roi_ratio[0]), int(h*roi_ratio[1])
    x2, y2 = int(w*roi_ratio[2]), int(h*roi_ratio[3])
    return image[y1:y2, x1:x2]

for d in all_imgs:
    d["roi"] = crop_dean_signature(d["img"])
```

### **Langkah 5 — Grayscale (Tahap 4)**

```python
# TAHAP 4: Konversi ke grayscale
for d in all_imgs:
    d["gray"] = cv2.cvtColor(d["roi"], cv2.COLOR_BGR2GRAY)
```

### **Langkah 6 — Thresholding (Tahap 5)**

```python
# TAHAP 5: Thresholding — Global (T=127) & Otsu
def global_threshold(gray, T=127):
    _, b = cv2.threshold(gray, T, 255, cv2.THRESH_BINARY_INV)
    return b

def otsu_threshold(gray):
    _, b = cv2.threshold(gray, 0, 255,
                         cv2.THRESH_BINARY_INV + cv2.THRESH_OTSU)
    return b

for d in all_imgs:
    d["th_global"] = global_threshold(d["gray"], 127)
    d["th_otsu"]   = otsu_threshold(d["gray"])
```

### **Langkah 7 — Morphological Operation (Tahap 6)**

```python
# TAHAP 6: Opening (hapus noise) + Closing (rapatkan goresan)
def morphological_ops(binary, k=3):
    kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (k, k))
    opened = cv2.morphologyEx(binary, cv2.MORPH_OPEN,  kernel, iterations=1)
    closed = cv2.morphologyEx(opened, cv2.MORPH_CLOSE, kernel, iterations=2)
    return opened, closed

for d in all_imgs:
    d["opened"], d["closed"] = morphological_ops(d["th_otsu"], k=3)
```

### **Langkah 8 — Pengukuran Piksel Foreground (Tahap 7)**

```python
# TAHAP 7: Hitung piksel foreground
def count_foreground(binary):
    fg = int(np.sum(binary == 255))
    return fg, fg / binary.size

for d in all_imgs:
    d["fg_final"], d["r_final"] = count_foreground(d["closed"])

df_measure = pd.DataFrame([{
    "File": d["fname"], "Label": d["label"],
    "FG Final": d["fg_final"],
    "Ratio Final": round(d["r_final"], 4),
} for d in all_imgs])

display(df_measure)
df_measure.to_csv(os.path.join(OUTPUT_DIR,
                  "tahap7_pengukuran_foreground.csv"), index=False)
```

### **Langkah 9 — Klasifikasi (Opsional, jika dibutuhkan)**

```python
# TAHAP 8: Klasifikasi SIGNATURE PRESENT / ABSENT
MIN_PIXEL = 300
MIN_RATIO = 0.005
MAX_RATIO = 0.35

def decide_signature(binary, min_pixel=MIN_PIXEL,
                     min_ratio=MIN_RATIO, max_ratio=MAX_RATIO):
    fg, ratio = count_foreground(binary)
    present = (fg >= min_pixel) and (min_ratio <= ratio <= max_ratio)
    return present, fg, ratio

for d in all_imgs:
    d["present"], d["fg_dec"], d["r_dec"] = decide_signature(d["closed"])

df_dec = pd.DataFrame([{
    "File": d["fname"], "Label Asli": d["label"],
    "Prediksi": "PRESENT" if d["present"] else "ABSENT",
    "Benar?": "✅" if ((d["present"] and d["label"]=="present")
                       or (not d["present"] and d["label"]=="absent"))
              else "❌"
} for d in all_imgs])

display(df_dec)

correct = (df_dec["Benar?"] == "✅").sum()
print(f"Akurasi: {correct}/{len(df_dec)} "
      f"({correct/len(df_dec)*100:.1f}%)")
```

### **Langkah 10 — Simpan Hasil ke Drive**

Semua hasil otomatis tersimpan di:

```
MyDrive/PCD6/output/
```

File yang dihasilkan:

- `tahap7_pengukuran_foreground.csv` — tabel pengukuran foreground
- `tahap8_klasifikasi.csv` — tabel klasifikasi (jika Tahap 9 dijalankan)
- `<label>_<nama_file>_analysis.png` — panel visualisasi per gambar

---

## 🧪 Analisis

### 1. Mengapa Thresholding Diperlukan Sebelum Analisis Keberadaan Tanda Tangan?

Thresholding memisahkan **foreground (goresan tanda tangan)** dari **background (kertas dokumen)**. Tanpa thresholding, citra hanya berupa nilai intensitas abu-abu (0–255) yang **sulit dikuantifikasi** untuk keperluan klasifikasi.

Setelah thresholding, citra menjadi **biner** sehingga kita dapat menghitung:

- Jumlah piksel foreground (tinta)
- Rasio foreground terhadap luas area
- Sebaran piksel (bounding box)

Angka-angka inilah yang menjadi **dasar objektif** untuk memutuskan PRESENT/ABSENT.

### 2. Apa Masalah Jika Threshold Terlalu Tinggi atau Terlalu Rendah?

| Kondisi | Dampak | Konsekuensi |
|---------|--------|-------------|
| **Terlalu tinggi** (mis. T = 220) | Kertas terang, bayangan, noise scanner ikut jadi foreground → **over-segmentation** | Gambar TANPA tanda tangan bisa salah diprediksi **PRESENT** (false positive) |
| **Terlalu rendah** (mis. T = 40) | Hanya piksel sangat gelap yang lolos → goresan tipis/pudar hilang → **under-segmentation** | Gambar BERTANDA TANGAN bisa salah diprediksi **ABSENT** (false negative) |

Karena itu, metode **Otsu** (threshold otomatis dari histogram) lebih disarankan karena adaptif terhadap kontras gambar.

---

## 🔬 Perbandingan Metode Thresholding

| Metode       | Kelebihan                        | Kekurangan                          |
|--------------|----------------------------------|-------------------------------------|
| **Global**   | Cepat, sederhana                 | Sensitif terhadap pencahayaan       |
| **Otsu**     | Otomatis, optimal untuk bimodal  | Kurang baik jika histogram flat     |
| **Adaptive** | Tahan pencahayaan tidak merata   | Rentan noise, butuh tuning parameter|

---

## 📊 Output Program

1. **Tabel CSV** — berisi file, label, jumlah foreground, dan rasio
2. **Panel visualisasi PNG** — perbandingan ROI, grayscale, thresholding, morphology, dan keputusan
3. **Akurasi klasifikasi** — ditampilkan di console
4. **Confusion matrix** — TP, TN, FP, FN

---

## 🛠️ Troubleshooting

| Masalah | Solusi |
|---------|--------|
| `drive.mount` gagal | Pastikan login dengan akun Google yang punya akses ke folder PCD6 |
| `FileNotFoundError: /content/drive/MyDrive/PCD6` | Pastikan folder `PCD6` ada di **My Drive** (bukan Shared Drive) |
| Gambar tidak terbaca | Cek ekstensi `.jpg/.jpeg/.png` dan pastikan file tidak corrupt |
| Akurasi rendah | Kalibrasi `ROI_RATIO`, `MIN_PIXEL`, dan `MIN_RATIO` berdasarkan data |
| Hasil crop tidak pas | Jalankan kode kalibrasi `ROI_RATIO` (lihat Lampiran) |

---

## 📎 Lampiran — Kalibrasi `ROI_RATIO`

Jika hasil crop tidak tepat, jalankan kode ini untuk mencoba beberapa `ROI_RATIO` sekaligus:

```python
sample_path = all_imgs[0]["path"]
img = cv2.imread(sample_path)

candidates = [
    (0.45, 0.55, 1.00, 1.00),
    (0.55, 0.70, 1.00, 1.00),
    (0.60, 0.75, 1.00, 1.00),
    (0.00, 0.65, 1.00, 1.00),
]

fig, axes = plt.subplots(1, len(candidates), figsize=(5*len(candidates), 8))
for ax, r in zip(axes, candidates):
    h, w = img.shape[:2]
    x1, y1 = int(w*r[0]), int(h*r[1])
    x2, y2 = int(w*r[2]), int(h*r[3])
    boxed = img.copy()
    cv2.rectangle(boxed, (x1,y1), (x2,y2), (0,0,255), 12)
    ax.imshow(cv2.cvtColor(boxed, cv2.COLOR_BGR2RGB))
    ax.set_title(f"ROI_RATIO = {r}", fontsize=10)
    ax.axis("off")
plt.tight_layout(); plt.show()
```

Pilih `ROI_RATIO` yang kotaknya paling pas membungkus tanda tangan dekan.

---

## 📁 Struktur Repository

```
signature-detection/
│
├── README.md                    ← dokumentasi ini
├── requirements.txt             ← daftar library
├── notebook.ipynb               ← notebook Colab
├── main.py                      ← script Python (opsional)
├── images/                      ← contoh gambar (opsional)
│   ├── signed/
│   └── unsigned/
└── output/                      ← hasil output (contoh)
    └── tahap7_pengukuran_foreground.csv
```

---

## 👤 Author

- **Nama**  : `Fadel Achmadinejab`
- **NIM**   : `F1G124048`
- **Kelas** : `A`
- **Mata Kuliah** : Pengolahan Citra Digital

---

## 📜 Lisensi

Proyek ini dibuat untuk keperluan **akademik** (Mini Project PCD). Bebas digunakan dan dimodifikasi untuk pembelajaran.
