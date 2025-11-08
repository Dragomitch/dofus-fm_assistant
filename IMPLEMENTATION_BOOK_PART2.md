# Implementation Book - Part 2
## Domain, FM Calculation, and UI Agents

---

# AGENT: DOMAIN-01 (Domain Models)

## Wave 2 Tasks

### W2-T21: Implement Line Class
**Duration**: 1 day
**Dependencies**: W2-T8, W2-T9
**Blocking**: ✅ YES

#### Steps:
1. Create `fm-core/src/main/java/com/dofus/fm/domain/Line.java`:
```java
package com.dofus.fm.domain;

import lombok.Data;

@Data
public class Line {
    private Integer effectId;
    private Double effectWeight;
    private Integer min;
    private Integer max;
    private String descriptionTemplate;
    private Integer value;
    private Integer lastModification;

    public Line(Integer effectId, Integer min, Integer max, Integer value,
                EffectRepository effectRepository) {
        this.effectId = effectId;
        this.min = min;
        this.max = max;
        this.value = value;
        this.lastModification = 0;

        // Fetch effect metadata from database
        Effect effect = effectRepository.findById(effectId)
            .orElseThrow(() -> new IllegalArgumentException("Unknown effect: " + effectId));

        this.effectWeight = effect.getWeight();
        this.descriptionTemplate = effect.getDescription().getDescriptionText();
    }

    public Double getWeight() {
        return effectWeight * value;
    }

    public Double getMaxWeight() {
        return effectWeight * max;
    }

    public String getDescription() {
        return descriptionTemplate.replace("#1{~1~2 à }#2", String.valueOf(value));
    }

    public void setValue(Integer newValue) {
        this.lastModification = newValue - this.value;
        this.value = newValue;
    }

    public void initValue(Integer newValue) {
        this.value = newValue;
    }

    public boolean isOvermax() {
        return value > max;
    }
}
```

2. Add negative effect mapping:
```java
package com.dofus.fm.domain;

import java.util.Map;

public class NegativeEffectMapping {
    private static final Map<Integer, Integer> NEGATIVE_TO_POSITIVE = Map.ofEntries(
        Map.entry(116, 117),   // - Range
        Map.entry(145, 112),   // - Damage
        // ... (all 48 mappings from PRD Section 3.3)
    );

    public static boolean isNegative(Integer effectId) {
        return NEGATIVE_TO_POSITIVE.containsKey(effectId);
    }

    public static Integer getPositiveCounterpart(Integer negativeEffectId) {
        return NEGATIVE_TO_POSITIVE.get(negativeEffectId);
    }
}
```

#### Validation:
- [ ] Weight calculation correct: `value * effectWeight`
- [ ] Max weight calculation correct: `max * effectWeight`
- [ ] `setValue()` tracks modification correctly
- [ ] `initValue()` doesn't change `lastModification`
- [ ] Description template replacement works

#### Test Cases:
```java
@Test
void testLineWeightCalculation() {
    // Vitality: weight 0.25, value 30
    Line line = new Line(125, 20, 40, 30, effectRepository);
    assertEquals(7.5, line.getWeight(), 0.01);
    assertEquals(10.0, line.getMaxWeight(), 0.01);
}

@Test
void testSetValueTracksModification() {
    Line line = new Line(125, 20, 40, 30, effectRepository);
    line.setValue(35);
    assertEquals(35, line.getValue());
    assertEquals(5, line.getLastModification());
}
```

#### Deliverables:
- `Line.java`
- `NegativeEffectMapping.java`
- Line unit tests

---

### W2-T22: Implement Rune Class
**Duration**: 0.5 day
**Dependencies**: W2-T8
**Blocking**: ✅ YES

#### Steps:
1. Create `fm-core/src/main/java/com/dofus/fm/domain/Rune.java`:
```java
package com.dofus.fm.domain;

import lombok.Data;

@Data
public class Rune {
    private Integer id;
    private String name;
    private Integer effectId;
    private Integer effectValue;
    private Double effectWeight;
    private String description;

    public Integer getWeight() {
        return (int) (effectValue * effectWeight);
    }
}
```

#### Validation:
- [ ] Weight calculation: `value * weight`
- [ ] All getters work

#### Deliverables:
- `Rune.java`

---

### W2-T23: Implement Item Class (Structure Only)
**Duration**: 1 day
**Dependencies**: W2-T21
**Blocking**: ✅ YES

#### Steps:
1. Create `fm-core/src/main/java/com/dofus/fm/domain/Item.java`:
```java
package com.dofus.fm.domain;

import lombok.Data;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

@Data
public class Item {
    private Integer id;
    private Integer level;
    private String name;

    private List<Line> originalLines = new ArrayList<>();
    private List<Line> exoticLines = new ArrayList<>();

    private Double reliquat = 0.0;
    private Double lastReliquatModification = 0.0;

    public List<Line> getLines() {
        return Stream.concat(originalLines.stream(), exoticLines.stream())
            .collect(Collectors.toList());
    }

    public Line getLineByEffectId(Integer effectId) {
        return getLines().stream()
            .filter(line -> line.getEffectId().equals(effectId))
            .findFirst()
            .orElse(null);
    }

    public Double getWeight() {
        return getLines().stream()
            .mapToDouble(Line::getWeight)
            .sum();
    }

    public void cleanLines() {
        exoticLines.removeIf(line ->
            line.getValue() == 0 && line.getLastModification() == 0
        );
    }
}
```

2. Note: FM calculation methods will be added by FMCALC-01 in W2-T36

#### Validation:
- [ ] `getLines()` combines original and exotic
- [ ] `getLineByEffectId()` finds lines correctly
- [ ] `getWeight()` sums all line weights

#### Deliverables:
- `Item.java` (structure)

---

### W2-T24: Implement RuneFactory
**Duration**: 0.5 day
**Dependencies**: W2-T22, W2-T8
**Blocking**: ✅ YES

#### Steps:
1. Create `fm-core/src/main/java/com/dofus/fm/service/RuneFactory.java`:
```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Rune;
import com.dofus.fm.repository.ItemRepository;
import org.springframework.stereotype.Service;

@Service
public class RuneFactory {

    private final ItemRepository itemRepository;

    public RuneFactory(ItemRepository itemRepository) {
        this.itemRepository = itemRepository;
    }

    public Rune createRune(Integer runeId) {
        // Fetch item data
        Object[] itemData = itemRepository.findItemBasicInfo(runeId)
            .orElseThrow(() -> new IllegalArgumentException("Rune not found: " + runeId));

        String name = (String) itemData[2];

        // Fetch rune effect data
        Object[] effectData = itemRepository.findItemEffectData(runeId)
            .orElseThrow(() -> new IllegalArgumentException("Rune effect not found: " + runeId));

        Integer effectId = (Integer) effectData[0];
        Integer effectValue = (Integer) effectData[1];
        Double effectWeight = (Double) effectData[2];
        String descriptionTemplate = (String) effectData[3];

        Rune rune = new Rune();
        rune.setId(runeId);
        rune.setName(name);
        rune.setEffectId(effectId);
        rune.setEffectValue(effectValue);
        rune.setEffectWeight(effectWeight);
        rune.setDescription(descriptionTemplate.replace("#1{~1~2 à }#2", String.valueOf(effectValue)));

        return rune;
    }
}
```

#### Validation:
- [ ] Creates rune from database
- [ ] Weight calculated correctly
- [ ] Description formatted

#### Deliverables:
- `RuneFactory.java`

---

### W2-T25: Implement ItemFactory
**Duration**: 1 day
**Dependencies**: W2-T23, W2-T8
**Blocking**: ✅ YES

#### Steps:
1. Create `fm-core/src/main/java/com/dofus/fm/service/ItemFactory.java`:
```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Line;
import com.dofus.fm.repository.ItemRepository;
import com.dofus.fm.repository.EffectRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class ItemFactory {

    private final ItemRepository itemRepository;
    private final EffectRepository effectRepository;

    public ItemFactory(ItemRepository itemRepository,
                       EffectRepository effectRepository) {
        this.itemRepository = itemRepository;
        this.effectRepository = effectRepository;
    }

    @Transactional(readOnly = true)
    public Item createItem(Integer itemId) {
        // Fetch item basic info
        Object[] basicInfo = itemRepository.findItemBasicInfo(itemId)
            .orElseThrow(() -> new IllegalArgumentException("Item not found: " + itemId));

        Item item = new Item();
        item.setId(itemId);
        item.setLevel((Integer) basicInfo[1]);
        item.setName((String) basicInfo[2]);

        // Fetch item effect lines
        List<Object[]> effectLines = itemRepository.findItemEffectLines(itemId);

        for (Object[] effectLine : effectLines) {
            Integer effectId = (Integer) effectLine[0];
            Integer min = (Integer) effectLine[1];
            Integer max = (Integer) effectLine[2];

            // Skip "exchangeable" flags (983, 984)
            if (effectId == 983 || effectId == 984) {
                continue;
            }

            Line line = new Line(effectId, min, max, 0, effectRepository);
            item.getOriginalLines().add(line);
        }

        return item;
    }
}
```

#### Validation:
- [ ] Creates item from database
- [ ] Original lines populated
- [ ] Skips exchangeable flags (983, 984)

#### Deliverables:
- `ItemFactory.java`

---

### W2-T26: Implement Weight Calculations
**Duration**: 0.5 day
**Dependencies**: W2-T21
**Blocking**: ✅ YES

_(Already implemented in Line class W2-T21)_

#### Validation:
- [ ] Line weight: `value * weight`
- [ ] Item weight: sum of all lines

#### Deliverables:
- Weight calculation methods

---

### W2-T27: Implement NegativeEffectMapping
**Duration**: 0.5 day
**Dependencies**: None
**Blocking**: ❌ NO

_(Already implemented in W2-T21)_

#### Deliverables:
- `NegativeEffectMapping.java`

---

### W2-T28: Write Domain Model Unit Tests
**Duration**: 1.5 days
**Dependencies**: W2-T21-26
**Blocking**: ❌ NO

#### Steps:
1. Test Line class:
   - Weight calculations
   - setValue() modification tracking
   - initValue() doesn't track
   - Description formatting

2. Test Item class:
   - getLines() combination
   - getLineByEffectId()
   - getWeight() summation
   - cleanLines() removes zeros

3. Test factories:
   - ItemFactory creates correct item
   - RuneFactory creates correct rune

#### Validation:
- [ ] All domain tests passing
- [ ] Coverage >80%

#### Deliverables:
- Domain model test suite

---

### W2-T29: Validate Weight Calculations vs Python
**Duration**: 1 day
**Dependencies**: W2-T28
**Blocking**: ❌ NO

#### Steps:
1. Create test with known Python values
2. Compare:
   - Line weights
   - Item total weight
   - Rune weights

#### Validation:
- [ ] 100% match with Python
- [ ] No calculation discrepancies

#### Deliverables:
- Validation test suite

---

# AGENT: FMCALC-01 (FM Calculation)

## Wave 2 Tasks

### W2-T30: Implement FMResultType Enum
**Duration**: 0.25 day
**Dependencies**: None
**Blocking**: ✅ YES

#### Steps:
1. Create `fm-core/src/main/java/com/dofus/fm/service/FMResultType.java`:
```java
package com.dofus.fm.service;

public enum FMResultType {
    SC("Success Clean"),
    SN("Success Neutral"),
    EC("Échec Clean"),
    EN("Échec Neutral");

    private final String displayName;

    FMResultType(String displayName) {
        this.displayName = displayName;
    }

    public String getDisplayName() {
        return displayName;
    }
}
```

#### Deliverables:
- `FMResultType.java`

---

### W2-T31: Implement Result Classification Logic
**Duration**: 1 day
**Dependencies**: W2-T30, W2-T23
**Blocking**: ✅ YES

#### Steps:
1. Add to FMCalculationService:
```java
private FMResultType classifyResult(CraftResultMessage craftResult, Item item) {
    // Check if any stat decreased
    boolean malus = item.getLines().stream()
        .anyMatch(line -> line.getLastModification() < 0);

    // Check if reliquat decreased or stats lowered
    boolean somethingLowered = malus || (craftResult.getMagicPoolStatus() == 3);

    if (craftResult.getCraftResult() == 2) { // Success
        return somethingLowered ? FMResultType.SN : FMResultType.SC;
    } else if (craftResult.getCraftResult() == 1) { // Failure
        return somethingLowered ? FMResultType.EC : FMResultType.EN;
    }

    throw new IllegalStateException("Unknown craftResult: " + craftResult.getCraftResult());
}
```

#### Validation:
- [ ] SC: Success, no malus, magicPoolStatus != 3
- [ ] SN: Success, malus OR magicPoolStatus == 3
- [ ] EC: Failure, malus OR magicPoolStatus == 3
- [ ] EN: Failure, no malus, magicPoolStatus != 3

#### Deliverables:
- Result classification method

---

### W2-T32: Implement Theoretical Weight Calculation
**Duration**: 0.5 day
**Dependencies**: W2-T30, W2-T22 (Rune)
**Blocking**: ✅ YES

#### Steps:
1. Add to FMCalculationService:
```java
private double calculateTheoreticalWeight(FMResultType resultType, Rune rune) {
    return switch (resultType) {
        case SC -> rune.getWeight();  // Clean success: +weight
        case SN, EN -> 0.0;           // Neutral: no weight
        case EC -> -rune.getWeight(); // Clean failure: -weight
    };
}
```

#### Validation:
- [ ] SC returns rune weight
- [ ] SN/EN return 0
- [ ] EC returns negative rune weight

#### Deliverables:
- Theoretical weight method

---

### W2-T33: Implement Real Weight Calculation
**Duration**: 0.5 day
**Dependencies**: W2-T21, W2-T23
**Blocking**: ✅ YES

#### Steps:
1. Add to FMCalculationService:
```java
private double calculateRealWeight(Item item) {
    return item.getLines().stream()
        .mapToDouble(line -> line.getLastModification() * line.getEffectWeight())
        .sum();
}
```

#### Validation:
- [ ] Sums all (modification × weight)
- [ ] Handles positive and negative modifications

#### Deliverables:
- Real weight method

---

### W2-T34: Implement Reliquat Modification Formula
**Duration**: 1 day
**Dependencies**: W2-T32, W2-T33
**Blocking**: ✅ YES

#### Steps:
1. Add to FMCalculationService:
```java
private void calculateReliquatModification(Item item, double theoreticalWeight, double realWeight) {
    double reliquatModification = -(realWeight - theoreticalWeight);
    item.setLastReliquatModification(reliquatModification);
    item.setReliquat(item.getReliquat() + reliquatModification);
}
```

#### Validation:
- [ ] Formula: `-(Real - Theoretical)`
- [ ] Reliquat updated correctly
- [ ] Last modification tracked

#### Deliverables:
- Reliquat calculation method

---

### W2-T35: Implement FMCalculationService.executeFM()
**Duration**: 2 days
**Dependencies**: W2-T31-34
**Blocking**: ✅ YES

#### Steps:
1. Create complete service:
```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Rune;
import com.dofus.fm.domain.Line;
import com.dofus.fm.network.types.CraftResultMessage;
import com.dofus.fm.network.types.Effect;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@Slf4j
public class FMCalculationService {

    public FMResultType executeFM(Item item, Rune rune, CraftResultMessage craftResult) {

        // Step 1: Update all stat line values from packet
        updateItemStatsFromPacket(item, craftResult.getEffects());

        // Step 2: Set missing lines to 0 (removed stats)
        removeMissingLines(item, craftResult.getEffects());

        // Step 3: Classify result type
        FMResultType resultType = classifyResult(craftResult, item);

        // Step 4: Calculate theoretical weight change
        double theoreticalWeight = calculateTheoreticalWeight(resultType, rune);

        // Step 5: Calculate real weight change
        double realWeight = calculateRealWeight(item);

        // Step 6: Calculate reliquat modification
        calculateReliquatModification(item, theoreticalWeight, realWeight);

        // Step 7: Clean up zero-value exotic lines
        item.cleanLines();

        // Step 8: Log results
        log.info("FM Result: {}", resultType);
        log.info("Theoretical Weight: {}", theoreticalWeight);
        log.info("Real Weight: {}", realWeight);
        log.info("Reliquat Modification: {}", item.getLastReliquatModification());
        log.info("New Reliquat: {}", item.getReliquat());

        return resultType;
    }

    private void updateItemStatsFromPacket(Item item, List<Effect> packetEffects) {
        for (Effect effect : packetEffects) {
            Line existingLine = item.getLineByEffectId(effect.getActionId());

            if (existingLine != null) {
                existingLine.setValue(effect.getValue());
            } else {
                // Add exotic line (TODO: needs EffectRepository)
                Line newLine = new Line(effect.getActionId(), 0, 0, effect.getValue(), effectRepository);
                item.getExoticLines().add(newLine);
            }
        }
    }

    private void removeMissingLines(Item item, List<Effect> packetEffects) {
        List<Integer> idsInPacket = packetEffects.stream()
            .map(Effect::getActionId)
            .collect(Collectors.toList());

        for (Line line : item.getLines()) {
            if (!idsInPacket.contains(line.getEffectId())) {
                line.setValue(0);
            }
        }
    }

    // ... (classification, weight methods from above)
}
```

2. Test all 6 steps execute correctly

#### Validation:
- [ ] All 8 steps execute in order
- [ ] Stats update correctly
- [ ] Missing lines set to 0
- [ ] Classification correct
- [ ] Weights calculated
- [ ] Reliquat updated
- [ ] Exotic lines cleaned

#### Deliverables:
- Complete `FMCalculationService.java`

---

### W2-T36: Implement Item.initLinesUsingPacket()
**Duration**: 1 day
**Dependencies**: W2-T23, W2-T16 (ExchangeObjectParser)
**Blocking**: ✅ YES

#### Steps:
1. Add to Item or create separate service:
```java
public void initializeItemStats(Item item, List<Effect> packetEffects, EffectRepository effectRepository) {
    // Clear exotic lines
    item.getExoticLines().clear();

    for (Effect effect : packetEffects) {
        Line existingLine = item.getLineByEffectId(effect.getActionId());

        if (existingLine != null) {
            // Initialize existing line (no modification tracking)
            existingLine.initValue(effect.getValue());
        } else {
            // Add exotic line
            Line exoticLine = new Line(
                effect.getActionId(),
                0,
                0,
                effect.getValue(),
                effectRepository
            );
            item.getExoticLines().add(exoticLine);
        }
    }
}
```

#### Validation:
- [ ] Clears exotic lines
- [ ] Uses `initValue()` not `setValue()`
- [ ] Adds exotic lines for new stats

#### Deliverables:
- Item initialization method

---

### W2-T37: Write FM Calculation Unit Tests (All 4 Types)
**Duration**: 2 days
**Dependencies**: W2-T35
**Blocking**: ❌ NO

#### Steps:
1. Test Clean Success (SC):
```java
@Test
void testCleanSuccess_ReliquatUnchanged() {
    // Setup: Item with 20 Vitality
    Item item = createTestItem();
    Line vitLine = item.getLineByEffectId(125);
    vitLine.initValue(20);

    // Rune: +10 Vitality (weight 2.5)
    Rune rune = createVitalityRune(10);

    // Result: SUCCESS, +10 Vitality
    CraftResultMessage craftResult = createSuccessResult(
        List.of(new Effect(125, 30))
    );

    // Execute
    FMResultType result = fmService.executeFM(item, rune, craftResult);

    // Assert
    assertEquals(FMResultType.SC, result);
    assertEquals(30, vitLine.getValue());
    assertEquals(10, vitLine.getLastModification());
    assertEquals(0.0, item.getLastReliquatModification(), 0.01);
}
```

2. Test Success with Sink (SN)
3. Test Clean Failure (EC)
4. Test Neutral Failure (EN)

#### Validation:
- [ ] All 4 result types tested
- [ ] Reliquat calculations verified
- [ ] Weight formulas correct

#### Deliverables:
- FM calculation test suite

---

### W2-T38: Validate FM Calculations vs Python
**Duration**: 2 days
**Dependencies**: W2-T37
**Blocking**: ❌ NO

#### Steps:
1. Set up Python logging to export:
   - Initial item state
   - Rune used
   - Result packet
   - Calculated reliquat

2. Replay in Java tests
3. Compare outputs

#### Validation:
- [ ] 100% match for 50+ FM attempts
- [ ] Reliquat tracking identical

#### Deliverables:
- Validation test suite

---

### W2-T39: Implement FMSessionService
**Duration**: 1 day
**Dependencies**: W2-T35, W2-T24, W2-T25
**Blocking**: ✅ YES

#### Steps:
1. Create orchestration service:
```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Rune;
import com.dofus.fm.network.types.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class FMSessionService {

    private final ItemFactory itemFactory;
    private final RuneFactory runeFactory;
    private final FMCalculationService fmCalculationService;
    private final DisplayService displayService;
    private final RuneIdentifier runeIdentifier;

    private Rune currentRune;
    private Item currentItem;

    // Constructor injection...

    public void handleExchangeObject(ExchangeObjectMessage message) {
        Integer objectGID = message.getObjectGID();

        if (runeIdentifier.isRune(objectGID)) {
            currentRune = runeFactory.createRune(objectGID);
            displayService.updateRune(currentRune);
            log.info("Rune selected: {}", currentRune.getName());
        } else {
            currentItem = itemFactory.createItem(objectGID);
            fmCalculationService.initializeItemStats(currentItem, message.getEffects());
            displayService.updateItem(currentItem);
            log.info("Item selected: {}", currentItem.getName());
        }
    }

    public void handleCraftResult(CraftResultMessage craftResult) {
        if (currentItem == null || currentRune == null) {
            log.warn("Received craft result but item or rune is null");
            return;
        }

        FMResultType resultType = fmCalculationService.executeFM(
            currentItem,
            currentRune,
            craftResult
        );

        displayService.updateItem(currentItem);
        log.info("FM Attempt completed: {}", resultType);
    }
}
```

2. Implement RuneIdentifier:
```java
@Component
public class RuneIdentifier {
    private static final Set<Integer> RUNE_IDS = Set.of(
        1557, 7435, 7433, 7438, // ... all rune IDs
    );

    public boolean isRune(int objectGID) {
        return RUNE_IDS.contains(objectGID);
    }
}
```

#### Validation:
- [ ] Handles item selection
- [ ] Handles rune selection
- [ ] Coordinates FM execution

#### Deliverables:
- `FMSessionService.java`
- `RuneIdentifier.java`

---

# AGENT: UI-01 (UI & Display)

## Wave 2 Tasks

### W2-T40: Set Up JavaFX Dependencies
**Duration**: 0.5 day
**Dependencies**: W1-T1
**Blocking**: ✅ YES

#### Steps:
1. Add to `fm-ui/pom.xml`:
```xml
<dependencies>
    <dependency>
        <groupId>org.openjfx</groupId>
        <artifactId>javafx-controls</artifactId>
        <version>21.0.5</version>
    </dependency>
    <dependency>
        <groupId>org.openjfx</groupId>
        <artifactId>javafx-fxml</artifactId>
        <version>21.0.5</version>
    </dependency>
    <dependency>
        <groupId>com.dofus</groupId>
        <artifactId>fm-core</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

2. Test JavaFX hello world

#### Validation:
- [ ] JavaFX dependencies resolve
- [ ] Simple window can open

#### Deliverables:
- JavaFX dependencies configured

---

### W2-T41: Design FXML Layout
**Duration**: 1 day
**Dependencies**: W2-T40
**Blocking**: ✅ YES

#### Steps:
1. Create `fm-ui/src/main/resources/fxml/main.fxml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<?import javafx.scene.control.*?>
<?import javafx.scene.layout.*?>

<VBox xmlns:fx="http://javafx.com/fxml"
      fx:controller="com.dofus.fm.ui.controller.MainController"
      spacing="10" padding="10">

    <!-- Item Info -->
    <HBox spacing="10">
        <Label text="Item :" styleClass="label-header"/>
        <Label fx:id="itemLabel" text="no item"/>
    </HBox>

    <!-- Rune Info -->
    <HBox spacing="10">
        <Label text="Rune :" styleClass="label-header"/>
        <Label fx:id="runeLabel" text="no rune"/>
    </HBox>

    <!-- Stat Lines Table -->
    <TableView fx:id="statLinesTable" VBox.vgrow="ALWAYS">
        <columns>
            <TableColumn fx:id="minColumn" text="Min" prefWidth="50"/>
            <TableColumn fx:id="maxColumn" text="Max" prefWidth="50"/>
            <TableColumn fx:id="effectColumn" text="Effet" prefWidth="200"/>
            <TableColumn fx:id="modifColumn" text="Modif" prefWidth="60"/>
            <TableColumn fx:id="weightColumn" text="Poids" prefWidth="100"/>
        </columns>
    </TableView>

    <!-- Reliquat -->
    <HBox spacing="10">
        <Label text="Reliquat :" styleClass="label-header"/>
        <Label fx:id="reliquatLabel" text="aucun"/>
    </HBox>

</VBox>
```

#### Validation:
- [ ] FXML loads without errors
- [ ] Layout matches specification

#### Deliverables:
- `main.fxml`

---

### W2-T42: Implement FMApplication Entry Point
**Duration**: 0.5 day
**Dependencies**: W2-T40
**Blocking**: ✅ YES

#### Steps:
1. Create `fm-ui/src/main/java/com/dofus/fm/ui/FMApplication.java`:
```java
package com.dofus.fm.ui;

import javafx.application.Application;
import javafx.fxml.FXMLLoader;
import javafx.scene.Scene;
import javafx.stage.Stage;
import org.springframework.boot.SpringApplication;
import org.springframework.context.ConfigurableApplicationContext;

public class FMApplication extends Application {

    private ConfigurableApplicationContext springContext;

    @Override
    public void init() {
        springContext = SpringApplication.run(DofusFMAssistantApplication.class);
    }

    @Override
    public void start(Stage primaryStage) throws Exception {
        FXMLLoader loader = new FXMLLoader(getClass().getResource("/fxml/main.fxml"));
        loader.setControllerFactory(springContext::getBean);

        Scene scene = new Scene(loader.load(), 600, 400);
        primaryStage.setTitle("FM Helper");
        primaryStage.setScene(scene);
        primaryStage.show();
    }

    @Override
    public void stop() {
        springContext.close();
    }

    public static void main(String[] args) {
        launch(args);
    }
}
```

#### Validation:
- [ ] Application starts
- [ ] Spring context loads
- [ ] Window displays

#### Deliverables:
- `FMApplication.java`

---

### W2-T43: Implement MainController Structure
**Duration**: 1 day
**Dependencies**: W2-T41
**Blocking**: ✅ YES

#### Steps:
1. Create controller with all FXML bindings
2. Initialize table columns
3. Set up row factory for color coding

#### Deliverables:
- `MainController.java` (structure)

---

_(Continue with remaining UI tasks W2-T44 through W2-T50...)_

---

## Task Assignment Summary

### Quick Reference Table

| Agent | Wave 1 Tasks | Wave 2 Tasks | Wave 3 Tasks | Wave 4 Tasks | Total |
|-------|--------------|--------------|--------------|--------------|-------|
| INFRA-01 | 4 | 5 | 1 | 3 | 13 |
| DATA-01 | 2 | 6 | 1 | 2 | 11 |
| NETWORK-01 | 2 | 9 | 1 | 1 | 13 |
| DOMAIN-01 | 0 | 9 | 2 | 1 | 12 |
| FMCALC-01 | 0 | 10 | 3 | 0 | 13 |
| UI-01 | 0 | 11 | 2 | 2 | 15 |

**Total Tasks**: 77

---

**This Implementation Book provides detailed steps for each task. Agents should execute tasks in wave order, checking dependencies before starting.**
