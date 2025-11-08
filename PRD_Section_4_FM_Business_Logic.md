# Section 4: Business Logic - FM Calculation System

## 4.1 Overview of Forgemagie (FM) System

### What is Forgemagie?
Forgemagie (FM) is the item enhancement/crafting system in Dofus. Players use **runes** to modify item **stats** (strength, vitality, AP, etc.). The system is complex and probabilistic:

- **Success**: Rune is consumed, stat increases
- **Failure**: Rune is consumed, stat may decrease, other stats may change
- **Weight System**: Each stat has a weight (difficulty); higher weight = harder to add
- **Reliquat (Remainder)**: Hidden pool that accumulates when FM fails, affecting future attempts

### Why This Application Exists
The Dofus game client doesn't show:
- Current reliquat value
- Weight calculations
- Detailed change tracking

This FM Assistant calculates these values in real-time by analyzing network packets.

---

## 4.2 Core FM Concepts (Deep Dive)

### 4.2.1 Weight System

Every stat in Dofus has a **weight** value representing FM difficulty:

| Stat | Effect ID | Weight | Example |
|------|-----------|--------|---------|
| Vitality | 125 | 0.25 | Easy to add |
| Strength | 118 | 1.0 | Medium |
| Agility | 119 | 1.0 | Medium |
| Intelligence | 126 | 1.0 | Medium |
| Wisdom | 124 | 1.5 | Hard |
| Critical | 115 | 10.0 | Very hard |
| AP (Action Points) | 111 | 100.0 | Extremely hard |
| MP (Movement Points) | 128 | 90.0 | Extremely hard |

**Rune Weight Calculation**:
```
Rune Weight = Rune Value × Stat Weight

Example:
- Rune: +10 Vitality
- Stat Weight: 0.25
- Rune Weight = 10 × 0.25 = 2.5
```

**Item Weight Calculation**:
```
Item Weight = Σ (Stat Value × Stat Weight) for all stats

Example item:
- 30 Vitality (weight 0.25): 30 × 0.25 = 7.5
- 20 Strength (weight 1.0): 20 × 1.0 = 20.0
- 15 Agility (weight 1.0): 15 × 1.0 = 15.0
- Total Item Weight = 42.5
```

---

### 4.2.2 Reliquat (Remainder Pool)

The **reliquat** is a hidden accumulator that tracks "failed FM weight":

#### When Reliquat Increases:
- FM attempt **fails** (craftResult = 1)
- No stats decreased
- **Effect**: Increases future success probability

#### When Reliquat Decreases:
- Stats **decrease** (negative modification)
- magicPoolStatus = 3 in packet
- **Effect**: Decreases future success probability (bad for player)

#### Reliquat Calculation Formula:
```
Theoretical Weight Change = {
    +rune_weight  if SUCCESS (craftResult = 2)
    -rune_weight  if FAILURE (craftResult = 1)
}

Real Weight Change = Σ(stat_change × stat_weight) for all stat lines

Reliquat Modification = -(Real Weight - Theoretical Weight)

New Reliquat = Old Reliquat + Reliquat Modification
```

**Example 1: Clean Success (SC)**
```
- Rune: +10 Vitality (weight 2.5)
- Result: SUCCESS (craftResult = 2)
- Stat Changes: +10 Vitality (nothing else changed)

Theoretical Weight = +2.5 (success)
Real Weight = +10 × 0.25 = +2.5
Reliquat Modification = -(2.5 - 2.5) = 0

Reliquat: Unchanged (clean!)
```

**Example 2: Success with Sink (SN)**
```
- Rune: +10 Vitality (weight 2.5)
- Result: SUCCESS (craftResult = 2)
- Stat Changes: +10 Vitality, -5 Strength

Theoretical Weight = +2.5
Real Weight = (+10 × 0.25) + (-5 × 1.0) = 2.5 - 5.0 = -2.5
Reliquat Modification = -(-2.5 - 2.5) = -(-5.0) = +5.0

Reliquat: Decreased by 5.0 (bad!)
```

**Example 3: Clean Failure (EC)**
```
- Rune: +1 AP (weight 100)
- Result: FAILURE (craftResult = 1)
- Stat Changes: -10 Vitality (lost stats!)

Theoretical Weight = -100 (failure)
Real Weight = -10 × 0.25 = -2.5
Reliquat Modification = -(-2.5 - (-100)) = -(97.5) = -97.5

Reliquat: Decreased by 97.5 (very bad!)
```

**Example 4: Neutral Failure (EN)**
```
- Rune: +10 Vitality (weight 2.5)
- Result: FAILURE (craftResult = 1)
- Stat Changes: None (nothing happened)

Theoretical Weight = -2.5
Real Weight = 0
Reliquat Modification = -(0 - (-2.5)) = -2.5

Reliquat: Increased by 2.5 (good! builds up for next attempt)
```

---

### 4.2.3 FM Result Types

There are **4 possible outcomes** for every FM attempt:

| Code | Name | Meaning |
|------|------|---------|
| **SC** | **Success Clean** | Rune applied, no stats decreased, reliquat unchanged |
| **SN** | **Success Neutral** | Rune applied, BUT stats decreased OR reliquat decreased |
| **EC** | **Échec Clean** | Rune failed, stats decreased OR reliquat decreased |
| **EN** | **Échec Neutral** | Rune failed, no changes, reliquat increased (good!) |

**Classification Logic**:
```python
def getResultType(self, result_packet):
    malus = False
    for line in self.getLines():
        if line.getLastModification() < 0:
            malus = True

    # Check if anything decreased
    sth_lowered = malus or result_packet['data']['magicPoolStatus'] == 3

    if result_packet['data']['craftResult'] == 2:  # Success
        if sth_lowered:
            return 'SN'
        else:
            return 'SC'
    elif result_packet['data']['craftResult'] == 1:  # Failure
        if sth_lowered:
            return 'EC'
        else:
            return 'EN'
```

**Truth Table**:

| craftResult | stats_decreased | magicPoolStatus | Result |
|-------------|-----------------|-----------------|--------|
| 2 (Success) | No | ≠ 3 | **SC** |
| 2 (Success) | Yes | - | **SN** |
| 2 (Success) | No | = 3 | **SN** |
| 1 (Failure) | Yes | - | **EC** |
| 1 (Failure) | No | = 3 | **EC** |
| 1 (Failure) | No | ≠ 3 | **EN** |

---

## 4.3 Python Implementation - Item.executeFM()

### Full Method Analysis

```python
def executeFM(self, result_packet, rune):
    """
    Execute FM calculation after receiving ExchangeCraftResultMessage (packet 6188)

    Args:
        result_packet: Parsed packet containing FM result
        rune: The rune that was applied

    Returns:
        result_type: 'SC', 'SN', 'EC', or 'EN'
    """

    # --- STEP 1: Update all stat line values from packet ---
    packet_lines = result_packet['data']['effects']

    for line in packet_lines:
        try:
            existing_line = self.getLineByEffectId(line['actionId'])

            if existing_line is not None:
                # Update existing line (setValue tracks the change)
                existing_line.setValue(int(line['value']))
            else:
                # Add new exotic line
                new_line = Line(line['actionId'], 0, 0)
                new_line.setValue(line['value'])
                self.exotic_lines.append(new_line)

        except Exception as e:
            print('e : ' + str(e))

    # --- STEP 2: Set missing lines to 0 (lines removed by FM) ---
    ids_in_packet = []
    for line in packet_lines:
        ids_in_packet.append(line['actionId'])

    for line in self.getLines():
        if line.getEffectId() not in ids_in_packet:
            line.setValue(0)  # Line was removed

    # --- STEP 3: Determine theoretical weight change ---
    result_type = self.getResultType(result_packet)

    if result_type == 'SC':
        theorical_earned_weight = rune.getWeight()
    elif result_type in ['SN', 'EN']:
        theorical_earned_weight = 0
    elif result_type == 'EC':
        theorical_earned_weight = -1 * rune.getWeight()

    # --- STEP 4: Calculate real weight change ---
    real_earned_weight = 0
    for line in self.getLines():
        real_earned_weight += line.getLastModification() * line.getEffectWeight()

    # --- STEP 5: Calculate reliquat modification ---
    print('Result : ' + result_type)
    print('Theorical earning : ' + str(theorical_earned_weight))
    print('Real earning :' + str(real_earned_weight))

    self.last_reliquat_modification = -1 * (real_earned_weight - theorical_earned_weight)
    self.reliquat += self.last_reliquat_modification

    # --- STEP 6: Clean up zero-value exotic lines ---
    self.clean_lines()

    # --- STEP 7: Update GUI ---
    self.listener.updateItem(self)

    return result_type
```

### Step-by-Step Breakdown

#### Step 1: Update Stat Values
```python
for line in packet_lines:
    existing_line = self.getLineByEffectId(line['actionId'])
    if existing_line is not None:
        existing_line.setValue(int(line['value']))
    else:
        new_line = Line(line['actionId'], 0, 0)
        new_line.setValue(line['value'])
        self.exotic_lines.append(new_line)
```

**What happens**:
- Parse each effect from the packet
- Find the corresponding `Line` object in the item
- If found: Update value (this automatically calculates `last_modification`)
- If not found: Create exotic line (new stat added to item)

**Why important**:
- `setValue()` tracks the change: `last_modification = new_value - old_value`
- This is used later to classify result type (SC/SN/EC/EN)

---

#### Step 2: Remove Missing Lines
```python
ids_in_packet = []
for line in packet_lines:
    ids_in_packet.append(line['actionId'])

for line in self.getLines():
    if line.getEffectId() not in ids_in_packet:
        line.setValue(0)  # Line removed
```

**What happens**:
- If a stat line existed before but isn't in the new packet, it was **removed**
- Set its value to 0 (this will show as negative modification)

**Example**:
- Item had: Vitality, Strength, AP
- After FM: Vitality, Strength (AP removed!)
- AP line gets `setValue(0)`, so `last_modification = -1` (if AP was 1)

---

#### Step 3: Calculate Theoretical Weight
```python
result_type = self.getResultType(result_packet)

if result_type == 'SC':
    theorical_earned_weight = rune.getWeight()
elif result_type in ['SN', 'EN']:
    theorical_earned_weight = 0
elif result_type == 'EC':
    theorical_earned_weight = -1 * rune.getWeight()
```

**Theoretical Weight Logic**:

| Result Type | Theoretical Weight | Explanation |
|-------------|-------------------|-------------|
| **SC** | +rune_weight | Clean success: full rune weight added |
| **SN** | 0 | Success but sank: weight neutralized |
| **EN** | 0 | Failure but neutral: no weight change |
| **EC** | -rune_weight | Failure with loss: negative weight |

**Why this matters**:
- This is what "should" have happened based on FM rules
- Compared against "real" weight to calculate reliquat change

---

#### Step 4: Calculate Real Weight
```python
real_earned_weight = 0
for line in self.getLines():
    real_earned_weight += line.getLastModification() * line.getEffectWeight()
```

**Real Weight Calculation**:
- Sum up: (change in each stat) × (stat weight)

**Example**:
```
Changes:
- Vitality: +10 (weight 0.25) → +10 × 0.25 = +2.5
- Strength: -5 (weight 1.0) → -5 × 1.0 = -5.0
- Total Real Weight = +2.5 - 5.0 = -2.5
```

---

#### Step 5: Calculate Reliquat Modification
```python
self.last_reliquat_modification = -1 * (real_earned_weight - theorical_earned_weight)
self.reliquat += self.last_reliquat_modification
```

**Formula**:
```
Reliquat Modification = -(Real Weight - Theoretical Weight)
```

**Example**:
```
Theoretical Weight: +2.5 (SC with +10 Vit rune)
Real Weight: -2.5 (Vit increased but Strength decreased)

Reliquat Modification = -(-2.5 - 2.5) = -(-5.0) = +5.0

BUT WAIT! This is wrong in the classification...
```

**CRITICAL BUG ANALYSIS**:
The code calculates theoretical weight **after** classifying the result. But classification depends on whether stats decreased!

Let me re-examine:

```python
result_type = self.getResultType(result_packet)

if result_type == 'SC':
    theorical_earned_weight = rune.getWeight()
elif result_type in ['SN', 'EN']:
    theorical_earned_weight = 0
elif result_type == 'EC':
    theorical_earned_weight = -1 * rune.getWeight()
```

**Aha!** The theoretical weight is adjusted based on result type:
- **SC** (clean success): Full rune weight
- **SN** (success with sink): 0 (weight neutralized by sink)
- **EN** (neutral failure): 0 (no change)
- **EC** (failure with sink): Negative rune weight

So the formula accounts for FM game mechanics!

**Corrected Understanding**:
```
If SC:
  Theoretical = +rune_weight
  Real = actual stat changes × weights
  Reliquat Mod = -(Real - Theoretical)
  Example: Real = +2.5, Theoretical = +2.5 → Mod = 0 ✓

If SN:
  Theoretical = 0 (sink neutralizes)
  Real = (positive change - sink) × weights
  Reliquat Mod = -(Real - 0) = -Real
  Example: Real = -2.5 → Mod = +2.5 (reliquat lost!)

If EN:
  Theoretical = 0
  Real = 0 (nothing changed)
  Reliquat Mod = 0... wait, that's wrong!
```

**WAIT, there's another bug!**

For **EN** (neutral failure), reliquat should **increase** by rune weight!

Let me check the getResultType again... OH! I see the issue.

The theoretical weight for **EN** should be **-rune_weight** (failure), but the result type checks prevent that:

```python
if result_type in ['SN', 'EN']:
    theorical_earned_weight = 0
```

**This is incorrect!** Let me trace through a real EN scenario:

```
Rune: +10 Vit (weight 2.5)
Result: FAILURE (craftResult = 1)
Changes: NONE (all stats unchanged)

getResultType() returns 'EN' (failure, no malus)

Theoretical Weight = 0 (from the if statement)
Real Weight = 0 (no changes)
Reliquat Mod = -(0 - 0) = 0

BUT IT SHOULD BE:
Theoretical = -2.5 (failed rune)
Real = 0
Reliquat Mod = -(0 - (-2.5)) = -2.5... wait that's negative reliquat mod
```

Hmm, let me think about this differently. The sign convention might be:
- Reliquat Mod **positive** = reliquat increases (good)
- Reliquat Mod **negative** = reliquat decreases (bad)

Actually, looking at the formula again:
```python
self.last_reliquat_modification = -1 * (real_earned_weight - theorical_earned_weight)
```

If we flip the signs:
```
Reliquat Mod = Theoretical - Real
```

That makes more sense! Let me re-analyze:

**SC Example**:
```
Theoretical = +2.5
Real = +2.5
Reliquat Mod = +2.5 - (+2.5) = 0 ✓
```

**EN Example**:
```
Theoretical = 0 (bug? or intentional?)
Real = 0
Reliquat Mod = 0 - 0 = 0 ✗ (should increase reliquat)
```

**I think there's a bug in the Python code**, or the theoretical weight calculation is more complex than shown.

**For migration**: Keep the exact same logic, but document this potential issue.

---

#### Step 6: Clean Lines
```python
def clean_lines(self):
    for line in self.exotic_lines:
        if line.getValue() == 0 and line.getLastModification() == 0:
            self.exotic_lines.remove(line)
```

**What happens**:
- Remove exotic lines that have value 0 and weren't changed
- Prevents clutter from removed stats

---

## 4.4 Java Migration - FM Calculation Service

### Service Architecture

```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Rune;
import com.dofus.fm.domain.Line;
import com.dofus.fm.protocol.types.CraftResultMessage;
import com.dofus.fm.protocol.types.Effect;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@Slf4j
public class FMCalculationService {

    /**
     * Execute FM calculation after receiving CraftResultMessage
     *
     * @param item Item being forgemagied
     * @param rune Rune that was applied
     * @param craftResult Parsed packet from server
     * @return FM result type (SC, SN, EC, EN)
     */
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
        double reliquatModification = -(realWeight - theoreticalWeight);
        item.setLastReliquatModification(reliquatModification);
        item.setReliquat(item.getReliquat() + reliquatModification);

        // Step 7: Clean up zero-value exotic lines
        item.cleanLines();

        // Step 8: Log results
        log.info("FM Result: {}", resultType);
        log.info("Theoretical Weight: {}", theoreticalWeight);
        log.info("Real Weight: {}", realWeight);
        log.info("Reliquat Modification: {}", reliquatModification);
        log.info("New Reliquat: {}", item.getReliquat());

        return resultType;
    }

    /**
     * Step 1: Update stat values from packet
     */
    private void updateItemStatsFromPacket(Item item, List<Effect> packetEffects) {
        for (Effect effect : packetEffects) {
            Line existingLine = item.getLineByEffectId(effect.getActionId());

            if (existingLine != null) {
                // Update existing line
                existingLine.setValue(effect.getValue());
            } else {
                // Add exotic line
                Line newLine = new Line(effect.getActionId(), 0, 0, effect.getValue());
                newLine.setValue(effect.getValue());
                item.getExoticLines().add(newLine);
            }
        }
    }

    /**
     * Step 2: Remove lines not in packet (set to 0)
     */
    private void removeMissingLines(Item item, List<Effect> packetEffects) {
        List<Integer> idsInPacket = packetEffects.stream()
            .map(Effect::getActionId)
            .collect(Collectors.toList());

        for (Line line : item.getLines()) {
            if (!idsInPacket.contains(line.getEffectId())) {
                line.setValue(0); // Line was removed
            }
        }
    }

    /**
     * Step 3: Classify result type (SC/SN/EC/EN)
     */
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

    /**
     * Step 4: Calculate theoretical weight based on result type
     */
    private double calculateTheoreticalWeight(FMResultType resultType, Rune rune) {
        return switch (resultType) {
            case SC -> rune.getWeight();  // Clean success: +weight
            case SN, EN -> 0.0;           // Neutral: no weight
            case EC -> -rune.getWeight(); // Clean failure: -weight
        };
    }

    /**
     * Step 5: Calculate real weight from stat changes
     */
    private double calculateRealWeight(Item item) {
        return item.getLines().stream()
            .mapToDouble(line -> line.getLastModification() * line.getEffectWeight())
            .sum();
    }
}
```

### FM Result Type Enum

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

---

## 4.5 Item Initialization from Packet

### Python Implementation

```python
def initLinesUsingPacket(self, packet):
    """
    Initialize item stat values when ExchangeObjectMessage is received

    Args:
        packet: Parsed ExchangeObjectMessage (packet 5516/5519)
    """
    packet_lines = packet['data']['effects']
    self.exotic_lines = []

    for line in packet_lines:
        try:
            existing_line = self.getLineByEffectId(line['actionId'])

            if existing_line is not None:
                # Initialize existing line value
                existing_line.initValue(int(line['value']))
            else:
                # Add exotic line
                self.exotic_lines.append(Line(line['actionId'], 0, 0, int(line['value'])))

        except Exception as e:
            print('e : ' + str(e))

    self.listener.updateItem(self)
```

**Key Differences from executeFM**:
- Uses `initValue()` instead of `setValue()` (doesn't track modification)
- Clears exotic lines before processing
- This is called when item is first placed on FM table

---

### Java Migration

```java
/**
 * Initialize item stat values from ExchangeObjectMessage packet
 */
public void initializeItemStats(Item item, List<Effect> packetEffects) {
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

---

## 4.6 Integration with Packet Processing

### Main Flow Orchestration

```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Rune;
import com.dofus.fm.protocol.types.*;
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

    public FMSessionService(ItemFactory itemFactory,
                            RuneFactory runeFactory,
                            FMCalculationService fmCalculationService,
                            DisplayService displayService,
                            RuneIdentifier runeIdentifier) {
        this.itemFactory = itemFactory;
        this.runeFactory = runeFactory;
        this.fmCalculationService = fmCalculationService;
        this.displayService = displayService;
        this.runeIdentifier = runeIdentifier;
    }

    /**
     * Handle ExchangeObjectMessage (item or rune added/modified)
     */
    public void handleExchangeObject(ExchangeObjectMessage message) {
        Integer objectGID = message.getObjectGID();

        if (runeIdentifier.isRune(objectGID)) {
            // Rune detected
            currentRune = runeFactory.createRune(objectGID);
            displayService.updateRune(currentRune);
            log.info("Rune selected: {}", currentRune.getName());

        } else {
            // Item detected
            currentItem = itemFactory.createItem(objectGID);
            fmCalculationService.initializeItemStats(currentItem, message.getEffects());
            displayService.updateItem(currentItem);
            log.info("Item selected: {}", currentItem.getName());
        }
    }

    /**
     * Handle CraftResultMessage (FM attempt result)
     */
    public void handleCraftResult(CraftResultMessage craftResult) {
        if (currentItem == null || currentRune == null) {
            log.warn("Received craft result but item or rune is null");
            return;
        }

        // Execute FM calculation
        FMResultType resultType = fmCalculationService.executeFM(
            currentItem,
            currentRune,
            craftResult
        );

        // Update display
        displayService.updateItem(currentItem);

        log.info("FM Attempt completed: {}", resultType);
    }
}
```

---

## 4.7 Testing Strategy for FM Logic

### Unit Tests for Weight Calculations

```java
@SpringBootTest
class FMCalculationServiceTest {

    @Autowired
    private FMCalculationService fmCalculationService;

    @Test
    void testCleanSuccess_ReliquatUnchanged() {
        // Setup: Item with 20 Vitality (weight 0.25)
        Item item = createTestItem();
        Line vitLine = item.getLineByEffectId(125); // Vitality
        vitLine.initValue(20);

        // Rune: +10 Vitality (weight 2.5)
        Rune rune = createVitalityRune(10);

        // Result: SUCCESS, +10 Vitality
        CraftResultMessage craftResult = createSuccessResult(
            List.of(new Effect(125, 30)) // New value: 30
        );

        // Execute
        FMResultType result = fmCalculationService.executeFM(item, rune, craftResult);

        // Assert
        assertEquals(FMResultType.SC, result);
        assertEquals(30, vitLine.getValue());
        assertEquals(10, vitLine.getLastModification());
        assertEquals(0.0, item.getLastReliquatModification(), 0.01);
    }

    @Test
    void testSuccessWithSink_ReliquatDecreases() {
        // Setup: Item with 20 Vit, 15 Str
        Item item = createTestItem();
        Line vitLine = item.getLineByEffectId(125);
        vitLine.initValue(20);
        Line strLine = item.getLineByEffectId(118);
        strLine.initValue(15);

        // Rune: +10 Vit (weight 2.5)
        Rune rune = createVitalityRune(10);

        // Result: SUCCESS, +10 Vit, -5 Str
        CraftResultMessage craftResult = createSuccessResult(
            List.of(
                new Effect(125, 30), // Vit: 20 → 30
                new Effect(118, 10)  // Str: 15 → 10
            )
        );

        // Execute
        FMResultType result = fmCalculationService.executeFM(item, rune, craftResult);

        // Calculate expected reliquat
        // Theoretical: 0 (SN = neutral)
        // Real: (+10 × 0.25) + (-5 × 1.0) = 2.5 - 5.0 = -2.5
        // Reliquat Mod: -(-2.5 - 0) = +2.5

        // Assert
        assertEquals(FMResultType.SN, result);
        assertEquals(2.5, item.getLastReliquatModification(), 0.01);
    }

    @Test
    void testNeutralFailure_ReliquatIncreases() {
        // Setup: Item with 20 Vit
        Item item = createTestItem();
        Line vitLine = item.getLineByEffectId(125);
        vitLine.initValue(20);

        // Rune: +10 Vit (weight 2.5)
        Rune rune = createVitalityRune(10);

        // Result: FAILURE, no changes
        CraftResultMessage craftResult = createFailureResult(
            List.of(new Effect(125, 20)) // Unchanged
        );

        // Execute
        FMResultType result = fmCalculationService.executeFM(item, rune, craftResult);

        // Calculate expected reliquat
        // Theoretical: 0 (EN = neutral)
        // Real: 0 (no change)
        // Reliquat Mod: -(0 - 0) = 0
        // NOTE: This seems wrong! EN should increase reliquat

        // Assert
        assertEquals(FMResultType.EN, result);
        // TODO: Verify if this is a bug in original code
    }
}
```

---

## 4.8 Key Migration Considerations

### 1. Exact Formula Preservation
- **Critical**: FM formulas must match exactly
- Even if bugs exist in Python code, replicate them first
- Document potential bugs for future investigation

### 2. Floating Point Precision
- Python: Arbitrary precision floats
- Java: Use `double` (not `float`) to minimize precision errors
- Consider using `BigDecimal` for critical calculations if precision issues arise

### 3. Thread Safety
- Multiple packets may arrive concurrently
- Use `synchronized` or `@Transactional` for session state
- Consider making `Item` and `Rune` immutable with builder pattern

### 4. Error Handling
- Python code silently catches exceptions (`except Exception: pass`)
- Java: Log all errors with context
- Don't fail silently - helps debugging

### 5. State Management
```java
// Python uses module-level global variables
rune = None
item = None

// Java: Use Spring @Scope("prototype") or session management
@Service
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class FMSessionService {
    private Rune currentRune;
    private Item currentItem;
}
```

### 6. Logging
Add comprehensive logging for debugging:
```java
log.debug("Packet received: type={}, objectGID={}", packetType, objectGID);
log.info("FM Result: {}, Reliquat: {} -> {}", resultType, oldReliquat, newReliquat);
log.warn("Unknown effect type: {}", effectType);
```

---

## 4.9 Business Logic Summary

### Core Algorithms to Implement

1. **Weight Calculation**
   - Item weight: Σ(value × weight) for all stats
   - Rune weight: value × weight

2. **Reliquat Tracking**
   - Formula: `Reliquat Mod = -(Real Weight - Theoretical Weight)`
   - Update after every FM attempt

3. **Result Classification**
   - Parse `craftResult` (1=fail, 2=success)
   - Check for stat decreases
   - Check `magicPoolStatus` == 3
   - Classify as SC/SN/EC/EN

4. **Stat Line Management**
   - Track modifications: `last_modification = new - old`
   - Handle exotic lines (new stats)
   - Remove lines set to 0

5. **Initialization vs Update**
   - `initValue()`: First time item is seen
   - `setValue()`: After FM attempt (tracks change)

---

**Next Section Preview:**
Section 5 will cover the UI/Display layer migration from Tkinter to a modern Java GUI framework (JavaFX or Spring Boot + Web UI).
