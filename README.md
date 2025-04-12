# CDAP Wrangler Enhancement - Aggregate Stats Directive with Units

This enhancement introduces a new directive to **CDAP Wrangler** that enables users to compute **aggregate statistics** (`min`, `max`, `avg`, `sum`, `count`) on grouped data, with support for **units** such as `BYTE_SIZE` and `TIME_DURATION`.

## ✨ Features

### 🔤 Grammar Modification (Directives.g4)

#### 📍 Location:
`wrangler-core/src/main/antlr4/.../Directives.g4`

#### 🎯 Objective:
Support parsing of `BYTE_SIZE` and `TIME_DURATION` in Wrangler directives by enhancing the ANTLR grammar.

---

### 1. ➕ Add Lexer Rules

Add new tokens for units using lexer rules.

```antlr
// Units
BYTE_SIZE      : [0-9]+ ('.' [0-9]+)? BYTE_UNIT ;
TIME_DURATION  : [0-9]+ ('.' [0-9]+)? TIME_UNIT ;

// Helper fragments
fragment BYTE_UNIT : [KMGTP]? ('B' | 'b') ;
fragment TIME_UNIT : ('ms' | 's' | 'sec' | 'm' | 'min' | 'h' | 'hr' | 'd' | 'day') ;

value
  : STRING
  | NUMBER
  | BYTE_SIZE
  | TIME_DURATION
  ;

byteSizeArg
  : BYTE_SIZE
  ;

timeDurationArg
  : TIME_DURATION
  ;
to run 
mvn clean compile
![image](https://github.com/user-attachments/assets/4a6cf6fa-346b-4796-9c06-94fd15d81fad)
## 📍 API Updates (wrangler-api module)

#### 🎯 Objective:
Introduce new Java classes for `ByteSize` and `TimeDuration` to extend the `Token` class, and update the API to support these token types.

---

### 1. ➕ Create New Java Classes

We will create two classes, `ByteSize.java` and `TimeDuration.java`, which extend the `Token` class. These classes will parse tokens like `"10KB"`, `"150ms"` and provide methods to retrieve the value in a canonical unit (e.g., bytes for `ByteSize`, milliseconds for `TimeDuration`).

#### Example: `ByteSize.java`
```java
package io.cdap.wrangler.api.parser;

public class ByteSize extends Token {
    private final long valueInBytes;

    public ByteSize(String token) {
        // Parse the token string (e.g., "10KB", "2MB")
        this.valueInBytes = parseByteSize(token);
    }

    private long parseByteSize(String token) {
        // Parse logic: Handle different byte units (KB, MB, GB, etc.)
        long sizeInBytes = 0;
        String unit = token.replaceAll("[0-9]", "").toUpperCase();
        double value = Double.parseDouble(token.replaceAll("[^0-9.]", ""));
        
        switch (unit) {
            case "KB":
                sizeInBytes = (long) (value * 1024);
                break;
            case "MB":
                sizeInBytes = (long) (value * 1024 * 1024);
                break;
            case "GB":
                sizeInBytes = (long) (value * 1024 * 1024 * 1024);
                break;
            // Add other units as needed
            default:
                sizeInBytes = (long) value; // Assuming bytes if no unit provided
        }
        return sizeInBytes;
    }

    public long getBytes() {
        return valueInBytes;
    }
}
Run the tests:
mvn test
![image](https://github.com/user-attachments/assets/d0845e00-32dd-44cb-b126-c54a87acd386)
