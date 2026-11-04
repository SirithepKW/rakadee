# API Spec (เจ้าของ: SA)

SA วางไฟล์ในโฟลเดอร์นี้:

- `openapi.yaml` รายการ API ทั้งระบบในรูปแบบ OpenAPI 3 **ไฟล์หลักที่ AI และทุกคนอ้างอิง**
- (ถ้าใช้) Postman collection export เป็น `rakadee.postman_collection.json`

กติกา:

- ทุก path ขึ้นต้นด้วย `/api` ชื่อ resource เป็นพหูพจน์ `kebab-case` เช่น `/api/listings/{id}/bids`
- body และ response ใช้ `camelCase`
- response สำเร็จ: `{ "success": true, "data": ... }`
- response ผิดพลาด: `{ "success": false, "error": { "code": "BID_TOO_LOW", "message": "..." } }`
- error code ใช้ `UPPER_SNAKE_CASE` และต้องระบุไว้ใน spec ของแต่ละ endpoint
- ระบุ role ที่เรียกได้ทุก endpoint: guest, bidder, seller, admin, super_admin
- หัวข้อโมดูล (tag): auth, users, wallet, payments, listings, bids, orders, shipments, disputes, notifications, admin


# API Specification 

## 1. ภาพรวมAPI 
- **Base URL:** `https://api.auction-system.local/api/v1`
- **Content-Type:** `application/json` (ยกเว้นอัปโหลดรูปภาพ ใช้ `multipart/form-data`)
- **Authentication:** ส่งผ่าน HTTP Header: `Authorization: Bearer <JWT_TOKEN>`
- **Role Definitions:** `guest`, `bidder`, `seller`, `admin`, `super_admin`
- **Response Format:**
  - สำเร็จ: `{ "success": true, "statusCode": 200, "message": "...", "data": ... }`
  - ผิดพลาด: `{ "success": false, "statusCode": 400, "error": { "code": "ERROR_CODE", "message": "..." } }`

---

## 2. โมดูลระบบสมาชิกและการยืนยันตัวตน (Auth Module)

### 2.1 สมัครสมาชิก (Register)
- **Method:** `POST`
- **Path:** `/auth/register`
- **Role:** `guest`
- **Request Body:**
```json
{
  "email": "*******@*******.***",
  "username": "somchai123",
  "password": "StrongPassword123",
  "fullName": "สมชาย ใจดี",
  "phone": "**********",
  "idCardNumber": "*************",
  "acceptTos": true
}
```
- **Success Response (201 Created):**
```json
{
  "success": true,
  "statusCode": 201,
  "message": "สมัครสมาชิกสำเร็จ กรุณายืนยันรหัส OTP",
  "data": {
    "userId": "usr_998877",
    "email": "*******@*******.***",
    "username": "somchai123",
    "idCardLast4": "4005"
  }
}
```
- **Error Codes:** `ERR_PHONE_ALREADY_EXISTS`, `ERR_EMAIL_ALREADY_EXISTS`, `ERR_USERNAME_ALREADY_EXISTS`, `ERR_INVALID_ID_CARD`, `ERR_TOS_REQUIRED`, `ERR_BLACKLISTED_USER`

### 2.2 ยืนยัน OTP (Verify OTP)
- **Method:** `POST`
- **Path:** `/auth/verify-otp`
- **Role:** `guest`
- **Request Body:**
```json
{
  "email": "*******@*******.***",
  "otpCode": "482910"
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "ยืนยัน OTP สำเร็จ บัญชีเปิดใช้งานและได้รับเครดิตกระเป๋าเงินเริ่มต้นเรียบร้อย (10,000 บาท ตาม D-05)",
  "data": {
    "verified": true,
    "initialWalletBalance": 10000.00
  }
}
```
- **Error Codes:** `ERR_INVALID_OTP`, `ERR_OTP_EXPIRED`

### 2.3 เข้าสู่ระบบ (Login)
- **Method:** `POST`
- **Path:** `/auth/login`
- **Role:** `guest`
- **Request Body:**
```json
{
  "identifier": "somchai123",
  "password": "StrongPassword123"
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "เข้าสู่ระบบสำเร็จ",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1...",
    "refreshToken": "rt_abc123xyz",
    "role": "bidder",
    "accountStatus": "active"
  }
}
```
- **Error Codes:** `ERR_INVALID_CREDENTIALS`, `ERR_ACCOUNT_SUSPENDED`, `ERR_ACCOUNT_RESTRICTED`, `ERR_ACCOUNT_INACTIVE`

### 2.4 ขอรีเซ็ตรหัสผ่าน (Forgot Password)
- **Method:** `POST`
- **Path:** `/auth/forgot-password`
- **Role:** `guest`
- **Request Body:**
```json
{
  "email": "*******@*******.***"
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "ส่งลิงก์รีเซ็ตรหัสผ่านไปยังอีเมลแล้ว (อายุการใช้งาน 30 นาที)"
}
```
- **Error Codes:** `ERR_EMAIL_NOT_FOUND`

### 2.5 เปลี่ยนรหัสผ่าน (Change Password)
- **Method:** `POST`
- **Path:** `/auth/change-password`
- **Role:** `bidder`, `seller`, `admin`, `super_admin`
- **Request Body:**
```json
{
  "oldPassword": "StrongPassword123",
  "newPassword": "NewStrongPassword456"
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "เปลี่ยนรหัสผ่านสำเร็จ กรุณาเข้าสู่ระบบใหม่อีกครั้ง"
}
```
- **Error Codes:** `ERR_INVALID_OLD_PASSWORD`, `ERR_WEAK_PASSWORD`

---

## 3. โมดูลจัดการสินค้าและรอบประมูล (Auctions Module)

### 3.1 ดึงรายการสินค้าประมูลทั้งหมด
- **Method:** `GET`
- **Path:** `/auctions`
- **Role:** `guest`, `bidder`, `seller`, `admin`
- **Query Parameters:** `page=1`, `limit=10`, `status=active`, `categoryId=2`
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "data": {
    "items": [
      {
        "auctionId": 1024,
        "title": "Tianchen Dreamout Card",
        "currentPrice": 2500.00,
        "bidIncrement": 100.00,
        "status": "active",
        "endTime": "2026-11-05T18:00:00Z"
      }
    ],
    "total": 45,
    "page": 1,
    "limit": 10
  }
}
```
- **Error Codes:** `ERR_INVALID_QUERY_PARAMS`

### 3.2 สร้างรายการประมูลใหม่
- **Method:** `POST`
- **Path:** `/auctions`
- **Role:** `seller`
- **Request Body:**
```json
{
  "title": "Tianchen Dreamout Card",
  "categoryId": 2,
  "startPrice": 1000.00,
  "bidIncrement": 100.00,
  "reservePrice": 3000.00,
  "buyNowPrice": 5000.00,
  "durationDays": 3,
  "description": "สภาพ 95%"
}
```
- **Success Response (201 Created):**
```json
{
  "success": true,
  "statusCode": 201,
  "message": "สร้างรายการประมูลสำเร็จ รอแอดมินตรวจสอบ",
  "data": {
    "auctionId": 1024,
    "status": "pending_approval"
  }
}
```
- **Error Codes:** `ERR_INVALID_PRICE_RULE`, `ERR_SELLER_NOT_VERIFIED`

---

## 4. โมดูลการประมูลและราคา (Bidding Module)

### 4.1 ยื่นเสนอราคา (Manual Bid)
- **Method:** `POST`
- **Path:** `/auctions/:id/bids`
- **Role:** `bidder`
- **URL Parameters:** `id=1024`
- **Request Body:**
```json
{
  "bidAmount": 2600.00
}
```
- **Success Response (201 Created):**
```json
{
  "success": true,
  "statusCode": 201,
  "message": "เสนอราคาสำเร็จ",
  "data": {
    "auctionId": 1024,
    "currentPrice": 2600.00,
    "totalBids": 12,
    "autoExtended": false,
    "newEndTime": "2026-11-05T18:00:00Z"
  }
}
```
- **Error Codes:** `ERR_BID_TOO_LOW`, `ERR_INSUFFICIENT_WALLET`, `ERR_AUCTION_NOT_ACTIVE`, `ERR_SELF_BIDDING`

### 4.2 ตั้งค่าประมูลอัตโนมัติ (Proxy Bid)
- **Method:** `POST`
- **Path:** `/auctions/:id/proxy-bid`
- **Role:** `bidder`
- **Request Body:**
```json
{
  "maxProxyAmount": 4500.00
}
```
- **Success Response (201 Created):**
```json
{
  "success": true,
  "statusCode": 201,
  "message": "ตั้งค่า Proxy Bid สำเร็จ",
  "data": {
    "proxyId": 889,
    "maxProxyAmount": 4500.00
  }
}
```
- **Error Codes:** `ERR_PROXY_BELOW_CURRENT_PRICE`, `ERR_INSUFFICIENT_WALLET`

---

## 5. โมดูลกระเป๋าเงินและการชำระเงิน (Wallet & Payment Module)

### 5.1 ตรวจสอบยอดเงินในกระเป๋า
- **Method:** `GET`
- **Path:** `/wallet/balance`
- **Role:** `bidder`, `seller`
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "data": {
    "availableBalance": 15000.00,
    "currency": "THB"
  }
}
```
- **Error Codes:** `ERR_UNAUTHORIZED`

### 5.2 จำลองการเติมเงิน (Mock Deposit)
- **Method:** `POST`
- **Path:** `/wallet/deposit`
- **Role:** `bidder`, `seller`
- **Request Body:**
```json
{
  "amount": 5000.00,
  "paymentMethod": "mock_promptpay"
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "เติมเงินสำเร็จ",
  "data": {
    "transactionId": "tx_mock_001",
    "newBalance": 20000.00
  }
}
```
- **Error Codes:** `ERR_INVALID_AMOUNT`

### 5.3 แนบหลักฐานสลิปโอนเงิน
- **Method:** `POST`
- **Path:** `/orders/:orderId/slip`
- **Role:** `bidder` (ผู้ชนะประมูล)
- **Request Body (Multipart Form-Data):**
  - `slipImage`: `[File Binary]`
  - `amount`: `2600.00`
- **Success Response (201 Created):**
```json
{
  "success": true,
  "statusCode": 201,
  "message": "อัปโหลดสลิปสำเร็จ เวลา 48 ชม. หยุดนับ รอแอดมินตรวจสอบภายใน 24 ชม.",
  "data": {
    "paymentId": 302,
    "status": "pending_verification"
  }
}
```
- **Error Codes:** `ERR_INVALID_FILE_FORMAT`, `ERR_ORDER_EXPIRED`

---

## 6. โมดูลการจัดส่งและคำสั่งซื้อ (Order & Shipment Module)

### 6.1 บันทึกเลขพัสดุ (Shipment Tracking)
- **Method:** `POST`
- **Path:** `/orders/:orderId/shipment`
- **Role:** `seller`
- **Request Body:**
```json
{
  "courierName": "Flash Express",
  "trackingNumber": "TH1234567890"
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "บันทึกข้อมูลจัดส่งสำเร็จ",
  "data": {
    "shipmentStatus": "shipped",
    "orderStatus": "shipped"
  }
}
```
- **Error Codes:** `ERR_INVALID_TRACKING`, `ERR_NOT_ORDER_SELLER`

### 6.2 ยืนยันรับสินค้าและปล่อย Escrow
- **Method:** `POST`
- **Path:** `/orders/:orderId/confirm-delivery`
- **Role:** `bidder`
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "ยืนยันรับสินค้าสำเร็จ เงินถูกโอนให้ผู้ขายแล้ว",
  "data": {
    "orderStatus": "completed",
    "escrowStatus": "released"
  }
}
```
- **Error Codes:** `ERR_ORDER_NOT_DELIVERED`, `ERR_NOT_ORDER_BUYER`

---

## 7. โมดูลข้อพิพาทและการจัดการ (Dispute & Admin Module)

### 7.1 เปิดข้อพิพาท (Open Dispute)
- **Method:** `POST`
- **Path:** `/orders/:orderId/disputes`
- **Role:** `bidder`, `seller`
- **Request Body:**
```json
{
  "reason": "item_not_as_described",
  "description": "สินค้าแตกหักไม่ตรงตามสภาพที่ระบุ",
  "evidenceImageUrls": ["https://storage.local/img1.jpg"]
}
```
- **Success Response (201 Created):**
```json
{
  "success": true,
  "statusCode": 201,
  "message": "เปิดข้อพิพาทสำเร็จ สถานะคำสั่งซื้อถูกระงับ",
  "data": {
    "disputeId": 55,
    "status": "opened"
  }
}
```
- **Error Codes:** `ERR_DISPUTE_PERIOD_EXPIRED`, `ERR_DISPUTE_ALREADY_EXISTS`

### 7.2 แอดมินตัดสินข้อพิพาท
- **Method:** `POST`
- **Path:** `/admin/disputes/:id/resolve`
- **Role:** `admin`, `super_admin`
- **Request Body:**
```json
{
  "resolution": "refund_buyer",
  "adminNote": "ตรวจสอบหลักฐานแล้วผู้ขายส่งสินค้าชำรุดจริง คืนเงินเต็มจำนวน"
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "ตัดสินข้อพิพาทและดำเนินการคืนเงินเรียบร้อย",
  "data": {
    "disputeStatus": "closed",
    "escrowStatus": "refunded"
  }
}
```
- **Error Codes:** `ERR_DISPUTE_ALREADY_CLOSED`, `ERR_ADMIN_ONLY`

---

## 8. โมดูลยืนยันตัวตน (KYC & Verification Module)

### 8.1 ส่งเอกสารยืนยันตัวตน (KYC Submission)
- **Method:** `POST`
- **Path:** `/users/kyc/submit`
- **Role:** `bidder`, `seller`
- **Request Body (Multipart Form-Data):**
  - `idCardNumber`: `*************`
  - `idCardImage`: `[File Binary]`
- **Success Response (201 Created):**
```json
{
  "success": true,
  "statusCode": 201,
  "message": "ส่งข้อมูลยืนยันตัวตนสำเร็จ รอแอดมินตรวจสอบ",
  "data": {
    "kycId": 401,
    "status": "pending"
  }
}
```
- **Error Codes:** `ERR_ID_CARD_ALREADY_EXISTS`, `ERR_INVALID_FILE_FORMAT`

### 8.2 แอดมินตรวจสอบและอนุมัติ/ปฏิเสธ
- **Method:** `PATCH`
- **Path:** `/admin/kyc/:id/review`
- **Role:** `admin`, `super_admin`
- **Request Body:**
```json
{
  "status": "approved",
  "rejectReason": null
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "อัปเดตสถานะสำเร็จ",
  "data": {
    "userId": "usr_998877",
    "kycStatus": "approved"
  }
}
```
- **Error Codes:** `ERR_KYC_ALREADY_REVIEWED`

---

## 9. โมดูลรีวิวและให้คะแนน (Reviews & Ratings Module)

### 9.1 เขียนรีวิวหลังคำสั่งซื้อเสร็จสมบูรณ์
- **Method:** `POST`
- **Path:** `/orders/:orderId/reviews`
- **Role:** `bidder`, `seller`
- **Request Body:**
```json
{
  "rating": 5,
  "comment": "สินค้าตรงปก ส่งไว ยอดเยี่ยมมากครับ"
}
```
- **Success Response (201 Created):**
```json
{
  "success": true,
  "statusCode": 201,
  "message": "บันทึกรีวิวสำเร็จ",
  "data": {
    "reviewId": 12,
    "rating": 5
  }
}
```
- **Error Codes:** `ERR_ORDER_NOT_COMPLETED`, `ERR_ALREADY_REVIEWED`

### 9.2 ดูรายการรีวิวของผู้ใช้หรือร้านค้า
- **Method:** `GET`
- **Path:** `/users/:userId/reviews`
- **Role:** `guest`, `bidder`, `seller`, `admin`
- **Query Parameters:** `page=1`, `limit=10`
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "data": {
    "averageRating": 4.85,
    "totalReviews": 34,
    "items": [
      {
        "reviewId": 12,
        "rating": 5,
        "comment": "สินค้าตรงปก ส่งไว",
        "reviewerName": "สวัสดี วันพุธ",
        "createdAt": "2026-11-02T10:00:00Z"
      }
    ]
  }
}
```

---

## 10. โมดูลการแจ้งเตือน (Notifications Module)

### 10.1 ดึงรายการแจ้งเตือนของผู้ใช้
- **Method:** `GET`
- **Path:** `/notifications`
- **Role:** `bidder`, `seller`
- **Query Parameters:** `unreadOnly=true`, `page=1`
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "data": {
    "unreadCount": 2,
    "items": [
      {
        "notificationId": 881,
        "title": "คุณชนะการประมูล!",
        "message": "ยินดีด้วยคุณชนะประมูล กรุณาชำระเงินภายใน 48 ชม.",
        "isRead": false,
        "createdAt": "2026-11-02T11:30:00Z"
      }
    ]
  }
}
```

### 10.2 ทำเครื่องหมายว่าอ่านแจ้งเตือนแล้ว
- **Method:** `PATCH`
- **Path:** `/notifications/:id/read`
- **Role:** `bidder`, `seller`
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "อัปเดตสถานะการอ่านสำเร็จ"
}
```

---

## 11. โมดูลการจัดการระบบและแอดมินขั้นสูง (Admin & System Config Module)

### 11.1 ตั้งค่าระบบ (System Configuration)
- **Method:** `PUT`
- **Path:** `/admin/settings`
- **Role:** `super_admin`
- **Request Body:**
```json
{
  "settingKey": "PLATFORM_FEE_PERCENTAGE",
  "settingValue": "5.00",
  "description": "ค่าธรรมเนียมแพลตฟอร์ม 5 เปอร์เซ็นต์"
}
```
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "message": "อัปเดตการตั้งค่าระบบสำเร็จ",
  "data": {
    "settingKey": "PLATFORM_FEE_PERCENTAGE",
    "settingValue": "5.00"
  }
}
```
- **Error Codes:** `ERR_INVALID_SETTING_VALUE`, `ERR_SUPER_ADMIN_ONLY`

### 11.2 ตรวจสอบประวัติการทำงานของแอดมิน (Admin Audit Logs)
- **Method:** `GET`
- **Path:** `/admin/audit-logs`
- **Role:** `super_admin`
- **Query Parameters:** `page=1`, `limit=20`, `adminId=usr_12345`
- **Success Response (200 OK):**
```json
{
  "success": true,
  "statusCode": 200,
  "data": {
    "items": [
      {
        "logId": 901,
        "adminId": "usr_12345",
        "action": "UPDATE_SYSTEM_SETTING",
        "target": "PLATFORM_FEE_PERCENTAGE",
        "ipAddress": "192.168.1.50",
        "createdAt": "2026-11-04T10:30:00Z"
      }
    ],
    "total": 120,
    "page": 1,
    "limit": 20
  }
}
```
- **Error Codes:** `ERR_SUPER_ADMIN_ONLY`