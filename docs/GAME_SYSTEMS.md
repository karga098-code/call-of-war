# Call of War - Complete Game Systems Architecture

## TABLE OF CONTENTS

1. [Game Engine Architecture](#game-engine-architecture)
2. [Map & Province System](#map--province-system)
3. [Military System](#military-system)
4. [Combat System](#combat-system)
5. [Logistics & Supply](#logistics--supply)
6. [Diplomacy & Coalition](#diplomacy--coalition)
7. [Economy & Market](#economy--market)
8. [Intelligence & Fog of War](#intelligence--fog-of-war)
9. [Research & Technology](#research--technology)
10. [Database Schema](#database-schema)
11. [API Reference](#api-reference)
12. [WebSocket Events](#websocket-events)
13. [Scaling & Performance](#scaling--performance)

---

## GAME ENGINE ARCHITECTURE

### Core Game Loop (100Hz Tick Rate)

```
[TICK CYCLE - 10ms per tick at 100Hz]
├─ Process Movements (calculate army positions, apply road bonuses)
├─ Process Combat (find collisions, resolve battles, apply casualties)
├─ Process Production (building queues, unit construction)
├─ Process Resources (income generation, upkeep costs)
├─ Process Logistics (supply chain, transport)
├─ Process Diplomacy (treaty expiration, embargo checks)
├─ Process Market (order matching, transactions)
└─ Persist State (every 100 ticks = 1 second)
```

### Real-Time Game Speed

- **Game Speed Multiplier:** 4x real-time
- **1 Real Hour = 4 In-Game Hours**
- **1 Real Day = 4 In-Game Days**
- **Resource Production:** Scales with 4x multiplier

### Tick Resolution

```typescript
const GAME_TICK_RATE = 100; // 100 ticks per second
const TICK_DURATION = 10; // milliseconds
const GAME_SPEED_MULTIPLIER = 4;

// Movement calculation
const distance_per_tick = baseSpeed * TICK_DURATION * GAME_SPEED_MULTIPLIER;

// Resource production
const production_per_tick = base_production / (100 * 60); // per second
```

---

## MAP & PROVINCE SYSTEM

### World Map Grid

- **Grid Size:** 100 x 100 = 10,000 total provinces
- **Total Cities:** 2,500 cities distributed across provinces
- **Player Starting Countries:** 8-500 players (randomly assigned)

### Province System

**Each Province Contains:**
- One city OR multiple non-critical logistics hubs
- Single terrain type (no mixed terrain)
- Resource nodes (food, oil, steel, etc.)
- Population
- Development level (1-10)

**Terrain Types (No Climate Penalties):**
```
- PLAIN: Base movement speed (1.0x)
- FOREST: Base movement speed (1.0x)
- MOUNTAIN: Base movement speed (1.0x)
- URBAN: Base movement speed (1.0x) - bonus to production
- COASTAL: Sea unit embarkation/disembarkation
```

### City Distribution Per Country

**Starting Territory:**
- 5 Critical Industrial Cities (military production hubs)
- 7 Resource Production Cities
- ~20-30 Logistics Cities
- Total: ~35-40 starting cities per country

### Strategic Movement System

```
Movement = Base Speed (1.0) + Road Bonus (0.5) + Airport Bonus (0.75)
- Own roads: +50% speed
- Allied roads: +50% speed (requires alliance)
- No vision bonus from roads (only movement)
- Roads cost to build/maintain
```

---

## MILITARY SYSTEM

### Unit Types (60+ Total Units)

#### Infantry (17 types)
```
Rifle Infantry, Militia, Motorized Infantry, Mechanized Infantry,
Shock Infantry, Special Forces, Paratroopers, Engineers, Marines,
Mountain Troops, Jungle Troops, Recon Teams, Guerrilla Units,
Commandos, Anti-Tank Infantry, Anti-Air Infantry,
Electronic Warfare Infantry
```

#### Armor/Tanks (12 types)
```
Light Tank, Medium Tank, Heavy Tank, Super Heavy Tank,
Tank Destroyer, Assault Tank, Fire Support Tank, Stealth Tank,
Drone Carrier Tank, Urban Warfare Tank, Anti-Air Tank,
Electronic Warfare Tank
```

#### Artillery (5 types)
```
Howitzer, Rocket Artillery, Heavy Mortar,
Mobile Artillery System, Anti-Drone Artillery
```

#### Air Units (15 types)
```
Fighter Jet, Interceptor, Stealth Fighter, Multi-Role Fighter,
CAS Aircraft, Tactical Bomber, Strategic Bomber, SEAD Aircraft,
Recon Aircraft, UAV Drone Swarm, Electronic Warfare Aircraft,
AWACS Aircraft, Transport Aircraft, Air Refueler,
Special Operations Aircraft
```

#### Naval Units (14 types)
```
Destroyer, Cruiser, Battleship, Aircraft Carrier,
Attack Submarine, Strategic Submarine, Torpedo Boat,
Amphibious Assault Ship, Drone Carrier Ship,
Electronic Warfare Ship, Logistics Tanker, Repair Ship,
Radar Ship, Coastal Defense Ship
```

#### Secret/Advanced Units (11 types)
```
Ghost Infantry, Nanotech Armored Infantry,
Reactive Armor Modules, Active Protection Systems,
Stealth Aircraft, ECM Warfare Unit, Drone Swarm Command,
Quantum Communication Unit, Energy Shield Experimental,
Shadow Submarine, Siege Heavy Tank

Requirements:
- Advanced research labs (tier 8+)
- Premium currency (gold) to unlock
- Counter systems available (radar, scouts, EW)
```

### Building System

#### Critical Industrial Cities (Can Build All)
- Infantry Barracks
- Armor Factory
- Artillery Facility
- Air Base
- Naval Base
- Research Lab
- Advanced Research Lab

#### Resource Production Cities (Limited)
- Farm (food production only)
- Oil Refinery
- Steel Mill
- Electronics Factory
- Trade Post

#### Logistics Cities (Support Only)
- Supply Depot
- Logistics Hub
- Transport Center
- Repair Facility
- Radar Station
- Communication Center

### Building Levels

```
Military Buildings:    Max Level 6
Logistics Buildings:   Max Level 4
Resource Buildings:    Max Level 4

Production Scaling:
Level 1: 1.0x base production
Level 2: 1.5x base production
Level 3: 2.0x base production
Level 4: 2.5x base production
Level 5: 3.0x base production
Level 6: 3.5x base production
```

### Unit Production

**Requirements:**
1. Building exists in city
2. Building must be correct level
3. Unit research must be completed
4. Sufficient resources
5. Sufficient manpower (recruitment)

**Production Formula:**
```
production_time = base_time / (building_level * research_tier * production_speed)
```

---

## COMBAT SYSTEM

### Combat Mechanics

**Combat Resolution:**
```
Duration: Base 5 minutes (300 seconds)
Tick Interval: 1 second (100 ticks)

Damage Per Tick = Unit Damage * Research Multiplier * Armor Reduction
Casualties = Damage / Unit Health

Ending Conditions:
- One side eliminated (0 units)
- One side routed (morale < 20%)
- 5 minute timer expiration (attacker retreats)
```

### Combat Factors

```
Offensive Modifiers:
+ Unit Type Advantage (hard/soft counters)
+ Research Tier Bonus (+10% per tier)
+ Morale Bonus (+5% per 10 morale points)
+ Experience Bonus (+2% per 100 experience)
+ Weather Terrain Bonus (T50-specific, none here)
- Stealth Detection Penalty (-50% if undetected)

Defensive Modifiers:
+ Unit Armor Rating
+ Fortification Bonus (if in home province)
+ Support Units (artillery, air support)
+ Entrenchment (after 5 rounds of combat)
- Surprise Attack Penalty (-30% defense first round)
```

### Unit Strength

```
Base Strength = (Unit Count * Individual HP) * (Attack Power * Defense Rating)

Damage Calculation:
Damage = (Attacker Strength * RNG 0.8-1.2) - (Defender Armor * RNG 0.9-1.1)
Casualties = Damage / Individual HP
```

### Morale & Routing

```
Morale Start: 100%
Morale Loss Per Round: -5% to -10% based on casualties
Route Threshold: 20%
Morale Recovery: +2% per minute out of combat
```

---

## LOGISTICS & SUPPLY

### Supply Chain System

**Supply Requirements:**
- Infantry: 0.5 supply/unit/hour
- Armor: 1.2 supply/unit/hour
- Artillery: 0.8 supply/unit/hour
- Air Units: 2.0 supply/unit/hour
- Naval Units: 3.0 supply/unit/hour

**Supply Calculation:**
```
Army Supply = Total Unit Count * Supply Per Unit
Available Supply = Resources in Province + Logistics Network Capacity

Supply Status:
100-75%: No penalty
75-50%: -10% combat effectiveness
50-25%: -25% combat effectiveness, morale -5%/hour
25-0%: -50% combat effectiveness, morale -10%/hour, desertion -1%/hour
```

### Transport & Distribution

**Logistics Cities provide:**
- Supply Depot: +500 supply capacity per level
- Logistics Hub: +1000 supply capacity, +50% distribution speed
- Transport Center: +300 capacity, enables rapid deployment

**Supply Distribution:**
- Base Range: 5 provinces from logistics city
- Range Bonus: +2 provinces per logistics level
- Distribution Speed: 100 units/hour base, affected by roads

### Resource Transport

**Naval Transport System:**

```
1. Embarkation (1 hour vulnerability window)
   - Army must be in coastal province
   - Conversion to transport ships
   - Attack/Defense = 0 during embarkation
   - Can be intercepted by naval units

2. At Sea
   - Speed: 2x normal movement
   - Naval radar required to detect units in transport
   - Without radar: enemy cannot see contents
   - Submarine threat: -5% cargo safety per enemy sub nearby

3. Disembarkation (1 hour vulnerability window)
   - Requires coastal province
   - Cannot move during unloading
   - Exposed to attack
   - Can be interrupted (cargo lost)
```

---

## DIPLOMACY & COALITION

### Coalition System

**Formation:**
- Leader country creates coalition
- Invites other countries to join
- Max 20 members per coalition
- Max 50 total coalitions

**Coalition Benefits:**
- Shared vision (optional intel sharing)
- Military passage rights
- Coordinated diplomatic stance
- Shared research (optional)
- Joint trade agreements

**Coalition Victory:**
- Condition: All enemy nations eliminated OR surrender
- Remaining coalition members split victory bonuses
- Territory distributed based on damage/casualties

### Peace Treaty Types

#### 1. Ceasefire
```
Duration: Immediate stop to combat
Effects:
- No combat allowed (24 hour enforcement)
- Mutual embargo (no trade)
- Military passage blocked
- Intelligence sharing blocked
```

#### 2. Peace Treaty
```
Duration: 7 days default
Effects:
- No combat allowed
- Normal trade allowed
- Military passage allowed (if allied)
- Intelligence sharing optional
```

#### 3. Reparations Peace
```
Duration: 30 days
Effects:
- Losing side pays resources daily
- Amount: 10% of daily income
- Transfers daily to victor
- Can be terminated early with gold payment
```

### Territory Rules

**Coalition War Territory:**
- Conquered cities return to original owner after war
- Temporary occupation only (max 3 months)
- Exception: Core territories remain permanently

**Non-Coalition War Territory:**
- Conquered territories remain permanently
- Ownership transfers to conquering nation
- Original owner cannot reclaim

### Diplomatic Actions

```
Offer Alliance
Declare War
Request Peace
Propose Trade Deal
Apply Embargo
Grant Military Passage
Share Intelligence
Request Technology
Propose Tribute/Reparations
```

### Embassy System

```
Each country can build 1 embassy per foreign city
Embassies provide:
- Diplomatic information
- Trade preferential rates
- Espionage/sabotage opportunities
- Can be destroyed by enemies

Position: Placed in enemy city
Detection: If within enemy intelligence radar range
Leak Information: Randomly leak intel if detected
```

---

## ECONOMY & MARKET

### Resource Types

```
FOOD
- Production: Farms
- Usage: Army supply
- Market Value: Base 10 gold/unit

OIL
- Production: Oil Refineries
- Usage: Air/Naval unit fuel
- Market Value: Base 15 gold/unit

STEEL
- Production: Steel Mills
- Usage: Unit construction (armor)
- Market Value: Base 12 gold/unit

ELECTRONICS
- Production: Electronics Factory
- Usage: Advanced unit construction, research
- Market Value: Base 25 gold/unit

MONEY
- Production: Trade Posts, taxation
- Usage: Building, research, gold conversion
- Market Value: Base 1 gold (equivalent)
```

### Global Market System

**Market Orders:**
```
ORDER TYPES:
- Buy Order: I need 1000 food at max 15 gold/unit
- Sell Order: I have 5000 oil for sale at 18 gold/unit

ORDER MATCHING:
- Automatic matching when buy/sell overlap
- Price: Whichever order was placed first (FIFO)
- Partial fills allowed
- Orders expire after 7 days

TAX:
- 5% tax per transaction (split between seller and buyer)
- Transaction fee: 2% market commission
```

**Player Armies for Sale:**
```
Army Market Listing:
- List entire army for sale
- Price: Based on unit composition and levels
- Buyer must have resources + manpower
- Sale includes all units and experience

Formula:
Price = SUM(unit_count * base_unit_cost * (1 + research_tier * 0.1) * morale_modifier)
```

### Trade Embargo System

```
EMBARGO EFFECTS:
- Blocked countries: Cannot trade with each other
- Neutral markets: Can still use global market
- Duration: Until peace treaty OR player cancels (with diplomacy cost)
- Cost to Impose: 5% of annual income

DETECTION:
- Embargo list public (all players see embargoes)
- Can be used as diplomacy leverage
- Breaking embargo: Automatic -5 diplomacy level
```

### Currency & Pricing

```
Base Currency: Gold
Exchange Rate: 1 Gold = Base value of resource

Dynamic Pricing:
- Supply & Demand
- Daily price adjustment: ±5% max
- Floor price: 50% of base
- Ceiling price: 150% of base

Price Formula:
New Price = Old Price * (1 + (Demand - Supply) / Total Supply)
```

---

## INTELLIGENCE & FOG OF WAR

### Fog of War System

**Base Rules:**
```
- Players only see their own territory
- Enemy territories are "fog of war" (hidden)
- Allied territories visible if intel shared
- No unit vision from buildings/scouts by default
```

### Multi-Layer Vision System

**Layer 1: Army Presence Detection**
```
Range: Radar coverage area
Shows: "Enemy units detected in province X"
Reveals: General unit count, no composition
Cost: Air radar station (level 1+)
```

**Layer 2: Unit Composition**
```
Range: Radar coverage area + scouting range
Shows: Unit types and quantities
Reveals: Army strength, morale estimate
Cost: Advanced radar or scouts
```

### Radar Systems

```
AIR RADAR
- Range: 10 provinces from city
- Detection: Aircraft, air transports
- Cooldown: Real-time
- Coverage: Cone pattern

GROUND RADAR
- Range: 15 provinces from city
- Detection: Land armies, vehicles
- Cooldown: Real-time
- Coverage: Circle pattern

NAVAL RADAR
- Range: 20 provinces at sea
- Detection: Naval units, transports
- Cooldown: Real-time
- Coverage: Sea grid pattern

ELECTRONIC WARFARE RADAR
- Range: 8 provinces
- Detection: Hidden units, stealth
- Specialization: Anti-stealth focus
- Allows detection of Ghost Infantry, Stealth Aircraft, etc.
```

### Scout Detection System

**Scout Mechanics:**
```
Scout Level >= Stealth Level → DETECTED
Scout Level < Stealth Level → NOT DETECTED

Scout Unit Types:
- Recon Teams (base 1x)
- Special Forces (1.5x detection)
- Electronic Warfare Units (2x stealth detection)
- AWACS Aircraft (3x air stealth)

Stealth Unit Types:
- Ghost Infantry (level 3 stealth)
- Stealth Fighter (level 2 stealth)
- Stealth Tank (level 2 stealth)
- Shadow Submarine (level 2 stealth)
```

### Intelligence Sharing

**Allied Intelligence Network:**
```
Share Type: Partial
- See allied army positions
- See allied building locations
- See allied research progress

Share Type: Full
- Same as Partial PLUS
- See enemy army positions within ally territory
- See discovered enemy composition
- Radar coverage sharing

Share Type: Complete
- Entire map knowledge exchange
- Real-time position updates
- All research data
- All resource levels
```

### Message Interception

**Encrypted Messages:**
```
Private message system between players
Messages encrypted by default
Interception: Possible if within enemy EW radar range

EW Radar Interception:
- 10% chance to intercept message
- 50% chance to read content
- Automatic log in enemy intelligence
- Diplomatic incident if discovered
```

---

## RESEARCH & TECHNOLOGY

### Research System

**Requirements:**
1. Research Lab (level 1+) in critical city
2. Unit must not exist (tier 1)
3. Previous tier must be completed
4. Sufficient resources allocated

**Tier Structure:**
```
Tier 1-7: Normal progression (free)
Tier 8-9: Premium progression (gold required)
Tier 10: Ultimate tier (massive gold cost)

Research Time Per Tier:
Tier 1: 1 hour real-time (4 hours game-time)
Tier 2: 2 hours real-time (8 hours game-time)
Tier 3: 4 hours real-time (16 hours game-time)
...
Tier 7: 64 hours real-time (256 hours game-time)
Tier 8: 128 hours real-time (512 hours game-time) [GOLD: 500]
Tier 9: 256 hours real-time (1024 hours game-time) [GOLD: 1500]
Tier 10: 512 hours real-time (2048 hours game-time) [GOLD: 5000]
```

### Research Bonuses

**Each Tier Provides:**
```
Damage Scaling: +10% per tier (1.0x → 2.0x at tier 10)
Defense Scaling: +10% per tier
Speed Scaling: +8% per tier
Detection Range: +5% per tier
Fuel Efficiency: +3% per tier
```

### Advanced Research Labs

```
Unlocked at: Country reaches tier 6 in 3 different units
Benefits:
- 2x research speed
- Access to secret units
- Premium unit research (tiers 8-10)
- Technology combining (fusion research)

Secret Units Require:
- Advanced lab level 3+
- Specific prerequisite techs
- 1000+ gold per tier
- Rare resource requirements
```

### Technology Tree Progression

```
EXAMPLE: Rifle Infantry Tech Tree
├── Tier 1: Basic Rifle Infantry (researched)
├── Tier 2: Improved Training
├── Tier 3: Modern Equipment
├── Tier 4: Combined Arms Tactics
├── Tier 5: Advanced Weaponry
├── Tier 6: Elite Forces Training
├── Tier 7: Weapon Systems Integration
├── Tier 8: Nanotechnology Fiber [GOLD REQUIRED]
├── Tier 9: Quantum Enhancement [GOLD REQUIRED]
└── Tier 10: Ghost Infantry Secret Unit [GOLD REQUIRED]

Each tier unlocks 10% additional unit strength
```

---

## DATABASE SCHEMA

### Core Entity Relationships

```
users
  ↓
countries ↔ coalitions
  ↓         ↓
provinces  coalition_members
  ↓
cities
  ↓
buildings, armies, resources, research

armies
  ↓
units

diplomacy_deals
market_orders
news_events
```

### Essential Tables

**users**
- id, email, username, password_hash, 2fa_enabled, created_at

**countries**
- id, player_id, name, capital_province_id, population, manpower, is_eliminated

**provinces**
- id, name, x, y, terrain, owner_country_id, population, development_level

**cities**
- id, name, province_id, type (critical/resource/logistics), population, garrison_army_id

**armies**
- id, country_id, current_province_id, status, target_province_id, morale, supply_level

**units**
- id, army_id, unit_type, count, health_per_unit, morale, experience, research_tier

**resources**
- country_id, food, oil, steel, electronics, money (indexed by country)

**buildings**
- id, city_id, type, level, is_constructing, completion_time, health

**research**
- id, country_id, unit_type, tier, is_completed, completion_time

**diplomacy_deals**
- id, type, country_a_id, country_b_id, status, terms, started_at, ended_at

**market_orders**
- id, seller_country_id, type (buy/sell), resource_type, quantity, price_per_unit, status

**coalitions**
- id, name, leader_country_id, created_at

**coalition_members**
- id, coalition_id, country_id, joined_at

**news_events**
- id, type, description, related_countries (JSONB), created_at

---

## API REFERENCE

### Authentication Endpoints

```
POST /api/auth/register
  Body: { email, username, password }
  Response: { userId, token }

POST /api/auth/login
  Body: { email, password }
  Response: { userId, token, country }

POST /api/auth/2fa/enable
POST /api/auth/2fa/verify
  Body: { code }
```

### Game State Endpoints

```
GET /api/game/state
  Response: { provinces[], cities[], armies[], countries[] }

GET /api/game/country/:id
  Response: { country data, resources, armies, buildings }

GET /api/game/armies/:countryId
  Response: { armies[] }

GET /api/game/map
  Query: { zoom, x, y }
  Response: { visible provinces based on fog of war }
```

### Military Endpoints

```
POST /api/game/army/create
  Body: { countryId, provinceId, name }
  Response: { army object }

POST /api/game/army/:armyId/move
  Body: { targetProvinceId }
  Response: { movement timestamp, ETA }

POST /api/game/army/:armyId/attack
  Body: { targetArmyId }
  Response: { combat initiated, duration }

GET /api/game/army/:armyId/units
  Response: { units[] }
```

### Diplomacy Endpoints

```
POST /api/diplomacy/propose
  Body: { type, targetCountryId, terms }
  Response: { deal created }

POST /api/diplomacy/:dealId/accept
POST /api/diplomacy/:dealId/reject

GET /api/diplomacy/relations/:countryId
  Response: { all relationships }

POST /api/diplomacy/coalition/create
  Body: { name }

POST /api/diplomacy/coalition/:coalitionId/invite
  Body: { countryId }
```

### Market Endpoints

```
GET /api/market/orders
  Query: { type, resource, sortBy }
  Response: { orders[] }

POST /api/market/order/create
  Body: { type, resource, quantity, price }
  Response: { order created }

POST /api/market/order/:orderId/cancel

POST /api/market/order/:orderId/accept
  Body: { quantity }
```

### Research Endpoints

```
GET /api/game/research/:countryId
  Response: { research progress[] }

POST /api/game/research/start
  Body: { unitType, tier }

POST /api/game/research/accelerate
  Body: { researchId, goldAmount }
```

---

## WEBSOCKET EVENTS

### Client → Server

```
authenticate
  { token }

join_game
  { countryId }

move_army
  { armyId, targetProvinceId }

attack_army
  { attackingArmyId, defendingArmyId }

build_unit
  { cityId, unitType, quantity }

build_building
  { cityId, buildingType }

send_diplomacy
  { type, targetCountryId, terms }

buy_order
  { resource, quantity, maxPrice }

sell_order
  { resource, quantity, minPrice }

request_intel
  { targetCountryId }

send_message
  { recipientCountryId, message }
```

### Server → Client

```
army_moved
  { armyId, newPosition, ETA }

combat_started
  { armyId1, armyId2, duration }

combat_update
  { combatId, damage, casualties, morale }

combat_ended
  { victoryCountryId, casualties, loot }

unit_produced
  { cityId, unitType, count, armyId }

building_completed
  { cityId, buildingType, level }

research_completed
  { countryId, unitType, tier }

resource_updated
  { countryId, resources }

diplomacy_notification
  { type, fromCountryId, terms }

market_order_filled
  { orderId, filledQuantity, price }

army_destroyed
  { armyId, reason }

news_broadcast
  { event description, affectedCountries }

map_update
  { visibleProvinces, armies, cities }
```

---

## SCALING & PERFORMANCE

### Architecture Design for 10,000+ Players

**Backend Server:**
- Node.js + Express (multi-core with cluster module)
- Socket.IO for real-time communication
- Horizontal scaling with load balancer

**Database:**
- PostgreSQL 15+ with connection pooling
- Read replicas for queries
- Partitioned tables by country_id for distributed queries

**Caching:**
- Redis for real-time state
- TTL-based expiration
- Sharded across multiple Redis instances

**Game Loop:**
- 100Hz tick rate (100 ticks/second)
- Distributed tick processing across worker threads
- State persistence every 1 second

### Performance Targets

```
Concurrent Players: 10,000
Active Games: 500+
Tick Rate: 100Hz (10ms resolution)
API Response Time: <100ms average
WebSocket Latency: <50ms average
Database Query Time: <10ms for indexed queries
Cache Hit Ratio: >95%
```

### Load Balancing Strategy

```
NGINX Load Balancer
  ↓
┌─────────────────────────────────────┐
│  Backend Server Cluster (16+ cores) │
├─────────────────────────────────────┤
│ API Server 1 (8 workers)            │
│ API Server 2 (8 workers)            │
│ API Server 3 (8 workers)            │
│ Game Engine Server (4 workers)      │
└─────────────────────────────────────┘
  ↓
PostgreSQL Primary
  ├── Replica 1 (Read)
  ├── Replica 2 (Read)
  └── Replica 3 (Backup)
  ↓
Redis Cluster (8 nodes sharded)
```

### Database Optimization

```
Connection Pooling:
- Min: 20 connections
- Max: 200 connections
- Idle timeout: 30 seconds

Query Optimization:
- Index on frequently accessed columns
- Batch queries when possible
- Cache hot data in Redis

Partitioning Strategy:
- Armies table: Partitioned by country_id
- Units table: Partitioned by army_id
- Market orders: Partitioned by timestamp

Replication:
- Synchronous replication for critical data
- Asynchronous for analytics
- WAL archiving to S3
```

### Caching Strategy

```
HOT DATA (updated every tick):
- Game state: game:state
- Army positions: army:{id}:position
- Country resources: country:{id}:resources
- Combat status: combat:{id}:state
- TTL: 1-5 seconds

WARM DATA (updated every minute):
- Research progress: research:{id}:progress
- Building construction: building:{id}:progress
- Market orders: market:orders:sell, market:orders:buy
- TTL: 30-60 seconds

COLD DATA (updated rarely):
- Building definitions: building:types:*
- Unit stats: unit:{type}:stats
- Technology tree: tech:tree:*
- TTL: 1 hour

Cache Invalidation:
- Time-based expiration
- Event-based invalidation
- Manual invalidation on updates
```

### Network Optimization

```
Message Compression: gzip for large payloads
Batch Updates: Group ticks into single message
Delta Updates: Send only changed values
Rate Limiting: 100 messages/second per player
Protocol: WebSocket with fallback to HTTP polling
```

---

## DEPLOYMENT GUIDE

### Docker Compose Setup

```yaml
services:
  postgres: PostgreSQL 15
  redis: Redis 7
  backend: Node.js API server
  frontend: React web client
  nginx: Reverse proxy & load balancer
```

### Kubernetes Deployment

```
- Backend pods: 16+ replicas
- Database pods: 3 (1 primary, 2 replicas)
- Redis pods: 8 (cluster mode)
- Ingress controller for traffic routing
- Persistent volumes for database storage
- ConfigMaps for configuration
- Secrets for credentials
```

### Monitoring & Logging

```
Metrics Collection:
- Prometheus for metrics
- Grafana for visualization
- Key metrics:
  - Tick latency
  - Player count
  - Database query time
  - Cache hit ratio
  - Network bandwidth
  - CPU/Memory usage

Logging:
- Winston logger
- Centralized logging (ELK stack)
- Game events logging
- Error tracking (Sentry)
```

---

## CONCLUSION

This comprehensive architecture provides:

✅ **Real-time Performance:** 100Hz game loop for responsive gameplay
✅ **Strategic Depth:** 60+ units, diplomacy, logistics, trade, stealth
✅ **Massive Scale:** 10,000+ concurrent players
✅ **Data Persistence:** PostgreSQL with replication & backup
✅ **Real-time Communication:** WebSocket with fallback
✅ **Advanced Systems:** Fog of war, radar, intelligence, coalition politics
✅ **Economy:** Dynamic market, resources, trade embargoes
✅ **Security:** JWT auth, 2FA, encrypted messages, encrypted transport

All systems designed for production-level deployment with horizontal scaling, failover redundancy, and high availability.

