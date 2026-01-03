# Database Design

## Overview

The database design follows a **relational database model** using MySQL 8.0+, with normalized schemas, foreign key constraints, and optimized indexes for performance-critical queries.

## Design Principles

### 1. Normalization
- **3NF (Third Normal Form)**: Minimized data redundancy
- **Foreign Key Constraints**: Referential integrity enforcement
- **Unique Constraints**: Data uniqueness guarantees

### 2. Performance Optimization
- **Strategic Indexing**: Indexes on frequently queried columns
- **Composite Indexes**: Multi-column indexes for complex queries
- **Query Optimization**: Indexed columns in WHERE and JOIN clauses

### 3. Data Integrity
- **ACID Transactions**: For critical operations
- **Foreign Keys**: Cascade rules for data consistency
- **NOT NULL Constraints**: Required field enforcement
- **Data Types**: Appropriate types for performance and accuracy

## Entity-Relationship Model

```
Users (1) ──< (N) IIFL_Accounts
Users (1) ──< (N) Sessions
Users (1) ──< (N) Orders
Users (1) ──< (N) Algo_Settings
Users (1) ──< (N) User_Scheduler_Settings
Users (1) ──< (N) Entry_Regenerate_Permission

IIFL_Accounts (1) ──< (N) Active_Sessions
IIFL_Accounts (1) ──< (N) Orders
IIFL_Accounts (1) ──< (N) Broker_Order_Book
IIFL_Accounts (1) ──< (N) Broker_Position

Sessions (1) ──< (N) Active_Sessions

Orders (N) ──> (1) Mainapp_Instruments (via exchange_token)
```

## Core Tables

### 1. Users Table

**Purpose**: Store user authentication and profile information

**Schema**:
```sql
users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
)
```

**Design Choices**:
- **Password Hash**: Stored as hash, not plaintext
- **Unique Constraints**: Username and email must be unique
- **Soft Delete**: `is_active` flag instead of hard delete
- **Timestamps**: Auto-managed created/updated timestamps

**Indexes**:
- Primary Key: `id`
- Unique: `username`
- Unique: `email`

**Trade-offs**:
- ✅ Normalized user data
- ✅ Unique constraints prevent duplicates
- ❌ Email uniqueness might be restrictive
- ❌ No user profile extensions

### 2. IIFL_Accounts Table

**Purpose**: Store broker account credentials and configuration per user

**Schema**:
```sql
iifl_accounts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    account_name VARCHAR(255) NOT NULL,
    broker ENUM('IIFL', 'WISDOM_CAPITAL', 'NIRMAL_BANG') DEFAULT 'IIFL',
    app_key TEXT,
    secret_key TEXT,
    base_url TEXT,
    client_id VARCHAR(255),
    userID VARCHAR(255),
    password VARCHAR(255),
    totp_base_url VARCHAR(500),
    totp_secret_key VARCHAR(255),
    cash_available DECIMAL(15,2) DEFAULT 0.00,
    margin_utilized DECIMAL(15,2) DEFAULT 0.00,
    net_margin DECIMAL(15,2) DEFAULT 0.00,
    total_mtm DECIMAL(15,2) DEFAULT 0.00,
    unrealized_mtm DECIMAL(15,2) DEFAULT 0.00,
    realized_mtm DECIMAL(15,2) DEFAULT 0.00,
    is_active BOOLEAN DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
)
```

**Design Choices**:
- **Multiple Accounts per User**: One-to-many relationship
- **Broker Enumeration**: Support for multiple brokers
- **Encrypted Credentials**: TEXT fields for sensitive data (encryption at application level)
- **Financial Fields**: DECIMAL for accurate monetary values
- **Soft Delete**: `is_active` flag

**Indexes**:
- Primary Key: `id`
- Foreign Key: `user_id`
- Index: `(user_id, is_active)` for active account queries

**Trade-offs**:
- ✅ Multi-account support
- ✅ Flexible broker support
- ❌ Credentials stored in database (requires encryption)
- ❌ Financial data might be stale (updated periodically)

### 3. Orders Table

**Purpose**: Store all order information with comprehensive lifecycle tracking

**Schema** (Key Fields):
```sql
orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    account_id INT NOT NULL,
    order_type VARCHAR(50) NOT NULL,
    symbol VARCHAR(255) NOT NULL,
    exchange_instrument_id VARCHAR(255),
    
    -- Entry Order Fields
    entry_price DECIMAL(15,4),
    entry_quantity INT,
    entry_stop_price DECIMAL(15,4),
    entry_order_status VARCHAR(50),
    entry_broker_order_id VARCHAR(255),
    
    -- Stop Loss Fields
    sl_price DECIMAL(15,4),
    sl_quantity INT,
    sl_stop_price DECIMAL(15,4),
    sl_order_status VARCHAR(50),
    sl_broker_order_id VARCHAR(255),
    
    -- Take Profit Fields
    tp_price DECIMAL(15,4),
    tp_quantity INT,
    tp_order_status VARCHAR(50),
    
    -- PnL and Position Fields
    pnl DECIMAL(15,4),
    open_position DECIMAL(15,4),
    LTP DECIMAL(15,4),
    margin DECIMAL(15,4),
    
    -- Fair Price Fields
    buy_fair_price DECIMAL(15,4),
    sell_fair_price DECIMAL(15,4),
    avg_fair_price DECIMAL(15,4),
    
    order_status VARCHAR(50) NOT NULL,
    order_created_at TIMESTAMP NULL,
    order_updated_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (account_id) REFERENCES iifl_accounts(id)
)
```

**Design Choices**:
- **Comprehensive Order Tracking**: All order types in single table
- **Denormalized Fields**: Entry, SL, TP fields for quick access
- **DECIMAL Precision**: 15,4 for accurate price representation
- **Status Fields**: Separate status for entry, SL, TP orders
- **Broker Order IDs**: Store broker-specific order identifiers
- **Timestamps**: Created/updated tracking for audit

**Indexes**:
- Primary Key: `id`
- Foreign Keys: `user_id`, `account_id`
- Composite: `(user_id, account_id)` for user-account queries
- Index: `exchange_instrument_id` for instrument-based queries
- Index: `order_status` for status filtering
- Index: `order_created_at` for time-based queries

**Trade-offs**:
- ✅ Single table for all orders (simpler queries)
- ✅ Comprehensive order lifecycle tracking
- ❌ Wide table (many columns)
- ❌ Potential NULL values for unused order types

### 4. Sessions Table

**Purpose**: Store user session tokens and expiration

**Schema**:
```sql
sessions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    session_token VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
)
```

**Design Choices**:
- **Unique Tokens**: Each session token is unique
- **Expiration Tracking**: `expires_at` for automatic cleanup
- **User Relationship**: Links to user for session management

**Indexes**:
- Primary Key: `id`
- Unique: `session_token` (for fast lookups)
- Foreign Key: `user_id`
- Index: `expires_at` for cleanup queries

**Trade-offs**:
- ✅ Simple session storage
- ✅ Fast token lookup
- ❌ Requires cleanup job for expired sessions
- ❌ No session metadata storage

### 5. Active_Sessions Table

**Purpose**: Store active broker API sessions per account

**Schema**:
```sql
active_sessions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    session_token VARCHAR(255) NOT NULL,
    account_id INT NOT NULL,
    client_id VARCHAR(255) NOT NULL,
    access_token TEXT,
    base_url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (session_token) REFERENCES sessions(session_token),
    FOREIGN KEY (account_id) REFERENCES iifl_accounts(id)
)
```

**Design Choices**:
- **Multi-Reference**: Links to users, sessions, and accounts
- **Access Token Storage**: TEXT field for broker API tokens
- **Account-Specific**: One session per user-account combination

**Indexes**:
- Primary Key: `id`
- Composite Unique: `(user_id, account_id)` - one active session per account
- Foreign Keys: `user_id`, `session_token`, `account_id`

**Trade-offs**:
- ✅ Efficient session reuse
- ✅ Account-level session isolation
- ❌ Complex foreign key relationships
- ❌ Requires cleanup for expired sessions

### 6. Broker_Order_Book Table

**Purpose**: Store real-time order book data from broker APIs

**Schema** (Key Fields):
```sql
broker_order_book (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    account_id INT NOT NULL,
    ExchangeInstrumentID VARCHAR(255),
    TradingSymbol VARCHAR(255),
    OrderAverageTradedPrice DECIMAL(15,4),
    OrderQuantity INT,
    ExchangeTransactTime DATETIME,
    OrderSide VARCHAR(50),
    OrderStatus VARCHAR(50),
    OrderUniqueIdentifier VARCHAR(255),
    -- ... additional broker fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

**Design Choices**:
- **Snapshot Storage**: Stores current state from broker APIs
- **Broker Schema**: Follows broker API field names
- **High Write Volume**: Optimized for frequent updates
- **Time-Based Queries**: Indexed on `ExchangeTransactTime`

**Indexes**:
- Primary Key: `id`
- Composite: `(user_id, account_id)` for user-account queries
- Composite: `(ExchangeInstrumentID, ExchangeTransactTime)` for PnL queries
- Index: `OrderUniqueIdentifier` for order lookup
- Index: `ExchangeTransactTime` for time-based queries

**Trade-offs**:
- ✅ Real-time broker data
- ✅ Supports complex PnL queries
- ❌ High write volume (performance impact)
- ❌ Potential data duplication

### 7. Broker_Position Table

**Purpose**: Store current positions from broker APIs

**Schema** (Key Fields):
```sql
broker_position (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    account_id INT NOT NULL,
    ExchangeInstrumentID VARCHAR(255),
    TradingSymbol VARCHAR(255),
    NetQty DECIMAL(15,4),
    BuyQty DECIMAL(15,4),
    SellQty DECIMAL(15,4),
    -- ... additional position fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
)
```

**Design Choices**:
- **Real-time Positions**: Updated from broker API
- **Net Quantity**: Stores net position per instrument
- **Buy/Sell Quantities**: Separate fields for long/short tracking

**Indexes**:
- Primary Key: `id`
- Composite: `(user_id, account_id, ExchangeInstrumentID)` for position lookups
- Foreign Keys: `user_id`, `account_id`

**Trade-offs**:
- ✅ Real-time position data
- ✅ Fast position queries
- ❌ Requires frequent updates
- ❌ Potential stale data between updates

### 8. BO_live_history Table

**Purpose**: Store calculated PnL data using FIFO matching algorithm

**Schema** (Key Fields):
```sql
BO_live_history (
    ExchangeInstrumentID VARCHAR(255),
    Entry_Order_id VARCHAR(255),
    Entry_datetime DATETIME,
    BO_Entry_Quantity DECIMAL(15,4),
    BO_entry_price DECIMAL(15,4),
    Sell_Order_id VARCHAR(255),
    SL_ExchangeTransactTime DATETIME,
    BO_SL_price DECIMAL(15,4),
    BO_SL_Quantity DECIMAL(15,4),
    BO_PNL DECIMAL(15,4),
    buy_fair_price DECIMAL(15,4),
    sell_fair_price DECIMAL(15,4),
    avg_fair_price DECIMAL(15,4),
    -- ... additional fields
)
```

**Design Choices**:
- **Calculated Data**: Populated by PnL query (not directly inserted)
- **FIFO Matching**: Stores matched entry-exit pairs
- **Truncate and Recalculate**: Table truncated before each PnL calculation
- **Fair Prices**: Includes current fair prices for unrealized PnL

**Indexes**:
- Composite: `(ExchangeInstrumentID, BO_Entry_date)` for efficient queries
- Index: `Entry_datetime` for chronological queries

**Trade-offs**:
- ✅ Accurate PnL calculation
- ✅ Fast queries after calculation
- ❌ Table truncated frequently (no history)
- ❌ Complex calculation query

## Data Flow Patterns

### Write Patterns

1. **Order Creation**
   ```
   User Request → Orders Table (INSERT)
   ```

2. **Broker Data Sync**
   ```
   Broker API → Broker_Order_Book (INSERT/UPDATE)
   Broker API → Broker_Position (INSERT/UPDATE)
   ```

3. **PnL Calculation**
   ```
   Broker_Order_Book → PnL Query → BO_live_history (TRUNCATE + INSERT)
   ```

4. **Order Updates**
   ```
   BO_live_history + Broker data → Orders Table (UPDATE)
   ```

### Read Patterns

1. **User Orders**
   ```sql
   SELECT * FROM orders WHERE user_id = ? AND account_id = ?
   ```

2. **PnL Calculation**
   ```sql
   -- Complex CTE query joining broker_order_book, broker_order_book_snapshot
   -- Calculates FIFO-matched PnL
   ```

3. **Position Queries**
   ```sql
   SELECT * FROM broker_position WHERE user_id = ? AND account_id = ?
   ```

4. **Session Lookup**
   ```sql
   SELECT * FROM active_sessions WHERE user_id = ? AND account_id = ?
   ```

## Indexing Strategy

### Primary Indexes
- All tables have `id` as primary key (AUTO_INCREMENT)
- Ensures unique row identification
- Supports foreign key relationships

### Foreign Key Indexes
- All foreign keys automatically indexed
- Supports JOIN operations
- Ensures referential integrity

### Composite Indexes
- `(user_id, account_id)` on orders, broker_order_book, broker_position
- Supports user-account queries (most common pattern)
- Covers multiple query patterns

### Query-Specific Indexes
- `ExchangeTransactTime` for time-based queries
- `OrderStatus` for status filtering
- `ExchangeInstrumentID` for instrument-based queries
- `session_token` for fast session lookup

### Index Trade-offs

**Benefits**:
- ✅ Faster query execution
- ✅ Efficient JOIN operations
- ✅ Reduced full table scans

**Costs**:
- ❌ Additional storage space
- ❌ Slower INSERT/UPDATE operations
- ❌ Index maintenance overhead

## Transaction Management

### Transaction Usage

1. **Order Placement**
   - BEGIN TRANSACTION
   - INSERT order
   - Update related tables
   - COMMIT

2. **PnL Calculation**
   - BEGIN TRANSACTION
   - TRUNCATE BO_live_history
   - INSERT calculated PnL
   - COMMIT (or ROLLBACK on error)

3. **Order Updates**
   - BEGIN TRANSACTION
   - UPDATE orders (multiple rows)
   - COMMIT (or ROLLBACK on error)

### Transaction Isolation

- **Default**: REPEATABLE READ (MySQL default)
- **Critical Operations**: Explicit transaction control
- **Long-running Queries**: Minimized to reduce lock contention

## Data Consistency Strategies

### 1. Foreign Key Constraints
- Enforces referential integrity
- Prevents orphaned records
- Cascade rules for deletions (if configured)

### 2. Unique Constraints
- Prevents duplicate data
- Enforces business rules (e.g., one active session per account)

### 3. NOT NULL Constraints
- Ensures required fields are populated
- Prevents incomplete data

### 4. Transactions
- Atomic operations
- All-or-nothing updates
- Rollback on errors

## Performance Considerations

### Query Optimization

1. **Index Usage**: Queries use indexes wherever possible
2. **JOIN Optimization**: Proper indexing on JOIN columns
3. **Query Patterns**: Common queries optimized with composite indexes
4. **EXPLAIN Analysis**: Query plans analyzed for optimization

### Scaling Challenges

1. **Write-Heavy Tables**: `broker_order_book` has high write volume
2. **Complex Queries**: PnL calculation query is computationally expensive
3. **Table Growth**: Orders table grows continuously
4. **Lock Contention**: Transactions can cause lock contention

### Optimization Strategies

1. **Partitioning**: Consider partitioning `broker_order_book` by date
2. **Read Replicas**: Use read replicas for reporting queries
3. **Caching**: Cache frequently accessed data (sessions, settings)
4. **Archive Strategy**: Archive old orders to reduce table size

## Failure Scenarios

### Database Connection Failure
- **Impact**: All operations fail
- **Mitigation**: Connection pooling, retry logic, graceful degradation

### Transaction Deadlock
- **Impact**: Transaction rollback, operation failure
- **Mitigation**: Retry logic, shorter transactions, proper indexing

### Query Timeout
- **Impact**: Long-running query fails
- **Mitigation**: Query optimization, timeout settings, query cancellation

### Data Corruption
- **Impact**: Inconsistent data
- **Mitigation**: Foreign key constraints, transactions, backup strategies

---

**Next**: [API Design](04-api-design.md)

