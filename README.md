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
![Screenshot 2025-04-11 201359](https://github.com/user-attachments/assets/cba1c526-612d-4a9a-a0df-261560a58da8)

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
 
(c) Core Parser Updates (wrangler-core module):
 I’ve already: Created new grammar rules in directive.g4 (Step 3a). Now we need to visit and process those parsed tokens in Java.
1. Find or Create the Visitor Class
Go to:
wrangler-core/src/main/java
Look for a file similar to: RecipeVisitor.java
RecipeEvaluator.java
You’re looking for a class that extends RecipeBaseVisitor (or RecipeBaseListener).

2. Add or Modify Visit Methods
Let’s say in diective.g4, I have  added something like:
byteSizeArg: BYTE_SIZE;
timeDurationArg: TIME_DURATION;


Than in java:
@Override
public Object visitByteSizeArg(RecipeParser.ByteSizeArgContext ctx) {
    String value = ctx.getText(); // e.g., "10MB"
    long bytes = convertToBytes(value); // helper function you write
    return new Token(TokenType.BYTE_SIZE, bytes);
}

@Override
public Object visitTimeDurationArg(RecipeParser.TimeDurationArgContext ctx) {
    String value = ctx.getText(); // e.g., "5ms"
    long millis = convertToMillis(value); // helper function
    return new Token(TokenType.TIME_DURATION, millis);
}

________________________________________
 3. Write Helper Functions
Add helper methods like:
/**
 * Converts a string representing a size (e.g., "10KB", "5MB", "2GB") to its equivalent size in bytes.
 *
 * @param input The size string (e.g., "10KB", "5MB", "2GB").
 * @return The size in bytes as a long value.
 */
private long convertToBytes(String input) {
    // Check if the input ends with "KB" and convert to bytes
    if (input.endsWith("KB")) return Long.parseLong(input.replace("KB", "")) * 1024;
    
    // Check if the input ends with "MB" and convert to bytes
    if (input.endsWith("MB")) return Long.parseLong(input.replace("MB", "")) * 1024 * 1024;
    
    // Check if the input ends with "GB" and convert to bytes
    if (input.endsWith("GB")) return Long.parseLong(input.replace("GB", "")) * 1024 * 1024 * 1024;
    
    // If no unit is found, return the value as it is (fallback)
    return Long.parseLong(input); 
}

/**
 * Converts a string representing a duration (e.g., "10ms", "5s", "2m") to its equivalent duration in milliseconds.
 *
 * @param input The duration string (e.g., "10ms", "5s", "2m").
 * @return The duration in milliseconds as a long value.
 */
private long convertToMillis(String input) {
    // Check if the input ends with "ms" and return the value in milliseconds
    if (input.endsWith("ms")) return Long.parseLong(input.replace("ms", ""));
    
    // Check if the input ends with "s" (seconds) and convert to milliseconds
    if (input.endsWith("s")) return Long.parseLong(input.replace("s", "")) * 1000;
    
    // Check if the input ends with "m" (minutes) and convert to milliseconds
    if (input.endsWith("m")) return Long.parseLong(input.replace("m", "")) * 1000 * 60;
    
    // If no unit is found, return the value as it is (fallback)
    return Long.parseLong(input); 
}

________________________________________
4. Add to TokenGroup
If youre using a TokenGroup object to collect parsed tokens:
tokenGroup.add(new Token(TokenType.BYTE_SIZE, bytes));
or

tokenGroup.add(new Token(TokenType.TIME_DURATION, millis));


 To execte the step 3 run the commond

 mvn clean compile exec:java -Dexec.mainClass="io.cdap.wrangler.api.parser.TestDuration"




Output 

1h
3600000


(d) New Directive Implementation (wrangler-core module): 
Great! Let's walk through how to structure the new directive for the wrangler-core module. Here's a full breakdown of what I have done 
________________________________________
Step-by-Step: Implementing the Aggregate Directive
1.	Create a New Directive Class As AggregateStats.java
Name it something like AggregateStats.
@Directive(name = "aggregate-stats", usage = "Aggregates byte sizes and time durations")
public class AggregateStats implements Directive {

2. Define Arguments
In define() method, declare your directive’s usage:
java
CopyEdit
@Override
public UsageDefinition define() {
    return UsageDefinition.builder("aggregate-stats")
        .addRequiredArg("sourceSizeCol")
        .addRequiredArg("sourceTimeCol")
        .addRequiredArg("targetSizeCol")
        .addRequiredArg("targetTimeCol")
        .addOptionalArg("sizeUnit")    // e.g., MB, GB
        .addOptionalArg("timeUnit")    // e.g., seconds, minutes
        .addOptionalArg("aggregationType") // total or average
        .build();
}

3. Initialize Directive with Arguments
java
CopyEdit
private String sourceSizeCol;
private String sourceTimeCol;
private String targetSizeCol;
private String targetTimeCol;
private String sizeUnit = "B";
private String timeUnit = "ns";
private String aggregationType = "total";

@Override
public void initialize(Arguments arguments) {
    sourceSizeCol = arguments.value("sourceSizeCol");
    sourceTimeCol = arguments.value("sourceTimeCol");
    targetSizeCol = arguments.value("targetSizeCol");
    targetTimeCol = arguments.value("targetTimeCol");

    sizeUnit = arguments.valueOrDefault("sizeUnit", "B").toUpperCase();
    timeUnit = arguments.valueOrDefault("timeUnit", "ns").toLowerCase();
    aggregationType = arguments.valueOrDefault("aggregationType", "total").toLowerCase();
}

4. Use ExecutorContext Store to Accumulate Totals
java
CopyEdit
private static final String TOTAL_SIZE_KEY = "agg_total_size";
private static final String TOTAL_TIME_KEY = "agg_total_time";
private static final String COUNT_KEY = "agg_count";

@Override
public List<Row> execute(List<Row> rows, ExecutorContext context) throws DirectiveExecuteException {
    long totalSize = context.getOrDefault(TOTAL_SIZE_KEY, 0L);
    long totalTime = context.getOrDefault(TOTAL_TIME_KEY, 0L);
    int count = context.getOrDefault(COUNT_KEY, 0);

    for (Row row : rows) {
        Object sizeVal = row.getValue(sourceSizeCol);
        Object timeVal = row.getValue(sourceTimeCol);

        long sizeInBytes = UnitParser.parseByteSize(sizeVal.toString()); // You’ll implement this
        long timeInNs = UnitParser.parseTimeDuration(timeVal.toString()); // Implement this too

        totalSize += sizeInBytes;
        totalTime += timeInNs;
        count++;
    }

    context.set(TOTAL_SIZE_KEY, totalSize);
    context.set(TOTAL_TIME_KEY, totalTime);
    context.set(COUNT_KEY, count);

    return Collections.emptyList(); // Aggregates don't emit intermediate rows
}
________________________________________
5. Finalization: Return Aggregate Result
java
CopyEdit
@Override
public List<Row> finalize(ExecutorContext context) {
    long totalSize = context.getOrDefault(TOTAL_SIZE_KEY, 0L);
    long totalTime = context.getOrDefault(TOTAL_TIME_KEY, 0L);
    int count = context.getOrDefault(COUNT_KEY, 1);

    if (aggregationType.equals("average")) {
        totalSize = totalSize / count;
        totalTime = totalTime / count;
    }

    double finalSize = UnitParser.convertBytes(totalSize, sizeUnit); // e.g., to MB
    double finalTime = UnitParser.convertTime(totalTime, timeUnit); // e.g., to seconds

    Row output = RowHelper.newRow()
        .add(targetSizeCol, finalSize)
        .add(targetTimeCol, finalTime)
        .build();

    return Collections.singletonList(output);
}
output

 
6. Helper: UnitParser Class
Make a utility class for parsing strings like 10 MB, 5 sec, 1.5 GB, etc., and converting to canonical unitsGpackage io.cdap.directives;
package io.cdap.directives;

import java.util.Locale;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Utility class to parse byte sizes (e.g., "1.5GB", "200 KB") and 
 * time durations (e.g., "5s", "2.5h") into their base units.
 * 
 * - Byte sizes are converted to bytes.
 * - Time durations are converted to nanoseconds.
 * 
 * It also supports converting base units back into target units 
 * (e.g., bytes to MB, nanoseconds to seconds).
 */
public class UnitParser {

    // Regex patterns to match size and time formats (e.g., "1.5GB", "500 ms")
    private static final Pattern SIZE_PATTERN = Pattern.compile("(\\d+(?:\\.\\d+)?)\\s*(B|KB|MB|GB|TB)?", Pattern.CASE_INSENSITIVE);
    private static final Pattern TIME_PATTERN = Pattern.compile("(\\d+(?:\\.\\d+)?)\\s*(ns|us|ms|s|m|h)?", Pattern.CASE_INSENSITIVE);

    // Multipliers for converting units to base units
    private static final long[] SIZE_MULTIPLIERS = {
        1L,                         // B
        1024L,                      // KB
        1024L * 1024,               // MB
        1024L * 1024 * 1024,        // GB
        1024L * 1024 * 1024 * 1024  // TB
    };

    private static final long[] TIME_MULTIPLIERS = {
        1L,                         // ns
        1_000L,                     // us
        1_000_000L,                 // ms
        1_000_000_000L,             // s
        60L * 1_000_000_000L,       // m
        3600L * 1_000_000_000L      // h
    };

    // Units supported for byte size and time duration
    private static final String[] SIZE_UNITS = {"B", "KB", "MB", "GB", "TB"};
    private static final String[] TIME_UNITS = {"ns", "us", "ms", "s", "m", "h"};

    /**
     * Parses a byte size string and converts it into bytes.
     *
     * @param sizeStr the input size string (e.g., "1.5GB", "1024 KB")
     * @return the size in bytes
     * @throws IllegalArgumentException if the format or unit is invalid
     */
    public static long parseByteSize(String sizeStr) {
        Matcher matcher = SIZE_PATTERN.matcher(sizeStr.trim().toUpperCase(Locale.ROOT));
        if (!matcher.matches()) {
            throw new IllegalArgumentException("Invalid byte size format: " + sizeStr);
        }

        double value = Double.parseDouble(matcher.group(1));
        String unit = matcher.group(2) == null ? "B" : matcher.group(2);

        for (int i = 0; i < SIZE_UNITS.length; i++) {
            if (unit.equals(SIZE_UNITS[i])) {
                return (long) (value * SIZE_MULTIPLIERS[i]);
            }
        }
        throw new IllegalArgumentException("Unknown size unit: " + unit);
    }

    /**
     * Parses a time duration string and converts it into nanoseconds.
     *
     * @param durationStr the input duration string (e.g., "500ms", "2h")
     * @return the time in nanoseconds
     * @throws IllegalArgumentException if the format or unit is invalid
     */
    public static long parseTimeDuration(String durationStr) {
        Matcher matcher = TIME_PATTERN.matcher(durationStr.trim().toLowerCase(Locale.ROOT));
        if (!matcher.matches()) {
            throw new IllegalArgumentException("Invalid time duration format: " + durationStr);
        }

        double value = Double.parseDouble(matcher.group(1));
        String unit = matcher.group(2) == null ? "ns" : matcher.group(2);

        for (int i = 0; i < TIME_UNITS.length; i++) {
            if (unit.equals(TIME_UNITS[i])) {
                return (long) (value * TIME_MULTIPLIERS[i]);
            }
        }
        throw new IllegalArgumentException("Unknown time unit: " + unit);
    }

    /**
     * Converts a byte value into the target unit (e.g., KB, MB).
     *
     * @param bytes the value in bytes
     * @param toUnit the unit to convert to (e.g., "MB")
     * @return the converted value as a double
     * @throws IllegalArgumentException if the unit is invalid
     */
    public static double convertBytes(long bytes, String toUnit) {
        toUnit = toUnit.toUpperCase(Locale.ROOT);
        for (int i = 0; i < SIZE_UNITS.length; i++) {
            if (toUnit.equals(SIZE_UNITS[i])) {
                return bytes / (double) SIZE_MULTIPLIERS[i];
            }
        }
        throw new IllegalArgumentException("Invalid byte unit for conversion: " + toUnit);
    }

    /**
     * Converts a nanosecond value into the target time unit (e.g., ms, s).
     *
     * @param nanos the value in nanoseconds
     * @param toUnit the unit to convert to (e.g., "s")
     * @return the converted value as a double
     * @throws IllegalArgumentException if the unit is invalid
     */
    public static double convertTime(long nanos, String toUnit) {
        toUnit = toUnit.toLowerCase(Locale.ROOT);
        for (int i = 0; i < TIME_UNITS.length; i++) {
            if (toUnit.equals(TIME_UNITS[i])) {
                return nanos / (double) TIME_MULTIPLIERS[i];
            }
        }
        throw new IllegalArgumentException("Invalid time unit for conversion: " + toUnit);
    }
}



Output  Expected Output Row
 
4. Test Case Specification: Aggregation
To test the aggregate-stats directive by passing sample data and recipe, and verifying the aggregation output.
________________________________________
🧪 Sample Input Data (List of Rows)
java
CopyEdit
List<Row> rows = Arrays.asList(
    new Row("data_transfer_size", "1MB").add("response_time", "5min"),
    new Row("data_transfer_size", "500KB").add("response_time", "120s"),
    new Row("data_transfer_size", "2GB").add("response_time", "2h")
);
________________________________________
Sample Recipe
String[] recipe = new String[] {
    "aggregate-stats :data_transfer_size :response_time total_size_mb total_time_sec MB seconds total average"
};
________________________________________
 Expected Output
Since this is an aggregate directive, it will emit a single row with computed values.
Manual Conversion Reference:
• Byte Sizes:
o 1MB = 1,048,576 bytes
o 500KB = 512,000 bytes
o 2GB = 2,147,483,648 bytes
o Total = 2,149,044,224 bytes
o MB = 2049.0 MB
• Time Durations:
o 5min = 300s
o 120s = 120s
o 2h = 7200s
o Total = 7620s
o Average = 2540s


Row expectedRow = new Row("total_size_mb", 2049.0);
expectedRow.add("total_time_sec", 2540.0);

JUnit Test Example

@Test
public void testAggregateStatsDirective() throws Exception {
    // Define the Wrangler recipe for the aggregate-stats directive.
    // Syntax: aggregate-stats :inputCols outputCol1 outputCol2 outputUnit1 outputUnit2 aggType1 aggType2
    // In this case: aggregate byte size and time duration, convert to MB and seconds, aggregate using total and average
    String[] recipe = new String[] {
        "aggregate-stats :data_transfer_size :response_time total_size_mb total_time_sec MB seconds total average"
    };

    // Prepare input rows with mixed unit values for size and time
    List<Row> rows = Arrays.asList(
        new Row("data_transfer_size", "1MB").add("response_time", "5min"),    // 1 MB, 300 sec
        new Row("data_transfer_size", "500KB").add("response_time", "120s"),  // 0.5 MB, 120 sec
        new Row("data_transfer_size", "2GB").add("response_time", "2h")       // 2048 MB, 7200 sec
    );

    // Execute the Wrangler transformation using the defined recipe
    TestingRig rig = new TestingRig(recipe);
    List<Row> results = rig.execute(rows);

    // Validate that a single output row is returned
    assertEquals(1, results.size());

    // Retrieve the result row
    Row result = results.get(0);

    // Validate the total aggregated size in MB: (1 + 0.5 + 2048) = 2049.5 ~ 2049.0
    assertEquals(2049.0, (Double) result.getValue("total_size_mb"), 0.1);

    // Validate the average time in seconds: (300 + 120 + 7200) / 3 = 2540 sec
    assertEquals(2540.0, (Double) result.getValue("total_time_sec"), 0.1);
}


Variants to Test
•	average vs total
•	Output in GB, KB, etc.
•	Durations in ms, min, h, etc.
•	Different percentiles: median, p95, p99
•	Empty input (should return 0 or null)
•	Invalid unit formats (should throw exception or log error)
 
AI Tools Usage
Tool Used: ChatGPT (OpenAI)
Purpose: Assistance with backend design, code implementation, ANTLR grammar modification, directive logic, and test writing.
________________________________________
Prompts Used
Help me understand the Zeotap Wrangler backend assignment. I need to implement a new directive called `aggregate-stats` that can aggregate byte sizes and time durations from multiple columns. It should accept optional output units and aggregation type. What should be my approach?
2. For modifying ANTLR grammar to support BYTE_SIZE and TIME_DURATION:
Here's my existing ANTLR grammar for a custom DSL. I want to add support for BYTE_SIZE (like 10MB, 2.3GB) and TIME_DURATION (like 5s, 3m, 1.5h). Help me update the lexer and parser rules.
3. For implementing the directive logic in Java:
How do I create a new directive in CDAP Wrangler that aggregates multiple byte size or time duration columns, optionally converts the unit, and stores the result in a new column?
4. For converting byte and time units:
Give me utility methods in Java to convert values with units like 'MB', 'GB', 's', 'm', 'h' into bytes and seconds respectively, and back to desired units.
5. For writing unit tests:
Help me write JUnit test cases for a directive that computes the sum, average, max, or min of multiple columns containing time or byte units.
6. For validating intermediate logic and debugging:
I'm facing an issue where the output column is showing the wrong converted value. Here's my conversion code — 


Prompt.txt file in github


 



 







 

