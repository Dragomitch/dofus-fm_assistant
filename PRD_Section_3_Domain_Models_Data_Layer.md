# Section 3: Domain Models & Data Layer

## 3.1 Overview

The application uses three core domain models:
1. **Item**: Represents a Dofus item being forgemagied (enhanced)
2. **Rune**: Represents a consumable rune used to modify items
3. **Line**: Represents a stat line/effect on an item (e.g., "+25 Vitality")

All models interact with a SQLite database containing static game data.

---

## 3.2 Database Schema (SQLite)

### Current Database Structure

```sql
-- Stores item descriptions (names)
CREATE TABLE description (
    id INTEGER PRIMARY KEY ASC,
    description_text TEXT
);

-- Stores stat effects metadata
CREATE TABLE effect (
    id INTEGER PRIMARY KEY ASC,
    description_id INTEGER,
    weight INTEGER,  -- FM difficulty weight
    FOREIGN KEY (description_id) REFERENCES description(id)
);

-- Stores items (equipment, runes)
CREATE TABLE item (
    id INTEGER PRIMARY KEY ASC,
    description_id INTEGER,
    level INTEGER,
    icon_id INTEGER,
    FOREIGN KEY (description_id) REFERENCES description(id)
);

-- Stores possible stat ranges for effects
CREATE TABLE effect_line (
    id INTEGER PRIMARY KEY ASC,
    effect_id INTEGER,
    min INTEGER,  -- Minimum possible value
    max INTEGER,  -- Maximum possible value
    FOREIGN KEY (effect_id) REFERENCES effect(id)
);

-- Junction table: Links items to their possible effects
CREATE TABLE item_effect_line (
    item_id INTEGER,
    effect_line_id INTEGER,
    FOREIGN KEY (item_id) REFERENCES item(id),
    FOREIGN KEY (effect_line_id) REFERENCES effect_line(id)
);
```

### Database Initialization

The `init/init.py` script populates the database from Dofus game data files:

1. **Source Files**:
   - `raw_data/i18n_fr.json` → Text descriptions
   - `raw_data/output/effects.json` → Effect metadata with weights
   - `raw_data/output/items.json` → Item data with possible effects

2. **Process**:
   - Load JSON files
   - Populate `description` table
   - Populate `effect` table with weights
   - Populate `item` table
   - Create `effect_line` entries for each item's possible stats
   - Link items to effects via `item_effect_line`

### Sample Data Queries

**Get item information**:
```sql
SELECT item.id, item.level, description.description_text
FROM item, description
WHERE item.id = ? AND item.description_id = description.id;
```

**Get item's possible stat lines**:
```sql
SELECT e.id, el.min, el.max
FROM item_effect_line iel, effect_line el, effect e
WHERE iel.item_id = ?
  AND iel.effect_line_id = el.id
  AND el.effect_id = e.id;
```

**Get rune/effect metadata**:
```sql
SELECT effect.id, effect_line.min, effect.weight, description.description_text
FROM item_effect_line, effect_line, effect, description
WHERE item_effect_line.item_id = ?
  AND item_effect_line.effect_line_id = effect_line.id
  AND effect_line.effect_id = effect.id
  AND effect.description_id = description.id;
```

---

## 3.3 Domain Model: Line (Stat Line)

### File: `line.py`

### Purpose
Represents a single stat line/effect on an item (e.g., "+25 Vitality", "10 AP").

### Python Implementation

```python
class Line:
    def __init__(self, effect_id, mini, maxi, value=0):
        # Fetch metadata from database
        connection = sqlite3.connect('database.sqlite')
        c = connection.cursor()
        result = c.execute(
            'SELECT e.weight, d.description_text FROM effect e, description d '
            'WHERE e.id=? AND e.description_id = d.id',
            [effect_id]
        ).fetchone()
        connection.close()

        self.effect_id = effect_id        # Stat type (e.g., 125 = Vitality)
        self.effect_weight = result[0]    # FM weight (difficulty)
        self.min = mini                   # Min possible value
        self.max = maxi                   # Max possible value
        self.description = result[1]      # Human-readable text
        self.value = value                # Current actual value
        self.last_modification = 0        # Change from last FM attempt

    # Getters
    def getEffectId(self):
        return self.effect_id

    def getEffectWeight(self):
        return self.effect_weight

    def getWeight(self):
        """Calculate current weight: effect_weight × current_value"""
        return self.effect_weight * self.value

    def getMaxWeight(self):
        """Calculate max possible weight: effect_weight × max_value"""
        return self.effect_weight * self.max

    def getValue(self):
        return self.value

    def setValue(self, value):
        """Update value and track the change"""
        self.last_modification = value - self.value
        self.value = value

    def initValue(self, value):
        """Set initial value without tracking change"""
        self.value = value

    def getLastModification(self):
        return self.last_modification

    def isOvermax(self):
        """Check if value exceeds maximum (exotic line)"""
        return self.value > self.max

    def getDescription(self):
        """Replace placeholder with actual value"""
        return self.description.replace("#1{~1~2 à }#2", str(self.value))
```

### Key Concepts

#### 1. **Effect ID** (Stat Type)
Each stat in Dofus has a unique ID:
- 125 = Vitality
- 118 = Strength
- 119 = Agility
- 126 = Intelligence
- 111 = AP (Action Points)
- 128 = MP (Movement Points)
- etc.

#### 2. **Weight** (FM Difficulty)
Each stat has a weight value that determines FM difficulty:
- Vitality: Weight ~0.25 (easy to add)
- Strength/Agi/Int: Weight ~1.0 (medium)
- AP: Weight ~100 (very hard)
- Critical: Weight ~10 (hard)

**Formula**: A rune that gives +10 Vitality (weight 0.25) has total weight = 10 × 0.25 = **2.5**

#### 3. **Min/Max Range**
Items have predefined stat ranges:
- Example: "20 to 35 Vitality"
  - min = 20
  - max = 35
  - Current value could be anywhere from 20-35 (or beyond if exotic)

#### 4. **Exotic Lines**
Lines that exceed the maximum are called "exotic" (overmaxed):
- Item normally has "max 30 Strength"
- Player FMs it to 35 Strength → Exotic!
- `isOvermax()` returns true

#### 5. **Last Modification Tracking**
When a FM attempt occurs:
- `setValue(new_value)` is called
- `last_modification = new_value - old_value`
- Used to calculate result type (SC/SN/EC/EN)
- Positive = stat increased (green in UI)
- Negative = stat decreased (red in UI)

### Negative Effects Mapping

Some stats can be negative (penalties):

```python
NEGATIVE_TO_POSITIVE = {
    116: 117,   # - Portée → + Portée (Range)
    145: 112,   # - Dommages → + Dommages (Damage)
    152: 123,   # - Chance → + Chance
    153: 125,   # - Vitalité → + Vitalité (Vitality)
    # ... (40+ mappings)
}
```

**Note**: The current implementation has `isNegative(effect_id)` method but it's not used (bug?). This mapping might be needed for future features.

---

### Java Migration: Line Entity

```java
package com.dofus.fm.domain;

import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * Represents a stat line/effect on an item.
 * This is a transient entity (not persisted) created from parsed packets.
 */
@Data
@NoArgsConstructor
public class Line {

    /**
     * Effect ID (stat type, e.g., 125 = Vitality)
     */
    private Integer effectId;

    /**
     * Effect weight (FM difficulty multiplier)
     */
    private Double effectWeight;

    /**
     * Minimum possible value for this stat
     */
    private Integer min;

    /**
     * Maximum possible value for this stat
     */
    private Integer max;

    /**
     * Human-readable description template (e.g., "#1{~1~2 à }#2 Vitalité")
     */
    private String descriptionTemplate;

    /**
     * Current actual value of this stat
     */
    private Integer value;

    /**
     * Change from last FM attempt (positive = increased, negative = decreased)
     */
    private Integer lastModification;

    /**
     * Constructor that fetches metadata from database
     */
    public Line(Integer effectId, Integer min, Integer max, Integer value,
                EffectRepository effectRepository) {
        this.effectId = effectId;
        this.min = min;
        this.max = max;
        this.value = value;
        this.lastModification = 0;

        // Fetch effect metadata
        Effect effect = effectRepository.findById(effectId)
            .orElseThrow(() -> new IllegalArgumentException("Unknown effect ID: " + effectId));

        this.effectWeight = effect.getWeight();
        this.descriptionTemplate = effect.getDescription().getDescriptionText();
    }

    /**
     * Calculate current weight: effect_weight × value
     */
    public Double getWeight() {
        return effectWeight * value;
    }

    /**
     * Calculate maximum possible weight: effect_weight × max
     */
    public Double getMaxWeight() {
        return effectWeight * max;
    }

    /**
     * Get human-readable description with value substituted
     */
    public String getDescription() {
        return descriptionTemplate.replace("#1{~1~2 à }#2", String.valueOf(value));
    }

    /**
     * Update value and track the change
     */
    public void setValue(Integer newValue) {
        this.lastModification = newValue - this.value;
        this.value = newValue;
    }

    /**
     * Set initial value without tracking change
     */
    public void initValue(Integer newValue) {
        this.value = newValue;
    }

    /**
     * Check if this is an exotic (overmaxed) line
     */
    public boolean isOvermax() {
        return value > max;
    }

    /**
     * Check if this effect is a negative stat
     */
    public boolean isNegative() {
        return NegativeEffectMapping.isNegative(effectId);
    }
}
```

### Negative Effect Mapping (Constants)

```java
package com.dofus.fm.domain;

import java.util.Map;

public class NegativeEffectMapping {

    private static final Map<Integer, Integer> NEGATIVE_TO_POSITIVE = Map.ofEntries(
        Map.entry(116, 117),   // - Range
        Map.entry(145, 112),   // - Damage
        Map.entry(152, 123),   // - Chance
        Map.entry(153, 125),   // - Vitality
        Map.entry(154, 119),   // - Agility
        Map.entry(155, 126),   // - Intelligence
        Map.entry(156, 124),   // - Wisdom
        Map.entry(157, 118),   // - Strength
        Map.entry(159, 158),   // - Pods
        Map.entry(162, 160),   // - AP Dodge
        Map.entry(163, 161),   // - MP Dodge
        Map.entry(168, 111),   // - AP
        Map.entry(169, 128),   // - MP
        Map.entry(171, 115),   // - Critical
        Map.entry(175, 174),   // - Initiative
        Map.entry(177, 176),   // - Prospecting
        Map.entry(179, 178),   // - Heals
        Map.entry(186, 138),   // - Power
        Map.entry(215, 210),   // - Earth Res %
        Map.entry(216, 211),   // - Water Res %
        Map.entry(217, 212),   // - Air Res %
        Map.entry(218, 213),   // - Fire Res %
        Map.entry(219, 214),   // - Neutral Res %
        Map.entry(245, 240),   // - Earth Res
        Map.entry(246, 241),   // - Water Res
        Map.entry(247, 242),   // - Air Res
        Map.entry(248, 243),   // - Fire Res
        Map.entry(249, 244),   // - Neutral Res
        Map.entry(411, 410),   // - AP Loss
        Map.entry(413, 412),   // - MP Loss
        Map.entry(415, 414),   // - Pushback Damage
        Map.entry(417, 416),   // - Pushback Res
        Map.entry(419, 418),   // - Critical Damage
        Map.entry(421, 420),   // - Critical Res
        Map.entry(423, 422),   // - Earth Damage
        Map.entry(425, 424),   // - Fire Damage
        Map.entry(427, 426),   // - Water Damage
        Map.entry(429, 428),   // - Air Damage
        Map.entry(431, 430),   // - Neutral Damage
        Map.entry(754, 752),   // - Lock
        Map.entry(755, 753),   // - Dodge
        Map.entry(2801, 2800), // - Melee Damage %
        Map.entry(2802, 2803), // - Melee Res %
        Map.entry(2805, 2804), // - Ranged Damage %
        Map.entry(2806, 2807), // - Ranged Res %
        Map.entry(2809, 2808), // - Weapon Damage %
        Map.entry(2813, 2812)  // - Spell Damage %
    );

    public static boolean isNegative(Integer effectId) {
        return NEGATIVE_TO_POSITIVE.containsKey(effectId);
    }

    public static Integer getPositiveCounterpart(Integer negativeEffectId) {
        return NEGATIVE_TO_POSITIVE.get(negativeEffectId);
    }
}
```

---

## 3.4 Domain Model: Rune

### File: `rune.py`

### Purpose
Represents a rune item used to enhance equipment in the FM process.

### Python Implementation

```python
class Rune:
    def __init__(self, id, listener):
        self.listener = listener  # Display object for GUI updates

        # Fetch rune data from database
        connection = sqlite3.connect('database.sqlite')
        c = connection.cursor()

        result = c.execute(
            'SELECT item.id, item.icon_id, description.description_text '
            'FROM item, description '
            'WHERE item.id=? AND item.description_id = description.id',
            [id]
        ).fetchone()

        self.id = id
        self.name = result[2]

        # Fetch the rune's effect
        c.execute(
            'SELECT effect.id, effect_line.min, effect.weight, description.description_text '
            'FROM item_effect_line, effect_line, effect, description '
            'WHERE item_effect_line.item_id=? '
            '  AND item_effect_line.effect_line_id=effect_line.id '
            '  AND effect_line.effect_id=effect.id '
            '  AND effect.description_id=description.id',
            [id]
        )
        result = c.fetchone()

        self.effect_id = result[0]           # What stat it modifies
        self.effect_value = result[1]        # How much it adds
        self.effect_weight = result[2]       # Weight per point
        self.description = result[3].replace("#1{~1~2 à }#2", str(self.effect_value))

        self.listener.updateRune(self)  # Update GUI
        connection.close()

    def getWeight(self):
        """Calculate total rune weight: value × weight"""
        return int(self.effect_value * self.effect_weight)
```

### Key Concepts

#### Rune Weight Calculation
- A rune that gives **+10 Vitality** with weight **0.25**:
  - Total weight = 10 × 0.25 = **2.5**
- A rune that gives **+1 AP** with weight **100**:
  - Total weight = 1 × 100 = **100**

**This weight is critical for FM calculations!**

---

### Java Migration: Rune Entity

```java
package com.dofus.fm.domain;

import lombok.Data;

/**
 * Represents a rune used to enhance items.
 * Runes are consumed when applied and add (or attempt to add) stats to items.
 */
@Data
public class Rune {

    /**
     * Rune item ID (from database)
     */
    private Integer id;

    /**
     * Rune name (e.g., "Ra Vi" for Vitality rune)
     */
    private String name;

    /**
     * Effect ID this rune modifies (e.g., 125 = Vitality)
     */
    private Integer effectId;

    /**
     * Value the rune adds (e.g., 10 for +10 Vitality)
     */
    private Integer effectValue;

    /**
     * Weight per point of this effect
     */
    private Double effectWeight;

    /**
     * Human-readable description
     */
    private String description;

    /**
     * Calculate total rune weight: value × weight
     */
    public Integer getWeight() {
        return (int) (effectValue * effectWeight);
    }
}
```

### Rune Factory/Service

```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Rune;
import com.dofus.fm.repository.*;
import org.springframework.stereotype.Service;

@Service
public class RuneFactory {

    private final ItemRepository itemRepository;
    private final EffectRepository effectRepository;

    public RuneFactory(ItemRepository itemRepository,
                       EffectRepository effectRepository) {
        this.itemRepository = itemRepository;
        this.effectRepository = effectRepository;
    }

    /**
     * Create a Rune object from database using item ID
     */
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

---

## 3.5 Domain Model: Item

### File: `item.py`

### Purpose
Represents a Dofus equipment item being forgemagied. This is the most complex domain model.

### Python Implementation (Simplified)

```python
class Item:
    def __init__(self, id, listener):
        self.listener = listener

        # Fetch item metadata
        connection = sqlite3.connect('database.sqlite')
        c = connection.cursor()

        result = c.execute(
            'SELECT item.id, item.level, item.icon_id, description.description_text '
            'FROM item, description '
            'WHERE item.id=? AND item.description_id=description.id',
            [id]
        ).fetchone()

        self.id = id
        self.level = result[1]
        self.name = result[3]

        # Fetch original stat lines (base item stats)
        self.original_lines = []
        lines = c.execute(
            'SELECT e.id, el.min, el.max '
            'FROM item_effect_line iel, effect_line el, effect e '
            'WHERE iel.item_id=? '
            '  AND iel.effect_line_id=el.id '
            '  AND el.effect_id=e.id',
            [id]
        ).fetchall()

        for line in lines:
            if line[0] not in [983, 984]:  # Skip "Exchangeable" flags
                self.original_lines.append(Line(line[0], line[1], line[2]))

        # Exotic lines (overmaxed or new stats not in base item)
        self.exotic_lines = []

        # Reliquat tracking
        self.reliquat = 0
        self.last_reliquat_modification = 0

        connection.close()

    def getLineByEffectId(self, effect_id):
        """Find a stat line by its effect ID"""
        for line in self.getLines():
            if line.getEffectId() == effect_id:
                return line

    def getWeight(self):
        """Calculate total item weight: sum of all line weights"""
        total = 0
        for line in self.getLines():
            total += line.getWeight()
        return total

    def initLinesUsingPacket(self, packet):
        """Initialize item stat values from ExchangeObjectMessage packet"""
        packet_lines = packet['data']['effects']
        self.exotic_lines = []

        for line in packet_lines:
            existing_line = self.getLineByEffectId(line['actionId'])

            if existing_line is not None:
                # Update existing line
                existing_line.initValue(int(line['value']))
            else:
                # Add exotic line
                self.exotic_lines.append(Line(line['actionId'], 0, 0, int(line['value'])))

        self.listener.updateItem(self)

    def executeFM(self, result_packet, rune):
        """Execute FM calculation after receiving CraftResultMessage"""
        # (Covered in Section 4 - Business Logic)
        pass
```

### Key Concepts

#### 1. **Original Lines vs Exotic Lines**
- **Original Lines**: Base stats the item normally has (from database)
  - Example: Item has "20-35 Vitality" as a base stat
- **Exotic Lines**: Stats added through FM that exceed max or aren't normally on the item
  - Example: Player FMs Vitality to 40 (exceeds 35 max)
  - Example: Player adds AP to an item that normally has no AP

#### 2. **Reliquat (Remainder Pool)**
- Tracks "failed weight" that accumulates over FM attempts
- Increases when FM fails
- Decreases when stats decrease
- **Critical for understanding FM mechanics**

#### 3. **Item Weight**
Total weight = sum of (each stat value × its weight)

Example:
- 30 Vitality (weight 0.25): 30 × 0.25 = 7.5
- 20 Strength (weight 1.0): 20 × 1.0 = 20
- **Total item weight**: 27.5

---

### Java Migration: Item Entity

```java
package com.dofus.fm.domain;

import lombok.Data;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

/**
 * Represents a Dofus equipment item being forgemagied.
 * Contains original stat lines, exotic lines, and reliquat tracking.
 */
@Data
public class Item {

    /**
     * Item ID (from database)
     */
    private Integer id;

    /**
     * Item level
     */
    private Integer level;

    /**
     * Item name
     */
    private String name;

    /**
     * Original stat lines (base item stats from database)
     */
    private List<Line> originalLines = new ArrayList<>();

    /**
     * Exotic stat lines (overmaxed or added stats)
     */
    private List<Line> exoticLines = new ArrayList<>();

    /**
     * Reliquat (remainder pool) - accumulates failed FM weight
     */
    private Double reliquat = 0.0;

    /**
     * Last reliquat change from most recent FM attempt
     */
    private Double lastReliquatModification = 0.0;

    /**
     * Get all stat lines (original + exotic)
     */
    public List<Line> getLines() {
        return Stream.concat(originalLines.stream(), exoticLines.stream())
            .collect(Collectors.toList());
    }

    /**
     * Find a line by its effect ID
     */
    public Line getLineByEffectId(Integer effectId) {
        return getLines().stream()
            .filter(line -> line.getEffectId().equals(effectId))
            .findFirst()
            .orElse(null);
    }

    /**
     * Calculate total item weight: sum of all line weights
     */
    public Double getWeight() {
        return getLines().stream()
            .mapToDouble(Line::getWeight)
            .sum();
    }

    /**
     * Remove exotic lines with zero value and no modification
     */
    public void cleanLines() {
        exoticLines.removeIf(line ->
            line.getValue() == 0 && line.getLastModification() == 0
        );
    }
}
```

---

## 3.6 JPA/Hibernate Entities for Database

### Entity: Description

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

### Entity: Effect

```java
package com.dofus.fm.entity;

import jakarta.persistence.*;
import lombok.Data;

@Entity
@Table(name = "effect")
@Data
public class Effect {

    @Id
    @Column(name = "id")
    private Integer id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "description_id")
    private Description description;

    @Column(name = "weight")
    private Double weight;
}
```

### Entity: ItemEntity (Database)

```java
package com.dofus.fm.entity;

import jakarta.persistence.*;
import lombok.Data;

@Entity
@Table(name = "item")
@Data
public class ItemEntity {

    @Id
    @Column(name = "id")
    private Integer id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "description_id")
    private Description description;

    @Column(name = "level")
    private Integer level;

    @Column(name = "icon_id")
    private Integer iconId;
}
```

### Entity: EffectLine

```java
package com.dofus.fm.entity;

import jakarta.persistence.*;
import lombok.Data;

@Entity
@Table(name = "effect_line")
@Data
public class EffectLine {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Integer id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "effect_id")
    private Effect effect;

    @Column(name = "min")
    private Integer min;

    @Column(name = "max")
    private Integer max;
}
```

### Entity: ItemEffectLine (Junction)

```java
package com.dofus.fm.entity;

import jakarta.persistence.*;
import lombok.Data;

@Entity
@Table(name = "item_effect_line")
@Data
public class ItemEffectLine {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; // Add surrogate key for JPA

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "item_id")
    private ItemEntity item;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "effect_line_id")
    private EffectLine effectLine;
}
```

---

## 3.7 Repository Layer (Spring Data JPA)

### ItemRepository

```java
package com.dofus.fm.repository;

import com.dofus.fm.entity.ItemEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.Optional;
import java.util.List;

@Repository
public interface ItemRepository extends JpaRepository<ItemEntity, Integer> {

    @Query("""
        SELECT i.id, i.level, d.descriptionText
        FROM ItemEntity i
        JOIN i.description d
        WHERE i.id = :itemId
        """)
    Optional<Object[]> findItemBasicInfo(Integer itemId);

    @Query("""
        SELECT e.id, el.min, el.max
        FROM ItemEffectLine iel
        JOIN iel.effectLine el
        JOIN el.effect e
        WHERE iel.item.id = :itemId
          AND e.id NOT IN (983, 984)
        """)
    List<Object[]> findItemEffectLines(Integer itemId);

    @Query("""
        SELECT e.id, el.min, e.weight, d.descriptionText
        FROM ItemEffectLine iel
        JOIN iel.effectLine el
        JOIN el.effect e
        JOIN e.description d
        WHERE iel.item.id = :itemId
        """)
    Optional<Object[]> findItemEffectData(Integer itemId);
}
```

### EffectRepository

```java
package com.dofus.fm.repository;

import com.dofus.fm.entity.Effect;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface EffectRepository extends JpaRepository<Effect, Integer> {
}
```

---

## 3.8 Factory/Service Layer for Domain Models

### ItemFactory

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

            Line line = new Line(effectId, min, max, 0, effectRepository);
            item.getOriginalLines().add(line);
        }

        return item;
    }
}
```

---

## 3.9 Database Initialization for Java

### Option 1: Keep Python Init Script
- Continue using `init/init.py` to populate database
- Java application reads from existing SQLite file
- **Pros**: Minimal migration effort
- **Cons**: Python dependency for database setup

### Option 2: Migrate to Java/Spring Boot
Create a CommandLineRunner or Flyway migration script:

```java
@Component
public class DatabaseInitializer implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        // Read JSON files
        ObjectMapper mapper = new ObjectMapper();

        // Load i18n_fr.json
        Map<String, String> descriptions = mapper.readValue(
            new File("init/raw_data/i18n_fr.json"),
            new TypeReference<>() {}
        );

        // Load effects.json
        List<EffectDto> effects = mapper.readValue(
            new File("init/raw_data/output/effects.json"),
            new TypeReference<>() {}
        );

        // Load items.json
        List<ItemDto> items = mapper.readValue(
            new File("init/raw_data/output/items.json"),
            new TypeReference<>() {}
        );

        // Populate database using JPA repositories
        // ... (similar logic to Python init.py)
    }
}
```

### Option 3: Use Flyway Migrations
Create SQL migration scripts in `src/main/resources/db/migration/`:

```sql
-- V1__create_schema.sql
CREATE TABLE description (
    id INTEGER PRIMARY KEY,
    description_text TEXT
);

CREATE TABLE effect (
    id INTEGER PRIMARY KEY,
    description_id INTEGER,
    weight REAL,
    FOREIGN KEY (description_id) REFERENCES description(id)
);

-- ... (rest of schema)
```

---

## 3.10 Key Migration Considerations

### 1. Entity vs Domain Model Separation
- **Entity**: JPA-mapped database tables (in `com.dofus.fm.entity`)
- **Domain Model**: Business objects (in `com.dofus.fm.domain`)
- Keep them separate for clean architecture

### 2. Lazy Loading
- Use `@ManyToOne(fetch = FetchType.LAZY)` to avoid N+1 queries
- Be careful with closed sessions (use `@Transactional` properly)

### 3. Database Connection Pooling
Python creates new connections for each operation (`sqlite3.connect()`), which is inefficient.

Java/Spring Boot: Use HikariCP (default in Spring Boot):

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:sqlite:database.sqlite
    driver-class-name: org.sqlite.JDBC
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
```

### 4. Integer Precision
- Python handles large integers automatically
- Java: Use `Integer` for IDs, `Double` for weights (avoid `float` precision issues)

### 5. Immutability Consideration
Consider making domain models immutable using:
- Lombok `@Value` annotation
- Builder pattern for construction
- **Benefit**: Thread safety for concurrent packet processing

---

**Next Section Preview:**
Section 4 will cover the complex FM calculation business logic, including weight calculations, reliquat tracking, and result classification (SC/SN/EC/EN).
