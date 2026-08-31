# Payable JustPay Tokenized Payments Guide

This documentation is for the integration of JustPay tokenized payments into your website.

---

## Additional Tokenization Features

- **Live Base URL**: `POST https://ipgpayment.payable.lk`
- **Sandbox Base URL**: `POST https://sandboxipgpayment.payable.lk`

### 1. List Saved Accounts

Retrieve all saved accounts for a customer.

**Endpoint**: `POST {{baseUrl}}/ipg/v2/tokenize/listCard`

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

### 2. Delete Saved Card

Remove a saved card from customer's account.

**Endpoint**: `POST {{baseUrl}}/ipg/v2/tokenize/deleteCard`

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

### 3. Edit Saved Card

Update card nickname or set as default card.

**Endpoint**: `POST {{baseUrl}}/ipg/v2/tokenize/editCard`

**Headers Required**:

```
Content-Type: application/json
Authorization: Bearer {access_token}
```

**How to Generate JWT Access Token**:

1. Create Basic Auth token: `base64(businessKey:businessToken)`
2. POST to `{{baseUrl}}/ipg/v2/auth/tokenize` with headers:
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

### 4. Pay with Saved Card

Process payment using a previously saved card token.

**Endpoint**: `POST {{baseUrl}}/ipg/v2/tokenize/pay`

**Headers Required**:

```
Content-Type: application/json
Authorization: Bearer {access_token}
```

**How to Generate Access Token**:

1. Create Basic Auth token: `base64(businessKey:businessToken)`
2. POST to `{{baseUrl}}/ipg/v2/auth/tokenize` with headers:
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
- `currencyCode` - Currency code
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

Error will be field validation (code : 3009) and other common errors.

This is the sample validation error json:

```json
{
  "status": 3009,
  "success": false,
  "error": {
    "startDate": ["Start date should be today date."]
  }
}
```

Other common error (status can be 400/500/any other):

```json
{
  "status": 400,
  "success": false,
  "error": "Something went wrong. Please contact your merchant."
}
```

---

#### Listening to Payment Notification Data

Payable Payment Gateway will send back to your website notifies the payment status to the `notifyUrl`. You need to get the request and send the response.

- It cannot test the payment notification by print/echo methods since `notifyUrl` never loads to the browser as it's a server callback. You can only test it by updating your database upon fetching the notification.
- It cannot test the payment notification on localhost. You need to submit a publicly accessible IP or domain based URL as your `notifyUrl` is to directly notify your server.

##### Server callback Json

**For One-Time Payment (paymentType = 1):**

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "payableOrderId": "oid-XXXXXXXX-XXX-XXXX-XXXX-XXXX",
  "payableTransactionId": "XXXXXXXX-XXX-XXXX-XXXX-XXXXXXXXX",
  "payableAmount": "100.00",
  "payableCurrency": "LKR",
  "invoiceNo": "YOUR_INVOICE_ID",
  "statusCode": 1,
  "statusMessage": "SUCCESS",
  "paymentType": 1,
  "paymentMethod": 1,
  "paymentScheme": "MASTERCARD",
  "custom1": "YOUR_CUSTOM_1",
  "custom2": "YOUR_CUSTOM_2",
  "cardHolderName": "CARD_HOLDER_NAME",
  "cardNumber": "512345xxxxxx0008",
  "checkValue": "YOUR_CHECK_VALUE"
}
```

**For Tokenize Payment (paymentType = 3):**

```json
{
  "merchantKey": "YOUR_MERCHANT_KEY",
  "statusCode": 1,
  "payableTransactionId": "XXXXXXXX-XXX-XXXX-XXXX-XXXXXXXXX",
  "paymentMethod": 1,
  "payableOrderId": "oid-XXXXXXXX-XXX-XXXX-XXXX-XXXX",
  "invoiceNo": "YOUR_INVOICE_ID",
  "payableAmount": "100.00",
  "payableCurrency": "LKR",
  "statusMessage": "SUCCESS",
  "paymentType": 1,
  "paymentScheme": "MASTERCARD",
  "cardHolderName": "CARD_HOLDER_NAME",
  "cardNumber": "512345xxxxxx0008",
  "paymentId": "YOUR_PAYMENT_ID",
  "custom1": null,
  "custom2": null,
  "checkValue": "YOUR_CHECK_VALUE",
  "customerRefNo": "YOUR CUSTOMER REF NO",
  "token": {
    "tokenId": "YOUR_TOKEN_ID",
    "maskedCardNo": "512345xxxxxx0008",
    "exp": "MMYY",
    "reference": null,
    "nickname": null,
    "tokenStatus": "SUCCESS",
    "defaultCard": 0
  },
  "merchantId": "YOUR_MERCHANT_ID",
  "customerId": "YOUR_CUSTOMER_ID",
  "uid": "YOUR_UID",
  "statusIndicator": "YOUR_STATUS_INDICATOR"
}
```

##### Description

**Common fields for both payment types:**

- `payableOrderId` - Unique Order Id generated by PAYable
- `payableTransactionId` - Unique Transaction Reference Id generated by PAYable for the processed payment
- `payableAmount` - Total amount of the Payment
- `payableCurrency` - Currency Code of the Payment (LKR Only)
- `invoiceNo` - Unique Id sent by Merchant to the Checkout page
- `statusMessage` - Message received from payment gateway which the customer tried to pay(SUCCESS/FAILURE)
- `paymentType` - Payment type selected during the Checkout
  1.  CARD (SUPPORTED)
  2.  BANKING (Not implemented yet)
  3.  WALLET (Not implemented yet)
- `paymentMethod` - Payment method selected during the Checkout
  1.  VISA / MASTERCARD / CUP(Visa and Mastercard are SUPPORTED / CUP Not implemented Yet)
  2.  AMEX / DINERS CLUB / DISCOVER
  3.  SAMPATH VISHWA (Not implemented yet)
- `paymentScheme` - Payment scheme selected by the customer (VISA / MASTERCARD)

If the customer made the payment by VISA or MASTER credit/debit card, following cardHolderName and cardNumber parameters will also be available.

- `cardHolderName` - Name on the Card
- `cardNumber` - Masked card number (Ex: **\*\*\*\***0008)

**Additional fields for Tokenize Payments only:**

- `customerRefNo` - Customer reference number used for tokenization
- `paymentId` - Payment ID
- `merchantId` - Merchant ID
- `customerId` - Customer ID
- `uid` - Unique identifier
- `statusIndicator` - Status indicator
- `token` - Token object containing card tokenization details
  - `tokenId` - Unique token ID for the saved card
  - `maskedCardNo` - Masked card number for display
  - `exp` - Card expiration date (MMYY format)
  - `reference` - Reference information (if any)
  - `nickname` - Card nickname (if set)
  - `tokenStatus` - Status of tokenization (SUCCESS/FAILED)
  - `defaultCard` - Whether this is the default card (0/1)

##### Send response to callback

```json
{
  "Status": 200
}
```

---

## React.js Integration Example

```jsx
import React from "react";
import { payablePayment } from "payable-ipg-js";

const TokenizePayment = () => {
  const handlePayment = () => {
    const payment = {
      checkValue: "Your Check Value",
      orderDescription: "Payment for furni",
      invoiceId: "INVGl2lrQHEEm",
      logoUrl: "https://ipgv2-comm.payable.lk/images/chocolate2.png",
      notifyUrl: "https://yoursite.com/v1/webhook/payment",
      returnUrl: "https://yoursite.com/receipt",
      merchantKey: "F4D847FB8BF0EF52",
      customerFirstName: "John",
      customerLastName: "Doe",
      customerMobilePhone: "0715117264",
      customerPhone: "1232131233",
      customerEmail: "john@example.com",
      billingCompanyName: "Example Company",
      billingAddressStreet: "123 Main Street",
      billingAddressStreet2: "Apt 1",
      billingAddressCity: "Colombo",
      billingAddressStateProvince: "Western",
      billingAddressCountry: "LK",
      billingAddressPostcodeZip: "10000",
      amount: "100.00",
      currencyCode: "LKR",
      paymentType: "1", // One-time payment
      isSaveCard: "1", // Save card for future use
      customerRefNo: "CUST123456789",
      doFirstPayment: "1", // Charge immediately
    };

    // Process payment with tokenization
    payablePayment(payment, true); // true = test mode (sandbox)
  };

  return (
    <div>
      <h2>Payment with Card Tokenization</h2>
      <button onClick={handlePayment}>Pay & Save Card</button>
    </div>
  );
};

export default TokenizePayment;
```

### Core Functions

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

**Formula for Tokenize Payment Webhook Validation (paymentType = 3):**

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

## Support

- **Documentation**: [PAYable Developer Docs](https://github.com/payable/ipg-sdk)
- **Support**: +94 11 777 6 777
- **Website**: https://www.payable.lk

## License

This tokenize payment React.js SDK is provided by PAYable (Pvt) Ltd.
