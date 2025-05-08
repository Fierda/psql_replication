# PostgreSQL Physical Replication Guide

## 1. Pengenalan Physical Replication

Physical Replication di PostgreSQL adalah metode replikasi berbasis block-level, di mana seluruh perubahan yang terjadi pada level disk dari database di Primary akan direplikasi secara real-time ke Standby. Ini berbeda dengan Logical Replication yang berbasis SQL statement.

---

## 2. Prasyarat

1. PostgreSQL versi yang sama di Primary dan Standby Server.
2. Koneksi jaringan yang stabil antara Primary dan Standby.
3. Akses SSH dari Primary ke Standby.
4. Pastikan konfigurasi `postgresql.conf` dan `pg_hba.conf` dapat diakses dan diubah.

---

## 3. Konfigurasi di Primary Server

### Ubah `postgresql.conf`

Tambahkan atau sesuaikan baris berikut:

```
wal_level = replica
max_wal_senders = 5
max_replication_slots = 5
hot_standby = on
```

### Ubah `pg_hba.conf`

Tambahkan akses untuk Standby:

```
# Allow replication connections from standby
host replication replicator 192.168.1.20/32 md5
```

> **Catatan:** Gantilah IP `192.168.1.20` dengan IP Standby Server.

### Buat User untuk Replication

```sql
CREATE USER replicator REPLICATION LOGIN ENCRYPTED PASSWORD 'strongpassword';
```

> **Catatan:** Gantilah `strongpassword` dengan password yang aman.

### Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

---

## 4. Konfigurasi di Standby Server

### Hentikan PostgreSQL

```bash
sudo systemctl stop postgresql
```

### Sinkronisasi Data dari Primary

```bash
pg_basebackup -h 192.168.1.10 -D /var/lib/pgsql/14/data -U replicator -P --wal-method=stream
```

> **Catatan:**
>
> * Gantilah `192.168.1.10` dengan IP Primary Server.
> * Pastikan direktori data PostgreSQL sudah kosong atau backup terlebih dahulu.

### Buat File Recovery

Buat file `recovery.conf` di dalam direktori data:

```
standby_mode = 'on'
primary_conninfo = 'host=192.168.1.10 port=5432 user=replicator password=strongpassword'
trigger_file = '/tmp/postgresql.trigger'
```

---

## 5. Menjalankan Standby Server

```bash
sudo systemctl start postgresql
```

---

## 6. Verifikasi Replication

Di Primary Server:

```sql
SELECT * FROM pg_stat_replication;
```

Di Standby Server:

```sql
SELECT pg_is_in_recovery();
```

Jika hasilnya `t`, maka Standby dalam mode recovery dan replikasi sudah berjalan.

---

## 7. Troubleshooting Umum

* **Permasalahan Koneksi:**

  * Cek konfigurasi `pg_hba.conf` di Primary.
  * Pastikan IP dan user sudah benar.

* **WAL Files tidak tersinkron:**

  * Cek space di disk Primary dan Standby.
  * Pastikan parameter `wal_keep_segments` cukup besar.

* **Standby tidak sinkron:**

  * Jalankan ulang `pg_basebackup` jika perlu sinkronisasi ulang.

---

## 8. Menghentikan dan Melanjutkan Replication

### Menghentikan Replication

Pada Standby Server:

```bash
sudo systemctl stop postgresql
```

Jika Anda ingin menghentikan hanya replikasinya tanpa menghentikan PostgreSQL, Anda bisa:

```sql
SELECT pg_terminate_backend(pid) FROM pg_stat_replication;
```

### Melanjutkan Replication Setelah Terputus

1. Pastikan Primary Server masih menghasilkan WAL files yang cukup.
2. Di Standby Server, pastikan PostgreSQL dalam keadaan berhenti:

```bash
sudo systemctl stop postgresql
```

3. Sinkronisasi ulang jika perlu:

```bash
pg_basebackup -h 192.168.1.10 -D /var/lib/pgsql/14/data -U replicator -P --wal-method=stream
```

4. Mulai kembali PostgreSQL di Standby:

```bash
sudo systemctl start postgresql
```

5. Verifikasi:

```sql
SELECT * FROM pg_stat_replication;
```

Jika status `streaming` muncul kembali, replikasi sudah tersambung kembali.

---

## 9. Manual Resynchronization After Long Downtime

Jika replikasi terputus lebih dari beberapa hari dan WAL files sudah tidak tersedia, langkah berikut dapat dilakukan:

### 1️⃣ Cek Ketersediaan WAL Files di Primary

```bash
ls -l /var/lib/pgsql/14/data/pg_wal/
```

Jika file WAL yang dibutuhkan tidak ada, Anda harus melakukan sinkronisasi manual.

### 2️⃣ Sinkronisasi Manual dengan `pg_basebackup`

```bash
pg_basebackup -h 192.168.1.10 -D /var/lib/pgsql/14/data -U replicator -P --wal-method=stream
```

### 3️⃣ Alternatif Cepat: Menggunakan `rsync`

```bash
rsync -av --progress /var/lib/pgsql/14/data/ replicator@192.168.1.20:/var/lib/pgsql/14/data/
```

Pastikan PostgreSQL di Standby dalam kondisi mati.

### 4️⃣ Start PostgreSQL di Standby

```bash
sudo systemctl start postgresql
```

### 5️⃣ Verifikasi

```sql
SELECT * FROM pg_stat_replication;
```

Jika status `streaming` sudah aktif kembali, sinkronisasi manual berhasil dilakukan.

---

