# Insurance Quote API Documentation

## Overview
This document outlines the API endpoints for the Insurance Quote system, including eligibility checks, quote generation, and quote fetching across multiple service instances.

---

## API Endpoints

### 1. Eligibility Check
**Base URL:** `https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api`

**Endpoint:** `/eligibility?requestId={requestId}`

**Method:** GET

**Description:** Check eligibility for insurance quotes based on request ID

**Parameters:**
- `requestId` (string, required): Unique identifier for the eligibility request

**Example:**
```
GET https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api/eligibility?requestId=ELIG-20250903-001
```

---

### 2. Generate Quotes
**Base URL:** `https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api`

**Endpoint:** `/quotes?eligibilityResponseId={eligibilityResponseId}`

**Method:** GET

**Description:** Generate insurance quotes based on eligibility response

**Parameters:**
- `eligibilityResponseId` (string, required): Response ID from eligibility check

**Example:**
```
GET https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api/quotes?eligibilityResponseId=ELIG-20250903-001
```

---

## Fetch Quotes by Queue ID

### 3. Fetch Quotes - Service Instance 1
**Base URL:** `https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api-2`

**Endpoint:** `/fetchQuotes?queueId={queueId}`

**Method:** GET

**Description:** Fetch insurance quotes for queue IDs in the range QUEUE-20250903-001 to QUEUE-20250903-005

**Queue ID Range:** `QUEUE-20250903-001` ↔ `QUEUE-20250903-005`

**Repository:** [insurance-quote-api-2/db.json](https://github.com/SanjeevSaniel/insurance-quote-api-2/blob/main/db.json)

**Supported Queue IDs:**
- QUEUE-20250903-001 (GEICO)
- QUEUE-20250903-002 (Allstate)
- QUEUE-20250903-003 (The Hartford)
- QUEUE-20250903-004 (State Farm)
- QUEUE-20250903-005 (Progressive)

**Example:**
```
GET https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api-2/fetchQuotes?queueId=QUEUE-20250903-001
```

---

### 4. Fetch Quotes - Service Instance 2
**Base URL:** `https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api-3`

**Endpoint:** `/fetchQuotes?queueId={queueId}`

**Method:** GET

**Description:** Fetch insurance quotes for queue IDs in the range QUEUE-20250903-006 to QUEUE-20250903-010

**Queue ID Range:** `QUEUE-20250903-006` ↔ `QUEUE-20250903-010`

**Repository:** [insurance-quote-api-3/db.json](https://github.com/SanjeevSaniel/insurance-quote-api-3/blob/main/db.json)

**Supported Queue IDs:**
- QUEUE-20250903-006 (Liberty Mutual)
- QUEUE-20250903-007 (Nationwide)
- QUEUE-20250903-008 (Travelers)
- QUEUE-20250903-009 (American Family Insurance)
- QUEUE-20250903-010 (Farmers Insurance)

**Example:**
```
GET https://my-json-server.typicode.com/SanjeevSaniel/insurance-quote-api-3/fetchQuotes?queueId=QUEUE-20250903-006
```

---

## API Flow

1. **Step 1:** Check eligibility using `/eligibility` endpoint
2. **Step 2:** Generate quotes using `/quotes` endpoint with eligibility response
3. **Step 3:** Fetch specific quotes using `/fetchQuotes` endpoint with appropriate service instance based on queue ID range

---

## Service Distribution

| Service Instance | Queue ID Range | Insurance Companies |
|------------------|----------------|-------------------|
| **insurance-quote-api-2** | QUEUE-20250903-001 to QUEUE-20250903-005 | GEICO, Allstate, Hartford, State Farm, Progressive |
| **insurance-quote-api-3** | QUEUE-20250903-006 to QUEUE-20250903-010 | Liberty Mutual, Nationwide, Travelers, American Family, Farmers |

---

## Notes

- All endpoints use GET method
- Queue IDs follow the format: `QUEUE-YYYYMMDD-NNN`
- Service instances are distributed based on queue ID ranges
- Each service instance handles 5 insurance companies
