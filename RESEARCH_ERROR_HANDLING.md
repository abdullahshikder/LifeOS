# Error Handling Strategy - Life OS Integrations

## Overview

This document provides comprehensive error handling strategies for all Life OS integrations, including API error codes, rate limit handling, network failure recovery, retry patterns, and user notification strategies.

**Last Updated**: December 2024

---

## Error Categories

### 1. Network Errors
- Connection timeout
- DNS resolution failure
- SSL/TLS errors
- Network unreachable

### 2. API Errors
- Authentication errors (401, 403)
- Rate limit errors (429)
- Server errors (500, 502, 503)
- Client errors (400, 404)

### 3. Data Errors
- Invalid data format
- Missing required fields
- Data validation failures
- Conflict errors

### 4. Platform Errors
- Permission denied
- Service unavailable
- Quota exceeded
- Feature not available

---

## API Error Code Reference

### Google Calendar API

**Error Codes:**
```typescript
enum GoogleCalendarError {
  INVALID_REQUEST = 400,        // Bad request
  UNAUTHORIZED = 401,            // Invalid credentials
  FORBIDDEN = 403,               // Insufficient permissions
  NOT_FOUND = 404,               // Resource not found
  RATE_LIMIT_EXCEEDED = 429,     // Too many requests
  INTERNAL_ERROR = 500,          // Server error
  SERVICE_UNAVAILABLE = 503      // Service temporarily unavailable
}
```

**Error Handling:**
```typescript
async function handleGoogleCalendarError(error: APIError): Promise<void> {
  switch (error.status) {
    case 400:
      // Invalid request - log and show user-friendly message
      throw new UserError('Invalid calendar request. Please try again.');
      
    case 401:
      // Unauthorized - token expired, refresh or re-authenticate
      await refreshAccessToken('google_calendar');
      throw new RetryableError('Authentication expired. Retrying...');
      
    case 403:
      // Forbidden - insufficient permissions
      throw new UserError('Calendar access denied. Please grant permissions in settings.');
      
    case 404:
      // Not found - resource doesn't exist
      throw new UserError('Calendar not found. It may have been deleted.');
      
    case 429:
      // Rate limit - implement exponential backoff
      const retryAfter = error.headers['retry-after'] || 60;
      throw new RateLimitError(`Rate limit exceeded. Retrying in ${retryAfter} seconds.`, retryAfter);
      
    case 500:
    case 503:
      // Server error - retry with exponential backoff
      throw new RetryableError('Google Calendar is temporarily unavailable. Retrying...');
      
    default:
      throw new UnknownError('An unexpected error occurred.');
  }
}
```

---

### Spotify Web API

**Error Codes:**
```typescript
enum SpotifyError {
  BAD_REQUEST = 400,
  UNAUTHORIZED = 401,
  FORBIDDEN = 403,
  NOT_FOUND = 404,
  TOO_MANY_REQUESTS = 429,
  INTERNAL_SERVER_ERROR = 500,
  BAD_GATEWAY = 502,
  SERVICE_UNAVAILABLE = 503
}
```

**Rate Limits:**
- 300 requests per 30 seconds per user
- Burst limit: 300 requests

**Error Handling:**
```typescript
async function handleSpotifyError(error: APIError): Promise<void> {
  switch (error.status) {
    case 401:
      // Token expired - refresh
      await refreshAccessToken('spotify');
      throw new RetryableError('Authentication expired. Retrying...');
      
    case 403:
      // Forbidden - user needs Premium for playback
      throw new UserError('Spotify Premium required for playback. Please upgrade your account.');
      
    case 429:
      // Rate limit - wait for retry-after
      const retryAfter = error.headers['retry-after'] || 30;
      throw new RateLimitError(`Rate limit exceeded. Retrying in ${retryAfter} seconds.`, retryAfter);
      
    case 500:
    case 502:
    case 503:
      // Server error - retry
      throw new RetryableError('Spotify is temporarily unavailable. Retrying...');
      
    default:
      throw new UnknownError('An unexpected error occurred with Spotify.');
  }
}
```

---

### Strava API

**Error Codes:**
```typescript
enum StravaError {
  BAD_REQUEST = 400,
  UNAUTHORIZED = 401,
  FORBIDDEN = 403,
  NOT_FOUND = 404,
  RATE_LIMIT_EXCEEDED = 429,
  INTERNAL_ERROR = 500,
  SERVICE_UNAVAILABLE = 503
}
```

**Rate Limits:**
- 600 requests per 15 minutes
- 30,000 requests per day

**Error Handling:**
```typescript
async function handleStravaError(error: APIError): Promise<void> {
  switch (error.status) {
    case 401:
      // Unauthorized - refresh token
      await refreshAccessToken('strava');
      throw new RetryableError('Authentication expired. Retrying...');
      
    case 403:
      // Forbidden - activity is private
      throw new UserError('This activity is private and cannot be accessed.');
      
    case 429:
      // Rate limit - check X-RateLimit-Limit header
      const limit = error.headers['x-ratelimit-limit'];
      const usage = error.headers['x-ratelimit-usage'];
      const retryAfter = error.headers['retry-after'] || 900; // 15 minutes
      throw new RateLimitError(`Rate limit exceeded (${usage}/${limit}). Retrying in ${retryAfter} seconds.`, retryAfter);
      
    case 500:
    case 503:
      // Server error - retry
      throw new RetryableError('Strava is temporarily unavailable. Retrying...');
      
    default:
      throw new UnknownError('An unexpected error occurred with Strava.');
  }
}
```

---

### HealthKit (iOS)

**Error Codes:**
```typescript
enum HealthKitError {
  NOT_AVAILABLE = 1,              // HealthKit not available on device
  HEALTH_DATA_UNAVAILABLE = 2,   // Health data not available
  INVALID_ARGUMENT = 3,           // Invalid parameters
  AUTHORIZATION_DENIED = 4,        // User denied permission
  AUTHORIZATION_NOT_DETERMINED = 5 // Permission not requested yet
}
```

**Error Handling:**
```typescript
async function handleHealthKitError(error: HealthKitError): Promise<void> {
  switch (error.code) {
    case HealthKitError.NOT_AVAILABLE:
      throw new UserError('HealthKit is not available on this device.');
      
    case HealthKitError.HEALTH_DATA_UNAVAILABLE:
      throw new UserError('Health data is not available. Please check your Health app.');
      
    case HealthKitError.AUTHORIZATION_DENIED:
      throw new UserError('Health data access denied. Please enable it in Settings > Privacy > Health.');
      
    case HealthKitError.AUTHORIZATION_NOT_DETERMINED:
      // Request permission
      await requestHealthKitPermission();
      throw new RetryableError('Permission requested. Please grant access.');
      
    default:
      throw new UnknownError('An unexpected error occurred with HealthKit.');
  }
}
```

---

### Health Connect (Android)

**Error Codes:**
```typescript
enum HealthConnectError {
  UNKNOWN = 0,
  INVALID_ARGUMENT = 1,
  IO_ERROR = 2,
  SECURITY_ERROR = 3,
  UNSUPPORTED_OPERATION = 4,
  RATE_LIMIT_EXCEEDED = 5,
  INTERNAL_ERROR = 6
}
```

**Error Handling:**
```typescript
async function handleHealthConnectError(error: HealthConnectError): Promise<void> {
  switch (error.code) {
    case HealthConnectError.SECURITY_ERROR:
      throw new UserError('Health Connect permission denied. Please grant access in settings.');
      
    case HealthConnectError.RATE_LIMIT_EXCEEDED:
      throw new RateLimitError('Rate limit exceeded. Please try again later.');
      
    case HealthConnectError.IO_ERROR:
      throw new RetryableError('Health Connect is temporarily unavailable. Retrying...');
      
    default:
      throw new UnknownError('An unexpected error occurred with Health Connect.');
  }
}
```

---

## Retry Strategy with Exponential Backoff

```typescript
class RetryManager {
  async retryWithBackoff<T>(
    operation: () => Promise<T>,
    maxRetries: number = 3,
    baseDelay: number = 1000,
    maxDelay: number = 60000
  ): Promise<T> {
    let lastError: Error;
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        // Don't retry non-retryable errors
        if (error instanceof NonRetryableError) {
          throw error;
        }
        
        // Don't retry on last attempt
        if (attempt === maxRetries) {
          break;
        }
        
        // Calculate delay with exponential backoff
        const delay = Math.min(
          baseDelay * Math.pow(2, attempt),
          maxDelay
        );
        
        // Add jitter to prevent thundering herd
        const jitter = Math.random() * 0.3 * delay;
        const totalDelay = delay + jitter;
        
        console.log(`Retry attempt ${attempt + 1}/${maxRetries} after ${totalDelay}ms`);
        await this.sleep(totalDelay);
      }
    }
    
    throw lastError;
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

**Usage:**
```typescript
const retryManager = new RetryManager();

try {
  const data = await retryManager.retryWithBackoff(
    () => apiClient.fetchData(),
    3,      // max retries
    1000,   // base delay (1 second)
    60000   // max delay (60 seconds)
  );
} catch (error) {
  // Handle final error
  handleError(error);
}
```

---

## Rate Limit Handling

```typescript
class RateLimitHandler {
  private rateLimitCache: Map<string, RateLimitInfo> = new Map();
  
  async handleRateLimit(
    provider: string,
    error: RateLimitError
  ): Promise<void> {
    const key = `${provider}_rate_limit`;
    const info: RateLimitInfo = {
      provider,
      retryAfter: error.retryAfter,
      resetAt: new Date(Date.now() + error.retryAfter * 1000),
      blocked: true
    };
    
    this.rateLimitCache.set(key, info);
    
    // Schedule retry
    setTimeout(() => {
      this.rateLimitCache.delete(key);
    }, error.retryAfter * 1000);
    
    // Notify user
    this.notifyUser(`Rate limit exceeded for ${provider}. Retrying in ${error.retryAfter} seconds.`);
    
    throw error;
  }
  
  isRateLimited(provider: string): boolean {
    const key = `${provider}_rate_limit`;
    const info = this.rateLimitCache.get(key);
    return info?.blocked ?? false;
  }
  
  getRetryAfter(provider: string): number | null {
    const key = `${provider}_rate_limit`;
    const info = this.rateLimitCache.get(key);
    if (!info) return null;
    
    const remaining = info.resetAt.getTime() - Date.now();
    return remaining > 0 ? Math.ceil(remaining / 1000) : null;
  }
}
```

---

## Network Failure Recovery

```typescript
class NetworkErrorHandler {
  async handleNetworkError(error: NetworkError): Promise<void> {
    // Check network connectivity
    const isOnline = await this.checkConnectivity();
    
    if (!isOnline) {
      // Queue for offline sync
      await this.queueForOfflineSync(error.operation);
      throw new OfflineError('No internet connection. Operation queued for sync when online.');
    }
    
    // Check if error is retryable
    if (this.isRetryableError(error)) {
      throw new RetryableError('Network error. Retrying...');
    }
    
    // Non-retryable error
    throw new UserError('Network error. Please check your connection and try again.');
  }
  
  private async checkConnectivity(): Promise<boolean> {
    try {
      const response = await fetch('https://www.google.com/favicon.ico', {
        method: 'HEAD',
        mode: 'no-cors',
        cache: 'no-cache'
      });
      return true;
    } catch {
      return false;
    }
  }
  
  private isRetryableError(error: NetworkError): boolean {
    // Retryable: timeout, DNS failure, connection reset
    // Non-retryable: SSL errors, certificate errors
    return error.type === 'timeout' || 
           error.type === 'dns' || 
           error.type === 'connection_reset';
  }
}
```

---

## User Notification Strategies

### Error Notification Levels

**1. Silent Retry (No User Notification)**
- Transient errors (network timeout, temporary server error)
- Automatic retry with exponential backoff
- User not notified unless retry fails

**2. Toast Notification (Brief)**
- Rate limit warnings
- Temporary service unavailability
- Auto-dismiss after 3-5 seconds

**3. In-App Banner (Persistent)**
- Authentication errors requiring user action
- Permission denied errors
- Dismissible by user

**4. Modal Dialog (Blocking)**
- Critical errors requiring immediate attention
- Data loss prevention
- User must acknowledge

### Error Message Guidelines

**Do's:**
- Use clear, user-friendly language
- Explain what went wrong
- Suggest what user can do
- Provide actionable steps
- Use appropriate tone (not technical jargon)

**Don'ts:**
- Don't expose technical details
- Don't blame the user
- Don't use error codes in user messages
- Don't be vague
- Don't use all caps or exclamation marks excessively

### Error Message Examples

**Bad:**
```
Error 401: Unauthorized access denied
```

**Good:**
```
Your session has expired. Please sign in again to continue.
```

**Bad:**
```
API rate limit exceeded. HTTP 429.
```

**Good:**
```
Too many requests. Please wait a moment and try again.
```

**Bad:**
```
HealthKit error: Authorization denied
```

**Good:**
```
Health data access is required for this feature. Please enable it in Settings > Privacy > Health.
```

---

## Error Logging and Monitoring

```typescript
class ErrorLogger {
  async logError(error: Error, context: ErrorContext): Promise<void> {
    const errorLog: ErrorLog = {
      timestamp: new Date(),
      error: {
        message: error.message,
        stack: error.stack,
        name: error.name
      },
      context: {
        userId: context.userId,
        integration: context.integration,
        operation: context.operation,
        userAgent: navigator.userAgent,
        platform: this.getPlatform()
      },
      severity: this.getSeverity(error)
    };
    
    // Log to console in development
    if (process.env.NODE_ENV === 'development') {
      console.error('Error logged:', errorLog);
    }
    
    // Send to error tracking service (Sentry, Bugsnag, etc.)
    await this.sendToErrorTracking(errorLog);
  }
  
  private getSeverity(error: Error): 'low' | 'medium' | 'high' | 'critical' {
    if (error instanceof UserError) return 'low';
    if (error instanceof RetryableError) return 'medium';
    if (error instanceof RateLimitError) return 'medium';
    if (error instanceof AuthenticationError) return 'high';
    return 'high';
  }
}
```

---

## Error Recovery Flows

### Authentication Error Recovery

```typescript
async function recoverFromAuthError(integration: string): Promise<void> {
  try {
    // Attempt token refresh
    await refreshAccessToken(integration);
    
    // Retry operation
    await retryFailedOperation(integration);
    
    // Notify user of successful recovery
    notifyUser('Connection restored successfully.');
  } catch (error) {
    // Refresh failed, need re-authentication
    await promptReAuthentication(integration);
  }
}
```

### Rate Limit Recovery

```typescript
async function recoverFromRateLimit(integration: string, retryAfter: number): Promise<void> {
  // Show user notification
  notifyUser(`Rate limit reached. Retrying in ${retryAfter} seconds...`);
  
  // Wait for rate limit to reset
  await sleep(retryAfter * 1000);
  
  // Retry operation
  await retryFailedOperation(integration);
  
  // Notify user of successful recovery
  notifyUser('Operation completed successfully.');
}
```

### Network Error Recovery

```typescript
async function recoverFromNetworkError(operation: Operation): Promise<void> {
  // Queue operation for offline sync
  await queueForOfflineSync(operation);
  
  // Show user notification
  notifyUser('No internet connection. Operation will sync when online.');
  
  // Monitor connectivity
  const connectivityWatcher = watchConnectivity(async () => {
    // Retry when online
    await retryFailedOperation(operation);
    connectivityWatcher.stop();
  });
}
```

---

## Testing Error Scenarios

### Unit Tests

```typescript
describe('Error Handling', () => {
  it('should handle 401 authentication error', async () => {
    mockAPI.mockReject({ status: 401 });
    
    await expect(apiClient.fetchData()).rejects.toThrow(AuthenticationError);
    expect(mockRefreshToken).toHaveBeenCalled();
  });
  
  it('should handle 429 rate limit error', async () => {
    mockAPI.mockReject({ status: 429, headers: { 'retry-after': '60' } });
    
    await expect(apiClient.fetchData()).rejects.toThrow(RateLimitError);
    expect(mockRetryManager.retryWithBackoff).toHaveBeenCalled();
  });
  
  it('should retry on network timeout', async () => {
    mockAPI.mockReject(new NetworkError('timeout'));
    
    await expect(apiClient.fetchData()).rejects.toThrow(RetryableError);
    expect(mockRetryManager.retryWithBackoff).toHaveBeenCalled();
  });
});
```

---

## Error Handling Checklist

### Implementation Checklist
- [ ] Error categories defined
- [ ] API error codes documented
- [ ] Retry strategy implemented
- [ ] Rate limit handling implemented
- [ ] Network error recovery implemented
- [ ] User notification system implemented
- [ ] Error logging configured
- [ ] Error monitoring set up
- [ ] Error recovery flows tested
- [ ] User-friendly error messages written

### Testing Checklist
- [ ] Network errors tested
- [ ] API errors tested
- [ ] Rate limit errors tested
- [ ] Authentication errors tested
- [ ] Retry logic tested
- [ ] User notifications tested
- [ ] Error logging tested
- [ ] Recovery flows tested

---

## Next Steps

1. **Week 1**: Implement error handling framework
2. **Week 2**: Add error handling for MVP integrations
3. **Week 3**: Implement retry and rate limit handling
4. **Week 4**: Add user notification system
5. **Week 5**: Set up error logging and monitoring
6. **Week 6**: Test error scenarios and recovery flows

---

## Resources

- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [Exponential Backoff](https://en.wikipedia.org/wiki/Exponential_backoff)
- [Error Handling Best Practices](https://www.joyent.com/node-js/production/design/errors)
- [Rate Limiting Strategies](https://stripe.com/docs/rate-limits)

---

## Notes

- Error handling is critical for user experience
- Always provide user-friendly error messages
- Implement retry logic for transient errors
- Monitor error rates and patterns
- Update error handling as APIs evolve
- Test error scenarios regularly

