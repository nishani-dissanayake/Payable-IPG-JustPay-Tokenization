# Payable JustPay Tokenized Payments Guide

This documentation is for the integration of JustPay tokenized payments into your website.

---

## Test Bank Accounts for Sandbox Testing

**Test Bank Name:** Test Bank  
**Test Bank Accounts:** 123456, 1234567, 12345678  
**Test OTP:** 1234  

## Additional Tokenization Features

- **Live Base URL**: `POST https://ipgpayment.payable.lk`
- **Sandbox Base URL**: `POST https://sandboxipgpayment.payable.lk`

### 1. List Saved Accounts

Retrieve all saved accounts for a customer.

**Endpoint**: `POST {{baseUrl}}/ipg/banking/tokenize/listAccounts`

**Headers Required**:

```
Content-Type: application/json
```

**Required Parameters**:

- `merchantId` - Your merchant ID
- `customerId` - Customer ID
- `checkValue` - Security hash

**CheckValue Generation:**

```
UPPERCASE(SHA512[merchantId|customerId|UPPERCASE(SHA512[merchantToken])])
```

### 2. Delete Saved Account

Remove a saved account from the customer's account.

**Endpoint**: `POST {{baseUrl}}/ipg/banking/tokenize/deleteAccount`

**Headers Required**:

```
Content-Type: application/json
```

**Required Parameters**:

- `merchantId` - Your merchant ID
- `customerId` - Customer ID
- `tokenId` - Token ID to delete
- `checkValue` - Security hash

**CheckValue Generation:**

```
UPPERCASE(SHA512[merchantId|customerId|tokenId|UPPERCASE(SHA512[merchantToken])])
```

### 3. Edit Saved Account

Update the account nickname or set as default account.

**Endpoint**: `POST {{baseUrl}}/ipg/banking/tokenize/editAccount`

**Headers Required**:

```
Content-Type: application/json
Authorization: Bearer {access_token}
```

**How to Generate JWT Access Token**:

1. Create Basic Auth token: `base64(businessKey:businessToken)`
2. POST to `{{baseUrl}}/ipg/auth/banking` with headers:
   ```
   Content-Type: application/json
   Authorization: {basicAuthToken}
   ```
   Body: `{"grant_type": "client_credentials"}`
3. Extract `accessToken` from response and use in Authorization header

**Required Parameters**:

- `customerId` - Get it from 1st callback
- `tokenId` - Get it from list card API
- `nickName` - Optional nickname for the card
- `isDefaultCard` - Set as default card (0 or 1)
- `checkValue` - Security hash

**CheckValue Generation:**

```
UPPERCASE(SHA512[merchantId|customerId|tokenId|UPPERCASE(SHA512[merchantToken])])
```

### 4. Pay with Saved Account

Process payment using a previously saved account token.

**Endpoint**: `POST {{baseUrl}}/ipg/banking/tokenize/pay`

**Headers Required**:

```
Content-Type: application/json
Authorization: Bearer {access_token}
```

**How to Generate Access Token**:

1. Create Basic Auth token: `base64(businessKey:businessToken)`
2. POST to `{{baseUrl}}/ipg/auth/banking` with headers:
   ```
   Content-Type: application/json
   Authorization: {basicAuthToken}
   ```
   Body: `{"grant_type": "client_credentials"}`
3. Extract `accessToken` from response and use in Authorization header

**Required Parameters**:

- `merchantId` - Get it from 1st callback
- `customerId` - Get it from 1st callback
- `tokenId` - Get it from list card API
- `invoiceId` - Invoice ID
- `amount` - Payment amount
- `currencyCode` - Currency code (Ex. LKR)
- `checkValue` - Security hash
- `webhookUrl` - Webhook URL for notifications (https://yoursite.com/webhook/payment)

**Optional Parameters**:

- `custom1` - Custom field 1 for merchant-specific data
- `custom2` - Custom field 2 for merchant-specific data

**CheckValue Generation:**

```
UPPERCASE(SHA512[merchantId|invoiceId|amount|currencyCode|customerId|tokenId|UPPERCASE(SHA512[merchantToken])])
```

---

**3.2.** Payment related Error details.

This is the sample validation error json:

```json
{
    "status": 400,
    "errors": {
        "amount": [
            "Amount is a required field."
        ]
    }
}
```

Other common error (status can be 400/500/any other):

```json
{
    "status": 404,
    "error": "Invalid authentication"
}
```

---

#### Listening to Payment Notification Data

Payable Payment Gateway will send back to your website notifies the payment status to the `notifyUrl`. You need to get the request and send the response.

- It cannot test the payment notification by print/echo methods since `notifyUrl` never loads to the browser as it's a server callback. You can only test it by updating your database upon fetching the notification.
- It cannot test the payment notification on localhost. You need to submit a publicly accessible IP or domain based URL as your `notifyUrl` is to directly notify your server.

##### Server callback Json for Pay with Saved Account

```json
{
  merchantKey: 'YOUR_MERCHANT_KEY',
  statusCode: 1,
  payableTransactionId: 'XXXXXXXX-XXX-XXXX-XXXX-XXXXXXXXX',
  paymentMethod: 3,
  payableOrderId: 'oid-XXXXXXXX-XXX-XXXX-XXXX-XXXX',
  invoiceNo: 'YOUR_INVOICE_ID',
  payableAmount: '1000.00',
  payableCurrency: 'LKR',
  statusMessage: 'SUCCESS',
  paymentType: 2,
  paymentScheme: 'JUSTPAY',
  paymentId: 'XXXXXXXX-XXX-XXXX-XXXX-XXXXXXXXX',
  custom1: "YOUR_CUSTOM_1",
  custom2: "YOUR_CUSTOM_2",
  checkValue: YOUR_CHECK_VALUE,
  accountHolderName: 'ACCOUNT_HOLDER_NAME',
  accountNumber: '1xxx23',
  customerRefNo: 'YOUR_CUSTOMER_REF_NO',
  merchantId: 'XXXXXXXX-XXX-XXXX-XXXX-XXXXXXXXX',
  customerId: 'XXXXXXXX-XXX-XXXX-XXXX-XXXXXXXXX'
}
```

##### Notes

- `payableOrderId` - Unique Order Id generated by PAYable
- `payableTransactionId` - Unique Transaction Reference Id generated by PAYable for the processed payment
- `payableAmount` - Total amount of the Payment
- `payableCurrency` - Currency Code of the Payment (LKR Only)
- `invoiceNo` - Unique ID sent by the merchant from the checkout page
- `statusMessage` - Message received from payment gateway for transactions (SUCCESS/FAILURE)
- `paymentScheme` - Payment scheme selected by the customer (VISA / MASTERCARD)
- `accountHolderName` - Name on the account
- `accountNumber` - Masked account number

##### Send response to callback after validating checkValue

```json
{
  "Status": 200
}
```

---

## Security

### Important Security Notes

- **merchantToken** is a **secret** shared by PAYable for your merchant
- Use exact field order and string concatenation with `|` as the delimiter
- Always validate webhook responses using checkValue
- Store tokens securely in your database
- Use HTTPS for all communications
- Never expose your merchantToken in client-side code

### CheckValue Validation

**Formula for Webhook Validation (One-Time Payment):**

```
UPPERCASE(SHA512[merchantKey|payableOrderId|payableTransactionId|payableAmount|currencyCode|invoiceNo|statusCode|UPPERCASE(SHA512[merchantToken])])
```

**Formula for Tokenize Payment Webhook Validation:**

```
UPPERCASE(SHA512[merchantKey|payableOrderId|payableTransactionId|payableAmount|currencyCode|invoiceNo|statusCode|customerRefNo|UPPERCASE(SHA512[merchantToken])])
```

```javascript
// Validate webhook response
function validateWebhook(webhookData, merchantToken) {
  const calculatedCheckValue = CryptoJS.SHA512(
    webhookData.merchantKey +
      "|" +
      webhookData.payableOrderId +
      "|" +
      webhookData.payableTransactionId +
      "|" +
      webhookData.payableAmount +
      "|" +
      webhookData.payableCurrency +
      "|" +
      webhookData.invoiceNo +
      "|" +
      webhookData.statusCode +
      "|" +
      CryptoJS.SHA512(merchantToken).toString().toUpperCase()
  )
    .toString()
    .toUpperCase();

  return calculatedCheckValue === webhookData.checkValue;
}
```
