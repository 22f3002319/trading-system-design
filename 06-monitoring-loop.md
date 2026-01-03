# Monitoring Loop Design

## Overview

The monitoring loop is a **comprehensive automated trading system** that runs every 60 seconds for each user with active WebSocket connections. It synchronizes broker data, calculates PnL, updates orders, and manages order lifecycle automatically.

## Problem Statement

### Core Problems Solved

1. **Real-time Data Synchronization**: Keep internal database synchronized with broker APIs
2. **PnL Calculation**: Calculate accurate profit/loss using FIFO matching
3. **Order Lifecycle Management**: Automatically place, modify, and manage orders
4. **Error Detection**: Detect and report errors in real-time
5. **Data Consistency**: Ensure data consistency across multiple operations

### Requirements

- **Frequency**: Run every 60 seconds
- **Latency**: Complete cycle within 60 seconds
- **Reliability**: Handle failures gracefully
- **Accuracy**: Accurate PnL calculation and order updates
- **Automation**: Minimal manual intervention

## Monitoring Loop Architecture

### High-Level Flow

```
User WebSocket Connection
    ↓
Monitoring Task Started (per user)
    ↓
[Every 60 seconds]
    ↓
Pre-condition Checks
    ├── Time restriction (9:26 AM IST)
    ├── Active connections
    └── Active accounts
    ↓
Execute Monitoring Cycle (10 steps)
    ↓
Error Reporting (if errors occurred)
    ↓
Wait 60 seconds
    ↓
Repeat
```

### Execution Model

**Per-User Execution**:
- One monitoring task per user (not per connection)
- Task runs continuously while user has connections
- 60-second cycle interval
- Conditional execution based on dependencies

**Concurrent Execution**:
- Multiple users monitored concurrently
- Each user's cycle independent
- Database and API calls handled with connection pooling

## Step-by-Step Design

### Step 1: Concurrent API Calls

**Purpose**: Fetch real-time order book and position data from broker APIs

**Implementation**:
- **Concurrency**: 2N concurrent calls (N = number of accounts)
  - N calls for Order Book API
  - N calls for Positions API
- **Method**: `asyncio.gather()` with `return_exceptions=True`
- **Parallel Execution**: All API calls execute simultaneously

**Success Path**:
- All API calls succeed
- Data stored in `broker_order_book` and `broker_position` tables
- Proceed to Step 2

**Failure Path - 404 Errors**:
- **Detection**: "404" or "not found" in error message
- **Action**: 
  - Set `session_needed = True`
  - Log session expiry warning
  - Send error updates
  - **STOP EXECUTION** (return early, retry next cycle)

**Failure Path - Other Errors**:
- **Detection**: Network, timeout, or other exceptions
- **Action**:
  - Log error
  - Add to `errors_occurred` list
  - Send error updates
  - **STOP EXECUTION**

**Design Choices**:
- Concurrent execution for performance
- Exception handling per call
- Early termination on critical failures
- Session expiry detection

**Trade-offs**:
- ✅ Fast execution (parallel API calls)
- ✅ Efficient resource usage
- ❌ Complex error handling
- ❌ All-or-nothing on critical failures

### Step 2: Run Broker Order Book PnL Query

**Purpose**: Calculate Profit & Loss using FIFO (First-In-First-Out) matching algorithm

**Implementation**:
- **Execution**: Sequential for each account (not concurrent)
- **Query**: `get_broker_order_book_pnl_query(user_id, account_id)`
- **Operations**:
  1. TRUNCATE `IIFL_ORDER.BO_live_history` table
  2. INSERT calculated PnL data with FIFO matching

**FIFO Matching Algorithm**:
- Matches entry orders with exit orders chronologically
- Handles partial fills correctly
- Calculates matched quantities
- Computes PnL for matched and unmatched positions

**Success Path**:
- All account PnL queries succeed
- Set `pnl_success = True`
- Proceed to Step 3

**Failure Path**:
- Any account PnL query fails
- Set `pnl_success = False`
- Log error
- Add to `errors_occurred` list
- **SKIP Steps 3-9**
- Proceed directly to Step 10 (error reporting)

**Design Choices**:
- Sequential execution per account (query complexity)
- Transaction-based (TRUNCATE + INSERT)
- Early termination on failure (data consistency)
- FIFO algorithm for accurate PnL

**Trade-offs**:
- ✅ Accurate PnL calculation
- ✅ Handles complex scenarios
- ❌ Expensive query execution
- ❌ Sequential execution (slower)

### Step 3: Run SQL Update Script (CONDITIONAL)

**Condition**: ONLY EXECUTES IF `pnl_success == True` (Step 2 succeeded)

**Purpose**: Synchronize internal `orders` table with broker data, positions, and calculated PnL

**Implementation**:
- **Execution**: Sequential for each account
- **SQL Script**: `get_update_orders_sql_statements()`
- **Transaction**: All statements in single transaction
- **Rollback**: If any statement fails, entire transaction rolls back

**SQL Operations**:
1. START TRANSACTION
2. SET SQL_SAFE_UPDATES = 0
3. Main UPDATE statement (joins orders with BO_live_history, broker_order_book, broker_position, mainapp_fairprice)
4. SET SQL_SAFE_UPDATES = 1
5. COMMIT

**Success Path**:
- All SQL statements succeed
- Set `sql_success = True`
- Proceed to Step 4

**Failure Path**:
- Any SQL statement fails
- Set `sql_success = False`
- Transaction rolled back
- Log error
- Add to `errors_occurred` list
- **SKIP Step 4** (SL/TP placement)
- Proceed to Step 5

**Design Choices**:
- Transaction for data consistency
- Conditional execution based on PnL success
- Rollback on failure
- Continue to next steps even if SQL update fails (non-critical)

**Trade-offs**:
- ✅ Data consistency (transaction)
- ✅ Atomic updates
- ❌ Lock contention (transaction)
- ❌ Sequential execution

### Step 4: Place SL/TP Orders (CONDITIONAL)

**Condition**: ONLY EXECUTES IF `sql_success == True` (Step 3 succeeded)

**Purpose**: Automatically place Stop Loss (SL) and Take Profit (TP) orders for running positions

**Implementation**:
- **Service**: `enhanced_sl_tp_order_placer.process_user_running_orders(user_id)`
- **Logic**: Identifies running orders needing SL/TP, places orders via broker API
- **Execution**: Single call for all user accounts

**Success Path**:
- SL/TP orders placed successfully
- Proceed to Step 5

**Failure Path**:
- SL/TP processing fails
- Log error
- Add to `errors_occurred` list
- Proceed to Step 5 (non-critical failure)

**Design Choices**:
- Conditional execution (only if SQL update succeeded)
- Unified service for SL/TP placement
- Non-critical failure (continues to next steps)

**Trade-offs**:
- ✅ Automatic protective order placement
- ✅ Data consistency (only after SQL update)
- ❌ External dependency (broker API)
- ❌ Potential order placement failures

### Step 5: Modify Entry Orders

**Purpose**: Adjust entry orders when `buy_fair_price >= entry_stop_price`

**Implementation**:
- **Method**: `_modify_entry_orders_by_fair_price(user_id)`
- **Logic**: 
  - Find orders where `buy_fair_price >= entry_stop_price`
  - Modify entry orders via broker API
  - Update order status in database

**Success Path**:
- Entry orders modified successfully
- Proceed to Step 6

**Failure Path**:
- Modification fails
- Log error
- Add to `errors_occurred` list
- Proceed to Step 6 (non-critical failure)

**Design Choices**:
- Always executes if Step 2 succeeded (independent of Steps 3-4)
- Fair price-based modification
- Non-critical failure handling

**Trade-offs**:
- ✅ Dynamic order adjustment
- ✅ Responsive to market conditions
- ❌ External dependency (broker API)
- ❌ Potential modification failures

### Step 6: Modify SL Orders

**Purpose**: Adjust SL orders when `sell_fair_price <= sl_stop_price`

**Implementation**:
- **Method**: `_modify_sl_orders_by_fair_price(user_id)`
- **Logic**:
  - Find orders where `sell_fair_price <= sl_stop_price`
  - Modify SL orders via broker API
  - Update order status in database

**Success Path**:
- SL orders modified successfully
- Proceed to Step 7

**Failure Path**:
- Modification fails
- Log error
- Add to `errors_occurred` list
- Proceed to Step 7 (non-critical failure)

**Design Choices**:
- Always executes if Step 2 succeeded
- Fair price-based modification
- Non-critical failure handling

**Trade-offs**:
- ✅ Dynamic SL adjustment
- ✅ Risk management
- ❌ External dependency
- ❌ Potential failures

### Step 7: Delete Rejected Orders

**Purpose**: Remove rejected orders from system

**Implementation**:
- **Method**: `_delete_rejected_orders_for_user(user_id)`
- **Logic**: 
  - Find orders with rejected status
  - Delete from database
  - Log deletion count

**Success Path**:
- Rejected orders deleted
- Proceed to Step 8 (permission check)

**Failure Path**:
- Deletion fails
- Log error
- Add to `errors_occurred` list
- Proceed to Step 8 (non-critical failure)

**Design Choices**:
- Always executes if Step 2 succeeded
- Cleanup operation
- Non-critical failure handling

**Trade-offs**:
- ✅ System cleanup
- ✅ Database maintenance
- ❌ Potential data loss if incorrect deletion
- ❌ Requires careful status identification

### Step 8: Entry Regenerate (CONDITIONAL)

**Condition**: ONLY EXECUTES IF user has permission AND strategies configured

**Purpose**: Regenerate entry orders for rejected orders based on strategy rules

**Implementation**:
- **Permission Check**: `_get_entry_regenerate_permissions(user_id)`
  - Returns: `{enabled: bool, strategy_names: List[str]}`
- **If Enabled**:
  1. TRUNCATE `entry_regenerate` table
  2. For each allowed strategy:
     - Run `entry_regenerate` query
     - Insert results into `entry_regenerate` table
  3. Proceed to Step 9

**If Not Enabled**:
- Skip Steps 8-9
- Add to `errors_occurred`: "Entry_regenerate not enabled"
- Proceed to Step 10

**Success Path**:
- Entry regenerate data populated
- Proceed to Step 9

**Failure Path**:
- Regenerate fails
- Log error
- Add to `errors_occurred` list
- Skip Step 9
- Proceed to Step 10

**Design Choices**:
- Permission-based execution
- Strategy filtering
- Conditional execution
- Table truncation before insert

**Trade-offs**:
- ✅ Flexible permission system
- ✅ Strategy-specific regeneration
- ❌ Complex conditional logic
- ❌ Additional database operations

### Step 9: Place Rejected Orders (CONDITIONAL)

**Condition**: ONLY EXECUTES IF Step 8 executed (entry regenerate enabled and succeeded)

**Purpose**: Place orders from `entry_regenerate` table where `status IS NULL` and `entry_orderstatus = 'Rejected'`

**Implementation**:
- **Method**: `_place_rejected_orders_for_user(user_id)`
- **Logic**:
  1. Fetch orders from `entry_regenerate` table
  2. Filter: `status IS NULL` AND `entry_orderstatus = 'Rejected'`
  3. For each order:
     - Prepare order payload
     - Call `api/algo-place-batch-orders` for respective account
     - Update `entry_regenerate` table status

**Success Path**:
- Rejected orders placed
- Proceed to Step 10

**Failure Path**:
- Placement fails
- Log error
- Add to `errors_occurred` list
- Proceed to Step 10

**Design Choices**:
- Conditional execution (only if Step 8 executed)
- Batch order placement
- Status tracking in `entry_regenerate` table

**Trade-offs**:
- ✅ Automatic retry for rejected orders
- ✅ Efficient batch placement
- ❌ External dependency (broker API)
- ❌ Complex conditional logic

### Step 10: Send Error Updates (CONDITIONAL)

**Condition**: ONLY EXECUTES IF `errors_occurred` list is not empty

**Purpose**: Report all errors that occurred during the monitoring cycle

**Implementation**:
- **Method**: `_send_error_updates(user_id, errors_occurred)`
- **Message Type**: `monitoring_errors`
- **Content**: List of all error messages

**Design Choices**:
- Conditional execution (only if errors occurred)
- Error aggregation (all errors in one message)
- Single message per cycle

**Trade-offs**:
- ✅ Efficient error reporting
- ✅ User notification
- ❌ Only reports if errors occurred (no success confirmation)

## Conditional Execution Matrix

| Step | Condition | If Condition Fails |
|------|-----------|-------------------|
| 1 | Always | STOP, Step 10 |
| 2 | Step 1 succeeded | STOP, Step 10 |
| 3 | Step 2 succeeded | SKIP, Step 5 |
| 4 | Step 3 succeeded | SKIP, Step 5 |
| 5 | Step 2 succeeded | Continue, Step 6 |
| 6 | Step 2 succeeded | Continue, Step 7 |
| 7 | Step 2 succeeded | Continue, Step 8 |
| 8 | Permission + Step 2 | SKIP Step 9, Step 10 |
| 9 | Step 8 executed | Step 10 |
| 10 | Errors occurred | End |

## Performance Characteristics

### Execution Time

- **Step 1**: ~2-5 seconds (concurrent API calls)
- **Step 2**: ~5-15 seconds (complex PnL query per account)
- **Step 3**: ~1-3 seconds (SQL update per account)
- **Step 4**: ~2-5 seconds (SL/TP placement)
- **Steps 5-6**: ~1-2 seconds each (order modifications)
- **Step 7**: <1 second (deletion)
- **Step 8**: ~2-5 seconds (entry regenerate)
- **Step 9**: ~2-5 seconds (batch order placement)
- **Step 10**: <1 second (error reporting)

**Total Cycle Time**: ~20-50 seconds (depending on number of accounts and orders)

### Resource Usage

- **Database Connections**: Multiple queries per cycle
- **Broker API Calls**: 2N calls (Step 1) + additional calls for modifications
- **Memory**: Temporary data storage during execution
- **CPU**: Complex SQL queries and data processing

## Error Handling Strategy

### Error Classification

1. **Critical Errors**: Step 1-2 failures → STOP execution
2. **Non-Critical Errors**: Step 3-9 failures → Continue execution
3. **Permission Errors**: Step 8 permission check → Skip Steps 8-9

### Error Recovery

- **Automatic Retry**: Next cycle retries failed operations
- **Error Reporting**: Errors sent to user via WebSocket
- **Logging**: All errors logged for debugging
- **Graceful Degradation**: System continues operating despite errors

## Failure Scenarios

### Scenario 1: Broker API Unavailable
- **Step**: Step 1 (API calls)
- **Handling**: STOP execution, error reported, retry next cycle
- **Impact**: No data synchronization, no PnL calculation

### Scenario 2: Database Query Failure
- **Step**: Step 2 (PnL query)
- **Handling**: STOP execution, error reported, retry next cycle
- **Impact**: No PnL calculation, no order updates

### Scenario 3: SQL Update Failure
- **Step**: Step 3 (SQL update)
- **Handling**: SKIP Step 4, continue to Step 5, error reported
- **Impact**: Orders not updated, SL/TP not placed

### Scenario 4: Session Expiry
- **Step**: Step 1 (API calls)
- **Handling**: STOP execution, session refresh needed, retry next cycle
- **Impact**: No data synchronization until session refreshed

---

**Next**: [Sequence Diagrams](07-sequence-diagrams.md)

