# Apache-Kafka-BigData
---

# 📦 Real-Time Warehouse Monitoring System with Kafka & PySpark

Sebuah proyek simulasi untuk memantau kondisi gudang logistik secara real-time menggunakan **Apache Kafka** dan **PySpark**. Sistem ini mensimulasikan pengiriman data suhu dan kelembaban dari beberapa gudang, melakukan **filtering**, dan mendeteksi kondisi kritis berdasarkan suhu tinggi dan kelembaban berlebih.

---

## 🎯 Latar Belakang Masalah

Perusahaan logistik menyimpan barang sensitif seperti makanan, obat-obatan, dan elektronik di gudang. Untuk menjaga kualitas penyimpanan:

* **Sensor Suhu** dan **Sensor Kelembaban** mengirim data setiap detik.
* Data perlu dipantau secara **real-time** untuk mencegah kerusakan.

---

## 📋 Tugas Mahasiswa

### 1. Buat Topik Kafka

Buat dua topik:

* `sensor-suhu-gudang`
* `sensor-kelembaban-gudang`

```bash
kafka-topics --create --topic sensor-suhu-gudang --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics --create --topic sensor-kelembaban-gudang --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

### 2. Simulasikan Data Sensor (Producer Kafka)

* Minimal 3 gudang: `G1`, `G2`, `G3`
* Format JSON:

  * Suhu: `{"gudang_id": "G1", "suhu": 82}`
  * Kelembaban: `{"gudang_id": "G1", "kelembaban": 75}`

### 3. Konsumsi dan Olah Data dengan PySpark

* Konsumsi data dari Kafka.
* Filtering:

  * Suhu > 80°C → ⚠️ Peringatan Suhu Tinggi
  * Kelembaban > 70% → ⚠️ Peringatan Kelembaban Tinggi

Contoh output:

```text
[Peringatan Suhu Tinggi]
Gudang G2: Suhu 85°C

[Peringatan Kelembaban Tinggi]
Gudang G3: Kelembaban 74%
```

### 4. Gabungkan Stream dari Dua Sensor

* Join stream berdasarkan `gudang_id` dan window waktu 10 detik.
* Jika suhu > 80°C dan kelembaban > 70% → 🔴 PERINGATAN KRITIS

Contoh output:

```text
[PERINGATAN KRITIS]
Gudang G1:
- Suhu: 84°C
- Kelembaban: 73%
- Status: Bahaya tinggi! Barang berisiko rusak

Gudang G3:
- Suhu: 85°C
- Kelembaban: 65%
- Status: Suhu tinggi, kelembaban normal
```
---

## 🎓 Tujuan Pembelajaran

Mahasiswa diharapkan dapat:

* Memahami arsitektur dan alur kerja **Apache Kafka**.
* Membuat **Kafka Producer dan Consumer**.
* Menerapkan **real-time filtering** menggunakan **PySpark**.
* Melakukan **stream joining** untuk analisis kondisi ganda.
* Menghasilkan output berbasis **logika bisnis** untuk pengambilan keputusan.
---
### PENGERJAAN

## 🛠️ Struktur Proyek

```
project-warehouse/
│
├── docker-compose.yml
├── producer/
│   ├── producer_suhu.py
│   └── producer_kelembaban.py
└── consumer/
    └── pyspark_consumer.py
```

---

## 🚀 Langkah Eksekusi

### 1. Setup Proyek

```bash
mkdir project-warehouse
cd project-warehouse
mkdir producer consumer
notepad docker-compose.yml
```

### 2. Buat File Producer

```bash
cd producer
notepad producer_suhu.py
notepad producer_kelembaban.py
```

### 3. Buat File PySpark Consumer

```bash
cd ../consumer
notepad pyspark_consumer.py
```

### 4. Jalankan Kafka dan Spark

```bash
cd ..
docker-compose up -d
```

### 5. Buat Kafka Topic di dalam container Kafka

```bash
docker exec -it kafka bash
kafka-topics --create --topic sensor-suhu-gudang --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
kafka-topics --create --topic sensor-kelembaban-gudang --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
exit
```

### 6. Jalankan Kafka Producer

Buka terminal baru:

```bash
docker exec -it producer bash
pip install kafka-python
python producer/producer_kelembaban.py
```

### 7. Jalankan PySpark Consumer

```bash
docker exec -it spark bash
pip install kafka-python
spark-submit \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.0 \
  consumer/pyspark_consumer.py
```

---
### Penjelasan Isi dari .py dan .yml
---

## 🔧 `docker-compose.yml`

Konfigurasi ini menyusun semua service yang diperlukan:

### 1. **Zookeeper**

* **Peran:** Service koordinasi untuk Kafka.
* **Port:** `2181` (default Zookeeper).
* **Image:** `confluentinc/cp-zookeeper:7.4.0`.

### 2. **Kafka**

* **Peran:** Message broker utama.
* **Port:**

  * `9092`: komunikasi Kafka ke luar container.
  * `9101`: monitoring JMX.
* **Konfigurasi penting:**

  * `KAFKA_ZOOKEEPER_CONNECT`: Menghubungkan ke zookeeper.
  * `KAFKA_ADVERTISED_LISTENERS`: Menyediakan dua jalur akses, satu untuk container (`kafka:29092`), satu untuk host (`localhost:9092`).

### 3. **Producer**

* **Peran:** Menghasilkan data suhu dan kelembaban.
* **Image:** `python:3.9-slim`.
* **Volume:** mount `./producer` ke dalam container, tapi working dir disetel ke `/app` (harus diperbaiki agar ke `/app/producer`).

### 4. **Spark (Consumer)**

* **Peran:** Mengkonsumsi data dari Kafka dan memproses stream dengan PySpark.
* **Port:**

  * `8080`: UI Spark.
  * `7077`: Cluster manager Spark.
* **Environment:** Spark mode disetel sebagai master, tanpa enkripsi (untuk dev/testing).

---

## 🔥 PySpark Consumer (`pyspark-consumer.py`)

Skrip ini menjalankan pipeline streaming sebagai **real-time analytics** untuk memantau kondisi gudang:

### 🔹 Langkah utama:

1. **Schema Definition**: Untuk data suhu dan kelembaban.
2. **Streaming Source**:

   * Kafka topik `sensor-suhu-gudang` dan `sensor-kelembaban-gudang`.
3. **Transformasi**:

   * Parse JSON, konversi `timestamp` ke `event_time`.
   * Filter untuk deteksi suhu > 80°C dan kelembaban > 70%.
4. **Stream Join**:

   * Join berdasarkan `gudang_id` dalam rentang waktu ±10 detik untuk korelasi antara suhu dan kelembaban.
5. **Output**:

   * Menampilkan status tiap batch:

     * Jika suhu dan kelembaban tinggi → **KRITIS**
     * Hanya salah satu tinggi → **Peringatan**
     * Semua normal → **Aman**

---

## 🧪 Kafka Producers

### ✅ `producer-suhu.py`

* Mengirim data suhu ke topik `sensor-suhu-gudang`.
* 30% kemungkinan menghasilkan suhu tinggi (80–90°C).
* Format JSON: `{"gudang_id": "G1", "suhu": 85, "timestamp": 1716352432}`.

### ✅ `producer-kelembapan.py`

* Mengirim data kelembaban ke topik `sensor-kelembaban-gudang`.
* 30% kemungkinan menghasilkan kelembaban tinggi (70–85%).

Keduanya menggunakan loop `while True` dan `KafkaProducer` dengan interval 1 detik.

---

## 💡 Catatan Teknis & Tips:

1. **Sinkronisasi waktu antar sensor penting** supaya join stream bekerja akurat.
2. `producer` dan `spark` menggunakan `tail -f /dev/null` agar tetap hidup — kamu bisa masuk ke dalam container pakai `docker exec` untuk menjalankan script secara manual, atau ubah command agar langsung menjalankan script-nya.
3. Bisa tambahkan **output sink (misalnya ke file, database, atau dashboard)** agar hasil stream tidak hanya di console.
4. Jika ingin menjalankan script langsung dalam container, update `command:` di docker-compose seperti:

   ```yaml
   command: python producer-suhu.py
   ```

---

## 📌 Catatan Tambahan

* Disarankan untuk menggunakan **Docker Desktop** agar lingkungan Kafka dan Spark terisolasi.
* Window join menggunakan Spark Structured Streaming dapat disesuaikan dengan kebutuhan.
* Pastikan timezone sinkron antara producer dan PySpark consumer untuk akurasi join stream.

## Hasil:

**1. producer_suhu.py**
 
![image](https://github.com/user-attachments/assets/954f8d19-b263-48ec-bbf1-1ff887aaebbc)

**2. producer_kelembapan.py**

![Screenshot (623)](https://github.com/user-attachments/assets/dafe4551-ec46-4199-83f9-f68ca998ab3c)

**3. pyspark_consumer.py**
![Screenshot (621)](https://github.com/user-attachments/assets/eace875d-5731-4d39-944b-e81ac41372df)

![Screenshot (622)](https://github.com/user-attachments/assets/887d419d-f037-4c40-81d2-c5379924ab26)




