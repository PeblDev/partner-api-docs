# Order API

## Overview

The Order API handles order creation and management for existing franchise entities. Orders generate payment links that customers can use to complete payments.

## Endpoints

### Create Order

**POST** `/api/partner/entity/{abn}/order/{invoiceId}`

Creates a new order for an existing entity. The entity must be fully activated (KYC complete and Stripe account active) before orders can be created.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `abn` | string | 11-digit Australian Business Number of the entity |
| `invoiceId` | string | Unique invoice identifier (used for reconciliation) |

#### Request Body

```json
{
  "total": 5000,
  "notes": "Order for invoice INV-001",
  "reconciliationID": "INV-001"
}
```

#### Request Field Requirements

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `total` | number | ✅ | Order total amount in cents (minimum 50 cents) |
| `notes` | string | ❌ | Optional order notes or description |
| `reconciliationID` | string | ❌ | External reference ID for reconciliation (defaults to invoiceId) |

#### Response Examples

**Successful Order Creation**

```json
{
  "success": true,
  "paymentLink": "m/Pennii/payments/entity-123/order/order-789",
  "orderStatus": "pending"
}
```

**Entity Not Ready for Orders**

```json
{
  "success": false,
  "error": "Entity not fully activated - complete KYC and Stripe setup first",
  "retryable": false
}
```

**Validation Error**

```json
{
  "success": false,
  "error": "Validation failed",
  "retryable": true,
  "validationErrors": [
    {
      "field": "total",
      "message": "Total must be a positive number",
      "suggestion": "Please provide a valid total amount greater than 0"
    }
  ]
}
```

### Get Order

**GET** `/api/partner/entity/{abn}/order/{invoiceId}`

Retrieves the current status of an existing order.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `abn` | string | 11-digit Australian Business Number of the entity |
| `invoiceId` | string | Unique invoice identifier |

#### Response Examples

**Order Found**

```json
{
  "success": true,
  "order": {
    "id": "order-789",
    "entityID": "entity-123",
    "branchID": "branch-456",
    "total": 5000,
    "orderStatus": "pending",
    "notes": "Order for invoice INV-001",
    "reconciliationID": "INV-001",
    "createdAt": "2024-01-15T10:30:00.000Z"
  }
}
```

**Order Not Found**

```json
{
  "success": false,
  "error": "Order not found",
  "retryable": false
}
```

## Order Management

### Order Lifecycle

Orders created through the Partner API follow the standard Pebl order lifecycle:

1. **Pending**: Order is created and ready for payment
2. **Completed**: Payment has been received
3. **Canceled**: Order was cancelled
4. **Failed**: Payment processing failed

### Order Types

Partner orders are created as "standard" orders, which means:
- They don't contain specific product items
- They're designed for invoice-based payments
- They support the full payment flow including Stripe integration

### Payment Processing

Once an order is created:
1. A payment link is generated in the format: `m/Pennii/payments/{entityId}/order/{orderId}`
2. Customers can use this link to complete payment
3. Orders automatically transition to "completed" status upon successful payment
4. Payment is processed through the entity's activated Stripe account

### Reconciliation

Orders include a `reconciliationID` field that:
- Defaults to the `invoiceId` parameter if not specified
- Allows partners to link orders to their internal invoice systems
- Provides audit trail for payment reconciliation

### Order Constraints

- **Entity Activation Required**: Entity must be fully activated (KYC complete, Stripe active)
- **Minimum Amount**: Orders must be at least 50 cents ($0.50)
- **Immutable After Creation**: Orders cannot be modified once created
- **Unique Invoice IDs**: Each invoice ID should be unique within an entity

## Idempotency

Order operations are **idempotent** using the invoice ID as the key:

- You can safely retry requests with the same invoice ID
- No duplicate orders will be created
- The API will return the current state of the order
- Orders can be updated while pending, but not after payment

### Idempotency

Order operations are idempotent using the invoice ID as the key. This means:

- You can safely retry requests with the same invoice ID
- No duplicate orders will be created
- Orders can be updated while pending, but not after payment
