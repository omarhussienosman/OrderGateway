# WSO2 Micro Integrator 4.6 — Order Gateway Project

## 1. Overview
The **Order Gateway** is an end-to-end integration solution developed using **WSO2 Micro Integrator (MI) 4.6**. It exposes a single, unified REST contract to clients while seamlessly orchestrating interactions between three distinct backend systems:
1. **REST Backend (DummyJSON)**: Fetches product information (`title`, `price`).
2. **SOAP Backend (Calculator Web Service)**: Calculates total order price using the `Multiply` operation.
3. **Data Service (OrderDataService - MySQL TrainingDB)**: Persists and retrieves orders in a relational database.

---

## 2. Architecture & Data Flow

```
Client (POST /order-gateway/orders)
   │
   ▼
WSO2 MI REST API (Orchestrator)
   │
   ├── [Step 1] Validate Request & Extract X-Request-ID
   │
   ├── [Step 2] GET https://dummyjson.com/products/{productId} (REST)
   │            Extract: productName, unitPrice
   │
   ├── [Step 3] POST http://www.dneonline.com/calculator.asmx (SOAP 1.1)
   │            Operation: Multiply(intA: unitPrice, intB: quantity)
   │            Extract: totalAmount
   │
   ├── [Step 4] POST http://localhost:8290/services/OrderDataService/orders (DSS)
   │            Inserts order into TrainingDB.orders table
   │
   └── [Step 5] Return Unified JSON Response (HTTP 200 OK)
```

---

## 3. Systems & Endpoints Used

| System | Protocol | Endpoint URL / Action | Description |
|---|---|---|---|
| **Client Gateway** | REST / JSON | `POST http://localhost:8290/order-gateway/orders` | Main Order Creation API |
| **Product Backend** | REST / JSON | `GET https://dummyjson.com/products/{productId}` | Product details catalog |
| **Calculator Backend** | SOAP 1.1 | `POST http://www.dneonline.com/calculator.asmx` (`SOAPAction: http://tempuri.org/Multiply`) | Arithmetic total price calculation |
| **Order Data Service** | DSS / REST | `http://localhost:8290/services/OrderDataService/orders` | Persistence in MySQL |
| **Database** | MySQL | `jdbc:mysql://localhost:3309/TrainingDB` | Docker MySQL container |

---

## 4. API Contracts

### 4.1 Client Request Contract
- **Method**: `POST`
- **Path**: `/order-gateway/orders`
- **Headers**:
  - `Content-Type: application/json`
  - `X-Request-ID: req-5001` (Optional, auto-generated if omitted)
- **Request Body**:
```json
{
  "orderId": 5001,
  "customerId": 1001,
  "productId": 10,
  "quantity": 2
}
```

### 4.2 Final Success Response Contract (200 OK)
```json
{
  "orderId": 5001,
  "customerId": 1001,
  "product": {
    "id": 10,
    "name": "Gucci Bloom Eau de",
    "unitPrice": 79.99
  },
  "quantity": 2,
  "totalAmount": 160,
  "status": "NEW"
}
```

### 4.3 Additional Challenge Endpoints
- `GET /order-gateway/products/{productId}`: Direct lookup returning simplified product details.
- `GET /order-gateway/orders/{orderId}`: Retrieves a single order by ID from `TrainingDB`.
- `GET /order-gateway/orders`: Retrieves all orders stored in `TrainingDB`.

---

## 5. Error Handling & HTTP Status Codes

The project utilizes a reusable sequence (`ErrorHandlerSequence.xml`) to format standardized error responses matching enterprise standards:

```json
{
  "status": "ERROR",
  "errorCode": 400,
  "errorMessage": "Bad Request: Missing or invalid required fields (orderId, customerId, productId, quantity > 0)",
  "errorDetail": "Validation failed",
  "requestId": "req-tc03-missing-qty"
}
```

| Scenario | HTTP Status | Fault Sequence |
|---|---|---|
| Missing input field or `quantity <= 0` | `400 Bad Request` | `BadRequestFaultSequence.xml` |
| Product ID does not exist in DummyJSON | `404 Not Found` | `ProductNotFoundFaultSequence.xml` |
| SOAP Calculator service unavailable | `502 Bad Gateway` | `SoapCalculatorFaultSequence.xml` |
| Database / DSS connection or query failure | `500 Internal Server Error` | `DatabaseFaultSequence.xml` |

---

## 6. Database Setup (`TrainingDB`)

The database runs in Docker on port `3309`.

### SQL Schema (`database/schema.sql`):
```sql
CREATE DATABASE IF NOT EXISTS TrainingDB;
USE TrainingDB;

CREATE TABLE IF NOT EXISTS orders (
    id INT PRIMARY KEY,
    customer_id INT NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    quantity INT NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'NEW',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 7. Project Structure

```
OrderGateway/
├── .vscode/
│   └── settings.json                  # VS Code WSO2 MI configuration
├── database/
│   └── schema.sql                     # MySQL database & table creation script
├── deployment/
│   ├── deployment.toml                # WSO2 MI server configuration
│   └── libs/
│       └── mysql-connector-j-8.0.31.jar # MySQL JDBC Driver
├── postman/
│   └── OrderGateway.postman_collection.json # Complete test suite
├── src/
│   └── main/
│       └── wso2mi/
│           └── artifacts/
│               ├── apis/
│               │   └── OrderGatewayAPI.xml
│               ├── data-services/
│               │   └── OrderDataService.dbs
│               ├── endpoints/
│               │   ├── CalculatorSoapEndpoint.xml
│               │   ├── OrderDataServiceEndpoint.xml
│               │   └── ProductHttpEndpoint.xml
│               └── sequences/
│                   ├── BadRequestFaultSequence.xml
│                   ├── DatabaseFaultSequence.xml
│                   ├── ErrorHandlerSequence.xml
│                   ├── ProductNotFoundFaultSequence.xml
│                   └── SoapCalculatorFaultSequence.xml
├── pom.xml                            # Maven CApp build configuration
└── README.md
```

---

## 8. Testing with Postman

Import `postman/OrderGateway.postman_collection.json` into Postman to test:
1. **TC01**: Create Order Success (200 OK)
2. **TC02**: Dynamic Values (200 OK)
3. **TC03**: Missing Field (400 Bad Request)
4. **TC04**: Invalid Quantity Zero (400 Bad Request)
5. **TC05**: Product Not Found (404 Not Found)
6. **TC06**: Standalone Product Lookup (200 OK)
7. **TC07**: Standalone Product Lookup Not Found (404 Not Found)
8. **TC08**: Get Order by ID (200 OK)
9. **TC09**: Get All Orders (200 OK)

---

## 9. 3-Minute Presentation Summary (Oral Defense Notes)
- **Problem**: In microservice architectures, clients shouldn't deal with protocol mismatches (REST vs SOAP vs DB).
- **Solution**: We built an Order Gateway in WSO2 MI that acts as an abstraction layer.
- **Key Highlights**:
  - Full protocol transformation (JSON to SOAP XML, SOAP to JSON, REST to DSS).
  - Production-aware observability with correlated `X-Request-ID` and custom logging.
  - Fail-safe error handling mapping each backend failure to correct HTTP status codes.
  - Data persisted and verified in MySQL `TrainingDB.orders`.
