# WebSocket Design

## Overview

The WebSocket design enables **real-time bidirectional communication** between clients and the server for monitoring order status, receiving updates, and automated trading operations.

## Design Goals

1. **Real-time Updates**: Sub-60-second latency for monitoring updates
2. **Scalability**: Support 1000+ concurrent connections
3. **Reliability**: Handle connection failures gracefully
4. **Efficiency**: Minimize resource usage per connection
5. **User Isolation**: Per-user connection management

## WebSocket Architecture

### Connection Model

```
Client (Browser)
    ↓
WebSocket Connection (/ws/{user_id})
    ↓
WebSocketManager
    ├── Connection Management
    ├── User Association
    └── Monitoring Loop Orchestration
        ↓
    Monitoring Service (per user)
        ↓
    60-second Monitoring Cycle
        ↓
    Real-time Updates (WebSocket)
```

### Connection Lifecycle

```
1. Client initiates connection → /ws/{user_id}
2. Server validates user_id and authentication
3. Server checks time restrictions (9:26 AM IST)
4. Connection accepted/rejected
5. Connection ID generated (UUID)
6. Connection stored in active_connections
7. User association created (user_connections)
8. Monitoring loop started (if first connection for user)
9. Connection maintained
10. Updates sent asynchronously
11. Connection closed (client/server)
12. Cleanup (connection removed, monitoring stopped if last connection)
```

## Connection Management

### WebSocketManager Structure

**Core Data Structures**:
```python
active_connections: Dict[str, WebSocket]  # connection_id -> websocket
user_connections: Dict[str, Set[str]]     # user_id -> set of connection_ids
monitoring_tasks: Dict[str, Task]         # user_id -> monitoring task
user_updates: Dict[str, List[Dict]]       # user_id -> list of updates
```

**Design Choices**:
- **UUID Connection IDs**: Unique identifier per connection
- **Multiple Connections per User**: User can have multiple browser tabs
- **Per-User Monitoring**: One monitoring loop per user (not per connection)
- **Update History**: Track sent updates (for debugging/audit)

**Trade-offs**:
- ✅ Efficient connection tracking
- ✅ Multiple connections per user supported
- ✅ Single monitoring loop per user (resource efficient)
- ❌ Connection state management complexity
- ❌ Memory overhead for connection storage

### Connection Methods

#### connect(websocket, user_id)
**Purpose**: Establish new WebSocket connection

**Steps**:
1. Accept WebSocket connection
2. Generate connection ID (UUID)
3. Store connection in `active_connections`
4. Add connection to `user_connections[user_id]`
5. Start monitoring task (if first connection for user)
6. Initialize `user_updates[user_id]`
7. Send welcome message
8. Return connection_id

**Design Choices**:
- Connection accepted before validation (FastAPI handles this)
- Monitoring started on first connection
- Welcome message for connection confirmation

#### disconnect(connection_id, user_id)
**Purpose**: Clean up WebSocket connection

**Steps**:
1. Close WebSocket (if still open)
2. Remove from `active_connections`
3. Remove from `user_connections[user_id]`
4. If last connection for user:
   - Cancel monitoring task
   - Remove from `monitoring_tasks`
   - Clean up `user_updates`
5. Log disconnect

**Design Choices**:
- Graceful closure attempt
- Monitoring stopped only when last connection closes
- Cleanup of all user-related data

#### send_personal_message(message, user_id)
**Purpose**: Send message to all connections of a user

**Steps**:
1. Get all connection IDs for user
2. For each connection:
   - Get WebSocket from `active_connections`
   - Send message (handle exceptions)
   - Remove dead connections
3. Track sent messages in `user_updates`

**Design Choices**:
- Broadcast to all user connections
- Exception handling per connection
- Dead connection cleanup
- Message tracking

## Monitoring Loop Design

### Per-User Monitoring

**Architecture**:
- One monitoring task per user (not per connection)
- Task runs continuously while user has active connections
- 60-second cycle interval
- Conditional execution based on time restrictions

### Monitoring Loop Flow

```
while True:
    await asyncio.sleep(60)  # Wait 60 seconds
    
    # Pre-conditions
    if not _is_websocket_time_allowed():
        # Before 9:26 AM IST
        stop_monitoring()
        disconnect_all_connections()
        break
    
    if user has no active connections:
        break  # Stop monitoring
    
    if user has no active accounts:
        continue  # Skip cycle, try next iteration
    
    # Execute monitoring cycle
    _execute_concurrent_monitoring_for_user(user_id, accounts)
```

### Time Restrictions

**Requirement**: WebSocket connections only allowed after 9:26 AM IST

**Implementation**:
- `_is_websocket_time_allowed()`: Checks current IST time
- Before 9:26 AM: Connection rejected or monitoring stopped
- After 9:26 AM: Normal operation

**Design Choices**:
- Timezone: Asia/Kolkata (IST)
- Time check: Before each monitoring cycle
- Action: Stop monitoring and disconnect if before allowed time

**Trade-offs**:
- ✅ Enforces trading hours
- ✅ Prevents unnecessary operations
- ❌ Connections closed during restricted hours
- ❌ Client needs to reconnect after 9:26 AM

## Monitoring Cycle Steps

### Step-by-Step Execution

The monitoring cycle executes 10 steps sequentially with conditional logic:

1. **Concurrent API Calls**: Fetch order book and positions (2N calls)
2. **PnL Calculation**: Run broker_order_book_pnl query
3. **SQL Update** (conditional): Update orders table (only if PnL succeeded)
4. **SL/TP Placement** (conditional): Place stop-loss/take-profit orders (only if SQL update succeeded)
5. **Entry Order Modification**: Modify entry orders based on buy_fair_price
6. **SL Order Modification**: Modify SL orders based on sell_fair_price
7. **Delete Rejected Orders**: Remove rejected orders
8. **Entry Regenerate** (conditional): Regenerate entry orders (only if permitted)
9. **Place Rejected Orders** (conditional): Place regenerated rejected orders
10. **Error Reporting**: Send error updates (only if errors occurred)

### Conditional Execution Logic

```
Step 1 (API Calls)
    ↓ (if failed → Step 10: Error Reporting, STOP)
Step 2 (PnL Calculation)
    ↓ (if failed → Step 10: Error Reporting, STOP)
Step 3 (SQL Update) [only if Step 2 succeeded]
    ↓ (if failed → Steps 5-7 continue, Step 4 skipped)
Step 4 (SL/TP Placement) [only if Step 3 succeeded]
    ↓
Step 5 (Entry Modification) [always executes if Step 2 succeeded]
    ↓
Step 6 (SL Modification) [always executes if Step 2 succeeded]
    ↓
Step 7 (Delete Rejected) [always executes if Step 2 succeeded]
    ↓
Step 8 (Entry Regenerate) [only if permitted and Step 2 succeeded]
    ↓ (if not permitted → Step 10, STOP)
Step 9 (Place Rejected) [only if Step 8 executed]
    ↓
Step 10 (Error Reporting) [only if errors occurred]
```

**Design Choices**:
- Sequential execution with dependencies
- Conditional execution based on previous step success
- Error collection throughout cycle
- Final error reporting if any errors occurred

**Trade-offs**:
- ✅ Data consistency (dependencies respected)
- ✅ Clear error handling
- ❌ Sequential execution (slower than parallel)
- ❌ Complex conditional logic

## Message Types

### Connection Established

```json
{
    "type": "connection_established",
    "connection_id": "uuid",
    "user_id": "integer",
    "timestamp": "iso_datetime"
}
```

**Purpose**: Confirm successful WebSocket connection

### Monitoring Errors

```json
{
    "type": "monitoring_errors",
    "errors": [
        "Error message 1",
        "Error message 2"
    ],
    "timestamp": "iso_datetime"
}
```

**Purpose**: Report errors from monitoring cycle

**Design Choices**:
- Sent only if errors occurred
- Multiple errors aggregated
- Timestamp for debugging

### Order Updates (Future)

```json
{
    "type": "order_update",
    "order_id": "integer",
    "status": "string",
    "details": {...}
}
```

**Purpose**: Real-time order status updates (future enhancement)

## Error Handling

### Connection Errors

**Error Scenarios**:
1. **Connection Loss**: Network interruption
2. **Server Error**: Application crash
3. **Timeout**: Inactive connection
4. **Invalid User**: Authentication failure

**Handling**:
- Dead connections detected during send
- Automatic cleanup on send failure
- Monitoring continues for other connections
- Client responsible for reconnection

### Monitoring Errors

**Error Scenarios**:
1. **API Call Failure**: Broker API unavailable
2. **Database Error**: Query failure
3. **Session Expiry**: Broker session expired
4. **Validation Error**: Data validation failure

**Handling**:
- Errors collected in `errors_occurred` list
- Conditional execution based on error type
- Error messages sent via WebSocket
- Monitoring continues for next cycle (unless critical failure)

## Performance Considerations

### Resource Usage

**Per-User Resources**:
- 1 monitoring task (asyncio.Task)
- N WebSocket connections (N = number of browser tabs)
- 60-second cycle overhead
- Memory for connection storage

**Scaling Challenges**:
- 1000 users = 1000 monitoring tasks
- Each task runs every 60 seconds
- Database queries per cycle
- Broker API calls per cycle

### Optimization Strategies

1. **Single Monitoring Loop per User**: Not per connection
2. **Concurrent API Calls**: 2N calls in parallel (order book + positions)
3. **Conditional Execution**: Skip steps if dependencies fail
4. **Error Aggregation**: Collect errors, report once
5. **Connection Pooling**: Reuse broker sessions

## Scalability Considerations

### Current Limitations

1. **Single Server Instance**: All connections on one server
2. **In-Memory State**: Connection state not shared across servers
3. **Database Load**: Each monitoring cycle queries database
4. **Broker API Limits**: Rate limits per account

### Scaling Strategies

1. **Horizontal Scaling**: Multiple server instances
   - Challenge: Connection state distribution
   - Solution: Sticky sessions or shared state (Redis)

2. **Connection Distribution**: Load balance WebSocket connections
   - Challenge: User connections on different servers
   - Solution: User affinity or shared monitoring service

3. **Monitoring Service Separation**: Separate monitoring from WebSocket server
   - Challenge: Communication between services
   - Solution: Message queue or shared database

4. **Database Optimization**: Read replicas, query optimization
   - Challenge: Write-heavy operations
   - Solution: Read replicas for queries, master for writes

## Security Considerations

### Connection Security

1. **WSS (WebSocket Secure)**: Encrypted connection
2. **User ID Validation**: Path parameter validated
3. **Authentication**: Token validation (potential enhancement)
4. **Time Restrictions**: Trading hours enforcement

### Message Security

1. **Input Validation**: Validate all incoming messages (future)
2. **Output Sanitization**: Sanitize data before sending
3. **Rate Limiting**: Limit message frequency (future)

## Failure Scenarios

### Connection Failures

**Scenario**: Network interruption
- **Detection**: Send failure, connection state
- **Handling**: Cleanup connection, continue for other connections
- **Recovery**: Client reconnects

### Monitoring Loop Failures

**Scenario**: Monitoring task crashes
- **Detection**: Task cancellation, exception
- **Handling**: Task cleanup, user notified (if possible)
- **Recovery**: New connection starts new monitoring loop

### Broker API Failures

**Scenario**: Broker API unavailable
- **Detection**: API call failure
- **Handling**: Error collected, cycle continues
- **Recovery**: Retry in next cycle

### Database Failures

**Scenario**: Database connection lost
- **Detection**: Query exception
- **Handling**: Error collected, cycle stops at failed step
- **Recovery**: Retry in next cycle

---

**Next**: [Monitoring Loop Design](06-monitoring-loop.md)

