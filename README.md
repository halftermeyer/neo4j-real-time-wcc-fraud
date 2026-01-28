# Real-Time Fraud Detection with Temporal Graphs

## 1. Introduction

In the rapidly evolving landscape of financial services, particularly in buy-now-pay-later (BNPL) and digital lending platforms, detecting fraudulent activity in real-time is critical for protecting both businesses and consumers. The Real-Time Fraud Detection use case addresses a fundamental challenge: identifying sophisticated patterns in event networks and synthetic identity schemes as they emerge, using temporal graph analysis to engineer features for machine learning models. By leveraging Neo4j's graph database capabilities and Cypher 25, organizations can model complex event sequences, track entity relationships over time, and compute graph-based metrics that reveal patterns invisible to traditional approaches. These capabilities enable proactive fraud prevention, reduce financial losses, and maintain trust in digital financial ecosystems.

## 2. Scenario

To understand the importance of real-time fraud detection with temporal graphs, consider the challenges faced by BNPL platforms that offer services with declarative identity verification (no mandatory ID documents, thus no strong authentication). The following key areas illustrate these challenges:

### 1. Synthetic Identity Fraud and Collaborative Fraud Rings

* Fraudsters create fake identities by mixing real and fabricated information, making detection difficult with traditional rule-based systems.
* Organized connected components coordinate attacks using shared infrastructure (IP addresses, devices, email domains) across multiple accounts.
* Without graph-based analysis, these connections remain hidden in isolated transaction records.
* The velocity and scale of modern fraud operations demand real-time detection capabilities.

### 2. Temporal Pattern Recognition

* Fraud patterns evolve over time as attackers probe defenses and adapt their tactics.
* Traditional static snapshots miss the dynamic nature of how connected components form and grow.
* Understanding "what did the event network look like at the time of each transaction" is crucial for accurate risk assessment.
* Historical analysis must avoid "future knowledge" contamination when training predictive models.

### 3. Feature Engineering for Machine Learning

* Effective fraud detection requires rich features that capture network effects and collaboration patterns.
* Graph metrics like connected component size, diameter, and temporal velocity provide powerful signals.
* Without structured graph analysis, ML models rely on limited transactional features and miss critical relationship patterns.
* Real-time computation of graph features enables immediate fraud scoring and intervention.

These scenarios highlight the need for an advanced solution like **Neo4j's Real-Time Fraud Detection with Temporal Graphs**, which leverages graph technology to model event sequences, track entity relationships over time, and engineer powerful features for supervised machine learning in fraud detection.

## 3. Solution

Advanced graph databases like Neo4j are essential for navigating the complexities of fraud detection in interconnected event data. They excel at representing temporal sequences, tracking entity reuse patterns, and computing graph metrics that reveal connected components, making it possible to engineer features for machine learning models in both training and real-time prediction scenarios.

### 3.1. How Graph Databases Can Help?

Graph databases offer a robust solution to the challenges of real-time fraud detection in financial services. Here are five key reasons why a graph database is indispensable:

1. **Temporal Event Modeling:** Graphs naturally capture sequences of events (logins, transactions, updates) connected through shared entities (credit cards, IP addresses, devices, emails), preserving the time-ordered nature of fraud patterns.

2. **Entity Relationship Tracking:** They enable tracking of how different identifiers (email addresses, phone numbers, bank accounts) connect across events, revealing connected components that share infrastructure.

3. **Connected Component Analysis:** Graphs support efficient identification of event networks through weakly connected component algorithms, grouping events that share any common entity.

4. **Temporal Graph Metrics:** They allow computation of "as-of-date" graph features, ensuring that feature engineering respects the temporal nature of data and avoids future knowledge contamination in ML training.

5. **Real-Time Feature Engineering:** Graphs provide efficient incremental updates to event network structures, enabling real-time computation of graph metrics for immediate fraud scoring.

These features position graph databases as key to engineering powerful features and gaining insights for real-time fraud detection in financial services.

### 3.2. Feature Engineering Workflow

The fraud detection solution follows a two-phase workflow with critical attention to temporal causality:

**Training Phase:**

1. Historical event data is ingested into an Analytics Neo4j database
2. Events are connected through shared entities, forming a temporal graph structure
3. Connected components are identified and component metrics (size, diameter, velocity) are computed and materialized
4. **For each training event:** Features are extracted by looking at the components it connects to through shared entities (as they existed at that event's timestamp)
5. These features, combined with transactional data, are used to train supervised machine learning models

**Real-Time Detection Phase:**

1. Live transaction events stream into a high-availability Neo4j transactional cluster
2. New events are incorporated into the graph structure
3. **For each new event:** Features are extracted by querying the current components it connects to through shared entities
4. Features feed into the trained ML model for immediate fraud prediction (typical query time: ~2ms)

**Key principle:** An event's risk is assessed based on the pre-existing components it connects to or bridges, not based on the component it eventually becomes part of. This ensures temporal consistency between training and production.

## 4. Modelling

This section describes the graph data model and provides data ingestion guidance. The goal is to demonstrate a flexible, generic Event-Thing model that can adapt to various fraud detection scenarios while supporting efficient temporal analysis.

### 4.1. Data Model

The data model uses a generic Event-Thing pattern where events (user actions) connect to various identifying entities:

**Fraud Detection Data Model:**

```
(Event)-[:WITH]->(CreditCard)
(Event)-[:WITH]->(IPAddress)
(Event)-[:WITH]->(EmailAddress)
(Event)-[:WITH]->(BankAccount)
(Event)-[:WITH]->(PhoneNumber)
(Event)-[:WITH]->(Device)
(Event)-[:WITH]->(Session)

(Event)-[:SEQUENTIALLY_RELATED]->(Event)  // Temporal ordering within entity groups
(Event)-[:COMPONENT_PARENT]->(Event)      // Union-find connected component structure

Event nodes gain :ComponentNode label when processed into the structure
```

**Connected Component Structure:**

```
Events connected through shared entities form weakly connected components:
- SEQUENTIALLY_RELATED: Orders events chronologically for each shared entity
- COMPONENT_PARENT: Links events into a union-find forest representing connected components
```

#### 4.1.1 Required Data Fields

Below are the fields required to get started:

**Event Node:**
* `event_id`: Unique identifier for the event
* `timestamp`: Event timestamp (datetime)
* `interaction_type`: Type of event (login, transaction, update)
* `transaction_amount`: Transaction amount (if applicable, float)

**Entity Nodes (CreditCard, IPAddress, EmailAddress, BankAccount, PhoneNumber, Device, Session):**
* Identifier property matching the node label (e.g., `credit_card_id`, `ip_address`, `email`)

**WITH Relationship:**
* Connects Event nodes to entity nodes
* No additional properties required

**SEQUENTIALLY_RELATED Relationship:**
* Connects consecutive events for each entity (temporal ordering)
* Creates a partial ordering within connected components
* Represents high probability of collaboration between events

**COMPONENT_PARENT Relationship:**
* Forms union-find forest structure for connected components
* Enables efficient "as-of-date" queries
* Points from older events to newer events in the connected component

**ComponentNode Label:**
* Applied to Event nodes that have been processed and incorporated into the COMPONENT_PARENT structure
* Used to track which events are already part of the connected component analysis
* Enables incremental updates by identifying new events with `Event&!ComponentNode`
* Removed during cleanup to reset the analysis

### 4.2. Demo Data

The following Cypher statement creates a small test dataset with realistic fraud patterns, including a chain-structured fraud ring with diameter > 1, a simpler fraud ring, a bridging event connecting both rings, and isolated legitimate events:

```cypher
// Create constraints and index
CREATE CONSTRAINT event_id_unique IF NOT EXISTS 
FOR (e:Event) REQUIRE (e.event_id) IS UNIQUE;
CREATE CONSTRAINT email_unique IF NOT EXISTS 
FOR (e:EmailAddress) REQUIRE (e.email) IS UNIQUE;
CREATE CONSTRAINT ip_address_unique IF NOT EXISTS 
FOR (i:IPAddress) REQUIRE (i.ip_address) IS UNIQUE;
CREATE CONSTRAINT credit_card_id_unique IF NOT EXISTS 
FOR (c:CreditCard) REQUIRE (c.credit_card_id) IS UNIQUE;
CREATE CONSTRAINT bank_account_id_unique IF NOT EXISTS 
FOR (b:BankAccount) REQUIRE (b.bank_account_id) IS UNIQUE;
CREATE CONSTRAINT device_id_unique IF NOT EXISTS 
FOR (d:Device) REQUIRE (d.device_id) IS UNIQUE;
CREATE CONSTRAINT session_id_unique IF NOT EXISTS 
FOR (s:Session) REQUIRE (s.session_id) IS UNIQUE;
CREATE INDEX event_timestamp IF NOT EXISTS
FOR (e:Event) ON (e.timestamp);

CALL db.awaitIndexes(300);

// Create entity nodes
CREATE (:EmailAddress {email: 'fraud@example.com'});
CREATE (:EmailAddress {email: 'fraud2@example.com'});
CREATE (:EmailAddress {email: 'legitimate1@example.com'});
CREATE (:EmailAddress {email: 'legitimate2@example.com'});
CREATE (:EmailAddress {email: 'legitimate3@example.com'});
CREATE (:IPAddress {ip_address: '192.168.1.10'});
CREATE (:IPAddress {ip_address: '192.168.1.11'});
CREATE (:IPAddress {ip_address: '10.0.0.5'});
CREATE (:IPAddress {ip_address: '172.16.0.20'});
CREATE (:IPAddress {ip_address: '203.0.113.50'});
CREATE (:CreditCard {credit_card_id: 'cc_stolen_001'});
CREATE (:CreditCard {credit_card_id: 'cc_legit_001'});
CREATE (:CreditCard {credit_card_id: 'cc_legit_002'});
CREATE (:CreditCard {credit_card_id: 'cc_legit_003'});
CREATE (:Device {device_id: 'device_fraud_001'});
CREATE (:Device {device_id: 'device_fraud_002'});
CREATE (:Device {device_id: 'device_legit_001'});
CREATE (:Device {device_id: 'device_legit_002'});
CREATE (:Device {device_id: 'device_legit_003'});
CREATE (:BankAccount {bank_account_id: 'ba_fraud_001'});
CREATE (:BankAccount {bank_account_id: 'ba_fraud_002'});
CREATE (:BankAccount {bank_account_id: 'ba_legit_001'});
CREATE (:BankAccount {bank_account_id: 'ba_legit_002'});
CREATE (:Session {session_id: 'session_001'});
CREATE (:Session {session_id: 'session_002'});
CREATE (:Session {session_id: 'session_003'});
CREATE (:Session {session_id: 'session_004'});
CREATE (:Session {session_id: 'session_005'});
CREATE (:Session {session_id: 'session_006'});
CREATE (:Session {session_id: 'session_007'});
CREATE (:Session {session_id: 'session_008'});

// Create events - Fraud Ring A (chain structure, diameter > 1)
CREATE (:Event {event_id: 'evt_fraud_a1', timestamp: datetime('2024-01-15T10:00:00Z'), interaction_type: 'login', transaction_amount: null});
CREATE (:Event {event_id: 'evt_fraud_a2', timestamp: datetime('2024-01-15T10:02:00Z'), interaction_type: 'transaction', transaction_amount: 1500.00});
CREATE (:Event {event_id: 'evt_fraud_a3', timestamp: datetime('2024-01-15T10:05:00Z'), interaction_type: 'transaction', transaction_amount: 2300.00});
CREATE (:Event {event_id: 'evt_fraud_a4', timestamp: datetime('2024-01-15T10:08:00Z'), interaction_type: 'update', transaction_amount: null});

// Create events - Fraud Ring B (star structure, diameter = 1)
CREATE (:Event {event_id: 'evt_fraud_b1', timestamp: datetime('2024-01-15T14:00:00Z'), interaction_type: 'transaction', transaction_amount: 500.00});
CREATE (:Event {event_id: 'evt_fraud_b2', timestamp: datetime('2024-01-15T14:30:00Z'), interaction_type: 'transaction', transaction_amount: 750.00});
CREATE (:Event {event_id: 'evt_fraud_b3', timestamp: datetime('2024-01-15T15:00:00Z'), interaction_type: 'transaction', transaction_amount: 1200.00});

// Create event - Bridging event (connects both rings)
CREATE (:Event {event_id: 'evt_bridge', timestamp: datetime('2024-01-15T16:00:00Z'), interaction_type: 'transaction', transaction_amount: 3500.00});

// Create events - Legitimate isolated events
CREATE (:Event {event_id: 'evt_legit_1', timestamp: datetime('2024-01-15T09:00:00Z'), interaction_type: 'login', transaction_amount: null});
CREATE (:Event {event_id: 'evt_legit_2', timestamp: datetime('2024-01-15T11:00:00Z'), interaction_type: 'transaction', transaction_amount: 45.00});
CREATE (:Event {event_id: 'evt_legit_3', timestamp: datetime('2024-01-15T13:00:00Z'), interaction_type: 'update', transaction_amount: null});

// Create WITH relationships - Fraud Ring A (chain: a1←IP1→a2←Email2/Bank→a3←Device2→a4)
MATCH (e:Event {event_id: 'evt_fraud_a1'}), (ip:IPAddress {ip_address: '192.168.1.10'}) CREATE (e)-[:WITH]->(ip);
MATCH (e:Event {event_id: 'evt_fraud_a1'}), (email:EmailAddress {email: 'fraud@example.com'}) CREATE (e)-[:WITH]->(email);
MATCH (e:Event {event_id: 'evt_fraud_a1'}), (s:Session {session_id: 'session_001'}) CREATE (e)-[:WITH]->(s);
MATCH (e:Event {event_id: 'evt_fraud_a2'}), (ip:IPAddress {ip_address: '192.168.1.10'}) CREATE (e)-[:WITH]->(ip);
MATCH (e:Event {event_id: 'evt_fraud_a2'}), (email:EmailAddress {email: 'fraud2@example.com'}) CREATE (e)-[:WITH]->(email);
MATCH (e:Event {event_id: 'evt_fraud_a2'}), (ba:BankAccount {bank_account_id: 'ba_fraud_001'}) CREATE (e)-[:WITH]->(ba);
MATCH (e:Event {event_id: 'evt_fraud_a2'}), (s:Session {session_id: 'session_002'}) CREATE (e)-[:WITH]->(s);
MATCH (e:Event {event_id: 'evt_fraud_a3'}), (email:EmailAddress {email: 'fraud2@example.com'}) CREATE (e)-[:WITH]->(email);
MATCH (e:Event {event_id: 'evt_fraud_a3'}), (ba:BankAccount {bank_account_id: 'ba_fraud_001'}) CREATE (e)-[:WITH]->(ba);
MATCH (e:Event {event_id: 'evt_fraud_a3'}), (d:Device {device_id: 'device_fraud_002'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_fraud_a3'}), (s:Session {session_id: 'session_003'}) CREATE (e)-[:WITH]->(s);
MATCH (e:Event {event_id: 'evt_fraud_a4'}), (d:Device {device_id: 'device_fraud_002'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_fraud_a4'}), (ip:IPAddress {ip_address: '192.168.1.11'}) CREATE (e)-[:WITH]->(ip);
MATCH (e:Event {event_id: 'evt_fraud_a4'}), (s:Session {session_id: 'session_004'}) CREATE (e)-[:WITH]->(s);

// Create WITH relationships - Fraud Ring B (star: all share credit card and device)
MATCH (e:Event {event_id: 'evt_fraud_b1'}), (cc:CreditCard {credit_card_id: 'cc_stolen_001'}) CREATE (e)-[:WITH]->(cc);
MATCH (e:Event {event_id: 'evt_fraud_b1'}), (d:Device {device_id: 'device_fraud_001'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_fraud_b1'}), (ip:IPAddress {ip_address: '10.0.0.5'}) CREATE (e)-[:WITH]->(ip);
MATCH (e:Event {event_id: 'evt_fraud_b1'}), (s:Session {session_id: 'session_005'}) CREATE (e)-[:WITH]->(s);
MATCH (e:Event {event_id: 'evt_fraud_b2'}), (cc:CreditCard {credit_card_id: 'cc_stolen_001'}) CREATE (e)-[:WITH]->(cc);
MATCH (e:Event {event_id: 'evt_fraud_b2'}), (d:Device {device_id: 'device_fraud_001'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_fraud_b2'}), (ip:IPAddress {ip_address: '172.16.0.20'}) CREATE (e)-[:WITH]->(ip);
MATCH (e:Event {event_id: 'evt_fraud_b2'}), (s:Session {session_id: 'session_006'}) CREATE (e)-[:WITH]->(s);
MATCH (e:Event {event_id: 'evt_fraud_b3'}), (cc:CreditCard {credit_card_id: 'cc_stolen_001'}) CREATE (e)-[:WITH]->(cc);
MATCH (e:Event {event_id: 'evt_fraud_b3'}), (d:Device {device_id: 'device_fraud_001'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_fraud_b3'}), (ip:IPAddress {ip_address: '203.0.113.50'}) CREATE (e)-[:WITH]->(ip);
MATCH (e:Event {event_id: 'evt_fraud_b3'}), (s:Session {session_id: 'session_007'}) CREATE (e)-[:WITH]->(s);

// Create WITH relationships - Bridging event
MATCH (e:Event {event_id: 'evt_bridge'}), (ip:IPAddress {ip_address: '192.168.1.10'}) CREATE (e)-[:WITH]->(ip);
MATCH (e:Event {event_id: 'evt_bridge'}), (cc:CreditCard {credit_card_id: 'cc_stolen_001'}) CREATE (e)-[:WITH]->(cc);
MATCH (e:Event {event_id: 'evt_bridge'}), (d:Device {device_id: 'device_fraud_001'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_bridge'}), (ba:BankAccount {bank_account_id: 'ba_fraud_002'}) CREATE (e)-[:WITH]->(ba);
MATCH (e:Event {event_id: 'evt_bridge'}), (s:Session {session_id: 'session_008'}) CREATE (e)-[:WITH]->(s);

// Create WITH relationships - Legitimate events (isolated)
MATCH (e:Event {event_id: 'evt_legit_1'}), (email:EmailAddress {email: 'legitimate1@example.com'}) CREATE (e)-[:WITH]->(email);
MATCH (e:Event {event_id: 'evt_legit_1'}), (cc:CreditCard {credit_card_id: 'cc_legit_001'}) CREATE (e)-[:WITH]->(cc);
MATCH (e:Event {event_id: 'evt_legit_1'}), (d:Device {device_id: 'device_legit_001'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_legit_2'}), (email:EmailAddress {email: 'legitimate2@example.com'}) CREATE (e)-[:WITH]->(email);
MATCH (e:Event {event_id: 'evt_legit_2'}), (cc:CreditCard {credit_card_id: 'cc_legit_002'}) CREATE (e)-[:WITH]->(cc);
MATCH (e:Event {event_id: 'evt_legit_2'}), (d:Device {device_id: 'device_legit_002'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_legit_2'}), (ba:BankAccount {bank_account_id: 'ba_legit_001'}) CREATE (e)-[:WITH]->(ba);
MATCH (e:Event {event_id: 'evt_legit_3'}), (email:EmailAddress {email: 'legitimate3@example.com'}) CREATE (e)-[:WITH]->(email);
MATCH (e:Event {event_id: 'evt_legit_3'}), (cc:CreditCard {credit_card_id: 'cc_legit_003'}) CREATE (e)-[:WITH]->(cc);
MATCH (e:Event {event_id: 'evt_legit_3'}), (d:Device {device_id: 'device_legit_003'}) CREATE (e)-[:WITH]->(d);
MATCH (e:Event {event_id: 'evt_legit_3'}), (ba:BankAccount {bank_account_id: 'ba_legit_002'}) CREATE (e)-[:WITH]->(ba);
```

The dataset includes 11 events organized into:

**Fraud Ring A (4 events):** Chain structure with diameter > 1
* Events: `evt_fraud_a1`, `evt_fraud_a2`, `evt_fraud_a3`, `evt_fraud_a4`
* Pattern: `a1 ← IP1 → a2 ← Email2/Bank → a3 ← Device2 → a4`
* Demonstrates complex multi-hop fraud network

**Fraud Ring B (3 events):** Star structure with diameter = 1
* Events: `evt_fraud_b1`, `evt_fraud_b2`, `evt_fraud_b3`
* Pattern: All share credit card `cc_stolen_001` and device `device_fraud_001`

**Bridging Event (1 event):** Connects both fraud rings
* Event: `evt_bridge`
* Shares IP with Ring A and credit card/device with Ring B
* Expected to show `distinct_connected_components_count = 2`

**Legitimate Events (3 events):** Isolated activity with no shared entities
* Events: `evt_legit_1`, `evt_legit_2`, `evt_legit_3`

<img width="1012" height="597" alt="graph_only" src="https://github.com/user-attachments/assets/b4b6e196-d32b-474b-8fed-a13384bc1262" />

#### 4.2.1 Alternative: Import Larger Dataset with Neo4j Data Importer

For testing at scale, you can use the provided [`data-importer-real-time-fraud.zip`](https://github.com/halftermeyer/neo4j-real-time-wcc-fraud/raw/refs/heads/main/data-importer-real-time-fraud.zip) archive containing a larger dataset with 7,800 events and the complete import model:

1. Download or locate the [`data-importer-real-time-fraud.zip`](https://github.com/halftermeyer/neo4j-real-time-wcc-fraud/raw/refs/heads/main/data-importer-real-time-fraud.zip) file
2. Open Neo4j Data Importer in your Neo4j environment
3. Click "Load Model (with data)" and select the archive file
4. The archive contains:
   * `credit_fraud_dataset.csv` - 7,800 synthetic fraud events
   * `neo4j_importer_model.json` - Complete graph model mapping
5. Review the mapping (Event-Thing model with all entity types)
6. Click "Run Import" to load the full dataset

This larger dataset provides more realistic patterns for testing component metrics, diameter computation, and feature extraction at scale. The same queries in this documentation will work with this dataset, but computation times of the data structure will be longer due to the larger graph size.

**Dataset characteristics:**
* 7,800 events spanning multiple days
* Realistic distribution of interaction types (login, transaction, update)
* Complex connected component patterns
* Suitable for performance testing and scalability analysis

## 5. Cypher Queries

> **Note:** These Cypher queries are compatible with Neo4j Version 2025.06+ and Cypher 25.

### 5.1. Set Analysis Parameters

Define parameters for temporal analysis and testing

#### 5.1.1. Demo dataset

```cypher
:param {
  asOfDate: datetime('2024-01-15T17:00:00Z')
  event_id: "evt_bridge"
}
```

#### 5.1.1. Alternative dataset

```cypher
:param {
  asOfDate: datetime('2024-01-15T17:00:00Z')
  event_id: "e85b9c886-f8f0-44fb-b579-a3537310249c"
}
```


### 5.2. Create SEQUENTIALLY_RELATED Relationships

This query creates temporal ordering relationships between consecutive events for each shared entity. The SEQUENTIALLY_RELATED relationship represents a strong indicator of potential collaboration (high probability that events sharing the same entity are related):

```cypher
MATCH (thing:BankAccount|CreditCard|Device|EmailAddress|Event|IPAddress|PhoneNumber|Session)
CALL (thing) {
  MATCH (e:Event)-[:WITH]->(thing)
  WITH DISTINCT e
  ORDER BY e.timestamp
  WITH collect(e) AS events
  WITH CASE size(events)
    WHEN 1 THEN [events[0], null]
    ELSE events END AS events
  UNWIND range(0, size(events)-2) AS ix
  WITH events[ix] AS source, events[ix+1] AS target
  MERGE (source)-[:SEQUENTIALLY_RELATED]->(target)
} IN TRANSACTIONS OF 100 ROWS
```

**Key optimization:** This query creates a path structure rather than a clique for events sharing the same entity. Since temporal ordering is transitive (if event A precedes B and B precedes C, then A transitively precedes C), we only need to connect consecutive events in chronological order. This dramatically reduces relationship count from O(n²) to O(n) for n events on the same entity, while preserving full graph connectivity through traversal patterns like `()-[:SEQUENTIALLY_RELATED*]->()`. For entities with many events (e.g., a popular IP address with 100 events), this saves 4,950 relationships per entity while maintaining the same analytical capabilities. This path optimization technique is detailed in [Mastering Fraud Detection with Temporal Graphs](https://neo4j.com/blog/developer/mastering-fraud-detection-temporal-graph/).

**Note on concurrency:** This query uses non-concurrent transactions to avoid potential deadlocks when multiple transactions attempt to create relationships on the same events. For further performance optimization, the WCC batching technique described in Section 5.4 could be applied here to enable concurrent processing while avoiding crashes.

### 5.3. Efficient GDS Projection for Connected Components

Project the temporal graph for weakly connected component analysis using GDS. This projection is optimized for performance using techniques described in [Optimizing Weakly Connected Component Projections](https://neo4j.com/blog/developer/optimize-weakly-connected-component-projections/):

```cypher
CYPHER runtime=parallel
MATCH (source:Event)
OPTIONAL MATCH (source)-[:SEQUENTIALLY_RELATED]->(target)
RETURN gds.graph.project(
  'wcc_graph', source, target, {});
```

<img width="1012" height="597" alt="graph_seq" src="https://github.com/user-attachments/assets/4c2180a3-6946-4dd9-96a6-6e65a92c9326" />


### 5.4. Build COMPONENT_PARENT Union-Find Structure

This query uses GDS to identify connected components and builds the COMPONENT_PARENT union-find forest structure in batches. **This is where the WCC batching optimization technique is applied** - using GDS WCC to identify components upfront, then batching the union-find structure creation by processing 100 connected components concurrently. This approach, detailed in [WCC Batching to Avoid Query Crashes](https://neo4j.com/blog/developer/wcc-to-avoid-cypher-query-crashing/) and [Optimizing WCC Projections](https://neo4j.com/blog/developer/optimize-weakly-connected-component-projections/), avoids query crashes and enables efficient parallel processing:

```cypher
CALL gds.wcc.stream('wcc_graph')
YIELD nodeId, componentId
WITH gds.util.asNode(nodeId) AS event, componentId
WITH componentId, collect(event) AS events
ORDER BY rand() // Mixing CCs for workload distribution
CALL (events) {
  UNWIND events AS e
  WITH e
  WHERE NOT e:ComponentNode
  ORDER BY e.timestamp ASC
  CALL (e) {
    SET e:ComponentNode
    WITH e
    MATCH (x:ComponentNode)-[:SEQUENTIALLY_RELATED]->(e)
    MATCH (x)-[:COMPONENT_PARENT]->*(cc WHERE NOT EXISTS {(cc)-[:COMPONENT_PARENT]->()})
    MERGE (cc)-[:COMPONENT_PARENT]->(e)
  }
} IN CONCURRENT TRANSACTIONS OF 100 ROWS
```

**Key points:**

**WCC Batching Technique:**
* GDS WCC algorithm efficiently identifies which events belong to the same connected component
* Each connected component becomes one batch unit (one row)
* Transactions process 100 connected components at a time with concurrent execution
* This batching approach avoids query crashes and enables safe parallel processing
* Random ordering (`ORDER BY rand()`) distributes workload evenly

**Union-Find Structure Building:**
* Within each CC, events are ordered chronologically to build the union-find structure
* The COMPONENT_PARENT relationships form a forest where each tree represents a connected component

### 5.5. Incremental Update of Connected Components

For real-time fraud detection, incrementally update the structure as new events arrive:

```cypher
// Update CC structure for new events
MATCH (e:Event&!ComponentNode)
WITH e ORDER BY e.timestamp
CALL (e) {
  SET e:ComponentNode
  WITH e
  MATCH (x:ComponentNode)-[:SEQUENTIALLY_RELATED]->(e)
  MATCH (x)-[:COMPONENT_PARENT]->*(cc WHERE NOT EXISTS {(cc)-[:COMPONENT_PARENT]->()})
  MERGE (cc)-[:COMPONENT_PARENT]->(e)
} IN TRANSACTIONS OF 100 ROWS
```

<img width="1050" height="637" alt="graph" src="https://github.com/user-attachments/assets/f128c1e3-7a9e-4819-bb89-4e2247df0e7b" />


### 5.6. Show Event's Connected Component with Context

View a specific event's connected component with all related entities:

```cypher
MATCH (ev:Event)
WHERE EXISTS {(ev)-[:COMPONENT_PARENT]->*(e:Event {event_id: $event_id})}
RETURN [(ev)-[r:WITH]->(x)| [ev, r, x]] AS events_with_things
```

<img width="1012" height="597" alt="comp_in_context" src="https://github.com/user-attachments/assets/95bcc6ef-36ae-498d-b314-d38e1d0f88dc" />

### 5.7. Temporal Analysis: Connected Components As-Of Date

Query the state of connected components as they existed at a specific point in time. This is crucial for training ML models without future knowledge contamination:

```cypher
CYPHER 25 runtime=parallel
MATCH (cc:Event
  WHERE cc.timestamp <= $asOfDate
  AND NOT EXISTS {(cc)-[:COMPONENT_PARENT]->(x:Event WHERE x.timestamp <= $asOfDate)}
  )
MATCH path = (ev)-[:COMPONENT_PARENT]->*(cc)
WITH cc, collect(ev) AS cc_elements
ORDER BY size(cc_elements) DESC
RETURN cc, cc_elements
```

```csv
cc.event_id,component
evt_fraud_a4,"[evt_fraud_a4, evt_fraud_a3, evt_fraud_a2, evt_fraud_a1]"
evt_fraud_b3,"[evt_fraud_b3, evt_fraud_b2, evt_fraud_b1]"
evt_legit_1,[evt_legit_1]
evt_legit_3,[evt_legit_3]
evt_legit_2,[evt_legit_2]
```

### 5.8. Compute Component Metrics

This query computes metrics for each connected component from every ComponentNode event's temporal perspective. Metrics are materialized as properties on ComponentNode events, representing the component state as it existed at each event's timestamp. These metrics will later be used to extract features for individual events:

```cypher
// compute and persist all component metrics
CYPHER 25
MATCH (source:Event)
CALL (source) {
    
    // retrieve component
	WITH source, COLLECT{
		MATCH (cc_element)-[:COMPONENT_PARENT]->*(source) RETURN cc_element
	} AS cc_elements
    RETURN source, cc_elements, toString(source.event_id) AS graph_name

    NEXT
    
    // compute component size, diameter and velocity
    CALL (source, cc_elements, graph_name){
        UNWIND cc_elements AS element
        MATCH (element)
        OPTIONAL MATCH (element)-[:WITH]->(thing)
        WITH gds.graph.project(
            graph_name,
            element,
            thing,
            {},
            {undirectedRelationshipTypes:['*']}
        ) AS graph
        CALL (cc_elements, graph_name, graph) {
             WHEN graph IS NOT NULL AND graph.relationshipCount > 0 THEN {
                  UNWIND cc_elements AS element
                  CALL gds.allShortestPaths.delta.stream(graph_name, {
                      sourceNode: element
                  })
                  YIELD path
                  WITH length(path) AS length
                  RETURN max(length) / 2 AS diameter
                }
            ELSE {
              RETURN INFINITY AS diameter
            }
          }
        CALL gds.graph.drop(graph_name, false)
        YIELD graphName
        WITH source, size(cc_elements) AS nb_elements, diameter
        SET source.component_size = nb_elements, source.component_diameter = diameter

        // compute velocity
        WITH source, cc_elements
        UNWIND cc_elements AS element
        WITH source, element ORDER BY element.timestamp ASC
        WITH source, collect(element) AS ordered_elements
        WITH
          source,
          duration.inSeconds(ordered_elements[0].timestamp, ordered_elements[-1].timestamp).seconds AS time_span,
          size(ordered_elements) AS nb_events
        SET source.component_velocity =
          CASE WHEN time_span > 0 
          THEN toFloat(nb_events) / time_span  // events per second
          ELSE 0.0 
          END
    }
} IN 8 CONCURRENT TRANSACTIONS OF 100 ROWS  // Adjust concurrency level based on infrastructure
```

**Key aspects of component metric computation:**

* For each ComponentNode event, compute metrics for its component as it existed at that event's timestamp
* Each ComponentNode represents a temporal snapshot - it's the "root" from its own time perspective
* The query `MATCH (cc_element)-[:COMPONENT_PARENT]->*(source)` retrieves all events that form the component as of `source`'s timestamp
* This union-find structure naturally enables "as-of" analysis for every event in the graph
* 8 concurrent transactions process events in batches of 100 rows for parallel computation (adjust concurrency level based on your infrastructure)

**Critical Performance Advantage:**

The union-find COMPONENT_PARENT structure enables retrieving a source-relative component almost for free with `MATCH (cc_element)-[:COMPONENT_PARENT]->*(source)`. This allows creating small, focused GDS projections (micro-projections) per event rather than projecting the entire dataset. Traditional time-threshold approaches would project the full dataset in the worst case, making per-event loops prohibitively expensive. With this approach, we efficiently process each event in a loop with its own micro-projection.

**Real-world impact:** For a tier 1 French bank customer, computing component metrics for the test dataset went from **1 week to 10 minutes** using this union-find micro-projection approach.

**Computed Metrics:**

* **Component Size:** Number of events in the connected component (stored as `component_size`)
* **Component Diameter:** Maximum distance between any two events in the bipartite graph (Events ↔ Things)
  - Computed using `gds.allShortestPaths.delta.stream` on the micro-projection
  - Divided by 2 because the graph is bipartite
  - Stored as `component_diameter`
* **Component Velocity:** Rate of event activity measured as events per second
  - Computed as: total events / time span from first to last event
  - Returns 0.0 when all events occur simultaneously
  - Stored as `component_velocity`
* These metrics are computed once and reused for feature extraction

### 5.9. Extract Features for Training Dataset

This query extracts features for events in the training dataset by examining the components they connect to through shared entities. This ensures temporal consistency - we only look at components that existed at or before each event's timestamp:

```cypher
// enrich training dataset with component metrics
CYPHER 25 runtime=parallel
MATCH (source:Event WHERE source.timestamp <= $asOfDate)
OPTIONAL MATCH (cc)-[:COMPONENT_PARENT]->(source)
WITH source, collect(cc) AS ccs
WITH
  source, ccs,
  [cc IN ccs|cc.component_size] AS sizes,
  [cc IN ccs|cc.component_diameter] AS diameters,
  [cc IN ccs|cc.component_velocity] AS velocities
RETURN 
  // compute your target metrics
  source.event_id AS event_id,
  coll.max(sizes) AS max_connected_component_size,
  coll.max(diameters) AS max_connected_component_diameter,
  coll.max(velocities) AS max_connected_component_velocity,
  size(ccs) AS distinct_connected_components_count
```

```csv
// Result
event_id,max_connected_component_size,max_connected_component_diameter,max_connected_component_velocity,distinct_connected_components_count
evt_fraud_a1,null,null,null,0
evt_fraud_a3,2,1,0.016666666666666666,1
evt_fraud_a2,1,0,0.0,1
evt_fraud_b1,null,null,null,0
evt_fraud_a4,3,2,0.01,1
evt_fraud_b3,2,1,0.0011111111111111111,1
evt_fraud_b2,1,0,0.0,1
evt_legit_1,null,null,null,0
evt_legit_3,null,null,null,0
evt_legit_2,null,null,null,0
```

**Key aspects:**

* Only includes events up to training cutoff date (`source.timestamp <= $asOfDate`)
* Traverses backward: `(cc)-[:COMPONENT_PARENT]->(source)` finds components this event connects to
* Aggregates metrics from all connected components (max values and count)
* Returns features in format suitable for ML training pipeline

### 5.10. Extract Features for Real-Time Scoring

This query extracts the same features for a new incoming event, enabling real-time fraud scoring:

```cypher
CYPHER 25
MATCH (new_event:Event {event_id: "e85b9c886-f8f0-44fb-b579-a3537310249c"})
MATCH (new_event)-[:WITH]->()<-[:WITH]-(related_event WHERE related_event.timestamp < new_event.timestamp)
 (
 ()-[:COMPONENT_PARENT]->(x:ComponentNode)
WHERE x <> new_event
 )*(cc:ComponentNode
WHERE NOT EXISTS {(cc)-[:COMPONENT_PARENT]->(y WHERE y <> new_event)}
)
WITH DISTINCT new_event, cc
WITH new_event, collect(cc) AS ccs
WITH
  new_event, ccs,
  [cc IN ccs|cc.component_size] AS sizes,
  [cc IN ccs|cc.component_diameter] AS diameters,
  [cc IN ccs|cc.component_velocity] AS velocities
RETURN
  // compute your target metrics
  new_event.event_id AS event_id,
  coll.max(sizes) AS max_connected_component_size,
  coll.max(diameters) AS max_connected_component_diameter,
  coll.max(velocities) AS max_connected_component_velocity,
  size(ccs) AS distinct_connected_components_count
```

```csv
// Result
event_id,max_connected_component_size,max_connected_component_diameter,max_connected_component_velocity,distinct_connected_components_count
evt_bridge,4,3,0.008333333333333333,2
```

**Key aspects:**

* Traverses through shared entities: `(new_event)-[:WITH]->()<-[:WITH]-()`
* Follows COMPONENT_PARENT paths while excluding the new event itself: `WHERE x <> new_event`
* Finds component heads while excluding paths to new_event: `WHERE NOT EXISTS {(cc)-[:COMPONENT_PARENT]->(y WHERE y <> new_event)}`
* **Deduplicates components:** `WITH DISTINCT new_event, cc` ensures each component is counted once even if reached through multiple shared entities
* This pattern ensures robust component detection even if new_event already exists in the graph
* Returns identical features as training query
* **Performance:** Typical execution time ~2ms, enabling real-time fraud detection

### 5.11. Clean Up for Re-analysis

Remove analysis structures to start fresh:

```cypher
// Remove SEQUENTIALLY_RELATED and COMPONENT_PARENT relationships
MATCH (n)-[r:SEQUENTIALLY_RELATED|COMPONENT_PARENT]->()
DELETE r;

// Remove ComponentNode label
MATCH (n:ComponentNode) 
REMOVE n:ComponentNode;

// Drop all GDS projections
CYPHER 25
CALL gds.graph.list()
YIELD graphName AS name
CALL gds.graph.drop(name)
YIELD graphName
RETURN graphName + " dropped";
```

## 6. Feature Engineering for Machine Learning

The graph-based approach enables computation of powerful features for supervised fraud detection models:

### 6.1. Key Graph Features

**Important:** These features are predictive indicators, not fraud labels. Connected components may contain legitimate or fraudulent activity. The ML model uses these features to assess risk, and final fraud determination is made by the model or through additional investigation.

**Critical Design Principle:** Features are extracted by examining the components an event connects to through shared entities, not the component the event eventually becomes part of. This ensures temporal consistency between training and production.

The following features are extracted for each event:

**`max_connected_component_size`:** Largest component size among all components connected through shared entities
* High values indicate the event connects to large, established networks
* Events bridging multiple large components may indicate coordination
* Range: 1 to total events in largest connected component

**`max_connected_component_diameter`:** Largest diameter among all connected components
* Measures maximum network complexity the event touches
* Higher values indicate connection to geographically or structurally dispersed networks
* Computed from ComponentNode events where diameter was pre-calculated for their temporal component snapshot

**`max_connected_component_velocity`:** Highest event frequency among connected components
* Measured as events per second
* High values indicate connection to components with burst activity patterns
* Range: 0.0 (simultaneous events or single event) to potentially very high rates

**`distinct_connected_components_count`:** Number of different components this event bridges
* **Critical feature:** Events connecting many distinct components are suspicious
* Indicates potential coordination or account takeover patterns
* Low values (1-2) are normal; high values (5+) warrant investigation
* This "bridging" behavior is a strong fraud indicator

**Example interpretation:**

```
Event with:
- max_connected_component_size = 50
- max_connected_component_diameter = 8
- max_connected_component_velocity = 2.5
- distinct_connected_components_count = 5

Interpretation: This event connects to 5 different established networks,
the largest having 50 events with complex structure (diameter 8) and
moderate activity (2.5 events/sec). High bridging count is suspicious.
```

### 6.2. Temporal Causality and Feature Extraction

The feature extraction approach ensures perfect temporal consistency between training and production:

**The Problem:**

Traditional approaches compute features for an event based on the component it eventually becomes part of. This creates a temporal mismatch:

* **Training:** Event E at time T is labeled with features from its final component state (which includes future events)
* **Production:** New event E has no final component state yet
* **Result:** Training features use future knowledge that isn't available at prediction time

**The Solution:**

Extract features by looking at components the event connects to through shared entities:

**During Training:**

* For historical event E with timestamp T
* Look at components that exist at or before time T
* Find components connected through E's entities (email, IP, device, etc.)
* Extract aggregate features from those pre-existing components
* These features represent what was observable at time T

**During Production:**

* New event E arrives
* Look at current components (all exist before E by definition)
* Find components connected through E's entities
* Extract identical aggregate features
* Pass to ML model for scoring

**Key Insight:**

An event's fraud risk is determined by the networks it connects to or bridges, not by where it eventually ends up. A legitimate event might eventually be part of a complex component, but at the time it occurred, it only connected to benign components.

### 6.3. Model Training Workflow

1. Build the temporal graph structure (SEQUENTIALLY_RELATED and COMPONENT_PARENT relationships)
2. Compute component metrics (size, diameter, velocity) for all ComponentNode events, capturing each event's temporal component snapshot
3. Extract features for training dataset:
   * Query events up to training cutoff date
   * For each event, find connected components through shared entities
   * Extract aggregate features (max size, max diameter, max velocity, distinct count)
   * Export features along with transactional data and fraud labels
4. Train supervised ML model (e.g., gradient boosting, neural network) on combined feature set
5. Deploy model to production environment
6. For new events:
   * Execute real-time feature extraction query (~2ms)
   * Combine with transactional features
   * Score with trained model
   * Flag high-risk events for review or blocking

**Performance Characteristics:**
* Component metrics: Computed once in batch (can be expensive but done offline)
* Feature extraction: ~2ms per event (enables real-time scoring)
* Model scoring: Typically <10ms
* **Total latency:** <20ms for complete fraud assessment

## 7. Performance Considerations

### 7.1. Incremental Updates

The union-find COMPONENT_PARENT structure supports efficient incremental updates:

* New events can be added without recomputing the entire graph
* Only SEQUENTIALLY_RELATED relationships to existing events need to be created
* The COMPONENT_PARENT structure is updated incrementally

### 7.2. Batch Processing Optimization

The GDS-based batching approach provides significant performance benefits:

* Connected components are identified efficiently using GDS WCC algorithm
* Transactions process 100 rows (connected components) at a time
* Multiple transactions execute concurrently for parallel workload distribution
* Random ordering (`ORDER BY rand()`) prevents bottlenecks

## 8. References

For deeper understanding of the techniques used in this solution:

* [Mastering Fraud Detection with Temporal Graphs](https://neo4j.com/blog/developer/mastering-fraud-detection-temporal-graph/) - Main conceptual foundation
* [Optimizing Weakly Connected Component Projections](https://neo4j.com/blog/developer/optimize-weakly-connected-component-projections/) - GDS projection optimization
* [WCC Batching to Avoid Query Crashes](https://neo4j.com/blog/developer/wcc-to-avoid-cypher-query-crashing/) - Batching strategy for large-scale processing

## 9. Conclusion

Real-time fraud detection with temporal graphs represents a powerful approach to combating sophisticated fraud schemes in financial services. By leveraging Neo4j's graph database capabilities, organizations can:

* Model complex event networks and patterns as connected components
* Compute component metrics (size, diameter, velocity) that capture network characteristics
* Extract temporally consistent features by examining pre-existing components an event connects to
* Deploy real-time fraud scoring with minimal latency (~2ms feature extraction)
* Ensure perfect consistency between training and production through proper temporal design

The combination of graph analysis, temporal reasoning, and machine learning provides a robust defense against evolving fraud tactics. The "bridging" patterns revealed by distinct_connected_components_count and aggregate component metrics offer powerful signals that traditional approaches miss, enabling organizations to detect sophisticated fraud schemes while maintaining high performance and temporal accuracy in the digital financial ecosystem.
