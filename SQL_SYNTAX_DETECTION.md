# Phase 3 Enhanced Semantic Analysis

## Overview
Phase 3 combines **security vulnerability detection**, **traditional compiler semantic analysis**, and **SQL syntax validation** to catch security issues, logic errors, and typos before runtime.

## All Features Explained

### 1. Symbol Table Management
**What it does:** Maintains a hierarchical database of all variables, functions, classes, and imports in your code, organized by scope (global, function-level, class-level).

**How it works:** As the analyzer walks through your code's Abstract Syntax Tree (AST), it creates nested scope structures. When entering a function, a new scope is pushed; when exiting, it's popped. Each scope tracks what symbols (variables, parameters) are defined within it.

**Why it's useful:** This is the foundation for all other semantic checks. Without knowing what's defined where, you can't detect undefined variables or duplicates. It mirrors how Python's own interpreter manages namespaces.

**Example:**
```python
x = 10              # Global scope: x
def foo(y):         # Function scope: y (parameter)
    z = x + y       # Function scope: z (local variable)
    return z
```
Symbol table tracks: Global {x, foo}, foo scope {y, z}

---

### 2. Type Inference System
**What it does:** Automatically determines the data type of variables by analyzing how they're created and used, without requiring explicit type annotations.

**How it works:** When you assign a value to a variable, the analyzer inspects the right-hand side. If it's a literal like `[1,2,3]`, it infers `list`. If it's a function call like `dict()`, it infers `dict`. The system has 15+ built-in type mappings for common Python patterns.

**Why it's useful:** Type information helps catch bugs early. If you later try to call `.append()` on something inferred as a `dict`, the analyzer knows that's suspicious. It also helps detect when variables change types unexpectedly.

**Example:**
```python
data = [1, 2, 3]        # Inferred: list
config = {"key": "val"} # Inferred: dict
count = 0               # Inferred: int
users = set()           # Inferred: set
```

---

### 3. Undefined Variable Detection
**What it does:** Catches errors where you try to use a variable before it has been assigned a value, preventing common "NameError" runtime crashes.

**How it works:** When the analyzer encounters a variable being loaded (read), it checks if that name exists in the current scope, parent scopes, or Python's builtin names. If not found anywhere, it reports an undefined variable error.

**Why it's useful:** These are among the most common Python bugs. Catching them during static analysis (before running code) saves debugging time. Typos in variable names are detected immediately.

**Example:**
```python
def calculate():
    result = dta + 10  # ERROR: 'dta' undefined (typo for 'data')
    return result
```

---

### 4. Duplicate Definition Detection
**What it does:** Identifies when you define the same function, class, or import multiple times in the same scope, which usually indicates copy-paste errors or refactoring mistakes.

**How it works:** As each function, class, or import is encountered, the analyzer checks if that name already exists in the current scope's symbol table. If yes, it flags a duplicate. This check is scope-aware (duplicates in different functions are OK).

**Why it's useful:** Duplicate definitions are confusing and often unintentional. The second definition silently overwrites the first, which can cause unexpected behavior. Python allows it, but it's rarely what you want.

**Example:**
```python
def process_data(x):    # First definition
    return x * 2

def process_data(x):    # ERROR: Duplicate function
    return x * 3        # This overwrites the first one

import json             # First import
import json             # ERROR: Redundant import
```

---

### 5. Control Flow Analysis
**What it does:** Analyzes the execution paths through functions to detect missing return statements and unused parameters/variables that suggest incomplete or buggy code.

**How it works:** 
- **Missing returns:** After parsing a function, checks if it has at least one return statement AND has parameters (suggesting it's meant to compute something). Functions without returns but with parameters likely forgot to return their result.
- **Unused parameters:** Tracks parameter usage; if a parameter is never referenced in the function body, it's flagged as unused.
- **Unused variables:** Tracks variable definitions; if defined but never loaded/used, it's flagged.

**Why it's useful:** Missing returns are logic bugs that cause functions to return `None` unexpectedly. Unused parameters suggest API mismatches or incomplete implementations. Unused variables are code clutter.

**Example:**
```python
def calculate_total(prices):  # ERROR: Missing return
    total = sum(prices)       # Computes but doesn't return!
    # Missing: return total

def process(data, format):    # ERROR: 'format' unused
    return data.upper()       # Never uses 'format' parameter
```

---

### 6. Cyclomatic Complexity Tracking
**What it does:** Calculates a complexity score for each function by counting decision points (branches in the code), helping identify functions that are too complex and hard to maintain.

**How it works:** Starts each function with complexity = 1. Adds +1 for each `if`, `elif`, `for`, `while`, `except`, `with`, boolean operator (`and`, `or`), etc. Higher scores = more execution paths = harder to test and understand.

**Why it's useful:** Complex functions (complexity > 10) are bug-prone and hard to test. This metric helps you identify candidates for refactoring. It's a standard software engineering metric used in code quality tools.

**Example:**
```python
def complex_function(x, y):  # Complexity = 8
    if x > 0:                # +1 = 2
        if y > 0:            # +1 = 3
            for i in range(10):  # +1 = 4
                if i % 2:    # +1 = 5
                    try:
                        pass
                    except:  # +1 = 6
                        pass
        elif y < 0:          # +1 = 7
            pass
    else:                    # Already counted in 'if'
        pass
    return x and y           # +1 = 8
```

---

### 7. SQL Syntax Validation (40+ Patterns)
**What it does:** Scans string literals containing SQL queries and detects common typos in SQL keywords and missing spaces between compound keywords.

**How it works:** 
1. Examines all string constants in your code
2. Checks if the string contains SQL keywords (SELECT, FROM, etc.)
3. Tokenizes the SQL string and compares tokens against a dictionary of 40+ known typos
4. Detects missing spaces in compound keywords (INSERTINTO should be INSERT INTO)
5. Reports exact line numbers with suggested corrections

**Why it's useful:** SQL syntax errors cause runtime database failures that are annoying to debug. Catching them during static analysis saves time. Common typos (SELCT, FORM, WHER) are detected before the query hits the database.

**Typo patterns detected:**
- **Common misspellings:** SELCT→SELECT, SLECT→SELECT, FORM→FROM, FRON→FROM, WHER→WHERE, WHRE→WHERE
- **Missing letters:** INSRT→INSERT, INSER→INSERT, UPDTE→UPDATE, UPDAE→UPDATE, DELET→DELETE
- **Extra letters:** DELLETE→DELETE
- **Transpositions:** JION→JOIN, JON→JOIN, ODER→ORDER, GROPU→GROUP
- **Missing spaces:** INSERTINTO→INSERT INTO, DELETEFROM→DELETE FROM, ORDERBY→ORDER BY, GROUPBY→GROUP BY, SELECTFROM→SELECT FROM

**Example:**
```python
# Multiple SQL errors detected
query = "SELCT name FORM users WHER age > 18 ORDERBY name"
# ERROR Line X: 'SELCT' should be 'SELECT'
# ERROR Line X: 'FORM' should be 'FROM'
# ERROR Line X: 'WHER' should be 'WHERE'
# ERROR Line X: Missing space in 'ORDERBY' (should be 'ORDER BY')

# Correct version (no errors)
query = "SELECT name FROM users WHERE age > 18 ORDER BY name"
```

## Quick Examples

```python
# Undefined Variable
result = data + 1  # ERROR: 'data' undefined

# Duplicate Function
def calc(): pass
def calc(): pass  # ERROR: Duplicate

# Missing Return
def get_total(items):  # ERROR: No return
    total = sum(items)

# SQL Typo
query = "SELCT * FORM users WHER id = 1"
# ERROR: SELCT→SELECT, FORM→FROM, WHER→WHERE
```

## Error Codes
- `UNDEFINED_VARIABLE` - Used before definition
- `DUPLICATE_FUNCTION` - Multiple definitions
- `DUPLICATE_CLASS` - Multiple class definitions
- `DUPLICATE_IMPORT` - Redundant imports
- `MISSING_RETURN` - Function lacks return
- `UNUSED_PARAMETER` - Parameter never used
- `UNUSED_VARIABLE` - Variable never referenced
- `SQL_SYNTAX_ERROR` - SQL keyword typo

## Testing

```bash
cd comptech_fp
python test_enhanced_semantic.py  # All features
python test_sql_syntax.py         # SQL only
```

**Test files**: [test_semantic_analysis.py](TestingFiles/test_semantic_analysis.py) (10 cases), [test_sql_syntax.py](TestingFiles/test_sql_syntax.py) (15 cases)

## Analysis Summary

```python
analyzer = SemanticAnalyzer(filename)
analyzer.analyze()
summary = analyzer.get_semantic_summary()
```

**Output:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 SEMANTIC ANALYSIS SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📂 Scopes: 5  |  📊 Symbols: 23  |  🔴 Findings: 8
   • Undefined: 2  |  Duplicates: 1  |  Returns: 1
   • Unused Params: 3  |  SQL Errors: 1
⚠️  Max Complexity: 8 (calculate_total)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Integration
- ✅ Flask web interface (Frontend/web_app.py)
- ✅ Command-line analysis
- ✅ Batch processing
- ✅ Real-time checking

## Performance
- **O(n)** linear complexity
- **O(1)** dictionary lookups for SQL validation
- No regex overhead
- Handles 1000+ SQL queries efficiently

## Configuration

**Add SQL typo patterns** in [phase3_LRparser.py](Backend/phase3_LRparser.py):
```python
SQL_SYNTAX_ERRORS = {
    'YOURCUSTOM': 'CORRECT',
}
```

**Disable SQL checking**:
```python
# self._check_sql_syntax(node.value, node.lineno)  # Commented
```

## Limitations
- Uppercase SQL only (convention)
- No full SQL grammar validation
- String literals only (not dynamic queries)
- No database schema validation

## Benefits
✅ Early error detection  
✅ Better code quality  
✅ Reduced debugging time  
✅ Automated code review  
✅ Security + quality analysis
