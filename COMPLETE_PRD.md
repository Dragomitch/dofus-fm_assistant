# Product Requirements Document (PRD)
# Python to Java 26 + Spring Boot 3.x Migration
## Dofus FM Assistant Application

---

## Section 1: Executive Summary & Application Overview

### 1.1 Document Purpose
This PRD provides a comprehensive migration guide for converting the Dofus FM Assistant from Python to Java 26 with Spring Boot 3.x (latest LTS). It details all components, business logic, and technical requirements needed to successfully reimplement the application in the target technology stack.

### 1.2 Application Overview

#### What is Dofus FM Assistant?
The Dofus FM Assistant is a **real-time network packet analyzer and helper tool** for the Forgemagie (FM) crafting system in the MMORPG game Dofus. It provides live assistance to players who are enhancing/crafting items by:

1. **Intercepting network packets** between the Dofus game client and server
2. **Parsing Dofus protocol messages** to extract item and rune information
3. **Calculating FM mechanics** (weight, reliquat/remainder pool, success/failure predictions)
4. **Displaying real-time information** in a GUI about the current FM session

#### Why Dofus FM System Exists
In Dofus, the Forgemagie (FM) system allows craftsmen to enhance items by applying runes that modify item statistics (strength, vitality, damage, etc.). This system is complex because:

- Each stat modification has a **weight** (difficulty factor)
- Failed attempts create a **reliquat** (remainder/pool) that affects future attempts
- There are 4 possible outcomes: SC (Success Clean), SN (Success Neutral), EC (Échec Clean), EN (Échec Neutral)
- Understanding the mechanics requires tracking weights, reliquat, and probability calculations

The FM Assistant helps players by automatically:
- Tracking current item stats vs maximum possible stats
- Calculating weight changes in real-time
- Monitoring the reliquat pool
- Displaying which rune is currently being applied

### 1.3 Current Python Application Architecture

#### High-Level Components
```
┌─────────────────────────────────────────────────────────┐
│                     main.py                             │
│              (Network Packet Sniffer)                   │
│                                                          │
│  Uses Scapy to capture packets from:                   │
│  Host: 213.248.126.61 (Dofus game server)              │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│              dofus_packet.py                            │
│           (Packet Parser & Decoder)                     │
│                                                          │
│  Parses binary Dofus protocol packets:                 │
│  - Packet ID 5516/5519: ExchangeObject (item/rune)     │
│  - Packet ID 6188: ExchangeCraftResult (FM result)     │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌──────────────┬──────────────┬──────────────────────────┐
│   item.py    │   rune.py    │      line.py             │
│ (Item Model) │ (Rune Model) │  (Stat Line Model)       │
└──────────────┴──────────────┴──────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│                 display.py                              │
│         (Tkinter GUI - Threading)                       │
│                                                          │
│  Shows: Item, Rune, Stat Lines, Reliquat               │
└─────────────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│              database.sqlite                            │
│                                                          │
│  Tables: item, effect, effect_line,                    │
│          item_effect_line, description                 │
└─────────────────────────────────────────────────────────┘
```

#### Core Technologies Used
- **Python 3.x**: Main programming language
- **Scapy**: Network packet capture and analysis
- **Tkinter**: GUI framework (Python's standard GUI library)
- **SQLite3**: Embedded database for game data
- **Threading**: Runs GUI in separate thread from packet sniffer

### 1.4 Key Business Flows

#### Flow 1: Application Startup
1. Initialize Display (Tkinter GUI in separate thread)
2. Connect to SQLite database
3. Start Scapy packet sniffer with filter: `host 213.248.126.61`
4. Wait for incoming packets

#### Flow 2: Item/Rune Detection
1. Packet arrives from Dofus server
2. Extract packet from raw TCP payload (custom Dofus protocol)
3. Check if packet ID is "interesting" (5516, 5519, or 6188)
4. Parse packet based on type:
   - **5516/5519**: ExchangeObjectMessage → Item or Rune
   - **6188**: ExchangeCraftResultWithObjectDescMessage → FM Result
5. Determine if object is rune (by checking GID against hardcoded list)
6. Create Item or Rune object, fetch data from database
7. Update GUI display

#### Flow 3: FM Execution & Calculation
1. Player applies rune to item in Dofus game
2. Server sends packet 6188 with result
3. Parse packet to extract:
   - craftResult (1=échec/failure, 2=success)
   - Updated item effects/stats
   - magicPoolStatus (reliquat status: 3 = reliquat decreased)
4. Calculate weight changes:
   - Theoretical weight = rune weight (+ for success, - for failure)
   - Real weight = sum of all stat line changes × their weights
   - Reliquat modification = -(real_weight - theoretical_weight)
5. Update reliquat pool
6. Classify result type (SC/SN/EC/EN)
7. Update GUI with color-coded changes (green=increase, red=decrease)

### 1.5 Critical Business Logic - The FM Weight System

#### What is "Weight" in FM?
Every stat in Dofus has a **weight** value that represents how difficult it is to add that stat to an item. For example:
- Vitality might have weight 0.25 (easy to add)
- AP (Action Points) might have weight 100 (very hard to add)

#### What is "Reliquat"?
The **reliquat** (French for "remainder") is a hidden pool/accumulator that:
- Increases when FM attempts fail
- Decreases when stats decrease
- Affects future success probabilities
- Must be tracked manually by the player (game doesn't show it)

#### The Weight Calculation Formula
```
Theoretical Weight Change = {
  +rune_weight if SUCCESS
  -rune_weight if FAILURE
}

Real Weight Change = Σ(stat_change × stat_weight) for all stat lines

Reliquat Modification = -(Real Weight - Theoretical Weight)

New Reliquat = Old Reliquat + Reliquat Modification
```

#### Result Classification
- **SC (Success Clean)**: Rune applied successfully, no stats decreased, reliquat unchanged
- **SN (Success Neutral)**: Rune applied successfully, BUT some stats decreased OR reliquat decreased
- **EC (Échec Clean)**: Rune failed, stats decreased OR reliquat decreased
- **EN (Échec Neutral)**: Rune failed, no stats changed, reliquat increased

### 1.6 Migration Objectives

#### Primary Goals
1. **Preserve exact business logic**: FM calculations must work identically
2. **Maintain real-time performance**: Network packet processing must be non-blocking
3. **Modernize architecture**: Use Spring Boot best practices (dependency injection, layered architecture)
4. **Improve maintainability**: Type safety, better error handling, logging
5. **Database evolution**: Keep SQLite but add JPA/Hibernate for ORM

#### Non-Functional Requirements
- **Performance**: Packet processing latency < 50ms
- **Reliability**: No packet drops during continuous 8+ hour FM sessions
- **Maintainability**: Clean separation of concerns (network, parsing, business logic, UI)
- **Testability**: Unit tests for all business logic, integration tests for packet parsing

#### Out of Scope
- Changing the FM calculation algorithms (must match exactly)
- Adding new features (this is a 1:1 migration)
- Supporting multiple simultaneous FM sessions
- Cloud deployment (remains desktop application)

---

**Next Sections Preview:**
- Section 2: Network Packet Parsing System (detailed protocol documentation)
- Section 3: Domain Models & Data Layer (entities, database schema)
- Section 4: Business Logic - FM Calculation System (algorithms in detail)
- Section 5: UI/Display Layer Migration (Tkinter → Java GUI framework)
- Section 6: Java/Spring Boot Architecture & Tech Stack
- Section 7: Migration Strategy & Implementation Plan
# Section 2: Network Packet Parsing System

## 2.1 Overview of Dofus Network Protocol

### What is Being Captured?
The application captures **TCP packets** between the Dofus game client and server at IP `213.248.126.61`. These packets contain the Dofus proprietary binary protocol, not HTTP or any standard protocol.

### Why Packet Sniffing?
Dofus doesn't provide an official API for FM mechanics. The only way to get real-time FM information is to:
1. Capture raw network traffic
2. Parse the proprietary Dofus protocol
3. Extract relevant messages about items, runes, and FM results

## 2.2 Current Python Implementation - Packet Capture Layer

### File: `main.py` - Packet Capture Logic

```python
# Uses Scapy library for packet sniffing
from scapy.all import *

filter = "host 213.248.126.61"  # Dofus game server

def handle(pkt):
    if pkt[IP].len > 40:  # Filter small packets
        try:
            pktdata = pkt[Raw].load  # Extract raw payload
            while True:
                pktdata, extracted = pop_pkt(pktdata)  # Extract one Dofus packet
                if extracted.isInteresting():
                    parsed_packet = extracted.parse()
                    # Handle different packet types...
                if len(pktdata) <= 2:
                    break
        except IndexError:
            pass

sniff(store=0, filter=filter, prn=handle)  # Start sniffing
```

### Key Functions in `main.py`

#### 1. **`get_pkt_id(pkt)` - Extract Packet ID**
```python
def get_pkt_id(pkt):
    oct0 = pkt[0]
    oct1 = (pkt[1] & 0b11111100)//4
    return oct1 + oct0*64
```

**What it does**: Extracts a 14-bit packet ID from the first 2 bytes
- First byte (oct0): Most significant 8 bits
- Second byte bits 2-7 (oct1): Least significant 6 bits
- Formula: `ID = (oct0 * 64) + oct1`

**Example**:
- Byte 0: `0x56` (86 decimal)
- Byte 1: `0x70` (112 decimal, binary: `01110000`)
- oct1 = `(112 & 0b11111100) / 4` = `112 / 4` = 28
- Packet ID = `86 * 64 + 28` = **5516** (ExchangeObjectMessage)

**Java Migration Note**: Use bitwise operations carefully; Java has unsigned issues. Use `& 0xFF` to treat bytes as unsigned.

---

#### 2. **`get_data_len_len(pkt)` - Get Length of Length Field**
```python
def get_data_len_len(pkt):
    return pkt[1] & 0b00000011
```

**What it does**: Extracts the last 2 bits of byte 1 to determine how many bytes are used to store the data length
- Result can be 0, 1, 2, or 3 bytes

**Example**:
- Byte 1: `0x72` (binary: `01110010`)
- Last 2 bits: `10` = 2
- This means the next 2 bytes contain the data length

---

#### 3. **`get_data_len(pkt, len_len)` - Extract Data Length**
```python
def get_data_len(pkt, len_len):
    return int.from_bytes(pkt[2:2+len_len], byteorder='big')
```

**What it does**: Reads `len_len` bytes starting at byte 2 to get the payload size
- Uses **big-endian** byte order

**Example**:
- len_len = 2
- Bytes 2-3: `0x00 0x1A` (26 decimal)
- Data length = 26 bytes

---

#### 4. **`get_data(pkt, len_len, data_len)` - Extract Payload**
```python
def get_data(pkt, len_len, data_len):
    return pkt[2+len_len:2+len_len+data_len]
```

**What it does**: Extracts the actual data payload

**Example**:
- len_len = 2
- data_len = 26
- Returns bytes from position 4 to position 30 (4 + 26)

---

#### 5. **`pop_pkt(pkt)` - Extract One Dofus Packet from Stream**
```python
def pop_pkt(pkt):
    id = get_pkt_id(pkt)
    data_len_len = get_data_len_len(pkt)
    data_len = get_data_len(pkt, data_len_len)
    data = get_data(pkt, data_len_len, data_len)
    extracted = DofusPacket(id, data_len, data)
    remaining = pkt[2+data_len_len+data_len:]
    return (remaining, extracted)
```

**What it does**: Pops one complete Dofus packet from the byte stream and returns the remaining bytes

**Important**: Multiple Dofus packets can be in one TCP packet, hence the `while True` loop in `handle()`

---

### Dofus Packet Structure (Binary Format)

```
Byte Position:  0        1           2..N        N+1..M
              ┌────────┬────────┬─────────────┬──────────┐
              │ ID (8) │ID(6)+LL│   Length    │   Data   │
              │  bits  │ (2bits)│  (0-3 bytes)│ (N bytes)│
              └────────┴────────┴─────────────┴──────────┘

Where:
- ID: 14-bit packet identifier (bits 0-13)
- LL: 2-bit length-length field (bits 14-15)
- Length: Variable length field (0-3 bytes) indicating data size
- Data: Actual payload (N bytes)
```

### Java Migration Strategy for Packet Capture

**Option 1: Pcap4j (Recommended)**
```java
// Pcap4j is a pure Java packet capture library
PcapHandle handle = new PcapNetworkInterface.Builder(nif)
    .build()
    .openLive(65536, PromiscuousMode.PROMISCUOUS, 10);

handle.setFilter("host 213.248.126.61", BpfCompileMode.OPTIMIZE);

PacketListener listener = packet -> {
    if (packet.contains(TcpPacket.class)) {
        byte[] payload = packet.get(TcpPacket.class).getPayload().getRawData();
        processDofusPackets(payload);
    }
};

handle.loop(-1, listener);
```

**Option 2: JNetPcap**
- Similar to Pcap4j but with native bindings
- Requires libpcap installation

**Option 3: Spring Boot Service Architecture**
```java
@Service
public class PacketCaptureService {

    @Async
    public void startCapture() {
        // Packet capture runs in separate thread
        while (running) {
            Packet packet = captureNextPacket();
            dofusPacketProcessor.process(packet);
        }
    }
}
```

**Important Notes**:
- Java requires **administrator/root privileges** for packet capture (same as Python/Scapy)
- Consider using **Spring @Async** for non-blocking packet processing
- Use **ByteBuffer** for efficient byte manipulation

---

## 2.3 Dofus Packet Parser Implementation

### File: `dofus_packet.py` - Packet Parsing Logic

### 2.3.1 DofusPacket Class Structure

```python
class DofusPacket:
    def __init__(self, id, len, raw_data):
        self.id = id           # Packet ID (e.g., 5516, 6188)
        self.len = len         # Data length
        self.raw_data = raw_data  # Binary payload
```

### 2.3.2 Interesting Packet IDs

The application only processes 3 packet types:

| Packet ID | Packet Name | Description | When Sent |
|-----------|-------------|-------------|-----------|
| **5516** | ExchangeObjectAddedMessage | Item or rune added to FM table | Player places item/rune |
| **5519** | ExchangeObjectModifiedMessage | Item or rune modified | Player swaps item/rune |
| **6188** | ExchangeCraftResultWithObjectDescMessage | FM result after applying rune | Server responds to FM attempt |

```python
def isInteresting(self):
    if self.id in [5516, 5519, 6188]:
        return True
    else:
        return False
```

**Java Migration**:
```java
public enum DofusPacketType {
    EXCHANGE_OBJECT_ADDED(5516),
    EXCHANGE_OBJECT_MODIFIED(5519),
    EXCHANGE_CRAFT_RESULT(6188);

    private final int id;

    public static boolean isInteresting(int packetId) {
        return Arrays.stream(values())
            .anyMatch(type -> type.id == packetId);
    }
}
```

---

### 2.3.3 Variable-Length Integer Readers

Dofus uses **variable-length encoding** for integers to save bandwidth (similar to Protocol Buffers).

#### **VarShort Reader** (16-bit signed)

```python
def readVarShort(self, bytes_array):
    result = 0
    progress = 0
    current_byte = 0
    continuer = False
    while(progress < 16):
        current_byte = bytes_array[0]
        bytes_array = bytes_array[1:]
        continuer = (current_byte & 0b10000000) == 0b10000000
        if progress > 0:
            result = result + ((current_byte & 0b01111111) << progress)
        else:
            result = result + (current_byte & 0b01111111)
        progress += 7
        if not continuer:
            if(result > 32767):
                result = result - 65536  # Convert to signed
            return (bytes_array, result)
    raise ValueError("Too much data")
```

**How VarShort Works**:
1. Each byte has 7 data bits + 1 continuation bit (MSB)
2. If MSB = 1, continue reading next byte
3. If MSB = 0, this is the last byte
4. Combine 7-bit chunks with left shifts
5. If result > 32767, convert to negative (two's complement)

**Example**: Reading value **300**
- 300 in binary: `0000000100101100`
- Split into 7-bit chunks: `0000010` `0101100`
- Byte 1: `10101100` (continuation bit = 1, data = `0101100` = 44)
- Byte 2: `00000010` (continuation bit = 0, data = `0000010` = 2)
- Result: `44 + (2 << 7)` = `44 + 256` = **300**

**Java Migration**:
```java
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
            int currentByte = bytes[offset++] & 0xFF; // Treat as unsigned
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

---

#### **VarInt Reader** (32-bit unsigned)

```python
def readVarInt(self, bytes_array):
    result = 0
    progress = 0
    current_byte = 0
    continuer = False
    while(progress < 32):
        current_byte = bytes_array[0]
        bytes_array = bytes_array[1:]
        continuer = (current_byte & 0b10000000) == 0b10000000
        if progress > 0:
            result = result + ((current_byte & 0b01111111) << progress)
        else:
            result = result + (current_byte & 0b01111111)
        progress += 7
        if not continuer:
            return (bytes_array, result)
    raise ValueError("Too much data")
```

**Difference from VarShort**:
- Max 32 bits instead of 16
- No signed conversion (always unsigned)

**Java Migration**:
```java
public static ReadResult readVarInt(byte[] bytes) {
    long result = 0; // Use long to prevent overflow
    int progress = 0;
    int offset = 0;

    while (progress < 32) {
        int currentByte = bytes[offset++] & 0xFF;
        boolean continuer = (currentByte & 0x80) == 0x80;

        if (progress > 0) {
            result += ((currentByte & 0x7F) << progress);
        } else {
            result += (currentByte & 0x7F);
        }

        progress += 7;

        if (!continuer) {
            byte[] remaining = Arrays.copyOfRange(bytes, offset, bytes.length);
            return new ReadResult(remaining, (int) result);
        }
    }

    throw new IllegalArgumentException("VarInt too long");
}
```

---

#### **Fixed-Length Integer Reader**

```python
def readIntFromBytes(self, bytes_array, size):
    return (bytes_array[size:], int.from_bytes(bytes_array[0:size], byteorder='big'))
```

**Java Migration**:
```java
public static ReadResult readIntFromBytes(byte[] bytes, int size) {
    ByteBuffer buffer = ByteBuffer.wrap(bytes, 0, size);
    buffer.order(ByteOrder.BIG_ENDIAN);

    int value;
    switch (size) {
        case 1: value = buffer.get() & 0xFF; break;
        case 2: value = buffer.getShort() & 0xFFFF; break;
        case 4: value = buffer.getInt(); break;
        default: throw new IllegalArgumentException("Invalid size: " + size);
    }

    byte[] remaining = Arrays.copyOfRange(bytes, size, bytes.length);
    return new ReadResult(remaining, value);
}
```

---

## 2.4 Packet Parsing - ExchangeObjectMessage (ID 5516/5519)

### Purpose
This packet contains information about an item or rune being placed on the FM table.

### Packet Structure

```python
def parse_ExchangeObjectMessage(self, raw_data):
    remaining = raw_data[1:]  # Skip first byte (unknown/flags)
    remaining, position = self.readIntFromBytes(remaining, 1)  # Position in exchange
    remaining, objectGID = self.readVarShort(remaining)  # Object Generic ID
    remaining, numberOfEffects = self.readIntFromBytes(remaining, 2)  # Effect count

    effects = []
    for i in range(numberOfEffects):
        remaining, effectType = self.readIntFromBytes(remaining, 2)

        if effectType == 70:  # ObjectEffectInteger
            remaining, actionId = self.readVarShort(remaining)
            remaining, value = self.readVarShort(remaining)
            effects.append({
                'actionId': actionId,
                'value': value
            })
        elif effectType == 82:  # ObjectEffectMinMax
            remaining, actionId = self.readVarShort(remaining)
            remaining, mini = self.readVarShort(remaining)
            remaining, maxi = self.readVarShort(remaining)
            effects.append({
                'actionId': actionId,
                'min': mini,
                'max': maxi
            })

    remaining, objectUID = self.readVarInt(remaining)  # Unique instance ID
    remaining, quantity = self.readVarInt(remaining)   # Quantity

    return {
        'position': position,
        'objectGID': objectGID,
        'effects': effects,
        'objectUID': objectUID,
        'quantity': quantity
    }
```

### Field Breakdown

| Field | Type | Size | Description |
|-------|------|------|-------------|
| (skip) | byte | 1 | Unknown flag byte |
| position | uint8 | 1 | Position in FM exchange window |
| objectGID | VarShort | 1-3 | Generic object ID (item/rune type) |
| numberOfEffects | uint16 | 2 | How many stat effects follow |
| **Effects Array** | - | - | - |
| effectType | uint16 | 2 | 70 = Integer, 82 = MinMax |
| actionId | VarShort | 1-3 | Stat ID (e.g., 125 = Vitality) |
| value | VarShort | 1-3 | (Type 70) Current value |
| min | VarShort | 1-3 | (Type 82) Minimum value |
| max | VarShort | 1-3 | (Type 82) Maximum value |
| objectUID | VarInt | 1-5 | Unique item instance ID |
| quantity | VarInt | 1-5 | Stack quantity |

### Effect Types

**Type 70 (ObjectEffectInteger)**: Used for runes and exotic stats
- Single `value` field
- Example: A rune that gives +10 Strength

**Type 82 (ObjectEffectMinMax)**: Used for item base stats
- `min` and `max` fields define the possible range
- Example: An item with "20 to 35 Vitality" rolled at 28
  - min = 20, max = 35, current value tracked separately

### Distinguishing Items from Runes

```python
def isRune(self, objectGID):
    if objectGID in [1557, 7435, 7433, ..., 18722]:  # 80+ rune IDs
        return True
    else:
        return False
```

**Java Migration**:
```java
@Component
public class RuneIdentifier {

    private static final Set<Integer> RUNE_IDS = Set.of(
        1557, 7435, 7433, 7438, 1519, 1545, 1551, 1524, 1549, 1555,
        // ... (all 80+ rune IDs)
        18719, 18723, 18720, 18724, 18721, 18722
    );

    public boolean isRune(int objectGID) {
        return RUNE_IDS.contains(objectGID);
    }
}
```

**Better Approach**: Store rune IDs in database with a `type` column, query at runtime:
```sql
SELECT type FROM item WHERE id = ?
```

---

## 2.5 Packet Parsing - ExchangeCraftResultWithObjectDescMessage (ID 6188)

### Purpose
This packet is sent after a FM attempt, containing the result and updated item stats.

### Packet Structure

```python
def parse_ExchangeCraftResultWithObjectDescMessage(self, raw_data):
    remaining, craftResult = self.readIntFromBytes(raw_data, 1)  # 1=fail, 2=success
    remaining, objectGID = self.readVarShort(remaining)
    remaining, numberOfEffects = self.readIntFromBytes(remaining, 2)

    effects = []
    for i in range(numberOfEffects):
        remaining, effectType = self.readIntFromBytes(remaining, 2)
        if effectType == 70:  # ObjectEffectInteger
            remaining, actionId = self.readVarShort(remaining)
            remaining, value = self.readVarShort(remaining)
            effects.append({
                'actionId': actionId,
                'value': value
            })
        elif effectType == 82:  # ObjectEffectMinMax
            remaining, actionId = self.readVarShort(remaining)
            remaining, mini = self.readVarShort(remaining)
            remaining, maxi = self.readVarShort(remaining)
            effects.append({
                'actionId': actionId,
                'min': mini,
                'max': maxi
            })

    remaining, objectUID = self.readVarInt(remaining)
    remaining, quantity = self.readVarInt(remaining)
    remaining, magicPoolStatus = self.readIntFromBytes(remaining, 1)

    return {
        'craftResult': craftResult,
        'objectGID': objectGID,
        'effects': effects,
        'objectUID': objectUID,
        'quantity': quantity,
        'magicPoolStatus': magicPoolStatus
    }
```

### Field Breakdown

| Field | Type | Size | Description |
|-------|------|------|-------------|
| craftResult | uint8 | 1 | **1** = Échec (Failure), **2** = Success |
| objectGID | VarShort | 1-3 | Item GID |
| numberOfEffects | uint16 | 2 | Number of stat lines |
| effects[] | - | Variable | Same format as ExchangeObjectMessage |
| objectUID | VarInt | 1-5 | Item UID |
| quantity | VarInt | 1-5 | Quantity |
| magicPoolStatus | uint8 | 1 | **3** = Reliquat decreased, other values unknown |

### Magic Pool Status Values

The `magicPoolStatus` field indicates what happened to the reliquat:

| Value | Meaning |
|-------|---------|
| **3** | Reliquat decreased (negative modification) |
| Other | Unknown (needs reverse engineering) |

**Critical**: The value `3` is used to detect "neutral" outcomes (SN/EN classification)

---

## 2.6 Java Architecture for Packet Parsing

### Recommended Package Structure

```
com.dofus.fm.network
├── capture
│   ├── PacketCaptureService.java      // Pcap4j integration
│   └── PacketListener.java             // Async packet handler
├── protocol
│   ├── DofusPacket.java                // Base packet class
│   ├── DofusPacketReader.java          // Binary reader utilities
│   ├── VarIntReader.java               // Variable-length integer decoder
│   ├── parsers
│   │   ├── PacketParser.java           // Interface
│   │   ├── ExchangeObjectParser.java   // Parses 5516/5519
│   │   └── CraftResultParser.java      // Parses 6188
│   └── types
│       ├── ExchangeObjectMessage.java  // DTO
│       ├── CraftResultMessage.java     // DTO
│       └── Effect.java                 // Effect DTO
└── RuneIdentifier.java                 // Rune detection utility
```

### Example Service Class

```java
@Service
@Slf4j
public class DofusPacketParserService {

    private final Map<Integer, PacketParser<?>> parsers;

    @Autowired
    public DofusPacketParserService(
        ExchangeObjectParser exchangeObjectParser,
        CraftResultParser craftResultParser
    ) {
        this.parsers = Map.of(
            5516, exchangeObjectParser,
            5519, exchangeObjectParser,
            6188, craftResultParser
        );
    }

    public Optional<ParsedPacket> parse(byte[] rawPacket) {
        try {
            int packetId = extractPacketId(rawPacket);

            if (!parsers.containsKey(packetId)) {
                return Optional.empty(); // Not interesting
            }

            byte[] payload = extractPayload(rawPacket);
            PacketParser<?> parser = parsers.get(packetId);

            return Optional.of(parser.parse(payload));

        } catch (Exception e) {
            log.error("Failed to parse packet", e);
            return Optional.empty();
        }
    }

    private int extractPacketId(byte[] packet) {
        int oct0 = packet[0] & 0xFF;
        int oct1 = (packet[1] & 0xFC) / 4;
        return oct1 + oct0 * 64;
    }

    private byte[] extractPayload(byte[] packet) {
        int lenLen = packet[1] & 0x03;
        // ... extract data length and return payload
    }
}
```

---

## 2.7 Key Migration Considerations

### 1. Byte Handling Differences
- **Python**: Bytes are automatically unsigned (0-255)
- **Java**: `byte` is signed (-128 to 127)
- **Solution**: Always mask with `& 0xFF` when treating bytes as unsigned

### 2. ByteBuffer vs Manual Array Slicing
- Python uses convenient slicing: `bytes_array[2:5]`
- Java options:
  - `Arrays.copyOfRange()` (creates copies, slower)
  - `ByteBuffer.wrap()` (efficient, no copying)
  - Maintain offset index (most efficient for streaming)

### 3. Threading and Concurrency
- Python uses basic threading for GUI
- Java/Spring Boot: Use `@Async`, `CompletableFuture`, or reactive streams
- Packet capture should run in separate thread pool

### 4. Error Handling
- Python uses exceptions casually (`except IndexError: pass`)
- Java: Proper exception handling with typed exceptions
- Log all parsing errors for debugging

### 5. Performance Considerations
- Packet parsing must be **extremely fast** (< 1ms per packet)
- Avoid creating excessive objects (use object pooling if needed)
- Use primitive types where possible
- Consider using `byte[]` pools to reduce GC pressure

---

**Next Section Preview:**
Section 3 will cover the domain models (Item, Rune, Line) and database schema migration to JPA/Hibernate.
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
# Section 5: UI/Display Layer Migration

## 5.1 Current Python UI - Tkinter Implementation

### File: `display.py`

The current application uses **Tkinter**, Python's standard GUI library, running in a separate thread.

### Architecture Overview

```python
class Display(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
        self.start()  # Start GUI thread

    def run(self):
        # Create Tkinter window and widgets
        self.root = Tk()
        self.root.title("FM Helper")
        # ... create widgets
        self.root.mainloop()  # Run event loop
```

**Key Points**:
- GUI runs in **separate thread** from packet sniffer
- Uses **observer pattern**: Item/Rune objects call `listener.updateItem()` to refresh UI
- **Synchronous updates**: Main thread calls GUI update methods

---

## 5.2 UI Layout and Components

### Current UI Structure

```
┌────────────────────────────────────────────────────────┐
│                   FM Helper                            │
├────────────────────────────────────────────────────────┤
│ Item :  [Gelano] (niveau : 199)                       │
│ Rune :  [Ra Vi] | +10 Vitalité (poids : 2.5)          │
├────────────────────────────────────────────────────────┤
│ Min  Max  Effet              Modif  Poids             │
│ 30   40   +35 Vitalité       +5     8.75/10.0         │
│ 20   30   +25 Force          +0     25.0/30.0         │
│ 10   20   +15 Agilité        -2     13.0/20.0  ◄ RED  │
│ 1    1    +1 PA              +0     100.0/100.0       │
├────────────────────────────────────────────────────────┤
│ Reliquat : 12.5 (+2.0)                                │
└────────────────────────────────────────────────────────┘
```

### UI Components Breakdown

#### 1. **Header Section**
- **Item Name & Level**: `[Gelano] (niveau : 199)`
- **Rune Info**: `[Ra Vi] | +10 Vitalité (poids : 2.5)`

#### 2. **Stat Lines Table**
Columns:
- **Min**: Minimum possible value for stat
- **Max**: Maximum possible value for stat
- **Effet**: Human-readable stat description with current value
- **Modif**: Change from last FM attempt (+5, -2, etc.)
- **Poids**: Current weight / Max weight (e.g., `8.75/10.0`)

**Color Coding**:
- **Green**: Stat increased (`last_modification > 0`)
- **Red**: Stat decreased (`last_modification < 0`)
- **Default**: No change (`last_modification == 0`)

#### 3. **Reliquat Display**
- **Format**: `Reliquat : 12.5 (+2.0)`
  - `12.5` = current total reliquat
  - `(+2.0)` = change from last FM attempt
- Shows with `+` or `-` sign for clarity

---

## 5.3 Python Implementation Details

### Main Window Creation

```python
def run(self):
    self.root = Tk()
    self.root.title("FM Helper")
    self.root.protocol("WM_DELETE_WINDOW", self.close)

    # Main container
    self.mainframe = ttk.Frame(self.root, padding="3 3 12 12")
    self.mainframe.grid(column=0, row=0, sticky=(N, W, E, S))

    # Item label
    ttk.Label(self.mainframe, text="Item :").grid(column=1, row=1, sticky=E)
    self.item = ttk.Label(self.mainframe, text="no item")
    self.item.grid(column=2, row=1, sticky=W)

    # Rune label
    ttk.Label(self.mainframe, text="Rune :").grid(column=1, row=2, sticky=E)
    self.rune = ttk.Label(self.mainframe, text="no rune")
    self.rune.grid(column=2, row=2, sticky=W)

    # Stat lines frame (dynamic)
    self.lines = ttk.Frame(self.mainframe, padding="3 3 12 12")
    self.lines.grid(column=1, columnspan=2, row=3, sticky=E+W)

    # Reliquat label
    ttk.Label(self.mainframe, text="Reliquat :").grid(column=1, row=4, sticky=E)
    self.reliquat = ttk.Label(self.mainframe, text="aucun")
    self.reliquat.grid(column=2, row=4, sticky=W)

    self.root.mainloop()
```

---

### Update Methods

#### Update Rune Display

```python
def updateRune(self, rune):
    self.rune["text"] = (
        rune.getName() + ' | ' +
        rune.getDescription() + ' (poids : ' +
        str(rune.getWeight()) + ')'
    )
```

**Example Output**: `Ra Vi | +10 Vitalité (poids : 2.5)`

---

#### Update Item Display

```python
def updateItem(self, item):
    # Update item name and level
    self.item["text"] = item.getName() + ' (niveau : ' + str(item.getLevel()) + ')'

    # Update reliquat
    self.reliquat["text"] = self.myStr(item.getReliquat())
    if item.getLastReliquatModification() != 0:
        self.reliquat["text"] += ' (' + self.myStrWithSign(item.getLastReliquatModification()) + ')'

    # Clear existing stat line widgets (except header)
    for widget in self.lines.winfo_children():
        if widget.grid_info()["row"] != 1:
            widget.destroy()

    # Create new stat line widgets
    row = 2
    for line in item.getLines():
        ttk.Label(self.lines, text=str(line.getMin())).grid(column=1, row=row, sticky=W)
        ttk.Label(self.lines, text=str(line.getMax())).grid(column=2, row=row, sticky=W)
        ttk.Label(self.lines, text=line.getDescription()).grid(column=3, row=row, sticky=W)
        ttk.Label(self.lines, text=self.myStrWithSign(line.getLastModification())).grid(column=4, row=row, sticky=W)
        ttk.Label(self.lines, text=self.myStr(line.getWeight()) + "/" + self.myStr(line.getMaxWeight())).grid(column=5, row=row, sticky=W)

        # Apply color based on modification
        if line.getLastModification() > 0:
            for widget in self.lines.winfo_children():
                if widget.grid_info()["row"] == row:
                    widget['foreground'] = 'green'
        elif line.getLastModification() < 0:
            for widget in self.lines.winfo_children():
                if widget.grid_info()["row"] == row:
                    widget['foreground'] = 'red'

        row += 1
```

**Key Logic**:
1. Destroy all stat line widgets except header
2. Recreate all widgets from current item state
3. Apply color coding based on `last_modification`

---

### Utility Methods

```python
def myStr(self, number):
    """Format number: remove decimal if integer"""
    if number == int(number):
        return str(int(number))
    else:
        return "%.1f" % number

def myStrWithSign(self, number):
    """Format number with + or - sign"""
    if number > 0:
        return ('+' + str(number))
    else:
        return str(number)
```

---

## 5.4 Java GUI Options

### Option 1: JavaFX (Recommended)

**Pros**:
- Modern, native Java GUI framework
- Rich component library
- FXML for declarative UI
- Good CSS styling support
- Active development

**Cons**:
- Requires separate dependency (not in JDK by default since Java 11)
- Learning curve if unfamiliar

**Maven Dependency**:
```xml
<dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-controls</artifactId>
    <version>21.0.1</version>
</dependency>
<dependency>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-fxml</artifactId>
    <version>21.0.1</version>
</dependency>
```

---

### Option 2: Swing (Not Recommended)

**Pros**:
- Built into JDK
- Mature, stable

**Cons**:
- Outdated look and feel
- Less modern than JavaFX
- Verbose API

---

### Option 3: Web UI (Spring Boot + Thymeleaf/React)

**Pros**:
- Modern web tech stack
- Easy to style with CSS
- Can add remote access capability
- Reactive updates with WebSockets

**Cons**:
- More complex architecture
- Requires browser
- Overkill for local desktop app

---

### Recommended: JavaFX

For a 1:1 migration maintaining desktop app nature, **JavaFX** is the best choice.

---

## 5.5 JavaFX Implementation

### Project Structure

```
com.dofus.fm.ui
├── FMApplication.java          // JavaFX Application entry point
├── controller
│   └── MainController.java     // FXML controller
├── service
│   └── DisplayService.java     // Business logic → UI bridge
└── fxml
    └── main.fxml               // UI layout definition
```

---

### JavaFX Application Entry Point

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
        // Initialize Spring Boot context
        springContext = SpringApplication.run(DofusFMAssistantApplication.class);
    }

    @Override
    public void start(Stage primaryStage) throws Exception {
        // Load FXML with Spring integration
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

---

### FXML Layout Definition

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?import javafx.scene.control.*?>
<?import javafx.scene.layout.*?>
<?import javafx.geometry.Insets?>

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

---

### MainController (FXML Controller)

```java
package com.dofus.fm.ui.controller;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Line;
import com.dofus.fm.domain.Rune;
import javafx.application.Platform;
import javafx.collections.FXCollections;
import javafx.collections.ObservableList;
import javafx.fxml.FXML;
import javafx.scene.control.*;
import javafx.scene.paint.Color;
import org.springframework.stereotype.Component;

@Component
public class MainController {

    @FXML private Label itemLabel;
    @FXML private Label runeLabel;
    @FXML private Label reliquatLabel;

    @FXML private TableView<LineViewModel> statLinesTable;
    @FXML private TableColumn<LineViewModel, String> minColumn;
    @FXML private TableColumn<LineViewModel, String> maxColumn;
    @FXML private TableColumn<LineViewModel, String> effectColumn;
    @FXML private TableColumn<LineViewModel, String> modifColumn;
    @FXML private TableColumn<LineViewModel, String> weightColumn;

    private ObservableList<LineViewModel> statLines = FXCollections.observableArrayList();

    @FXML
    public void initialize() {
        // Bind table columns
        minColumn.setCellValueFactory(cell -> cell.getValue().minProperty());
        maxColumn.setCellValueFactory(cell -> cell.getValue().maxProperty());
        effectColumn.setCellValueFactory(cell -> cell.getValue().descriptionProperty());
        modifColumn.setCellValueFactory(cell -> cell.getValue().modificationProperty());
        weightColumn.setCellValueFactory(cell -> cell.getValue().weightProperty());

        // Apply row coloring
        statLinesTable.setRowFactory(tv -> new TableRow<LineViewModel>() {
            @Override
            protected void updateItem(LineViewModel item, boolean empty) {
                super.updateItem(item, empty);
                if (item == null || empty) {
                    setStyle("");
                } else if (item.getModificationValue() > 0) {
                    setStyle("-fx-text-fill: green;");
                } else if (item.getModificationValue() < 0) {
                    setStyle("-fx-text-fill: red;");
                } else {
                    setStyle("");
                }
            }
        });

        statLinesTable.setItems(statLines);
    }

    /**
     * Update rune display (called from DisplayService)
     */
    public void updateRune(Rune rune) {
        Platform.runLater(() -> {
            String runeText = String.format("%s | %s (poids : %d)",
                rune.getName(),
                rune.getDescription(),
                rune.getWeight()
            );
            runeLabel.setText(runeText);
        });
    }

    /**
     * Update item display (called from DisplayService)
     */
    public void updateItem(Item item) {
        Platform.runLater(() -> {
            // Update item label
            String itemText = String.format("%s (niveau : %d)",
                item.getName(),
                item.getLevel()
            );
            itemLabel.setText(itemText);

            // Update reliquat
            String reliquatText = formatNumber(item.getReliquat());
            if (item.getLastReliquatModification() != 0) {
                reliquatText += " (" + formatNumberWithSign(item.getLastReliquatModification()) + ")";
            }
            reliquatLabel.setText(reliquatText);

            // Update stat lines table
            statLines.clear();
            for (Line line : item.getLines()) {
                statLines.add(new LineViewModel(line));
            }
        });
    }

    private String formatNumber(double number) {
        if (number == (int) number) {
            return String.valueOf((int) number);
        } else {
            return String.format("%.1f", number);
        }
    }

    private String formatNumberWithSign(double number) {
        if (number > 0) {
            return "+" + formatNumber(number);
        } else {
            return formatNumber(number);
        }
    }
}
```

---

### LineViewModel (Table Row Data)

```java
package com.dofus.fm.ui.controller;

import com.dofus.fm.domain.Line;
import javafx.beans.property.SimpleStringProperty;
import javafx.beans.property.StringProperty;

/**
 * ViewModel for displaying a stat line in the table
 */
public class LineViewModel {

    private final Line line;

    private final StringProperty min;
    private final StringProperty max;
    private final StringProperty description;
    private final StringProperty modification;
    private final StringProperty weight;

    public LineViewModel(Line line) {
        this.line = line;

        this.min = new SimpleStringProperty(String.valueOf(line.getMin()));
        this.max = new SimpleStringProperty(String.valueOf(line.getMax()));
        this.description = new SimpleStringProperty(line.getDescription());

        String modifStr = line.getLastModification() > 0
            ? "+" + line.getLastModification()
            : String.valueOf(line.getLastModification());
        this.modification = new SimpleStringProperty(modifStr);

        String weightStr = formatNumber(line.getWeight()) + "/" + formatNumber(line.getMaxWeight());
        this.weight = new SimpleStringProperty(weightStr);
    }

    public StringProperty minProperty() { return min; }
    public StringProperty maxProperty() { return max; }
    public StringProperty descriptionProperty() { return description; }
    public StringProperty modificationProperty() { return modification; }
    public StringProperty weightProperty() { return weight; }

    public int getModificationValue() {
        return line.getLastModification();
    }

    private String formatNumber(double number) {
        if (number == (int) number) {
            return String.valueOf((int) number);
        } else {
            return String.format("%.1f", number);
        }
    }
}
```

---

### DisplayService (Bridge between Business Logic and UI)

```java
package com.dofus.fm.service;

import com.dofus.fm.domain.Item;
import com.dofus.fm.domain.Rune;
import com.dofus.fm.ui.controller.MainController;
import org.springframework.stereotype.Service;

/**
 * Service layer that bridges business logic and UI updates
 * Replaces Python's listener pattern
 */
@Service
public class DisplayService {

    private MainController mainController;

    /**
     * Register the UI controller (called after FXML loads)
     */
    public void setController(MainController controller) {
        this.mainController = controller;
    }

    /**
     * Update rune display
     */
    public void updateRune(Rune rune) {
        if (mainController != null) {
            mainController.updateRune(rune);
        }
    }

    /**
     * Update item display
     */
    public void updateItem(Item item) {
        if (mainController != null) {
            mainController.updateItem(item);
        }
    }
}
```

---

## 5.6 Threading Considerations

### Python Threading Model
```python
class Display(threading.Thread):
    # GUI runs in separate thread
```

### JavaFX Threading Model

**Critical**: JavaFX UI updates **must** occur on the JavaFX Application Thread!

```java
// WRONG - will crash!
public void updateItem(Item item) {
    itemLabel.setText(item.getName()); // Called from packet handler thread
}

// CORRECT - use Platform.runLater()
public void updateItem(Item item) {
    Platform.runLater(() -> {
        itemLabel.setText(item.getName());
    });
}
```

**Why**: JavaFX (like most GUI frameworks) is single-threaded. All UI updates must happen on the UI thread.

---

### Packet Handler → UI Update Flow

```
Packet Sniffer Thread
    ↓
DofusPacketParser
    ↓
FMSessionService.handleCraftResult()
    ↓
FMCalculationService.executeFM()
    ↓
DisplayService.updateItem()
    ↓
Platform.runLater(() -> { ... })  ← Switch to JavaFX thread
    ↓
MainController.updateItem()
    ↓
UI Updated
```

---

## 5.7 CSS Styling (Optional Enhancement)

JavaFX supports CSS for styling:

```css
/* styles.css */

.label-header {
    -fx-font-weight: bold;
    -fx-font-size: 14px;
}

.table-row-cell {
    -fx-font-family: monospace;
}

.positive-change {
    -fx-text-fill: green;
}

.negative-change {
    -fx-text-fill: red;
}
```

Load in FXML:
```xml
<VBox stylesheets="@../styles.css">
```

---

## 5.8 Alternative: Web UI with Spring Boot

### Architecture

```
Spring Boot Backend (REST API)
    ↓
WebSocket for real-time updates
    ↓
React/Vue.js Frontend
```

### Example REST Endpoint

```java
@RestController
@RequestMapping("/api/fm")
public class FMController {

    @Autowired
    private FMSessionService sessionService;

    @GetMapping("/current-item")
    public ItemDTO getCurrentItem() {
        return sessionService.getCurrentItem();
    }

    @GetMapping("/current-rune")
    public RuneDTO getCurrentRune() {
        return sessionService.getCurrentRune();
    }
}
```

### WebSocket for Real-Time Updates

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/fm-updates").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic");
    }
}

@Service
public class DisplayService {

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    public void updateItem(Item item) {
        messagingTemplate.convertAndSend("/topic/item-update", item);
    }
}
```

### Frontend (React Example)

```jsx
import SockJS from 'sockjs-client';
import Stomp from 'stompjs';

const socket = new SockJS('/fm-updates');
const stompClient = Stomp.over(socket);

stompClient.connect({}, () => {
    stompClient.subscribe('/topic/item-update', (message) => {
        const item = JSON.parse(message.body);
        updateItemDisplay(item);
    });
});
```

**Pros**:
- Modern, responsive UI
- Can access from any device on network
- Easy to add charts/visualizations

**Cons**:
- More complex than desktop app
- Requires browser
- Additional dependencies

---

## 5.9 Recommended Approach

### For 1:1 Migration: **JavaFX**

**Reasons**:
1. Closest to original Tkinter desktop app
2. Native performance
3. Self-contained executable
4. No browser required
5. Simple architecture

### Implementation Steps:

1. **Setup JavaFX dependencies** (Maven/Gradle)
2. **Create FXML layout** matching Tkinter UI
3. **Implement MainController** with update methods
4. **Create DisplayService** to bridge business logic → UI
5. **Use Platform.runLater()** for thread-safe UI updates
6. **Test with real packets** to ensure updates work correctly

---

## 5.10 Key Migration Considerations

### 1. Thread Safety
- Always use `Platform.runLater()` for UI updates from non-UI threads
- Consider using `@Async` for packet processing to keep UI responsive

### 2. Data Binding
- Use JavaFX `ObservableList` for TableView
- Use `Property` types for reactive updates

### 3. Color Coding
- Implement custom row factory for TableView
- Apply styles based on `last_modification` value

### 4. Number Formatting
- Replicate Python's `myStr()` and `myStrWithSign()` helpers
- Consider using `DecimalFormat` for consistent formatting

### 5. Window Lifecycle
- Handle close button properly (`setOnCloseRequest`)
- Stop packet capture when window closes
- Clean up resources

### 6. Error Handling
- Show error dialogs for packet parsing failures
- Log exceptions for debugging

---

## 5.11 UI Testing Strategy

### Manual Testing Checklist
- [ ] Item display updates when packet received
- [ ] Rune display updates when packet received
- [ ] Stat lines show correct values
- [ ] Positive changes show in green
- [ ] Negative changes show in red
- [ ] Reliquat updates correctly
- [ ] Reliquat modification shows in parentheses
- [ ] Exotic lines appear/disappear correctly
- [ ] Window can be closed without errors

### Integration Testing
```java
@SpringBootTest
class DisplayServiceTest {

    @Autowired
    private DisplayService displayService;

    @Test
    void testUpdateItem_UIRefreshes() {
        // Create test item
        Item item = createTestItem();

        // Update display
        displayService.updateItem(item);

        // Verify UI updated (using TestFX)
        // ...
    }
}
```

### UI Testing with TestFX
```java
@ExtendWith(ApplicationExtension.class)
class MainControllerTest {

    @Test
    void testItemLabelUpdates(FxRobot robot) {
        Item item = createTestItem();
        item.setName("Test Item");
        item.setLevel(100);

        robot.interact(() -> {
            controller.updateItem(item);
        });

        Label itemLabel = robot.lookup("#itemLabel").query();
        assertEquals("Test Item (niveau : 100)", itemLabel.getText());
    }
}
```

---

**Next Section Preview:**
Section 6 will cover the complete Java/Spring Boot architecture, technology stack selection, and project structure.
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
