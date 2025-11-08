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
