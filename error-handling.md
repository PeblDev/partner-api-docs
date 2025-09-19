# Error Handling

## Common Error Responses

### Validation Errors

```json
{
  "success": false,
  "error": "Validation failed with 2 error(s)",
  "retryable": true,
  "validationErrors": [
    {
      "field": "business.abn",
      "message": "ABN must be 11 digits",
      "suggestion": "Please provide a valid 11-digit ABN"
    },
    {
      "field": "business.entityType",
      "message": "Entity type is required",
      "suggestion": "Please provide a valid entity type: company, individual, or non_profit",
      "example": "company"
    }
  ],
  "debug": {
    "validationTimeMs": 150,
    "totalErrors": 2
  }
}
```

### Order-Specific Errors

**Entity Not Ready for Orders**

```json
{
  "success": false,
  "error": "Entity not fully activated - complete KYC and Stripe setup first",
  "retryable": false
}
```

**Order Validation Error**

```json
{
  "success": false,
  "error": "Validation failed",
  "retryable": true,
  "validationErrors": [
    {
      "field": "total",
      "message": "Total must be a positive number greater than 0",
      "suggestion": "Please provide a valid total amount"
    }
  ]
}
```

### KYC Required (Step Failure)

```json
{
  "success": false,
  "error": "Franchise setup failed",
  "details": "Some steps failed: check-kyc-completion",
  "retryable": true,
  "kycUrls": [
    {
      "directorName": "John Smith",
      "verificationUrl": "https://<environment>-pebl.web.app/api/identity/verify/<session-id>",
      "percentOwnership": 50,
      "kycStatus": "pending"
    }
  ],
  "totalOwnershipPercentage": 100,
  "verifiedOwnershipPercentage": 0,
  "kycRequired": true
}
```

### System Errors

```json
{
  "success": false,
  "error": "Internal server error during franchise setup",
  "details": "An unexpected error occurred while processing your request. Please check your request format and try again.",
  "retryable": true,
  "validationErrors": [],
  "debug": {
    "processingTimeMs": 5000,
    "totalErrors": 1,
    "errorType": "Error",
    "errorMessage": "Service temporarily unavailable"
  }
}
```

### Basic Request Errors

**Missing Request Body**

```json
{
  "success": false,
  "error": "Request body is required",
  "details": "Please provide a JSON payload with the franchise setup data",
  "retryable": false
}
```

**Missing ABN**

```json
{
  "success": false,
  "error": "ABN is required",
  "details": "Please provide a valid ABN in the business object",
  "example": {
    "business": {
      "abn": "12345678901",
      "entityType": "company"
    }
  },
  "retryable": false
}
```

### Authentication & Authorization Errors

**Inactive Partner Account**

```json
{
  "success": false,
  "error": "Partner account is inactive",
  "details": "Your partner account must be active to create franchises",
  "retryable": false
}
```

**Invalid API Key**

```json
{
  "success": false,
  "error": "Unauthorized - Invalid API key",
  "retryable": false
}
```

## Error Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | Always `false` for errors |
| `error` | string | Human-readable error message |
| `details` | string | Additional error context (for some errors) |
| `retryable` | boolean | Whether the request can be retried |
| `validationErrors` | array | Detailed validation errors (if applicable) |
| `kycUrls` | array | KYC verification URLs (if KYC step failed) |
| `totalOwnershipPercentage` | number | Total ownership percentage (if KYC step failed) |
| `verifiedOwnershipPercentage` | number | Verified ownership percentage (if KYC step failed) |
| `kycRequired` | boolean | Whether KYC is required (if KYC step failed) |
| `debug` | object | Debug information for troubleshooting |

## Error Handling Best Practices

### Retry Logic

1. **Check `retryable` field**: Only retry if `retryable: true`
2. **Exponential backoff**: Wait progressively longer between retries
3. **Maximum retry attempts**: Set a reasonable limit (e.g., 3-5 retries)
4. **Handle different error types**: Different errors may require different retry strategies

### Example Retry Implementation

```javascript
async function createEntityWithRetry(payload, maxRetries = 3) {
  let lastError;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await createEntity(payload);
      
      if (response.success) {
        return response;
      }
      
      // Check if we should retry
      if (!response.retryable) {
        throw new Error(`Non-retryable error: ${response.error}`);
      }
      
      lastError = response.error;
      
      // Wait before retrying (exponential backoff)
      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000; // 2s, 4s, 8s
        await sleep(delay);
      }
      
    } catch (error) {
      lastError = error.message;
      
      if (attempt === maxRetries) {
        throw error;
      }
      
      // Wait before retrying
      const delay = Math.pow(2, attempt) * 1000;
      await sleep(delay);
    }
  }
  
  throw new Error(`Max retries exceeded. Last error: ${lastError}`);
}
```

### KYC Error Handling

When KYC verification is required:

1. **Extract KYC URLs**: Get verification URLs from the error response
2. **Distribute URLs**: Send URLs to directors for verification
3. **Poll for completion**: Check status periodically
4. **Retry when ready**: Retry the original request once KYC is complete

```javascript
async function handleKYCRequired(response, originalPayload) {
  if (response.kycRequired && response.kycUrls) {
    console.log('KYC verification required for directors:');
    
    response.kycUrls.forEach(director => {
      console.log(`${director.directorName}: ${director.verificationUrl}`);
    });
    
    // Poll for KYC completion
    let kycComplete = false;
    let attempts = 0;
    const maxAttempts = 60; // 10 minutes with 10-second intervals
    
    while (!kycComplete && attempts < maxAttempts) {
      await sleep(10000); // Wait 10 seconds
      
      // Retry the original request to check KYC status
      const retryResponse = await createEntity(originalPayload);
      
      if (retryResponse.success) {
        kycComplete = true;
        return retryResponse;
      }
      
      if (!retryResponse.kycRequired) {
        // KYC is no longer required, but there's another error
        throw new Error(`KYC completed but setup failed: ${retryResponse.error}`);
      }
      
      attempts++;
    }
    
    if (!kycComplete) {
      throw new Error('KYC verification timeout');
    }
  }
}
```

## Troubleshooting

### Common Issues

1. **"Entity not fully activated"**
   - Ensure KYC verification is complete for all required directors
   - Check that Stripe account is active
   - Verify entity status using GET endpoint

2. **"Validation failed"**
   - Check all required fields are present
   - Verify field formats (e.g., ABN must be 11 digits)
   - Review validation error details for specific issues

3. **"Unauthorized - Invalid API key"**
   - Verify API key is correct
   - Check that API key is included in request headers
   - Contact support if key appears correct

4. **"Partner account is inactive"**
   - Contact your Pebl representative to activate account
   - Verify account status

### Debug Information

The `debug` object in error responses provides useful information:

- `processingTimeMs`: How long the request took to process
- `totalErrors`: Number of validation errors
- `cachedResultUsed`: Whether a cached result was returned
- `validationTimeMs`: Time spent on validation
- `errorType` and `errorMessage`: Technical error details
