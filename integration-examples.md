# Integration Examples

## Basic Integration

### Create Franchise

```javascript
async function createFranchise(franchiseData) {
  const payload = {
    business: {
      abn: franchiseData.abn,
      entityType: 'company',
      entityStructure: franchiseData.entityStructure,
      mcc: franchiseData.mcc || '1520',
      website: franchiseData.website,
      logoUrl: franchiseData.logoUrl
    },
    address: franchiseData.address,
    branch: franchiseData.branch,
    bankAccount: franchiseData.bankAccount,
    directors: franchiseData.directors
  };
  
  let response = await fetch('/api/partner/entity', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': apiKey
    },
    body: JSON.stringify(payload)
  });
  
  let result = await response.json();
  
  // Handle retryable errors
  if (!result.success && result.retryable) {
    // Wait and retry
    await sleep(2000);
    
    response = await fetch('/api/partner/entity', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': apiKey
      },
      body: JSON.stringify(payload)
    });
    
    result = await response.json();
  }
  
  return result;
}
```

### Create Order

```javascript
async function createOrder(abn, invoiceId, orderData) {
  const payload = {
    total: orderData.total,
    notes: orderData.notes || '',
    reconciliationID: orderData.reconciliationID || invoiceId
  };
  
  const response = await fetch(`/api/partner/entity/${abn}/order/${invoiceId}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': apiKey
    },
    body: JSON.stringify(payload)
  });
  
  const result = await response.json();
  
  if (result.success) {
    return {
      orderId: result.paymentLink.split('/').pop(),
      paymentLink: result.paymentLink,
      status: result.orderStatus
    };
  } else {
    throw new Error(result.error);
  }
}

// Example usage
const order = await createOrder('12345678901', 'INV-001', {
  total: 5000, // $50.00
  notes: 'Payment for consulting services',
  reconciliationID: 'INV-001'
});

console.log('Payment link:', order.paymentLink);
```

## Complete Workflow Example

### Setup Franchise and Create Order

```javascript
async function setupFranchiseAndCreateOrder(franchiseData, orderData) {
  try {
    // Step 1: Create franchise entity
    const franchise = await createFranchise(franchiseData);
    
    if (!franchise.success) {
      throw new Error(`Franchise setup failed: ${franchise.error}`);
    }
    
    // Step 2: Wait for activation (poll status)
    let entityReady = false;
    let attempts = 0;
    const maxAttempts = 30; // 5 minutes with 10-second intervals
    
    while (!entityReady && attempts < maxAttempts) {
      await sleep(10000); // Wait 10 seconds
      
      const status = await getEntityStatus(franchise.entity.id);
      if (status.status === 'active') {
        entityReady = true;
      }
      
      attempts++;
    }
    
    if (!entityReady) {
      throw new Error('Entity activation timeout');
    }
    
    // Step 3: Create order
    const order = await createOrder(franchise.entity.abn, orderData.invoiceId, orderData);
    
    return {
      franchise,
      order
    };
    
  } catch (error) {
    console.error('Workflow failed:', error);
    throw error;
  }
}
```

## Advanced Error Handling

### Retry with Exponential Backoff

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

## Sample Data

### Complete Franchise Setup

```javascript
const franchiseData = {
  abn: "12345678901",
  entityStructure: "proprietary_limited",
  mcc: "1520",
  website: "https://example.com",
  logoUrl: "https://example.com/logo.png",
  address: {
    line1: "123 Business St",
    line2: "",
    city: "Sydney",
    state: "NSW",
    postcode: "2000",
    country: "Australia",
    countryCode: "AU"
  },
  branch: {
    name: "Sydney Branch",
    contact: {
      email: "jane@business.com",
      phone: "+61412345678"
    }
  },
  bankAccount: {
    accountName: "Business Name",
    accountNumber: "123456789",
    bsb: "012-345"
  },
  directors: [
    {
      firstName: "John",
      lastName: "Smith",
      email: "john@business.com",
      phone: "+61412345679",
      percentOwnership: 50,
      dateOfBirth: {
        day: 1,
        month: 1,
        year: 1980
      },
      address: {
        line1: "789 Director St",
        line2: "",
        city: "Sydney",
        state: "NSW",
        postcode: "2000",
        country: "Australia",
        countryCode: "AU"
      },
      relationship: "director"
    }
  ]
};
```

### Order Creation

```javascript
const orderData = {
  invoiceId: "INV-001",
  total: 5000, // $50.00 in cents
  notes: "Payment for consulting services",
  reconciliationID: "INV-001"
};
```

## Utility Functions

### Sleep Function

```javascript
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

### Get Entity Status

```javascript
async function getEntityStatus(entityId) {
  const response = await fetch(`/api/partner/entity/${entityId}`, {
    method: 'GET',
    headers: {
      'x-api-key': apiKey
    }
  });
  
  return await response.json();
}
```

## Environment Configuration

```javascript
const config = {
  staging: {
    baseUrl: 'https://<environment>-pebl.web.app',
    apiKey: 'your-api-key-here'
  },
  production: {
    baseUrl: 'https://<environment>-pebl.web.app',
    apiKey: 'your-api-key-here'
  }
};

// Use environment-specific configuration
const environment = process.env.NODE_ENV === 'production' ? 'production' : 'staging';
const { baseUrl, apiKey } = config[environment];
```
