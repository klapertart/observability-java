# Observability dengan OpenTelemetry: Diagram Umum dan Ringkasan Diskusi

Dokumen ini merangkum gambaran umum arsitektur observability berbasis OpenTelemetry (OTel) beserta hasil diskusi: peran tiap komponen, alur data, konfigurasi, keputusan yang sudah diambil, dan hal yang masih terbuka.

Fokus saat ini (Phase 1): **trace request dan stack trace error, dilihat lewat Jaeger**. Metrics dan log adalah Phase 2.

---

# 1. Diagram umum

```text
+------------------------------------------------------------------+
| APLIKASI (Spring Boot + Micrometer/OTel)                         |
|                                                                  |
|   Service A ------> Service B ------> Service C                  |
|          (header traceparent diteruskan antar service)           |
+---------------------------------+--------------------------------+
                                  |
                                  |  OTLP  (trace, metrics, log)
                                  v
+------------------------------------------------------------------+
| OTEL COLLECTOR  (proses terpisah, bukan bagian dari Jaeger)      |
|                                                                  |
|   Receivers --> Processors --------------> Exporters             |
|   (OTLP)        (tail sampling,            (ke tiap backend)     |
|                  filter, batch)                                  |
+---------+----------------------+------------------------+--------+
          |                      |                        |
          | traces               | metrics                | logs
          v                      v                        v
  +---------------+      +---------------+        +---------------+
  |    Jaeger     |      |  Prometheus   |        |     Loki      |
  |  query + UI   |      | TSDB + UI     |        |  (tanpa UI)   |
  |   [Phase 1]   |      | sederhana     |        |               |
  +-------+-------+      |   [nanti]     |        |    [nanti]    |
          |              +-------+-------+        +-------+-------+
          v                      |                        |
  +---------------+              +-----------+------------+
  |  OpenSearch   |                          |
  | (storage      |                          v
  |  trace)       |                  +---------------+
  +---------------+                  |    Grafana    |
                                     |  (dashboard)  |
                                     |    [nanti]    |
                                     +---------------+
```

Cara membaca:

- Aplikasi tidak tahu backend mana yang dipakai. Aplikasi hanya mengirim OTLP ke Collector.
- Collector yang memutuskan data dikirim ke mana dan apa yang dibuang atau disaring (tail sampling untuk trace, filter severity untuk log).
- Backend bisa diganti tanpa mengubah kode aplikasi.

---

# 2. Peran tiap komponen

| Komponen | Peran | Menyimpan data? | UI |
|---|---|---|---|
| Aplikasi (Micrometer + OTel bridge) | Menghasilkan trace, metrics, log | Tidak | Tidak |
| OTel Collector | Menerima, memproses, meneruskan telemetry | Tidak (hanya buffer sementara) | Tidak |
| Jaeger | Menerima trace, menyediakan query dan UI trace | Lewat storage backend | Ya (Jaeger UI) |
| OpenSearch | Storage backend trace untuk Jaeger | Ya | OpenSearch Dashboards (opsional) |
| Prometheus | Menyimpan dan meng-query metrics | Ya (TSDB lokal, retensi default sekitar 15 hari) | Ada, sederhana (port 9090) |
| Loki | Menyimpan dan meng-query log | Ya (hanya label yang diindeks) | Tidak ada, memakai Grafana |
| Grafana | Dashboard untuk Prometheus, Loki, dan bila perlu Jaeger | Tidak | Ya |

## 2.1 OTel Collector dan Jaeger adalah dua hal berbeda

- Collector adalah pipeline. Jaeger adalah salah satu backend tujuan.
- Sejak Jaeger v2, binary Jaeger dibangun di atas framework OTel Collector, sehingga kode dan format konfigurasinya mirip. Namun di arsitektur kita keduanya tetap dua proses dengan tugas berbeda.
- Di Jaeger v1 ada komponen bernama `jaeger-collector` (menulis ke storage). Itu bukan OTel Collector. Jangan tertukar.

---

# 3. OTLP (OpenTelemetry Protocol)

OTLP adalah format dan aturan standar untuk mengirim telemetry dari satu pihak ke pihak lain. Aplikasi mengirim dengan OTLP, dan siapa pun yang mengerti OTLP (Collector, Jaeger v2, Loki, Prometheus, dan banyak vendor) bisa membacanya.

| Cara kirim | Port | Pembeda jenis data |
|---|---|---|
| OTLP/gRPC | 4317 | Tidak ada path |
| OTLP/HTTP | 4318 | Path per jenis data |

Path pada OTLP/HTTP:

| Sinyal | Path |
|---|---|
| Trace | `/v1/traces` |
| Metrics | `/v1/metrics` |
| Log | `/v1/logs` |

Yang dipakai di project ini adalah OTLP/HTTP.

---

# 4. Aplikasi ke Collector

## 4.1 Trace dan metrics

Spring Boot 4.x (dependency: `spring-boot-starter-opentelemetry`):

```properties
management.tracing.sampling.probability=1.0
management.opentelemetry.tracing.export.otlp.endpoint=http://otel-collector:4318/v1/traces
management.otlp.metrics.export.url=http://otel-collector:4318/v1/metrics
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  opentelemetry:
    tracing:
      export:
        otlp:
          endpoint: http://otel-collector:4318/v1/traces
  otlp:
    metrics:
      export:
        url: http://otel-collector:4318/v1/metrics
```

Spring Boot 3.x (dependency: `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp`, ditambah `micrometer-registry-otlp` bila metrics OTLP ingin aktif):

```properties
management.tracing.sampling.probability=1.0
management.otlp.tracing.endpoint=http://otel-collector:4318/v1/traces
management.otlp.metrics.export.url=http://otel-collector:4318/v1/metrics
```

Kalau nama property salah, exporter tidak jalan dan biasanya tanpa error yang jelas. Kalau trace tidak sampai ke Collector, cek nama property ini lebih dulu.

`management.tracing.sampling.probability` dibiarkan `1.0` selama tail sampling di Collector dipakai. Menurunkannya di aplikasi berarti kembali ke head-based sampling yang bisa melewatkan trace error.

## 4.2 Log (Phase 2)

Log punya endpoint sendiri, tapi berbeda dari trace dan metrics: **mengisi endpoint saja belum cukup**.

Property (Spring Boot 4.x):

```properties
management.opentelemetry.logging.export.otlp.endpoint=http://otel-collector:4318/v1/logs
```

Untuk Spring Boot 3.x, nama yang dijumpai di demo tim Spring adalah `management.otlp.logging.endpoint`. Belum diverifikasi di 3.5, jadi cek dokumentasi versi yang dipakai.

Tambahan yang wajib, karena appender log OTel bukan bagian dari Spring Boot:

1. Dependency `io.opentelemetry.instrumentation:opentelemetry-logback-appender-1.0` (status alpha, pin versinya dan uji ulang setiap naik versi).
2. Appender di `logback-spring.xml`:

```xml
<configuration>
  <include resource="org/springframework/boot/logging/logback/base.xml"/>

  <appender name="OTEL"
            class="io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender"/>

  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="OTEL"/>
  </root>
</configuration>
```

3. Pasang instance OpenTelemetry ke appender saat startup:

```java
@Component
public class InstallOpenTelemetryAppender implements InitializingBean {

    private final OpenTelemetry openTelemetry;

    public InstallOpenTelemetryAppender(OpenTelemetry openTelemetry) {
        this.openTelemetry = openTelemetry;
    }

    @Override
    public void afterPropertiesSet() {
        OpenTelemetryAppender.install(openTelemetry);
    }
}
```

Satu proyek contoh melaporkan bahwa di Spring Boot 4 log dibuang diam-diam bila `management.logging.export.otlp.enabled` tidak di-set `true`. Dokumentasi resmi tidak menyebutnya. Cek hanya bila log tidak muncul di Collector.

Alternatif tanpa dependency alpha di aplikasi: log tetap ke stdout, lalu dikumpulkan di level platform (misalnya Fluent Bit atau logging bawaan OpenShift). `traceId` harus tetap ada di baris log agar bisa dikorelasikan.

---

# 5. Collector ke backend

## 5.1 Trace ke Jaeger (push via OTLP)

```yaml
exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
```

Jalur ini tidak melibatkan Prometheus, dan endpoint-nya sama untuk Jaeger v1 maupun v2 selama Jaeger menerima OTLP di port 4317.

## 5.2 Metrics ke Prometheus (dua pola)

**Pola A: Collector mengirim (push) lewat OTLP.** Receiver OTLP di Prometheus nonaktif secara default. Aktifkan dengan flag `--web.enable-otlp-receiver`, dan metrics diterima di path `/api/v1/otlp/v1/metrics`. Endpoint ini tidak punya autentikasi bawaan, jadi batasi aksesnya lewat network policy atau firewall.

```yaml
exporters:
  otlphttp/prometheus:
    metrics_endpoint: http://prometheus:9090/api/v1/otlp/v1/metrics
```

Di `prometheus.yml`, disarankan mengatur `otlp.promote_resource_attributes` (agar `service.name` dan `service.instance.id` menjadi label) dan `out_of_order_time_window`. Nama metrics diterjemahkan (titik menjadi underscore), jadi namanya di Prometheus bisa berbeda dari di aplikasi.

**Pola B: Prometheus menarik (pull) dari Collector.**

```yaml
# otel-collector-config.yaml
exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: otel-collector
    static_configs:
      - targets: ['otel-collector:8889']
```

Pilih B kalau Prometheus sudah berjalan dengan pola scrape dan sulit menambah flag baru. Pilih A kalau ingin lebih sedikit bagian yang bergerak dan Prometheus-nya versi baru.

## 5.3 Log ke Loki (push via OTLP)

Loki menerima log OpenTelemetry secara native lewat HTTP, sehingga cukup exporter `otlphttp` biasa (bukan exporter khusus Loki):

```yaml
exporters:
  otlphttp/loki:
    endpoint: http://loki:3100/otlp
    tls:
      insecure: true      # hanya untuk lokal/internal tanpa TLS
```

Path `/otlp` ditulis di endpoint, dan Collector menambahkan `/v1/logs` sendiri.

Catatan sisi Loki:

- Data OTLP disimpan sebagai structured metadata. Config Loki harus mengizinkannya (`allow_structured_metadata: true`); di Loki 3.0 ke atas sudah aktif secara default.
- Atribut resource seperti nama service dijadikan label. Nilai yang unik atau sangat banyak (seperti `traceId`) tidak dijadikan label, tetapi bisa dicari dengan filter LogQL. Nama field persisnya belum diverifikasi, cek di Grafana Explore setelah log pertama masuk.
- Kalau Loki dilindungi basic auth, pakai extension `basicauth` di Collector.

Grafana tidak menerima log dari Collector. Grafana hanya membaca dari Loki, dan perlu dihubungkan lewat Connections dengan menambahkan Loki sebagai data source.

## 5.4 Gabungan pipeline

Wajib memakai image **`otel/opentelemetry-collector-contrib`**, karena `tail_sampling` hanya ada di distribusi contrib.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 50000
    policies:
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 1000
      - name: baseline-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10      # 5-10 di production

  filter/logs_severity:
    logs:
      log_record:
        - 'severity_number < SEVERITY_NUMBER_WARN'

  batch: {}

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  otlphttp/prometheus:
    metrics_endpoint: http://prometheus:9090/api/v1/otlp/v1/metrics
  otlphttp/loki:
    endpoint: http://loki:3100/otlp
    tls:
      insecure: true
  debug:
    verbosity: basic

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, batch]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/prometheus]     # Phase 1: [debug]
    logs:
      receivers: [otlp]
      processors: [filter/logs_severity, batch]
      exporters: [otlphttp/loki]           # Phase 1: [debug]
```

Selama Phase 1, pipeline `metrics` dan `logs` cukup diarahkan ke exporter `debug`. Pipeline itu tetap harus ada supaya kiriman dari aplikasi tidak ditolak dan tidak memunculkan error export di log aplikasi.

Dengan tail sampling `decision_wait: 10s`, trace baru muncul di Jaeger sekitar 10 detik setelah request selesai. Ini normal.

Catatan skala: kalau Collector gateway lebih dari satu replica, tail sampling butuh semua span dari satu `traceId` masuk ke replica yang sama. Perlu routing berbasis `traceId`, bukan round-robin biasa.

---

# 6. Keputusan yang sudah diambil

| Topik | Keputusan | Alasan singkat |
|---|---|---|
| Mekanisme instrumentasi | Micrometer Tracing + OTel bridge (di Spring Boot 4 dikemas dalam `spring-boot-starter-opentelemetry`) | Nyambung ke Actuator, konfigurasi lewat `management.*`, tanpa Java agent |
| Komunikasi antar service | `RestClient` yang di-inject dari Spring | Propagasi `traceparent` otomatis |
| Scope | Backend service saja | Web frontend, gateway, dan broker ditunda |
| Tujuan utama | Trace request dan stack trace error di Jaeger | Log pipeline dan dashboard metrics ditunda |
| Sampling | Tail-based di Collector, `errors-policy` wajib aktif | Menjamin semua trace error tersimpan |
| Storage trace | Jaeger dengan OpenSearch | Persisten untuk production, UI tetap Jaeger |
| Log | Loki (UI lewat Grafana) | Pilihan yang dipertahankan. OpenSearch tetap alternatif yang valid bila ingin satu storage |
| Level log | Root `INFO` di aplikasi, filter `WARN` ke atas di Collector | Console tetap informatif, storage tidak kebanjiran |
| Format konfigurasi | `application.yml` atau `application.properties` (pilih satu per service) | Keduanya didokumentasikan |
| Target Java | 21 | Record, virtual threads |

Perbandingan singkat log: Loki hanya mengindeks label sehingga storage lebih hemat, sedangkan OpenSearch mengindeks seluruh isi log sehingga pencarian full-text lebih kuat tapi storage lebih besar. Karena OpenSearch sudah dipakai untuk Jaeger, memakainya untuk log juga masuk akal (lewat exporter `opensearch` di distribusi contrib atau Data Prepper), tetapi status stabilitas dukungan log-nya perlu dicek dulu.

---

# 7. Koreksi dan peringatan

1. **Jaeger v1 sudah end-of-life** pada 31 Desember 2025. Image `jaegertracing/all-in-one:1.60` di dokumen `spring-boot-otel-sample-implementation.md` dan di bagian 12 dokumen `spring-boot-observability-open-telemetry.md` adalah versi lama itu, sehingga tidak cocok untuk production. Syarat "Jaeger 1.53 ke atas untuk OpenSearch" di dokumen yang sama juga sudah usang.
2. **Jaeger v2** memakai image `cr.jaegertracing.io/jaegertracing/jaeger`, dengan konfigurasi berbentuk YAML gaya OTel Collector (bukan env var `SPAN_STORAGE_TYPE`). OpenSearch tetap didukung. Format config v2 untuk OpenSearch belum diverifikasi, jadi belum dituliskan di dokumen mana pun.
3. **Appender log OTel berstatus alpha.** Pin versi dan uji ulang setiap upgrade.
4. **Prometheus OTLP receiver tanpa autentikasi.** Aktifkan hanya bila akses bisa dikontrol.
5. **`logging.pattern.level` tidak perlu.** Spring Boot 3.2 ke atas otomatis menyisipkan `traceId` dan `spanId` ke log. Memasangnya manual membuat ID muncul dua kali.
6. **Collector harus varian contrib** untuk `tail_sampling`.
7. **Exception yang ditangani `@ExceptionHandler`** tidak otomatis menandai span server sebagai error. Untuk kegagalan sistem, panggil `ServerHttpObservationFilter.findObservationContext(request)...setError(ex)` di handler.

---

# 8. Hal yang masih terbuka

| Hal | Yang dibutuhkan |
|---|---|
| Retensi trace di OpenSearch | Keputusan lama retensi (14 atau 30 hari) dan konfirmasi plugin ISM aktif di cluster |
| Deploy Jaeger v2 | Verifikasi format config v2 dengan OpenSearch, lalu perbarui `docker-compose.yml` sample dan bagian 12 dokumen observability |
| Metrics ke Prometheus | Pilih pola A (push OTLP) atau pola B (pull dari Collector) |
| Pengiriman log | Pilih appender OTel di aplikasi atau kumpulkan dari stdout di level platform |
| Topologi Collector di cluster | Agent per node dan gateway terpusat, termasuk routing berbasis `traceId` untuk tail sampling |
| Akses Jaeger UI | Route OpenShift atau port-forward, dengan pembatasan akses karena Jaeger UI tidak punya autentikasi bawaan |

---

# 9. Dokumen terkait

| Dokumen | Isi |
|---|---|
| `spring-boot-observability-open-telemetry.md` | Standar observability: konsep, keputusan, sampling, roadmap, akses Jaeger |
| `spring-boot-otel-sample-implementation.md` | Project contoh Java 21: order, payment, stock service, lengkap dengan skenario uji |
| `observability-otel-overview.md` (dokumen ini) | Diagram umum dan ringkasan diskusi |

---

# 10. Glosarium singkat

| Istilah | Arti |
|---|---|
| Trace | Jejak satu request yang melewati beberapa service |
| Span | Satu langkah di dalam trace |
| `traceId` | ID yang sama untuk semua span dan log dari satu request |
| OTLP | OpenTelemetry Protocol, format standar pengiriman telemetry |
| Collector | Pipeline penerima, pemroses, dan pengirim telemetry |
| Tail sampling | Keputusan simpan atau buang trace diambil setelah trace lengkap, sehingga error dan trace lambat bisa selalu disimpan |
| Head sampling | Keputusan simpan atau buang diambil di awal request, sebelum hasilnya diketahui |
| TSDB | Time-series database, tipe database yang dipakai Prometheus |
| LogQL / PromQL | Bahasa query untuk Loki / Prometheus |
