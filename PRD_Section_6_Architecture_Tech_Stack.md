# Section 6: Java/Spring Boot Architecture & Tech Stack

## 6.1 Technology Stack Overview

### Core Technologies

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Language** | Java | 26 | Core programming language |
| **Framework** | Spring Boot | 3.4.x (Latest LTS) | Application framework |
| **Build Tool** | Maven | 3.9+ | Dependency management |
| **Database** | SQLite | 3.x | Static game data storage |
| **ORM** | Spring Data JPA + Hibernate | 6.x | Database access layer |
| **GUI** | JavaFX | 21+ | User interface |
| **Packet Capture** | Pcap4j | 1.8.2 | Network packet sniffing |
| **Logging** | SLF4J + Logback | 2.x | Logging framework |
| **Testing** | JUnit 5 + Mockito | 5.10+ | Unit testing |

---

## 6.2 Project Structure

### Maven Multi-Module Project (Recommended)

```
dofus-fm-assistant/
├── pom.xml                          # Parent POM
├── fm-core/                         # Core business logic module
│   ├── pom.xml
│   └── src/main/java/com/dofus/fm/
│       ├── domain/                  # Domain models (Item, Rune, Line)
│       ├── service/                 # Business services
│       ├── repository/              # Data repositories
│       └── entity/                  # JPA entities
├── fm-network/                      # Network packet capture module
│   ├── pom.xml
│   └── src/main/java/com/dofus/fm/network/
│       ├── capture/                 # Packet capture
│       ├── protocol/                # Packet parsing
│       └── parsers/                 # Message parsers
├── fm-ui/                           # JavaFX UI module
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/dofus/fm/ui/
│       │   ├── controller/          # FXML controllers
│       │   └── FMApplication.java   # JavaFX entry point
│       └── resources/
│           ├── fxml/                # FXML layouts
│           └── styles/              # CSS stylesheets
└── fm-app/                          # Main application module
    ├── pom.xml
    └── src/main/
        ├── java/com/dofus/fm/
        │   └── DofusFMAssistantApplication.java
        └── resources/
            ├── application.yml      # Spring configuration
            └── logback.xml          # Logging configuration
```

---

## 6.3 Detailed Package Structure

### Core Module (`fm-core`)

```
com.dofus.fm
├── domain                           # Domain models (not JPA entities)
│   ├── Item.java
│   ├── Rune.java
│   ├── Line.java
│   └── NegativeEffectMapping.java
├── entity                           # JPA entities (database tables)
│   ├── ItemEntity.java
│   ├── Effect.java
│   ├── EffectLine.java
│   ├── Description.java
│   └── ItemEffectLine.java
├── repository                       # Spring Data JPA repositories
│   ├── ItemRepository.java
│   ├── EffectRepository.java
│   └── EffectLineRepository.java
├── service                          # Business logic services
│   ├── FMCalculationService.java
│   ├── FMSessionService.java
│   ├── ItemFactory.java
│   ├── RuneFactory.java
│   └── DisplayService.java
├── dto                              # Data Transfer Objects
│   ├── ItemDTO.java
│   └── RuneDTO.java
└── config                           # Configuration classes
    └── DatabaseConfig.java
```

### Network Module (`fm-network`)

```
com.dofus.fm.network
├── capture                          # Packet capture
│   ├── PacketCaptureService.java
│   ├── PacketListener.java
│   └── DofusPacketExtractor.java
├── protocol                         # Protocol utilities
│   ├── DofusPacket.java
│   ├── DofusPacketType.java
│   ├── VarIntReader.java
│   └── PacketReader.java
├── parsers                          # Message parsers
│   ├── PacketParser.java           # Interface
│   ├── ExchangeObjectParser.java
│   └── CraftResultParser.java
├── types                            # Parsed message DTOs
│   ├── ParsedPacket.java
│   ├── ExchangeObjectMessage.java
│   ├── CraftResultMessage.java
│   └── Effect.java
└── config
    └── NetworkConfig.java
```

### UI Module (`fm-ui`)

```
com.dofus.fm.ui
├── FMApplication.java               # JavaFX Application entry
├── controller
│   ├── MainController.java
│   └── LineViewModel.java
└── config
    └── JavaFXConfig.java
```

---

## 6.4 Spring Boot Configuration

### Parent POM (`pom.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.0</version> <!-- Latest LTS -->
        <relativePath/>
    </parent>

    <groupId>com.dofus</groupId>
    <artifactId>fm-assistant</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>Dofus FM Assistant</name>
    <description>Forgemagie helper tool for Dofus MMORPG</description>

    <properties>
        <java.version>26</java.version>
        <maven.compiler.source>26</maven.compiler.source>
        <maven.compiler.target>26</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- Dependency versions -->
        <javafx.version>21.0.5</javafx.version>
        <pcap4j.version>1.8.2</pcap4j.version>
        <sqlite.version>3.46.1.3</sqlite.version>
        <lombok.version>1.18.34</lombok.version>
    </properties>

    <modules>
        <module>fm-core</module>
        <module>fm-network</module>
        <module>fm-ui</module>
        <module>fm-app</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <!-- Internal modules -->
            <dependency>
                <groupId>com.dofus</groupId>
                <artifactId>fm-core</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.dofus</groupId>
                <artifactId>fm-network</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.dofus</groupId>
                <artifactId>fm-ui</artifactId>
                <version>${project.version}</version>
            </dependency>

            <!-- JavaFX -->
            <dependency>
                <groupId>org.openjfx</groupId>
                <artifactId>javafx-controls</artifactId>
                <version>${javafx.version}</version>
            </dependency>
            <dependency>
                <groupId>org.openjfx</groupId>
                <artifactId>javafx-fxml</artifactId>
                <version>${javafx.version}</version>
            </dependency>

            <!-- Pcap4j for packet capture -->
            <dependency>
                <groupId>org.pcap4j</groupId>
                <artifactId>pcap4j-core</artifactId>
                <version>${pcap4j.version}</version>
            </dependency>
            <dependency>
                <groupId>org.pcap4j</groupId>
                <artifactId>pcap4j-packetfactory-static</artifactId>
                <version>${pcap4j.version}</version>
            </dependency>

            <!-- SQLite JDBC driver -->
            <dependency>
                <groupId>org.xerial</groupId>
                <artifactId>sqlite-jdbc</artifactId>
                <version>${sqlite.version}</version>
            </dependency>

            <!-- Lombok -->
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
                <scope>provided</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>26</source>
                    <target>26</target>
                    <enablePreview>true</enablePreview>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

### Core Module POM (`fm-core/pom.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <groupId>com.dofus</groupId>
        <artifactId>fm-assistant</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>fm-core</artifactId>

    <dependencies>
        <!-- Spring Boot Starter Data JPA -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- SQLite JDBC -->
        <dependency>
            <groupId>org.xerial</groupId>
            <artifactId>sqlite-jdbc</artifactId>
        </dependency>

        <!-- Hibernate SQLite dialect -->
        <dependency>
            <groupId>org.hibernate.orm</groupId>
            <artifactId>hibernate-community-dialects</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

---

### Network Module POM (`fm-network/pom.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <groupId>com.dofus</groupId>
        <artifactId>fm-assistant</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>fm-network</artifactId>

    <dependencies>
        <!-- Pcap4j -->
        <dependency>
            <groupId>org.pcap4j</groupId>
            <artifactId>pcap4j-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.pcap4j</groupId>
            <artifactId>pcap4j-packetfactory-static</artifactId>
        </dependency>

        <!-- Spring Context (for @Service, @Component) -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
        </dependency>

        <!-- SLF4J -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

---

### Application Configuration (`application.yml`)

```yaml
spring:
  application:
    name: dofus-fm-assistant

  datasource:
    url: jdbc:sqlite:database.sqlite
    driver-class-name: org.sqlite.JDBC
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
      connection-timeout: 30000

  jpa:
    database-platform: org.hibernate.community.dialect.SQLiteDialect
    hibernate:
      ddl-auto: validate  # Don't auto-create schema
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true

# Network packet capture configuration
dofus:
  network:
    server-ip: 213.248.126.61
    capture-filter: "host 213.248.126.61"
    network-interface: auto  # Auto-detect, or specify "eth0", "en0", etc.

# Logging
logging:
  level:
    root: INFO
    com.dofus.fm: DEBUG
    org.hibernate.SQL: DEBUG
    org.pcap4j: INFO
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/fm-assistant.log
    max-size: 10MB
    max-history: 7
```

---

## 6.5 Application Entry Point

### Main Application Class

```java
package com.dofus.fm;

import com.dofus.fm.ui.FMApplication;
import javafx.application.Application;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DofusFMAssistantApplication {

    public static void main(String[] args) {
        // Launch JavaFX application (which will initialize Spring context)
        Application.launch(FMApplication.class, args);
    }
}
```

---

## 6.6 Database Configuration

### SQLite Dialect Configuration

```java
package com.dofus.fm.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@Configuration
@EnableJpaRepositories(basePackages = "com.dofus.fm.repository")
public class DatabaseConfig {
    // SQLite-specific configurations if needed
}
```

### Handling SQLite with Hibernate

**Important**: SQLite has limitations with Hibernate:
- No `AUTO_INCREMENT` for primary keys (use `AUTOINCREMENT` in SQLite)
- Foreign keys must be enabled explicitly

**Workaround**: Use `GenerationType.IDENTITY` or `GenerationType.SEQUENCE` carefully.

**Alternative**: Since the database is read-only after initialization, most JPA features aren't needed. Consider using `JdbcTemplate` for simpler read-only access:

```java
@Repository
public class ItemRepository {

    private final JdbcTemplate jdbcTemplate;

    public ItemRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public Optional<ItemBasicInfo> findItemBasicInfo(Integer itemId) {
        String sql = """
            SELECT i.id, i.level, d.description_text
            FROM item i
            JOIN description d ON i.description_id = d.id
            WHERE i.id = ?
            """;

        return jdbcTemplate.query(sql, rs -> {
            if (rs.next()) {
                return Optional.of(new ItemBasicInfo(
                    rs.getInt("id"),
                    rs.getInt("level"),
                    rs.getString("description_text")
                ));
            }
            return Optional.empty();
        }, itemId);
    }
}
```

---

## 6.7 Asynchronous Processing

### Packet Capture Service with @Async

```java
package com.dofus.fm.network.capture;

import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class PacketCaptureService {

    private final PacketListener packetListener;
    private volatile boolean running = false;

    public PacketCaptureService(PacketListener packetListener) {
        this.packetListener = packetListener;
    }

    @Async
    public void startCapture() {
        running = true;

        try (PcapHandle handle = openPcapHandle()) {
            handle.setFilter("host 213.248.126.61", BpfCompileMode.OPTIMIZE);

            while (running) {
                Packet packet = handle.getNextPacket();
                if (packet != null) {
                    packetListener.onPacketReceived(packet);
                }
            }
        } catch (Exception e) {
            log.error("Packet capture error", e);
        }
    }

    public void stopCapture() {
        running = false;
    }
}
```

### Enable Async Support

```java
package com.dofus.fm.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;

@Configuration
@EnableAsync
public class AsyncConfig {
    // Optional: Custom thread pool configuration
}
```

---

## 6.8 Logging Configuration

### Logback Configuration (`logback.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="LOG_PATTERN"
              value="%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"/>

    <!-- Console appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- File appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/fm-assistant.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/fm-assistant-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- Logger levels -->
    <logger name="com.dofus.fm" level="DEBUG"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.pcap4j" level="INFO"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

---

## 6.9 Build and Packaging

### Maven Assembly Plugin (Create Executable JAR)

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <mainClass>com.dofus.fm.DofusFMAssistantApplication</mainClass>
            </configuration>
        </plugin>

        <!-- JavaFX Maven Plugin -->
        <plugin>
            <groupId>org.openjfx</groupId>
            <artifactId>javafx-maven-plugin</artifactId>
            <version>0.0.8</version>
            <configuration>
                <mainClass>com.dofus.fm.ui.FMApplication</mainClass>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Build Commands

```bash
# Build all modules
mvn clean package

# Run application
java -jar fm-app/target/fm-app-1.0.0-SNAPSHOT.jar

# Or use Spring Boot plugin
mvn spring-boot:run
```

---

## 6.10 Dependency Injection Architecture

### Service Dependency Graph

```
DofusFMAssistantApplication
    ↓
FMApplication (JavaFX)
    ↓
├─ MainController
│   └─ DisplayService
│
├─ PacketCaptureService
│   └─ PacketListener
│       └─ DofusPacketParserService
│           └─ FMSessionService
│               ├─ ItemFactory
│               │   ├─ ItemRepository
│               │   └─ EffectRepository
│               ├─ RuneFactory
│               │   └─ ItemRepository
│               ├─ FMCalculationService
│               └─ DisplayService
│                   └─ MainController
```

### Example Service Wiring

```java
@Service
public class FMSessionService {

    private final ItemFactory itemFactory;
    private final RuneFactory runeFactory;
    private final FMCalculationService fmCalculationService;
    private final DisplayService displayService;

    // Constructor injection (recommended)
    @Autowired
    public FMSessionService(
        ItemFactory itemFactory,
        RuneFactory runeFactory,
        FMCalculationService fmCalculationService,
        DisplayService displayService
    ) {
        this.itemFactory = itemFactory;
        this.runeFactory = runeFactory;
        this.fmCalculationService = fmCalculationService;
        this.displayService = displayService;
    }

    // Business methods...
}
```

---

## 6.11 Error Handling Strategy

### Global Exception Handler

```java
package com.dofus.fm.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Component
@Slf4j
public class GlobalExceptionHandler {

    public void handlePacketParsingException(Exception e, byte[] packet) {
        log.error("Failed to parse packet: {}", bytesToHex(packet), e);
        // Optionally show UI alert
    }

    public void handleFMCalculationException(Exception e) {
        log.error("FM calculation error", e);
        // Show error dialog to user
    }

    private String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        for (byte b : bytes) {
            sb.append(String.format("%02X ", b));
        }
        return sb.toString();
    }
}
```

---

## 6.12 Performance Considerations

### 1. Object Pooling for Packets
```java
@Component
public class ByteArrayPool {
    private final Queue<byte[]> pool = new ConcurrentLinkedQueue<>();

    public byte[] acquire(int size) {
        byte[] buffer = pool.poll();
        if (buffer == null || buffer.length < size) {
            return new byte[size];
        }
        return buffer;
    }

    public void release(byte[] buffer) {
        pool.offer(buffer);
    }
}
```

### 2. Caching Database Queries
```java
@Service
public class EffectRepository {

    private final Map<Integer, Effect> effectCache = new ConcurrentHashMap<>();

    @Cacheable("effects")
    public Effect findById(Integer id) {
        return effectCache.computeIfAbsent(id, this::loadFromDatabase);
    }
}
```

### 3. Use CompletableFuture for Async Operations
```java
@Service
public class PacketProcessor {

    @Async
    public CompletableFuture<Void> processPacketAsync(byte[] packet) {
        return CompletableFuture.runAsync(() -> {
            processPacket(packet);
        });
    }
}
```

---

## 6.13 Security Considerations

### 1. Network Capture Permissions
- **Requires root/admin privileges** to capture packets
- Document this requirement for users
- Consider using setcap on Linux:
  ```bash
  sudo setcap cap_net_raw,cap_net_admin=eip /path/to/java
  ```

### 2. Database Security
- SQLite database is read-only after initialization
- No user input goes into database queries
- Use parameterized queries anyway (already done with JPA)

### 3. Packet Validation
```java
public class PacketValidator {

    public boolean isValid(byte[] packet) {
        // Validate packet structure
        if (packet.length < 2) return false;

        int packetId = extractPacketId(packet);
        return DofusPacketType.isInteresting(packetId);
    }
}
```

---

## 6.14 Testing Strategy

### Unit Tests
```java
@SpringBootTest
class FMCalculationServiceTest {
    @Autowired
    private FMCalculationService service;

    @Test
    void testWeightCalculation() {
        // Test logic
    }
}
```

### Integration Tests
```java
@SpringBootTest
@Testcontainers
class PacketParsingIntegrationTest {
    // Test full packet flow
}
```

### UI Tests with TestFX
```java
@ExtendWith(ApplicationExtension.class)
class MainControllerTest {
    // Test UI updates
}
```

---

## 6.15 Key Architecture Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Framework** | Spring Boot 3.4.x | LTS support, DI, modularity |
| **GUI** | JavaFX | Native desktop, closest to Tkinter |
| **Database Access** | JdbcTemplate | Simpler than JPA for read-only DB |
| **Packet Capture** | Pcap4j | Pure Java, cross-platform |
| **Build Tool** | Maven | Better Spring Boot integration |
| **Module Structure** | Multi-module | Separation of concerns |
| **Async Processing** | @Async + CompletableFuture | Non-blocking packet processing |
| **Logging** | SLF4J + Logback | Industry standard |

---

**Next Section Preview:**
Section 7 will provide the complete migration strategy with implementation phases, testing approach, and deployment plan.
