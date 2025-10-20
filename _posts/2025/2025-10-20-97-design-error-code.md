---
layout: post
title: "97. [Draft][Design] - Error code"
date: 2025-10-20 08:02:00 +0000
category: technical
---

In any systems, clear and informative error code is important on understanding the system behaviour, especially on operation and tracing issues. 

In this article, with the help of **Claude.ai**, I'm trying to propose an Error Code design for mutile modules applicaiton. Though, it can be used in Web API application too.

# Context 
On Spring Applicaiton, we have different modules, which can be called Database or Rest API. And these modules can be shared the same tables in database  or the same Rest API. 

Error code have to 
- Descriptive: Include what went wrong and ideally what to do about it
- Consistency/Extensibility: Same format across the entire system and leave room for extension. Ex: Http 4xx is client error, 5xx is server error. 
- Categorized: support application with different moudles, and can be shared the same DB/API resource. 
- **Start minimal, add more when needed**: Don't over-design or use too detail error code.

# Proposal 

Error code format:  

```
MODULE-CATEGORY-SEQUENCE
```

- **MODULE**: Application module (AUTH, USER, ORDER, PAYMENT, etc.)
- **CATEGORY**: Error type (VAL, BIZ, DB, API, SYS, etc.)
- **SEQUENCE**: Specific error number (001, 002, etc.)

Example: ORDER-BIZ-001 → Order module, Business logic, Error #001

## Architecture Decision: Shared vs Module-Specific Errors

**Shared Infrastructure Errors (DB, SYS, NET, API).**

These are NOT module-specific because:
- Same infrastructure used across all modules
- Same handling regardless of module
- Avoids code duplication
- No manual translation needed

Use these across ALL modules:
```
ASSET-DB-001: Database operation failed
ASSET-DB-002: Resource not found

ORD-API-001: Order service failed
ORD-API-002: Order service, submit transaction api failed. (Specific when needed)

PAY-API-001: Payment service failed

APP-SYS-001: Internal server error (This is critical and unexpected error.)
```

**Module-Specific Errors (VAL, BIZ)**
These ARE module-specific because:

- Different validation rules per module
- Different business logic per module
- Need to know which module's business rule failed

Use module prefix:
```
AUTH-VAL-001, AUTH-BIZ-001
SHIP-VAL-001, SHIP-BIZ-001
```

## Validation Errors (VAL)

### Core Philosophy
Validation = checking **input format/structure** before business logic
- Technical checks (format, type, length)
- No business context needed (often, no need external connection like API or DB)
- Happens at the API/input layer

| Code | Description | Example |
|------|-------------|---------|
| VAL-001 | Required field missing | Missing email field |
| VAL-002 | Invalid format | Email doesn't contain @ |
| VAL-003 | Value out of range | Age = -5 or 999 |
| VAL-004 | Type mismatch | Expected number, got string |
| VAL-005 | Too long | String exceeds max length |
| VAL-006 | Too short | Password < 8 characters |


**When NOT to use VAL:**
- ❌ "Email already exists" → Use **BIZ-001** (business rule)
- ❌ "Insufficient funds" → Use **BIZ-002** (business rule)
- ❌ "Cannot delete order" → Use **BIZ-003** (business rule)

---

## Business Logic Errors (BIZ)

### Core Philosophy
Business = **domain rules and state** checking
- Requires database/context check
- Business domain knowledge needed
- Happens after validation passes

| Code | Description | Example |
|------|-------------|---------|
| BIZ-001 | Resource already exists | Email already registered |
| BIZ-002 | Insufficient resources | Not enough balance/stock |
| BIZ-003 | Invalid state transition | Cannot cancel completed order |
| BIZ-004 | Constraint violation | Cannot delete user with active orders |
| BIZ-005 | Business rule violation | Generic business rule failed |
| BIZ-006 | Limit exceeded | Exceeded daily withdrawal limit |
| BIZ-007 | Resource expired | Coupon/offer expired |
| BIZ-008 | Operation not allowed | Action not permitted in current state |


---

## Database (DB) and API Errors

### Core Philosophy
Database/API = **infrastructure failures** you can't control
- Not validation errors
- Not business errors
- Technical failures only

| Code | Description | When It Happens |
|------|-------------|-----------------|
| DB-001 | Database operation failed | Generic DB error, connection lost |
| DB-002 | Resource not found | Query returned no results |
| API-001 | API operation failed | Generic API error, connection lost |

**That's it! Just 2 codes.**

**Why so few?**
- As a consumer, you can't handle most DB/API errors differently
- Specific DB errors (timeout, deadlock) are infrastructure concerns
- Most DB errors = log and return 500
- "Not found" is the only one users care about
- If there is a frequently fail on a specific operation, we can add additional error code there, and start monitoring it. 

# Sample code on Java 

Define the ErrorCode class 
```java
// ========== Enums ==========

public enum Module {
    AUTH,
    USER,
    ORDER,
    PAYMENT,
    PRODUCT,
    NOTIF,
    SYSTEM  // For non-module specific errors
}

public enum ErrorCategory {
    VAL,    // Validation
    BIZ,    // Business Logic
    DB,     // Database (shared)
    API,    // External API (shared)
    SYS     // System (shared)
}

// ========== ErrorCode Class ==========

public class ErrorCode {
    private final String code;
    private final String description;
    private final Module module;
    private final ErrorCategory category;

    public ErrorCode(Module module, ErrorCategory category, int sequence, 
                     String description) {
        // For shared infrastructure errors (DB, API, INFRA, SYS), use SYSTEM module
        if (isSharedCategory(category)) {
            this.code = String.format("%s-%03d", category.name(), sequence);
            this.module = Module.SYSTEM;
        } else {
            // For module-specific errors (VAL, BIZ)
            this.code = String.format("%s-%s-%03d", module.name(), category.name(), sequence);
            this.module = module;
        }
        
        this.category = category;
        this.description = description;
    }
    
    private boolean isSharedCategory(ErrorCategory category) {
        return category == ErrorCategory.DB || 
               category == ErrorCategory.API || 
               category == ErrorCategory.SYS;
    }

    // Getters
    public String getCode() { return code; }
    public String getDescription() { return description; }
    public Module getModule() { return module; }
    public ErrorCategory getCategory() { return category; }

    @Override
    public String toString() {
        return code + ": " + description;
    }
} 
```

Assumse that each module has a fews ErrorCode, then, we can use 1 single class. if the business logic increase, we can separate error code of each module into separate class 

```java 
public class ErrorCodeConstants {
    
    // ==================== SHARED INFRASTRUCTURE ERRORS ====================
    // These are used across ALL modules
    
    // Database Errors (DB-xxx)
    public static final ErrorCode DB_OPERATION_FAILED = 
        new ErrorCode(Module.SYSTEM, ErrorCategory.DB, 1, 
            "Database operation failed");
    
    public static final ErrorCode DB_RESOURCE_NOT_FOUND = 
        new ErrorCode(Module.SYSTEM, ErrorCategory.DB, 2, 
            "Resource not found");
    
    // API Errors (API-xxx)
    public static final ErrorCode API_EXTERNAL_SERVICE_FAILED = 
        new ErrorCode(Module.SYSTEM, ErrorCategory.API, 1, 
            "External service failed");
    
    public static final ErrorCode API_EXTERNAL_SERVICE_TIMEOUT = 
        new ErrorCode(Module.SYSTEM, ErrorCategory.API, 2, 
            "External service timeout");
    
    public static final ErrorCode API_STRIPE_FAILED = 
        new ErrorCode(Module.SYSTEM, ErrorCategory.API, 10, 
            "Stripe payment service failed");
    
    public static final ErrorCode API_EMAIL_SERVICE_FAILED = 
        new ErrorCode(Module.SYSTEM, ErrorCategory.API, 20, 
            "Email service failed");
    
    
    // System Errors (SYS-xxx)
    public static final ErrorCode SYS_INTERNAL_ERROR = 
        new ErrorCode(Module.SYSTEM, ErrorCategory.SYS, 1, 
            "Internal server error");
    
    // ==================== AUTH MODULE ====================
    
    // Validation Errors
    public static final ErrorCode AUTH_MISSING_CREDENTIALS = 
        new ErrorCode(Module.AUTH, ErrorCategory.VAL, 1, 
            "Missing credentials");
    
    // Business Logic Errors
    public static final ErrorCode AUTH_INVALID_CREDENTIALS = 
        new ErrorCode(Module.AUTH, ErrorCategory.BIZ, 1, 
            "Invalid credentials");
    
    public static final ErrorCode AUTH_ACCOUNT_LOCKED = 
        new ErrorCode(Module.AUTH, ErrorCategory.BIZ, 2, 
            "Account locked");
}

```

ApplicationException class. Beside errorCode, we need to add the context object to provide additional information on how the exception is trigger.

```java
public class ApplicationException extends RuntimeException {
    private final ErrorCode errorCode;
    private final Map<String, Object> context;
    
    public ApplicationException(ErrorCode errorCode, String message) {
        this(errorCode, message, null, new HashMap<>());
    }
    
    public ApplicationException(ErrorCode errorCode, String message, Throwable cause) {
        this(errorCode, message, cause, new HashMap<>());
    }
    
    public ApplicationException(ErrorCode errorCode, String message, 
                               Throwable cause, Map<String, Object> context) {
        super(String.format("[%s] %s - %s", 
              errorCode.getCode(), errorCode.getDescription(), message), cause);
        this.errorCode = errorCode;
        this.context = context;
    }
    
    public ErrorCode getErrorCode() { return errorCode; }
    public Map<String, Object> getContext() { return context; }
    
    public ApplicationException addContext(String key, Object value) {
        this.context.put(key, value);
        return this;
    }


}
```









