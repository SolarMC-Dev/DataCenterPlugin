# DataCenterPlugin

Centralized data persistence layer for Minecraft server backends, providing unified database access, caching, and cross-plugin data consistency.

---

## Why This Exists

In a multi-plugin Minecraft server, each plugin typically manages its own data, leading to:

- duplicated database connections
- inconsistent data models
- race conditions across plugins
- difficulty maintaining consistent updates across systems

DataCenterPlugin centralizes persistence into a single layer, enforcing consistent data access patterns and reducing duplication across plugins.

---

## Overview

DataCenterPlugin acts as the core data service for a Minecraft server backend.

It provides:

- unified database access
- schema management
- caching for performance
- consistent read/write patterns

Other plugins depend on it for reliable player data storage and retrieval.

---

## Core Responsibilities

- Manage database connections using HikariCP
- Register schemas and create tables automatically
- Provide unified data access via `DataKey` abstractions
- Maintain in-memory caching with TTL
- Ensure consistency across plugins accessing shared data
- Perform atomic upsert operations

---

## Architecture / Design

### Internal Components

- **DataCenter**  
  Central manager handling connections and schema registration

- **DataKey**  
  Defines a data domain (table + schema metadata)

- **Column**  
  Maps Java types to SQL types

- **Data / SingleData / MultipleData**  
  Represent stored entities and persistence logic

---

### Data Flow

```
Player Event
   ↓
Plugin Logic
   ↓
DataCenter.get(DataKey, UUID)
   ↓
Cache lookup (if enabled)
   ↓
Database query (on miss)
   ↓
Data object returned
   ↓
Modification → Upsert
   ↓
Cache update
```

---

### Cross-Plugin Interaction

Plugins access the system through a shared `DataCenter` instance.

Each plugin:

1. Registers its `DataKey`
2. Uses typed access to read/write data
3. Relies on DataCenter for consistency

This avoids tight coupling while maintaining shared state.

---

## Data & State Management

### Storage Architecture

- **Primary storage:** MariaDB
- **Connection pooling:** HikariCP
- **Cache:** In-memory, per-module TTL-based
- **Configuration:** YAML-based

---

### Data Models

- **SingleData**  
  One value per player (e.g., credits)

- **MultipleData**  
  Structured multi-column data (e.g., stats, rewards)

---

### Consistency Model

- Writes use an upsert strategy (UPDATE → fallback INSERT)
- Cache is invalidated immediately on write
- Read-after-write consistency is guaranteed within a single server instance
- No distributed locking; consistency relies on controlled access through DataCenter

---

## Key Features

- Modular data registration via `DataKey`
- Automatic table creation
- TTL-based caching for frequently accessed data
- Type-safe schema definitions
- Built-in support for leaderboard queries

---

## Interesting Engineering Decisions

### Upsert-First Persistence

Instead of separating INSERT and UPDATE:

- Attempt UPDATE
- If no rows affected → INSERT

This ensures:

- simpler calling code
- correct handling of new players
- idempotent operations

---

### CSV-Based Storage (Rewards)

Some modules store multiple values in a single column using CSV.

Tradeoff:

- simpler queries
- less normalized schema

Chosen due to limited relational complexity needs.

---

### High-Precision Currency

Credits stored as:

```
NUMERIC(15,3)
```

Avoids floating-point rounding issues common in game economies.

---

## Design Tradeoffs

- Simplicity over full normalization (e.g., CSV storage)
- Local caching instead of distributed caching (single-server assumption)
- Explicit SQL instead of ORM for control and predictability
- Auto-commit transactions instead of manual transaction control

---

## Challenges & Solutions

### 1. Cross-Plugin Data Access

**Problem:**  
Multiple plugins required shared access to player data without tight coupling.

**Solution:**  
Introduced `DataKey` registry pattern:

- plugins declare schemas
- receive typed access objects
- remain independent but consistent

---

### 2. Performance Under Load

**Problem:**  
Frequent reads (e.g., credits during PvP) caused excessive DB load.

**Solution:**  
Added module-specific TTL caching:

- reduced redundant queries
- maintained acceptable staleness

---

### 3. Schema Evolution

**Problem:**  
Schema changes risk breaking existing deployments.

**Solution:**

- used `CREATE TABLE IF NOT EXISTS`
- avoided destructive migrations
- allowed incremental schema updates

---

## Example Flow: Player Earns Credits

1. Plugin requests credits:
   ```
   Credits credits = dataCenter.get(CreditsKey.INSTANCE, playerUUID);
   ```

2. Cache is checked

3. On miss → database query executed

4. Plugin updates value:
   ```
   credits.add(100.0);
   ```

5. Upsert operation persists data

6. Cache updated

---

## Limitations

- Designed for single-server architecture (no multi-server sync)
- No distributed cache coherence
- No explicit transaction boundaries beyond single queries
- Schema migrations are manual
- No concurrency control beyond server-thread model

---

## Usage

This is a library plugin.

Example:

```
DataCenter dataCenter = SolarDataCenter.ins.getDataCenter();
Credits credits = dataCenter.get(CreditsKey.INSTANCE, playerUuid);
```

---

## Tech Stack

- Java 8
- Spigot API (1.8.8)
- MariaDB
- HikariCP
- YAML configuration
- Maven

---

## Notes

Used in a live multiplayer server environment with frequent player-driven data updates.

The system prioritizes:

- consistency
- simplicity
- performance

over feature completeness.

---

## TODO / Improvements

- Persist schema metadata and detect missing columns on startup
- Safely add new columns (`ALTER TABLE ADD COLUMN`) without breaking existing data
- Replace all raw SQL with prepared statements
- Centralize query construction to avoid unsafe dynamic SQL
- Improve cache handling (configurable TTL, better invalidation)
- Add basic transaction support for multi-step operations
- Improve error handling and logging  