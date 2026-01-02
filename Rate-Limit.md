# Rate Limiting Documentation

## Purpose

This document provides comprehensive documentation for the multi-layered rate limiting system implemented in the Capline backend services. The system employs a three-tier architecture to protect API endpoints from abuse, ensure fair resource usage, and maintain system stability under varying load conditions.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Layer 1: Cloudflare Rate Limiting](#layer-1-cloudflare-rate-limiting)
4. [Layer 2: IP-Based Throttling](#layer-2-ip-based-throttling)
5. [Layer 3: User-Based Rate Limiting](#layer-3-user-based-rate-limiting)
   - [Part 1: Authentication Rate Limiting](#part-1-authentication-rate-limiting)
   - [Part 2: Read Operations Rate Limiting](#part-2-read-operations-rate-limiting)
   - [Part 3: Write Operations Rate Limiting](#part-3-write-operations-rate-limiting)
6. [Implementation Details](#implementation-details)
7. [Configuration](#configuration)
8. [Usage Examples](#usage-examples)
9. [Monitoring and Logging](#monitoring-and-logging)
10. [Error Handling](#error-handling)

## Overview

The rate limiting system consists of three distinct layers that work together to provide comprehensive protection:

- **Layer 1**: Cloudflare-managed DDoS protection and rate limiting (edge-level protection)
- **Layer 2**: Application-level IP-based throttling (200 requests per minute per route)
- **Layer 3**: User-based rate limiting with three categories:
  - Authentication endpoints: 5 requests per minute
  - Read operations: 50 requests per minute per user
  - Write operations: 10 requests per minute per user

Each layer operates independently and provides defense at different stages of the request lifecycle, ensuring robust protection against various attack vectors.

## Architecture

The rate limiting system follows a layered architecture where requests flow through multiple checkpoints:

```
┌───────────────────────────────────────────────────────────-──┐
│                    Cloudflare Layer                          │
│              DDoS Protection + Rate Limit                    │
│                                                              │
└──────────────────────────┬──────────────────────────────────-┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────-───┐
│                  Throttle Layer (Layer 2)                    │
│              IP-Based: 200 requests/min/route                │
│                      (IMPLEMENTED)                           │
└──────────────────────────┬─────────────────────────────────-─┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────-───┐
│            Backend Services Rate Limiting                    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │  Auth Services                                     │      │
│  │  5 requests/min                                    │      │
│  │  (IMPLEMENTED)                                     │      │
│  └────────────────────────────────────────────────────┘      │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │  Read Services                                     │      │
│  │  50 requests/min/user                              │      │
│  │  Tracks user activity                              │      │
│  │  (IMPLEMENTED)                                     │      │
│  └────────────────────────────────────────────────────┘      │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │  Write Services                                    │      │
│  │  10 requests/min                                   │      │
│  │  Tracks activity                                   │      │
│  │  (IMPLEMENTED)                                     │      │
│  └────────────────────────────────────────────────────┘      │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │  Logger                                            │      │
│  │  APP_ID/USER_ID/ACTION                             │      │
│  │  (IMPLEMENTED)                                     │      │
│  └────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────-┘
```

## Layer 1: Cloudflare Rate Limiting

### Description

Cloudflare provides the first line of defense at the edge network level. This layer handles DDoS protection and rate limiting before requests reach the application servers.

### Status

**To Be Implemented**

### Features

- DDoS protection and mitigation
- Edge-level rate limiting
- Geographic-based filtering
- Bot detection and mitigation
- Web Application Firewall (WAF) capabilities

### Configuration

Cloudflare rate limiting is managed through the Cloudflare dashboard and is configured at the DNS/edge level. Configuration details should be documented separately in the infrastructure documentation.

### Benefits

- Reduces load on application servers by filtering malicious traffic at the edge
- Provides protection against large-scale DDoS attacks
- Offers global distribution of rate limiting logic
- Reduces bandwidth costs by blocking unwanted traffic early

## Layer 2: IP-Based Throttling

### Description

Layer 2 implements application-level throttling based on the client's IP address. This provides a second line of defense after Cloudflare and protects against abuse from specific IP addresses.

### Implementation

The throttling is implemented using the `@nestjs/throttler` package, which provides a global guard that applies to all routes by default.

### Configuration

**Location**: `src/constants.ts`

```typescript
export const THROTTLE_CONFIG = {
    GENERAL: {
        TTL: 60_000,  // Time window: 60 seconds (1 minute)
        LIMIT: 200    // Maximum requests per window
    }
}
```

### Rate Limit

- **Limit**: 200 requests per minute per route
- **Scope**: Per IP address
- **Window**: 60 seconds (rolling window)

### How It Works

1. The `ThrottlerGuard` is registered as a global guard in `src/app.ts`
2. For each request, the guard extracts the client's IP address
3. It checks the request count against Redis storage
4. If the limit is exceeded, the request is rejected with HTTP 429 (Too Many Requests)
5. The counter resets after the time window expires

### Key Characteristics

- Applied globally to all routes unless explicitly skipped
- Uses Redis for distributed rate limiting across multiple server instances
- IP-based identification ensures protection even for unauthenticated requests
- Per-route tracking means each endpoint has its own counter per IP

### Skipping Throttling

To skip throttling for specific routes, use the `@SkipThrottle()` decorator from `@nestjs/throttler`:

```typescript
import { SkipThrottle } from '@nestjs/throttler';

@SkipThrottle()
@Get('/health')
async healthCheck() {
  return { status: 'ok' };
}
```

## Layer 3: User-Based Rate Limiting

Layer 3 provides fine-grained rate limiting based on user identity and operation type. This layer is divided into three parts, each serving a specific purpose.

### Part 1: Authentication Rate Limiting

#### Description

Authentication rate limiting protects login and authentication endpoints from brute-force attacks and credential stuffing attempts.

#### Implementation

Implemented using custom rate limiting logic in `src/_common/utils/utils.ts` with configuration in `src/auth/constants.ts`.

#### Configuration

**Location**: `src/auth/constants.ts`

```typescript
export const RATE_LIMIT_CONFIG = {
  DEFAULT: {
    MAX_ATTEMPTS: 5,
    WINDOW_SECONDS: 60,
    COOLDOWN_SECONDS: 300,  // 5 minutes base cooldown
  },
  // Additional configurations for different auth endpoints
  ADMIN_LOGIN: { ... },
  CREDENTIALING_LOGIN: { ... },
  EV_CLIENT_LOGIN: { ... },
  // ... etc
}
```

#### Rate Limit

- **Limit**: 5 requests per minute
- **Scope**: Per user/identifier (email, username, or IP)
- **Window**: 60 seconds
- **Cooldown**: 5 minutes (with incremental cooldown)

#### Features

1. **Incremental Cooldown**: Cooldown period increases with repeated violations
   - 1st violation: 5 minutes
   - 2nd violation: 10 minutes
   - 3rd violation: 20 minutes
   - 4th violation: 30 minutes
   - 5th+ violation: 60 minutes (capped)

2. **Redis-Based Storage**: Uses Redis to track attempts across server instances

3. **Automatic Reset**: Attempts reset after the window expires or cooldown period ends

#### Usage

The rate limiting is automatically applied to authentication endpoints. The `Utils.checkRateLimit()` method is called with a unique key (typically based on email/username/IP) and the appropriate configuration.

#### Error Response

When rate limit is exceeded:

```json
{
  "message": "Too many login attempts. Please try again in X minutes and Y seconds.",
  "retryAfterSeconds": 300
}
```

HTTP Status: `429 Too Many Requests`

### Part 2: Read Operations Rate Limiting

#### Description

Read operations rate limiting controls the number of read requests (GET, HEAD, OPTIONS) that a user can make per minute. This prevents excessive data retrieval and ensures fair resource distribution.

#### Implementation

Implemented using:
- `UserRateLimitGuard` in `src/audit/guards/user-rate-limit.guard.ts`
- `RateLimitService` in `src/audit/services/rate-limit.service.ts`
- `@RateLimit('READ')` decorator for route-level configuration

#### Configuration

**Location**: `src/audit/constants.ts`

```typescript
export const RATE_LIMITS = {
    READ: {
        limit: 50,
        windowSeconds: 60,
        cooldownSeconds: 300,  // 5 minutes
    },
    // ...
}
```

#### Rate Limit

- **Limit**: 50 requests per minute per user
- **Scope**: Per authenticated user (identified by user ID from JWT token)
- **Window**: 60 seconds (rolling window)
- **Cooldown**: 5 minutes after limit violation

#### How It Works

1. The `UserRateLimitGuard` extracts the user ID from the JWT token
2. It checks if the route has the `@RateLimit('READ')` decorator
3. The `RateLimitService` uses Redis sorted sets to track requests
4. Each request timestamp is stored in a sorted set
5. Old timestamps outside the window are automatically removed
6. If the count exceeds 50, the user is blocked for the cooldown period

#### Activity Tracking

All read operations are tracked by the `AuditInterceptor`, which logs:
- User ID
- Endpoint accessed
- Request method
- Operation type (READ)
- IP address
- User agent
- Timestamp
- Response details

This tracking enables:
- User activity monitoring
- Anomaly detection
- Usage analytics
- Security auditing

#### Usage

Apply the `@RateLimit('READ')` decorator to read endpoints:

```typescript
import { RateLimit } from '@audit/decorators/rate-limit.decorator';

@Get('/patients')
@RateLimit('READ')
async getPatients() {
  // Implementation
}
```

#### Skipping Rate Limiting

To skip rate limiting for specific routes:

```typescript
import { SkipRateLimit } from '@audit/decorators/skip-rate-limit.decorator';

@Get('/public-data')
@SkipRateLimit()
async getPublicData() {
  // Implementation
}
```

#### Error Response

When rate limit is exceeded:

```json
{
  "message": "Rate limit exceeded. READ operations are limited to 50 requests per minute. Please try again in X seconds.",
  "retryAfterSeconds": 300,
  "limit": 50,
  "remaining": 0,
  "operationType": "READ"
}
```

HTTP Status: `429 Too Many Requests`

### Part 3: Write Operations Rate Limiting

#### Description

Write operations rate limiting controls the number of write requests (POST, PUT, PATCH, DELETE) that a user can make per minute. This is more restrictive than read operations to prevent data manipulation abuse.

#### Implementation

Same implementation as Read operations, but with different limits and configuration.

#### Configuration

**Location**: `src/audit/constants.ts`

```typescript
export const RATE_LIMITS = {
    WRITE: {
        limit: 10,
        windowSeconds: 60,
        cooldownSeconds: 300,  // 5 minutes
    },
    // ...
}
```

#### Rate Limit

- **Limit**: 10 requests per minute per user
- **Scope**: Per authenticated user (identified by user ID from JWT token)
- **Window**: 60 seconds (rolling window)
- **Cooldown**: 5 minutes after limit violation

#### Activity Tracking

Similar to read operations, all write operations are tracked with additional focus on:
- Data modification tracking
- Resource ID tracking
- Change volume monitoring
- Anomaly detection for suspicious write patterns

#### Usage

Apply the `@RateLimit('WRITE')` decorator to write endpoints:

```typescript
import { RateLimit } from '@audit/decorators/rate-limit.decorator';

@Post('/patients')
@RateLimit('WRITE')
async createPatient() {
  // Implementation
}

@Put('/patients/:id')
@RateLimit('WRITE')
async updatePatient() {
  // Implementation
}

@Delete('/patients/:id')
@RateLimit('WRITE')
async deletePatient() {
  // Implementation
}
```

#### Error Response

When rate limit is exceeded:

```json
{
  "message": "Rate limit exceeded. WRITE operations are limited to 10 requests per minute. Please try again in X seconds.",
  "retryAfterSeconds": 300,
  "limit": 10,
  "remaining": 0,
  "operationType": "WRITE"
}
```

HTTP Status: `429 Too Many Requests`

## Implementation Details

### Redis Storage Strategy

The rate limiting system uses Redis for distributed rate limiting across multiple server instances. The implementation uses:

1. **Sorted Sets (ZSET)**: For tracking request timestamps in rolling windows
   - Key format: `audit:rate:{READ|WRITE}:{userId}:window`
   - Score: Unix timestamp in seconds
   - Value: Timestamp string (for uniqueness)

2. **String Keys**: For tracking cooldown/block periods
   - Key format: `audit:rate:{READ|WRITE}:{userId}:blocked`
   - Value: Unix timestamp when block expires

### Rate Limit Service

The `RateLimitService` (`src/audit/services/rate-limit.service.ts`) provides:

- `checkRateLimit(userId, operationType)`: Checks if a request should be allowed
- `resetRateLimit(userId, operationType)`: Manually reset rate limits for a user
- `getRateLimitStatus(userId, operationType)`: Get current rate limit status without incrementing

### Guard Implementation

The `UserRateLimitGuard` (`src/audit/guards/user-rate-limit.guard.ts`):

1. Extracts user ID from JWT token (from `request.user.sub` or token payload)
2. Checks for `@RateLimit()` decorator on the route
3. Checks for `@SkipRateLimit()` decorator to bypass rate limiting
4. Calls `RateLimitService` to check limits
5. Throws `HttpException` with 429 status if limit exceeded

### Audit Interceptor

The `AuditInterceptor` (`src/audit/interceptors/audit.interceptor.ts`) automatically:

1. Tracks all API requests (unless `@SkipAudit()` is used)
2. Determines operation type (READ/WRITE) from decorator or HTTP method
3. Logs user activity to Redis buffer
4. Updates leaderboard metrics
5. Tracks resource IDs, response sizes, and durations

### Window Management

The system uses a rolling window approach:

1. Old timestamps outside the window are removed using `ZREMRANGEBYSCORE`
2. Current request timestamp is added using `ZADD`
3. Request count is determined using `ZCARD`
4. Window expiration is set using `EXPIRE` with a buffer (window + 60 seconds)

## Configuration

### Environment Variables

No specific environment variables are required for rate limiting. The system uses:

- Redis connection (configured in `config/cache.ts`)
- Application configuration (configured in `config/app.ts`)

### Rate Limit Constants

All rate limit configurations are defined in constants files:

1. **Throttle Config**: `src/constants.ts`
   - General IP-based throttling

2. **Auth Rate Limits**: `src/auth/constants.ts`
   - Authentication endpoint rate limits

3. **Operation Rate Limits**: `src/audit/constants.ts`
   - Read and Write operation rate limits

### Modifying Rate Limits

To modify rate limits, update the appropriate constants file:

```typescript
// For Read/Write operations
export const RATE_LIMITS = {
    READ: {
        limit: 50,        // Change this value
        windowSeconds: 60,
        cooldownSeconds: 300,
    },
    WRITE: {
        limit: 10,        // Change this value
        windowSeconds: 60,
        cooldownSeconds: 300,
    },
} as const;

// For IP-based throttling
export const THROTTLE_CONFIG = {
    GENERAL: {
        TTL: 60_000,      // Window in milliseconds
        LIMIT: 200        // Change this value
    }
}

// For Auth rate limits
export const RATE_LIMIT_CONFIG = {
  DEFAULT: {
    MAX_ATTEMPTS: 5,      // Change this value
    WINDOW_SECONDS: 60,
    COOLDOWN_SECONDS: 300,
  },
}
```

## Usage Examples

### Applying Rate Limits to Routes

#### Read Operations

```typescript
import { Controller, Get } from '@nestjs/common';
import { RateLimit } from '@audit/decorators/rate-limit.decorator';

@Controller('patients')
export class PatientController {
  @Get()
  @RateLimit('READ')
  async getAllPatients() {
    // Returns list of patients
    // Limited to 50 requests per minute per user
  }

  @Get(':id')
  @RateLimit('READ')
  async getPatientById(@Param('id') id: string) {
    // Returns single patient
    // Limited to 50 requests per minute per user
  }
}
```

#### Write Operations

```typescript
import { Controller, Post, Put, Delete } from '@nestjs/common';
import { RateLimit } from '@audit/decorators/rate-limit.decorator';

@Controller('patients')
export class PatientController {
  @Post()
  @RateLimit('WRITE')
  async createPatient(@Body() data: CreatePatientDto) {
    // Creates new patient
    // Limited to 10 requests per minute per user
  }

  @Put(':id')
  @RateLimit('WRITE')
  async updatePatient(@Param('id') id: string, @Body() data: UpdatePatientDto) {
    // Updates patient
    // Limited to 10 requests per minute per user
  }

  @Delete(':id')
  @RateLimit('WRITE')
  async deletePatient(@Param('id') id: string) {
    // Deletes patient
    // Limited to 10 requests per minute per user
  }
}
```

#### Skipping Rate Limits

```typescript
import { Controller, Get } from '@nestjs/common';
import { SkipRateLimit } from '@audit/decorators/skip-rate-limit.decorator';

@Controller('health')
export class HealthController {
  @Get()
  @SkipRateLimit()
  async healthCheck() {
    // Health check endpoint
    // No rate limiting applied
    return { status: 'ok' };
  }
}
```

### Checking Rate Limit Status

To check rate limit status programmatically:

```typescript
import { Injectable } from '@nestjs/common';
import { RateLimitService } from '@audit/services/rate-limit.service';

@Injectable()
export class SomeService {
  constructor(private readonly rateLimitService: RateLimitService) {}

  async checkUserRateLimit(userId: string) {
    const readStatus = await this.rateLimitService.getRateLimitStatus(userId, 'READ');
    const writeStatus = await this.rateLimitService.getRateLimitStatus(userId, 'WRITE');

    return {
      read: {
        allowed: readStatus.allowed,
        remaining: readStatus.remaining,
        limit: readStatus.limit,
        resetAt: new Date(readStatus.resetAt * 1000),
      },
      write: {
        allowed: writeStatus.allowed,
        remaining: writeStatus.remaining,
        limit: writeStatus.limit,
        resetAt: new Date(writeStatus.resetAt * 1000),
      },
    };
  }
}
```

### Resetting Rate Limits

To manually reset rate limits (e.g., for admin operations):

```typescript
import { Injectable } from '@nestjs/common';
import { RateLimitService } from '@audit/services/rate-limit.service';

@Injectable()
export class AdminService {
  constructor(private readonly rateLimitService: RateLimitService) {}

  async resetUserRateLimit(userId: string) {
    await this.rateLimitService.resetRateLimit(userId, 'READ');
    await this.rateLimitService.resetRateLimit(userId, 'WRITE');
  }
}
```

## Monitoring and Logging

### Audit Logging

All API requests are automatically logged by the `AuditInterceptor` with the following information:

- **User Information**: User ID, user type, user role
- **Request Details**: Endpoint, HTTP method, operation type
- **Request Data**: Query parameters, path parameters, request body (sanitized)
- **Response Details**: Status code, response size, record count
- **Resource Information**: Resource IDs accessed
- **Network Information**: IP address, user agent
- **Performance**: Request duration
- **Error Information**: Error messages (if applicable)

### Log Storage

Logs are stored in:

1. **Redis Buffer**: Temporary storage for batching
2. **PostgreSQL Database**: Permanent storage in `api_access_log` table
3. **Leaderboard Metrics**: Aggregated metrics for analytics

### Log Format

Log entries follow this structure:

```typescript
{
  userId: string;
  userTableType: string;
  userRole: string;
  endpoint: string;
  method: string;
  operationType: 'READ' | 'WRITE';
  module: string;
  queryParams?: Record<string, any>;
  pathParams?: Record<string, any>;
  body?: Record<string, any>;
  responseCode: number;
  responseSize?: number;
  recordCount?: number;
  resourceIds?: number[];
  hasError: boolean;
  errorMessage?: string;
  ipAddress?: string;
  userAgent?: string;
  duration?: number;
  accessedAt: Date;
}
```

### Rate Limit Violations

When rate limits are exceeded, the following information is logged:

1. **Guard Level**: `UserRateLimitGuard` logs violations
2. **Service Level**: `RateLimitService` tracks blocked users
3. **Audit Log**: Full request details are logged even for blocked requests

### Monitoring Metrics

The system tracks:

- **User Activity**: Per-user request counts and patterns
- **Rate Limit Violations**: Frequency and distribution
- **Resource Usage**: Most accessed endpoints and resources
- **Anomaly Detection**: Unusual patterns in user behavior
- **Leaderboard**: Top users by activity (daily/hourly)

## Error Handling

### Rate Limit Exceeded Responses

All rate limit violations return consistent error responses:

#### HTTP Status Code

`429 Too Many Requests`

#### Response Body Structure

```json
{
  "message": "Rate limit exceeded. {OPERATION_TYPE} operations are limited to {LIMIT} requests per minute. Please try again in {RETRY_AFTER} seconds.",
  "retryAfterSeconds": 300,
  "limit": 50,
  "remaining": 0,
  "operationType": "READ"
}
```

#### Response Headers

Standard rate limit headers (if implemented):

```
X-RateLimit-Limit: 50
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1234567890
Retry-After: 300
```

### Error Handling in Code

The rate limiting system handles errors gracefully:

1. **Redis Connection Errors**: If Redis is unavailable, the system allows requests (fail-open) and logs the error
2. **Token Parsing Errors**: If JWT token cannot be parsed, rate limiting is skipped (fail-open)
3. **Missing User ID**: If user ID cannot be extracted, rate limiting is skipped (fail-open)

### Client-Side Handling

Clients should:

1. Check for `429` status code
2. Read `retryAfterSeconds` from response body
3. Implement exponential backoff for retries
4. Display user-friendly error messages
5. Respect the `Retry-After` header if present

Example client-side handling:

```typescript
async function makeRequest(url: string) {
  try {
    const response = await fetch(url);
    if (response.status === 429) {
      const error = await response.json();
      const retryAfter = error.retryAfterSeconds || 60;
      console.log(`Rate limited. Retry after ${retryAfter} seconds`);
      // Implement retry logic with backoff
      return;
    }
    return response.json();
  } catch (error) {
    // Handle other errors
  }
}
```

## Best Practices

### When to Apply Rate Limits

1. **Apply to all user-facing endpoints**: Protect against abuse
2. **Use READ limits for data retrieval**: Prevent excessive data scraping
3. **Use WRITE limits for mutations**: Protect against data manipulation abuse
4. **Skip for health checks**: Ensure monitoring systems can always check health
5. **Skip for public endpoints**: If appropriate for your use case

### Rate Limit Values

Consider these factors when setting rate limits:

1. **Normal Usage Patterns**: Analyze typical user behavior
2. **Resource Costs**: Consider database and API costs
3. **Business Requirements**: Balance security with usability
4. **User Experience**: Ensure limits don't frustrate legitimate users
5. **System Capacity**: Ensure limits align with infrastructure capacity

### Monitoring Recommendations

1. **Track Violation Rates**: Monitor how often limits are hit
2. **Analyze Patterns**: Identify legitimate vs. malicious traffic
3. **Adjust Limits**: Periodically review and adjust based on data
4. **Alert on Anomalies**: Set up alerts for unusual patterns
5. **Review Logs**: Regularly review audit logs for security issues

## Troubleshooting

### Common Issues

#### Rate Limits Too Restrictive

**Symptoms**: Legitimate users frequently hitting rate limits

**Solutions**:
1. Review usage patterns in audit logs
2. Increase limits in configuration files
3. Consider different limits for different user roles
4. Implement whitelisting for trusted users

#### Rate Limits Not Working

**Symptoms**: Requests not being rate limited

**Solutions**:
1. Verify `@RateLimit()` decorator is applied
2. Check that `UserRateLimitGuard` is registered globally
3. Verify Redis connection is working
4. Check that user ID is being extracted correctly
5. Review application logs for errors

#### Redis Connection Issues

**Symptoms**: Rate limiting fails or allows all requests

**Solutions**:
1. Verify Redis is running and accessible
2. Check Redis connection configuration
3. Review connection pool settings
4. Monitor Redis memory usage
5. Implement Redis failover if using cluster

### Debugging

Enable debug logging to troubleshoot rate limiting:

```typescript
// In rate-limit.service.ts
console.log('Rate limit check:', {
  userId,
  operationType,
  result,
  requestCount,
  windowKey,
});
```

## Conclusion

The multi-layered rate limiting system provides comprehensive protection for the Capline backend services. By combining edge-level protection (Cloudflare), IP-based throttling, and user-based rate limiting, the system ensures robust defense against various attack vectors while maintaining good user experience for legitimate users.

Regular monitoring, log analysis, and configuration adjustments ensure the system continues to meet security and performance requirements as the application evolves.

