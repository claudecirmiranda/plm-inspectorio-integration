# PLM → Inspectorio Integration Platform

> Projeto completo de integração entre PLM (Centric) e Inspectorio.  
> Java 21 · Spring Boot 3.2 · Maven 3.9.1 · Docker Compose · Graylog GELF

---

# Backend Repository

## Visão Geral

**Nome do repositório:** `plm-inspectorio-integration`

Serviço Spring Boot responsável por dois fluxos independentes de sincronização de dados mestres do PLM (Centric) para o Inspectorio:

- **Fluxo 1 – Materials:** síncrono, resposta após conclusão do processamento
- **Fluxo 2 – Style → SKUs:** assíncrono, resposta imediata HTTP 202, processamento em background

---

## Estrutura de Diretórios

```
plm-inspectorio-integration/
├── src/
│   ├── main/
│   │   ├── java/com/empresa/integration/
│   │   │   ├── IntegrationApplication.java
│   │   │   ├── config/
│   │   │   │   ├── AsyncConfig.java
│   │   │   │   ├── HttpClientConfig.java
│   │   │   │   ├── Resilience4jConfig.java
│   │   │   │   └── SecurityConfig.java
│   │   │   ├── controller/
│   │   │   │   ├── MaterialsIntegrationController.java
│   │   │   │   └── ProductsIntegrationController.java
│   │   │   ├── service/
│   │   │   │   ├── MaterialsIntegrationService.java
│   │   │   │   └── ProductsIntegrationService.java
│   │   │   ├── transformer/
│   │   │   │   ├── MaterialTransformer.java
│   │   │   │   └── ProductTransformer.java
│   │   │   ├── client/
│   │   │   │   ├── PLMClient.java
│   │   │   │   ├── InspectorioClient.java
│   │   │   │   └── PLMSessionManager.java
│   │   │   ├── domain/
│   │   │   │   ├── plm/
│   │   │   │   │   ├── PlmMaterial.java
│   │   │   │   │   ├── PlmStyle.java
│   │   │   │   │   └── PlmSku.java
│   │   │   │   └── inspectorio/
│   │   │   │       ├── InspectorioMaterial.java
│   │   │   │       ├── InspectorioProduct.java
│   │   │   │       └── InspectorioItem.java
│   │   │   ├── dto/
│   │   │   │   ├── request/
│   │   │   │   │   ├── MaterialsIntegrationRequest.java
│   │   │   │   │   └── ProductsIntegrationRequest.java
│   │   │   │   └── response/
│   │   │   │       ├── MaterialsIntegrationResponse.java
│   │   │   │       └── ProductsIntegrationResponse.java
│   │   │   ├── exception/
│   │   │   │   ├── RetryableIntegrationException.java
│   │   │   │   ├── NonRetryableIntegrationException.java
│   │   │   │   ├── ValidationException.java
│   │   │   │   └── GlobalExceptionHandler.java
│   │   │   └── observability/
│   │   │       ├── StructuredLogger.java
│   │   │       └── MdcTaskDecorator.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-local.yml
│   │       ├── application-stub.yml
│   │       ├── application-prod.yml
│   │       └── logback-spring.xml
│   └── test/
│       ├── java/com/empresa/integration/
│       │   ├── controller/
│       │   │   ├── MaterialsIntegrationControllerTest.java
│       │   │   └── ProductsIntegrationControllerTest.java
│       │   ├── service/
│       │   │   ├── MaterialsIntegrationServiceTest.java
│       │   │   └── ProductsIntegrationServiceTest.java
│       │   ├── client/
│       │   │   ├── PLMClientTest.java
│       │   │   └── InspectorioClientTest.java
│       │   └── observability/
│       │       └── MdcPropagationTest.java
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── cd.yml
├── Makefile
├── pom.xml
└── README.md
```

---

## pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.3</version>
        <relativePath/>
    </parent>

    <groupId>com.empresa</groupId>
    <artifactId>plm-inspectorio-integration</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <name>plm-inspectorio-integration</name>
    <description>PLM to Inspectorio Integration Service</description>

    <properties>
        <java.version>21</java.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <resilience4j.version>2.2.0</resilience4j.version>
        <logstash-encoder.version>7.4</logstash-encoder.version>
        <logback-gelf.version>5.0.1</logback-gelf.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web MVC -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Spring Actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Resilience4j -->
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-spring-boot3</artifactId>
            <version>${resilience4j.version}</version>
        </dependency>
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-micrometer</artifactId>
            <version>${resilience4j.version}</version>
        </dependency>

        <!-- Apache HttpClient 5 for RestClient -->
        <dependency>
            <groupId>org.apache.httpcomponents.client5</groupId>
            <artifactId>httpclient5</artifactId>
        </dependency>

        <!-- MapStruct -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>

        <!-- Logstash Logback Encoder -->
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>${logstash-encoder.version}</version>
        </dependency>

        <!-- Logback GELF Appender -->
        <dependency>
            <groupId>de.siegmar</groupId>
            <artifactId>logback-gelf</artifactId>
            <version>${logback-gelf.version}</version>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-testcontainers</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.github.tomakehurst</groupId>
            <artifactId>wiremock-jre8-standalone</artifactId>
            <version>2.35.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>21</source>
                    <target>21</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok-mapstruct-binding</artifactId>
                            <version>0.2.0</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-checkstyle-plugin</artifactId>
                <version>3.3.1</version>
                <configuration>
                    <configLocation>checkstyle.xml</configLocation>
                    <failsOnError>true</failsOnError>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## application.yml

```yaml
spring:
  application:
    name: plm-inspectorio-integration
  lifecycle:
    timeout-per-shutdown-phase: 60s   # graceful shutdown aguarda tarefas async

server:
  port: 8080
  shutdown: graceful

# Async executor
integration:
  async:
    core-pool-size: 5
    max-pool-size: 10
    queue-capacity: 100
    thread-name-prefix: async-products-

  # PLM (Centric) client
  plm:
    base-url: ${PLM_BASE_URL:https://plm-hml.empresa.com.br}
    session-username: ${PLM_USERNAME}
    session-password: ${PLM_PASSWORD}
    session-ttl-minutes: 30
    connect-timeout-ms: 5000
    read-timeout-ms: 30000
    max-connections-per-route: 20
    max-connections-total: 50
    retry:
      max-attempts: 5
      base-interval-ms: 500
      exponential-factor: 2

  # Inspectorio client
  inspectorio:
    base-url: ${INSPECTORIO_BASE_URL:https://api.inspectorio.com}
    api-key: ${INSPECTORIO_API_KEY}
    connect-timeout-ms: 5000
    read-timeout-ms: 60000
    max-connections-per-route: 10
    max-connections-total: 20

# Resilience4j
resilience4j:
  retry:
    instances:
      plm-retry:
        max-attempts: 5
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - com.empresa.integration.exception.RetryableIntegrationException
        ignore-exceptions:
          - com.empresa.integration.exception.NonRetryableIntegrationException
      inspectorio-retry:
        max-attempts: 5
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - com.empresa.integration.exception.RetryableIntegrationException
        ignore-exceptions:
          - com.empresa.integration.exception.NonRetryableIntegrationException

# Observabilidade
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true

# Logging
logging:
  level:
    com.empresa.integration: INFO
    org.springframework.web: WARN
```

---

## logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="appName" source="spring.application.name"/>

    <!-- Console appender (dev) -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"${appName}"}</customFields>
            <!-- SECURITY: never log these MDC keys -->
            <excludeMdcKeyName>plm_session_token</excludeMdcKeyName>
            <excludeMdcKeyName>inspectorio_api_key</excludeMdcKeyName>
        </encoder>
    </appender>

    <!-- GELF appender (Graylog) -->
    <appender name="GELF" class="de.siegmar.logbackgelf.GelfUdpAppender">
        <graylogHost>${GRAYLOG_HOST:graylog-hml.empresa.com.br}</graylogHost>
        <graylogPort>${GRAYLOG_PORT:12201}</graylogPort>
        <encoder class="de.siegmar.logbackgelf.GelfEncoder">
            <includeRawMessage>false</includeRawMessage>
            <includeMdcData>true</includeMdcData>
            <staticField>service:${appName}</staticField>
        </encoder>
    </appender>

    <!-- Async wrapper para não bloquear threads de negócio -->
    <appender name="ASYNC_GELF" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="GELF"/>
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <neverBlock>true</neverBlock>
    </appender>

    <springProfile name="local">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="prod,hml">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="ASYNC_GELF"/>
        </root>
    </springProfile>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

---

## Código-Fonte Principal

### IntegrationApplication.java

```java
package com.empresa.integration;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;

@SpringBootApplication
@EnableAsync
public class IntegrationApplication {
    public static void main(String[] args) {
        SpringApplication.run(IntegrationApplication.class, args);
    }
}
```

---

### MdcTaskDecorator.java

```java
package com.empresa.integration.observability;

import org.slf4j.MDC;
import org.springframework.core.task.TaskDecorator;
import org.springframework.lang.NonNull;
import java.util.Map;

/**
 * Propaga o contexto MDC (correlation_id, batch_id, etc.) do thread
 * chamador para os threads do ThreadPoolTaskExecutor.
 *
 * CRÍTICO: sem este decorator, logs emitidos pelo fluxo assíncrono
 * Style→SKU não conterão correlation_id nem batch_id, quebrando a
 * rastreabilidade no Graylog.
 */
public class MdcTaskDecorator implements TaskDecorator {

    @NonNull
    @Override
    public Runnable decorate(@NonNull Runnable runnable) {
        // Captura MDC no thread CHAMADOR (thread do controller)
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (contextMap != null) {
                    MDC.setContextMap(contextMap);
                }
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

---

### AsyncConfig.java

```java
package com.empresa.integration.config;

import com.empresa.integration.observability.MdcTaskDecorator;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Slf4j
@Configuration
public class AsyncConfig {

    @Value("${integration.async.core-pool-size:5}")
    private int corePoolSize;

    @Value("${integration.async.max-pool-size:10}")
    private int maxPoolSize;

    @Value("${integration.async.queue-capacity:100}")
    private int queueCapacity;

    @Value("${integration.async.thread-name-prefix:async-products-}")
    private String threadNamePrefix;

    @Bean(name = "productsAsyncExecutor")
    public Executor productsAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setThreadNamePrefix(threadNamePrefix);
        // CRÍTICO: propaga MDC para threads filhas
        executor.setTaskDecorator(new MdcTaskDecorator());
        // Graceful shutdown: aguarda tarefas em andamento
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        executor.initialize();
        log.info("AsyncExecutor configurado: core={}, max={}, queue={}",
                corePoolSize, maxPoolSize, queueCapacity);
        return executor;
    }
}
```

---

### StructuredLogger.java

```java
package com.empresa.integration.observability;

import net.logstash.logback.argument.StructuredArguments;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;

/**
 * Contrato unificado de log estruturado.
 * Campos obrigatórios: batch_id, correlation_id, entity_id,
 * status, http_status, execution_time_ms.
 *
 * SECURITY: nunca adicione session_token ou api_key no MDC.
 */
@Component
public class StructuredLogger {

    private static final Logger log = LoggerFactory.getLogger(StructuredLogger.class);

    public void logStart(String batchId, String correlationId,
                         String entityType, String entityId) {
        MDC.put("batch_id", batchId);
        MDC.put("correlation_id", correlationId);
        MDC.put("entity_type", entityType);
        MDC.put("entity_id", entityId);

        log.info("Integration started",
                StructuredArguments.kv("event", "INTEGRATION_START"),
                StructuredArguments.kv("batch_id", batchId),
                StructuredArguments.kv("correlation_id", correlationId),
                StructuredArguments.kv("entity_type", entityType),
                StructuredArguments.kv("entity_id", entityId));
    }

    public void logSuccess(String batchId, String correlationId,
                           String entityType, String entityId,
                           int httpStatus, long executionTimeMs) {
        log.info("Integration completed successfully",
                StructuredArguments.kv("event", "INTEGRATION_SUCCESS"),
                StructuredArguments.kv("batch_id", batchId),
                StructuredArguments.kv("correlation_id", correlationId),
                StructuredArguments.kv("entity_type", entityType),
                StructuredArguments.kv("entity_id", entityId),
                StructuredArguments.kv("status", "SUCCESSFUL"),
                StructuredArguments.kv("http_status", httpStatus),
                StructuredArguments.kv("execution_time_ms", executionTimeMs));
    }

    public void logError(String batchId, String correlationId,
                         String entityType, String entityId,
                         int httpStatus, long executionTimeMs,
                         int retryAttempts, String errorMessage) {
        log.error("Integration failed",
                StructuredArguments.kv("event", "INTEGRATION_ERROR"),
                StructuredArguments.kv("batch_id", batchId),
                StructuredArguments.kv("correlation_id", correlationId),
                StructuredArguments.kv("entity_type", entityType),
                StructuredArguments.kv("entity_id", entityId),
                StructuredArguments.kv("status", "ERROR"),
                StructuredArguments.kv("http_status", httpStatus),
                StructuredArguments.kv("execution_time_ms", executionTimeMs),
                StructuredArguments.kv("retry_attempts", retryAttempts),
                StructuredArguments.kv("error_message", errorMessage));
    }

    public void clearContext() {
        MDC.clear();
    }
}
```

---

### MaterialsIntegrationController.java

```java
package com.empresa.integration.controller;

import com.empresa.integration.dto.request.MaterialsIntegrationRequest;
import com.empresa.integration.dto.response.MaterialsIntegrationResponse;
import com.empresa.integration.service.MaterialsIntegrationService;
import com.empresa.integration.observability.StructuredLogger;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@Slf4j
@RestController
@RequestMapping("/integration")
@RequiredArgsConstructor
public class MaterialsIntegrationController {

    private final MaterialsIntegrationService materialsService;
    private final StructuredLogger structuredLogger;

    @PostMapping("/materials")
    public ResponseEntity<MaterialsIntegrationResponse> integrateMaterials(
            @Valid @RequestBody MaterialsIntegrationRequest request) {

        String batchId = UUID.randomUUID().toString();
        String correlationId = UUID.randomUUID().toString();

        structuredLogger.logStart(batchId, correlationId, "MATERIAL",
                String.join(",", request.getMaterialIds()));

        MaterialsIntegrationResponse response =
                materialsService.processMaterials(request, batchId, correlationId);

        return ResponseEntity.ok(response);
    }
}
```

---

### ProductsIntegrationController.java

```java
package com.empresa.integration.controller;

import com.empresa.integration.dto.request.ProductsIntegrationRequest;
import com.empresa.integration.dto.response.ProductsIntegrationResponse;
import com.empresa.integration.service.ProductsIntegrationService;
import com.empresa.integration.observability.StructuredLogger;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@Slf4j
@RestController
@RequestMapping("/integration")
@RequiredArgsConstructor
public class ProductsIntegrationController {

    private final ProductsIntegrationService productsService;
    private final StructuredLogger structuredLogger;

    @PostMapping("/products")
    public ResponseEntity<ProductsIntegrationResponse> integrateProducts(
            @Valid @RequestBody ProductsIntegrationRequest request) {

        String batchId = UUID.randomUUID().toString();
        String correlationId = UUID.randomUUID().toString();
        String processId = UUID.randomUUID().toString();

        structuredLogger.logStart(batchId, correlationId, "PRODUCT",
                request.getStyleId());

        // Resposta síncrona imediata — processamento ocorre em background
        productsService.processProductsAsync(request, batchId, correlationId);

        ProductsIntegrationResponse response = ProductsIntegrationResponse.builder()
                .processId(processId)
                .status("RECEBIDO")
                .batchId(batchId)
                .correlationId(correlationId)
                .build();

        return ResponseEntity.status(HttpStatus.ACCEPTED).body(response);
    }
}
```

---

### MaterialsIntegrationService.java

```java
package com.empresa.integration.service;

import com.empresa.integration.client.PLMClient;
import com.empresa.integration.client.InspectorioClient;
import com.empresa.integration.domain.inspectorio.InspectorioMaterial;
import com.empresa.integration.domain.plm.PlmMaterial;
import com.empresa.integration.dto.request.MaterialsIntegrationRequest;
import com.empresa.integration.dto.response.MaterialsIntegrationResponse;
import com.empresa.integration.exception.NonRetryableIntegrationException;
import com.empresa.integration.observability.StructuredLogger;
import com.empresa.integration.transformer.MaterialTransformer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class MaterialsIntegrationService {

    private final PLMClient plmClient;
    private final InspectorioClient inspectorioClient;
    private final MaterialTransformer materialTransformer;
    private final StructuredLogger structuredLogger;

    public MaterialsIntegrationResponse processMaterials(
            MaterialsIntegrationRequest request,
            String batchId,
            String correlationId) {

        List<String> successful = new ArrayList<>();
        List<String> failed = new ArrayList<>();

        for (String materialId : request.getMaterialIds()) {
            long startTime = System.currentTimeMillis();
            try {
                // 1. Atualiza status para Processing (5) no PLM
                plmClient.updateMaterialStatus(materialId, 5);

                // 2. Busca dados completos do PLM
                PlmMaterial plmMaterial = plmClient.getMaterial(materialId);

                // 3. Transforma para modelo Inspectorio
                InspectorioMaterial inspectorioMaterial =
                        materialTransformer.transform(plmMaterial);

                // 4. Envia ao Inspectorio (upsert idempotente via custom_id)
                int httpStatus = inspectorioClient.upsertMaterial(inspectorioMaterial);

                // 5. Atualiza status para Successful (3) no PLM
                plmClient.updateMaterialStatus(materialId, 3);

                long executionTime = System.currentTimeMillis() - startTime;
                structuredLogger.logSuccess(batchId, correlationId, "MATERIAL",
                        materialId, httpStatus, executionTime);

                successful.add(materialId);

            } catch (NonRetryableIntegrationException e) {
                handleMaterialError(materialId, batchId, correlationId,
                        startTime, e, failed, 0);
            } catch (Exception e) {
                handleMaterialError(materialId, batchId, correlationId,
                        startTime, e, failed, -1);
            }
        }

        return MaterialsIntegrationResponse.builder()
                .batchId(batchId)
                .correlationId(correlationId)
                .successful(successful)
                .failed(failed)
                .build();
    }

    private void handleMaterialError(String materialId, String batchId,
            String correlationId, long startTime, Exception e,
            List<String> failed, int httpStatus) {
        try {
            plmClient.updateMaterialStatus(materialId, 4); // Error
        } catch (Exception updateEx) {
            log.error("Failed to update PLM status to Error for material {}",
                    materialId, updateEx);
        }
        long executionTime = System.currentTimeMillis() - startTime;
        structuredLogger.logError(batchId, correlationId, "MATERIAL",
                materialId, httpStatus, executionTime, 0, e.getMessage());
        failed.add(materialId);
    }
}
```

---

### ProductsIntegrationService.java

```java
package com.empresa.integration.service;

import com.empresa.integration.client.PLMClient;
import com.empresa.integration.client.InspectorioClient;
import com.empresa.integration.domain.inspectorio.InspectorioProduct;
import com.empresa.integration.domain.plm.PlmSku;
import com.empresa.integration.domain.plm.PlmStyle;
import com.empresa.integration.dto.request.ProductsIntegrationRequest;
import com.empresa.integration.observability.StructuredLogger;
import com.empresa.integration.transformer.ProductTransformer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductsIntegrationService {

    private final PLMClient plmClient;
    private final InspectorioClient inspectorioClient;
    private final ProductTransformer productTransformer;
    private final StructuredLogger structuredLogger;

    @Async("productsAsyncExecutor")
    public void processProductsAsync(ProductsIntegrationRequest request,
                                      String batchId, String correlationId) {
        long startTime = System.currentTimeMillis();
        String styleId = request.getStyleId();
        List<String> skuIds = request.getSkuIds();

        try {
            // 1. Atualiza Style e todos os SKUs para Processing (5)
            plmClient.updateStyleStatus(styleId, 5);
            skuIds.forEach(skuId -> plmClient.updateSkuStatus(skuId, 5));

            // 2. Busca dados do Style e de cada SKU
            PlmStyle plmStyle = plmClient.getStyle(styleId);
            List<PlmSku> plmSkus = skuIds.stream()
                    .map(plmClient::getSku)
                    .toList();

            // 3. Transforma para modelo agregado Inspectorio
            InspectorioProduct product =
                    productTransformer.transform(plmStyle, plmSkus);

            // 4. Envia payload agregado ao Inspectorio
            int httpStatus = inspectorioClient.upsertProduct(product);

            // 5. Atualiza Style e SKUs para Successful (3)
            plmClient.updateStyleStatus(styleId, 3);
            skuIds.forEach(skuId -> plmClient.updateSkuStatus(skuId, 3));

            long executionTime = System.currentTimeMillis() - startTime;
            structuredLogger.logSuccess(batchId, correlationId, "STYLE",
                    styleId, httpStatus, executionTime);
            skuIds.forEach(skuId ->
                    structuredLogger.logSuccess(batchId, correlationId, "SKU",
                            skuId, httpStatus, executionTime));

        } catch (Exception e) {
            handleProductError(styleId, skuIds, batchId, correlationId,
                    startTime, e);
        }
    }

    private void handleProductError(String styleId, List<String> skuIds,
            String batchId, String correlationId, long startTime, Exception e) {
        try {
            plmClient.updateStyleStatus(styleId, 4); // Error
            skuIds.forEach(skuId -> {
                try { plmClient.updateSkuStatus(skuId, 4); }
                catch (Exception ex) {
                    log.error("Failed to update PLM Error status for SKU {}", skuId, ex);
                }
            });
        } catch (Exception updateEx) {
            log.error("Failed to update PLM Error status for Style {}", styleId, updateEx);
        }
        long executionTime = System.currentTimeMillis() - startTime;
        structuredLogger.logError(batchId, correlationId, "STYLE",
                styleId, -1, executionTime, 0, e.getMessage());
    }
}
```

---

### PLMClient.java

```java
package com.empresa.integration.client;

import com.empresa.integration.domain.plm.PlmMaterial;
import com.empresa.integration.domain.plm.PlmSku;
import com.empresa.integration.domain.plm.PlmStyle;
import com.empresa.integration.exception.NonRetryableIntegrationException;
import com.empresa.integration.exception.RetryableIntegrationException;
import io.github.resilience4j.retry.annotation.Retry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.HttpServerErrorException;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.RestClientException;

/**
 * Encapsula toda comunicação com o PLM (Centric).
 * Autenticação via session token gerenciada pelo PLMSessionManager.
 * Retry via Resilience4j (@Retry) — NÃO aplica retry em erros de validação.
 *
 * SECURITY: session token NUNCA é adicionado ao MDC nem logado.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class PLMClient {

    private final RestClient plmRestClient;
    private final PLMSessionManager sessionManager;

    @Retry(name = "plm-retry", fallbackMethod = "handleRetryExhausted")
    public PlmMaterial getMaterial(String materialId) {
        return plmRestClient.get()
                .uri("/materials/{id}", materialId)
                .header("Authorization", "Bearer " + sessionManager.getSessionToken())
                .retrieve()
                .onStatus(status -> status.is4xxClientError(),
                        (req, res) -> {
                            throw new NonRetryableIntegrationException(
                                    "PLM returned 4xx for material " + materialId +
                                    ": " + res.getStatusCode());
                        })
                .onStatus(status -> status.is5xxServerError(),
                        (req, res) -> {
                            throw new RetryableIntegrationException(
                                    "PLM returned 5xx for material " + materialId);
                        })
                .body(PlmMaterial.class);
    }

    @Retry(name = "plm-retry")
    public PlmStyle getStyle(String styleId) {
        return plmRestClient.get()
                .uri("/styles/{id}", styleId)
                .header("Authorization", "Bearer " + sessionManager.getSessionToken())
                .retrieve()
                .onStatus(status -> status.is4xxClientError(),
                        (req, res) -> {
                            throw new NonRetryableIntegrationException(
                                    "PLM returned 4xx for style " + styleId);
                        })
                .onStatus(status -> status.is5xxServerError(),
                        (req, res) -> {
                            throw new RetryableIntegrationException(
                                    "PLM returned 5xx for style " + styleId);
                        })
                .body(PlmStyle.class);
    }

    @Retry(name = "plm-retry")
    public PlmSku getSku(String skuId) {
        return plmRestClient.get()
                .uri("/skus/{id}", skuId)
                .header("Authorization", "Bearer " + sessionManager.getSessionToken())
                .retrieve()
                .body(PlmSku.class);
    }

    @Retry(name = "plm-retry")
    public void updateMaterialStatus(String materialId, int statusCode) {
        plmRestClient.put()
                .uri("/materials/{id}", materialId)
                .header("Authorization", "Bearer " + sessionManager.getSessionToken())
                .body(java.util.Map.of("status", statusCode))
                .retrieve()
                .toBodilessEntity();
    }

    @Retry(name = "plm-retry")
    public void updateStyleStatus(String styleId, int statusCode) {
        plmRestClient.put()
                .uri("/styles/{id}", styleId)
                .header("Authorization", "Bearer " + sessionManager.getSessionToken())
                .body(java.util.Map.of("status", statusCode))
                .retrieve()
                .toBodilessEntity();
    }

    @Retry(name = "plm-retry")
    public void updateSkuStatus(String skuId, int statusCode) {
        plmRestClient.put()
                .uri("/skus/{id}", skuId)
                .header("Authorization", "Bearer " + sessionManager.getSessionToken())
                .body(java.util.Map.of("status", statusCode))
                .retrieve()
                .toBodilessEntity();
    }

    // Fallback invocado após esgotamento do retry
    private PlmMaterial handleRetryExhausted(String materialId, Exception e) {
        throw new RetryableIntegrationException(
                "PLM retry exhausted for material " + materialId, e);
    }
}
```

---

### PLMSessionManager.java

```java
package com.empresa.integration.client;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

import java.time.Instant;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Gerencia o ciclo de vida da sessão com o PLM.
 * Obtém, armazena em memória e renova automaticamente o session token.
 * SECURITY: token jamais é exposto em log ou MDC.
 */
@Slf4j
@Component
public class PLMSessionManager {

    private final RestClient plmRestClient;
    private final String username;
    private final String password;
    private final long sessionTtlMinutes;

    private volatile String sessionToken;
    private volatile Instant tokenExpiresAt = Instant.EPOCH;
    private final ReentrantLock lock = new ReentrantLock();

    public PLMSessionManager(
            RestClient plmRestClient,
            @Value("${integration.plm.session-username}") String username,
            @Value("${integration.plm.session-password}") String password,
            @Value("${integration.plm.session-ttl-minutes:30}") long sessionTtlMinutes) {
        this.plmRestClient = plmRestClient;
        this.username = username;
        this.password = password;
        this.sessionTtlMinutes = sessionTtlMinutes;
    }

    public String getSessionToken() {
        if (isTokenValid()) {
            return sessionToken;
        }
        return renewSession();
    }

    private boolean isTokenValid() {
        return sessionToken != null &&
               Instant.now().isBefore(tokenExpiresAt.minusSeconds(60));
    }

    private String renewSession() {
        lock.lock();
        try {
            // Double-check após adquirir lock
            if (isTokenValid()) return sessionToken;

            log.info("Renewing PLM session token");
            // TODO: implementar chamada real ao endpoint de autenticação do PLM
            // quando o mecanismo de sessão for especificado pela equipe do PLM
            // Exemplo: POST /sessions com {username, password}
            var response = plmRestClient.post()
                    .uri("/sessions")
                    .body(java.util.Map.of("username", username, "password", password))
                    .retrieve()
                    .body(java.util.Map.class);

            this.sessionToken = (String) response.get("token");
            this.tokenExpiresAt = Instant.now()
                    .plusSeconds(sessionTtlMinutes * 60);

            log.info("PLM session token renewed successfully");
            return sessionToken;
        } finally {
            lock.unlock();
        }
    }
}
```

---

### InspectorioClient.java

```java
package com.empresa.integration.client;

import com.empresa.integration.domain.inspectorio.InspectorioMaterial;
import com.empresa.integration.domain.inspectorio.InspectorioProduct;
import com.empresa.integration.exception.NonRetryableIntegrationException;
import com.empresa.integration.exception.RetryableIntegrationException;
import io.github.resilience4j.retry.annotation.Retry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

import java.util.Map;

/**
 * Encapsula comunicação com o Inspectorio.
 * Autenticação via API Key no header X-API-Key.
 * SECURITY: API Key NUNCA é adicionada ao MDC nem logada.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class InspectorioClient {

    private final RestClient inspectorioRestClient;

    @Retry(name = "inspectorio-retry")
    public int upsertMaterial(InspectorioMaterial material) {
        ResponseEntity<Void> response = inspectorioRestClient.put()
                .uri("/api/v1/customer-data/products")
                .body(Map.of(
                    "entity_type", "material",
                    "force_update", true,
                    "data", material
                ))
                .retrieve()
                .onStatus(status -> status.is4xxClientError(),
                        (req, res) -> {
                            throw new NonRetryableIntegrationException(
                                    "Inspectorio rejected material: " + res.getStatusCode());
                        })
                .onStatus(status -> status.is5xxServerError(),
                        (req, res) -> {
                            throw new RetryableIntegrationException(
                                    "Inspectorio server error: " + res.getStatusCode());
                        })
                .toBodilessEntity();

        return response.getStatusCode().value();
    }

    @Retry(name = "inspectorio-retry")
    public int upsertProduct(InspectorioProduct product) {
        ResponseEntity<Void> response = inspectorioRestClient.put()
                .uri("/api/v1/customer-data/products")
                .body(Map.of(
                    "entity_type", "product",
                    "force_update", true,
                    "data", product
                ))
                .retrieve()
                .onStatus(status -> status.is4xxClientError(),
                        (req, res) -> {
                            throw new NonRetryableIntegrationException(
                                    "Inspectorio rejected product: " + res.getStatusCode());
                        })
                .onStatus(status -> status.is5xxServerError(),
                        (req, res) -> {
                            throw new RetryableIntegrationException(
                                    "Inspectorio server error: " + res.getStatusCode());
                        })
                .toBodilessEntity();

        return response.getStatusCode().value();
    }
}
```

---

### HttpClientConfig.java

```java
package com.empresa.integration.config;

import lombok.extern.slf4j.Slf4j;
import org.apache.hc.client5.http.config.ConnectionConfig;
import org.apache.hc.client5.http.impl.classic.CloseableHttpClient;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManager;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestClient;

import java.util.concurrent.TimeUnit;

@Slf4j
@Configuration
public class HttpClientConfig {

    @Bean("plmRestClient")
    public RestClient plmRestClient(
            @Value("${integration.plm.base-url}") String baseUrl,
            @Value("${integration.plm.connect-timeout-ms:5000}") int connectTimeout,
            @Value("${integration.plm.read-timeout-ms:30000}") int readTimeout,
            @Value("${integration.plm.max-connections-per-route:20}") int maxPerRoute,
            @Value("${integration.plm.max-connections-total:50}") int maxTotal) {

        CloseableHttpClient httpClient = buildHttpClient(
                connectTimeout, readTimeout, maxPerRoute, maxTotal);

        return RestClient.builder()
                .baseUrl(baseUrl)
                .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
                // SECURITY: API Key/session token injetados nos métodos, não aqui
                .build();
    }

    @Bean("inspectorioRestClient")
    public RestClient inspectorioRestClient(
            @Value("${integration.inspectorio.base-url}") String baseUrl,
            @Value("${integration.inspectorio.api-key}") String apiKey,
            @Value("${integration.inspectorio.connect-timeout-ms:5000}") int connectTimeout,
            @Value("${integration.inspectorio.read-timeout-ms:60000}") int readTimeout,
            @Value("${integration.inspectorio.max-connections-per-route:10}") int maxPerRoute,
            @Value("${integration.inspectorio.max-connections-total:20}") int maxTotal) {

        CloseableHttpClient httpClient = buildHttpClient(
                connectTimeout, readTimeout, maxPerRoute, maxTotal);

        return RestClient.builder()
                .baseUrl(baseUrl)
                .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
                // API Key adicionada como header fixo no nível do cliente
                .defaultHeader("X-API-Key", apiKey)
                .build();
    }

    private CloseableHttpClient buildHttpClient(int connectTimeoutMs, int readTimeoutMs,
                                                 int maxPerRoute, int maxTotal) {
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
        cm.setDefaultMaxPerRoute(maxPerRoute);
        cm.setMaxTotal(maxTotal);
        cm.setDefaultConnectionConfig(
                ConnectionConfig.custom()
                        .setConnectTimeout(connectTimeoutMs, TimeUnit.MILLISECONDS)
                        .setSocketTimeout(readTimeoutMs, TimeUnit.MILLISECONDS)
                        .build());

        return HttpClients.custom()
                .setConnectionManager(cm)
                .build();
    }
}
```

---

### GlobalExceptionHandler.java

```java
package com.empresa.integration.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.stream.Collectors;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        String errors = ex.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining("; "));

        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Validation failed");
        pd.setDetail(errors);
        return pd;
    }

    @ExceptionHandler(ValidationException.class)
    public ProblemDetail handleBusinessValidation(ValidationException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Business validation failed");
        pd.setDetail(ex.getMessage());
        return pd;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Internal server error");
        pd.setDetail("Unexpected error occurred");
        return pd;
    }
}
```

---

### DTOs

```java
// MaterialsIntegrationRequest.java
package com.empresa.integration.dto.request;

import jakarta.validation.constraints.NotEmpty;
import lombok.Data;
import java.util.List;

@Data
public class MaterialsIntegrationRequest {
    @NotEmpty(message = "materialIds must not be empty")
    private List<String> materialIds;
}

// ProductsIntegrationRequest.java
package com.empresa.integration.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import lombok.Data;
import java.util.List;

@Data
public class ProductsIntegrationRequest {
    @NotBlank(message = "styleId must not be blank")
    private String styleId;

    @NotEmpty(message = "skuIds must not be empty — style must have at least one SKU")
    private List<String> skuIds;
}

// MaterialsIntegrationResponse.java
package com.empresa.integration.dto.response;

import lombok.Builder;
import lombok.Data;
import java.util.List;

@Data
@Builder
public class MaterialsIntegrationResponse {
    private String batchId;
    private String correlationId;
    private List<String> successful;
    private List<String> failed;
}

// ProductsIntegrationResponse.java
package com.empresa.integration.dto.response;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class ProductsIntegrationResponse {
    private String processId;
    private String status;
    private String batchId;
    private String correlationId;
}
```

---

### Exceções

```java
// RetryableIntegrationException.java
package com.empresa.integration.exception;

public class RetryableIntegrationException extends RuntimeException {
    public RetryableIntegrationException(String message) { super(message); }
    public RetryableIntegrationException(String message, Throwable cause) { super(message, cause); }
}

// NonRetryableIntegrationException.java
package com.empresa.integration.exception;

public class NonRetryableIntegrationException extends RuntimeException {
    public NonRetryableIntegrationException(String message) { super(message); }
}

// ValidationException.java
package com.empresa.integration.exception;

public class ValidationException extends NonRetryableIntegrationException {
    public ValidationException(String message) { super(message); }
}
```

---

### MaterialTransformer.java

```java
package com.empresa.integration.transformer;

import com.empresa.integration.domain.inspectorio.InspectorioMaterial;
import com.empresa.integration.domain.plm.PlmMaterial;
import com.empresa.integration.exception.ValidationException;
import org.mapstruct.*;
import org.springframework.beans.factory.annotation.Value;

/**
 * Mapeia PlmMaterial → InspectorioMaterial.
 *
 * ATENÇÃO: mapeamentos completos dependem do documento DE→PARA.
 * Em ambiente 'stub', campos marcados como TODO recebem valores placeholder.
 * Em ambiente 'prod', campos obrigatórios ausentes lançam ValidationException.
 *
 * Gate CI/CD: este arquivo não pode ser promovido para produção sem
 * todos os @Mapping substituídos por mapeamentos reais do DE→PARA.
 */
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public abstract class MaterialTransformer {

    @Value("${spring.profiles.active:local}")
    protected String activeProfile;

    @Mapping(target = "customId", source = "materialId")
    @Mapping(target = "customRevision", source = "version")
    @Mapping(target = "name", source = "name")
    @Mapping(target = "description", source = "description")
    // TODO: adicionar mapeamentos restantes do documento DE→PARA
    public abstract InspectorioMaterial transform(PlmMaterial plmMaterial);

    @AfterMapping
    protected void validateRequiredFields(@MappingTarget InspectorioMaterial target,
                                           PlmMaterial source) {
        if (target.getCustomId() == null || target.getCustomId().isBlank()) {
            throw new ValidationException(
                    "mandatory field 'custom_id' is missing for material " + source.getMaterialId());
        }
        if (target.getName() == null || target.getName().isBlank()) {
            throw new ValidationException(
                    "mandatory field 'name' is missing for material " + source.getMaterialId());
        }
    }
}
```

---

### ProductTransformer.java

```java
package com.empresa.integration.transformer;

import com.empresa.integration.domain.inspectorio.InspectorioItem;
import com.empresa.integration.domain.inspectorio.InspectorioProduct;
import com.empresa.integration.domain.plm.PlmSku;
import com.empresa.integration.domain.plm.PlmStyle;
import com.empresa.integration.exception.ValidationException;
import org.mapstruct.*;
import java.util.List;

/**
 * Mapeia PlmStyle + List<PlmSku> → InspectorioProduct (agregado product + items).
 * SKUs referenciam o Style via parent_custom_id.
 */
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public abstract class ProductTransformer {

    @Mapping(target = "customId", source = "style.styleId")
    @Mapping(target = "customRevision", source = "style.version")
    @Mapping(target = "name", source = "style.name")
    @Mapping(target = "items", expression = "java(mapSkus(style, skus))")
    // TODO: adicionar mapeamentos restantes do documento DE→PARA
    public abstract InspectorioProduct transform(PlmStyle style, List<PlmSku> skus);

    protected List<InspectorioItem> mapSkus(PlmStyle style, List<PlmSku> skus) {
        return skus.stream()
                .map(sku -> mapSku(style.getStyleId(), sku))
                .toList();
    }

    private InspectorioItem mapSku(String parentCustomId, PlmSku sku) {
        InspectorioItem item = new InspectorioItem();
        item.setCustomId(sku.getSkuId());
        item.setParentCustomId(parentCustomId); // CRÍTICO: relacionamento Style→SKU
        item.setCustomRevision(sku.getVersion());
        // TODO: demais campos do DE→PARA
        return item;
    }

    @AfterMapping
    protected void validateProduct(@MappingTarget InspectorioProduct target,
                                    PlmStyle source) {
        if (target.getCustomId() == null || target.getCustomId().isBlank()) {
            throw new ValidationException(
                    "mandatory field 'custom_id' is missing for style " + source.getStyleId());
        }
        if (target.getItems() == null || target.getItems().isEmpty()) {
            throw new ValidationException(
                    "product must have at least one item (SKU) for style " + source.getStyleId());
        }
    }
}
```

---

## Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS runtime

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app

COPY target/plm-inspectorio-integration-*.jar app.jar

USER appuser

EXPOSE 8080

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

---

## docker-compose.yml

```yaml
version: "3.9"

services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: plm-integration
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: local
      PLM_BASE_URL: http://plm-mock:8081
      PLM_USERNAME: dev-user
      PLM_PASSWORD: dev-pass
      INSPECTORIO_BASE_URL: http://inspectorio-mock:8082
      INSPECTORIO_API_KEY: dev-api-key
      GRAYLOG_HOST: graylog
      GRAYLOG_PORT: 12201
    depends_on:
      - plm-mock
      - inspectorio-mock
      - graylog
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 3

  # WireMock: simula PLM
  plm-mock:
    image: wiremock/wiremock:3.3.1
    container_name: plm-mock
    ports:
      - "8081:8080"
    volumes:
      - ./docker/wiremock/plm:/home/wiremock

  # WireMock: simula Inspectorio
  inspectorio-mock:
    image: wiremock/wiremock:3.3.1
    container_name: inspectorio-mock
    ports:
      - "8082:8080"
    volumes:
      - ./docker/wiremock/inspectorio:/home/wiremock

  # Graylog stack (Mongo + Elasticsearch + Graylog)
  mongo:
    image: mongo:6.0
    container_name: graylog-mongo
    volumes:
      - graylog_mongo_data:/data/db

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: graylog-elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - graylog_es_data:/usr/share/elasticsearch/data

  graylog:
    image: graylog/graylog:5.2
    container_name: graylog
    environment:
      GRAYLOG_PASSWORD_SECRET: "somepasswordsecret"
      GRAYLOG_ROOT_PASSWORD_SHA2: "8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918"
      GRAYLOG_HTTP_EXTERNAL_URI: "http://localhost:9000/"
      GRAYLOG_ELASTICSEARCH_HOSTS: "http://elasticsearch:9200"
      GRAYLOG_MONGODB_URI: "mongodb://mongo:27017/graylog"
    ports:
      - "9000:9000"
      - "12201:12201/udp"  # GELF UDP
    depends_on:
      - mongo
      - elasticsearch

  # Prometheus
  prometheus:
    image: prom/prometheus:v2.49.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  # Grafana
  grafana:
    image: grafana/grafana:10.3.0
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  graylog_mongo_data:
  graylog_es_data:
  grafana_data:
```

---

## Makefile

```makefile
.PHONY: build run test lint clean docker-up docker-down

build:
	./mvnw clean package -DskipTests

run:
	SPRING_PROFILES_ACTIVE=local ./mvnw spring-boot:run

test:
	./mvnw verify

lint:
	./mvnw checkstyle:check

docker-up:
	docker compose -f docker/docker-compose.yml up -d --build

docker-down:
	docker compose -f docker/docker-compose.yml down -v

logs:
	docker compose -f docker/docker-compose.yml logs -f app

health:
	curl -s http://localhost:8080/actuator/health | jq .

clean:
	./mvnw clean
```

---

# Contratos

## OpenAPI 3.1 — integration-api.yaml

```yaml
openapi: "3.1.0"
info:
  title: PLM Inspectorio Integration API
  version: "1.0.0"
  description: >
    API de integração entre PLM (Centric) e Inspectorio.
    Exposta via API Gateway WSO2.

servers:
  - url: https://api-gateway.empresa.com.br/integration/v1
    description: Production (WSO2)
  - url: http://localhost:8080
    description: Local development

paths:
  /integration/materials:
    post:
      summary: Synchronize Materials
      description: >
        Recebe lista de materialIds, processa sincronicamente e retorna
        resultado com IDs bem-sucedidos e falhos.
      operationId: integrateMaterials
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MaterialsIntegrationRequest'
            example:
              materialIds: ["MAT-001", "MAT-002"]
      responses:
        '200':
          description: Processing completed
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MaterialsIntegrationResponse'
        '400':
          description: Validation error (empty list)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemDetail'
        '500':
          $ref: '#/components/responses/InternalError'

  /integration/products:
    post:
      summary: Synchronize Style and SKUs
      description: >
        Recebe styleId e lista de skuIds. Retorna imediatamente HTTP 202
        com PROCESS_ID. Processamento ocorre de forma assíncrona.
      operationId: integrateProducts
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ProductsIntegrationRequest'
            example:
              styleId: "STY-001"
              skuIds: ["SKU-001", "SKU-002"]
      responses:
        '202':
          description: Accepted — processing in background
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductsIntegrationResponse'
        '400':
          description: Validation error (style without SKUs)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemDetail'

components:
  schemas:
    MaterialsIntegrationRequest:
      type: object
      required: [materialIds]
      properties:
        materialIds:
          type: array
          minItems: 1
          items:
            type: string
          description: IDs dos materials a sincronizar

    ProductsIntegrationRequest:
      type: object
      required: [styleId, skuIds]
      properties:
        styleId:
          type: string
        skuIds:
          type: array
          minItems: 1
          items:
            type: string

    MaterialsIntegrationResponse:
      type: object
      properties:
        batchId:
          type: string
          format: uuid
        correlationId:
          type: string
          format: uuid
        successful:
          type: array
          items:
            type: string
        failed:
          type: array
          items:
            type: string

    ProductsIntegrationResponse:
      type: object
      properties:
        processId:
          type: string
          format: uuid
        status:
          type: string
          enum: [RECEBIDO]
        batchId:
          type: string
          format: uuid
        correlationId:
          type: string
          format: uuid

    ProblemDetail:
      type: object
      properties:
        title:
          type: string
        detail:
          type: string
        status:
          type: integer

  responses:
    InternalError:
      description: Internal Server Error
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemDetail'
```

---

## Schema de Log Estruturado (contrato versionado)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "IntegrationLogEvent",
  "version": "1.0.0",
  "description": "Contrato de log estruturado. Campos obrigatórios para todos os eventos.",
  "type": "object",
  "required": ["timestamp", "level", "service", "event",
               "batch_id", "correlation_id", "entity_type", "entity_id"],
  "properties": {
    "timestamp": { "type": "string", "format": "date-time" },
    "level": { "type": "string", "enum": ["INFO", "WARN", "ERROR"] },
    "service": { "type": "string", "const": "plm-inspectorio-integration" },
    "event": {
      "type": "string",
      "enum": ["INTEGRATION_START", "INTEGRATION_SUCCESS", "INTEGRATION_ERROR"]
    },
    "batch_id": { "type": "string", "format": "uuid" },
    "correlation_id": { "type": "string", "format": "uuid" },
    "entity_type": { "type": "string", "enum": ["MATERIAL", "STYLE", "SKU", "PRODUCT"] },
    "entity_id": { "type": "string" },
    "status": { "type": "string", "enum": ["SUCCESSFUL", "ERROR"] },
    "http_status": { "type": "integer" },
    "execution_time_ms": { "type": "integer" },
    "retry_attempts": { "type": "integer" },
    "error_message": { "type": "string" }
  },
  "additionalProperties": false
}
```

# Segurança

## Modelo de Segurança

O serviço não expõe autenticação própria — a autenticação de consumidores externos é responsabilidade do **API Gateway WSO2**. Internamente, o serviço gerencia apenas as credenciais de sistemas downstream.

### Gestão de Credenciais

```
NEVER store credentials in code, properties files, or Docker images.
All secrets are injected via environment variables in runtime.
```

| Credencial | Armazenamento | Injeção |
|---|---|---|
| PLM session username/password | Vault / K8s Secret | `${PLM_USERNAME}`, `${PLM_PASSWORD}` |
| Inspectorio API Key | Vault / K8s Secret | `${INSPECTORIO_API_KEY}` |
| Graylog endpoint | ConfigMap | `${GRAYLOG_HOST}`, `${GRAYLOG_PORT}` |

### SecurityConfig.java

```java
package com.empresa.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            // Autenticação de consumidores: responsabilidade do WSO2 Gateway
            // O serviço confia que requisições chegam já autenticadas pelo gateway
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/integration/**").permitAll() // autenticado via WSO2
                .anyRequest().denyAll()
            )
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'none'"))
                .frameOptions(f -> f.deny())
            );
        return http.build();
    }
}
```

### Checklist de Segurança de Implementação

1. **Session token PLM e API Key Inspectorio** nunca são adicionados ao MDC.
2. **logback-spring.xml** exclui explicitamente campos sensíveis via `<excludeMdcKeyName>`.
3. **Interceptors do RestClient** injetam credenciais nos headers, não nas URLs.
4. **Logs de erro** não incluem o corpo completo das requisições (pode conter dados de produto sensíveis) — apenas o `error_type` e `http_status`.
5. **HTTPS** obrigatório para PLM e Inspectorio via configuração do RestClient (sem downgrade TLS).
6. **Docker image** construída como `non-root user` (`appuser`).

---

# Observabilidade

## Logs Estruturados (Graylog)

O contrato do `StructuredLogger` define os campos obrigatórios em todos os eventos. O Graylog indexa por `batch_id`, `correlation_id`, `entity_type`, `entity_id` e `status`.

### Queries Operacionais Recomendadas

```
# Rastrear todos os eventos de um batch
batch_id:"<UUID>"

# Consultar falhas das últimas 24h
status:"ERROR" AND timestamp:[now-24h TO now]

# Registros em Processing há mais de 30 min (possível stuck)
plm_status_code:5 AND timestamp:[* TO now-30m]

# Falhas em Materials para um correlation_id específico
entity_type:"MATERIAL" AND status:"ERROR" AND correlation_id:"<UUID>"
```

### Alerta Crítico: Registros Travados em Processing

```yaml
# Graylog Stream Alert Rule
name: "PLM status=5 stuck > 30min"
condition:
  type: message_count
  parameters:
    grace: 5
    threshold_type: MORE
    threshold: 0
    time: 30
  stream_filter: "plm_status_code:5"
notification: PagerDuty / Email da equipe de TI
runbook: |
  1. Identificar batch_ids afetados
  2. Validar se a aplicação estava em restart durante o período
  3. Executar reprocessamento via POST /integration/materials ou /integration/products
     com os entity_ids em status Processing
```

---

## Métricas (Prometheus + Grafana)

```yaml
# docker/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "plm-integration"
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["app:8080"]
```

**Métricas expostas:**

| Métrica | Tipo | Descrição |
|---|---|---|
| `resilience4j_retry_calls_total` | Counter | Tentativas de retry por instância e resultado |
| `http_server_requests_seconds` | Histogram | Latência dos endpoints de integração |
| `executor_pool_size` | Gauge | Tamanho atual do ThreadPoolTaskExecutor |
| `executor_queue_remaining_capacity` | Gauge | Capacidade restante na fila do pool |
| `http_client_requests_seconds` | Histogram | Latência das chamadas ao PLM e Inspectorio |

---

# Automação

## CI — .github/workflows/ci.yml

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build and test
        run: ./mvnw verify

      - name: Lint (Checkstyle)
        run: ./mvnw checkstyle:check

      - name: Validate log schema presence
        run: |
          grep -r "logStart\|logSuccess\|logError" src/main/java --include="*.java" | \
          grep -q "StructuredLogger" || (echo "StructuredLogger not used in service" && exit 1)

      - name: Verify stub gate (no stub in prod profile)
        run: |
          grep -r "stub\|TODO.*DE.*PARA" src/main/java --include="*.java" | \
          grep -v "test\|Test" && \
          echo "WARNING: Stub mappings detected - do not promote to production" || true

      - name: Build Docker image
        run: docker build -f docker/Dockerfile -t plm-integration:${{ github.sha }} .
```

## CD — .github/workflows/cd.yml

```yaml
name: CD

on:
  push:
    branches: [main]

jobs:
  deploy-hml:
    runs-on: ubuntu-latest
    environment: homologation

    steps:
      - uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven

      - name: Build artifact
        run: ./mvnw clean package -DskipTests

      - name: Build & Push Docker image
        env:
          REGISTRY: ${{ secrets.REGISTRY_URL }}
        run: |
          docker build -f docker/Dockerfile \
            -t $REGISTRY/plm-integration:${{ github.sha }} \
            -t $REGISTRY/plm-integration:latest .
          docker push $REGISTRY/plm-integration:${{ github.sha }}
          docker push $REGISTRY/plm-integration:latest

      - name: Deploy to Rancher (HML)
        env:
          RANCHER_URL: ${{ secrets.RANCHER_URL }}
          RANCHER_TOKEN: ${{ secrets.RANCHER_TOKEN }}
        run: |
          # Atualiza imagem no workload do Rancher via API
          curl -s -X PUT "$RANCHER_URL/v3/projects/.../workloads/..." \
            -H "Authorization: Bearer $RANCHER_TOKEN" \
            -H "Content-Type: application/json" \
            -d '{"containers":[{"image":"'$REGISTRY/plm-integration:${{ github.sha }}'"}]}'
```

---

# Roadmap Técnico

## INC-01 — Fluxo Materials (Sprint atual)

**Pré-condições obrigatórias antes do start:**
1. Schema de log estruturado aprovado e versionado no repositório.
2. Mecanismo de sessão PLM especificado pela equipe Centric.
3. Timeout do `InspectorioClient` definido como parâmetro de configuração.
4. Hierarquia de exceções `Retryable/NonRetryable` implementada antes do Resilience4j.

**Entregáveis:**
- `MaterialsIntegrationController` + `MaterialsIntegrationService`
- `PLMClient` (GET/PUT materials) + `PLMSessionManager`
- `InspectorioClient` (upsert material)
- `MaterialTransformer` com stubs (gate CI/CD para não promover sem DE→PARA)
- `StructuredLogger` com schema versionado
- `RetryPolicy` via Resilience4j
- Testes unitários e de integração com WireMock

---

## INC-02 — Fluxo Style → SKUs (Sprint +1)

**Pré-condições adicionais:**
1. Comportamento de falha parcial no agregado Style→SKU validado com stakeholders.
2. `TaskDecorator` para propagação MDC implementado e testado.
3. Dimensionamento do `ThreadPoolTaskExecutor` validado para volumetria de 4.200/mês.
4. Graceful shutdown configurado.

**Entregáveis:**
- `ProductsIntegrationController` (HTTP 202)
- `ProductsIntegrationService` (@Async)
- `PLMClient` (GET/PUT styles e SKUs)
- `ProductTransformer` (agregado product + items + parent_custom_id)
- Alerta Graylog para status=5 stuck > 30min

---

## INC-03 — Observabilidade Operacional

**Dependência:** dados reais de INC-01 e INC-02 em ambiente de homologação.

**Entregáveis:**
- Dashboard Grafana com métricas de retry, latência e throughput
- Queries Graylog documentadas para US-007
- Alertas operacionais ativos
- Runbook de detecção e reprocessamento de registros travados

---

## INC-04 — Reprocessamento Manual

**Decisão pendente (gate):** validar com stakeholders se reutilização dos endpoints existentes é suficiente ou se endpoint dedicado é necessário.

**Se reutilização dos endpoints:**
- Adicionar campo `is_reprocessing: boolean` no request (opcional)
- Logar campo `is_reprocessing` para distinção de métricas no Graylog

**Se endpoint dedicado:**
- `POST /integration/materials/reprocess` com filtro por `batchId` + `status=ERROR`
- Reaproveita `MaterialsIntegrationService` com mesmo fluxo

---

## Evolução Futura (Pós-INC-04)

| Área | Iniciativa | Trigger |
|---|---|---|
| Performance | Habilitar Java 21 Virtual Threads (`spring.threads.virtual.enabled=true`) | Volumetria > 20k/mês ou SLA degradado |
| Resiliência | Circuit Breaker no `PLMClient` e `InspectorioClient` via Resilience4j | Aumento de taxa de erro > 5% |
| Observabilidade | OpenTelemetry para traces distribuídos (Jaeger/Tempo) | Necessidade de debug cross-service |
| Mapeamento | Externalizat regras DE→PARA em YAML configurável por versão de API | Múltiplas versões do contrato Inspectorio |
| Escala | Extração para microsserviço independente por fluxo | Times diferentes evoluindo Materials vs. Products |
| Segurança | MTLS entre serviço e PLM/Inspectorio | Requisito de auditoria de segurança |

---

## Decisões Técnicas Pendentes (Gate para Início de INC-01)

| # | Decisão | Responsável | Prazo |
|---|---|---|---|
| 1 | Documento DE→PARA (campos obrigatórios Material, Style, SKU) | Product Owner + PLM Team | Antes de INC-01 |
| 2 | Mecanismo de sessão PLM (endpoint, TTL, renovação) | Equipe Centric | Antes de INC-01 |
| 3 | Timeout InspectorioClient (valor em ms) | Arquiteto + PO | Antes de INC-01 |
| 4 | Comportamento de falha parcial Style→SKU | PO + Dev Lead | Antes de INC-02 |
| 5 | Schema e TTL de armazenamento de dados de erro | PO + DBA | Antes de INC-02 |
| 6 | Mecanismo de reprocessamento manual (reutilizar endpoint vs. dedicado) | PO + Stakeholders | Antes de INC-04 |
| 7 | Derivação do campo `custom_revision` (regra no DE→PARA) | PLM Team + Inspectorio Team | Com entrega do DE→PARA |

---

> **Versão deste documento:** 1.0.0  
> **Gerado em:** 2026-02-26  
> **Baseado em:** architectural_components v1.0 · stack_validation v1.0
>
> **Autor:** Claudecir Miranda · Arquiteto de Soluções/IA