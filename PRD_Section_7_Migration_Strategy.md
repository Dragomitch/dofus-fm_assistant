# Section 7: Migration Strategy & Implementation Plan

## 7.1 Migration Approach

### Strategy: **Incremental Rewrite with Parallel Testing**

The migration will follow a phased approach, where each component is rewritten and tested independently before integration. The Python version will remain functional throughout the migration for comparison testing.

### Key Principles

1. **Exact Logic Preservation**: Replicate Python behavior exactly, even if bugs exist
2. **Test-Driven Migration**: Write tests first, then implement
3. **Modular Development**: Build each layer independently
4. **Continuous Validation**: Compare outputs between Python and Java versions
5. **Documentation**: Document every decision and deviation

---

## 7.2 Implementation Phases

### Phase 0: Project Setup (1-2 days)

#### Objectives
- Set up Java 26 development environment
- Create Maven multi-module project structure
- Configure Spring Boot 3.4.x
- Set up version control and build pipeline

#### Tasks
- [ ] Install Java 26 JDK
- [ ] Create Maven parent POM with module structure
- [ ] Add Spring Boot dependencies
- [ ] Configure IDE (IntelliJ IDEA recommended)
- [ ] Set up Git repository
- [ ] Create initial project structure:
  - `fm-core/`
  - `fm-network/`
  - `fm-ui/`
  - `fm-app/`
- [ ] Configure logging (Logback)
- [ ] Set up unit test infrastructure (JUnit 5)

#### Deliverables
- Compiling multi-module Maven project
- "Hello World" Spring Boot application
- CI/CD pipeline configured

---

### Phase 1: Database Layer (3-5 days)

#### Objectives
- Migrate SQLite database access
- Create JPA entities or use JdbcTemplate
- Implement repository layer
- Test database queries

#### Tasks

**1. Database Access Setup**
- [ ] Add SQLite JDBC driver dependency
- [ ] Configure Spring Data JPA (or JdbcTemplate)
- [ ] Copy `database.sqlite` to Java project
- [ ] Test database connection

**2. Create Entities (if using JPA)**
- [ ] `Description` entity
- [ ] `Effect` entity
- [ ] `ItemEntity` entity
- [ ] `EffectLine` entity
- [ ] `ItemEffectLine` entity

**Alternative: JdbcTemplate Repositories**
- [ ] `ItemRepository` with custom queries
- [ ] `EffectRepository` with custom queries

**3. Test Queries**
- [ ] Test item basic info query
- [ ] Test item effect lines query
- [ ] Test rune effect data query
- [ ] Compare results with Python queries

#### Validation
```bash
# Run Python version
python main.py
# Observe item ID 12345 is loaded with effects

# Run Java test
mvn test -Dtest=ItemRepositoryTest#testFindItemBasicInfo
# Verify same data is returned
```

#### Deliverables
- Functional repository layer
- 100% query parity with Python
- Unit tests for all repositories

---

### Phase 2: Domain Models (2-3 days)

#### Objectives
- Implement `Line`, `Rune`, and `Item` domain classes
- Create factory services
- Test weight calculations

#### Tasks

**1. Implement Line Class**
- [ ] Create `Line.java` with all getters
- [ ] Implement weight calculation: `getWeight()`
- [ ] Implement max weight calculation: `getMaxWeight()`
- [ ] Implement value tracking: `setValue()`, `initValue()`
- [ ] Test modification tracking

**2. Implement Rune Class**
- [ ] Create `Rune.java`
- [ ] Implement weight calculation
- [ ] Create `RuneFactory` service

**3. Implement Item Class**
- [ ] Create `Item.java`
- [ ] Implement original lines vs exotic lines
- [ ] Implement `getLineByEffectId()`
- [ ] Implement `getWeight()`
- [ ] Create `ItemFactory` service

**4. Test Domain Logic**
- [ ] Test Line weight calculations
- [ ] Test Rune creation from database
- [ ] Test Item creation with original lines
- [ ] Test exotic line handling

#### Validation
```java
@Test
void testLineWeightCalculation() {
    Line vitLine = new Line(125, 20, 40, 30, effectRepository);
    // Vitality weight = 0.25
    assertEquals(7.5, vitLine.getWeight(), 0.01);
    assertEquals(10.0, vitLine.getMaxWeight(), 0.01);
}
```

#### Deliverables
- `Line`, `Rune`, `Item` domain models
- Factory services
- Comprehensive unit tests
- Weight calculation validation against Python

---

### Phase 3: Network Packet Parsing (5-7 days)

#### Objectives
- Implement Dofus packet protocol parsing
- Create packet capture service
- Parse ExchangeObjectMessage and CraftResultMessage
- Test with real packet captures

#### Tasks

**1. Packet Structure Parsing**
- [ ] Implement `DofusPacket` class
- [ ] Implement `get_pkt_id()` logic
- [ ] Implement `get_data_len_len()` logic
- [ ] Implement `get_data_len()` logic
- [ ] Implement `pop_pkt()` logic
- [ ] Test with sample packets

**2. Variable-Length Integer Readers**
- [ ] Implement `readVarShort()` with exact Python logic
- [ ] Implement `readVarInt()` with exact Python logic
- [ ] Implement `readIntFromBytes()`
- [ ] Test with known values

**3. Message Parsers**
- [ ] Implement `ExchangeObjectParser` (packets 5516/5519)
- [ ] Implement `CraftResultParser` (packet 6188)
- [ ] Handle effect types (70 = Integer, 82 = MinMax)
- [ ] Test parsing with captured packets

**4. Packet Capture Service**
- [ ] Integrate Pcap4j
- [ ] Implement packet filtering: `host 213.248.126.61`
- [ ] Extract TCP payload
- [ ] Feed to packet parser
- [ ] Test in async thread

#### Validation

**Create Test Packet Captures**:
```bash
# In Python version, add packet logging
print(binascii.hexlify(pktdata))
```

**Use in Java Tests**:
```java
@Test
void testParseExchangeObjectMessage() {
    byte[] packet = hexToBytes("15 8C 01 A8 12 00 02 ..."); // Real capture
    ExchangeObjectMessage msg = parser.parse(packet);
    assertEquals(2680, msg.getObjectGID()); // Expected item ID
}
```

#### Deliverables
- Complete packet parsing implementation
- Parsers for all 3 packet types
- Unit tests with real packet captures
- Packet capture service running in separate thread

---

### Phase 4: FM Calculation Logic (4-6 days)

#### Objectives
- Implement FM weight calculations
- Implement reliquat tracking
- Implement result classification (SC/SN/EC/EN)
- Validate against Python version

#### Tasks

**1. Implement FMCalculationService**
- [ ] `executeFM()` method with all 6 steps
- [ ] Update stat values from packet
- [ ] Remove missing lines
- [ ] Classify result type
- [ ] Calculate theoretical weight
- [ ] Calculate real weight
- [ ] Calculate reliquat modification

**2. Implement Item Initialization**
- [ ] `initializeItemStats()` method
- [ ] Handle exotic lines
- [ ] Test with initial item packet

**3. Implement Result Classification**
- [ ] Create `FMResultType` enum
- [ ] Implement classification logic
- [ ] Test all 4 result types

**4. Test FM Scenarios**
- [ ] Test Clean Success (SC)
- [ ] Test Success with Sink (SN)
- [ ] Test Clean Failure (EC)
- [ ] Test Neutral Failure (EN)
- [ ] Compare reliquat values with Python

#### Validation

**Parallel Testing**:
1. Run Python version and log:
   - Initial item state
   - Rune used
   - Result packet
   - Calculated reliquat

2. Feed same packets to Java version
3. Compare outputs line by line

```java
@Test
void testFMCalculation_MatchesPython() {
    // Use exact packet from Python log
    Item item = createItemFromPythonLog();
    Rune rune = createRuneFromPythonLog();
    CraftResultMessage result = parsePythonPacketLog();

    FMResultType resultType = fmService.executeFM(item, rune, result);

    // Compare with Python output
    assertEquals(FMResultType.SC, resultType);
    assertEquals(12.5, item.getReliquat(), 0.01);
}
```

#### Deliverables
- Complete FM calculation service
- All 4 result types tested
- Reliquat calculation validated
- Weight calculation verified

---

### Phase 5: Session Management (2-3 days)

#### Objectives
- Implement FM session service
- Handle item/rune selection
- Coordinate packet processing with business logic

#### Tasks

**1. Create FMSessionService**
- [ ] Track current item and rune
- [ ] Handle ExchangeObjectMessage (detect item vs rune)
- [ ] Handle CraftResultMessage
- [ ] Integrate with FMCalculationService

**2. Implement Rune Detection**
- [ ] Create `RuneIdentifier` with hardcoded rune IDs
- [ ] Alternatively: Load from database

**3. Test Session Flow**
- [ ] Test item selection
- [ ] Test rune selection
- [ ] Test FM execution
- [ ] Test multiple FM attempts in sequence

#### Deliverables
- FMSessionService coordinating all components
- Rune vs item detection working
- End-to-end session tests

---

### Phase 6: JavaFX UI (5-7 days)

#### Objectives
- Create JavaFX GUI matching Tkinter layout
- Implement display updates
- Handle threading correctly

#### Tasks

**1. JavaFX Setup**
- [ ] Add JavaFX dependencies
- [ ] Create `FMApplication` entry point
- [ ] Integrate Spring Boot with JavaFX

**2. Create UI Layout**
- [ ] Design FXML layout file
- [ ] Create `MainController`
- [ ] Implement table for stat lines
- [ ] Add item/rune labels
- [ ] Add reliquat label

**3. Implement Display Updates**
- [ ] `updateItem()` method
- [ ] `updateRune()` method
- [ ] Color coding (green/red for changes)
- [ ] Number formatting

**4. Threading**
- [ ] Ensure `Platform.runLater()` is used
- [ ] Test UI updates from packet handler thread

**5. Create DisplayService**
- [ ] Bridge between business logic and UI
- [ ] Replace Python's listener pattern

#### Validation

**Manual Testing Checklist**:
- [ ] Run application
- [ ] Place item in FM table (in Dofus)
- [ ] Verify item appears in Java UI
- [ ] Place rune in FM table
- [ ] Verify rune appears in Java UI
- [ ] Apply rune in Dofus
- [ ] Verify stat changes display correctly
- [ ] Verify color coding (green = increase, red = decrease)
- [ ] Verify reliquat updates

#### Deliverables
- Functional JavaFX UI
- Real-time updates working
- UI matches Python version

---

### Phase 7: Integration & Testing (3-5 days)

#### Objectives
- Integrate all components
- End-to-end testing
- Performance testing
- Bug fixes

#### Tasks

**1. Full Integration**
- [ ] Wire all services together
- [ ] Test packet capture → parsing → calculation → UI flow
- [ ] Fix any integration issues

**2. End-to-End Testing**
- [ ] Run alongside Python version
- [ ] Compare outputs for 50+ FM attempts
- [ ] Verify reliquat tracking over long session
- [ ] Test exotic lines

**3. Performance Testing**
- [ ] Measure packet processing latency
- [ ] Ensure < 50ms per packet
- [ ] Profile memory usage
- [ ] Optimize if needed

**4. Error Handling**
- [ ] Test invalid packets
- [ ] Test unknown item IDs
- [ ] Test malformed data
- [ ] Add user-friendly error messages

**5. Bug Fixes**
- [ ] Address all failing tests
- [ ] Fix any calculation discrepancies
- [ ] Resolve UI issues

#### Validation

**Comprehensive Test Suite**:
```bash
# Run all tests
mvn clean test

# Expected: 100% pass rate
# Coverage: >80% for business logic
```

**Live Testing**:
- Run both Python and Java versions simultaneously
- Perform 100 FM attempts
- Compare:
  - Reliquat values (should be identical)
  - Result classifications (SC/SN/EC/EN)
  - Item weights
  - Stat tracking

#### Deliverables
- Fully integrated application
- All tests passing
- Performance validated
- Bug-free release candidate

---

### Phase 8: Documentation & Deployment (2-3 days)

#### Objectives
- Create user documentation
- Package application
- Create deployment instructions

#### Tasks

**1. User Documentation**
- [ ] Installation guide
- [ ] System requirements (Java 26, admin privileges)
- [ ] Usage instructions
- [ ] Troubleshooting guide

**2. Developer Documentation**
- [ ] Architecture overview
- [ ] Code structure guide
- [ ] Contributing guidelines
- [ ] API documentation (JavaDoc)

**3. Packaging**
- [ ] Create executable JAR with dependencies
- [ ] Bundle database.sqlite
- [ ] Create installer (optional: jpackage)
- [ ] Test on clean machine

**4. Deployment**
- [ ] Create GitHub release
- [ ] Publish binaries
- [ ] Update README with installation instructions

#### Deliverables
- User manual
- Developer guide
- Packaged application
- Release notes

---

## 7.3 Testing Strategy

### Unit Testing

**Coverage Goals**:
- Business logic: **90%+**
- Repositories: **80%+**
- Parsers: **85%+**
- UI: **50%+** (manual testing primary)

**Key Test Classes**:
```
src/test/java/com/dofus/fm/
├── domain/
│   ├── LineTest.java
│   ├── RuneTest.java
│   └── ItemTest.java
├── service/
│   ├── FMCalculationServiceTest.java
│   ├── ItemFactoryTest.java
│   └── RuneFactoryTest.java
├── network/
│   ├── VarIntReaderTest.java
│   ├── ExchangeObjectParserTest.java
│   └── CraftResultParserTest.java
└── repository/
    └── ItemRepositoryTest.java
```

---

### Integration Testing

**Test Scenarios**:
1. **Full Packet Flow**: Capture → Parse → Calculate → Display
2. **Multiple FM Attempts**: Sequence of 10+ attempts
3. **Exotic Line Handling**: Add stats not on base item
4. **Reliquat Accumulation**: Track across 50+ attempts

**Example Integration Test**:
```java
@SpringBootTest
class FMIntegrationTest {

    @Test
    void testFullFMFlow() {
        // 1. Parse item packet
        ExchangeObjectMessage itemMsg = parseItemPacket();
        sessionService.handleExchangeObject(itemMsg);

        // 2. Parse rune packet
        ExchangeObjectMessage runeMsg = parseRunePacket();
        sessionService.handleExchangeObject(runeMsg);

        // 3. Parse craft result packet
        CraftResultMessage craftMsg = parseCraftResultPacket();
        sessionService.handleCraftResult(craftMsg);

        // 4. Verify item updated correctly
        Item item = sessionService.getCurrentItem();
        assertEquals(expectedValue, item.getReliquat(), 0.01);
    }
}
```

---

### Validation Against Python

**Comparison Test Framework**:

1. **Log Python Outputs**:
```python
# Add to Python version
import json
with open('fm_log.json', 'a') as f:
    json.dump({
        'timestamp': time.time(),
        'item_id': item.id,
        'rune_id': rune.id,
        'result_type': result_type,
        'reliquat': item.reliquat,
        'lines': [{'id': line.getEffectId(), 'value': line.getValue()} for line in item.getLines()]
    }, f)
    f.write('\n')
```

2. **Replay in Java**:
```java
@Test
void testAgainstPythonLog() {
    List<FMEvent> pythonEvents = loadPythonLog("fm_log.json");

    for (FMEvent event : pythonEvents) {
        // Replay event in Java
        FMResultType result = replayEvent(event);

        // Compare outputs
        assertEquals(event.getResultType(), result);
        assertEquals(event.getReliquat(), getCurrentReliquat(), 0.01);
    }
}
```

---

## 7.4 Migration Risks & Mitigation

### Risk 1: Packet Parsing Errors

**Risk**: Incorrect parsing leads to wrong calculations

**Mitigation**:
- Capture real packets from live game sessions
- Create test suite with 100+ real packets
- Validate byte-by-byte against Python

**Contingency**:
- Log unparsable packets for investigation
- Fall back to "skip" for unknown packet structures

---

### Risk 2: Floating-Point Precision Differences

**Risk**: Java `double` vs Python arbitrary precision

**Mitigation**:
- Use `BigDecimal` for critical calculations
- Compare outputs with epsilon tolerance (±0.01)
- Test edge cases (very large reliquat values)

**Contingency**:
- Document any precision differences
- Consider switching to `BigDecimal` if issues arise

---

### Risk 3: Threading Issues

**Risk**: Race conditions between packet handler and UI updates

**Mitigation**:
- Use `Platform.runLater()` for all UI updates
- Synchronize session state access
- Test under high packet volume

**Contingency**:
- Add explicit locking (`synchronized`)
- Use immutable domain objects

---

### Risk 4: Pcap4j Permissions

**Risk**: Packet capture requires root/admin privileges

**Mitigation**:
- Document requirement clearly
- Provide platform-specific setup guides
- Consider using `setcap` on Linux

**Contingency**:
- Offer "replay mode" using packet logs
- Consider alternative capture methods

---

### Risk 5: Unknown Python Behavior

**Risk**: Undocumented logic or bugs in Python version

**Mitigation**:
- Test extensively with Python version running in parallel
- Document any discovered quirks
- Ask original developer if available

**Contingency**:
- Match Python behavior even if it seems wrong
- Add TODO comments for future investigation

---

## 7.5 Success Criteria

### Functional Requirements

- [ ] Application captures and parses Dofus packets correctly
- [ ] Item and rune detection works 100% of the time
- [ ] FM calculations match Python version exactly
- [ ] Reliquat tracking accurate over 500+ FM attempts
- [ ] UI displays updates in real-time
- [ ] All 4 result types (SC/SN/EC/EN) classified correctly

### Non-Functional Requirements

- [ ] Packet processing latency < 50ms
- [ ] Application starts in < 5 seconds
- [ ] Memory usage < 512 MB
- [ ] No memory leaks after 8-hour session
- [ ] Unit test coverage > 80%
- [ ] Zero crash rate during 100+ FM attempts

### User Acceptance

- [ ] User can perform FM session without referring to Python version
- [ ] UI is intuitive and matches expected layout
- [ ] Reliquat calculations feel accurate (user feedback)
- [ ] Application is stable across different network conditions

---

## 7.6 Deployment Plan

### Pre-Deployment Checklist

- [ ] All tests passing
- [ ] Code reviewed
- [ ] Documentation complete
- [ ] Packaging tested on all platforms (Windows, Linux, macOS)
- [ ] Performance validated
- [ ] Security review (admin privileges documented)

### Release Process

**Version 1.0.0**:
1. Tag release in Git: `git tag v1.0.0`
2. Build release JAR: `mvn clean package -Prelease`
3. Create GitHub release with binary
4. Update README with download link
5. Announce release (Discord, forums, etc.)

### Platform-Specific Notes

**Windows**:
- Requires "Run as Administrator" for packet capture
- May need WinPcap or Npcap installed
- Consider using jpackage to create `.exe`

**Linux**:
- Requires `sudo` or `setcap` for packet capture
- Package as `.deb` or `.rpm` (optional)
- Test on Ubuntu, Fedora

**macOS**:
- Requires `sudo` for packet capture
- Package as `.dmg` (optional)
- May require security settings adjustment

---

## 7.7 Timeline Summary

| Phase | Duration | Dependencies | Deliverables |
|-------|----------|--------------|--------------|
| 0. Project Setup | 1-2 days | None | Maven project structure |
| 1. Database Layer | 3-5 days | Phase 0 | Repository layer + tests |
| 2. Domain Models | 2-3 days | Phase 1 | Item, Rune, Line classes |
| 3. Network Parsing | 5-7 days | Phase 0 | Packet parsers + capture |
| 4. FM Calculation | 4-6 days | Phase 2, 3 | Calculation service |
| 5. Session Management | 2-3 days | Phase 4 | Session service |
| 6. JavaFX UI | 5-7 days | Phase 5 | UI + display service |
| 7. Integration & Testing | 3-5 days | All phases | Full integration |
| 8. Documentation & Deployment | 2-3 days | Phase 7 | Release v1.0.0 |

**Total Estimated Time**: **27-41 days** (5-8 weeks)

**Recommended Schedule**: **6-7 weeks** with buffer for unexpected issues

---

## 7.8 Post-Migration Tasks

### Maintenance Plan

**Regular Tasks**:
- Monitor for Dofus protocol changes
- Update rune ID list when new runes added
- Performance monitoring
- Bug fixes based on user feedback

**Potential Enhancements** (Post v1.0):
- [ ] Probability calculator for FM outcomes
- [ ] Historical FM statistics tracking
- [ ] Export FM session logs
- [ ] Multiple item tracking simultaneously
- [ ] Web UI version for remote access
- [ ] Mobile companion app

---

## 7.9 Rollback Plan

If critical issues arise post-deployment:

1. **Immediate**: Revert to Python version
2. **Communicate**: Notify users of issue
3. **Investigate**: Root cause analysis
4. **Fix**: Hotfix release or delay next version
5. **Re-test**: Validate fix before re-deployment

**Backup Strategy**:
- Keep Python version available as fallback
- Maintain Git history for easy rollback
- Provide both versions during transition period

---

## 7.10 Final Migration Checklist

### Code Quality
- [ ] All TODOs resolved or documented
- [ ] Code follows Java conventions
- [ ] No compiler warnings
- [ ] SonarQube analysis passing (optional)

### Testing
- [ ] 100% unit tests passing
- [ ] Integration tests passing
- [ ] Manual testing completed
- [ ] Comparison testing vs Python successful

### Documentation
- [ ] README updated
- [ ] JavaDoc complete for public APIs
- [ ] User guide written
- [ ] Troubleshooting guide created

### Deployment
- [ ] JAR builds successfully
- [ ] Database bundled correctly
- [ ] Tested on clean machine
- [ ] Installation instructions verified

### User Communication
- [ ] Migration announcement prepared
- [ ] Tutorial video/screenshots ready (optional)
- [ ] Support channel established (GitHub Issues)

---

## 7.11 Conclusion

This migration plan provides a comprehensive, phased approach to converting the Dofus FM Assistant from Python to Java 26 with Spring Boot 3.x. By following these phases sequentially and validating each component against the original Python version, we ensure:

1. **Exact functional parity** with the existing system
2. **Improved performance** through Java's optimization
3. **Better maintainability** with Spring Boot architecture
4. **Modern UI** with JavaFX
5. **Robust testing** ensuring reliability

The estimated timeline of **6-7 weeks** provides adequate time for thorough development, testing, and validation. The modular approach allows for independent testing of each component before integration, reducing risk and ensuring quality.

**Next Steps**:
1. Review and approve this PRD
2. Set up development environment (Phase 0)
3. Begin Phase 1: Database Layer
4. Follow the plan, validating each phase before proceeding

**Success depends on**:
- Meticulous attention to detail in replicating Python logic
- Comprehensive testing at each phase
- Continuous validation against the original implementation
- Patience with packet parsing complexities

With this detailed roadmap, the migration should be achievable while maintaining the exact functionality users depend on.
