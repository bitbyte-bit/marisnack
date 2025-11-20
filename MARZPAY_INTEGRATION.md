# MarzPay Payment Integration - UGX Currency

## Overview

The Express server has been configured with complete MarzPay payment integration supporting UGX (Ugandan Shillings) as the primary currency. All payment transactions are processed exclusively in UGX.

---

## API Endpoints

### 1. Initiate Payment
**Endpoint**: `POST /api/payments/marzpay/initiate`

**Description**: Creates a payment intent with MarzPay in UGX currency.

**Request Body**:
```json
{
  "amount": 50000,           // Amount in USD (will be converted to UGX)
  "orderId": "order_12345",  // Unique order ID
  "userId": "user_67890",    // Customer user ID
  "userEmail": "user@example.com",  // (Optional) Customer email
  "userName": "John Doe"     // (Optional) Customer name
}
```

**Response (Success)**:
```json
{
  "success": true,
  "paymentId": "pay_uuid",
  "paymentIntentId": "pi_marzpay_id",
  "amount": 50000,
  "currency": "UGX",
  "orderId": "order_12345",
  "paymentUrl": "https://checkout.marzpay.com/...",
  "clientSecret": "secret_key",
  "status": "initiated",
  "message": "Payment intent created successfully"
}
```

**Response (Error)**:
```json
{
  "error": "Missing required fields: amount, orderId, userId"
}
```

**Status Codes**:
- `200`: Payment intent created successfully
- `400`: Missing or invalid required fields
- `500`: Server or API error

---

### 2. Payment Status Check
**Endpoint**: `GET /api/payments/marzpay/status/:paymentIntentId`

**Description**: Retrieves the current status of a payment from MarzPay.

**URL Parameters**:
- `paymentIntentId` (required): The payment intent ID returned from initiation

**Response (Success)**:
```json
{
  "success": true,
  "paymentIntentId": "pi_marzpay_id",
  "status": "succeeded",  // Or: processing, failed, pending
  "amount": 50000,
  "currency": "UGX",
  "orderId": "order_12345",
  "createdAt": "2025-11-20T10:30:00Z"
}
```

**Example Request**:
```bash
curl http://localhost:5505/api/payments/marzpay/status/pi_marzpay_id
```

---

### 3. Webhook Handler
**Endpoint**: `POST /api/payments/marzpay/webhook`

**Description**: Receives payment status updates from MarzPay. This endpoint should be configured in your MarzPay dashboard webhook settings.

**Webhook URL** (to register in MarzPay):
```
https://your-domain.com/api/payments/marzpay/webhook
```

**Payload**:
```json
{
  "id": "pi_marzpay_id",
  "status": "succeeded",  // succeeded, failed, processing, pending
  "order_id": "order_12345",
  "amount": 50000
}
```

**Actions Triggered**:
- Updates payment status in database
- If succeeded: Updates order status to 'Confirmed'
- Logs all webhook events for audit trail

---

## Configuration

### Environment Variables

Add these to your `.env` file:

```bash
# MarzPay API Credentials
MARZPAY_API_KEY=marz_zuCBBQnpFcokbH2a
MARZPAY_API_SECRET=bsX7IbrHcTEWnG5wYeU5brAkr46CxRwt
MARZPAY_BASE_URL=https://api.marzpay.com

# Redirect URLs
REDIRECT_URL=http://localhost:3000/index.html
API_BASE_URL=http://localhost:5505
```

**Default Values**:
- If environment variables are not set, hardcoded defaults are used
- `MARZPAY_BASE_URL`: `https://api.marzpay.com`
- `REDIRECT_URL`: `http://localhost:3000/index.html`

---

## Currency: UGX (Ugandan Shillings)

### Conversion Details

- **Display Currency**: UGX
- **Conversion**: Amounts are converted to UGX with integer rounding
- **API Format**: All API responses include `currency: "UGX"`
- **Database Storage**: Amounts stored in UGX in the payments table

### Example Conversion

```javascript
// Input: $50 USD
const amountInUGX = Math.round(50 * 1);  // 50 UGX (simplified for demo)

// In production: Use real exchange rate conversion
// const exchangeRate = getUSDtoUGXRate();
// const amountInUGX = Math.round(amount * exchangeRate);
```

---

## Database Schema

### Payments Table

The following fields are used for payment tracking:

```sql
CREATE TABLE payments (
  id TEXT PRIMARY KEY,           -- Unique payment ID
  orderId TEXT NOT NULL,         -- Associated order ID
  amount REAL NOT NULL,          -- Amount in UGX
  method TEXT,                   -- Payment method (e.g., 'marzpay', 'cod')
  status TEXT,                   -- Payment status (initiated, completed, failed, processing)
  transactionId TEXT,            -- MarzPay payment intent ID
  timestamp TEXT DEFAULT CURRENT_TIMESTAMP
);
```

### Status Values

- `initiated`: Payment intent created, awaiting customer action
- `processing`: Payment is being processed by MarzPay
- `completed`: Payment successful
- `failed`: Payment failed
- `pending`: Default status

---

## Authentication

### Basic Auth Header

All requests to MarzPay API use HTTP Basic Authentication:

```
Authorization: Basic {base64(API_KEY:API_SECRET)}
```

**Generation**:
```javascript
const authString = Buffer.from(`${MARZPAY_API_KEY}:${MARZPAY_API_SECRET}`).toString('base64');
// Header: Authorization: Basic marz_zuCBBQnpFcokbH2a:bsX7IbrHcTEWnG5wYeU5brAkr46CxRwt
```

---

## Error Handling

### Common Errors

| Error | Cause | Resolution |
|-------|-------|-----------|
| `Missing required fields` | Amount, orderId, or userId not provided | Include all required fields in request |
| `Invalid amount` | Amount <= 0 | Provide amount > 0 |
| `Failed to create payment intent` | MarzPay API error | Check credentials, API status, network |
| `Failed to save payment record` | Database error | Check database connectivity |
| `Payment intent ID is required` | Status check without paymentIntentId | Provide valid payment intent ID |

### Error Response Format

```json
{
  "error": "Error message",
  "details": "Additional error details"
}
```

---

## Integration Examples

### Frontend - Initiate Payment

```javascript
// In index.html (already implemented in confirmPayment())
async function initiatePayment(amount, orderId, userId) {
  const response = await fetch('/api/payments/marzpay/initiate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      amount: amount,
      orderId: orderId,
      userId: userId,
      userEmail: currentUser.email,
      userName: currentUser.name
    })
  });
  
  const data = await response.json();
  
  if (data.success) {
    // Redirect to payment URL or open payment modal
    window.location.href = data.paymentUrl;
  } else {
    console.error('Payment initiation failed:', data.error);
  }
}
```

### Frontend - Check Payment Status

```javascript
async function checkPaymentStatus(paymentIntentId) {
  const response = await fetch(`/api/payments/marzpay/status/${paymentIntentId}`);
  const data = await response.json();
  
  console.log('Payment Status:', data.status);
  
  if (data.status === 'succeeded') {
    console.log('Payment completed!');
  }
}
```

### Backend - Process Order After Payment

```javascript
// After payment webhook confirms success
db.run(
  'UPDATE orders SET status = ? WHERE id = ?',
  ['Confirmed', orderId],
  (err) => {
    if (!err) {
      console.log('Order status updated to Confirmed');
      // Send notification to customer
    }
  }
);
```

---

## Testing

### Test Payment Flow

1. **Initiate Payment**:
   ```bash
   curl -X POST http://localhost:5505/api/payments/marzpay/initiate \
     -H "Content-Type: application/json" \
     -d '{
       "amount": 50000,
       "orderId": "test_order_123",
       "userId": "test_user_456",
       "userEmail": "test@example.com",
       "userName": "Test User"
     }'
   ```

2. **Check Status**:
   ```bash
   curl http://localhost:5505/api/payments/marzpay/status/pi_xxxxx
   ```

3. **Simulate Webhook** (for testing):
   ```bash
   curl -X POST http://localhost:5505/api/payments/marzpay/webhook \
     -H "Content-Type: application/json" \
     -d '{
       "id": "pi_xxxxx",
       "status": "succeeded",
       "order_id": "test_order_123",
       "amount": 50000
     }'
   ```

---

## Security Considerations

### In Production

1. **Webhook Verification**: Verify webhook signatures from MarzPay
   ```javascript
   // Add signature verification
   const expectedSignature = crypto
     .hmac('sha256', WEBHOOK_SECRET, req.body)
     .digest('hex');
   if (req.headers['x-marzpay-signature'] !== expectedSignature) {
     return res.status(401).json({ error: 'Invalid signature' });
   }
   ```

2. **HTTPS Only**: All endpoints must use HTTPS
3. **Rate Limiting**: Implement rate limiting on payment endpoints
4. **CORS**: Restrict CORS to authorized domains
5. **Credentials**: Store API credentials in secure environment variables
6. **Logging**: Log all payment transactions for audit trails

### Current Implementation Notes

- API credentials currently use defaults if not set in environment
- Webhook signature verification not yet implemented (TODO for production)
- Rate limiting not yet implemented (TODO)

---

## Status Codes

| Code | Meaning |
|------|---------|
| 200 | Request successful |
| 400 | Bad request (missing/invalid fields) |
| 401 | Unauthorized (auth failed) |
| 404 | Resource not found |
| 500 | Server error |

---

## Database Queries

### Get All Payments

```sql
SELECT * FROM payments ORDER BY timestamp DESC;
```

### Get Payments by Order

```sql
SELECT * FROM payments WHERE orderId = 'order_12345';
```

### Get Payment by Intent ID

```sql
SELECT * FROM payments WHERE transactionId = 'pi_xxxxx';
```

### Get Completed Payments

```sql
SELECT * FROM payments WHERE status = 'completed' ORDER BY timestamp DESC;
```

---

## Troubleshooting

### Payment Intent Not Created

**Symptoms**: `Failed to create payment intent with MarzPay`

**Solutions**:
1. Verify MarzPay API credentials in environment variables
2. Check MarzPay API status at https://status.marzpay.com
3. Ensure network connectivity to api.marzpay.com
4. Check request payload format (amount, orderId, userId required)

### Webhook Not Triggering

**Symptoms**: Payment completed but order status not updated

**Solutions**:
1. Register webhook URL in MarzPay dashboard
2. Ensure webhook URL is publicly accessible
3. Check server logs for webhook requests
4. Verify webhook payload format matches expected structure

### Payment Status Shows "Processing" for Too Long

**Symptoms**: Payment stuck in processing state

**Solutions**:
1. Use status check endpoint to verify current status
2. Contact MarzPay support if stuck > 24 hours
3. Check MarzPay API logs for errors

---

## Next Steps

1. Configure environment variables with actual MarzPay credentials
2. Register webhook URL in MarzPay dashboard
3. Test payment flow in sandbox environment
4. Implement webhook signature verification for production
5. Add comprehensive logging and error tracking
6. Set up monitoring for failed payments
7. Create admin dashboard for payment management

---

**Last Updated**: November 20, 2025  
**MarzPay API Version**: v1  
**Currency**: UGX (Ugandan Shillings)
