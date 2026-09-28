# Sample Implementasi OpenTelemetry di Spring Boot (Java 21)

Use case: **order-service**, **payment-service**, **stock-service**

Dokumen ini pelengkap dari dokumen *Observability Standard untuk Spring Boot Microservice*. Isinya project contoh yang bisa langsung dijalankan untuk membuktikan tujuan utama Phase 1:

> Membuka satu request lewat `traceId` di Jaeger, melihat service mana yang gagal, dan membaca stack trace-nya, tanpa login ke pod satu per satu.

Mekanisme instrumentasi yang dipakai adalah **Opsi 2: Micrometer Tracing + OTel bridge** (konfigurasi lewat `management.*`), sesuai keputusan di dokumen sebelumnya.

---

# 1. Use case dan skenario

## 1.1 Alur bisnis

```text
Client
  |  POST /api/orders
  v
order-service (8081)
  |
  |-- (1) POST /api/stocks/{productId}/reserve ----> stock-service (8083)
  |
  |-- (2) POST /api/payments ----------------------> payment-service (8082)
  |
  '-- (3) jika payment gagal:
          DELETE /api/stocks/reservations/{id} ---> stock-service   (kompensasi)
```

- `order-service` menyimpan order ke database (H2 in-memory, lewat JPA) dan mengorkestrasi dua service lain lewat `RestClient`.
- `stock-service` dan `payment-service` sengaja dibuat sederhana (data in-memory) supaya fokusnya ke trace.

## 1.2 Skenario uji

| # | Skenario | Input | Hasil | Yang mau dibuktikan di trace |
|---|---|---|---|---|
| 1 | Sukses | `SKU-001`, qty 2, amount 50.000 | `201`, order `CONFIRMED` | Satu trace utuh melewati tiga service |
| 2 | Stok habis | `SKU-002` (stok 0) | `409`, order `REJECTED` | Penolakan bisnis, bukan error sistem |
| 3 | Pembayaran gagal | amount 2.000.000 | `502`, order `FAILED`, stok dikembalikan | Trace error, stack trace, langkah kompensasi |
| 4 | Lambat | `SKU-003` | `201` setelah sekitar 1,6 detik | Trace lambat ikut tersimpan lewat latency policy |
| 5 | Produk tidak dikenal | `SKU-999` | `500`, order `FAILED` | Exception yang tidak ditangani tetap tercatat sebagai error |

Data awal stok: `SKU-001` = 10, `SKU-002` = 0, `SKU-003` = 5.

---

# 2. Versi dan prasyarat

| Komponen | Versi | Catatan |
|---|---|---|
| Java | 21 | Memakai record, virtual threads, `Thread.sleep(Duration)` |
| Spring Boot | 4.0.x (contoh: 4.0.8) | Untuk Spring Boot 3.5.x lihat bagian 13 |
| Build tool | Maven 3.9+ | |
| Docker Compose | v2 | Untuk OTel Collector dan Jaeger |
| OTel Collector | `otel/opentelemetry-collector-contrib` | Wajib varian **contrib**, karena `tail_sampling` hanya ada di sana. Pin ke versi tertentu untuk production |
| Jaeger | `jaegertracing/all-in-one:1.60` | Storage in-memory, hanya untuk sample. Production pakai OpenSearch (dokumen observability, bagian 12) |

Catatan versi Spring Boot:

- Spring Boot 4 punya starter resmi `spring-boot-starter-opentelemetry`. Isinya Micrometer Tracing bridge OTel, exporter OTLP, dan `micrometer-registry-otlp`, dikonfigurasi lewat `management.*`. Konsepnya sama dengan Opsi 2, hanya dikemas dalam satu starter.
- Spring Boot 4.1 (rilis Juni 2026) memuat pembaruan dukungan OpenTelemetry. Kalau memakai 4.1.x, cek release notes untuk perubahan nama property atau default sebelum menyalin konfigurasi di bawah.

---

# 3. Struktur project

```text
otel-sample/
├── pom.xml                          (parent, packaging pom)
├── docker-compose.yml               (OTel Collector + Jaeger)
├── otel-collector-config.yaml
├── stock-service/
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/example/sample/stock/
│       │   ├── StockServiceApplication.java
│       │   ├── StockController.java
│       │   ├── StockService.java
│       │   ├── StockExceptionHandler.java
│       │   ├── ReserveRequest.java
│       │   ├── ReserveResponse.java
│       │   ├── InsufficientStockException.java
│       │   └── ProductNotFoundException.java
│       └── resources/application.yml
├── payment-service/
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/example/sample/payment/
│       │   ├── PaymentServiceApplication.java
│       │   ├── PaymentController.java
│       │   ├── PaymentService.java
│       │   ├── PaymentExceptionHandler.java
│       │   ├── ChargeRequest.java
│       │   ├── PaymentResponse.java
│       │   └── PaymentGatewayException.java
│       └── resources/application.yml
└── order-service/
    ├── pom.xml
    └── src/main/
        ├── java/com/example/sample/order/
        │   ├── OrderServiceApplication.java
        │   ├── OrderController.java
        │   ├── OrderService.java
        │   ├── OrderExceptionHandler.java
        │   ├── TraceIdFilter.java
        │   ├── OrderEntity.java
        │   ├── OrderStatus.java
        │   ├── OrderRepository.java
        │   ├── CreateOrderRequest.java
        │   ├── OrderResponse.java
        │   ├── OrderRejectedException.java
        │   ├── OrderProcessingException.java
        │   └── client/
        │       ├── StockClient.java
        │       ├── PaymentClient.java
        │       ├── ReserveRequest.java
        │       ├── ReserveResponse.java
        │       ├── ChargeRequest.java
        │       └── PaymentResponse.java
        └── resources/application.yml
```

Record request/response di `order-service/client` adalah salinan kontrak dari service tujuan. Di project nyata, boleh dipindah ke modul kontrak bersama atau dibiarkan terpisah per service.

---

# 4. Infrastruktur lokal (Collector dan Jaeger)

## 4.1 `docker-compose.yml`

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:1.60
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    ports:
      - "16686:16686"   # Jaeger UI

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest   # pin ke versi tertentu di production
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol/config.yaml:ro
    ports:
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP (dipakai aplikasi)
    depends_on:
      - jaeger
```

## 4.2 `otel-collector-config.yaml`

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
      # 1. Semua trace yang mengandung span error selalu disimpan
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]

      # 2. Semua trace yang lebih lambat dari 1 detik selalu disimpan
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 1000

      # 3. Sisanya disampling. Untuk sample dibuat 100% supaya trace sukses
      #    mudah dilihat. Di production turunkan ke 5-10.
      - name: baseline-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 100

  batch: {}

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
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
    # Metrics belum dipakai di Phase 1. Pipeline ini hanya menampung kiriman
    # metrics dari aplikasi supaya tidak ada error export di log aplikasi.
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

Karena memakai tail sampling dengan `decision_wait: 10s`, **trace baru muncul di Jaeger sekitar 10 detik setelah request selesai**. Ini normal, bukan tanda ada yang rusak.

---

# 5. Maven

## 5.1 `pom.xml` (root)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.8</version>   <!-- pakai patch 4.0.x terbaru -->
    <relativePath/>
  </parent>

  <groupId>com.example</groupId>
  <artifactId>otel-sample</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>

  <properties>
    <java.version>21</java.version>
  </properties>

  <modules>
    <module>stock-service</module>
    <module>payment-service</module>
    <module>order-service</module>
  </modules>
</project>
```

## 5.2 `stock-service/pom.xml` dan `payment-service/pom.xml`

Sama, hanya `artifactId` yang beda.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>com.example</groupId>
    <artifactId>otel-sample</artifactId>
    <version>1.0.0</version>
  </parent>

  <artifactId>stock-service</artifactId>   <!-- payment-service untuk service payment -->

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-webmvc</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-opentelemetry</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

## 5.3 `order-service/pom.xml`

Bedanya: butuh starter `restclient` (di Spring Boot 4, auto-configuration `RestClient.Builder` dipisah dari starter web), JPA, dan H2.

```xml
<artifactId>order-service</artifactId>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webmvc</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-restclient</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

(Header `<parent>` dan `<build>` sama seperti 5.2.)

---

# 6. Konfigurasi per service

Pilih salah satu format per service, `application.yml` **atau** `application.properties`, jangan dua-duanya.

## 6.1 Template `application.yml`

```yaml
server:
  port: 8081                    # order=8081, payment=8082, stock=8083

spring:
  application:
    name: order-service         # jadi service.name di Jaeger
  threads:
    virtual:
      enabled: true             # virtual threads (Java 21)

management:
  tracing:
    sampling:
      probability: 1.0          # semua span dikirim ke Collector, keputusan simpan/buang di tail sampling
  opentelemetry:
    tracing:
      export:
        otlp:
          endpoint: http://localhost:4318/v1/traces
  otlp:
    metrics:
      export:
        url: http://localhost:4318/v1/metrics
```

Tambahan khusus `order-service`:

```yaml
services:
  stock:
    url: http://localhost:8083
  payment:
    url: http://localhost:8082
```

## 6.2 Template `application.properties`

```properties
server.port=8081
spring.application.name=order-service
spring.threads.virtual.enabled=true

management.tracing.sampling.probability=1.0
management.opentelemetry.tracing.export.otlp.endpoint=http://localhost:4318/v1/traces
management.otlp.metrics.export.url=http://localhost:4318/v1/metrics
```

Tambahan khusus `order-service`:

```properties
services.stock.url=http://localhost:8083
services.payment.url=http://localhost:8082
```

## 6.3 Nilai per service

| Service | `server.port` | `spring.application.name` |
|---|---|---|
| order-service | 8081 | `order-service` |
| payment-service | 8082 | `payment-service` |
| stock-service | 8083 | `stock-service` |

## 6.4 Padanan environment variable (untuk Kubernetes/OpenShift)

Di cluster, endpoint Collector biasanya berbeda per environment, jadi lebih praktis di-override lewat env var pada Deployment:

| Property | Environment variable |
|---|---|
| `management.opentelemetry.tracing.export.otlp.endpoint` | `MANAGEMENT_OPENTELEMETRY_TRACING_EXPORT_OTLP_ENDPOINT` |
| `management.otlp.metrics.export.url` | `MANAGEMENT_OTLP_METRICS_EXPORT_URL` |
| `management.tracing.sampling.probability` | `MANAGEMENT_TRACING_SAMPLING_PROBABILITY` |
| `spring.application.name` | `SPRING_APPLICATION_NAME` |

Contoh di Deployment:

```yaml
env:
  - name: MANAGEMENT_OPENTELEMETRY_TRACING_EXPORT_OTLP_ENDPOINT
    value: http://otel-collector:4318/v1/traces
```

## 6.5 Catatan log

Spring Boot 3.2 ke atas otomatis menyisipkan `traceId` dan `spanId` ke setiap baris log begitu Micrometer Tracing ada di classpath. **Tidak perlu** mengatur `logging.pattern.level` sendiri. Kalau dipasang bersamaan, ID akan muncul dua kali di baris log.

---

# 7. stock-service (port 8083)

Package: `com.example.sample.stock`

## 7.1 Application, DTO, dan exception

```java
package com.example.sample.stock;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class StockServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(StockServiceApplication.class, args);
    }
}
```

```java
package com.example.sample.stock;

public record ReserveRequest(String orderId, int quantity) {}
```

```java
package com.example.sample.stock;

public record ReserveResponse(String reservationId, String productId, int quantity) {}
```

```java
package com.example.sample.stock;

public class InsufficientStockException extends RuntimeException {
    public InsufficientStockException(String productId, int requested, int available) {
        super("Insufficient stock for %s: requested=%d, available=%d"
                .formatted(productId, requested, available));
    }
}
```

```java
package com.example.sample.stock;

public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(String productId) {
        super("Product not found: " + productId);
    }
}
```

## 7.2 `StockService`

Ada dua hal yang sengaja dibuat untuk keperluan uji: `SKU-002` stoknya 0 (skenario stok habis), dan `SKU-003` sengaja lambat 1,5 detik (skenario latency).

```java
package com.example.sample.stock;

import java.time.Duration;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.locks.ReentrantLock;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class StockService {

    private static final Logger log = LoggerFactory.getLogger(StockService.class);

    private final Map<String, Integer> stock =
            new HashMap<>(Map.of("SKU-001", 10, "SKU-002", 0, "SKU-003", 5));
    private final Map<String, ReserveResponse> reservations = new HashMap<>();
    private final ReentrantLock lock = new ReentrantLock();

    public ReserveResponse reserve(String productId, ReserveRequest request) {
        simulateSlowLookup(productId);

        lock.lock();
        try {
            Integer available = stock.get(productId);
            if (available == null) {
                throw new ProductNotFoundException(productId);
            }
            if (available < request.quantity()) {
                throw new InsufficientStockException(productId, request.quantity(), available);
            }

            stock.put(productId, available - request.quantity());
            var reservation = new ReserveResponse(
                    UUID.randomUUID().toString(), productId, request.quantity());
            reservations.put(reservation.reservationId(), reservation);

            log.info("Reserved {} x {} for order {} (reservation {})",
                    request.quantity(), productId, request.orderId(), reservation.reservationId());
            return reservation;
        } finally {
            lock.unlock();
        }
    }

    public boolean release(String reservationId) {
        lock.lock();
        try {
            var reservation = reservations.remove(reservationId);
            if (reservation == null) {
                return false;
            }
            stock.merge(reservation.productId(), reservation.quantity(), Integer::sum);
            log.info("Released reservation {} ({} x {})",
                    reservationId, reservation.quantity(), reservation.productId());
            return true;
        } finally {
            lock.unlock();
        }
    }

    private void simulateSlowLookup(String productId) {
        if ("SKU-003".equals(productId)) {
            try {
                Thread.sleep(Duration.ofMillis(1500));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

## 7.3 Controller dan exception handler

```java
package com.example.sample.stock;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/stocks")
public class StockController {

    private final StockService stockService;

    public StockController(StockService stockService) {
        this.stockService = stockService;
    }

    @PostMapping("/{productId}/reserve")
    public ResponseEntity<ReserveResponse> reserve(@PathVariable String productId,
                                                   @RequestBody ReserveRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(stockService.reserve(productId, request));
    }

    @DeleteMapping("/reservations/{reservationId}")
    public ResponseEntity<Void> release(@PathVariable String reservationId) {
        return stockService.release(reservationId)
                ? ResponseEntity.noContent().build()
                : ResponseEntity.notFound().build();
    }
}
```

```java
package com.example.sample.stock;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class StockExceptionHandler {

    // Penolakan bisnis (4xx) sengaja TIDAK ditandai sebagai error di span server.
    // Ini hasil yang wajar, bukan kegagalan sistem.

    @ExceptionHandler(InsufficientStockException.class)
    ProblemDetail insufficient(InsufficientStockException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(ProductNotFoundException.class)
    ProblemDetail notFound(ProductNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }
}
```

---

# 8. payment-service (port 8082)

Package: `com.example.sample.payment`

## 8.1 Application, DTO, dan exception

```java
package com.example.sample.payment;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}
```

```java
package com.example.sample.payment;

import java.math.BigDecimal;

public record ChargeRequest(String orderId, BigDecimal amount) {}
```

```java
package com.example.sample.payment;

import java.math.BigDecimal;

public record PaymentResponse(String paymentId, String orderId, String status, BigDecimal amount) {}
```

```java
package com.example.sample.payment;

public class PaymentGatewayException extends RuntimeException {
    public PaymentGatewayException(String message) {
        super(message);
    }
}
```

## 8.2 `PaymentService` dengan custom span

Pemanggilan ke payment gateway (disimulasikan) dibungkus `Observation` sendiri, supaya muncul sebagai span terpisah `payment.gateway.call` di dalam trace. Kalau melempar exception di dalam `observe(...)`, exception otomatis tercatat di span itu.

Aturan: transaksi di atas 1.000.000 dianggap gagal (simulasi gateway timeout).

```java
package com.example.sample.payment;

import java.math.BigDecimal;
import java.time.Duration;
import java.util.UUID;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class PaymentService {

    private static final Logger log = LoggerFactory.getLogger(PaymentService.class);
    private static final BigDecimal GATEWAY_LIMIT = new BigDecimal("1000000");

    private final ObservationRegistry observationRegistry;

    public PaymentService(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    public PaymentResponse charge(ChargeRequest request) {
        log.info("Charging order {} amount {}", request.orderId(), request.amount());

        Observation.createNotStarted("payment.gateway.call", observationRegistry)
                .lowCardinalityKeyValue("payment.provider", "mock-gateway")
                .observe(() -> callGateway(request));

        var payment = new PaymentResponse(
                UUID.randomUUID().toString(), request.orderId(), "APPROVED", request.amount());
        log.info("Payment {} approved for order {}", payment.paymentId(), request.orderId());
        return payment;
    }

    private void callGateway(ChargeRequest request) {
        try {
            Thread.sleep(Duration.ofMillis(100));   // simulasi latency gateway
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        if (request.amount().compareTo(GATEWAY_LIMIT) > 0) {
            log.error("Gateway rejected order {}: amount {} exceeds limit",
                    request.orderId(), request.amount());
            throw new PaymentGatewayException(
                    "Gateway timeout while charging order " + request.orderId());
        }
    }
}
```

## 8.3 Controller dan exception handler

`PaymentGatewayException` adalah kegagalan sistem (bukan penolakan bisnis), jadi **wajib ditandai error** di span server lewat `setError`. Tanpa baris itu, exception yang ditangani `@RestControllerAdvice` dianggap "sudah beres" oleh instrumentation, dan span server tetap terlihat hijau meski response-nya 502.

```java
package com.example.sample.payment;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    private final PaymentService paymentService;

    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    @PostMapping
    public ResponseEntity<PaymentResponse> charge(@RequestBody ChargeRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(paymentService.charge(request));
    }
}
```

```java
package com.example.sample.payment;

import jakarta.servlet.http.HttpServletRequest;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.filter.ServerHttpObservationFilter;

@RestControllerAdvice
public class PaymentExceptionHandler {

    @ExceptionHandler(PaymentGatewayException.class)
    ProblemDetail gatewayFailure(PaymentGatewayException ex, HttpServletRequest request) {
        // Wajib: tandai span server sebagai error, karena exception sudah "ditangani" di sini
        ServerHttpObservationFilter.findObservationContext(request)
                .ifPresent(context -> context.setError(ex));
        return ProblemDetail.forStatusAndDetail(HttpStatus.BAD_GATEWAY, ex.getMessage());
    }
}
```

---

# 9. order-service (port 8081)

Package: `com.example.sample.order`. Ini service yang paling banyak berisi konsep observability.

## 9.1 Domain dan JPA

```java
package com.example.sample.order;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```java
package com.example.sample.order;

public enum OrderStatus { PENDING, CONFIRMED, REJECTED, FAILED }
```

```java
package com.example.sample.order;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

@Entity
@Table(name = "orders")
public class OrderEntity {

    @Id
    private String id;
    private String productId;
    private int quantity;
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private Instant createdAt;

    protected OrderEntity() {}

    public static OrderEntity pending(String productId, int quantity, BigDecimal amount) {
        var order = new OrderEntity();
        order.id = UUID.randomUUID().toString();
        order.productId = productId;
        order.quantity = quantity;
        order.amount = amount;
        order.status = OrderStatus.PENDING;
        order.createdAt = Instant.now();
        return order;
    }

    public String getId() { return id; }
    public String getProductId() { return productId; }
    public int getQuantity() { return quantity; }
    public BigDecimal getAmount() { return amount; }
    public OrderStatus getStatus() { return status; }
    public Instant getCreatedAt() { return createdAt; }
    public void setStatus(OrderStatus status) { this.status = status; }
}
```

```java
package com.example.sample.order;

import org.springframework.data.jpa.repository.JpaRepository;

public interface OrderRepository extends JpaRepository<OrderEntity, String> {}
```

```java
package com.example.sample.order;

import java.math.BigDecimal;

public record CreateOrderRequest(String productId, int quantity, BigDecimal amount) {}
```

```java
package com.example.sample.order;

import java.math.BigDecimal;

public record OrderResponse(String id, String productId, int quantity,
                            BigDecimal amount, OrderStatus status) {

    public static OrderResponse from(OrderEntity order) {
        return new OrderResponse(order.getId(), order.getProductId(), order.getQuantity(),
                order.getAmount(), order.getStatus());
    }
}
```

```java
package com.example.sample.order;

public class OrderRejectedException extends RuntimeException {
    public OrderRejectedException(String message) {
        super(message);
    }
}
```

```java
package com.example.sample.order;

public class OrderProcessingException extends RuntimeException {
    public OrderProcessingException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

## 9.2 Client ke service lain (RestClient)

Aturan paling penting untuk propagation: **`RestClient.Builder` harus di-inject dari Spring**, jangan memakai `RestClient.create()` atau `new RestTemplate()`. Builder dari Spring sudah dipasangi instrumentation yang menambahkan header `traceparent` ke setiap request keluar. Client yang dibuat manual tidak punya itu, dan trace akan putus.

Record kontrak (salinan dari service tujuan), package `com.example.sample.order.client`:

```java
package com.example.sample.order.client;

public record ReserveRequest(String orderId, int quantity) {}
```

```java
package com.example.sample.order.client;

public record ReserveResponse(String reservationId, String productId, int quantity) {}
```

```java
package com.example.sample.order.client;

import java.math.BigDecimal;

public record ChargeRequest(String orderId, BigDecimal amount) {}
```

```java
package com.example.sample.order.client;

import java.math.BigDecimal;

public record PaymentResponse(String paymentId, String orderId, String status, BigDecimal amount) {}
```

```java
package com.example.sample.order.client;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Component
public class StockClient {

    private final RestClient restClient;

    // Builder di-inject dari Spring (auto-instrumented), lalu di-build untuk base URL service ini
    public StockClient(RestClient.Builder builder, @Value("${services.stock.url}") String baseUrl) {
        this.restClient = builder.baseUrl(baseUrl).build();
    }

    public ReserveResponse reserve(String productId, String orderId, int quantity) {
        return restClient.post()
                .uri("/api/stocks/{productId}/reserve", productId)
                .body(new ReserveRequest(orderId, quantity))
                .retrieve()
                .body(ReserveResponse.class);
    }

    public void release(String reservationId) {
        restClient.delete()
                .uri("/api/stocks/reservations/{id}", reservationId)
                .retrieve()
                .toBodilessEntity();
    }
}
```

```java
package com.example.sample.order.client;

import java.math.BigDecimal;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Component
public class PaymentClient {

    private final RestClient restClient;

    public PaymentClient(RestClient.Builder builder, @Value("${services.payment.url}") String baseUrl) {
        this.restClient = builder.baseUrl(baseUrl).build();
    }

    public PaymentResponse charge(String orderId, BigDecimal amount) {
        return restClient.post()
                .uri("/api/payments")
                .body(new ChargeRequest(orderId, amount))
                .retrieve()
                .body(PaymentResponse.class);
    }
}
```

## 9.3 `OrderService`: orkestrasi, custom observation, dan kompensasi

Poin observability di class ini:

1. **Custom observation `order.place`** membungkus seluruh proses order, sehingga di Jaeger ada satu span yang merepresentasikan "proses bisnis order" dengan atribut sendiri.
2. **High-cardinality key value** (`order.id`, `order.product_id`) untuk atribut yang nilainya unik atau banyak. Cocok untuk trace, tapi **jangan** dijadikan low-cardinality karena akan ikut menjadi label metrics dan membuat cardinality meledak.
3. **Low-cardinality key value** (`order.status`) untuk nilai yang jumlahnya terbatas dan aman jadi label metrics.
4. **Penolakan bisnis** (`OrderRejectedException`) tidak dicatat sebagai error di observation. **Kegagalan sistem** dicatat lewat `observation.error(e)`.
5. **Kompensasi best-effort**: kalau pelepasan stok gagal, exception di-catch (proses tetap lanjut), tetapi dicatat manual lewat `span.error(e)` supaya tetap terlihat di trace. Ini kasus "exception yang di-catch tapi tetap ingin terlihat".
6. **Event** `stock.released` ditempel ke span sebagai penanda langkah kompensasi berhasil.
7. Tidak ada `@Transactional` yang membungkus panggilan remote. Transaksi database tidak boleh menahan koneksi selama menunggu service lain.

```java
package com.example.sample.order;

import java.util.Optional;

import com.example.sample.order.client.PaymentClient;
import com.example.sample.order.client.ReserveResponse;
import com.example.sample.order.client.StockClient;
import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import io.micrometer.tracing.Tracer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.ResourceAccessException;
import org.springframework.web.client.RestClientResponseException;

@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final OrderRepository repository;
    private final StockClient stockClient;
    private final PaymentClient paymentClient;
    private final ObservationRegistry observationRegistry;
    private final Tracer tracer;

    public OrderService(OrderRepository repository,
                        StockClient stockClient,
                        PaymentClient paymentClient,
                        ObservationRegistry observationRegistry,
                        Tracer tracer) {
        this.repository = repository;
        this.stockClient = stockClient;
        this.paymentClient = paymentClient;
        this.observationRegistry = observationRegistry;
        this.tracer = tracer;
    }

    public Optional<OrderEntity> findById(String id) {
        return repository.findById(id);
    }

    public OrderEntity placeOrder(CreateOrderRequest request) {
        var order = OrderEntity.pending(request.productId(), request.quantity(), request.amount());

        var observation = Observation.createNotStarted("order.place", observationRegistry)
                .highCardinalityKeyValue("order.id", order.getId())
                .highCardinalityKeyValue("order.product_id", order.getProductId())
                .start();

        try (var ignored = observation.openScope()) {
            repository.save(order);
            log.info("Placing order {} for product {} qty {}",
                    order.getId(), order.getProductId(), order.getQuantity());

            var reservation = reserveStock(order);
            chargePayment(order, reservation);
            return finish(order, OrderStatus.CONFIRMED, observation);

        } catch (OrderRejectedException e) {
            // Penolakan bisnis: status REJECTED, tidak ditandai error
            finish(order, OrderStatus.REJECTED, observation);
            throw e;

        } catch (RuntimeException e) {
            // Kegagalan sistem: status FAILED, ditandai error di observation
            finish(order, OrderStatus.FAILED, observation);
            observation.error(e);
            throw e;

        } finally {
            observation.stop();
        }
    }

    private ReserveResponse reserveStock(OrderEntity order) {
        try {
            return stockClient.reserve(order.getProductId(), order.getId(), order.getQuantity());
        } catch (HttpClientErrorException.Conflict e) {
            log.warn("Order {} rejected: insufficient stock for {}", order.getId(), order.getProductId());
            throw new OrderRejectedException("Insufficient stock for product " + order.getProductId());
        }
        // Status 404 (produk tidak dikenal) sengaja tidak ditangani di sini,
        // supaya tampak bagaimana exception tak terduga muncul di trace (skenario 5).
    }

    private void chargePayment(OrderEntity order, ReserveResponse reservation) {
        try {
            paymentClient.charge(order.getId(), order.getAmount());
        } catch (RestClientResponseException | ResourceAccessException e) {
            log.error("Payment failed for order {}, releasing reservation {}",
                    order.getId(), reservation.reservationId());
            releaseStockBestEffort(reservation);
            throw new OrderProcessingException("Payment failed for order " + order.getId(), e);
        }
    }

    private void releaseStockBestEffort(ReserveResponse reservation) {
        try {
            stockClient.release(reservation.reservationId());
            var span = tracer.currentSpan();
            if (span != null) {
                span.event("stock.released");
            }
        } catch (RuntimeException e) {
            // Exception di-catch supaya proses tetap lanjut, tapi tetap direkam ke span
            log.warn("Failed to release reservation {}", reservation.reservationId(), e);
            var span = tracer.currentSpan();
            if (span != null) {
                span.error(e);
            }
        }
    }

    private OrderEntity finish(OrderEntity order, OrderStatus status, Observation observation) {
        order.setStatus(status);
        observation.lowCardinalityKeyValue("order.status", status.name().toLowerCase());
        return repository.save(order);
    }
}
```

## 9.4 Controller

```java
package com.example.sample.order;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> create(@RequestBody CreateOrderRequest request) {
        var order = orderService.placeOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(OrderResponse.from(order));
    }

    @GetMapping("/{id}")
    public OrderResponse get(@PathVariable String id) {
        return orderService.findById(id)
                .map(OrderResponse::from)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Order not found"));
    }
}
```

## 9.5 Exception handler dan `traceId` di response

Dua hal yang langsung menjawab kebutuhan "trace saat error":

- Setiap error response berisi field **`traceId`**, sehingga penerima error bisa melaporkan ID itu dan tim tinggal membuka `http://<jaeger>/trace/<traceId>`.
- Tidak ada handler `@ExceptionHandler(Exception.class)` yang menampung semua exception. Exception yang tidak dikenal dibiarkan lewat, dan `ServerHttpObservationFilter` otomatis menandainya sebagai error. Kalau ada catch-all, error framework seperti JSON tidak valid (seharusnya 400) ikut berubah jadi 500.

```java
package com.example.sample.order;

import io.micrometer.tracing.Tracer;
import jakarta.servlet.http.HttpServletRequest;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.filter.ServerHttpObservationFilter;

@RestControllerAdvice
public class OrderExceptionHandler {

    private final Tracer tracer;

    public OrderExceptionHandler(Tracer tracer) {
        this.tracer = tracer;
    }

    @ExceptionHandler(OrderRejectedException.class)
    ProblemDetail rejected(OrderRejectedException ex) {
        return problem(HttpStatus.CONFLICT, ex.getMessage());   // bisnis, bukan error span
    }

    @ExceptionHandler(OrderProcessingException.class)
    ProblemDetail processingFailed(OrderProcessingException ex, HttpServletRequest request) {
        ServerHttpObservationFilter.findObservationContext(request)
                .ifPresent(context -> context.setError(ex));    // kegagalan sistem: tandai error
        return problem(HttpStatus.BAD_GATEWAY, ex.getMessage());
    }

    private ProblemDetail problem(HttpStatus status, String detail) {
        var problem = ProblemDetail.forStatusAndDetail(status, detail);
        var span = tracer.currentSpan();
        if (span != null) {
            problem.setProperty("traceId", span.context().traceId());
        }
        return problem;
    }
}
```

## 9.6 Header `X-Trace-Id` di setiap response

Karena `order-service` adalah entry point, setiap response (sukses maupun gagal) membawa header `X-Trace-Id`. Ini memudahkan mencocokkan satu request dengan trace-nya.

```java
package com.example.sample.order;

import java.io.IOException;

import io.micrometer.tracing.Tracer;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

@Component
class TraceIdFilter extends OncePerRequestFilter {

    private final Tracer tracer;

    TraceIdFilter(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        var span = tracer.currentSpan();
        if (span != null) {
            response.setHeader("X-Trace-Id", span.context().traceId());
        }
        filterChain.doFilter(request, response);
    }
}
```

---

# 10. Menjalankan dan menguji

## 10.1 Menjalankan

```bash
# 1. Infrastruktur (Collector + Jaeger)
docker compose up -d

# 2. Build semua modul
mvn -q -DskipTests package

# 3. Jalankan tiga service, masing-masing di terminal terpisah
mvn -pl stock-service spring-boot:run
mvn -pl payment-service spring-boot:run
mvn -pl order-service spring-boot:run
```

Jaeger UI: <http://localhost:16686>

## 10.2 Skenario uji

**Skenario 1: sukses**

```bash
curl -i -X POST http://localhost:8081/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"productId":"SKU-001","quantity":2,"amount":50000}'
```

Harapan: `201 Created`, `status: CONFIRMED`, dan header `X-Trace-Id` berisi trace ID.

**Skenario 2: stok habis**

```bash
curl -i -X POST http://localhost:8081/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"productId":"SKU-002","quantity":1,"amount":50000}'
```

Harapan: `409 Conflict`, body `ProblemDetail` berisi `traceId`.

**Skenario 3: pembayaran gagal (trace error)**

```bash
curl -i -X POST http://localhost:8081/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"productId":"SKU-001","quantity":1,"amount":2000000}'
```

Harapan: `502 Bad Gateway`, body berisi `traceId`. Di log `stock-service` muncul `Released reservation ...` (kompensasi berjalan).

**Skenario 4: lambat**

```bash
curl -i -X POST http://localhost:8081/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"productId":"SKU-003","quantity":1,"amount":50000}'
```

Harapan: `201 Created` setelah sekitar 1,6 detik.

**Skenario 5: produk tidak dikenal (exception tidak ditangani)**

```bash
curl -i -X POST http://localhost:8081/api/orders \
  -H 'Content-Type: application/json' \
  -d '{"productId":"SKU-999","quantity":1,"amount":50000}'
```

Harapan: `500 Internal Server Error`, header `X-Trace-Id` tetap ada.

---

# 11. Membaca hasilnya di Jaeger

Tunggu sekitar 10 detik setelah request (tail sampling `decision_wait`), lalu buka Jaeger UI.

## 11.1 Cara mencari

- **Berdasarkan `traceId`** (paling cepat untuk kasus error): buka `http://localhost:16686/trace/<traceId>`. Trace ID diambil dari header `X-Trace-Id` atau field `traceId` di body error.
- **Semua trace error**: pilih service `order-service`, isi Tags dengan `error=true`, klik *Find Traces*.
- **Trace lambat**: isi *Min Duration* dengan `1s`.

## 11.2 Bentuk trace yang diharapkan

Nama span persis bisa sedikit berbeda antar versi Spring Boot. Yang perlu dicek adalah **strukturnya**.

**Skenario 1 (sukses):**

```text
order-service   POST /api/orders                        (server)
 └─ order.place                                          (custom observation)
     ├─ POST  → stock-service                            (client, RestClient)
     │    └─ stock-service   POST /api/stocks/{productId}/reserve   (server)
     └─ POST  → payment-service                          (client, RestClient)
          └─ payment-service POST /api/payments          (server)
               └─ payment.gateway.call                   (custom observation)
```

Satu `traceId` untuk ketiga service. Span `order.place` membawa atribut `order.id`, `order.product_id`, dan `order.status=confirmed`.

**Skenario 3 (pembayaran gagal):**

```text
order-service   POST /api/orders                        [ERROR]
 └─ order.place                                          [ERROR]  event: stock.released
     ├─ POST  → stock-service (reserve)                  
     │    └─ stock-service ... reserve
     ├─ POST  → payment-service                          [ERROR]
     │    └─ payment-service POST /api/payments          [ERROR]
     │         └─ payment.gateway.call                   [ERROR]  exception.stacktrace
     └─ DELETE → stock-service (release, kompensasi)
          └─ stock-service ... release
```

Klik span `payment.gateway.call`, buka bagian *Logs* pada detail span. Di situ ada `exception.type`, `exception.message`, dan `exception.stacktrace` lengkap. Ini bukti tujuan utama tercapai: dari satu `traceId`, langsung ketahui service dan baris kode penyebabnya.

**Skenario 2 (stok habis):** span client ke `stock-service` umumnya tercatat error (RestClient melempar exception untuk status 409), padahal span `order.place` dan span server `order-service` tidak ditandai error karena ini penolakan bisnis. Dengan `errors-policy`, trace ini tetap tersimpan. Verifikasi sendiri di Jaeger.

## 11.3 Checklist verifikasi

- [ ] Skenario 1: ketiga service muncul di **satu** trace (bukan tiga trace terpisah).
- [ ] Span `order.place` dan `payment.gateway.call` muncul, lengkap dengan atributnya.
- [ ] Skenario 3: pencarian `error=true` menemukan trace-nya, stack trace terbaca di span `payment.gateway.call`.
- [ ] Skenario 3: event `stock.released` ada di span `order.place`.
- [ ] Skenario 4: trace lambat tersimpan dan bisa dicari lewat *Min Duration*.
- [ ] Skenario 5: trace error muncul walau tidak ada `@ExceptionHandler` untuk kasus itu.
- [ ] Baris log tiap service memuat `traceId` yang sama dengan yang terlihat di Jaeger.

---

# 12. Jebakan yang paling sering bikin trace putus atau menyesatkan

| Gejala | Penyebab | Solusi |
|---|---|---|
| Trace `order-service` dan `stock-service` terpisah | Client dibuat dengan `RestClient.create()` atau `new RestTemplate()` | Selalu inject `RestClient.Builder` / `RestTemplateBuilder` dari Spring |
| Response 5xx tapi span server tidak merah | Exception ditangani `@ExceptionHandler`, sehingga instrumentation menganggap sukses | Panggil `ServerHttpObservationFilter.findObservationContext(request)...setError(ex)` di handler untuk kegagalan sistem |
| Error dari catch-all mengubah 400 jadi 500 | Handler `@ExceptionHandler(Exception.class)` menelan exception framework | Jangan buat catch-all. Exception tak dikenal ditandai error otomatis oleh filter |
| Exception di-catch dan hanya di-log, tidak muncul di trace | Span dianggap selesai normal | Rekam manual: `tracer.currentSpan().error(e)` |
| Trace tidak muncul di Jaeger sesaat setelah request | Tail sampling menunggu `decision_wait` | Tunggu sekitar 10 detik (atau kecilkan `decision_wait` di lingkungan dev) |
| Trace error tidak tersimpan padahal ada | `errors-policy` tidak aktif, atau Collector bukan varian **contrib** | Pakai image `opentelemetry-collector-contrib` dan pastikan policy terdaftar |
| Span hilang dan `traceId` kosong di log setelah `@Async` atau thread pool sendiri | `ThreadLocal` trace context tidak ikut pindah thread | Daftarkan bean `ContextPropagatingTaskDecorator`. Virtual threads dari `spring.threads.virtual.enabled` tidak bermasalah |
| `traceId` muncul dua kali di baris log | `logging.pattern.level` diatur manual di atas default Spring Boot | Hapus pengaturan itu, cukup default |
| Atribut unik (order id, user id) membuat metrics membengkak | Dipasang sebagai `lowCardinalityKeyValue` | Pakai `highCardinalityKeyValue` untuk nilai unik atau banyak |
| Query database tidak muncul sebagai span | Micrometer bridge tidak meng-instrument JDBC otomatis | Lihat dokumen observability bagian 7.3 (`datasource-micrometer`), atau cukup andalkan metrics connection pool |

Contoh `ContextPropagatingTaskDecorator` (baru relevan kalau nanti memakai `@Async`):

```java
@Configuration(proxyBeanMethods = false)
public class ContextPropagationConfiguration {

    @Bean
    ContextPropagatingTaskDecorator contextPropagatingTaskDecorator() {
        return new ContextPropagatingTaskDecorator();
    }
}
```

---

# 13. Jika masih memakai Spring Boot 3.5.x

Seluruh kode Java di bagian 7 sampai 9 **tidak berubah**. Yang berbeda hanya dependency dan nama property.

## 13.1 Dependency

Ganti versi parent ke `3.5.x` terbaru, lalu:

| Spring Boot 4 | Spring Boot 3.5.x |
|---|---|
| `spring-boot-starter-webmvc` | `spring-boot-starter-web` |
| `spring-boot-starter-restclient` | (tidak perlu, `RestClient.Builder` sudah ikut starter web) |
| `spring-boot-starter-opentelemetry` | `io.micrometer:micrometer-tracing-bridge-otel` + `io.opentelemetry:opentelemetry-exporter-otlp` (+ `io.micrometer:micrometer-registry-otlp` kalau metrics OTLP ingin aktif) |

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
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

## 13.2 Property

```yaml
management:
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces      # Boot 3.x
    metrics:
      export:
        url: http://localhost:4318/v1/metrics
```

```properties
management.tracing.sampling.probability=1.0
management.otlp.tracing.endpoint=http://localhost:4318/v1/traces
management.otlp.metrics.export.url=http://localhost:4318/v1/metrics
```

Perbedaan nama utama: `management.otlp.tracing.endpoint` (Boot 3.x) menjadi `management.opentelemetry.tracing.export.otlp.endpoint` (Boot 4.x). Kalau nama property salah, exporter tidak jalan dan biasanya **tanpa error yang jelas**. Kalau trace tidak sampai ke Collector, cek nama property ini lebih dulu.

---

# 14. Menuju production

Yang berubah dari sample ke production, semuanya sudah dibahas di dokumen observability:

| Hal | Sample | Production |
|---|---|---|
| Storage trace | Jaeger in-memory | Jaeger + OpenSearch (dokumen observability, bagian 12.1), lengkap dengan retensi index (ISM) |
| Baseline sampling | 100% | 5 sampai 10%, dengan `errors-policy` dan `latency-policy` tetap aktif |
| Collector | Satu container | Topologi agent + gateway. Kalau gateway lebih dari satu replica, tail sampling butuh routing berdasarkan `traceId` |
| Endpoint OTLP | `localhost:4318` di property | Env var di Deployment (bagian 6.4) |
| Akses Jaeger UI | `localhost:16686` | `Route` OpenShift atau `port-forward`, dengan pembatasan akses karena Jaeger UI tidak punya autentikasi bawaan |
| `management.tracing.sampling.probability` | 1.0 | Tetap 1.0 selama tail sampling di Collector dipakai. Menurunkannya di aplikasi berarti kembali ke head-based sampling yang bisa melewatkan trace error |
| Versi image | `latest` | Pin versi Collector dan Jaeger |

Urutan rollout yang disarankan: pasang di satu service percobaan, jalankan skenario error secara manual di lingkungan staging, pastikan trace error benar-benar muncul di Jaeger, baru diperluas ke service lain.
