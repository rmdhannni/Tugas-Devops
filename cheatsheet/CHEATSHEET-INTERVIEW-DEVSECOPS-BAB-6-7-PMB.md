# Cheatsheet Interview Ujian Praktik DevSecOps
## Bab 6 – Centralized Logging, Bab 7 – Monitoring, dan Studi Kasus Portal PMB

> Panduan demonstrasi dan jawaban lisan. Jalankan perintah di lab sendiri dan gunakan output aktual sebagai evidence. Sesuaikan nama service, path, database, tabel, dan kolom dengan konfigurasi yang benar-benar ada di lab.

## Daftar isi
- [Persiapan](#1-persiapan-lab)
- [Bab 6 – Centralized Logging](#2-bab-6--centralized-logging)
- [Bab 7 – Monitoring](#3-bab-7--monitoring)
- [Studi Kasus Portal PMB](#4-studi-kasus-portal-pmb)
- [Ringkasan istilah](#5-ringkasan-istilah)
- [Strategi menjawab](#6-strategi-menjawab-penguji)

---

# 1. Persiapan lab

Pastikan stack dan environment tersedia sebelum ujian. Contoh direktori berikut mengikuti susunan lab yang digunakan; ubah jika lokasi lokal berbeda.

```bash
# Bab 6
cd ~/docker-lab/bab-6
docker compose config --quiet
docker compose ps -a

# Bab 7
cd ~/docker-lab/bab-7
docker compose config --quiet
docker compose ps -a

# Portal PMB
cd ~/docker-lab/bab-6-pmb/pmb-devsecops-lab
docker compose config --quiet
docker compose ps -a
```

Jangan menampilkan isi `.env`, password, token, atau secret saat demonstrasi. Periksa nama service aktual dengan `docker compose config --services`.

# 2. Bab 6 – Centralized Logging

## Soal 1 – Alur log end-to-end

**Demonstrasi**

```bash
docker compose ps -a
docker compose logs --tail=80 fluent-bit
docker compose logs --tail=20 generator-auth generator-api generator-worker
```

**Alur yang dijelaskan**

1. Generator menulis event ke stdout/stderr.
2. Docker logging driver Fluentd menangkap log container.
3. Log dikirim melalui Forward Protocol ke input Fluent Bit, umumnya port 24224.
4. Fluent Bit menerima dan memproses record.
5. Output PostgreSQL mengirim record ke database.
6. PostgreSQL menyimpan record ke tabel, pada lab ini ditujukan ke `logs.fluentbit`.

Tag membantu membedakan sumber/container. Sebutkan nama driver, alamat, port, dan nama service sesuai konfigurasi aktual.

**Pertanyaan lanjutan – Apa jika fluentd-address diubah menjadi `fluent-bit:24224`?**

Alamat tersebut menggunakan DNS internal Docker. Nama `fluent-bit` harus sama dengan nama service/container yang dapat di-resolve, dan pengirim serta collector harus berada pada network Docker yang saling terhubung. Jika syarat itu tidak terpenuhi, pengiriman log gagal atau tidak tersambung.

## Soal 2 – Verifikasi record dan JSONB

**Demonstrasi**

Masuk ke PostgreSQL menggunakan kredensial dari environment tanpa mencetak password:

```bash
docker compose exec postgres-db sh
psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

Query:

```sql
SELECT tag, COUNT(*)
FROM logs.fluentbit
GROUP BY tag
ORDER BY tag;

SELECT pg_typeof(data), COUNT(*)
FROM logs.fluentbit
GROUP BY pg_typeof(data);

SELECT
    data->>'container_name' AS container_name,
    data->>'source' AS source,
    data->>'log' AS message
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 10;
```

**Penjelasan**

JSONB adalah tipe PostgreSQL untuk menyimpan objek JSON secara terstruktur dan mendukung operasi/query terhadap properti JSON. Pada skema lab, metadata container, source, dan isi log berada di kolom `data`; nama properti harus dikonfirmasi dari record aktual.

**Pertanyaan lanjutan – Bagaimana severity menjadi field tersendiri?**

Parse severity pada tahap pemrosesan Fluent Bit (parser/filter) dan simpan sebagai properti JSON, misalnya `severity: "ERROR"`. Setelah itu query dapat menggunakan:

```sql
SELECT data->>'severity' AS severity, COUNT(*)
FROM logs.fluentbit
GROUP BY data->>'severity';
```

## Soal 3 – Konfigurasi output PostgreSQL

**Demonstrasi**

```bash
cat fluent-bit/fluent-bit.conf
```

Periksa search path (sesuaikan service dan database):

```bash
docker compose exec postgres-db sh -lc \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SHOW search_path;"'
```

**Penjelasan**

Fluent Bit mengetahui tujuan penulisan dari konfigurasi output PostgreSQL: host, port, database, user, password, dan tabel. Schema dapat ditentukan eksplisit pada nama tabel atau melalui mekanisme search_path sesuai kemampuan/konfigurasi plugin. Jangan mengasumsikan search_path otomatis menentukan schema jika konfigurasi plugin memakai nama tabel/schema secara eksplisit.

**Pertanyaan lanjutan – Error `relation fluentbit does not exist`**

Periksa secara berurutan:
1. Database tujuan dan kredensial.
2. Apakah schema `logs` tersedia.
3. Apakah tabel `logs.fluentbit` tersedia.
4. Nama tabel/schema pada konfigurasi output.
5. `search_path` dan hak akses user.
6. Apakah skrip inisialisasi SQL berhasil dijalankan.

```sql
\dn+
\dt logs.*
\d logs.fluentbit
SHOW search_path;
```

## Soal 4 – Least privilege akun ingest

**Demonstrasi**

```sql
\dp logs.fluentbit
\dn+ logs
\du
```

**Penjelasan**

Akun ingest seperti `fluentbit_user` sebaiknya memiliki hak minimum sesuai kebutuhan: CONNECT ke database, USAGE pada schema, dan INSERT pada tabel. SELECT hanya diberikan jika memang dibutuhkan oleh konfigurasi atau operasi. Hak aktual harus dibaca dari output privilege, bukan diasumsikan.

**Pertanyaan lanjutan – Apa dampak jika kredensial bocor?**

Kredensial ingest yang dibatasi dapat mengurangi dampak kebocoran menjadi operasi yang diizinkan pada objek tertentu. Kredensial administrator memiliki cakupan lebih luas, berpotensi memungkinkan perubahan schema, penghapusan data, pengelolaan role, dan akses ke objek lain.

## Soal 5 – Buffering saat backend mati

**Demonstrasi**

```bash
./scripts/test-buffering.sh
docker compose logs --tail=200 fluent-bit
```

**Penjelasan**

Saat PostgreSQL dihentikan, output tidak dapat mengirim record. Fluent Bit dapat melakukan retry dan menahan data pada buffer. Bedakan:
- Buffer memori, yang dibatasi oleh konfigurasi seperti `Mem_Buf_Limit`.
- Storage filesystem, jika `storage.type filesystem` dan `storage.path` dikonfigurasi.

Setelah backend pulih, Fluent Bit mencoba mengirim backlog. Pastikan bukti menunjukkan data tertahan lalu terkirim, bukan hanya bahwa container hidup.

**Pertanyaan lanjutan – Mengapa PASS bukan berarti tidak ada error?**

Kegagalan koneksi saat backend sengaja dimatikan memang diharapkan. Kriteria PASS adalah data tertahan sesuai mekanisme buffer dan berhasil dikirim setelah backend pulih, dengan jumlah/identitas event yang dapat diverifikasi.

## Soal 6 – Monitoring collector

**Demonstrasi**

```bash
curl -fsS http://127.0.0.1:2020/api/v1/health
curl -fsS http://127.0.0.1:2020/api/v1/metrics
```

**Penjelasan**

Health endpoint menunjukkan status health collector. Metrics menyediakan indikator input/output, jumlah record, error, retry, dan penggunaan buffer (nama metric bergantung versi dan konfigurasi).

**Pertanyaan lanjutan – Health OK tetapi tabel tidak bertambah?**

Periksa metric output PostgreSQL terlebih dahulu: record yang dikirim, error, retry, dan backlog. Lanjutkan dengan memeriksa konektivitas, kredensial, schema/tabel, privilege, dan log Fluent Bit. Health OK tidak membuktikan output berhasil menulis ke database.

## Soal 7 – Rekonsiliasi kelengkapan event

**Demonstrasi**

```sql
SELECT
    tag,
    COUNT(*) AS total_event,
    MIN(time) AS waktu_awal,
    MAX(time) AS waktu_akhir
FROM logs.fluentbit
GROUP BY tag
ORDER BY tag;
```

**Penjelasan**

Setiap generator menghasilkan 20 event per run. Bandingkan tiga titik:
1. Event yang dibuat generator.
2. Event yang diterima Fluent Bit.
3. Event yang tersimpan di PostgreSQL.

Selisih dapat disebabkan filter, kegagalan pengiriman, buffer overflow, retry, atau duplikasi. Pastikan rentang waktu dan run ID sama saat membandingkan.

**Pertanyaan lanjutan – Bagaimana mendeteksi/mencegah duplikasi?**

Berikan `event_id` unik pada setiap event, lalu gunakan unique constraint atau mekanisme upsert/idempotensi pada penyimpanan. Duplikasi dapat diperiksa dengan:

```sql
SELECT data->>'event_id' AS event_id, COUNT(*)
FROM logs.fluentbit
GROUP BY data->>'event_id'
HAVING COUNT(*) > 1;
```

Sesuaikan nama field dengan struktur JSONB aktual.

## Soal 8 – Redaction dengan nilai canary

**Demonstrasi**

1. Buat canary sintetis yang tidak mengandung secret sungguhan.
2. Tambahkan ke salah satu generator.
3. Terapkan filter redaction sebelum output penyimpanan.
4. Jalankan generator.
5. Cari canary pada database.

```sql
SELECT COUNT(*)
FROM logs.fluentbit
WHERE data::text ILIKE '%CANARY-UNIK-ANDA%';
```

Hasil yang diharapkan: nol record yang mengandung canary setelah redaction diterapkan.

**Pertanyaan lanjutan – Selain PostgreSQL, ke mana canary bisa bocor?**

Ke Docker logs, buffer filesystem Fluent Bit, log internal collector, terminal/rekaman sesi, backup, dashboard, tracing, atau sistem observability lain. Redaction perlu dilakukan sebelum data masuk ke jalur-jalur penyimpanan tersebut.

## Soal 9 – Kebijakan retensi

**Demonstrasi**

```bash
grep -n -i -E 'DELETE|retention' queries/analysis.sql
```

Hitung data yang lebih tua dari tujuh hari:

```sql
SELECT COUNT(*)
FROM logs.fluentbit
WHERE time < NOW() - INTERVAL '7 days';
```

Contoh perintah penghapusan (tetap dikomentari sampai kebijakan disetujui):

```sql
-- DELETE FROM logs.fluentbit
-- WHERE time < NOW() - INTERVAL '7 days';
```

**Pertanyaan lanjutan – Risiko TIMESTAMP WITHOUT TIME ZONE**

Tipe ini tidak menyimpan informasi zona waktu. Jika producer dan database memakai zona waktu berbeda, korelasi event dan perhitungan retensi dapat meleset. Gunakan UTC secara konsisten dan pertimbangkan `TIMESTAMPTZ`. Pastikan zona waktu sumber dan database terdokumentasi.

## Soal 10 – Ekspor log sebagai evidence

**Demonstrasi**

Di psql:

```sql
\copy (
    SELECT time, tag, data
    FROM logs.fluentbit
    WHERE time >= NOW() - INTERVAL '1 hour'
    ORDER BY time
) TO 'log-evidence.csv' CSV HEADER;
```

Kemudian pada shell tempat file dibuat:

```bash
sha256sum log-evidence.csv
```

**Penjelasan**

Evidence ekspor sebaiknya disertai rentang waktu dan zona waktu, query, waktu ekspor, database/tabel sumber, jumlah record, identitas operator, kebijakan redaction, serta checksum SHA-256.

**Pertanyaan lanjutan – Kapan PostgreSQL tidak cocok sebagai backend log utama?**

Ketika volume ingest, retensi, kebutuhan pencarian terdistribusi, cardinality, atau skalabilitas sudah melampaui kapasitas dan pola operasional PostgreSQL. Pemilihan backend harus berdasarkan beban dan kebutuhan nyata, bukan sekadar asumsi.

---

# 3. Bab 7 – Monitoring

## Soal 11 – Arsitektur dan exposure stack

**Demonstrasi**

```bash
docker compose ps
ss -lntp
cat compose.yaml
cat prometheus/prometheus.yml
```

**Penjelasan**

- Prometheus: scrape dan menyimpan metrik.
- Node Exporter: menyediakan metrik host.
- cAdvisor: menyediakan metrik container.
- Grafana: visualisasi dan dashboard.

Prometheus menjangkau exporter melalui DNS/network internal Docker, misalnya `node-exporter:9100` dan `cadvisor:8080`. Port host harus dilihat dari konfigurasi aktual; contoh umum lab adalah Prometheus 9090 dan Grafana 3000 yang dibatasi ke `127.0.0.1`.

**Pertanyaan lanjutan – Mengapa cAdvisor memakai privileged?**

cAdvisor memerlukan akses ke informasi cgroup dan resource container/host. Mode privileged memberikan hak sangat luas, sehingga harus dihindari pada mesin shared atau production yang tidak benar-benar dipercaya. Gunakan privilege minimum yang terbukti cukup.

## Soal 12 – Validasi rule sebelum dijalankan

**Demonstrasi**

```bash
docker compose run --rm --no-deps \
  --entrypoint /bin/promtool prometheus \
  check config /etc/prometheus/prometheus.yml
```

```bash
docker compose run --rm --no-deps \
  --entrypoint /bin/promtool prometheus \
  test rules /etc/prometheus/tests/rules.test.yml
```

**Penjelasan**

- `check config` memvalidasi sintaks/struktur konfigurasi Prometheus dan referensi konfigurasi yang dapat diperiksa.
- `test rules` mengevaluasi rule terhadap input sampel dan ekspektasi dalam file pengujian.

Keduanya tidak membuktikan target nyata dapat dijangkau, exporter sehat, atau notifikasi alert diterima.

**Pertanyaan lanjutan – Jika eval_time diubah menjadi 1m?**

Hasil bergantung pada sampel dan ekspektasi test. Alert dengan `for: 1m` baru firing setelah kondisinya terus terpenuhi selama durasi tersebut; pada batas waktu evaluasi, perilaku dapat dipengaruhi interval evaluasi dan waktu sampel. Jalankan test dan jelaskan output aktual, jangan menebak.

## Soal 13 – Status target dan makna up

**Demonstrasi**

```bash
curl -fsS http://127.0.0.1:9090/api/v1/targets
./scripts/verify-stack.sh
```

**Penjelasan**

`up = 1` berarti scrape target berhasil pada scrape tersebut. `up = 0` berarti scrape gagal. UP tidak menjamin fungsi bisnis sehat.

**Pertanyaan lanjutan – Semua target UP tetapi pengguna terdampak?**

Contoh: endpoint metrics masih dapat di-scrape, tetapi proses pendaftaran lambat atau gagal karena dependency. Periksa latency, error rate, queue depth, saturation, dan metrik bisnis.

## Soal 14 – Counter dan rate()

**Demonstrasi di Prometheus Graph**

```promql
node_cpu_seconds_total
```

Lalu:

```promql
rate(node_cpu_seconds_total[5m])
```

**Penjelasan**

`node_cpu_seconds_total` adalah counter kumulatif waktu CPU. `rate(...[5m])` menghitung rata-rata laju perubahan per detik selama rentang lima menit, dengan penanganan reset counter oleh Prometheus.

**Pertanyaan lanjutan – Mengapa rate kosong sesaat setelah stack hidup?**

Prometheus belum memiliki cukup sampel dalam rentang yang diminta. Tunggu beberapa scrape interval atau gunakan rentang yang sesuai; jangan menyimpulkan exporter rusak hanya dari query yang belum memiliki cukup data.

## Soal 15 – Recording rule CPU host

**Demonstrasi**

Tampilkan file rule:

```bash
cat prometheus/rules/recording-rules.yml
```

Query manual (sesuaikan label dan definisi rule):

```promql
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
```

Query recording rule (jika rule menyimpan rasio 0–1):

```promql
instance:node_cpu_utilisation:ratio_rate5m * 100
```

**Penjelasan**

Recording rule mengevaluasi ekspresi PromQL secara berkala dan menyimpan hasilnya sebagai time series baru, sehingga query dashboard dapat lebih sederhana/efisien. Pastikan satuan rule: rasio 0–1 atau persentase 0–100.

**Pertanyaan lanjutan – Mengapa hasil manual dan rule sedikit berbeda?**

Perbedaan dapat berasal dari waktu evaluasi, scrape interval, range window, label grouping, jumlah CPU/instance, atau definisi ekspresi yang tidak identik.

## Soal 16 – Load test dan denominator CPU

**Demonstrasi**

```bash
docker compose --profile load up -d load-generator
docker compose ps
```

Amati metrik CPU container dan host selama sekitar dua menit, lalu hentikan generator:

```bash
docker compose --profile load stop load-generator
```

**Penjelasan**

CPU container mengukur pemakaian oleh container tertentu, sedangkan CPU host menggambarkan pemakaian keseluruhan host/VM. Perhatikan satuan, normalisasi jumlah core, dan definisi metric yang dibandingkan.

**Pertanyaan lanjutan – Bagaimana membuktikan generator penyebab kenaikan CPU?**

Ambil baseline, jalankan generator, amati CPU container generator dan host, lalu hentikan generator dan amati pemulihan. Bandingkan dengan proses lain dan gunakan beberapa pengukuran; korelasi waktu mendukung analisis tetapi tidak otomatis membuktikan kausalitas tunggal.

## Soal 17 – Negative test TargetDown

**Demonstrasi**

Sebelum menghentikan exporter:

```bash
curl -G -fsS http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=ALERTS{alertname="TargetDown"}'
```

Hentikan Node Exporter dan tunggu melewati durasi `for` pada rule (soal meminta 75 detik):

```bash
docker compose stop node-exporter
sleep 75
```

Periksa lagi:

```bash
curl -G -fsS http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=ALERTS{alertname="TargetDown"}'
```

Pulihkan:

```bash
docker compose start node-exporter
```

**Penjelasan**

Ketika scrape gagal, kondisi alert menjadi pending. Setelah kondisi terus terpenuhi selama durasi `for`, alert menjadi firing. Setelah target pulih dan kondisi tidak lagi terpenuhi, alert akan resolved sesuai evaluasi berikutnya.

**Pertanyaan lanjutan – Apa yang belum dibuktikan?**

Negative test ini belum membuktikan Alertmanager mengirim notifikasi, kanal notifikasi berfungsi, operator menerima pesan, atau prosedur incident response dijalankan.

## Soal 18 – Provisioning dan akses Grafana

**Demonstrasi**

```bash
cat grafana/provisioning/datasources/datasources.yml
cat grafana/provisioning/dashboards/provider.yml
```

Coba edit panel melalui UI dan amati apakah perubahan dapat disimpan.

**Penjelasan**

Provisioning mengelola datasource/dashboard secara deklaratif dari file. Pengaturan seperti `editable` atau `allowUiUpdates` memengaruhi apakah perubahan melalui UI diizinkan/disimpan. Akses Grafana perlu dibatasi dengan akun, role, password kuat, dan exposure jaringan yang sesuai.

**Pertanyaan lanjutan – No data di Grafana tetapi query berhasil di Prometheus?**

Urutan diagnosis:
1. Pastikan datasource yang dipilih benar.
2. Periksa time range dan timezone dashboard.
3. Jalankan query pada Explore.
4. Periksa nama metric dan label.
5. Periksa template variables dan panel transformations.
6. Periksa konektivitas Grafana ke Prometheus dan log Grafana.

## Soal 19 – Cardinality dan storage TSDB

**Demonstrasi**

```bash
curl -fsS http://127.0.0.1:9090/api/v1/status/tsdb
```

**Penjelasan**

Cardinality adalah jumlah time series unik. Kombinasi label yang berbeda menghasilkan series berbeda. Label dengan nilai unik/berubah sangat sering dapat meningkatkan cardinality dan penggunaan memori/storage.

Label yang umumnya stabil dan berguna: `job`, `instance`, `service`, `environment`. Label berisiko tinggi: container ID, request ID, user ID, URL dinamis, dan nilai unik lain. Pilih berdasarkan kebutuhan observability dan metric aktual di lab.

## Soal 20 – Integrasi log Bab 6 ke Grafana

**Demonstrasi**

```bash
cd ~/docker-lab/bab-7
cat scripts/connect-bab6.sh
./scripts/connect-bab6.sh
```

Di Grafana Explore pilih datasource PostgreSQL log dan gunakan query yang sesuai:

```sql
SELECT
    time AS "time",
    tag,
    data
FROM logs.fluentbit
WHERE $__timeFilter(time)
ORDER BY time;
```

**Penjelasan**

Script integrasi pada lab dapat menyiapkan koneksi network dan/atau role read-only serta konfigurasi datasource. Buka isi script dan jelaskan tindakan yang benar-benar dilakukannya; jangan mengasumsikan script melakukan langkah yang tidak ada di dalamnya.

Gunakan rentang waktu yang sama untuk membandingkan log dengan kenaikan CPU. Korelasi dapat membantu mengidentifikasi event yang terjadi bersamaan, tetapi tidak sendirinya membuktikan penyebab.

**Pertanyaan lanjutan – Network Bab 7 belum tersedia?**

Network Docker yang dibutuhkan belum ada atau nama yang dirujuk script tidak cocok. Periksa:

```bash
docker compose ps
docker network ls
docker compose config
```

Jalankan stack yang membuat network tersebut, verifikasi nama network aktual, lalu jalankan kembali script.

---

# 4. Studi Kasus Portal PMB

## Prinsip investigasi

Jangan menyimpulkan kesehatan sistem hanya dari status container. Hubungkan centralized log dengan database transaksi menggunakan `request_id` yang sama. Catat rentang waktu, urutan event, service sumber, dan status transaksi. Gunakan nama tabel/kolom aktual setelah memeriksa schema.

Komponen skenario:
- `registration-api`: menerima pendaftaran.
- `auth-service`: autentikasi.
- `verification-worker`: proses verifikasi.
- `app-db`: database transaksi.
- `log-db`: database log.
- `fluent-bit`: collector log.

## Soal 21 – Tahap A: Baseline transaksi normal

**Demonstrasi**

```bash
cd ~/docker-lab/bab-6-pmb/pmb-devsecops-lab
sed -n '1,260p' scripts/demo-normal.sh
./scripts/demo-normal.sh
```

Catat request_id dan waktu pengujian dari output aktual. Cari kronologi log:

```bash
docker compose logs --since=15m --no-color \
  registration-api auth-service verification-worker
```

Filter menggunakan request ID aktual:

```bash
docker compose logs --since=15m --no-color \
  registration-api auth-service verification-worker \
  | grep 'REQUEST_ID_AKTUAL'
```

Periksa schema transaksi:

```bash
docker compose exec app-db sh
# Masuk psql menggunakan kredensial/environment lab
```

Di psql:

```sql
\dt
\d+ NAMA_TABEL_TRANSAKSI

SELECT *
FROM NAMA_TABEL_TRANSAKSI
WHERE request_id = 'REQUEST_ID_AKTUAL';
```

**Penjelasan**

Baseline berhasil jika kronologi menunjukkan alur normal dan transaksi dengan request ID yang sama berakhir VERIFIED pada database transaksi.

**Pertanyaan lanjutan – Event time, collector receive time, database time?**

- Event time: waktu kejadian menurut aplikasi.
- Collector receive time: waktu collector menerima record.
- Database insert time: waktu record disimpan ke database.

Gunakan event time yang tervalidasi untuk kronologi bisnis, lalu bandingkan dengan waktu penerimaan dan penyimpanan untuk mengukur keterlambatan. Perhatikan sinkronisasi jam dan timezone.

## Soal 22 – Tahap B: Failure injection mode timeout

**Demonstrasi**

Baca script terlebih dahulu agar memahami cara mode insiden diaktifkan:

```bash
sed -n '1,260p' scripts/set-incident.sh
sed -n '1,260p' scripts/demo-incident.sh
./scripts/demo-incident.sh
```

Cari event terkait:

```bash
docker compose logs --since=15m --no-color \
  registration-api auth-service verification-worker \
  | grep -Ei 'WARN|ERROR|verification_started|verification_retry|dependency_timeout'
```

**Penjelasan**

Gunakan request ID insiden. Urutkan event berdasarkan timestamp, identifikasi service pertama yang mengeluarkan WARN/ERROR, kemudian rekonstruksi `verification_started`, `verification_retry`, dan `dependency_timeout`. Buktikan dampak akhirnya dengan membaca record transaksi di app-db.

**Pertanyaan lanjutan – Mengapa container Up tidak membuktikan fungsi bisnis sehat?**

Status Up hanya menunjukkan proses container berjalan. Aplikasi masih dapat gagal memproses request, timeout ke dependency, mengalami antrean menumpuk, atau gagal menyelesaikan transaksi.

## Soal 23 – Tahap C: Recovery dan transaksi gagal

**Demonstrasi**

Kembalikan mode ke `none` menggunakan mekanisme yang diperlihatkan oleh `scripts/set-incident.sh`. Setelah itu kirim transaksi baru:

```bash
./scripts/demo-normal.sh
```

Periksa transaksi insiden dan transaksi baru secara terpisah menggunakan request ID masing-masing.

**Penjelasan**

Transaksi baru dapat menjadi VERIFIED setelah dependency pulih. Transaksi insiden tetap FAILED jika aplikasi tidak memiliki mekanisme pemulihan otomatis yang mengubah/memproses ulang transaksi lama. Pulihnya layanan tidak otomatis membatalkan riwayat kegagalan transaksi.

**Pertanyaan lanjutan – Retry terkontrol atau dead-letter queue**

Gunakan exponential backoff, batas maksimum percobaan, klasifikasi error retryable/non-retryable, idempotency key, audit retry, dan dead-letter queue untuk pesan yang tetap gagal. Retry harus dapat diamati dan dioperasikan secara terkontrol.

Risiko retry tanpa idempotensi: transaksi atau efek samping dapat dilakukan lebih dari sekali, misalnya pembuatan data ganda atau pemanggilan dependency berulang.

## Soal 24 – Tahap D: Resilience logging saat log-db mati

**Demonstrasi**

```bash
docker compose stop log-db
```

Buat transaksi baru saat log-db berhenti, lalu amati retry/buffer:

```bash
docker compose logs --since=10m --no-color fluent-bit
```

Pulihkan database:

```bash
docker compose start log-db
docker compose logs --since=10m --no-color fluent-bit
```

Verifikasi backlog pada tabel log menggunakan request ID transaksi yang dibuat saat backend mati. Bandingkan jumlah event sebelum dan sesudah recovery.

**Penjelasan**

Saat log-db tidak tersedia, output gagal dan Fluent Bit melakukan retry. Data dapat tertahan pada buffer memori/filesystem sesuai konfigurasi. Setelah database pulih, backlog dikirim. Buktikan dengan record yang sebelumnya belum tersimpan kemudian muncul setelah recovery.

**Pertanyaan lanjutan – Apa jika buffer mencapai 256 MB?**

Buffer memiliki batas; ketika penuh, sistem dapat mengalami backpressure dan record baru dapat tertahan, ditolak, atau hilang bergantung konfigurasi input/storage dan kebijakan overflow. Monitor kapasitas disk, backlog, retry, output error, dan buffer limit. Jangan menyatakan data pasti aman setelah batas tercapai.

Untuk mendeteksi duplikasi, gunakan event ID unik dan query duplikasi:

```sql
SELECT data->>'event_id' AS event_id, COUNT(*)
FROM logs.fluentbit
GROUP BY data->>'event_id'
HAVING COUNT(*) > 1;
```

Sesuaikan tabel dan field dengan schema aktual.

## Soal 25 – Privasi, kepercayaan, dan kesiapan produksi

**Demonstrasi**

Pemeriksaan pola secret (sesuaikan schema/tabel):

```sql
SELECT COUNT(*)
FROM nama_schema.nama_tabel_log
WHERE data::text ~* '(password|token|secret|api[_-]?key)';
```

Pemeriksaan pola email:

```sql
SELECT COUNT(*)
FROM nama_schema.nama_tabel_log
WHERE data::text ILIKE '%@%';
```

Hitung checksum file konfigurasi yang benar-benar dipakai:

```bash
sha256sum compose.yaml fluent-bit/fluent-bit.conf \
  scripts/demo-normal.sh scripts/demo-incident.sh
```

Jika path berbeda, gunakan path aktual. Catat checksum, waktu pengujian, branch/commit konfigurasi, dan versi image.

**Penjelasan**

Nama dan email tidak dimasukkan ke structured log karena tidak diperlukan untuk observability dan dapat meningkatkan risiko paparan PII. Gunakan data minimum dan redaction. Request ID dari client harus divalidasi format/panjangnya, dibatasi karakter yang diperbolehkan, dan tidak boleh dianggap sebagai identitas atau bukti otorisasi. Server dapat membuat correlation ID sendiri atau mencatat relasi yang tervalidasi.

**Pertanyaan lanjutan – Kontrol sebelum produksi**

1. TLS/mTLS untuk komunikasi antarlayanan.
2. Secret manager, rotasi kredensial, dan larangan hardcode secret.
3. Least privilege dan segmentasi network.
4. Redaction PII/secret pada seluruh jalur log.
5. Retensi, backup, enkripsi, dan kontrol akses data.
6. Monitoring buffer, disk, retry, dan dropped records.
7. Idempotency, retry policy, dan dead-letter queue.
8. Sinkronisasi waktu UTC.
9. Audit trail dan pemantauan akses.
10. Alerting, incident response, disaster recovery, serta uji pemulihan berkala.

---

# 5. Ringkasan istilah

| Istilah | Makna singkat |
|---|---|
| Fluent Bit | Collector dan processor log |
| Forward Protocol | Protokol pengiriman record ke Fluent Bit |
| JSONB | Tipe PostgreSQL untuk data JSON terstruktur |
| Buffering | Penahanan data sementara saat output bermasalah |
| Least privilege | Hak akses minimum sesuai kebutuhan |
| Redaction | Penyembunyian/penghapusan data sensitif |
| Retention | Kebijakan masa penyimpanan data |
| Prometheus | Sistem scrape dan penyimpanan metrik |
| Node Exporter | Exporter metrik host |
| cAdvisor | Exporter metrik container |
| Grafana | Visualisasi dan dashboard |
| Counter | Metric kumulatif |
| rate() | Laju perubahan counter per detik |
| Recording rule | Hasil ekspresi PromQL yang dievaluasi dan disimpan berkala |
| Alert rule | Aturan deteksi kondisi |
| Cardinality | Jumlah time series unik |
| request_id | ID korelasi request |
| Idempotency | Menjaga operasi berulang tidak menggandakan efek |
| Dead-letter queue | Penampung pesan yang gagal diproses |
| SHA-256 | Hash untuk pemeriksaan integritas file |

# 6. Strategi menjawab penguji

Gunakan pola empat langkah pada setiap soal:

1. **Demonstrasi:** jalankan perintah yang diminta.
2. **Observasi:** sebutkan output aktual yang terlihat.
3. **Analisis:** hubungkan konfigurasi, alur data, dan hasil.
4. **Kesimpulan:** nyatakan apakah pengujian berhasil serta batas pembuktiannya.

## Checklist sebelum ujian

- [ ] Stack Bab 6, Bab 7, dan Portal PMB dapat dijalankan.
- [ ] File konfigurasi dan script yang diminta dapat dibuka.
- [ ] Nama service, network, database, schema, tabel, dan kolom sudah diverifikasi.
- [ ] Kredensial tersedia tanpa perlu menampilkan secret.
- [ ] Perintah demonstrasi sudah dicoba di lab.
- [ ] Evidence berasal dari sesi pengujian sendiri.
- [ ] Dapat menjelaskan perbedaan status container, health, dan keberhasilan fungsi bisnis.
- [ ] Dapat menghubungkan event log dengan transaksi menggunakan request ID.
- [ ] Tidak menjalankan DELETE atau mengubah konfigurasi destruktif tanpa persetujuan.

**Catatan penting:** jangan mengarang jumlah record, status transaksi, metric, atau output terminal. Gunakan bukti aktual. Jika hasil berbeda dari ekspektasi, jelaskan perbedaannya dan lakukan diagnosis.
