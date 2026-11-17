# Paket A – Interview Ujian Praktik DevSecOps

Paket ini berisi lima soal pilihan dari cheatsheet utama: **Soal 1, 5, 13, 17, dan 21**. Setiap soal dilengkapi tujuan, langkah demonstrasi, penjelasan konsep, kriteria keberhasilan, dan contoh jawaban lisan.

> **Catatan:** Jalankan perintah pada direktori lab yang sesuai. Verifikasi nama service, path konfigurasi, schema, tabel, dan kolom dari repository serta konfigurasi aktual. Jangan mengarang output; gunakan hasil pengujian sendiri sebagai evidence.

## Daftar soal

| No. | Bab | Materi |
|---|---|---|
| 1 | Bab 6 | Membuktikan alur centralized logging end-to-end |
| 5 | Bab 6 | Membuktikan buffering ketika backend logging mati |
| 13 | Bab 7 | Memeriksa status target dan memahami metric `up` |
| 17 | Bab 7 | Negative test alert `TargetDown` |
| 21 | PMB | Membuktikan baseline transaksi normal |

---

## Soal 1 – Bab 6: End-to-end centralized logging

**Pertanyaan ujian:** Demonstrasikan dan jelaskan alur log secara end-to-end, mulai dari log yang dihasilkan container aplikasi hingga diterima Fluent Bit dan tersimpan di PostgreSQL. Tunjukkan bukti bahwa log dari generator benar-benar masuk ke database, serta jelaskan peran Docker logging driver dan Forward Protocol.

### Tujuan
Membuktikan bahwa log dari aplikasi/container dapat melewati Docker logging driver, diterima Fluent Bit, lalu disimpan ke PostgreSQL.

### Langkah demonstrasi

1. Masuk ke direktori lab Bab 6 dan periksa container:

```bash
cd ~/docker-lab/bab-6
docker compose ps -a
docker compose config --services
```

2. Periksa log generator dan Fluent Bit. Sesuaikan nama service dengan hasil `docker compose config --services`.

```bash
docker compose logs --tail=50 fluent-bit
docker compose logs --tail=30 generator-auth generator-api generator-worker
```

3. Periksa konfigurasi Fluent Bit dan Docker logging driver:

```bash
cat fluent-bit/fluent-bit.conf
grep -n -A8 -B3 -i 'fluentd' compose.yaml
```

4. Periksa data pada PostgreSQL. Contoh berikut mengasumsikan service database bernama `postgres-db`, database menggunakan environment variable standar, dan tabel target `logs.fluentbit`. Sesuaikan bila berbeda.

```bash
docker compose exec postgres-db sh
psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

Di dalam psql:

```sql
SELECT tag, COUNT(*) AS total
FROM logs.fluentbit
GROUP BY tag
ORDER BY tag;

SELECT time, tag, data
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 10;
```

### Alur yang harus dijelaskan

1. Generator aplikasi menulis log ke stdout/stderr.
2. Docker logging driver Fluentd menangkap log container.
3. Driver mengirim record menggunakan Forward Protocol ke input Fluent Bit (umumnya port 24224).
4. Fluent Bit menerima dan memproses record.
5. Output PostgreSQL mengirim record ke database.
6. PostgreSQL menyimpan record pada tabel yang ditentukan konfigurasi.

Tag digunakan untuk membedakan sumber log. Nama service, alamat, port, database, schema, dan tabel harus disebut sesuai konfigurasi aktual.

### Contoh jawaban lisan

“Log dihasilkan oleh container aplikasi melalui stdout atau stderr. Docker logging driver meneruskan log menggunakan Forward Protocol ke Fluent Bit. Fluent Bit menerima dan memproses record, kemudian mengirimkannya ke PostgreSQL. Keberhasilan alur saya buktikan dengan memeriksa log collector dan keberadaan record pada tabel tujuan, termasuk tag sumbernya.”

### Kriteria PASS

- Generator dan Fluent Bit berjalan atau menghasilkan log sesuai skenario.
- Konfigurasi menunjukkan jalur pengiriman log yang benar.
- Record dari sumber yang diuji ditemukan pada tabel PostgreSQL.
- Mahasiswa dapat menjelaskan fungsi Docker logging driver, Forward Protocol, Fluent Bit, dan PostgreSQL.

---

## Soal 5 – Bab 6: Buffering ketika backend mati

**Pertanyaan ujian:** Demonstrasikan apa yang terjadi ketika backend PostgreSQL dimatikan saat Fluent Bit sedang menerima log. Jelaskan bagaimana mekanisme buffering/retry bekerja, bagaimana membedakan memory buffer dan filesystem buffering, serta buktikan bahwa backlog log berhasil dikirim setelah backend kembali pulih.

### Tujuan
Membuktikan bahwa Fluent Bit dapat menahan atau melakukan retry pengiriman log saat PostgreSQL tidak tersedia, kemudian mengirim backlog setelah backend pulih.

### Langkah demonstrasi

1. Periksa konfigurasi dan script pengujian:

```bash
cd ~/docker-lab/bab-6
grep -n -E 'Mem_Buf_Limit|storage.type|storage.path|Retry_Limit' fluent-bit/fluent-bit.conf
sed -n '1,240p' scripts/test-buffering.sh
```

2. Jalankan skenario buffering:

```bash
./scripts/test-buffering.sh
```

3. Amati log Fluent Bit selama backend mati:

```bash
docker compose logs --tail=200 fluent-bit
```

4. Pastikan backend telah pulih, kemudian periksa log Fluent Bit kembali:

```bash
docker compose ps -a
docker compose logs --since=10m fluent-bit
```

5. Verifikasi record pada PostgreSQL. Gunakan query dan identitas event yang sesuai dengan skenario lab:

```sql
SELECT tag, COUNT(*) AS total
FROM logs.fluentbit
GROUP BY tag
ORDER BY tag;

SELECT time, tag, data
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 20;
```

### Konfigurasi aktual pada lab

Berdasarkan pemeriksaan `fluent-bit/fluent-bit.conf` pada lab, konfigurasi yang ditemukan adalah:

```ini
storage.path              /var/log/flb-storage
Mem_Buf_Limit 10M
storage.type  filesystem
```

Interpretasinya:

- `storage.path /var/log/flb-storage` menetapkan direktori penyimpanan chunk/buffer filesystem di dalam container Fluent Bit. Pastikan direktori ini memiliki permission yang sesuai dan, jika ingin bertahan melewati penggantian container, lokasi tersebut perlu didukung volume/persistensi pada konfigurasi Docker.
- `Mem_Buf_Limit 10M` menetapkan batas penggunaan memory buffer untuk input sebesar 10 MB. Ini bukan berarti kapasitas filesystem buffer hanya 10 MB.
- `storage.type filesystem` mengaktifkan filesystem buffering untuk input tersebut.

Hasil grep yang tidak menampilkan `Retry_Limit` berarti baris itu tidak ditemukan pada file yang diperiksa. Jangan menyimpulkan nilai retry tertentu tanpa memeriksa konfigurasi output dan default Fluent Bit yang berlaku. Secara umum output akan melakukan retry sesuai kebijakan retry yang dikonfigurasi.

### Cara kerja script pengujian

Script `scripts/test-buffering.sh` yang diperiksa melakukan urutan berikut:

1. Membaca variabel database dari file `.env`.
2. Menghitung jumlah record awal pada `logs.fluentbit`.
3. Menghentikan service `postgres-db`.
4. Menjalankan ulang `generator-auth`, `generator-api`, dan `generator-worker`.
5. Menunggu 25 detik ketika PostgreSQL dihentikan.
6. Menyalakan kembali PostgreSQL dan menunggu sampai `pg_isready` menyatakan database siap.
7. Memeriksa jumlah record secara berkala. Script mengeluarkan PASS jika jumlah record bertambah.

Jalankan dengan:

```bash
cd ~/docker-lab/bab-6
bash scripts/test-buffering.sh
```

**Hal yang perlu diperhatikan:** script menghitung kenaikan total record, bukan mencocokkan identitas setiap log yang dibuat selama PostgreSQL mati. Karena itu, untuk bukti yang lebih kuat, cocokkan timestamp, tag, atau ID event dari log generator dengan record yang masuk setelah recovery. Periksa juga apakah menjalankan `docker compose up` untuk generator ikut menyalakan dependency PostgreSQL melalui `depends_on`; jika iya, skenario backend mati bisa tidak berlangsung sesuai rencana. Verifikasi dengan `docker compose ps -a` dan log aktual.



 Fluent Bit menggunakan filesystem buffering dengan direktori `/var/log/flb-storage`, `storage.type filesystem`, dan batas memory buffer input 10 MB. Saat PostgreSQL dimatikan, Fluent Bit tidak dapat mengirim log ke output sehingga pengiriman akan dicoba kembali dan data ditahan sesuai mekanisme buffer. Script pengujian menghitung jumlah record awal, menghentikan PostgreSQL, menjalankan generator selama 25 detik, lalu menyalakan PostgreSQL kembali. Setelah database siap, script memeriksa apakah jumlah record bertambah. Untuk memastikan backlog benar-benar pulih, saya juga memeriksa log Fluent Bit dan mencocokkan record berdasarkan tag, waktu, atau ID event. Error koneksi sementara saat backend mati memang diharapkan dan bukan otomatis berarti pengujian gagal.

### Kriteria PASS

- Backend benar-benar dibuat tidak tersedia selama pengujian.
- Terlihat bukti retry atau mekanisme buffering pada log/configuration.
- Backend berhasil dipulihkan.
- Record yang dihasilkan selama gangguan dapat diverifikasi masuk setelah recovery.
- Mahasiswa dapat membedakan memory buffer dan filesystem buffering.

**Catatan:** PASS bukan berarti tidak ada pesan error. Error koneksi sementara memang diharapkan pada pengujian ini. Yang dinilai adalah pemulihan pengiriman dan bukti data, bukan sekadar container kembali berstatus Up.

---

## Soal 13 – Bab 7: Status target dan metric `up`

**Pertanyaan ujian:** Tunjukkan status target pada Prometheus dan jelaskan arti metric `up = 1` serta `up = 0`. Apakah seluruh target yang berstatus UP menjamin aplikasi sehat secara keseluruhan? Berikan alasan dan contoh.

### Tujuan
Memeriksa status target Prometheus dan menjelaskan arti metric `up` untuk keberhasilan scrape.

### Langkah demonstrasi

1. Masuk ke direktori Bab 7:

```bash
cd ~/docker-lab/bab-7
docker compose ps -a
```

2. Periksa target melalui API Prometheus:

```bash
curl -fsS http://127.0.0.1:9090/api/v1/targets
```

3. Jalankan script verifikasi stack jika tersedia:

```bash
./scripts/verify-stack.sh
```

4. Buka Prometheus Graph pada `http://127.0.0.1:9090` dan jalankan query:

```promql
up
```

Untuk melihat target tertentu:

```promql
up{job="node"}
```

Sesuaikan label `job` berdasarkan konfigurasi Prometheus aktual.

### Konsep yang harus dijelaskan

- `up = 1`: Prometheus berhasil melakukan scrape target pada scrape tersebut.
- `up = 0`: scrape ke target gagal.
- Target berstatus UP hanya menunjukkan endpoint metrics dapat dijangkau dan scrape berhasil.
- Status UP **tidak** membuktikan seluruh fungsi bisnis aplikasi berjalan normal.

### Contoh jawaban lisan

“Metric up menunjukkan hasil scrape Prometheus terhadap suatu target. Nilai 1 berarti scrape berhasil, sedangkan nilai 0 berarti scrape gagal. Namun, target yang UP belum tentu berarti aplikasi sehat secara bisnis. Contohnya endpoint metrics tetap merespons, tetapi proses pendaftaran pengguna mengalami timeout karena dependency lain bermasalah.”

Maksud dari ratio_rate5m adalah nilai rasio penggunaan CPU yang dihitung berdasarkan perubahan data selama rentang waktu 5 menit terakhir.

Kita pecah menjadi dua bagian:
1. Ratio (rasio)
Ratio berarti perbandingan penggunaan CPU terhadap total kapasitas CPU.

Contoh:
- 0.25 = CPU digunakan 25%.
- 0.50 = CPU digunakan 50%.
- 0.80 = CPU digunakan 80%.
Jadi, hasilnya berupa angka desimal antara 0 sampai 1.
2. Rate 5m
rate5m mengacu pada perhitungan berdasarkan data dalam rentang 5 menit terakhir.

Prometheus menggunakan data metric yang dikumpulkan secara berkala untuk menghitung tingkat perubahan selama rentang tersebut.

Misalnya:
Waktu	Penggunaan CPU
14.00	20%
14.01	30%
14.02	40%
14.03	35%
14.04	50%

Nilai yang dihasilkan merepresentasikan estimasi penggunaan CPU berdasarkan data dalam rentang tersebut, bukan hanya mengambil angka pada satu detik terakhir.
3. Mengapa menggunakan 5 menit?
Karena penggunaan CPU bisa naik turun dengan cepat.

Dengan rentang 5 menit, monitoring dapat memberikan gambaran penggunaan CPU yang lebih stabil dan tidak terlalu terpengaruh lonjakan sesaat.

Kesimpulan sederhana
- Ratio: seberapa besar CPU digunakan dibandingkan kapasitas total.
- Rate 5m: perhitungan berdasarkan perubahan data selama 5 menit terakhir.
- Ratio_rate5m: rasio penggunaan CPU yang dihitung menggunakan data dalam jendela 5 menit.

"ratio menunjukkan hasil dalam bentuk perbandingan atau persentase penggunaan CPU, sedangkan rate5m menunjukkan bahwa perhitungan menggunakan rentang data lima menit terakhir. Tujuannya agar monitoring penggunaan CPU memberikan gambaran kondisi yang lebih stabil, bukan hanya berdasarkan penggunaan CPU pada satu waktu tertentu."

Catatan: secara teknis, rate() di PromQL menghitung laju perubahan counter per detik. Untuk metric CPU ini, nama rate5m merupakan bagian dari penamaan Recording Rule; rumus aslinya perlu dilihat untuk memastikan detail perhitungannya.

### Kriteria PASS

- Mahasiswa dapat menampilkan daftar target Prometheus.
- Mahasiswa dapat menjalankan query `up`.
- Mahasiswa dapat menjelaskan perbedaan scrape availability dan business health.
- Mahasiswa tidak menyimpulkan seluruh aplikasi sehat hanya berdasarkan nilai `up = 1`.

---

## Soal 17 – Bab 7: Negative test alert `TargetDown`

**Pertanyaan ujian:** Lakukan negative test untuk alert `TargetDown` dengan menghentikan Node Exporter. Buktikan perubahan status alert hingga firing setelah durasi `for` terpenuhi, kemudian pulihkan exporter dan tunjukkan bahwa alert kembali resolved. Jelaskan apa yang dibuktikan dan yang belum dibuktikan oleh pengujian ini.

### Tujuan
Membuktikan bahwa alert `TargetDown` berubah menjadi firing ketika target exporter tidak tersedia selama durasi yang ditentukan, kemudian pulih setelah exporter dijalankan kembali.

### Persiapan

Periksa nama service exporter dan definisi alert terlebih dahulu:

```bash
cd ~/docker-lab/bab-7
docker compose config --services
grep -R -n -A12 -B4 'TargetDown' prometheus
```

Catat nilai `for` pada rule `TargetDown`. Waktu tunggu pengujian harus lebih panjang daripada durasi `for`, dengan mempertimbangkan scrape interval dan evaluation interval.

### Langkah demonstrasi

1. Periksa kondisi awal alert melalui API Prometheus:

```bash
curl -G -fsS http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=ALERTS{alertname="TargetDown"}'
```

2. Hentikan service Node Exporter. Gunakan nama service yang benar dari konfigurasi:

```bash
docker compose stop node-exporter
```

3. Tunggu sampai melewati durasi `for` pada rule. Jika konfigurasi dan interval sesuai skenario praktikum, waktu tunggu dapat menggunakan 75 detik. Jangan menganggap 75 detik selalu cukup tanpa memeriksa rule.

```bash
sleep 75
```

4. Periksa alert kembali:

```bash
curl -G -fsS http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=ALERTS{alertname="TargetDown"}'
```

5. Jalankan kembali exporter:

```bash
docker compose start node-exporter
```

6. Periksa target dan alert sampai kondisi pulih:

```bash
curl -fsS http://127.0.0.1:9090/api/v1/targets
```

Jalankan kembali query `ALERTS{alertname="TargetDown"}` untuk mengamati perubahan status.

### Konsep yang harus dijelaskan

- **Pending:** kondisi ekspresi alert terpenuhi, tetapi durasi `for` belum terlampaui.
- **Firing:** kondisi terus terpenuhi selama durasi `for`.
- **Resolved/inactive:** kondisi tidak lagi terpenuhi setelah target pulih dan Prometheus mengevaluasi ulang rule.

### Contoh jawaban lisan

“Negative test dilakukan dengan menghentikan Node Exporter agar Prometheus gagal melakukan scrape. Kondisi TargetDown kemudian terpenuhi. Alert awalnya pending dan berubah menjadi firing setelah durasi for terlampaui. Setelah exporter dijalankan kembali, scrape pulih dan alert akan resolved pada evaluasi berikutnya.”

### Kriteria PASS

- Target exporter terbukti gagal di-scrape ketika dihentikan.
- Alert `TargetDown` dapat diamati berubah menjadi firing setelah durasi yang ditentukan.
- Exporter berhasil dijalankan kembali dan target kembali UP.
- Mahasiswa dapat menjelaskan pending, firing, dan resolved.

**Batas pembuktian:** Negative test ini membuktikan evaluasi alert di Prometheus. Pengujian ini belum membuktikan Alertmanager mengirim notifikasi, kanal notifikasi berfungsi, atau operator benar-benar menerima pesan.

---

## Soal 21 – Portal PMB: Baseline transaksi normal

**Pertanyaan ujian:** Jalankan skenario baseline transaksi normal pada Portal PMB. Gunakan `request_id` untuk menelusuri kronologi log pada `registration-api`, `auth-service`, dan `verification-worker`, lalu verifikasi status akhir transaksi pada database aplikasi. Jelaskan perbedaan event time, collector receive time, dan database insert time.

### Tujuan
Membuktikan alur transaksi pendaftaran normal melalui log beberapa service dan status akhir transaksi pada database aplikasi.

### Langkah demonstrasi

1. Masuk ke direktori project Portal PMB dan periksa stack:

```bash
cd ~/docker-lab/bab-6-pmb/pmb-devsecops-lab
docker compose ps -a
docker compose config --services
```

2. Baca script sebelum menjalankannya:

```bash
sed -n '1,260p' scripts/demo-normal.sh
```

3. Jalankan transaksi normal:

```bash
./scripts/demo-normal.sh
```

4. Catat **request_id aktual** dan timestamp yang ditampilkan script. Jangan menggunakan request ID contoh atau placeholder sebagai bukti.

5. Periksa kronologi log dari service terkait:

```bash
docker compose logs --since=15m --no-color \
  registration-api auth-service verification-worker
```

Filter berdasarkan request ID aktual (ganti placeholder berikut):

```bash
docker compose logs --since=15m --no-color \
  registration-api auth-service verification-worker \
  | grep 'REQUEST_ID_AKTUAL'
```

6. Periksa schema database aplikasi:

```bash
docker compose exec app-db sh
```

Masuk ke psql menggunakan kredensial yang ditetapkan pada environment lab, lalu periksa tabel dan kolom:

```sql
\dt
\d+ NAMA_TABEL_TRANSAKSI
```

Setelah nama tabel dan kolom dipastikan, cari transaksi berdasarkan request ID aktual:

```sql
SELECT *
FROM NAMA_TABEL_TRANSAKSI
WHERE request_id = 'REQUEST_ID_AKTUAL';
```

Ganti `NAMA_TABEL_TRANSAKSI` dan `REQUEST_ID_AKTUAL` dengan nilai aktual. Jangan menyalin query sebelum memastikan nama schema, tabel, dan kolom.

### Alur yang harus dijelaskan

1. `registration-api` menerima request pendaftaran.
2. `auth-service` menjalankan proses autentikasi sesuai alur aplikasi.
3. `verification-worker` menjalankan proses verifikasi.
4. Log dari service-service tersebut dikorelasikan menggunakan request ID yang sama.
5. Database aplikasi menyimpan status akhir transaksi.

Baseline dinyatakan normal apabila kronologi log menunjukkan alur yang diharapkan dan record transaksi dengan request ID yang sama memiliki status akhir **VERIFIED**.

### Contoh jawaban lisan

“Pada baseline, saya menjalankan script demo-normal untuk membuat transaksi normal. Saya mencatat request ID dari output, lalu menelusuri log registration-api, auth-service, dan verification-worker menggunakan ID yang sama. Setelah itu saya memeriksa database aplikasi. Hasil yang diharapkan adalah rangkaian proses normal dan transaksi berakhir dengan status VERIFIED.”

### Perbedaan tiga timestamp

- **Event time:** waktu kejadian menurut aplikasi.
- **Collector receive time:** waktu Fluent Bit menerima record.
- **Database insert time:** waktu record disimpan ke database log.

Ketiganya dapat berbeda karena jeda jaringan, buffering, retry, dan pemrosesan. Untuk membangun kronologi, perhatikan sinkronisasi jam, timezone, dan definisi timestamp yang digunakan setiap komponen.

### Kriteria PASS

- Script baseline berhasil dijalankan.
- Request ID dan timestamp dicatat dari output aktual.
- Log dari service yang relevan dapat ditelusuri menggunakan request ID yang sama.
- Record transaksi ditemukan pada database aplikasi.
- Status akhir transaksi sesuai ekspektasi, yaitu VERIFIED.
- Mahasiswa dapat menjelaskan korelasi log dan transaksi serta perbedaan timestamp.

---

## Checklist akhir Paket A

- [ ] Soal 1: alur log dari generator hingga PostgreSQL terbukti.
- [ ] Soal 5: retry/buffering dan pengiriman backlog setelah recovery terbukti.
- [ ] Soal 13: status target dan arti metric `up` dijelaskan dengan benar.
- [ ] Soal 17: alert TargetDown terbukti firing dan pulih setelah exporter dinyalakan.
- [ ] Soal 21: transaksi normal dapat ditelusuri dari log hingga status VERIFIED.
- [ ] Semua bukti berasal dari output aktual praktikum.
- [ ] Tidak ada password, token, atau secret yang ditampilkan.
- [ ] Nama service, file konfigurasi, tabel, dan kolom telah diverifikasi sebelum menjalankan perintah.

## Referensi

- [Cheatsheet Interview DevSecOps Bab 6–7 dan PMB](./CHEATSHEET-INTERVIEW-DEVSECOPS-BAB-6-7-PMB.md)
