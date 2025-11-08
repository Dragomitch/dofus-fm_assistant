# Implementation Book
## Dofus FM Assistant - Python to Java Migration
## Multi-Agent Task Reference

**Version**: 1.0
**Purpose**: Detailed task specifications for each agent
**Usage**: Each agent refers to their section for step-by-step implementation

---

## How to Use This Book

1. **Find your agent section** (INFRA-01, DATA-01, etc.)
2. **Execute tasks in Wave order** (Wave 1 → Wave 2 → Wave 3 → Wave 4)
3. **Check dependencies** before starting each task
4. **Mark tasks complete** when done
5. **Report blockers** immediately if stuck

### Task Format

Each task includes:
- **Task ID**: Unique identifier
- **Duration**: Estimated time
- **Dependencies**: Tasks that must complete first
- **Blocking**: Whether this blocks other agents
- **Steps**: Detailed implementation steps
- **Validation**: How to verify completion
- **Deliverables**: Expected outputs

---

# AGENT: INFRA-01 (Infrastructure & Setup)

## Wave 1 Tasks

### W1-T1: Create Maven Multi-Module Structure
**Duration**: 1 day
**Dependencies**: None
**Blocking**: ✅ YES - All agents depend on this

#### Steps:
1. Create root directory: `dofus-fm-assistant/`
2. Create parent `pom.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.0</version>
    </parent>

    <groupId>com.dofus</groupId>
    <artifactId>fm-assistant</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <properties>
        <java.version>26</java.version>
        <maven.compiler.source>26</maven.compiler.source>
        <maven.compiler.target>26</maven.compiler.target>
    </properties>

    <modules>
        <module>fm-core</module>
        <module>fm-network</module>
        <module>fm-ui</module>
        <module>fm-app</module>
    </modules>
</project>
```

3. Create module directories:
```bash
mkdir -p fm-core/src/{main,test}/java/com/dofus/fm
mkdir -p fm-network/src/{main,test}/java/com/dofus/fm/network
mkdir -p fm-ui/src/{main,test}/{java/com/dofus/fm/ui,resources}
mkdir -p fm-app/src/{main,test}/{java/com/dofus/fm,resources}
```

4. Create module `pom.xml` files (see PRD Section 6.4)

5. Test compilation:
```bash
mvn clean install
```

#### Validation:
- [ ] `mvn clean install` succeeds
- [ ] All 4 modules created
- [ ] Directory structure matches specification

#### Deliverables:
- `pom.xml` (parent)
- `fm-core/pom.xml`
- `fm-network/pom.xml`
- `fm-ui/pom.xml`
- `fm-app/pom.xml`
- Complete directory structure

---

### W1-T2: Configure Spring Boot 3.4.x
**Duration**: 0.5 day
**Dependencies**: W1-T1
**Blocking**: ✅ YES

#### Steps:
1. Add Spring Boot dependencies to `fm-core/pom.xml`:
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

2. Create `fm-app/src/main/java/com/dofus/fm/DofusFMAssistantApplication.java`:
```java
package com.dofus.fm;

import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DofusFMAssistantApplication {
    public static void main(String[] args) {
        // Will be called by JavaFX
    }
}
```

3. Create `fm-app/src/main/resources/application.yml`:
```yaml
spring:
  application:
    name: dofus-fm-assistant
```

4. Test Spring Boot starts:
```bash
cd fm-app
mvn spring-boot:run
```

#### Validation:
- [ ] Spring Boot starts without errors
- [ ] Application context loads
- [ ] Logs show "Started DofusFMAssistantApplication"

#### Deliverables:
- `DofusFMAssistantApplication.java`
- `application.yml`

---

### W1-T3: Set Up Logging (Logback)
**Duration**: 0.5 day
**Dependencies**: W1-T2
**Blocking**: ❌ NO

#### Steps:
1. Create `fm-app/src/main/resources/logback.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="LOG_PATTERN"
              value="%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

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

    <logger name="com.dofus.fm" level="DEBUG"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

2. Update `application.yml`:
```yaml
logging:
  level:
    root: INFO
    com.dofus.fm: DEBUG
```

3. Test logging:
```java
@SpringBootApplication
public class DofusFMAssistantApplication {
    private static final Logger log = LoggerFactory.getLogger(DofusFMAssistantApplication.class);

    public static void main(String[] args) {
        log.info("Application starting...");
    }
}
```

#### Validation:
- [ ] Logs appear in console
- [ ] `logs/fm-assistant.log` created
- [ ] Log format matches specification

#### Deliverables:
- `logback.xml`
- Updated `application.yml`

---

### W1-T4: Configure JUnit 5 Test Infrastructure
**Duration**: 0.5 day
**Dependencies**: W1-T1
**Blocking**: ✅ YES

#### Steps:
1. Add test dependencies to parent `pom.xml`:
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

2. Create sample test in `fm-core/src/test/java/com/dofus/fm/SmokeTest.java`:
```java
package com.dofus.fm;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SmokeTest {
    @Test
    void testInfrastructure() {
        assertTrue(true, "Test infrastructure working");
    }
}
```

3. Run tests:
```bash
mvn test
```

#### Validation:
- [ ] `mvn test` passes
- [ ] Test reports generated
- [ ] Coverage tools configured (optional)

#### Deliverables:
- Test dependencies configured
- Sample test passing

---

## Wave 2 Tasks

### W2-T1: Configure Async Execution (@Async)
**Duration**: 0.5 day
**Dependencies**: W1-T2
**Blocking**: ❌ NO

#### Steps:
1. Create `fm-core/src/main/java/com/dofus/fm/config/AsyncConfig.java`:
```java
package com.dofus.fm.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.context.annotation.Bean;
import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "packetProcessorExecutor")
    public Executor packetProcessorExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("packet-processor-");
        executor.initialize();
        return executor;
    }
}
```

2. Test async execution:
```java
@Service
public class TestAsyncService {
    @Async("packetProcessorExecutor")
    public CompletableFuture<String> testAsync() {
        return CompletableFuture.completedFuture("Async works");
    }
}
```

#### Validation:
- [ ] Async service runs in separate thread
- [ ] Thread pool created with correct config

#### Deliverables:
- `AsyncConfig.java`

---

### W2-T2: Set Up CI/CD Pipeline (GitHub Actions)
**Duration**: 1 day
**Dependencies**: W1-T1
**Blocking**: ❌ NO

#### Steps:
1. Create `.github/workflows/build.yml`:
```yaml
name: Build and Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Set up JDK 26
      uses: actions/setup-java@v3
      with:
        java-version: '26'
        distribution: 'temurin'

    - name: Build with Maven
      run: mvn clean install

    - name: Run tests
      run: mvn test

    - name: Upload coverage
      uses: codecov/codecov-action@v3
```

2. Test workflow locally (optional):
```bash
act -j build
```

#### Validation:
- [ ] GitHub Actions workflow triggers
- [ ] Build passes
- [ ] Tests run

#### Deliverables:
- `.github/workflows/build.yml`

---

### W2-T3: Configure Build Profiles (dev/prod)
**Duration**: 0.5 day
**Dependencies**: W1-T1
**Blocking**: ❌ NO

#### Steps:
1. Add profiles to parent `pom.xml`:
```xml
<profiles>
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <spring.profiles.active>dev</spring.profiles.active>
        </properties>
    </profile>

    <profile>
        <id>prod</id>
        <properties>
            <spring.profiles.active>prod</spring.profiles.active>
        </properties>
    </profile>
</profiles>
```

2. Create `application-dev.yml` and `application-prod.yml`

3. Test profiles:
```bash
mvn clean install -Pdev
mvn clean install -Pprod
```

#### Validation:
- [ ] Dev profile active by default
- [ ] Prod profile can be activated

#### Deliverables:
- Build profiles configured
- Environment-specific configs

---

### W2-T4: Set Up Maven Assembly for JAR Packaging
**Duration**: 1 day
**Dependencies**: W1-T1
**Blocking**: ❌ NO (but needed for W4-T9)

#### Steps:
1. Add plugin to `fm-app/pom.xml`:
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
    </plugins>
</build>
```

2. Test packaging:
```bash
cd fm-app
mvn clean package
java -jar target/fm-app-1.0.0-SNAPSHOT.jar
```

#### Validation:
- [ ] JAR created
- [ ] All dependencies included
- [ ] JAR is executable

#### Deliverables:
- Executable JAR configuration

---

### W2-T5: Document Project Structure
**Duration**: 0.5 day
**Dependencies**: W1-T1
**Blocking**: ❌ NO

#### Steps:
1. Create `PROJECT_STRUCTURE.md`
2. Document module responsibilities
3. Document package structure
4. Add build instructions

#### Deliverables:
- `PROJECT_STRUCTURE.md`

---

## Wave 3 Tasks
_(INFRA-01 assists with integration testing)_

### W3-T9: Set Up Integration Test Framework
**Duration**: 1 day
**Dependencies**: W1-T4
**Blocking**: ✅ YES

#### Steps:
1. Create integration test base class
2. Configure Spring test context
3. Set up test data fixtures

#### Deliverables:
- Integration test framework

---

## Wave 4 Tasks

### W4-T1: Performance Profiling
**Duration**: 1 day
**Dependencies**: W3-T11
**Blocking**: ❌ NO

#### Steps:
1. Use JProfiler or VisualVM
2. Profile packet processing
3. Identify bottlenecks

#### Deliverables:
- Performance report

---

### W4-T9: Create Executable JAR with Dependencies
**Duration**: 1 day
**Dependencies**: W2-T4
**Blocking**: ✅ YES

#### Steps:
1. Build final JAR
2. Test with all dependencies
3. Verify all resources bundled

#### Deliverables:
- Production-ready JAR

---

# AGENT: DATA-01 (Database & Repository)

## Wave 1 Tasks

### W1-T5: Set Up SQLite Connection in Spring
**Duration**: 0.5 day
**Dependencies**: W1-T2
**Blocking**: ✅ YES

#### Steps:
1. Add SQLite dependency to `fm-core/pom.xml`:
```xml
<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.46.1.3</version>
</dependency>
```

2. Configure datasource in `application.yml`:
```yaml
spring:
  datasource:
    url: jdbc:sqlite:database.sqlite
    driver-class-name: org.sqlite.JDBC
```

3. Copy `database.sqlite` to project root

#### Validation:
- [ ] Database file accessible
- [ ] Connection established

#### Deliverables:
- SQLite configuration
- `database.sqlite` in project

---

### W1-T6: Test Database Connectivity
**Duration**: 0.5 day
**Dependencies**: W1-T5
**Blocking**: ✅ YES

#### Steps:
1. Create test query:
```java
@SpringBootTest
class DatabaseConnectivityTest {
    @Autowired
    private DataSource dataSource;

    @Test
    void testConnection() throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            assertNotNull(conn);
        }
    }
}
```

2. Run test

#### Validation:
- [ ] Connection test passes
- [ ] No errors in logs

#### Deliverables:
- Database connectivity test

---

## Wave 2 Tasks

### W2-T6: Create JPA Entities
**Duration**: 1.5 days (if using JPA)
**Dependencies**: W1-T5
**Blocking**: ✅ YES (for DOMAIN-01)

#### Steps:
1. Create `Description` entity:
```java
package com.dofus.fm.entity;

import jakarta.persistence.*;
import lombok.Data;

@Entity
@Table(name = "description")
@Data
public class Description {
    @Id
    @Column(name = "id")
    private Integer id;

    @Column(name = "description_text", columnDefinition = "TEXT")
    private String descriptionText;
}
```

2. Create remaining entities (see PRD Section 3.6):
   - `Effect`
   - `ItemEntity`
   - `EffectLine`
   - `ItemEffectLine`

3. Configure Hibernate dialect:
```yaml
spring:
  jpa:
    database-platform: org.hibernate.community.dialect.SQLiteDialect
    hibernate:
      ddl-auto: validate
```

#### Validation:
- [ ] All entities created
- [ ] Hibernate validates schema
- [ ] No mapping errors

#### Deliverables:
- 5 JPA entity classes

---

### W2-T7: OR: Implement JdbcTemplate Repositories
**Duration**: 2 days (alternative to W2-T6)
**Dependencies**: W1-T5
**Blocking**: ✅ YES (for DOMAIN-01)

#### Steps:
1. Create `ItemRepository`:
```java
package com.dofus.fm.repository;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

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

2. Create DTO classes for query results

#### Validation:
- [ ] Repositories created
- [ ] Queries execute successfully

#### Deliverables:
- Repository classes
- DTO classes

---

### W2-T8: Implement ItemRepository Queries
**Duration**: 1 day
**Dependencies**: W2-T6 or W2-T7
**Blocking**: ✅ YES

#### Steps:
1. Implement `findItemBasicInfo()`
2. Implement `findItemEffectLines()`
3. Test queries match Python output

#### Validation:
- [ ] Queries return correct data
- [ ] Results match Python version

#### Deliverables:
- Complete ItemRepository

---

### W2-T9: Implement EffectRepository Queries
**Duration**: 0.5 day
**Dependencies**: W2-T6 or W2-T7
**Blocking**: ✅ YES

#### Steps:
1. Implement `findById()`
2. Add caching for performance

#### Validation:
- [ ] Effect queries working
- [ ] Cache configured

#### Deliverables:
- Complete EffectRepository

---

### W2-T10: Write Repository Unit Tests
**Duration**: 1 day
**Dependencies**: W2-T8, W2-T9
**Blocking**: ❌ NO

#### Steps:
1. Test all repository methods
2. Use test data from database
3. Verify result formats

#### Validation:
- [ ] All repository tests passing
- [ ] Coverage >80%

#### Deliverables:
- Repository test suite

---

### W2-T11: Validate Queries Against Python Version
**Duration**: 1 day
**Dependencies**: W2-T10
**Blocking**: ❌ NO

#### Steps:
1. Run same queries in Python
2. Compare results field-by-field
3. Document any differences

#### Validation:
- [ ] 100% query parity
- [ ] No data discrepancies

#### Deliverables:
- Validation report

---

# AGENT: NETWORK-01 (Network & Protocol)

## Wave 1 Tasks

### W1-T7: Research Pcap4j Integration
**Duration**: 1 day
**Dependencies**: None
**Blocking**: ❌ NO

#### Steps:
1. Review Pcap4j documentation
2. Test basic packet capture
3. Verify permissions requirements

#### Validation:
- [ ] Pcap4j works on target platform
- [ ] Capture filter syntax understood

#### Deliverables:
- Pcap4j spike/proof-of-concept

---

### W1-T8: Capture Sample Dofus Packets for Testing
**Duration**: 1 day
**Dependencies**: None
**Blocking**: ⚠️ SEMI (needed for W2-T19)

#### Steps:
1. Run Python version with packet logging:
```python
import binascii
print(binascii.hexlify(pktdata))
```

2. Capture 50+ packets of each type:
   - 10+ ExchangeObjectMessage (5516/5519) for items
   - 10+ ExchangeObjectMessage for runes
   - 30+ CraftResultMessage (6188) with all result types

3. Save to test resources:
```
fm-network/src/test/resources/packets/
├── exchange_object_item_001.hex
├── exchange_object_rune_001.hex
├── craft_result_sc_001.hex
├── craft_result_sn_001.hex
├── craft_result_ec_001.hex
└── craft_result_en_001.hex
```

#### Validation:
- [ ] 50+ real packets captured
- [ ] All packet types represented
- [ ] Packets include known expected values

#### Deliverables:
- Packet capture files in test resources

---

## Wave 2 Tasks

### W2-T12: Implement DofusPacket Extraction Logic
**Duration**: 1 day
**Dependencies**: W1-T1
**Blocking**: ✅ YES

#### Steps:
1. Create `DofusPacket.java`:
```java
package com.dofus.fm.network.protocol;

import lombok.Data;

@Data
public class DofusPacket {
    private final int id;
    private final int length;
    private final byte[] data;

    public DofusPacket(int id, int length, byte[] data) {
        this.id = id;
        this.length = length;
        this.data = data;
    }
}
```

2. Implement packet extraction methods:
```java
public class DofusPacketExtractor {

    public static int getPacketId(byte[] packet) {
        int oct0 = packet[0] & 0xFF;
        int oct1 = (packet[1] & 0xFC) / 4;
        return oct1 + oct0 * 64;
    }

    public static int getDataLenLen(byte[] packet) {
        return packet[1] & 0x03;
    }

    public static int getDataLen(byte[] packet, int lenLen) {
        ByteBuffer buffer = ByteBuffer.wrap(packet, 2, lenLen);
        buffer.order(ByteOrder.BIG_ENDIAN);

        return switch (lenLen) {
            case 1 -> buffer.get() & 0xFF;
            case 2 -> buffer.getShort() & 0xFFFF;
            case 3 -> (buffer.get() & 0xFF) << 16 | (buffer.getShort() & 0xFFFF);
            default -> 0;
        };
    }

    public static byte[] getData(byte[] packet, int lenLen, int dataLen) {
        return Arrays.copyOfRange(packet, 2 + lenLen, 2 + lenLen + dataLen);
    }

    public static Result popPacket(byte[] stream) {
        int id = getPacketId(stream);
        int lenLen = getDataLenLen(stream);
        int dataLen = getDataLen(stream, lenLen);
        byte[] data = getData(stream, lenLen, dataLen);

        DofusPacket packet = new DofusPacket(id, dataLen, data);
        byte[] remaining = Arrays.copyOfRange(stream, 2 + lenLen + dataLen, stream.length);

        return new Result(remaining, packet);
    }

    public record Result(byte[] remaining, DofusPacket packet) {}
}
```

#### Validation:
- [ ] Packet ID extracted correctly
- [ ] Data length calculated correctly
- [ ] Payload extracted correctly
- [ ] Test with captured packets

#### Deliverables:
- `DofusPacket.java`
- `DofusPacketExtractor.java`

---

### W2-T13: Implement VarShort Reader
**Duration**: 1 day
**Dependencies**: W2-T12
**Blocking**: ✅ YES

#### Steps:
1. Create `VarIntReader.java`:
```java
package com.dofus.fm.network.protocol;

import java.nio.ByteBuffer;

public class VarIntReader {

    public static class ReadResult {
        public final byte[] remaining;
        public final int value;

        public ReadResult(byte[] remaining, int value) {
            this.remaining = remaining;
            this.value = value;
        }
    }

    public static ReadResult readVarShort(byte[] bytes) {
        int result = 0;
        int progress = 0;
        int offset = 0;

        while (progress < 16) {
            int currentByte = bytes[offset++] & 0xFF;
            boolean continuer = (currentByte & 0x80) == 0x80;

            if (progress > 0) {
                result += ((currentByte & 0x7F) << progress);
            } else {
                result += (currentByte & 0x7F);
            }

            progress += 7;

            if (!continuer) {
                // Convert to signed short
                if (result > 32767) {
                    result -= 65536;
                }

                byte[] remaining = Arrays.copyOfRange(bytes, offset, bytes.length);
                return new ReadResult(remaining, result);
            }
        }

        throw new IllegalArgumentException("VarShort too long");
    }
}
```

2. Test with known values:
   - Value 300 → bytes `[0xAC, 0x02]`
   - Value -100 → specific byte sequence

#### Validation:
- [ ] Reads positive values correctly
- [ ] Reads negative values correctly
- [ ] Handles continuation bit correctly
- [ ] Matches Python output exactly

#### Deliverables:
- `VarIntReader.readVarShort()`

---

### W2-T14: Implement VarInt Reader
**Duration**: 0.5 day
**Dependencies**: W2-T12
**Blocking**: ✅ YES

#### Steps:
1. Add `readVarInt()` to `VarIntReader`:
```java
public static ReadResult readVarInt(byte[] bytes) {
    long result = 0;
    int progress = 0;
    int offset = 0;

    while (progress < 32) {
        int currentByte = bytes[offset++] & 0xFF;
        boolean continuer = (currentByte & 0x80) == 0x80;

        if (progress > 0) {
            result += ((long)(currentByte & 0x7F) << progress);
        } else {
            result += (currentByte & 0x7F);
        }

        progress += 7;

        if (!continuer) {
            byte[] remaining = Arrays.copyOfRange(bytes, offset, bytes.length);
            return new ReadResult(remaining, (int)result);
        }
    }

    throw new IllegalArgumentException("VarInt too long");
}
```

#### Validation:
- [ ] Reads large integers correctly
- [ ] No overflow issues
- [ ] Matches Python output

#### Deliverables:
- `VarIntReader.readVarInt()`

---

### W2-T15: Implement readIntFromBytes
**Duration**: 0.5 day
**Dependencies**: W2-T12
**Blocking**: ✅ YES

#### Steps:
1. Add to `VarIntReader`:
```java
public static ReadResult readIntFromBytes(byte[] bytes, int size) {
    ByteBuffer buffer = ByteBuffer.wrap(bytes, 0, size);
    buffer.order(ByteOrder.BIG_ENDIAN);

    int value = switch (size) {
        case 1 -> buffer.get() & 0xFF;
        case 2 -> buffer.getShort() & 0xFFFF;
        case 4 -> buffer.getInt();
        default -> throw new IllegalArgumentException("Invalid size: " + size);
    };

    byte[] remaining = Arrays.copyOfRange(bytes, size, bytes.length);
    return new ReadResult(remaining, value);
}
```

#### Validation:
- [ ] Reads 1-byte correctly
- [ ] Reads 2-byte correctly
- [ ] Reads 4-byte correctly
- [ ] Big-endian byte order verified

#### Deliverables:
- `VarIntReader.readIntFromBytes()`

---

### W2-T16: Implement ExchangeObjectParser
**Duration**: 2 days
**Dependencies**: W2-T13, W2-T14, W2-T15
**Blocking**: ✅ YES

#### Steps:
1. Create DTOs:
```java
package com.dofus.fm.network.types;

import lombok.Data;
import java.util.List;

@Data
public class ExchangeObjectMessage {
    private int position;
    private int objectGID;
    private List<Effect> effects;
    private int objectUID;
    private int quantity;
}

@Data
public class Effect {
    private int actionId;
    private Integer value;      // For type 70
    private Integer min;         // For type 82
    private Integer max;         // For type 82
}
```

2. Implement parser (see PRD Section 2.4 for complete logic)

3. Test with captured packets

#### Validation:
- [ ] Parses item packets correctly
- [ ] Parses rune packets correctly
- [ ] Handles both effect types (70, 82)
- [ ] Matches Python output exactly

#### Deliverables:
- `ExchangeObjectMessage.java`
- `Effect.java`
- `ExchangeObjectParser.java`

---

### W2-T17: Implement CraftResultParser
**Duration**: 1.5 days
**Dependencies**: W2-T13, W2-T14, W2-T15
**Blocking**: ✅ YES

#### Steps:
1. Create DTO:
```java
@Data
public class CraftResultMessage {
    private int craftResult;      // 1=fail, 2=success
    private int objectGID;
    private List<Effect> effects;
    private int objectUID;
    private int quantity;
    private int magicPoolStatus;  // 3 = reliquat decreased
}
```

2. Implement parser (similar to ExchangeObjectParser)

#### Validation:
- [ ] Parses all result types
- [ ] magicPoolStatus extracted correctly
- [ ] Matches Python output

#### Deliverables:
- `CraftResultMessage.java`
- `CraftResultParser.java`

---

### W2-T18: Integrate Pcap4j Packet Capture
**Duration**: 2 days
**Dependencies**: W1-T7
**Blocking**: ✅ YES

#### Steps:
1. Add Pcap4j dependency
2. Create capture service:
```java
package com.dofus.fm.network.capture;

import org.pcap4j.core.*;
import org.springframework.stereotype.Service;

@Service
public class PacketCaptureService {

    public void startCapture() throws PcapNativeException {
        PcapNetworkInterface nif = getNetworkInterface();

        PcapHandle handle = nif.openLive(65536,
            PcapNetworkInterface.PromiscuousMode.PROMISCUOUS,
            10);

        handle.setFilter("host 213.248.126.61",
            BpfProgram.BpfCompileMode.OPTIMIZE);

        PacketListener listener = packet -> {
            // Process packet
        };

        handle.loop(-1, listener);
    }

    private PcapNetworkInterface getNetworkInterface() throws PcapNativeException {
        // Auto-detect or use configured interface
    }
}
```

#### Validation:
- [ ] Captures packets from Dofus server
- [ ] Filter works correctly
- [ ] Extracts TCP payload

#### Deliverables:
- `PacketCaptureService.java`

---

### W2-T19: Write Parser Unit Tests with Real Packets
**Duration**: 2 days
**Dependencies**: W2-T16, W2-T17, W1-T8
**Blocking**: ❌ NO

#### Steps:
1. Load captured packets from test resources
2. Test each parser:
```java
@Test
void testParseExchangeObjectMessage() {
    byte[] packet = loadTestPacket("exchange_object_item_001.hex");
    ExchangeObjectMessage msg = parser.parse(packet);

    assertEquals(2680, msg.getObjectGID());
    assertEquals(2, msg.getEffects().size());
    // etc.
}
```

3. Test all captured packets

#### Validation:
- [ ] All parsers tested
- [ ] 50+ real packets validated
- [ ] No parsing errors

#### Deliverables:
- Parser test suite

---

### W2-T20: Implement Async PacketCaptureService
**Duration**: 1 day
**Dependencies**: W2-T18
**Blocking**: ✅ YES

#### Steps:
1. Make capture async:
```java
@Service
public class PacketCaptureService {

    @Async("packetProcessorExecutor")
    public void startCaptureAsync() {
        // Packet capture loop
    }
}
```

2. Test async execution

#### Validation:
- [ ] Runs in separate thread
- [ ] Non-blocking

#### Deliverables:
- Async packet capture

---

# AGENT: DOMAIN-01 (Domain Models)

_(Complete task details for DOMAIN-01, FMCALC-01, and UI-01 continue in next section due to length...)_

---

## Coordination Checklist

Each agent should maintain a status board:

```
AGENT: [NAME]
Current Wave: [1/2/3/4]
Active Task: [Task ID]
Status: [In Progress / Blocked / Complete]
Blockers: [List dependencies or issues]
Next Task: [Task ID]
ETA: [Date]
```

Update daily in shared communication channel.

---

**Continue to Part 2 for remaining agents (DOMAIN-01, FMCALC-01, UI-01) and coordination details...**
