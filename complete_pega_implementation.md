# Complete Pega Implementation - All 4 Components

## Component 1: REST Connector for Quote Request API (Async)

### Connector Configuration

**Navigation**: App Studio → Integration → Connectors → REST

```Plaintext
Connector Name: QuoteRequestAsyncConnector
Description: Async quote request to insurance providers
Service Package: Integration.Insurance.External
Context: Work
```

### Service Configuration

```
Service Name: QuoteRequestAsyncService
Base URL: https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api
Authentication: None
Request Timeout: 30 seconds
Response Timeout: 30 seconds
```

### Method Configuration

```
Method Name: RequestQuoteAsync
HTTP Method: POST
Resource Path: /quotes
Content-Type: application/json
Accept: application/json
```

### Request Data Transform (D_QuoteRequestAsyncMapping)

```javascript
// Transform case data to exact API format from your specification

// Basic request information
.requestId = .QuoteRequestID
.eligibilityResponseId = .EligibilityResponseID

// Customer details mapping (exact format from spec)
.customerDetails.name = .CustomerDetails.Name
.customerDetails.businessName = .CustomerDetails.BusinessName

// Address mapping
.customerDetails.address.street = .CustomerDetails.Address.Street
.customerDetails.address.city = .CustomerDetails.Address.City
.customerDetails.address.state = .CustomerDetails.Address.State
.customerDetails.address.zip = .CustomerDetails.Address.ZIP

// Contact information
.customerDetails.contact.email = .CustomerDetails.Contact.Email
.customerDetails.contact.phone = .CustomerDetails.Contact.Phone

// Prior claims mapping (array format)
For Each .CustomerDetails.PriorClaims {
    .customerDetails.priorClaims(+).date = .Date
    .customerDetails.priorClaims().type = .Type
    .customerDetails.priorClaims().amount = .Amount
}

// Asset details mapping (exact format from spec)
.assetDetails.propertyType = .AssetDetails.PropertyType
.assetDetails.squareFootage = .AssetDetails.SquareFootage
.assetDetails.constructionType = .AssetDetails.ConstructionType
.assetDetails.yearBuilt = .AssetDetails.YearBuilt

// Coverage details mapping
.coverageDetails.coverageType = .CoverageDetails.CoverageType
.coverageDetails.coverageLimit = .CoverageDetails.CoverageLimit
.coverageDetails.deductible = .CoverageDetails.Deductible

// Add-ons mapping (array format)
For Each .CoverageDetails.AddOns {
    .coverageDetails.addOns(+) = .AddOnType
}

// Payment preferences
.paymentPreferences.paymentMode = .PaymentPreferences.PaymentMode
.paymentPreferences.autoRenew = .PaymentPreferences.AutoRenew

// Documents mapping (array format)
For Each .Documents {
    .documents(+).documentType = .DocumentType
    .documents().documentUrl = .DocumentURL
}
```

### Response Data Transform (D_QuoteRequestAsyncResponse)

```javascript
// Map API response back to case properties (exact format from spec)

.QueueID = .queueId
.InsuranceCompanyID = .insuranceCompanyId
.QuoteRequestStatus = .status
.EstimatedCompletionTime = .estimatedCompletionTime
.ResponseTimestamp = .timestamp
.APIRequestID = .requestId

// Set processing flags
.IsQuoteProcessing = "true"
.QuoteRequestTimestamp = @Current DateTime
.NextRetrievalTime = @ParseDateTime(.estimatedCompletionTime)
```

### Error Data Transform (D_QuoteRequestAsyncError)

```javascript
// Handle async quote request errors

If .pxHTTPStatus == "400" Then
    .ErrorType = "ValidationError"
    .ErrorMessage = "Invalid quote request parameters"
    .RetryAllowed = "false"
Else If .pxHTTPStatus == "401" Then
    .ErrorType = "AuthenticationError"
    .ErrorMessage = "Authentication failed with insurance provider"
    .RetryAllowed = "true"
Else If .pxHTTPStatus == "429" Then
    .ErrorType = "RateLimitError"
    .ErrorMessage = "Rate limit exceeded"
    .RetryAllowed = "true"
Else If .pxHTTPStatus == "500" Then
    .ErrorType = "ServerError"
    .ErrorMessage = "Insurance provider server error"
    .RetryAllowed = "true"
Else
    .ErrorType = "UnknownError"
    .ErrorMessage = "Unexpected error during quote request"
    .RetryAllowed = "true"
End-If

// Set retry timing
If .RetryAllowed == "true" Then
    .NextRetryTime = @AddMinutes(@Current DateTime, 15)
End-If
```

---

## Component 2: REST Connector for Quote Retrieval API (Queue)

### Connector Configuration

```
Connector Name: QuoteRetrievalQueueConnector
Description: Retrieve processed quotes using queue ID
Service Package: Integration.Insurance.External
Context: Work
```

### Service Configuration

```
Service Name: QuoteRetrievalQueueService
Base URL: https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api
Authentication: None
Request Timeout: 30 seconds
Response Timeout: 30 seconds
```

### Method Configuration

```
Method Name: RetrieveQuoteByQueue
HTTP Method: GET
Resource Path: /quotes?queueId={QueueId}
Content-Type: application/json
Accept: application/json
```

### Request Data Transform (D_QuoteRetrievalQueueMapping)

```javascript
// Set query parameter for queue-based retrieval
@SetParam("QueueId", .QueueID)
```

### Response Data Transform (D_QuoteRetrievalQueueResponse)

```javascript
// Handle array response from filter query
If @PropertyExists(.0) Then
    .QuoteData = .0  // Get first item from filtered array
Else
    .QuoteData = .   // Direct object response
End-If

// Map response fields (exact format from spec)
.QueueID = .QuoteData.queueId
.InsuranceCompanyID = .QuoteData.insuranceCompanyId
.QuoteStatus = .QuoteData.status
.ResponseTimestamp = .QuoteData.timestamp

// Process based on quote status
If .QuoteData.status == "Quote Ready" Then
    // Map complete quote details (exact format from spec)
    .QuoteDetails.QuoteID = .QuoteData.quoteDetails.quoteId
    .QuoteDetails.Premium = .QuoteData.quoteDetails.premium
    .QuoteDetails.CoverageLimit = .QuoteData.quoteDetails.coverageLimit
    .QuoteDetails.Deductible = .QuoteData.quoteDetails.deductible
    .QuoteDetails.ValidUntil = .QuoteData.quoteDetails.validUntil
    .QuoteDetails.QuotePDFUrl = .QuoteData.quoteDetails.quoteSummaryPDFUrl

    // Map add-ons array
    For Each .QuoteData.quoteDetails.addOns {
        .QuoteDetails.AddOns(+).AddOnType = .
    }

    .IsQuoteReady = "true"
    .QuoteCompletedTimestamp = .QuoteData.timestamp

Else If .QuoteData.status == "Quote Processing" Then
    .IsQuoteReady = "false"
    .QuoteStillProcessing = "true"
    .EstimatedCompletionTime = .QuoteData.estimatedCompletionTime

Else If .QuoteData.status == "Quote Failed" Then
    .IsQuoteReady = "false"
    .QuoteFailed = "true"
    .QuoteFailureReason = .QuoteData.reason
End-If

.LastRetrievalTimestamp = @Current DateTime
```

### Error Data Transform (D_QuoteRetrievalQueueError)

```javascript
// Handle quote retrieval errors

If .pxHTTPStatus == "404" Then
    .ErrorType = "QueueNotFound"
    .ErrorMessage = "Queue ID not found or expired"
    .RetryAllowed = "false"
Else If .pxHTTPStatus == "408" Then
    .ErrorType = "RequestTimeout"
    .ErrorMessage = "Request timeout - quote may still be processing"
    .RetryAllowed = "true"
Else If .pxHTTPStatus == "500" Then
    .ErrorType = "ServerError"
    .ErrorMessage = "Insurance provider server error"
    .RetryAllowed = "true"
Else
    .ErrorType = "UnknownError"
    .ErrorMessage = "Unexpected error during quote retrieval"
    .RetryAllowed = "true"
End-If

// Set retry timing for retrieval
If .RetryAllowed == "true" Then
    .NextRetrievalRetryTime = @AddMinutes(@Current DateTime, 30)
End-If
```

---

## Component 3: REST Service for Quote Request API (Async)

### Service Configuration

**Navigation**: App Studio → Integration → Services → REST Service

```
Service Name: QuoteRequestAsyncRESTService
Service Package: Integration.Insurance.Internal
Service Class: Rule-Service-REST
Version: 01-01-01
Description: Internal REST service for async quote requests
```

### Resource Configuration

```
Resource Name: quoterequest
Resource Path: /insurance/quotes/request
HTTP Methods: POST
Description: Accept and process async quote requests
```

### POST Method Configuration

#### Request JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "requestId": { "type": "string", "pattern": "^QUOTE-\\d{8}-\\d{3}$" },
    "eligibilityResponseId": {
      "type": "string",
      "pattern": "^ELIG-\\d{8}-\\d{3}$"
    },
    "customerDetails": {
      "type": "object",
      "properties": {
        "name": { "type": "string", "minLength": 1 },
        "businessName": { "type": "string" },
        "address": {
          "type": "object",
          "properties": {
            "street": { "type": "string", "minLength": 1 },
            "city": { "type": "string", "minLength": 1 },
            "state": { "type": "string", "pattern": "^[A-Z]{2}$" },
            "zip": { "type": "string", "pattern": "^\\d{5}(-\\d{4})?$" }
          },
          "required": ["street", "city", "state", "zip"]
        },
        "contact": {
          "type": "object",
          "properties": {
            "email": { "type": "string", "format": "email" },
            "phone": { "type": "string", "pattern": "^\\d{3}-\\d{3}-\\d{4}$" }
          },
          "required": ["email", "phone"]
        },
        "priorClaims": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "date": { "type": "string", "format": "date" },
              "type": { "type": "string" },
              "amount": { "type": "number", "minimum": 0 }
            },
            "required": ["date", "type", "amount"]
          }
        }
      },
      "required": ["name", "address", "contact"]
    },
    "assetDetails": {
      "type": "object",
      "properties": {
        "propertyType": { "type": "string", "minLength": 1 },
        "squareFootage": { "type": "number", "minimum": 0 },
        "constructionType": { "type": "string" },
        "yearBuilt": { "type": "number", "minimum": 1800, "maximum": 2030 }
      },
      "required": ["propertyType", "squareFootage"]
    },
    "coverageDetails": {
      "type": "object",
      "properties": {
        "coverageType": { "type": "string", "minLength": 1 },
        "coverageLimit": { "type": "number", "minimum": 0 },
        "deductible": { "type": "number", "minimum": 0 },
        "addOns": {
          "type": "array",
          "items": { "type": "string" }
        }
      },
      "required": ["coverageType", "coverageLimit", "deductible"]
    },
    "paymentPreferences": {
      "type": "object",
      "properties": {
        "paymentMode": {
          "type": "string",
          "enum": ["Annual", "Semi-Annual", "Quarterly", "Monthly"]
        },
        "autoRenew": { "type": "boolean" }
      }
    },
    "documents": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "documentType": { "type": "string", "minLength": 1 },
          "documentUrl": { "type": "string", "format": "uri" }
        },
        "required": ["documentType", "documentUrl"]
      }
    }
  },
  "required": [
    "requestId",
    "eligibilityResponseId",
    "customerDetails",
    "assetDetails",
    "coverageDetails"
  ]
}
```

#### Request Mapping (D_QuoteRequestServiceMapping)

```javascript
// Map incoming REST request to internal case structure (exact spec format)

// Create new quote request case
@New-Page .QuoteRequestCase of class "Work-InsuranceQuoteRequest"

// Map basic information
.QuoteRequestCase.QuoteRequestID = .requestId
.QuoteRequestCase.EligibilityResponseID = .eligibilityResponseId
.QuoteRequestCase.RequestTimestamp = @Current DateTime
.QuoteRequestCase.Status = "Quote Request Initiated"

// Map customer details (exact format from spec)
.QuoteRequestCase.CustomerDetails.Name = .customerDetails.name
.QuoteRequestCase.CustomerDetails.BusinessName = .customerDetails.businessName

// Map address
.QuoteRequestCase.CustomerDetails.Address.Street = .customerDetails.address.street
.QuoteRequestCase.CustomerDetails.Address.City = .customerDetails.address.city
.QuoteRequestCase.CustomerDetails.Address.State = .customerDetails.address.state
.QuoteRequestCase.CustomerDetails.Address.ZIP = .customerDetails.address.zip

// Map contact information
.QuoteRequestCase.CustomerDetails.Contact.Email = .customerDetails.contact.email
.QuoteRequestCase.CustomerDetails.Contact.Phone = .customerDetails.contact.phone

// Map prior claims (array from spec)
For Each .customerDetails.priorClaims {
    .QuoteRequestCase.CustomerDetails.PriorClaims(+).Date = .date
    .QuoteRequestCase.CustomerDetails.PriorClaims().Type = .type
    .QuoteRequestCase.CustomerDetails.PriorClaims().Amount = .amount
}

// Map asset details (exact format from spec)
.QuoteRequestCase.AssetDetails.PropertyType = .assetDetails.propertyType
.QuoteRequestCase.AssetDetails.SquareFootage = .assetDetails.squareFootage
.QuoteRequestCase.AssetDetails.ConstructionType = .assetDetails.constructionType
.QuoteRequestCase.AssetDetails.YearBuilt = .assetDetails.yearBuilt

// Map coverage details (exact format from spec)
.QuoteRequestCase.CoverageDetails.CoverageType = .coverageDetails.coverageType
.QuoteRequestCase.CoverageDetails.CoverageLimit = .coverageDetails.coverageLimit
.QuoteRequestCase.CoverageDetails.Deductible = .coverageDetails.deductible

// Map add-ons array
For Each .coverageDetails.addOns {
    .QuoteRequestCase.CoverageDetails.AddOns(+).AddOnType = .
}

// Map payment preferences
.QuoteRequestCase.PaymentPreferences.PaymentMode = .paymentPreferences.paymentMode
.QuoteRequestCase.PaymentPreferences.AutoRenew = .paymentPreferences.autoRenew

// Map documents array
For Each .documents {
    .QuoteRequestCase.Documents(+).DocumentType = .documentType
    .QuoteRequestCase.Documents().DocumentURL = .documentUrl
}
```

#### Service Activity (ProcessAsyncQuoteRequest)

```java
// Step 1: Validate incoming request data
Activity: ValidateQuoteRequestData
Method: Call
Parameters: Current page
Step Page: QuoteRequestCase

// Step 2: Check for duplicate request ID
Activity: CheckDuplicateQuoteRequest
Method: Call
Parameters:
  RequestID: QuoteRequestCase.QuoteRequestID

// Step 3: Create new case instance
Activity: Work-.New
Method: Call
Parameters:
  Work Type: InsuranceQuoteRequest
  Primary Page: NewQuoteCase

// Step 4: Copy validated data to case
Property-Set-Messages
Source: QuoteRequestCase
Target: NewQuoteCase

// Step 5: Generate queue ID for async processing
Property-Set
Property: .NewQuoteCase.QueueID
Value: "QUEUE-" + @FormatDateTime(@Current DateTime, "yyyyMMdd") + "-" + @Random(100,999)

// Step 6: Start async quote processing workflow
Activity: Work-.StartProcess
Method: Call
Parameters:
  Process Name: AsyncQuoteRequestProcess
  Primary Page: NewQuoteCase

// Step 7: Queue for background processing
Activity: Queue-For-Processing
Method: Call
Parameters:
  Queue Name: QuoteRequestAsyncQueue
  Queue Item: NewQuoteCase

// Step 8: Prepare response (exact format from spec)
Property-Set
Property: .response.requestId
Value: @NewQuoteCase.QuoteRequestID

Property-Set
Property: .response.insuranceCompanyId
Value: "INSCO-INTERNAL"

Property-Set
Property: .response.queueId
Value: @NewQuoteCase.QueueID

Property-Set
Property: .response.status
Value: "Quote Processing"

Property-Set
Property: .response.estimatedCompletionTime
Value: @FormatDateTime(@AddDays(@Current DateTime, 1), "yyyy-MM-ddTHH:mm:ssZ")

Property-Set
Property: .response.timestamp
Value: @FormatDateTime(@Current DateTime, "yyyy-MM-ddTHH:mm:ssZ")
```

#### Response JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "requestId": { "type": "string" },
    "insuranceCompanyId": { "type": "string" },
    "queueId": { "type": "string" },
    "status": {
      "type": "string",
      "enum": ["Quote Processing", "Quote Failed"]
    },
    "estimatedCompletionTime": { "type": "string", "format": "date-time" },
    "timestamp": { "type": "string", "format": "date-time" }
  },
  "required": [
    "requestId",
    "insuranceCompanyId",
    "queueId",
    "status",
    "timestamp"
  ]
}
```

#### Response Mapping (D_QuoteRequestServiceResponse)

```javascript
// Map internal response to REST service response (exact spec format)

.requestId = .response.requestId
.insuranceCompanyId = .response.insuranceCompanyId
.queueId = .response.queueId
.status = .response.status
.estimatedCompletionTime = .response.estimatedCompletionTime
.timestamp = .response.timestamp
```

---

## Component 4: REST Service for Quote Retrieval API (Queue)

### Service Configuration

```
Service Name: QuoteRetrievalQueueRESTService
Service Package: Integration.Insurance.Internal
Service Class: Rule-Service-REST
Version: 01-01-01
Description: Internal REST service for queue-based quote retrieval
```

### Resource Configuration

```
Resource Name: quoteretrieve
Resource Path: /insurance/quotes/retrieve
HTTP Methods: POST
Description: Retrieve quotes using queue ID
```

### POST Method Configuration

#### Request JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "queueId": { "type": "string", "pattern": "^QUEUE-\\d{8}-\\d{3}$" },
    "requestId": { "type": "string", "pattern": "^QUOTE-\\d{8}-\\d{3}$" },
    "agencyId": { "type": "string", "pattern": "^[A-Z]+-AGENCY-\\d{3}$" }
  },
  "required": ["queueId", "agencyId"]
}
```

#### Request Mapping (D_QuoteRetrievalServiceMapping)

```javascript
// Map incoming retrieval request to internal lookup (exact spec format)

.QueueID = .queueId
.RequestID = .requestId
.AgencyID = .agencyId
.RetrievalTimestamp = @Current DateTime

// Validate agency access rights
Activity: ValidateAgencyAccess
Method: Call
Parameters: Current page
```

#### Service Activity (ProcessQueueRetrievalRequest)

```java
// Step 1: Validate queue ID format and agency access
Activity: ValidateQueueRetrievalRequest
Method: Call
Parameters: Current page

// Step 2: Find quote request case by queue ID
Activity: FindQuoteByQueueID
Method: Call
Parameters:
  QueueID: QueueID
  Result Page: FoundQuoteCase

// Step 3: Check if case was found
Decision: Was quote case found?
When: FoundQuoteCase.pyID != ""
  // Continue with quote retrieval
  Property-Set
  Property: .CaseFound
  Value: "true"
Otherwise:
  // Queue ID not found
  Property-Set
  Property: .CaseFound
  Value: "false"

  Property-Set
  Property: .response.error
  Value: "Queue ID not found"

  Return

// Step 4: Check quote processing status
Decision: Quote processing status?
When: FoundQuoteCase.QuoteStatus == "Quote Ready"
  Activity: PrepareCompleteQuoteResponse
  Method: Call
  Parameters: FoundQuoteCase

When: FoundQuoteCase.QuoteStatus == "Quote Processing"
  Activity: PrepareProcessingResponse
  Method: Call
  Parameters: FoundQuoteCase

When: FoundQuoteCase.QuoteStatus == "Quote Failed"
  Activity: PrepareFailureResponse
  Method: Call
  Parameters: FoundQuoteCase

Otherwise:
  Activity: PrepareUnknownStatusResponse
  Method: Call
  Parameters: FoundQuoteCase

// Step 5: Log retrieval attempt for audit
Activity: LogQuoteRetrievalAttempt
Method: Call
Parameters:
  QueueID: QueueID
  AgencyID: AgencyID
  Status: FoundQuoteCase.QuoteStatus
```

#### PrepareCompleteQuoteResponse Activity

```java
// Step 1: Map quote details (exact format from spec)
Property-Set
Property: .response.queueId
Value: @FoundQuoteCase.QueueID

Property-Set
Property: .response.insuranceCompanyId
Value: @FoundQuoteCase.InsuranceCompanyID

Property-Set
Property: .response.status
Value: "Quote Ready"

// Step 2: Map complete quote details object
Property-Set
Property: .response.quoteDetails.quoteId
Value: @FoundQuoteCase.QuoteDetails.QuoteID

Property-Set
Property: .response.quoteDetails.premium
Value: @FoundQuoteCase.QuoteDetails.Premium

Property-Set
Property: .response.quoteDetails.coverageLimit
Value: @FoundQuoteCase.QuoteDetails.CoverageLimit

Property-Set
Property: .response.quoteDetails.deductible
Value: @FoundQuoteCase.QuoteDetails.Deductible

Property-Set
Property: .response.quoteDetails.validUntil
Value: @FoundQuoteCase.QuoteDetails.ValidUntil

Property-Set
Property: .response.quoteDetails.quoteSummaryPDFUrl
Value: @FoundQuoteCase.QuoteDetails.QuotePDFUrl

// Step 3: Map add-ons array
For Each .FoundQuoteCase.QuoteDetails.AddOns {
    .response.quoteDetails.addOns(+) = .AddOnType
}

Property-Set
Property: .response.timestamp
Value: @FormatDateTime(@Current DateTime, "yyyy-MM-ddTHH:mm:ssZ")
```

#### Response JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "queueId": { "type": "string" },
    "insuranceCompanyId": { "type": "string" },
    "status": {
      "type": "string",
      "enum": ["Quote Ready", "Quote Processing", "Quote Failed"]
    },
    "quoteDetails": {
      "type": "object",
      "properties": {
        "quoteId": { "type": "string" },
        "premium": { "type": "number", "minimum": 0 },
        "coverageLimit": { "type": "number", "minimum": 0 },
        "deductible": { "type": "number", "minimum": 0 },
        "addOns": {
          "type": "array",
          "items": { "type": "string" }
        },
        "validUntil": { "type": "string", "format": "date" },
        "quoteSummaryPDFUrl": { "type": "string", "format": "uri" }
      },
      "required": ["quoteId", "premium", "coverageLimit", "deductible"]
    },
    "reason": { "type": "string" },
    "timestamp": { "type": "string", "format": "date-time" }
  },
  "required": ["queueId", "insuranceCompanyId", "status", "timestamp"]
}
```

#### Response Mapping (D_QuoteRetrievalServiceResponse)

```javascript
// Map internal quote data to REST service response (exact spec format)

.queueId = .response.queueId
.insuranceCompanyId = .response.insuranceCompanyId
.status = .response.status
.timestamp = .response.timestamp

// Include quote details only if quote is ready
If .response.status == "Quote Ready" Then
    .quoteDetails.quoteId = .response.quoteDetails.quoteId
    .quoteDetails.premium = .response.quoteDetails.premium
    .quoteDetails.coverageLimit = .response.quoteDetails.coverageLimit
    .quoteDetails.deductible = .response.quoteDetails.deductible
    .quoteDetails.validUntil = .response.quoteDetails.validUntil
    .quoteDetails.quoteSummaryPDFUrl = .response.quoteDetails.quoteSummaryPDFUrl

    // Map add-ons array
    For Each .response.quoteDetails.addOns {
        .quoteDetails.addOns(+) = .
    }
Else If .response.status == "Quote Failed" Then
    .reason = .response.reason
End-If
```

---

## Supporting Configuration

### Case Type Properties

```javascript
// Quote Request Case Properties
Work-InsuranceQuoteRequest
├── QuoteRequestID (Text)
├── EligibilityResponseID (Text)
├── QueueID (Text)
├── Status (Text)
├── RequestTimestamp (DateTime)
├── CustomerDetails (Page)
│   ├── Name (Text)
│   ├── BusinessName (Text)
│   ├── Address (Page)
│   │   ├── Street (Text)
│   │   ├── City (Text)
│   │   ├── State (Text)
│   │   └── ZIP (Text)
│   ├── Contact (Page)
│   │   ├── Email (Text)
│   │   └── Phone (Text)
│   └── PriorClaims (Page List)
│       ├── Date (Date)
│       ├── Type (Text)
│       └── Amount (Decimal)
├── AssetDetails (Page)
│   ├── PropertyType (Text)
│   ├── SquareFootage (Integer)
│   ├── ConstructionType (Text)
│   └── YearBuilt (Integer)
├── CoverageDetails (Page)
│   ├── CoverageType (Text)
│   ├── CoverageLimit (Integer)
│   ├── Deductible (Integer)
│   └── AddOns (Page List)
│       └── AddOnType (Text)
├── PaymentPreferences (Page)
│   ├── PaymentMode (Text)
│   └── AutoRenew (True/False)
├── Documents (Page List)
│   ├── DocumentType (Text)
│   └── DocumentURL (Text)
└── QuoteDetails (Page)
    ├── QuoteID (Text)
    ├── Premium (Decimal)
    ├── CoverageLimit (Integer)
    ├── Deductible (Integer)
    ├── ValidUntil (Date)
    ├── QuotePDFUrl (Text)
    └── AddOns (Page List)
        └── AddOnType (Text)
```

### Queue Processor Configuration

```
Queue Processor Name: QuoteRequestAsyncProcessor
Queue Name: QuoteRequestAsyncQueue
Processing Mode: Standard
Batch Size: 10
Max Threads: 5
Error Handling: Retry 3 times with exponential backoff
```

### Authentication Profiles

```
// For external API calls
Authentication Profile Name: InsuranceProviderAuth
Authentication Type: OAuth 2.0 / API Key
Token URL: [Provider specific]
Client ID: [From provider]
Client Secret: [From provider]
```

## Testing Scripts

### Test All 4 Components

```bash
# Test 1: External Quote Request Connector
curl -X POST http://localhost:8080/prweb/api/insurance/quotes/request \
  -H "Content-Type: application/json" \
  -d '{
    "requestId": "QUOTE-20250910-001",
    "eligibilityResponseId": "ELIG-20250910-001",
    "customerDetails": {
      "name": "John Doe",
      "businessName": "Doe Enterprises",
      "address": {
        "street": "123 Main St",
        "city": "Austin",
        "state": "TX",
        "zip": "78701"
      },
      "contact": {
        "email": "john.doe@example.com",
        "phone": "512-555-1234"
      }
    },
    "assetDetails": {
      "propertyType": "Retail Store",
      "squareFootage": 2500
    },
    "coverageDetails": {
      "coverageType": "General Liability",
      "coverageLimit": 1000000,
      "deductible": 500
    }
  }'

# Test 2: Quote Retrieval by Queue ID
curl -X POST http://localhost:8080/prweb/api/insurance/quotes/retrieve \
  -H "Content-Type: application/json" \
  -d '{
    "queueId": "QUEUE-20250910-001",
    "requestId": "QUOTE-20250910-001",
    "agencyId": "USAA-AGENCY-001"
  }'

# Test 3: Mock API Quote Request
curl -X POST https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api/quotes \
  -H "Content-Type: application/json" \
  -d '{
    "queueId": "QUEUE-TEST-001",
    "requestId": "QUOTE-TEST-001",
    "status": "Quote Processing",
    "customerDetails": {
      "name": "Test Customer",
      "businessName": "Test Business"
    }
  }'

# Test 4: Mock API Quote Retrieval
curl "https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api/quotes?queueId=QUEUE-TEST-001"
```

## Integration Workflow

### Complete End-to-End Process Flow

```
1. [Document Upload] → IDP Processing
   ↓
2. [Assignment] Review Extracted Data
   ↓
3. [Connector: Component 1] Quote Request (Async)
   ├─ Call External Insurance APIs
   ├─ Receive Queue IDs
   └─ Store for tracking
   ↓
4. [Wait/Timer] Async Processing Delay
   ↓
5. [Connector: Component 2] Quote Retrieval (Queue)
   ├─ Check all Queue IDs
   ├─ Retrieve completed quotes
   └─ Update case with results
   ↓
6. [Assignment] Quote Comparison & Selection
   ↓
7. [End] Quote Selection Complete
```

### Activity for Complete Workflow (MainQuoteProcess)

```java
// Step 1: Extract data from uploaded document
Activity: ProcessUploadedDocument
Method: Call
Parameters: Current page

// Step 2: Validate extracted customer and asset data
Activity: ValidateExtractedData
Method: Call
Parameters: Current page

// Step 3: Check eligibility with multiple providers
Activity: CheckMultiProviderEligibility
Method: Call
Parameters: Current page

// Step 4: Request quotes from eligible providers (Component 1)
For Each .EligibleProviders {
    Activity: CallQuoteRequestAsyncConnector
    Method: Call
    Parameters:
      Provider: .ProviderID
      CustomerData: Current page

    // Store queue ID for later retrieval
    Property-Set
    Property: .QuoteRequests(+).QueueID
    Value: @ConnectorResponse.queueId

    Property-Set
    Property: .QuoteRequests().ProviderID
    Value: .ProviderID

    Property-Set
    Property: .QuoteRequests().Status
    Value: "Processing"
}

// Step 5: Wait for quote processing (configurable delay)
Activity: Wait-For-Quote-Processing
Method: Call
Parameters:
  WaitTime: 24 hours
  PeriodicCheck: Every 1 hour

// Step 6: Retrieve all quotes using queue IDs (Component 2)
For Each .QuoteRequests {
    If .Status == "Processing" Then
        Activity: CallQuoteRetrievalQueueConnector
        Method: Call
        Parameters:
          QueueID: .QueueID

        // Update status based on retrieval result
        If @ConnectorResponse.status == "Quote Ready" Then
            .Status = "Ready"
            .QuoteDetails = @ConnectorResponse.quoteDetails
        Else If @ConnectorResponse.status == "Quote Failed" Then
            .Status = "Failed"
            .FailureReason = @ConnectorResponse.reason
        End-If
    End-If
}

// Step 7: Present quotes for comparison
Activity: PrepareQuoteComparison
Method: Call
Parameters: Current page

// Step 8: Create assignment for quote selection
Activity: CreateQuoteSelectionAssignment
Method: Call
Parameters: Current page
```

## Error Handling and Monitoring

### Comprehensive Error Handling Activity

```java
// Activity: HandleQuoteProcessingErrors

// Step 1: Categorize error type
Decision: Error Category
When: .ErrorType == "ConnectivityError"
    Activity: HandleConnectivityError
    Method: Call
When: .ErrorType == "AuthenticationError"
    Activity: RefreshAuthenticationToken
    Method: Call
When: .ErrorType == "ValidationError"
    Activity: HandleValidationError
    Method: Call
When: .ErrorType == "TimeoutError"
    Activity: HandleTimeoutError
    Method: Call
Otherwise:
    Activity: HandleUnknownError
    Method: Call

// Step 2: Implement retry logic
Decision: Should retry?
When: .RetryAllowed == "true" AND .RetryCount < 3
    Property-Set
    Property: .RetryCount
    Value: @(.RetryCount + 1)

    // Exponential backoff
    Property-Set
    Property: .NextRetryTime
    Value: @AddMinutes(@Current DateTime, (.RetryCount * 15))

    Activity: ScheduleRetry
    Method: Call
Otherwise:
    Property-Set
    Property: .ProcessingStatus
    Value: "Failed"

    Activity: NotifyProcessingFailure
    Method: Call

// Step 3: Log error for monitoring
Activity: LogErrorForMonitoring
Method: Call
Parameters:
  ErrorType: .ErrorType
  ErrorMessage: .ErrorMessage
  RetryCount: .RetryCount
  CaseID: .pyID
```

### Monitoring Data Page (D_QuoteProcessingMetrics)

```javascript
// Data Page: D_QuoteProcessingMetrics
// Refresh Strategy: Always
// Source: Database

SELECT
    COUNT(*) as TotalQuotes,
    SUM(CASE WHEN Status = 'Quote Ready' THEN 1 ELSE 0 END) as CompletedQuotes,
    SUM(CASE WHEN Status = 'Quote Processing' THEN 1 ELSE 0 END) as ProcessingQuotes,
    SUM(CASE WHEN Status = 'Quote Failed' THEN 1 ELSE 0 END) as FailedQuotes,
    AVG(CASE WHEN Status = 'Quote Ready'
        THEN EXTRACT(EPOCH FROM (CompletedTime - RequestTime))/3600
        ELSE NULL END) as AvgProcessingHours
FROM QuoteRequests
WHERE RequestTime >= CURRENT_DATE - INTERVAL '7 days'
```

## Configuration Summary

### All 4 Components Summary Table

| Component | Type           | Purpose                      | Input            | Output                 |
| --------- | -------------- | ---------------------------- | ---------------- | ---------------------- |
| 1         | REST Connector | External Quote Request       | Case Data        | Queue ID               |
| 2         | REST Connector | External Quote Retrieval     | Queue ID         | Quote Details          |
| 3         | REST Service   | Internal Quote Request API   | JSON Request     | Queue ID Response      |
| 4         | REST Service   | Internal Quote Retrieval API | Queue ID Request | Quote Details Response |

### Connector URLs Configuration

```javascript
// Component 1 & 2 (External Connectors)
Base URL: https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api

// Component 3 & 4 (Internal Services)
Base URL: http://localhost:8080/prweb/api/insurance
```

### Authentication Configuration

```javascript
// External APIs (Components 1 & 2)
Authentication: None (Mock API)
Headers: Content-Type: application/json

// Internal Services (Components 3 & 4)
Authentication: Pega Authentication
Authorization: Based on operator privileges
```

## Deployment Checklist

### Pre-Deployment Validation

```
□ All 4 components created and configured
□ Data transforms tested with sample data
□ Mock API repository updated with db.json
□ Case type properties match API specifications
□ Error handling activities implemented
□ Queue processor configured and enabled
□ Authentication profiles set up
□ Monitoring data pages created
□ Test scenarios executed successfully
```

### Testing Checklist

```
□ Component 1: Quote request connector test passed
□ Component 2: Quote retrieval connector test passed
□ Component 3: Internal quote request service test passed
□ Component 4: Internal quote retrieval service test passed
□ End-to-end workflow test completed
□ Error scenarios tested (timeouts, failures)
□ Performance testing under load
□ Security testing (authentication, authorization)
□ Mock API data consistency verified
```

### Go-Live Checklist

```
□ Production mock API endpoints configured
□ Pega environment ready (Dev/Test/Prod)
□ Monitoring dashboards configured
□ Alert notifications set up
□ Documentation updated and distributed
□ Training materials prepared
□ Support procedures documented
□ Rollback plan prepared
```

## Performance Optimization

### Connector Optimization Settings

```javascript
// Connection Pooling
Max Connections: 20
Connection Timeout: 30 seconds
Read Timeout: 30 seconds
Keep-Alive: Enabled

// Caching Strategy
Response Caching: 5 minutes (for company data)
Authentication Token Caching: 50 minutes
Eligibility Results Caching: 1 hour

// Queue Processing
Batch Size: 10 quotes per batch
Max Threads: 5 concurrent threads
Processing Frequency: Every 5 minutes
```

### Database Optimization

```sql
-- Indexes for performance
CREATE INDEX idx_quote_queue_id ON QuoteRequests(QueueID);
CREATE INDEX idx_quote_status ON QuoteRequests(Status);
CREATE INDEX idx_quote_timestamp ON QuoteRequests(RequestTimestamp);
CREATE INDEX idx_customer_id ON CustomerDetails(CustomerID);
```

This completes the implementation of all 4 components with exact API specifications, comprehensive error handling, monitoring, and deployment guidelines. Each component is designed to work seamlessly with your mock API and follows Pega best practices for production readiness.
