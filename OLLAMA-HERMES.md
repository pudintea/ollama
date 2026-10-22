# Tutorial lengkap: Ollama + Hermes Agent untuk DevOps server 4 vCPU / RAM 8 GB

Kita akan menyiapkan AI lokal untuk membantu memeriksa log, menganalisis error, dan melakukan troubleshooting server Linux. Fokusnya adalah koneksi yang stabil, penggunaan RAM yang terkendali, serta keamanan server produksi.

Ada satu keputusan teknis yang harus kita bereskan lebih dahulu: jangan memaksa Qwen2.5-Coder 3B menjadi model 64K hanya dengan mengubah konfigurasi Hermes.

Menurut katalog resmi Ollama, Qwen2.5-Coder 3B memiliki context window 32K. Sementara itu, dokumentasi Hermes Agent saat ini mensyaratkan minimal 64K untuk workflow agent. Mengubah angka di konfigurasi Hermes saja tidak menambah kemampuan model.

ollama.com

+1

## 1. Tentukan jalur instalasi

Opsi A — Ollama + Qwen2.5-Coder 3B

Hemat resourceJalankan model untuk percakapan lokal, analisis log, dan bantuan scripting. Ini pilihan awal yang masuk akal untuk RAM 8 GB, tetapi bukan konfigurasi Hermes Agent penuh yang memenuhi syarat konteks 64K.

Opsi B — Hermes Agent dengan model yang memenuhi syarat

Pilih model dengan konteks minimal 64K dan kemampuan tool-calling yang baik. Bisa memakai provider hosted atau menguji model lokal yang mendukung konteks tersebut. Pada 4 vCPU dan RAM 8 GB, model lokal 64K tetap perlu diuji dari sisi memori dan kecepatan.

Saya sarankan tetap menggunakan Qwen2.5-Coder 3B untuk pengujian awal Ollama, lalu menentukan jalur Hermes berdasarkan hasil tes dan kebutuhan penggunaan sebenarnya.

## 2. Instal Ollama di Ubuntu atau Debian

Jalankan setiap bagian secara berurutan melalui SSH.

### 2.1 Periksa resource server

bash
```
cat /etc/os-release
nproc
free -h
df -h /
```

Pastikan ruang disk cukup untuk model dan cache. Jangan lanjut mengaktifkan agent otomatis sebelum memastikan server memiliki ruang RAM yang memadai.

zstd
```
sudo apt-get install zstd
```

### 2.2 Instal dan aktifkan Ollama

bash
```
curl -fsSL https://ollama.com/install.sh | sh
```

Periksa hasil instalasi:

bash
```
ollama --version
sudo systemctl enable --now ollama
sudo systemctl status ollama --no-pager
```

Uji API lokal:

bash
```
curl http://127.0.0.1:11434/api/tags
```

Jika hasilnya berupa JSON, API Ollama sudah merespons. Gunakan `127.0.0.1` agar koneksi tidak perlu dibuka ke internet.

## 3. Instal Qwen2.5-Coder 3B

Unduh model:

bash
```
ollama pull qwen2.5-coder:3b
```

Periksa:

bash
```
ollama list
```

Tes:

bash
```
ollama run qwen2.5-coder:3b
```

Ketik:

text
```
Jelaskan cara memeriksa service Linux yang gagal,
penggunaan RAM, penggunaan disk, dan error Nginx.
Jangan menjalankan perintah atau mengubah sistem.
```

Keluar dengan `/bye`.

Kamu juga bisa menguji API OpenAI-compatible yang nantinya dipakai Hermes:

bash
```
curl http://127.0.0.1:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-coder:3b",
    "messages": [
      {
        "role": "user",
        "content": "Berikan perintah untuk memeriksa service Linux yang gagal."
      }
    ],
    "stream": false
  }'
```

Jika model mengembalikan jawaban, endpoint kompatibel tersebut berfungsi. Namun, tes chat ini belum menguji kemampuan tool-calling Hermes.

## 4. Solusi context length: gunakan model yang tepat

Ada dua angka yang berbeda:

- Context window model: batas konteks yang memang didukung model.
- Context yang dialokasikan Ollama: kapasitas yang disiapkan untuk sesi saat model berjalan.

Hermes perlu mengetahui batas sebenarnya, sedangkan Ollama harus dikonfigurasi untuk menyediakan kapasitas tersebut. Keduanya harus cocok.

Hermes Agent

+1

Untuk servermu, ada alternatif lokal yang lebih sesuai untuk diuji: Qwen3 4B. Katalog Ollama mencantumkan model ini dengan context window 256K, sehingga memiliki ruang konteks yang cukup untuk konfigurasi Hermes 64K. Model ini bukan model khusus coding seperti Qwen2.5-Coder, tetapi lebih layak untuk eksperimen agent yang membutuhkan konteks panjang.

ollama.com

### 4.1 Unduh Qwen3 4B

bash
```
ollama pull qwen3:4b
```

Buat varian yang meminta context 64K:

bash
```
cat > Modelfile.qwen3-64k <<'EOF'
FROM qwen3:4b
PARAMETER num_ctx 65536
EOF
```

Kemudian:

bash
```
ollama create qwen3:4b-64k -f Modelfile.qwen3-64k
```

Perintah ini membuat varian lokal dengan pengaturan konteks 65.536 token. Ini bukan jaminan bahwa server RAM 8 GB mampu menjalankannya dengan lancar. Penggunaan RAM aktual harus diuji karena cache konteks yang lebih besar membutuhkan memori tambahan.

### 4.2 Tes konteks dan RAM

Jalankan:

bash
```
ollama run qwen3:4b-64k
```

Di terminal SSH kedua:

bash
```
watch -n 2 free -h
```

Setelah itu, di terminal pertama, kirim beberapa prompt dan biarkan model memprosesnya. Periksa penggunaan memori:

bash
```
ollama ps
```

Perhatikan kolom `CONTEXT`, penggunaan RAM, dan apakah sistem mulai menggunakan swap secara berat.

Jika server kehabisan memori atau proses terhenti, hentikan pengujian dan kembali ke konteks yang lebih kecil untuk penggunaan Ollama biasa. Jangan mengklaim bahwa Hermes memenuhi persyaratan 64K jika konfigurasi aktual atau kemampuan model belum terverifikasi.

## 5. Hubungkan Hermes Agent dengan Ollama

Untuk integrasi resmi, Ollama menyediakan perintah `ollama launch hermes` yang dapat membantu instalasi dan konfigurasi awal Hermes Agent.

GitHub

### 5.1 Instal dan jalankan Hermes

Jika Hermes belum terinstal, jalankan:

bash
```
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

Ikuti wizard yang muncul. Jika Hermes sudah terinstal, lanjutkan ke konfigurasi model.

Untuk jalur integrasi Ollama, kamu juga dapat mencoba:

bash
```
ollama launch hermes --model qwen2.5-coder:3b
```

Jika versi Ollama kamu tidak menyediakan perintah tersebut, gunakan wizard Hermes secara langsung.

### 5.2 Konfigurasi provider

Jalankan:

bash
```
hermes model
```

Pilih `Custom endpoint`, lalu masukkan:

| Pengaturan | Nilai |
| --- | --- |
| API base URL | `http://127.0.0.1:11434/v1` |
| API key | Kosong jika diizinkan |
| Model | `qwen3:4b-64k` |
| Context length | `65536` |

Jika wizard tidak mendeteksi model secara otomatis, pastikan nama model sama persis dengan hasil:

bash
```
ollama list
```

Jika kamu tetap memilih `qwen2.5-coder:3b`, jangan masukkan `65536` sebagai klaim bahwa model tersebut sudah mendukung 64K. Pilihan itu tidak menyelesaikan batas konteks model.

Dokumentasi konfigurasi resmi: [Hermes Agent — LLM Providers](<https://hermes-agent.nousresearch.com/docs/integrations/providers/?utm_source=chatgpt.com>).

### 5.3 Jalankan Hermes dan tes tool-calling

bash
```
hermes
```

Mulai dengan tugas yang aman:

text
```
Saya mengelola server Linux.
Bantu saya menganalisis kondisi server.

Jelaskan perintah read-only untuk:
1. Memeriksa service yang gagal.
2. Memeriksa penggunaan RAM dan disk.
3. Membaca 50 baris terakhir error Nginx.

Jangan menjalankan perintah apa pun tanpa persetujuan saya.
```

Periksa apakah Hermes dapat menjawab dengan benar, memanggil tool bila diperlukan, dan mengembalikan hasilnya tanpa error. Jawaban teks yang benar saja belum cukup untuk membuktikan workflow agent berfungsi.

## 6. Pastikan konfigurasi konteks benar

Setelah Hermes terhubung, periksa status model melalui CLI Hermes. Jika tersedia pada versi yang kamu gunakan, gunakan `/model` dan `/usage` untuk memeriksa model aktif serta konteks yang terdeteksi.

Di terminal lain, jalankan:

bash
```
ollama ps
```

Kamu ingin memastikan tiga hal:

- Hermes memakai model yang benar.
- Context aktual Ollama sesuai konfigurasi yang direncanakan.
- RAM masih cukup ketika model sedang aktif.

Jangan hanya mengandalkan nilai `context_length` di konfigurasi Hermes. Nilai tersebut mengatur pemahaman Hermes terhadap kapasitas model; nilai itu tidak memperbesar kapasitas yang benar-benar disediakan Ollama.

Hermes Agent

+1

Jika konteks benar tetapi server terlalu lambat, jangan langsung menambah context window. Mulai dengan prompt lebih pendek, kurangi data log yang dikirim, dan evaluasi penggunaan RAM.

## 7. Optimalkan penggunaan RAM 8 GB

Konfigurasi awal yang saya sarankan:

| Komponen | Pengaturan awal |
| --- | --- |
| Ollama | Satu model aktif |
| Model untuk eksperimen Hermes | Qwen3 4B dengan konteks 64K, jika lolos tes RAM |
| Qwen2.5-Coder 3B | Gunakan untuk coding dan analisis biasa |
| Monitoring | Periksa RAM, CPU, disk dan swap |
| Eksekusi perintah | Persetujuan manual |
| Akses API Ollama | Localhost saja |

Untuk monitoring berkala, lebih baik script mengirimkan ringkasan log dan metrik ketika ada anomali daripada mengirim seluruh log server terus-menerus. Ini mengurangi beban model dan membantu menjaga konteks tetap terkendali.

## 8. Troubleshooting yang umum

### Hermes menolak model karena context terlalu kecil

Periksa apakah model yang digunakan mendukung minimal 64K. Untuk Qwen2.5-Coder 3B, gunakan model lain yang mendukung konteks panjang jika ingin memenuhi persyaratan Hermes secara resmi.

### API Ollama tidak bisa diakses

Jalankan:

bash
```
sudo systemctl status ollama --no-pager
curl http://127.0.0.1:11434/v1/models
```

Pastikan Hermes dan Ollama berada pada host yang sama. Jika Hermes berjalan di container, `127.0.0.1` menunjuk ke container itu sendiri, bukan host.

### Server lambat atau kehabisan RAM

Jalankan:

bash
```
free -h
ollama ps
sudo journalctl -u ollama -n 100 --no-pager
```

Turunkan konteks atau gunakan model yang lebih ringan untuk penggunaan lokal biasa. Jangan memaksa 64K pada model yang tidak mendukungnya.

### Hermes bisa chat tetapi gagal menjalankan tools

Uji satu tugas sederhana, misalnya membaca status service. Model yang bagus untuk percakapan belum tentu bagus untuk mengikuti format tool-calling. Pastikan juga endpoint, nama model, dan konteks aktual benar.

## 9\. Rekomendasi akhir saya

Untuk server 4 vCPU dan RAM 8 GB, saya akan menggunakan Qwen2.5-Coder 3B untuk bantuan coding lokal dan menguji Qwen3 4B dengan konteks 64K untuk Hermes Agent. Jika pengujian 64K menyebabkan kehabisan RAM atau terlalu lambat, solusi yang lebih andal adalah memakai model hosted untuk Hermes sambil tetap mempertahankan Ollama lokal untuk tugas coding.

Jangan langsung memberikan akses root atau mengaktifkan perbaikan otomatis. Mulai dengan monitoring read-only, notifikasi, dan persetujuan manual.

Agar kita tidak salah mengubah konfigurasi servermu, langkah berikutnya yang paling berguna adalah mengetahui kondisi aktualnya. Jalankan perintah berikut dan kirim hasilnya (hapus informasi sensitif terlebih dahulu):

bash
```
cat /etc/os-release
free -h
ollama --version
ollama list
ollama ps
hermes --version
```

Dengan hasil tersebut, kita bisa menentukan apakah sebaiknya melanjutkan dengan Qwen2.5-Coder 3B, beralih ke Qwen3 4B untuk Hermes, atau menggunakan model hosted sebagai provider utama tanpa membebani server.
