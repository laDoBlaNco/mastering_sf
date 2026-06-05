<!-- basicLogicalOperators.md -->
# The Trailmaker | Salesforce Developer Series - 6 thru 8 | Mastering the Basic Logical Operators

---

### Basic Operators

- Basically used for mathematical operations
    - \+ : addition
    - \- : subtraction
    - \* : multiplication
    - / : division
    - = : value assignment

### Logical Operators

- Basically used in the comparison of objects to get Boolean results
    - == : Equal To
    - && : AND
        - true && true = true
        - true && false = false
        - false && true = false
        - false && false = false
    - || : OR
        - true || true = true
        - true || false = true
        - false || true = true
        - false || false = false
    - < : Less Than
    - \> : Greater Than
    - <= : Less Than or Equal To
    - \>= : Greater Than or Equal To
    - != : Not Equal To

### Advanced Operators
- Some of the more advanced operators
    - ++ (increment operator - there is also +=, -=, *=, /=, etc for increment and decrement assignment)
    - \-- (decrement operator)
    - ! (inverts the value of a Boolean so that true is false and false is true. Negation)
    - () using them to control the precedence of an operation

### Apex Operator Precedence

|Precedence|Operators|Description|
|---|---|---|
|1|`{} () ++ --`|Grouping and prefix increments and decrements|
|2|`! -x +x (type) new`|Unary negation, type cast and object creation|
|3|`* /`|Multiplication and Division|
|4|`+ -`|Addition and Subtraction|
|5|`< <= > >= instanceof`|Greater-than and less-than comparisons, referernce tests|
|6|`== !=`|Comparisons: equal and not-equal|
|7|`&&`|Logical AND|
|8|`\|\|`|Logical OR|
|9|`= += -= *= /= &=`|Assignment operators|

