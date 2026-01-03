# API Design

## Overview

The system provides both **REST API** endpoints for standard HTTP operations and **WebSocket** connections for real-time bidirectional communication.

## API Design Principles

### 1. RESTful Design
- Resource-based URLs
- HTTP methods for operations (GET, POST, PUT, DELETE)
- Standard HTTP status codes
- JSON request/response format

### 2. Authentication
- Session-based authentication
- Token in request headers
- Token validation middleware
- Session expiration handling

### 3. Error Handling
- Consistent error response format
- Meaningful error messages
- Appropriate HTTP status codes
- Error logging and tracking

### 4. Rate Limiting
- Per-user rate limits
- Different limits for different endpoints
- Rate limit headers in responses
- Graceful rate limit errors

## REST API Endpoints

### Authentication Endpoints

#### POST /api/login
**Purpose**: User authentication and session creation

**Request**:
```json
{
    "username": "string",
    "password": "string"
}
```

**Response (Success - 200)**:
```json
{
    "session_token": "string",
    "user": {
        "id": "integer",
        "username": "string",
        "email": "string"
    },
    "expires_at": "timestamp"
}
```

**Response (Error - 401)**:
```json
{
    "error": "Invalid credentials"
}
```

**Design Choices**:
- Session token generation on successful login
- Token stored in database and cache
- Token expiration tracked

**Trade-offs**:
- ✅ Server-side session control
- ✅ Easy session revocation
- ❌ Database access for validation
- ❌ Not stateless (scaling challenges)

#### POST /api/logout
**Purpose**: User logout and session termination

**Request Headers**:
```
Authorization: Bearer <session_token>
```

**Response (Success - 200)**:
```json
{
    "message": "Logged out successfully"
}
```

**Design Choices**:
- Session token invalidated
- Cache cleared
- Database updated

### Account Management Endpoints

#### GET /api/accounts
**Purpose**: Get user's trading accounts

**Request Headers**:
```
Authorization: Bearer <session_token>
```

**Response (Success - 200)**:
```json
{
    "accounts": [
        {
            "id": "integer",
            "account_name": "string",
            "broker": "string",
            "cash_available": "decimal",
            "margin_utilized": "decimal",
            "net_margin": "decimal",
            "is_active": "boolean"
        }
    ]
}
```

**Design Choices**:
- Returns only user's accounts
- Filtered by user_id from session
- Sensitive data (credentials) excluded

**Trade-offs**:
- ✅ Secure data access
- ✅ User isolation
- ❌ Requires authentication for every request

#### POST /api/accounts
**Purpose**: Add new trading account

**Request**:
```json
{
    "account_name": "string",
    "broker": "string",
    "app_key": "string",
    "secret_key": "string",
    "client_id": "string",
    "userID": "string",
    "password": "string"
}
```

**Response (Success - 201)**:
```json
{
    "account": {
        "id": "integer",
        "account_name": "string",
        "broker": "string"
    }
}
```

**Design Choices**:
- Credentials stored encrypted
- Account linked to authenticated user
- Validation before storage

### Order Management Endpoints

#### POST /api/orders
**Purpose**: Place a new order

**Request**:
```json
{
    "account_id": "integer",
    "symbol": "string",
    "exchange_segment": "string",
    "product_type": "string",
    "order_side": "BUY|SELL",
    "order_quantity": "integer",
    "limit_price": "decimal",
    "stop_price": "decimal",
    "order_type": "string"
}
```

**Response (Success - 201)**:
```json
{
    "order": {
        "id": "integer",
        "order_status": "string",
        "iifl_order_id": "string",
        "symbol": "string"
    }
}
```

**Response (Error - 400)**:
```json
{
    "error": "Validation error",
    "details": {...}
}
```

**Design Choices**:
- Validation before broker API call
- Session validation for account
- Transaction for order creation
- Broker API call for order placement

**Trade-offs**:
- ✅ Validation prevents invalid orders
- ✅ Transaction ensures data consistency
- ❌ Multiple operations (slower)
- ❌ External dependency (broker API)

#### POST /api/algo-orders
**Purpose**: Place algorithmic order with SL/TP

**Request**:
```json
{
    "account_id": "integer",
    "trading_symbol": "string",
    "exchange_token": "string",
    "no_of_lots": "integer",
    "entry_price": "decimal",
    "entry_stop_price": "decimal",
    "sl_price": "decimal",
    "sl_stop_price": "decimal",
    "tp_price": "decimal",
    "strategy_name": "string"
}
```

**Response**: Similar to `/api/orders`

**Design Choices**:
- Multiple orders created (entry, SL, TP)
- Order relationships tracked
- Unified order processing service

#### POST /api/algo-place-batch-orders
**Purpose**: Place multiple orders in batch

**Request**:
```json
{
    "account_id": "integer",
    "orders": [
        {
            "external_id": "string",
            "trading_symbol": "string",
            "no_of_lots": "integer",
            "entry_price": "decimal",
            "order_side": "BUY|SELL",
            "sl_price": "decimal",
            "targets": [...]
        }
    ]
}
```

**Response (Success - 201)**:
```json
{
    "results": [
        {
            "external_id": "string",
            "status": "success|error",
            "order_id": "integer",
            "error": "string (if error)"
        }
    ]
}
```

**Design Choices**:
- Batch processing for efficiency
- Partial success handling
- Individual order error tracking

**Trade-offs**:
- ✅ Efficient for multiple orders
- ✅ Reduces API calls
- ❌ Complex error handling
- ❌ Partial failures possible

#### GET /api/orders
**Purpose**: Get user's orders

**Query Parameters**:
- `account_id` (optional): Filter by account
- `status` (optional): Filter by order status
- `limit` (optional): Limit results
- `offset` (optional): Pagination offset

**Response (Success - 200)**:
```json
{
    "orders": [
        {
            "id": "integer",
            "symbol": "string",
            "order_status": "string",
            "entry_order_status": "string",
            "sl_order_status": "string",
            "pnl": "decimal",
            "open_position": "decimal"
        }
    ],
    "total": "integer",
    "limit": "integer",
    "offset": "integer"
}
```

**Design Choices**:
- Filtered by authenticated user
- Pagination support
- Filtering options

### PnL and Reporting Endpoints

#### GET /api/broker-order-book-pnl
**Purpose**: Get calculated PnL data

**Query Parameters**:
- `account_id` (required): Account ID

**Response (Success - 200)**:
```json
{
    "pnl_data": [
        {
            "ExchangeInstrumentID": "string",
            "Entry_Order_id": "string",
            "BO_Entry_Quantity": "decimal",
            "BO_entry_price": "decimal",
            "BO_SL_price": "decimal",
            "BO_PNL": "decimal",
            "buy_fair_price": "decimal",
            "sell_fair_price": "decimal"
        }
    ]
}
```

**Design Choices**:
- Executes PnL calculation query
- Returns calculated results
- Account-specific data

**Trade-offs**:
- ✅ Accurate PnL calculation
- ✅ Real-time data
- ❌ Expensive query execution
- ❌ Potential timeout for large datasets

## WebSocket API

### Connection Endpoint

#### WS /ws/{user_id}
**Purpose**: Establish WebSocket connection for real-time updates

**Connection Flow**:
1. Client connects to `/ws/{user_id}`
2. Server validates user_id and authentication
3. Connection accepted/rejected
4. Monitoring loop started for user
5. Real-time updates sent

**Design Choices**:
- Path parameter for user_id
- Authentication validation
- Per-user monitoring loop
- Time restriction check (9:26 AM IST)

**Trade-offs**:
- ✅ Real-time bidirectional communication
- ✅ Efficient updates
- ❌ Connection management complexity
- ❌ Resource intensive (per-user loops)

### Message Types

#### Connection Established
```json
{
    "type": "connection_established",
    "connection_id": "uuid",
    "user_id": "integer",
    "timestamp": "iso_datetime"
}
```

#### Monitoring Errors
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

#### Order Updates (Future)
```json
{
    "type": "order_update",
    "order_id": "integer",
    "status": "string",
    "details": {...}
}
```

## Request/Response Patterns

### Request Validation

**Validation Layers**:
1. **Schema Validation**: Request body structure
2. **Business Validation**: Business rules (e.g., market hours)
3. **Authorization Validation**: User permissions
4. **Resource Validation**: Resource existence and access

### Error Response Format

**Standard Error Response**:
```json
{
    "error": "Error message",
    "code": "ERROR_CODE",
    "details": {...}
}
```

**HTTP Status Codes**:
- `200`: Success
- `201`: Created
- `400`: Bad Request (validation error)
- `401`: Unauthorized (authentication failure)
- `403`: Forbidden (authorization failure)
- `404`: Not Found
- `429`: Too Many Requests (rate limit)
- `500`: Internal Server Error

### Success Response Format

**Standard Success Response**:
```json
{
    "data": {...},
    "message": "Success message (optional)"
}
```

## Authentication Flow

### Token-Based Authentication

```
1. Client → POST /api/login (credentials)
2. Server → Validate credentials
3. Server → Generate session token
4. Server → Store token (DB + cache)
5. Server → Return token to client
6. Client → Store token
7. Client → Include token in subsequent requests (Authorization header)
8. Server → Validate token (middleware)
9. Server → Process request
```

### Token Validation

**Validation Steps**:
1. Extract token from Authorization header
2. Check cache for token (fast path)
3. If not in cache, check database
4. Validate expiration
5. Load user data
6. Attach user to request context

**Design Choices**:
- Cache-first lookup for performance
- Database fallback for cache miss
- Token expiration checked
- User data attached to request

**Trade-offs**:
- ✅ Fast validation (cache hit)
- ✅ Reliable validation (database fallback)
- ❌ Cache invalidation complexity
- ❌ Stale cache possibility

## Rate Limiting

### Rate Limit Strategy

**Per-User Limits**:
- Different limits for different endpoints
- Token bucket algorithm
- Per-user tracking

**Rate Limit Headers**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640995200
```

**Rate Limit Response (429)**:
```json
{
    "error": "Rate limit exceeded",
    "retry_after": 60
}
```

**Design Choices**:
- Per-user rate limiting
- Configurable limits per endpoint
- Clear error messages
- Retry-after information

**Trade-offs**:
- ✅ Prevents abuse
- ✅ Fair resource allocation
- ❌ Additional processing overhead
- ❌ State management complexity

## API Versioning

### Current Approach
- No explicit versioning
- Backward compatibility maintained
- Breaking changes avoided

### Future Considerations
- URL versioning: `/api/v1/orders`
- Header versioning: `API-Version: v1`
- Content negotiation

## API Documentation

### Documentation Approach
- OpenAPI/Swagger specification (potential)
- Inline code documentation
- This design document

### Documentation Needs
- Endpoint descriptions
- Request/response schemas
- Error codes and messages
- Authentication requirements
- Rate limits
- Examples

## Performance Considerations

### API Performance Targets
- **Response Time**: < 100ms (p95)
- **Throughput**: 100+ requests/second
- **Concurrent Connections**: 1000+ WebSocket connections

### Optimization Strategies
1. **Caching**: Session tokens, user data, settings
2. **Database Optimization**: Indexes, query optimization
3. **Connection Pooling**: Database and HTTP client pools
4. **Async Processing**: Asynchronous request handling
5. **Load Balancing**: Horizontal scaling (future)

## Security Considerations

### Security Measures
1. **HTTPS/WSS**: All communications encrypted
2. **Authentication**: Session token validation
3. **Authorization**: Permission checks
4. **Input Validation**: Request validation
5. **SQL Injection Prevention**: Parameterized queries
6. **Rate Limiting**: Abuse prevention
7. **CORS**: Controlled cross-origin access

### Security Trade-offs
- ✅ Multiple security layers
- ✅ Defense in depth
- ❌ Performance overhead
- ❌ Complexity

---

**Next**: [WebSocket Design](05-websocket-design.md)

