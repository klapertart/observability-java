# Observability Standard untuk Spring Boot Microservice

## Tujuan

Dokumen ini menjadi dasar diskusi dan implementasi observability untuk microservice berbasis Spring Boot.

Untuk tahap awal, teknologi observability yang dipilih adalah **OpenTelemetry (OTel)**.

Scope implementasi tahap awal: **backend service** (Spring Boot microservice yang saling berkomunikasi lewat HTTP/RestClient). Komponen di luar itu (API Gateway, message broker, web frontend, multi-site topology) didokumentasikan sebagai catatan di bagian akhir, belum masuk scope implementasi.

---

# 1. Apa itu Observability?

Observability adalah kemampuan untuk memahami kondisi internal sebuah aplikasi berdasarkan data yang dihasilkan aplikasi tersebut.

Untuk microservice, kita ingin bisa menjawab:

- Apakah service sedang sehat?
- Request mana yang gagal?
- Mengapa request lambat?
- Service mana yang menjadi bottleneck?
- Apakah ada masalah pada database?
- Apa yang terjadi pada satu request ketika melewati beberapa service?

Secara sederhana:

```text
                    Observability
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
        Logs          Metrics         Traces
```

Tiga data utama tersebut saling melengkapi.

---

# 2. Data Apa yang Di-observe?

## 2.1 Logs

Logs menjawab:

> Apa yang terjadi?

Log sebaiknya memiliki informasi seperti:

```text
timestamp
level
service
environment
traceId
spanId
message
```

Untuk production, gunakan **structured log** (JSON) agar mudah dicari dan dianalisis.

Contoh:

```json
{
  "timestamp": "2026-09-27T10:00:00Z",
  "level": "ERROR",
  "service": "order-service",
  "traceId": "abc123",
  "spanId": "def456",
  "message": "Failed to call payment-service"
}
```

Jangan memasukkan informasi sensitif seperti password, token, secret, atau data pribadi yang tidak diperlukan.

**Strategi level log (disepakati):**

- Root level di aplikasi: `INFO` — supaya `kubectl logs` / log file lokal tetap informatif saat live-debug manual.
- Filter di OTel Collector sebelum masuk backend observability jangka panjang (Loki/Elastic): buang di bawah `WARN`, supaya storage tidak kebanjiran log INFO yang jarang dibuka lagi.
- Trade-off yang disadari: saat investigasi error, log INFO/DEBUG di sekitar kejadian tidak ikut tersimpan jangka panjang. Untuk itu, trace (lihat bagian 2.3) jadi andalan utama detail per-request, bukan log level rendah.

Contoh konfigurasi filter di Collector:

```yaml
processors:
  filter/logs_severity:
    logs:
      log_record:
        - 'severity_number < SEVERITY_NUMBER_WARN'

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [filter/logs_severity, batch]
      exporters: [otlp/loki]
```

---

## 2.2 Metrics

Metrics menjawab:

> Seberapa banyak? Seberapa cepat? Seberapa sering?

Contoh metrics:

```text
request count
request rate
error rate
request latency
JVM memory
JVM GC
CPU
thread count
database connection pool
```

Contoh sederhana:

```text
HTTP request rate = 1,000 req/s
HTTP error rate   = 2%
p95 latency       = 250 ms
JVM heap          = 65%
```

Metrics cocok untuk melihat kondisi service secara terus-menerus dan membuat alert.

---

## 2.3 Traces

Trace menjawab:

> Request ini melewati service mana saja dan bagian mana yang lambat?

Untuk arsitektur backend kita (lihat bagian 5), contoh flow:

```text
Client / Upstream
   |
   v
Service A (entry point)
   |
   +----> PostgreSQL (Service A)
   |
   +----> Service B (via RestClient)
              |
              +----> PostgreSQL (Service B)
```

Satu request memiliki satu `traceId`. Di dalam trace ada beberapa `span`:

```text
Trace
 |
 +-- Span: Service A (HTTP request masuk)
 |      |
 |      +-- Span: PostgreSQL (Service A)
 |
 +-- Span: Service B (dipanggil via RestClient)
        |
        +-- Span: PostgreSQL (Service B)
```

Dengan ini kita tahu bagian mana yang menyebabkan latency.

---

# 3. Hubungan Logs, Metrics, dan Traces

Ketiganya jangan dipandang sebagai sistem yang terpisah.

```text
                    Request
                       |
          +------------+------------+
          |            |            |
          v            v            v
        Logs        Metrics       Traces
          |            |            |
          +------------+------------+
                       |
                  Investigation
```

Contoh incident:

```text
API lambat
   |
   v
Metrics menunjukkan p95 latency naik
   |
   v
Trace menunjukkan span "Service B" lambat
   |
   v
Log menunjukkan koneksi database timeout di Service B
```

Jadi:

```text
Metrics -> menemukan masalah
Trace   -> menemukan lokasi masalah
Log     -> melihat detail kejadian (level WARN/ERROR)
```

---

# 4. Kenapa Memilih OpenTelemetry?

OpenTelemetry adalah standard/tooling untuk menghasilkan, mengumpulkan, dan mengirim telemetry (logs, metrics, traces), tanpa mengikat aplikasi ke satu vendor observability tertentu.

```text
Spring Boot
     |
     v
OpenTelemetry
     |
     +---- Logs
     |
     +---- Metrics
     |
     +---- Traces
```

---

# 5. Arsitektur Backend & Flow Observability

## 5.1 Gambaran umum sistem

Scope tahap awal: beberapa Spring Boot microservice yang saling memanggil lewat HTTP (`RestClient`), masing-masing punya database sendiri.

```text
                                +----------------------+
                                |   OTel Collector      |
                                +----------+-----------+
                                           |
                  +------------------------+------------------------+
                  |                        |                        |
                  v                        v                        v
              Traces                   Metrics                    Logs
                  |                        |                        |
                  v                        v                        v
          Jaeger / Tempo             Prometheus              Loki / Elastic
                  |                        |                        |
                  +------------------------+------------------------+
                                           |
                                      Grafana (visualisasi)
```

Aplikasi mengirim OTLP (traces, metrics, logs) ke OTel Collector, Collector meneruskan ke backend masing-masing.

## 5.2 Flow request antar service

```text
Client / Upstream
   |
   v
Service A  (entry point, misal: order-service)
   |  RestClient
   +--------------------------> Service B (misal: product-service)
   |                                  |
   |  RestClient                      +--> PostgreSQL
   +--------------------------> Service C (misal: payment-service)
                                       |
                                       +--> PostgreSQL
```

- Setiap panggilan antar service pakai `RestClient` (bean yang di-manage Spring) — trace context (`traceparent`) otomatis ikut di header HTTP, tidak perlu konfigurasi tambahan di luar setup dasar OTel.
- Setiap service yang menerima request tanpa header `traceparent` masuk (yaitu entry point pertama) otomatis membuat root span baru.

## 5.3 Di luar scope tahap ini (dicatat, belum diimplementasi)

- **API Gateway di depan (Kong)** — bisa jadi entry point trace paling awal kalau nanti diaktifkan (plugin `opentelemetry` bawaan Kong).
- **Message broker (ActiveMQ Artemis)** — kalau ada service yang komunikasi async lewat broker ini, trace context tidak otomatis ikut dan perlu propagation manual lewat message property. Berlaku juga untuk broker lain seperti Kafka bila dipakai di masa depan.
- **Web frontend (browser/React)** — trace baru mulai dari service backend pertama yang menerima request, bukan dari klik user di browser. Bisa ditambah OTel Web SDK belakangan kalau dibutuhkan investigasi masalah di sisi client.
- **Topologi multi-site** — kalau ada request yang bisa lintas site (bukan dua site yang independen), perlu dipastikan backend observability (Jaeger/Tempo) terpusat, bukan terpisah per site.

---

# 6. Keputusan Mekanisme Instrumentasi

Ada tiga pilihan cara instrumentasi Spring Boot dengan OTel:

| Opsi | Cara Kerja | Config | JDBC span otomatis? |
|---|---|---|---|
| 1. Java Agent | `-javaagent`, zero-code | env var (`OTEL_*`) | Ya |
| **2. Micrometer Tracing + OTel bridge (dipilih)** | Dependency + Observation API | `application.yml` (`management.*`) | Tidak, perlu tambahan |
| 3. OTel Spring Boot Starter murni | Dependency, OTel API langsung | `application.properties` (`otel.*`) | Tidak, perlu tambahan |

**Keputusan: Opsi 2 (Micrometer Tracing + OTel bridge).**

Alasan:
- Paling nyambung dengan Actuator yang sudah jadi kebiasaan tim.
- Config lewat `management.*` di `application.yml`, konsisten dengan cara existing metrics/health check dikonfigurasi.
- Tidak butuh Java agent terpisah yang harus di-attach saat startup.

Konsekuensi yang disadari:
- JDBC span (query ke database) **tidak otomatis** muncul — perlu tambahan `datasource-micrometer` (P6Spy-based) kalau span per query dibutuhkan. Kalau tidak, cukup andalkan metrics `HikariCP` (connection pool) untuk observability database.

---

# 7. Contoh Implementasi per Komponen

## 7.1 Setup dasar tiap Spring Boot service

**`pom.xml`:**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

**`application.yml`:**

```yaml
spring:
  application:
    name: order-service   # ganti sesuai nama masing-masing service

management:
  tracing:
    sampling:
      probability: 1.0   # head-based sampling di sisi aplikasi, lihat bagian 8
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
    metrics:
      export:
        url: http://otel-collector:4318/v1/metrics

logging:
  level:
    root: INFO
  pattern:
    level: "%5p [${spring.application.name},%X{traceId:-},%X{spanId:-}]"
```

> Catatan: nama property OTLP di atas (`management.otlp.tracing.endpoint`) mengikuti Spring Boot 3.x umum. Beberapa versi lebih baru memakai `management.opentelemetry.tracing.export.otlp.endpoint` — sesuaikan dengan versi Spring Boot yang dipakai.

## 7.2 Propagation antar service (RestClient)

Tidak perlu konfigurasi tambahan selama `RestClient` di-manage sebagai bean Spring:

```java
@Configuration
public class RestClientConfig {

    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder.build();   // auto-instrumented selama builder ini dari Spring context
    }
}
```

```java
@Service
public class OrderService {

    private final RestClient restClient;

    public OrderService(RestClient restClient) {
        this.restClient = restClient;
    }

    public ProductResponse getProduct(String productId) {
        return restClient.get()
            .uri("http://product-service/api/products/{id}", productId)
            .retrieve()
            .body(ProductResponse.class);
        // traceparent header otomatis ditambahkan, trace lanjut ke product-service
    }
}
```

**Yang perlu dihindari:** `RestClient.create()` yang dibuat manual di luar Spring context (bukan lewat `RestClient.Builder` yang di-inject) — instance seperti itu tidak kena auto-instrumentation, trace akan putus di panggilan itu.

## 7.3 JDBC (opsional, kalau span per query dibutuhkan)

```xml
<dependency>
  <groupId>net.ttddyy.observation</groupId>
  <artifactId>datasource-micrometer-spring-boot</artifactId>
</dependency>
```

Tanpa ini, query ke database tetap berjalan normal, hanya tidak muncul sebagai span terpisah di trace — cukup terlihat dari durasi span service secara keseluruhan, plus metrics `HikariCP` (connection pool) yang otomatis sudah tersedia dari Actuator.

## 7.4 Custom span manual (untuk business logic yang butuh detail tambahan)

```java
@Autowired
ObservationRegistry registry;

public void processOrder(Order order) {
    Observation.createNotStarted("process.order", registry)
        .lowCardinalityKeyValue("order.type", order.getType())
        .observe(() -> {
            // business logic di sini
        });
}
```

Gunakan ini hanya untuk logic yang butuh telemetry spesifik — jangan buat span manual untuk setiap method kecil (lihat bagian 9.3).

---

# 8. Strategi Sampling

Yang di-sample adalah **trace**, bukan log. Log diatur lewat level + filter (bagian 2.1), trace diatur lewat sampling.

## 8.1 Head-based sampling (di aplikasi)

Keputusan sample diambil di awal request, sebelum tahu hasilnya error atau tidak. Cocok untuk mulai cepat, tapi berisiko request yang justru bermasalah malah tidak ter-sample.

```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # 10% request di-trace penuh
```

## 8.2 Tail-based sampling (di OTel Collector) — direkomendasikan untuk production

Semua request tetap menghasilkan span di aplikasi, tapi keputusan simpan/buang diambil di Collector setelah trace lengkap terkumpul — sehingga request yang **error atau lambat selalu tersimpan**, sisanya (trace normal) disampling kecil.

```yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
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
          sampling_percentage: 5

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, batch]
      exporters: [otlp/jaeger]
```

**Catatan skala:** kalau Collector dideploy lebih dari satu replica, tail sampling butuh semua span dari satu `traceId` masuk ke replica yang sama — perlu load balancing berbasis `traceId`, bukan round-robin biasa.

**Rekomendasi tahap awal:** mulai dengan head-based `probability: 1.0` (semua di-trace) selama trafik masih kecil dan tim masih validasi setup, baru pindah ke tail-based sampling begitu volume mulai jadi pertimbangan biaya storage.

---

# 9. Prinsip Implementasi

## 9.1 Instrumentation otomatis dulu, manual seperlunya

Jangan tambah kode observability manual di setiap method kalau instrumentasi otomatis (HTTP request, RestClient, JDBC bila ditambah) sudah cukup. Manual instrumentation (bagian 7.4) dipakai untuk telemetry spesifik business logic saja.

## 9.2 Korelasi via traceId/spanId

Semua log yang berasal dari request yang sama akan punya `traceId` yang sama — inilah yang menghubungkan log, trace, dan (secara tidak langsung) metrics jadi satu investigasi.

## 9.3 Jangan berlebihan

Hindari:

```text
- log setiap detail object
- memasukkan payload besar ke log
- metrics dengan cardinality sangat tinggi
- membuat span untuk setiap method kecil
```

Tujuannya: telemetry yang cukup untuk **Detect → Investigate → Understand → Resolve**, bukan telemetry sebanyak mungkin.

---

# 10. Roadmap Implementasi

## Phase 1 — Basic (scope dokumen ini)

```text
Setiap Spring Boot service
    |
    +-- Micrometer Tracing + OTel bridge
          |
          +-- Traces (RestClient antar service otomatis ter-propagate)
          +-- Metrics (HTTP, JVM)
          +-- Logs (level INFO, filter WARN+ di Collector)
```

Target:
- Setiap service menghasilkan telemetry dan bisa export ke Collector.
- Setiap request antar service (via RestClient) punya satu trace yang utuh, tidak putus.
- Log bisa dikorelasikan dengan trace lewat `traceId`/`spanId`.
- Sampling head-based aktif sebagai baseline.

## Phase 2 — Refinement

```text
- Tail-based sampling di Collector
- JDBC span (kalau dibutuhkan)
- Database metrics (connection pool, query time)
```

## Phase 3 — Komponen di luar scope Phase 1

```text
- API Gateway (Kong) sebagai entry point trace
- Message broker (ActiveMQ / lainnya) — propagation manual
- Multi-site topology — backend observability terpusat
- Web frontend (opsional, kalau dibutuhkan investigasi sisi client)
```

## Phase 4 — Alerting & SLO

```text
Dashboard
Alert
SLO
Incident investigation workflow
```

---

# 11. Next Step

Setelah Phase 1 disepakati dan diimplementasikan di satu service percobaan, evaluasi:

1. Apakah trace antar service (via RestClient) benar-benar nyambung end-to-end (cek di Jaeger/Tempo).
2. Apakah volume trace dari sampling `probability: 1.0` masih wajar untuk storage, atau perlu langsung pindah ke tail-based sampling.
3. Baru lanjut rollout ke service lain, dan pertimbangkan Phase 3 sesuai prioritas.
