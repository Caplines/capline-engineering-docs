# API Access Log (Audit Module) Documentation

## Purpose

This document provides comprehensive documentation for the API Access Log (Audit Module) system implemented in the Capline backend services. The audit module automatically tracks, logs, and analyzes all API requests to provide comprehensive audit trails, security monitoring, usage analytics, and anomaly detection capabilities.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Components](#core-components)
4. [Data Flow](#data-flow)
5. [Log Entry Structure](#log-entry-structure)
6. [Database Schema](#database-schema)
7. [Configuration](#configuration)
8. [Usage](#usage)
9. [Metrics and Analytics](#metrics-and-analytics)
10. [Anomaly Detection](#anomaly-detection)
11. [Storage and Persistence](#storage-and-persistence)
12. [Performance Considerations](#performance-considerations)
13. [Troubleshooting](#troubleshooting)

## Overview

The API Access Log (Audit Module) is a comprehensive logging and monitoring system that automatically captures detailed information about every API request made to the backend services. The system provides:

- **Complete Audit Trail**: Every API request is logged with full context including user, endpoint, request/response details, and timing information
- **Security Monitoring**: Tracks user activity patterns, IP addresses, and detects suspicious behavior
- **Usage Analytics**: Provides insights into API usage patterns, popular endpoints, and user activity
- **Anomaly Detection**: Automatically detects unusual patterns in user behavior and generates alerts
- **Performance Monitoring**: Tracks response times, data volumes, and system performance metrics
- **Compliance**: Maintains detailed logs for compliance and regulatory requirements

The system operates transparently in the background, requiring minimal configuration and automatically handling log buffering, persistence, and analysis.

## Architecture

The audit module follows a multi-stage architecture designed for high performance and reliability:

```
┌─────────────────────────────────────────────────────────────┐
│                    API Request                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              AuditInterceptor (Global Interceptor)          │
│  - Captures request/response data                           │
│  - Extracts user information                                │
│  - Sanitizes sensitive data                                 │
│  - Determines operation type                                │
│  - Calculates metrics                                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ├──────────────────────────────┐
                           │                              │
                           ▼                              ▼
┌──────────────────────────────┐    ┌──────────────────────────────┐
│   AuditBufferService         │    │  AuditLeaderboardService     │
│   (Redis Buffer)             │    │  (Real-time Metrics)         │
│                              │    │                              │
│  - Buffers logs in Redis     │    │  - Updates leaderboards      │
│  - Batches for efficiency    │    │  - Tracks unique resources   │
│  - 5-minute batches          │    │  - Monitors data volume      │
│                              │    │  - Detects anomalies         │
└──────────────┬───────────────┘    └──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│         FlushAuditLogsCron (Every 5 minutes)                │
│  - Reads batch from Redis                                   │
│  - Resolves user IDs from UUIDs                             │
│  - Transforms to database format                            │
│  - Batch inserts to PostgreSQL                              │
│  - Rotates to new batch                                     │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL Database                            │
│              api_access_log table                           │
│  - Permanent storage                                        │
│  - Indexed for fast queries                                 │
│  - Historical data analysis                                 │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### AuditInterceptor

**Location**: `src/audit/interceptors/audit.interceptor.ts`

The `AuditInterceptor` is a global NestJS interceptor that automatically captures information about every API request. It is registered as a global interceptor in `src/app.ts`.

#### Responsibilities

1. **Request Capture**: Intercepts all incoming HTTP requests
2. **Data Extraction**: Extracts user information, request details, and response data
3. **Data Sanitization**: Removes sensitive information (passwords, tokens, secrets)
4. **Operation Classification**: Determines if the operation is READ or WRITE
5. **Module Detection**: Identifies which module/feature the endpoint belongs to
6. **Resource ID Extraction**: Extracts resource IDs from requests and responses
7. **Metrics Calculation**: Calculates response size, record count, and duration
8. **Log Creation**: Creates complete audit log entries
9. **Async Processing**: Sends logs to buffer and metrics services asynchronously

#### Key Features

- **Automatic Operation Type Detection**: Automatically classifies GET/HEAD/OPTIONS as READ, others as WRITE
- **Sensitive Data Sanitization**: Automatically redacts passwords, tokens, secrets, and API keys
- **Resource ID Tracking**: Extracts resource IDs from multiple sources (path params, query params, body, response)
- **Error Handling**: Logs both successful and failed requests with error details
- **Performance Tracking**: Measures request duration from start to completion

#### Skip Audit

Routes can skip audit logging using the `@SkipAudit()` decorator:

```typescript
import { SkipAudit } from '@audit/decorators/skip-audit.decorator';

@Get('/health')
@SkipAudit()
async healthCheck() {
  return { status: 'ok' };
}
```

### AuditBufferService

**Location**: `src/audit/services/audit-buffer.service.ts`

The `AuditBufferService` manages temporary storage of audit logs in Redis before they are flushed to the database.

#### Responsibilities

1. **Log Buffering**: Stores logs in Redis lists for batching
2. **Batch Management**: Manages batch IDs and rotation
3. **Efficient Storage**: Uses Redis LPUSH for fast append operations
4. **TTL Management**: Sets expiration on buffers to prevent memory leaks

#### How It Works

1. Each log entry is added to a Redis list using `LPUSH`
2. Logs are grouped into batches identified by batch IDs (ISO timestamps)
3. The current batch ID is stored in Redis with a TTL
4. When a batch is flushed, a new batch ID is created
5. Buffers have a 7-day TTL to allow for recovery if needed

#### Redis Keys

- `audit:buffer:{batchId}`: List of log entries for a batch
- `audit:buffer:current`: Current batch ID

### AuditLeaderboardService

**Location**: `src/audit/services/audit-leaderboard.service.ts`

The `AuditLeaderboardService` provides real-time metrics and analytics on user activity.

#### Responsibilities

1. **Leaderboard Updates**: Maintains daily and hourly leaderboards of user activity
2. **Unique Resource Tracking**: Tracks unique resources accessed per user per day
3. **Data Volume Monitoring**: Monitors data volume transferred per user
4. **Write Operation Counting**: Tracks write operations per user per day
5. **Anomaly Detection**: Detects unusual patterns and generates alerts

#### Metrics Tracked

1. **Request Count**: Total requests per user per day/hour
2. **Unique Resources**: Number of unique resource IDs accessed
3. **Data Volume**: Total bytes transferred (response sizes)
4. **Write Operations**: Count of WRITE operations per day
5. **Top Users**: Users with highest activity (daily/hourly)

#### Redis Data Structures

- **Sorted Sets (ZSET)**: For leaderboards (user ID as member, count as score)
- **HyperLogLog (PFADD/PFCOUNT)**: For unique resource counting
- **Strings (INCR)**: For data volume and write counts
- **Lists**: For storing alerts

### FlushAuditLogsCron

**Location**: `src/audit/cron/flush-audit-logs.cron.ts`

The `FlushAuditLogsCron` is a scheduled job that runs every 5 minutes to flush buffered logs from Redis to PostgreSQL.

#### Responsibilities

1. **Batch Retrieval**: Retrieves the current batch of logs from Redis
2. **User ID Resolution**: Resolves UUIDs from JWT tokens to integer IDs in database
3. **Data Transformation**: Transforms log entries to database schema format
4. **Batch Insertion**: Inserts logs into PostgreSQL in batches
5. **Error Handling**: Handles individual insert failures gracefully
6. **Buffer Cleanup**: Clears processed buffers from Redis
7. **Batch Rotation**: Creates new batch ID for next cycle

#### Execution Flow

1. Runs every 5 minutes (configurable via `AUDIT_CONFIG.BUFFER_FLUSH_INTERVAL_MS`)
2. Retrieves current batch ID from Redis
3. Fetches all logs from the batch
4. For each log:
   - Resolves user UUID to integer ID based on `userTableType`
   - Fetches user role from database
   - Maps enums and transforms data
5. Batch inserts all logs to PostgreSQL
6. Clears the processed buffer
7. Rotates to a new batch ID

#### User ID Resolution

The system supports multiple user table types:
- `ADMIN_DETAILS`: AdminDetails table (superAdmin, admin, lead, auditor, associate)
- `USER_DETAILS`: UserDetails table (user)
- `EV_CLIENT_USER_DETAILS`: EvClientUserDetails table (evClient)
- `CREDENTIALING_CLIENT_DETAILS`: CredentialingClientDetails table (credentialingClient)

### ApiAccessLogRepository

**Location**: `src/audit/db/api-access-log.repository.ts`

The `ApiAccessLogRepository` provides database access methods for the `api_access_log` table using Prisma.

#### Responsibilities

1. **Database Operations**: Provides CRUD operations for audit logs
2. **Batch Insertion**: Supports efficient batch insertion of logs
3. **Query Interface**: Provides methods for querying audit logs

## Data Flow

### Request Processing Flow

1. **Request Arrives**: HTTP request arrives at the API
2. **Interceptor Activation**: `AuditInterceptor` intercepts the request
3. **Data Collection**: Interceptor collects:
   - User information from JWT token
   - Request details (endpoint, method, params, body)
   - IP address and user agent
   - Start timestamp
4. **Request Processing**: Request is processed by the controller/service
5. **Response Capture**: Interceptor captures response data
6. **Log Creation**: Complete log entry is created with:
   - Request details
   - Response details (code, size, record count)
   - Resource IDs
   - Duration
   - Error information (if any)
7. **Async Processing**: Log is sent to:
   - `AuditBufferService`: For storage in Redis buffer
   - `AuditLeaderboardService`: For real-time metrics update
8. **Response Return**: Response is returned to client

### Log Persistence Flow

1. **Buffer Accumulation**: Logs accumulate in Redis buffer (5-minute batches)
2. **Cron Trigger**: `FlushAuditLogsCron` runs every 5 minutes
3. **Batch Retrieval**: Current batch is retrieved from Redis
4. **Data Transformation**: Logs are transformed to database format
5. **User Resolution**: UUIDs are resolved to integer IDs
6. **Batch Insert**: Logs are inserted into PostgreSQL in batches
7. **Buffer Cleanup**: Processed buffer is cleared from Redis
8. **Batch Rotation**: New batch ID is created for next cycle

## Log Entry Structure

### AuditLogEntry Interface

```typescript
interface AuditLogEntry {
  // User Information
  userId?: string;              // UUID from JWT token
  userTableType?: string;      // Type of user table (admin, user, etc.)
  userRole?: string;            // User role name
  
  // Request Details
  endpoint: string;            // API endpoint path
  method: string;               // HTTP method (GET, POST, etc.)
  operationType: 'READ' | 'WRITE';  // Operation classification
  module?: string;              // Module name (extracted from endpoint)
  
  // Request Payload (sanitized)
  queryParams?: Record<string, any>;  // Query parameters
  pathParams?: Record<string, any>;   // Path parameters
  body?: Record<string, any>;         // Request body (sanitized)
  
  // Response Metadata
  responseCode: number;         // HTTP status code
  responseSize?: number;        // Response size in bytes
  recordCount?: number;         // Number of records returned
  resourceIds?: number[];       // IDs of resources accessed
  hasError: boolean;            // Whether request resulted in error
  errorMessage?: string;        // Error message if hasError is true
  
  // Tracking Information
  ipAddress?: string;           // Client IP address
  userAgent?: string;           // User agent string
  duration?: number;            // Request duration in milliseconds
  
  // Timestamp
  accessedAt: Date;            // When the request was made
}
```

### Data Sanitization

The interceptor automatically sanitizes sensitive data in request/response payloads:

**Sanitized Fields**:
- `password`
- `token`
- `secret`
- `authorization`
- `auth`
- `key`
- `apiKey`

Any field containing these keywords (case-insensitive) is replaced with `[REDACTED]`.

### Resource ID Extraction

The system extracts resource IDs from multiple sources:

1. **Path Parameters**: `id`, `Id` from URL path
2. **Query Parameters**: `id`, `Id` from query string
3. **Request Body**: `id`, `Id` from request body (supports arrays)
4. **Response Data**: `id`, `databaseId`, `_id`, `dbId` from response
5. **Raw Data**: Custom `_auditRawData` property on request object

Only positive integer IDs are extracted and stored.

## Database Schema

### ApiAccessLog Model

The `api_access_log` table stores all audit log entries permanently.

#### Table Structure

```prisma
model ApiAccessLog {
  id   Int    @id @default(autoincrement())
  uuid String @unique @default(uuid())

  // User identification
  userId        Int                       @map("user_id")
  userTableType ApiAccessLogUserTableType @map("user_table_type")
  userRole      ApiAccessLogUserRole?     @map("user_role")

  // Request details
  endpoint      String
  method        ApiAccessLogHttpMethod
  operationType ApiAccessLogOperationType @map("operation_type")
  module        String

  // Request payload (sanitized)
  queryParams Json? @map("query_params")
  pathParams  Json? @map("path_params")
  body        Json?

  // Response metadata
  responseCode Int     @map("response_code")
  responseSize Int?    @map("response_size")
  recordCount  Int?    @map("record_count")
  resourceIds  Int[]   @default([]) @map("resource_ids")
  hasError     Boolean @default(false) @map("has_error")
  errorMessage String? @map("error_message")

  // Tracking
  ipAddress String? @map("ip_address")
  userAgent String? @map("user_agent")
  duration  Int?

  // Timestamps
  accessedAt DateTime @map("accessed_at")
  createdAt  DateTime @default(now()) @map("created_at")

  @@index([userId, accessedAt])
  @@index([operationType, accessedAt])
  @@index([module, accessedAt])
  @@index([responseCode, accessedAt])
  @@map("api_access_log")
}
```

#### Enums

**ApiAccessLogUserTableType**:
- `ADMIN_DETAILS`: AdminDetails table
- `USER_DETAILS`: UserDetails table
- `EV_CLIENT_USER_DETAILS`: EvClientUserDetails table
- `CREDENTIALING_CLIENT_DETAILS`: CredentialingClientDetails table

**ApiAccessLogUserRole**:
- `SUPER_ADMIN`
- `ADMIN`
- `USER`
- `EV_CLIENT`
- `CREDENTIALING_CLIENT`
- `ASSOCIATE`
- `AUDITOR`
- `LEAD`

**ApiAccessLogHttpMethod**:
- `GET`
- `POST`
- `PUT`
- `PATCH`
- `DELETE`

**ApiAccessLogOperationType**:
- `READ`
- `WRITE`

#### Indexes

The table has four composite indexes for efficient querying:

1. `[userId, accessedAt]`: Fast queries by user and time range
2. `[operationType, accessedAt]`: Fast queries by operation type and time
3. `[module, accessedAt]`: Fast queries by module and time
4. `[responseCode, accessedAt]`: Fast queries by response code and time

## Configuration

### Audit Configuration

**Location**: `src/audit/constants.ts`

```typescript
export const AUDIT_CONFIG = {
    BUFFER_FLUSH_INTERVAL_MS: 5 * 60 * 1000,  // 5 minutes
    LEADERBOARD_TTL_SECONDS: 7 * 24 * 60 * 60,  // 7 days
    USER_STATS_TTL_SECONDS: 24 * 60 * 60,  // 24 hours
    ANOMALY_THRESHOLDS: {
        MAX_DAILY_REQUESTS: 5000,
        MAX_UNIQUE_RESOURCES: 1000,
        MAX_DATA_VOLUME_MB: 100,
        MAX_WRITES_PER_DAY: 500,
        TOP_N_ALERT: 3,
    },
} as const;
```

### Configuration Parameters

#### BUFFER_FLUSH_INTERVAL_MS

- **Default**: 5 minutes (300,000 ms)
- **Purpose**: How often logs are flushed from Redis to PostgreSQL
- **Impact**: Lower values = more frequent writes, higher database load. Higher values = more logs in memory, risk of data loss if Redis fails

#### LEADERBOARD_TTL_SECONDS

- **Default**: 7 days
- **Purpose**: How long leaderboard data is kept in Redis
- **Impact**: Determines how far back you can query leaderboard data from Redis

#### USER_STATS_TTL_SECONDS

- **Default**: 24 hours
- **Purpose**: How long user statistics (unique resources, data volume, write counts) are kept in Redis
- **Impact**: Determines retention period for daily user metrics in Redis

#### ANOMALY_THRESHOLDS

Thresholds for anomaly detection:

- **MAX_DAILY_REQUESTS**: Maximum requests per user per day before alert (default: 5000)
- **MAX_UNIQUE_RESOURCES**: Maximum unique resources accessed per day before alert (default: 1000)
- **MAX_DATA_VOLUME_MB**: Maximum data volume in MB per day before alert (default: 100 MB)
- **MAX_WRITES_PER_DAY**: Maximum write operations per day before alert (default: 500)
- **TOP_N_ALERT**: Alert if user is in top N by request count (default: 3)

### Redis Keys

All Redis keys used by the audit module:

- `audit:buffer:{batchId}`: Log buffer for a batch
- `audit:buffer:current`: Current batch ID
- `audit:leaderboard:daily:{dateKey}`: Daily leaderboard (sorted set)
- `audit:leaderboard:hourly:{hourKey}`: Hourly leaderboard (sorted set)
- `audit:user:{userId}:resources:{dateKey}`: Unique resources per user per day (HyperLogLog)
- `audit:user:{userId}:volume:{dateKey}`: Data volume per user per day (string)
- `audit:user:{userId}:writes:{dateKey}`: Write count per user per day (string)
- `audit:alerts:{userId}`: User alerts (list)

## Usage

### Automatic Logging

By default, all API endpoints are automatically logged. No configuration is required.

### Skipping Audit Logging

To skip audit logging for specific endpoints:

```typescript
import { Controller, Get } from '@nestjs/common';
import { SkipAudit } from '@audit/decorators/skip-audit.decorator';

@Controller('health')
export class HealthController {
  @Get()
  @SkipAudit()
  async healthCheck() {
    return { status: 'ok' };
  }
}
```

**When to Skip Audit**:
- Health check endpoints
- Internal monitoring endpoints
- High-frequency endpoints that don't need auditing
- Public endpoints with no user context

### Querying Audit Logs

Use the `ApiAccessLogRepository` to query audit logs:

```typescript
import { Injectable } from '@nestjs/common';
import { ApiAccessLogRepository } from '@audit/db';

@Injectable()
export class AuditService {
  constructor(
    private readonly apiAccessLogRepository: ApiAccessLogRepository,
  ) {}

  async getUserLogs(userId: number, startDate: Date, endDate: Date) {
    return this.apiAccessLogRepository.findMany({
      where: {
        userId,
        accessedAt: {
          gte: startDate,
          lte: endDate,
        },
      },
      orderBy: {
        accessedAt: 'desc',
      },
    });
  }

  async getModuleLogs(module: string, startDate: Date, endDate: Date) {
    return this.apiAccessLogRepository.findMany({
      where: {
        module,
        accessedAt: {
          gte: startDate,
          lte: endDate,
        },
      },
    });
  }
}
```

### Accessing Metrics

Use the `AuditLeaderboardService` to access real-time metrics:

```typescript
import { Injectable } from '@nestjs/common';
import { AuditLeaderboardService } from '@audit/services';
import { formatDateKey } from '@audit/constants';

@Injectable()
export class AnalyticsService {
  constructor(
    private readonly auditLeaderboardService: AuditLeaderboardService,
  ) {}

  async getTopUsers(date: Date = new Date()) {
    const dateKey = formatDateKey(date);
    return this.auditLeaderboardService.getTopUsersDaily(dateKey, 10);
  }

  async getUserAlerts(userId: string) {
    return this.auditLeaderboardService.getUserAlerts(userId, 10);
  }
}
```

## Metrics and Analytics

### Leaderboards

The system maintains real-time leaderboards showing the most active users:

#### Daily Leaderboard

- **Key**: `audit:leaderboard:daily:{YYYYMMDD}`
- **Data Structure**: Redis Sorted Set
- **Score**: Request count
- **TTL**: 7 days

#### Hourly Leaderboard

- **Key**: `audit:leaderboard:hourly:{YYYYMMDDHH}`
- **Data Structure**: Redis Sorted Set
- **Score**: Request count
- **TTL**: 7 days

### User Statistics

Per-user, per-day statistics are tracked:

#### Unique Resources

- **Metric**: Number of unique resource IDs accessed
- **Storage**: Redis HyperLogLog
- **Key**: `audit:user:{userId}:resources:{YYYYMMDD}`
- **TTL**: 24 hours

#### Data Volume

- **Metric**: Total bytes transferred (response sizes)
- **Storage**: Redis String (integer)
- **Key**: `audit:user:{userId}:volume:{YYYYMMDD}`
- **TTL**: 24 hours

#### Write Operations

- **Metric**: Count of WRITE operations
- **Storage**: Redis String (integer)
- **Key**: `audit:user:{userId}:writes:{YYYYMMDD}`
- **TTL**: 24 hours

### Querying Metrics

```typescript
import { AuditLeaderboardService } from '@audit/services';
import { formatDateKey } from '@audit/constants';

// Get top 10 users for today
const dateKey = formatDateKey(new Date());
const topUsers = await auditLeaderboardService.getTopUsersDaily(dateKey, 10);

// Get user alerts
const alerts = await auditLeaderboardService.getUserAlerts(userId, 10);
```

## Anomaly Detection

The system automatically detects anomalies in user behavior and generates alerts.

### Anomaly Types

1. **High Request Count**: User exceeds maximum daily requests
2. **High Unique Resources**: User accesses too many unique resources
3. **High Data Volume**: User transfers excessive data
4. **High Write Operations**: User performs too many write operations
5. **Top User Alert**: User is in top N by request count

### Alert Storage

Alerts are stored in Redis lists:

- **Key**: `audit:alerts:{userId}`
- **Format**: `{timestamp}: {alert message}`
- **Limit**: Last 100 alerts per user
- **TTL**: 30 days

### Alert Format

```
2025-01-15T10:30:00.000Z: High request count: 5234 requests (threshold: 5000)
2025-01-15T11:15:00.000Z: High unique resource access: 1234 resources (threshold: 1000)
2025-01-15T12:00:00.000Z: User in top 3 by request count (rank: 2)
```

### Accessing Alerts

```typescript
import { AuditLeaderboardService } from '@audit/services';

const alerts = await auditLeaderboardService.getUserAlerts(userId, 10);
// Returns: Array of alert strings
```

### Customizing Thresholds

Modify thresholds in `src/audit/constants.ts`:

```typescript
export const AUDIT_CONFIG = {
    // ...
    ANOMALY_THRESHOLDS: {
        MAX_DAILY_REQUESTS: 10000,  // Increase threshold
        MAX_UNIQUE_RESOURCES: 2000,
        MAX_DATA_VOLUME_MB: 200,
        MAX_WRITES_PER_DAY: 1000,
        TOP_N_ALERT: 5,  // Alert top 5 instead of top 3
    },
} as const;
```

## Storage and Persistence

### Redis Storage

Redis is used for:
1. **Log Buffering**: Temporary storage before database flush
2. **Metrics**: Real-time leaderboards and statistics
3. **Alerts**: User anomaly alerts

**Memory Considerations**:
- Log buffers have 7-day TTL
- Leaderboards have 7-day TTL
- User stats have 24-hour TTL
- Alerts have 30-day TTL

### PostgreSQL Storage

PostgreSQL is used for permanent storage of all audit logs.

**Storage Considerations**:
- Logs are never deleted automatically
- Consider implementing archival strategy for old logs
- Indexes are optimized for common query patterns
- JSON columns are used for flexible payload storage

### Backup and Recovery

**Redis**:
- Logs in buffers are at risk if Redis fails before flush
- TTL ensures buffers don't accumulate indefinitely
- Consider Redis persistence (RDB/AOF) for critical data

**PostgreSQL**:
- Standard database backup procedures apply
- Consider partitioning by date for large datasets
- Implement archival strategy for compliance

## Performance Considerations

### Impact on Request Processing

The audit interceptor is designed to minimize impact on request processing:

1. **Asynchronous Processing**: Log buffering and metrics updates are non-blocking
2. **Efficient Data Structures**: Uses Redis data structures optimized for performance
3. **Batch Processing**: Logs are flushed in batches, not individually
4. **Minimal Overhead**: Data extraction and sanitization are lightweight operations

### Scalability

The system is designed to scale:

1. **Horizontal Scaling**: Multiple application instances can share Redis
2. **Batch Processing**: Reduces database write load
3. **Efficient Indexes**: Database indexes support fast queries
4. **TTL Management**: Prevents unbounded memory growth in Redis

### Optimization Tips

1. **Skip Unnecessary Endpoints**: Use `@SkipAudit()` for high-frequency, non-critical endpoints
2. **Monitor Buffer Sizes**: Watch Redis memory usage
3. **Adjust Flush Interval**: Balance between memory usage and data loss risk
4. **Database Indexing**: Ensure indexes are optimized for your query patterns
5. **Archive Old Logs**: Implement archival for logs older than retention period

## Troubleshooting

### Logs Not Appearing in Database

**Symptoms**: Logs are being created but not appearing in PostgreSQL

**Possible Causes**:
1. Cron job not running
2. Redis connection issues
3. Database connection issues
4. User ID resolution failures

**Solutions**:
1. Check cron job logs: `FlushAuditLogsCron` should log every 5 minutes
2. Verify Redis connection
3. Verify PostgreSQL connection
4. Check for user resolution errors in logs

### High Redis Memory Usage

**Symptoms**: Redis memory usage is high

**Possible Causes**:
1. Buffers not being flushed
2. TTL not working
3. Too many logs accumulating

**Solutions**:
1. Verify cron job is running
2. Check buffer TTL settings
3. Reduce flush interval if needed
4. Monitor buffer sizes

### Missing User Information

**Symptoms**: Logs have null or missing user information

**Possible Causes**:
1. JWT token not present
2. User UUID not found in database
3. User table type mismatch

**Solutions**:
1. Verify JWT token is being sent
2. Check user exists in appropriate table
3. Verify `userTableType` mapping is correct

### Performance Issues

**Symptoms**: API requests are slow

**Possible Causes**:
1. Audit interceptor overhead
2. Redis connection issues
3. Database write bottlenecks

**Solutions**:
1. Use `@SkipAudit()` for high-frequency endpoints
2. Check Redis connection pool
3. Optimize database indexes
4. Consider increasing flush interval

### Anomaly Alerts Not Working

**Symptoms**: Anomalies not being detected or alerts not generated

**Possible Causes**:
1. Thresholds too high
2. Metrics not being updated
3. Alert storage issues

**Solutions**:
1. Review and adjust thresholds
2. Verify `AuditLeaderboardService` is being called
3. Check Redis connection for alert storage

## Best Practices

### When to Audit

- **Always Audit**: User actions, data modifications, sensitive operations
- **Consider Skipping**: Health checks, internal monitoring, public endpoints without user context

### Data Retention

- **Compliance Requirements**: Determine retention period based on regulations
- **Storage Costs**: Balance retention with storage costs
- **Archival Strategy**: Implement archival for old logs

### Monitoring

- **Regular Review**: Review audit logs regularly for security issues
- **Anomaly Investigation**: Investigate anomaly alerts promptly
- **Performance Monitoring**: Monitor system performance impact

### Security

- **Sensitive Data**: System automatically sanitizes sensitive data
- **Access Control**: Restrict access to audit logs
- **Encryption**: Consider encrypting audit logs at rest

## Conclusion

The API Access Log (Audit Module) provides comprehensive logging, monitoring, and analytics capabilities for the Capline backend services. The system operates transparently in the background, automatically capturing detailed information about every API request while maintaining high performance and scalability.

Regular monitoring, log analysis, and configuration adjustments ensure the system continues to meet security, compliance, and performance requirements as the application evolves.

