# Panduan Langkah-demi-Langkah: Konfigurasi PostgreSQL Physical Replication di Ubuntu

Physical replication (replikasi fisik) di PostgreSQL memungkinkan Anda untuk membuat salinan persis dari server database utama (primary) ke server sekunder (standby). Panduan ini akan menjelaskan cara mengatur replikasi fisik PostgreSQL di Ubuntu dengan benar dan tanpa kesalahan.

## Prasyarat

- Dua server Ubuntu (misalnya Ubuntu 20.04/22.04)
- PostgreSQL versi 12 atau lebih baru diinstal pada kedua server
- Koneksi jaringan antara kedua server
- Akses root atau sudo pada kedua server
- Firewall yang memungkinkan koneksi pada port PostgreSQL (default: 5432)

## Langkah 1: Persiapan Server Primary

Pertama, kita perlu mengkonfigurasi server primary (master):

1. **Edit file postgresql.conf**:

```bash
sudo nano /etc/postgresql/[versi]/main/postgresql.conf
```

Ubah parameter berikut:

```
listen_addresses = '*'            # Mendengarkan pada semua antarmuka
wal_level = replica               # Minimal untuk replikasi fisik
max_wal_senders = 10              # Jumlah maksimum proses pengirim WAL
max_replication_slots = 10        # Jumlah maksimum slot replikasi
wal_keep_size = 1GB               # Ukuran WAL yang disimpan untuk replikasi
hot_standby = on                  # Memungkinkan koneksi read-only pada standby
```

2. **Edit file pg_hba.conf untuk mengizinkan replikasi**:

```bash
sudo nano /etc/postgresql/[versi]/main/pg_hba.conf
```

Tambahkan baris berikut (ganti dengan alamat IP server standby yang sebenarnya):

```
# Izinkan replikasi dari server standby
host    replication     replicator      [alamat_IP_standby]/32       md5
```

3. **Buat pengguna khusus untuk replikasi**:

```bash
sudo -u postgres psql
```

Jalankan SQL berikut:

```sql
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'password_yang_kuat';
```

4. **Restart PostgreSQL untuk menerapkan perubahan**:

```bash
sudo systemctl restart postgresql
```

## Langkah 2: Persiapan Server Standby

1. **Hentikan layanan PostgreSQL pada server standby**:

```bash
sudo systemctl stop postgresql
```

2. **Kosongkan direktori data PostgreSQL pada standby**:

```bash
sudo rm -rf /var/lib/postgresql/[versi]/main/*
```

## Langkah 3: Buat Base Backup dari Primary

Pada server primary, buat backup awal untuk digunakan pada standby:

```bash
sudo -u postgres pg_basebackup -h localhost -D /tmp/basebackup -U replicator -P -Xs -R
```

Parameter:
- `-h localhost`: Host untuk terhubung
- `-D /tmp/basebackup`: Direktori tempat backup akan disimpan
- `-U replicator`: Pengguna replikasi
- `-P`: Menampilkan kemajuan
- `-Xs`: Gunakan streaming untuk WAL selama backup
- `-R`: Tulis konfigurasi replikasi otomatis

## Langkah 4: Salin Backup ke Server Standby

1. **Transfer backup ke server standby**:

```bash
# Pada server primary
sudo tar -czf /tmp/pg_backup.tar.gz -C /tmp basebackup
scp /tmp/pg_backup.tar.gz [user]@[alamat_IP_standby]:/tmp/
```

2. **Ekstrak backup pada server standby**:

```bash
# Pada server standby
sudo tar -xzf /tmp/pg_backup.tar.gz -C /tmp
sudo cp -R /tmp/basebackup/* /var/lib/postgresql/[versi]/main/
sudo chown -R postgres:postgres /var/lib/postgresql/[versi]/main/
```

## Langkah 5: Konfigurasi Server Standby

1. **Buat atau verifikasi file recovery.conf** (pada PostgreSQL 12 atau lebih baru, parameter recovery ada di postgresql.conf dan standby.signal):

```bash
# Untuk PostgreSQL 12+ buat file standby.signal
sudo touch /var/lib/postgresql/[versi]/main/standby.signal
sudo chown postgres:postgres /var/lib/postgresql/[versi]/main/standby.signal
```

2. **Edit postgresql.conf pada standby**:

```bash
sudo nano /etc/postgresql/[versi]/main/postgresql.conf
```

Tambahkan atau ubah parameter berikut:

```
primary_conninfo = 'host=[alamat_IP_primary] port=5432 user=replicator password=password_yang_kuat'
hot_standby = on
```

## Langkah 6: Mulai PostgreSQL pada Server Standby

```bash
sudo systemctl start postgresql
```

## Langkah 7: Verifikasi Replikasi

1. **Periksa status replikasi pada server primary**:

```bash
sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"
```

2. **Periksa status replikasi pada server standby**:

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Hasilnya harus `t` (true) jika server berjalan sebagai standby.

```bash
sudo -u postgres psql -c "SELECT sender_host, status, slot_name FROM pg_stat_wal_receiver;"
```

## Pengujian Replikasi

1. **Buat tabel dan masukkan data pada server primary**:

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE test;
\c test
CREATE TABLE test_table (id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO test_table (name) VALUES ('Test data 1');
```

2. **Verifikasi data ada di server standby** (ingat standby bersifat read-only):

```bash
sudo -u postgres psql -d test -c "SELECT * FROM test_table;"
```

## Pemantauan dan Pemeliharaan

1. **Pantau lag replikasi**:

```bash
# Pada primary
sudo -u postgres psql -c "SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;"
```

2. **Periksa ukuran WAL yang tersisa**:

```bash
sudo -u postgres psql -c "SELECT * FROM pg_ls_waldir() ORDER BY name DESC LIMIT 10;"
```

## Pemecahan Masalah

Jika replikasi tidak berfungsi:

1. **Periksa log PostgreSQL**:

```bash
sudo tail -f /var/log/postgresql/postgresql-[versi]-main.log
```

2. **Verifikasi koneksi jaringan**:

```bash
ping [alamat_IP_primary_atau_standby]
telnet [alamat_IP_primary] 5432
```

3. **Periksa pengaturan firewall**:

```bash
sudo ufw status
```

4. **Verifikasi hak akses**:

```bash
ls -la /var/lib/postgresql/[versi]/main/
```

## Skenario Failover Manual

Untuk failover manual jika server primary mengalami kegagalan:

1. **Promosikan standby menjadi primary baru**:

```bash
# Pada server standby
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/[versi]/main/
```

2. **Verifikasi server sudah tidak dalam mode recovery**:

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Hasilnya harus `f` (false) jika promosi berhasil.

## Tips Keamanan

1. **Gunakan SSL untuk enkripsi replikasi**:
   - Buat sertifikat SSL
   - Tambahkan `sslmode=require` ke `primary_conninfo`
   - Aktifkan SSL di postgresql.conf

2. **Gunakan autentikasi berbasis sertifikat daripada password**

3. **Batasi akses replikasi hanya dari alamat IP yang diizinkan**

## Kesimpulan

Dengan mengikuti langkah-langkah di atas, Anda telah berhasil mengatur replikasi fisik PostgreSQL antara server primary dan standby. Konfigurasi ini memberikan redundansi data dan memungkinkan untuk strategi high availability dengan failover manual atau otomatis menggunakan tools tambahan seperti Patroni atau repmgr.

Replikasi fisik PostgreSQL sangat handal untuk kebutuhan backup real-time dan disaster recovery. Namun ingat bahwa ini mereplikasi seluruh cluster, tidak bisa dipilih per database atau per tabel. Untuk replikasi selektif, gunakan replikasi logis yang tersedia sejak PostgreSQL 10.
