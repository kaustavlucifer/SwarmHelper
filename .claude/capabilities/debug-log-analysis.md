# Capability: Debug Log Analysis

## Log Format

Salesforce debug logs are pipe-delimited. Each line:
```
TIMESTAMP (NANOS)|EVENT_TYPE|LINE_REF|...additional fields...
```

---

## Analysis Approach

1. **Start with FATAL_ERROR** — terminated the transaction
2. **Work backwards** — find the EXCEPTION_THROWN that caused it
3. **Check call stack** — what code path led to the exception
4. **Look for patterns** (see below)
5. **Check governor limits** even without LimitException — near-threshold = risk
6. **Flag duplicate SOQL** — optimization opportunity

---

## Key Events

### Exceptions
```
TIMESTAMP|EXCEPTION_THROWN|[LINE]|ExceptionType: message
TIMESTAMP|FATAL_ERROR|ExceptionType: message
```
Also check for: `Exception:\s`, `Assertion Failed`, `threw an unexpected exception`

### Stack Traces (follow exception lines)
```
Class.ClassName.methodName: line N, column N
Trigger.TriggerName: line N, column N
AnonymousBlock: line N, column N
```

### SOQL
```
TIMESTAMP|SOQL_EXECUTE_BEGIN|[LINE]|SELECT ... FROM ...
TIMESTAMP|SOQL_EXECUTE_END|[LINE]|Rows:N
```
Flag: duplicate queries, high row counts, SOQL in loops

### DML
```
TIMESTAMP|DML_BEGIN|[LINE]|Op:Insert|Type:ObjectName|Rows:N
TIMESTAMP|DML_END|[LINE]
```

### Callouts
```
TIMESTAMP|CALLOUT_REQUEST|[LINE]|method=POST endpoint=https://...
TIMESTAMP|CALLOUT_RESPONSE|[LINE]|StatusCode=200
```
No CALLOUT_RESPONSE = timeout/connection failure

### Governor Limits
```
TIMESTAMP|LIMIT_USAGE_FOR_NS|(default)|
  Number of SOQL queries: 95 out of 100
  Maximum CPU time: 8500 out of 10000
```
Flag: ≥90% = Critical, ≥75% = Warning

### Validation Failures
```
TIMESTAMP|VALIDATION_FAIL|[LINE]|RuleName|ObjectType
```

---

## Known Exception Types

```
System.NullPointerException    System.DmlException
System.QueryException          System.LimitException
System.SObjectException        System.ListException
System.TypeException           System.AsyncException
System.JSONException           System.CalloutException
System.StringException         System.MathException
System.NoAccessException       System.IllegalArgumentException
System.AuraHandledException    System.SecurityException
```

---

## Pattern Detection

| Pattern | Meaning | Action |
|---|---|---|
| Same exception repeating N times | Loop failing on each record | Fix root cause |
| SOQL inside a loop (duplicates) | N+1 query anti-pattern | Query before loop, use Map |
| >90% governor limit | Will break on larger data | Optimize |
| DML before callout | Transaction order violation | Callout first, DML after |
| Double namespace prefix | Config/code error | Fix source |
| "Field not editable" on system fields | Copying all fields including read-only | Filter non-writable |

---

## Deduplication

Group identical exception messages:
- Unique exception type + message
- Number of occurrences
- First line number
- Stack trace (from first occurrence)

57 exceptions that are 7 unique patterns × N repetitions → report as 7 unique with counts.

---

## PII Redaction (MANDATORY)

Before analyzing or reporting debug log content:

1. **NEVER output raw query result values** — report the query structure and row counts, not the data
2. **Mask identifiable data in VARIABLE_ASSIGNMENT lines:**
   - Email addresses: `user@domain.com` → `[EMAIL_REDACTED]`
   - Phone numbers: any 10+ digit sequence → `[PHONE_REDACTED]`
   - SSN patterns (XXX-XX-XXXX): → `[SSN_REDACTED]`
   - Credit card patterns (16 digits): → `[CC_REDACTED]`
   - Customer names in string values → `[NAME_REDACTED]`
3. **For SOQL results:** Report object name, field names queried, row count — NOT field values
4. **For USER_DEBUG lines:** Report the debug level and class/method context, redact any customer PII in the message
5. **Exception messages are OK** — class names, method names, error types, line numbers are not PII
6. **Org IDs and Record IDs are OK** — these are internal identifiers, not customer PII

**Safe to report:** Class names, method names, line numbers, exception types, governor limit counts, SOQL query structure, object names, field API names, execution flow, timing data, trigger names, namespace prefixes.

**Never report:** Customer names, email addresses, phone numbers, physical addresses, financial account numbers, SSNs, date of birth, health information, or any field value that identifies a specific person.

---

## Extracting Org ID from Log

Debug logs may contain org ID in header lines:
- `Organization: 00Dxx...`
- `OrgId=00Dxx...`
- Pattern: `00D[a-zA-Z0-9]{12,15}`
