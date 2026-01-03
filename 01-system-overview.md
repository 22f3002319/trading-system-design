# System Overview

## Problem Statement

The system is designed to solve the following core problems:

1. **Real-time Order Management**: Provide a platform for algorithmic trading where users can place, modify, and monitor orders across multiple broker accounts in real-time.

2. **Automated Monitoring**: Continuously monitor broker APIs, calculate profit/loss, and automatically manage order lifecycle (entry, stop-loss, take-profit) without manual intervention.

3. **Multi-Account Coordination**: Handle multiple trading accounts per user, synchronize data across accounts, and ensure proper isolation and permissions.

4. **PnL Accuracy**: Calculate accurate profit/loss using FIFO (First-In-First-Out) matching algorithm that correctly handles partial fills, multiple entry orders, and split exits.

5. **Order Lifecycle Automation**: Automatically place protective orders (SL/TP), modify orders based on market conditions, and regenerate rejected orders based on strategy rules.

## Core Requirements

### Functional Requirements

1. **User Authentication & Authorization**
   - Secure user authentication with session management
   - Multi-account support per user
   - Permission-based feature access

2. **Order Management**
   - Place orders (Entry, Stop Loss, Take Profit)
   - Modify orders based on fair price changes
   - Cancel/reject order handling
   - Batch order placement

3. **Real-time Monitoring**
   - WebSocket-based real-time updates
   - 60-second monitoring cycles
   - Broker API synchronization
   - Position and order status tracking

4. **PnL Calculation**
   - FIFO-based matching algorithm
   - Real-time profit/loss updates
   - Historical PnL tracking
   - Unrealized vs Realized PnL

5. **Strategy Execution**
   - Strategy-based order placement
   - Entry regeneration for rejected orders
   - Trailing stop loss management
   - Scheduler-based automated orders

### Non-Functional Requirements

1. **Performance**
   - Sub-100ms API response time
   - 60-second monitoring cycle latency
   - Support 1000+ concurrent WebSocket connections
   - Handle 100+ orders per second

2. **Reliability**
   - 99.9% uptime during trading hours (9:26 AM - 3:30 PM IST)
   - Graceful error handling and recovery
   - Data consistency with ACID transactions
   - Automatic session refresh

3. **Security**
   - Secure credential storage
   - Session-based authentication
   - Rate limiting
   - User data isolation

4. **Scalability**
   - Horizontal scaling capability
   - Database optimization for growth
   - Efficient resource utilization
   - Caching strategies

## Constraints

### Technical Constraints

1. **Broker API Limitations**
   - Rate limits on API calls
   - Session expiry and refresh requirements
   - API availability during market hours only
   - Limited concurrent connection support

2. **Database Constraints**
   - MySQL 8.0+ compatibility
   - Transaction overhead for complex queries
   - Index maintenance for large datasets
   - Query optimization for real-time performance

3. **Network Constraints**
   - WebSocket connection stability
   - Network latency considerations
   - Connection timeout handling
   - Bandwidth limitations

4. **Time Constraints**
   - Trading hours: 9:26 AM - 3:30 PM IST
   - Real-time monitoring requirements
   - API call frequency limits
   - Session refresh windows

### Business Constraints

1. **Regulatory Compliance**
   - Trading hour restrictions
   - Order validation requirements
   - Audit trail maintenance
   - Data retention policies

2. **User Experience**
   - Real-time feedback requirements
   - Error message clarity
   - Minimal latency expectations
   - Concurrent operation support

3. **Cost Constraints**
   - Infrastructure costs
   - API call costs
   - Database storage costs
   - Network bandwidth costs

### Design Constraints

1. **Architecture Constraints**
   - Service-oriented design
   - Modular component structure
   - Clear separation of concerns
   - Repository pattern for data access

2. **Technology Constraints**
   - Python/FastAPI backend
   - MySQL database
   - WebSocket for real-time communication
   - REST API for broker integration

## System Boundaries

### In Scope

- Order placement and management
- Real-time monitoring and synchronization
- PnL calculation and reporting
- User authentication and session management
- Multi-account coordination
- Strategy-based order execution
- Automated order lifecycle management

### Out of Scope

- Broker API implementation (external dependency)
- Market data feed (external dependency)
- Payment processing
- User registration and onboarding
- Advanced charting and analysis tools
- Mobile application (web-based only)
- Third-party integrations beyond broker APIs

## Key Design Decisions

### 1. Real-time Communication: WebSocket over Polling

**Decision**: Use WebSocket for real-time updates instead of HTTP polling

**Rationale**:
- Lower latency for real-time updates
- Reduced server load (no constant polling)
- Bidirectional communication capability
- Better resource utilization

**Trade-offs**:
- ✅ Lower latency
- ✅ Reduced server load
- ❌ More complex connection management
- ❌ Connection stability challenges

### 2. FIFO PnL Matching Algorithm

**Decision**: Use FIFO (First-In-First-Out) algorithm for PnL calculation

**Rationale**:
- Accurate profit/loss calculation
- Handles partial fills correctly
- Standard accounting practice
- Supports complex order scenarios

**Trade-offs**:
- ✅ Accurate PnL calculation
- ✅ Handles complex scenarios
- ❌ Complex SQL queries (CTEs)
- ❌ Higher computational cost

### 3. Conditional Execution Workflow

**Decision**: Implement step-by-step conditional execution in monitoring loop

**Rationale**:
- Ensures data consistency
- Prevents cascading failures
- Clear dependency management
- Easier error handling and debugging

**Trade-offs**:
- ✅ Data consistency
- ✅ Clear error handling
- ❌ Sequential execution (slower)
- ❌ Complex conditional logic

### 4. Session-Based Authentication

**Decision**: Use session tokens instead of JWT for authentication

**Rationale**:
- Better server-side control
- Easier session revocation
- Simpler token management
- Better security for sensitive operations

**Trade-offs**:
- ✅ Better control
- ✅ Easier revocation
- ❌ Server-side storage required
- ❌ Stateless scaling challenges

### 5. Transaction-Based Database Operations

**Decision**: Use database transactions for critical operations

**Rationale**:
- Data consistency guarantee
- Atomic operations
- Rollback capability
- ACID compliance

**Trade-offs**:
- ✅ Data consistency
- ✅ Rollback safety
- ❌ Lock contention
- ❌ Performance overhead

## System Architecture Overview

```
┌─────────────┐
│   Client    │
│  (Browser)  │
└──────┬──────┘
       │
       │ HTTPS/WSS
       │
┌──────▼─────────────────────────────────────────┐
│          FastAPI Application                    │
│  ┌──────────────────────────────────────────┐  │
│  │  REST API Endpoints                      │  │
│  │  - Authentication                        │  │
│  │  - Order Management                      │  │
│  │  - Account Management                    │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │  WebSocket Server                        │  │
│  │  - Real-time Updates                     │  │
│  │  - Monitoring Loop                       │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │  Services Layer                          │  │
│  │  - Trading Service                       │  │
│  │  - Session Manager                       │  │
│  │  - Order Processing                      │  │
│  │  - Scheduler Service                     │  │
│  └──────────────────────────────────────────┘  │
└──────┬─────────────────────────────────────────┘
       │
       │ SQL Queries
       │
┌──────▼──────────┐
│   MySQL DB      │
│  - Users        │
│  - Orders       │
│  - Accounts     │
│  - Sessions     │
└─────────────────┘
       │
       │ API Calls
       │
┌──────▼──────────┐
│  Broker APIs    │
│  (IIFL, etc.)   │
└─────────────────┘
```

## Success Criteria

The system is considered successful if it meets:

1. **Functional Success**
   - All orders placed successfully
   - Real-time updates delivered within 60 seconds
   - Accurate PnL calculations
   - Proper order lifecycle management

2. **Performance Success**
   - API response time < 100ms (p95)
   - Monitoring cycle completes in < 60 seconds
   - Supports 1000+ concurrent connections
   - Handles 100+ orders/second

3. **Reliability Success**
   - 99.9% uptime during trading hours
   - < 0.1% data inconsistency
   - Automatic error recovery
   - Graceful degradation

4. **User Experience Success**
   - Real-time updates without noticeable delay
   - Clear error messages
   - Minimal failed operations
   - Responsive UI interactions

---

**Next**: [System Architecture](02-architecture.md)

