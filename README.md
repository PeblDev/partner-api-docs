# Partner API Documentation

## Overview

The Partner API provides programmatic access to Pebl's franchise setup process. This API allows partners to create and manage franchise entities, handle KYC verification, activate payment processing accounts, and create orders for their entities.

## Base URLs

- **Staging**: `https://stage-pebl.web.app`
- **Production**: `https://prod-pebl.web.app`

## Authentication

All API requests require authentication using an API key. Contact your Pebl representative for API credentials.

### Authentication Headers

Include your API key in the `x-api-key` header:

```bash
x-api-key: your-api-key-here
```

Alternatively, you can use the `Authorization` header with Bearer format:

```bash
Authorization: Bearer your-api-key-here
```

## API Endpoints

### Entity Management
- **[Entity API](./entity-api.md)** - Create and manage franchise entities
- **[Order API](./order-api.md)** - Create and manage orders for entities

## Sample Request

Here's an example of how to make a request with authentication:

```bash
curl -X POST "https://stage-pebl.web.app/api/partner/entity" \
  -H "x-api-key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "business": {
      "abn": "12345678901",
      "entityType": "company",
      "entityStructure": "proprietary_limited",
      "mcc": "1520",
      "website": "https://example.com",
      "logoUrl": "https://example.com/logo.png"
    },
    "address": {
      "line1": "123 Business St",
      "line2": "",
      "city": "Sydney",
      "state": "NSW",
      "postcode": "2000",
      "country": "Australia",
      "countryCode": "AU"
    },
    "branch": {
      "name": "Sydney Branch",
      "contact": {
        "email": "jane@business.com",
        "phone": "+61412345678"
      }
    },
    "bankAccount": {
      "accountName": "Business Name",
      "accountNumber": "123456789",
      "bsb": "012-345"
    },
    "directors": [
      {
        "firstName": "John",
        "lastName": "Smith",
        "email": "john@business.com",
        "phone": "+61412345679",
        "percentOwnership": 50,
        "dateOfBirth": {
          "day": 1,
          "month": 1,
          "year": 1980
        },
        "address": {
          "line1": "789 Director St",
          "line2": "",
          "city": "Sydney",
          "state": "NSW",
          "postcode": "2000",
          "country": "Australia",
          "countryCode": "AU"
        },
        "relationship": "director"
      }
    ]
  }'
```

## Quick Start

1. **Create Entity**: Use the Entity API to set up a new franchise
2. **Complete KYC**: Directors complete identity verification
3. **Create Orders**: Use the Order API to generate payment links

## Key Features

- **Idempotent Operations**: Safe to retry requests
- **Multi-step Process**: Handles complex franchise setup automatically
- **KYC Integration**: Built-in identity verification
- **Order Management**: Create payment links for invoices

## Documentation

- [Entity API](./entity-api.md) - Complete entity management documentation
- [Order API](./order-api.md) - Order creation and management
- [Integration Examples](./integration-examples.md) - Code examples and workflows
- [Error Handling](./error-handling.md) - Common errors and troubleshooting

## Support

For technical support or questions about integration, please contact your Pebl representative.
