# BLM4522 — Veritabanı Yönetim Sistemleri | Proje 3. Ödev

Proje-1 Video Linki: https://drive.google.com/file/d/1kCCvulZxg5XM4sEuOw6s4-nn5y6PGLQB/view?usp=drive_link
Proje-3 Video Linki: https://drive.google.com/file/d/1kpscRiMA794A_KSP9V_ccnpgjIomF8sU/view?usp=drive_link


---

## 📁 Proje Yapısı

```
BLM4522/
├── Proje1/          # Veritabanı Performans Optimizasyonu
│   ├── Veritabani_Optimizasyon_Rehberi.txt
│   ├── Proje1_Rapor_Taslagi.txt
│   └── Veritabani_Sonuc_Raporu.txt
├── Proje3/          # Veritabanı Güvenliği ve İzolasyon
│   ├── Guvenlik_Uygulama_Rehberi.txt
│   ├── Proje3_Rapor_Taslagi.txt
│   ├── Veritabani_Guvenlik_Sonuc_Raporu.txt
│   └── sqlinjectiontest.py
└── BLM4522-Rapor.pdf
```

---

## 🚀 Proje 1 — Veritabanı Performans Optimizasyonu ve İzleme

**Veri Kümesi:** NYC Taxi Trips (~1.5 Milyon Kayıt, 221 MB)  
**Veritabanı:** `nyc_taxi`

Bu projede devasa bir PostgreSQL veritabanı üzerinde performans analizi yapılmış, sorgu optimizasyonu sağlanmış, farklı indeks stratejilerinin (B-Tree vs BRIN) verimliliği ölçülmüş ve kapsamlı bir rol tabanlı erişim kontrolü (RBAC) politikası uygulanmıştır.

### 📊 1.1 İzleme (Monitoring) — pg_stat_statements

Performans darboğazlarını tespit etmek için `pg_stat_statements` eklentisi etkinleştirilmiş ve sistemi en çok yavaşlatan sorgular hedefli (targeted) optimizasyon yaklaşımıyla analiz edilmiştir.

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- En yavaş 5 sorgu
SELECT query, calls, total_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```

### 🧹 1.2 Disk Yönetimi — VACUUM ANALYZE

`UPDATE` ve `DELETE` işlemlerinin diskte biriktirdiği Dead Tuple (ölü hücreler) temizlenerek veritabanı tazelenmiştir.

```sql
-- Tablo boyutu kontrolü
SELECT pg_size_pretty(pg_total_relation_size('taxi_trips'));

-- Disk temizliği ve istatistik güncelleme
VACUUM ANALYZE taxi_trips;
```

### ⚡ 1.3 Sorgu Optimizasyonu — EXPLAIN ANALYZE & İndeksleme

Tarih ve yolcu sayısına göre filtreleme yapan analitik sorgu üç aşamada optimize edilmiştir:

| Aşama | Yöntem | Süre |
|---|---|---|
| Aşama 1 | İndekssiz (Sequential Scan) | ~184 ms |
| Aşama 2 | B-Tree Index | ~60–65 ms |
| Aşama 3 | BRIN Index | **~37 ms** |

```sql
-- Aşama 1: İndekssiz ölçüm (Sequential Scan)
SET max_parallel_workers_per_gather = 0;
EXPLAIN ANALYZE
SELECT passenger_count, AVG(trip_duration)
FROM taxi_trips
WHERE pickup_datetime >= '2016-01-01' AND pickup_datetime < '2016-03-01'
GROUP BY passenger_count;

-- Aşama 2: B-Tree indeks oluşturma
CREATE INDEX idx_taxi_pickup_datetime ON taxi_trips(pickup_datetime);

-- Aşama 3: BRIN indeks oluşturma (büyük veri için)
CREATE INDEX idx_brin_pickup ON taxi_trips USING BRIN(pickup_datetime);

-- Kullanılmayan indekslerin tespiti
SELECT relname AS tablename, indexrelname AS indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%pk%';
```

> **Neden BRIN?** Zaman damgası (timestamp) gibi doğal sıralı artan büyük veri kümelerinde BRIN, B-Tree'ye kıyasla çok daha az disk alanı kullanır ve bellek tüketimi düşüktür.

### 🔐 1.4 Rol Tabanlı Erişim Kontrolü (RBAC)

"En Az Ayrıcalık" ilkesiyle 3 farklı kullanıcı rolü tanımlanmıştır:

```sql
CREATE ROLE db_admin WITH LOGIN PASSWORD 'admin123' SUPERUSER;
CREATE ROLE data_analyst WITH LOGIN PASSWORD 'analyst123';
CREATE ROLE data_entry WITH LOGIN PASSWORD 'entry123';

GRANT SELECT ON taxi_trips TO data_analyst;
GRANT SELECT, INSERT ON taxi_trips TO data_entry;
```

| Rol | Yetki | Açıklama |
|---|---|---|
| `db_admin` | SUPERUSER | Tam yönetici erişimi |
| `data_analyst` | SELECT | Sadece okuma/raporlama |
| `data_entry` | SELECT, INSERT | Veri girişi (silme yasak) |

---

## 🛡️ Proje 3 — Veritabanı Güvenliği, İzolasyon ve Siber Savunma

**Veri Kümesi:** E-Ticaret (Müşteri & Fatura verisi, Müşteri ID: 17850)  
**Veritabanı:** `ecommerce`

Siber saldırılara, içeriden gelen tehditlere ve veri sızıntılarına karşı çok katmanlı bir güvenlik mimarisi kurulmuştur.

### 🔑 3.1 Kolon Bazlı Şifreleme — pgcrypto & Bcrypt

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

INSERT INTO musteriler (musteri_id, ulke, sifre_hash)
SELECT DISTINCT ON (CustomerID) CustomerID, Country,
       crypt('gizli_sifre_123', gen_salt('bf'))
FROM ham_veri WHERE CustomerID IS NOT NULL;
```

**Sonuç:** Veritabanı dosyaları çalınsa bile Bcrypt + Salt mekanizması sayesinde şifre kırılması imkânsızdır.

```
musteri_id |  ulke  | sifre_hash
-----------+--------+--------------------------------------------------------------
     17850 | Turkey | $2a$06$xAWN7MbRE7d6AFf7sdZ2cO...
```

### 👥 3.2 RBAC + Satır Düzeyi Güvenlik (RLS)

```sql
-- Rol tanımları
CREATE ROLE app_admin;
CREATE ROLE app_user;
GRANT ALL PRIVILEGES ON faturalar TO app_admin;
GRANT SELECT, INSERT ON faturalar TO app_user;

-- RLS: Müşteri yalnızca kendi faturasını görebilir
ALTER TABLE faturalar ENABLE ROW LEVEL SECURITY;
CREATE POLICY musteri_izolasyonu ON faturalar FOR SELECT TO app_user
USING (musteri_id::text = current_setting('app.current_customer_id', true));

-- RLS performans indeksi
CREATE INDEX idx_faturalar_musteri_id ON faturalar(musteri_id);
```

### 🌐 3.3 Ağ Güvenliği — pg_hba.conf

Beyaz liste (whitelist) IP kısıtlaması ile yetkisiz ağ bağlantıları engellenmiştir. Sunucu yalnızca tanımlı IP adreslerinden gelen bağlantıları kabul eder.

### 🎭 3.4 Veri Maskeleme (Data Masking)

```sql
-- Kredi kartı maskeleme
SELECT LEFT(kredi_karti, 4) || '****' FROM kartlar;
-- Çıktı: 4545****

-- Güvenli View (proxy)
CREATE VIEW guvenli_faturalar AS
SELECT fatura_no, miktar FROM faturalar;
```

### 🔔 3.5 Denetim İzleri (Audit Logs) — Trigger

```sql
CREATE TRIGGER audit_silme
BEFORE DELETE ON faturalar
FOR EACH ROW EXECUTE FUNCTION engelle_ve_logla();
```

`app_admin` dışındaki herhangi biri silme işlemi denerse sistem otomatik `RAISE EXCEPTION` fırlatır ve olayı `guvenlik_loglari` tablosuna kaydeder.

### 💉 3.6 SQL Injection Testi — Python

`sqlinjectiontest.py` ile üç senaryo test edilmiştir:

```
[TEST 1] Normal Kullanım       → Sadece ilgili fatura döner ✅
[TEST 2] SQL Injection Saldırısı → Güvensiz sistemde TÜM faturalar sızdı ⚠️
[TEST 3] Parametrik Sorgu       → Saldırı engellendi, "fatura bulunamadı" ✅
```

```python
# Güvensiz (string birleştirme) - SALDIRIYA AÇIK
sql = f"SELECT * FROM faturalar WHERE fatura_no = '{girdi}'"

# Güvenli (parametrik sorgu) - KORUNAN
cursor.execute("SELECT * FROM faturalar WHERE fatura_no = %s", (girdi,))
```

---

## 🧰 Kurulum & Çalıştırma

### Gereksinimler

- PostgreSQL 18 (Postgres.app)
- Python 3.x
- `psycopg2` kütüphanesi

```bash
pip install psycopg2-binary
```

### SQL Injection Testini Çalıştırma

```bash
cd Proje3
python3 sqlinjectiontest.py
```

---

## 📈 Genel Sonuç

| Konu | Yöntem | Sonuç |
|---|---|---|
| Sorgu Hızı | BRIN Index | %500 iyileşme (184ms → 37ms) |
| Disk Yönetimi | VACUUM ANALYZE | Dead Tuple temizliği |
| İzleme | pg_stat_statements | Hedefli optimizasyon |
| Erişim Kontrolü | RBAC (3 rol) | En az ayrıcalık ilkesi |
| Şifreleme | pgcrypto Bcrypt | Kırılamaz hash |
| İzolasyon | RLS + Trigger | Satır düzeyi güvenlik |
| Saldırı Koruması | Parametrik Sorgu | SQL Injection engeli |
