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
