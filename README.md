# Observability Standard untuk Spring Boot Microservice

## Tujuan

Dokumen ini menjadi dasar diskusi dan implementasi observability untuk microservice berbasis Spring Boot.

Untuk tahap awal, teknologi observability yang dipilih adalah **OpenTelemetry (OTel)**.

Scope implementasi tahap awal: **backend service** (Spring Boot microservice yang saling berkomunikasi lewat HTTP/RestClient). Komponen di luar itu (API Gateway, message broker, web frontend, multi-site topology) didokumentasikan sebagai catatan di bagian akhir, belum masuk scope implementasi.

**Tujuan utama saat ini: trace request, khususnya saat terjadi error.** Yang ingin dicapai adalah kemampuan membuka satu request tertentu (via `traceId`) dan melihat di service mana request itu gagal, beserta stack trace-nya, langsung dari web Jaeger — tanpa perlu login ke server/pod satu per satu.

Log terstruktur ke backend terpisah (Loki/Elastic) dan dashboard metrics **belum jadi prioritas** — ditunda ke Phase 2/4 (lihat bagian 10). Exception yang terjadi tetap terekam otomatis di dalam span sebagai stack trace (bagian 7.5), jadi kebutuhan "lihat kenapa request gagal" sudah terjawab tanpa perlu log pipeline terpisah dulu.

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
              Jaeger                  Prometheus              Loki / Elastic
                  |
                  v
              OpenSearch
          (storage backend,
             production)
```

Aplikasi mengirim OTLP (traces, metrics, logs) ke OTel Collector, Collector meneruskan ke backend masing-masing.

> **Fokus Phase 1:** jalur **Traces → Jaeger → OpenSearch** yang aktif dipakai (storage backend production, lihat bagian 12). Jalur Metrics → Prometheus tetap jalan otomatis (tidak perlu setup tambahan) tapi belum dipakai. Jalur Logs → Loki/Elastic **belum diaktifkan** — lihat bagian "Tujuan" dan bagian 10.

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
| **2. Micrometer Tracing + OTel bridge (dipilih)** | Dependency + Observation API | `application.yml` atau `application.properties` (`management.*`) | Tidak, perlu tambahan |
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
  # traceId dan spanId otomatis muncul di log (Spring Boot 3.2+), tidak perlu logging.pattern.level
```

**`application.properties`** (setara dengan `application.yml` di atas, pilih salah satu format saja, jangan dua-duanya di service yang sama):

```properties
spring.application.name=order-service

# head-based sampling di sisi aplikasi, lihat bagian 8
management.tracing.sampling.probability=1.0

# export trace dan metrics ke OTel Collector via OTLP/HTTP
management.otlp.tracing.endpoint=http://otel-collector:4318/v1/traces
management.otlp.metrics.export.url=http://otel-collector:4318/v1/metrics

# log: level INFO. traceId dan spanId otomatis muncul di log (Spring Boot 3.2+),
# tidak perlu logging.pattern.level
logging.level.root=INFO
```

> Catatan versi: `management.otlp.tracing.endpoint` berlaku untuk Spring Boot 3.x. Di Spring Boot 4.x namanya `management.opentelemetry.tracing.export.otlp.endpoint`, dan dependency-nya cukup satu `spring-boot-starter-opentelemetry` (isinya bridge + exporter OTLP, konsepnya tetap Opsi 2). Di Spring Boot 3.x, export metrics via OTLP juga butuh dependency `io.micrometer:micrometer-registry-otlp`; tanpa itu property `management.otlp.metrics.export.url` diabaikan. Kalau nama property salah, exporter tidak jalan dan biasanya tanpa error yang jelas. Berlaku sama untuk format `.yml` maupun `.properties`; contoh varian Boot 4 di `.properties`:
>
> ```properties
> management.opentelemetry.tracing.export.otlp.endpoint=http://otel-collector:4318/v1/traces
> management.opentelemetry.resource-attributes.service.name=order-service
> ```

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

## 7.5 Exception otomatis muncul sebagai stack trace di span

Ini yang langsung memenuhi tujuan utama dokumen ini (lihat bagian "Tujuan"). Kalau exception **propagate keluar** dari operasi yang sudah ter-instrument (HTTP request masuk, RestClient call keluar), Micrometer/OTel **otomatis**:

- Set status span jadi `ERROR`.
- Attach exception sebagai span event berisi `exception.type`, `exception.message`, dan `exception.stacktrace` lengkap.

Tidak perlu kode tambahan. Di Jaeger UI, span yang error ditandai warna merah, dan detail stack trace-nya ada di bagian "Logs" pada span tersebut (lihat bagian 12 untuk cara akses Jaeger UI).

**Batasan:** ini hanya jalan kalau exception benar-benar dilempar (propagate). Kalau exception di-catch dan hanya di-log tanpa dilempar ulang (misal ada fallback logic), span dianggap selesai normal — exception-nya **tidak otomatis** muncul di trace. Untuk kasus ini, rekam manual:

```java
try {
    restClient.get().uri("http://payment-service/api/charge").retrieve().body(Void.class);
} catch (Exception e) {
    Span.current().recordException(e);   // exception tetap kelihatan di trace meski di-catch
    // fallback logic di sini
}
```

---

# 8. Strategi Sampling

Yang di-sample adalah **trace**, bukan log. Log diatur lewat level + filter (bagian 2.1), trace diatur lewat sampling.

## 8.1 Head-based sampling (di aplikasi) — tidak cocok untuk tujuan saat ini

Keputusan sample diambil di awal request, sebelum tahu hasilnya error atau tidak. **Tidak direkomendasikan** untuk tujuan dokumen ini (trace saat error) — ada risiko nyata request yang justru error malah tidak ter-sample, sehingga tidak muncul di Jaeger saat dibutuhkan.

```yaml
management:
  tracing:
    sampling:
      probability: 1.0   # untuk sementara: trace semua request dulu, lihat bagian 8.2
```

Versi `application.properties`:

```properties
management.tracing.sampling.probability=1.0
```

Dipakai untuk sementara dengan `probability: 1.0` (semua di-trace) selama volume trafik masih kecil dan tail-based sampling di Collector belum aktif — supaya tidak ada trace error yang terlewat sejak awal.

## 8.2 Tail-based sampling (di OTel Collector) — wajib untuk tujuan "trace saat error"

Semua request tetap menghasilkan span di aplikasi, tapi keputusan simpan/buang diambil di Collector setelah trace lengkap terkumpul — sehingga request yang **error atau lambat selalu tersimpan**, sisanya (trace normal) disampling kecil.

Karena tujuan utama saat ini adalah memastikan **semua trace error bisa diakses**, `errors-policy` di bawah ini adalah bagian paling penting dari config ini — pastikan policy ini aktif sebelum mengecilkan sampling rate baseline.

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

## Phase 1 — Trace & error visibility (fokus dokumen ini)

```text
Setiap Spring Boot service
    |
    +-- Micrometer Tracing + OTel bridge
          |
          +-- Traces (RestClient antar service otomatis ter-propagate)
          +-- Exception otomatis jadi stack trace di span (bagian 7.5)
```

Target:
- Setiap service menghasilkan trace dan bisa export ke Collector.
- Setiap request antar service (via RestClient) punya satu trace yang utuh, tidak putus.
- Tail-based sampling dengan `errors-policy` aktif — semua trace error pasti tersimpan.
- Trace + stack trace error bisa diakses lewat web Jaeger (bagian 12).

**Metrics dan log pipeline (Loki/Elastic) tidak dikerjakan di Phase 1** — metrics tetap ter-generate otomatis dari Actuator sebagai bonus, tapi belum dikonsumsi/dipakai. Log tetap ke console/file seperti biasa, belum dikirim ke backend observability terpisah.

## Phase 2 — Refinement

```text
- JDBC span (kalau dibutuhkan)
- Log pipeline (OTLP logs -> Loki/Elastic) untuk pencarian log lintas waktu
- Grafana sebagai visualisasi terpadu (kalau nanti butuh "klik trace, lihat log")
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

1. Apakah trace antar service (via RestClient) benar-benar nyambung end-to-end (cek di Jaeger).
2. Apakah `errors-policy` di tail-based sampling benar-benar menangkap semua trace error (coba trigger error manual, cek muncul di Jaeger).
3. Baru lanjut rollout ke service lain, dan pertimbangkan Phase 2/3 sesuai prioritas.

---

# 12. Akses Web Jaeger untuk Melihat Trace

## 12.1 Deploy Jaeger

**Keputusan storage: OpenSearch** (bukan in-memory) — karena target akhirnya production, trace harus tetap ada meski Jaeger/pod di-restart.

Jaeger secara resmi mendukung OpenSearch sebagai storage backend sejak versi **1.53** — pastikan image Jaeger yang dipakai di atas versi itu. Reuse cluster OpenSearch yang sudah ada di infrastruktur kalau memang sudah tersedia; kalau belum, perlu disediakan terpisah.

**Dev/lokal (opsional, in-memory) — untuk uji coba cepat sebelum ke OpenSearch:**

```yaml
# docker-compose (dev/lokal)
jaeger:
  image: jaegertracing/all-in-one:1.60
  environment:
    COLLECTOR_OTLP_ENABLED: "true"
  ports:
    - "16686:16686"   # Jaeger UI
```

**Production — dengan OpenSearch:**

```yaml
jaeger:
  image: jaegertracing/all-in-one:1.60
  environment:
    COLLECTOR_OTLP_ENABLED: "true"
    SPAN_STORAGE_TYPE: opensearch
    ES_SERVER_URLS: http://opensearch:9200
    ES_TLS_ENABLED: "true"          # sesuaikan kalau OpenSearch-nya pakai TLS
    ES_USERNAME: ${OPENSEARCH_USERNAME}
    ES_PASSWORD: ${OPENSEARCH_PASSWORD}
  ports:
    - "16686:16686"
```

Untuk OpenShift/Kubernetes, deploy `jaeger-collector` dan `jaeger-query` (UI) sebagai `Deployment` + `Service` terpisah (bukan `all-in-one`) supaya masing-masing bisa di-scale independen — keduanya diarahkan ke OpenSearch yang sama lewat env var yang sama seperti di atas. `jaeger-collector` menerima OTLP dari OTel Collector (port `4317`/`4318`), `jaeger-query` yang serve UI (port `16686`).

**Retensi data:** OpenSearch tidak punya TTL native seperti Cassandra — retensi diatur lewat index lifecycle policy (rollover + delete index lama). Ini perlu disiapkan supaya index trace tidak tumbuh tanpa batas — misal, simpan trace 14–30 hari lalu index lama dihapus otomatis. Detail policy ini di luar scope dokumen ini, tapi perlu ditandai sebagai task terpisah sebelum go-live.

## 12.2 Cara mengakses web UI-nya

**Lokal / docker-compose:**
Langsung buka `http://localhost:16686` di browser — tidak perlu langkah tambahan.

**Di OpenShift (sesuai arsitektur Anda):**
Opsi paling umum, pakai `Route` supaya bisa diakses lewat browser tanpa VPN/port-forward setiap saat:

```bash
oc expose service jaeger --port=16686 --name=jaeger-ui
oc get route jaeger-ui   # dapatkan URL publik/internal-nya
```

Kalau cuma butuh akses sementara untuk development (belum mau expose permanen), pakai port-forward:

```bash
oc port-forward svc/jaeger 16686:16686
# lalu buka http://localhost:16686
```

**Catatan keamanan:** kalau di-expose lewat `Route`, pertimbangkan apakah perlu dibatasi akses-nya (misal lewat network policy, atau taruh di belakang Kong/gateway internal juga) — Jaeger UI default tidak punya autentikasi bawaan.

## 12.3 Cara mencari trace yang error

Di halaman utama Jaeger UI:

1. Pilih **Service** yang mau dicek dari dropdown (nama-nama ini datang dari `spring.application.name` tiap service, bagian 7.1).
2. Di kolom **Tags**, isi `error=true` — ini akan filter dan tampilkan hanya trace yang mengandung span berstatus error.
3. Klik salah satu trace dari hasil pencarian → akan terbuka detail trace dengan span-span-nya. Span yang error ditandai warna merah.
4. Klik span yang error tersebut → expand bagian **Logs** di detail span → di situ muncul `exception.type`, `exception.message`, dan `exception.stacktrace` lengkap (hasil dari mekanisme otomatis di bagian 7.5).

Kalau sudah tahu `traceId` spesifik (misal dari korelasi manual atau catatan incident), bisa langsung akses lewat URL:
```text
http://<jaeger-host>/trace/<traceId>
```
