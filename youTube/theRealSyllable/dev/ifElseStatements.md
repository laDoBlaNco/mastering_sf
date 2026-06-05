<!-- ifElseStatements.md -->
# The Trailmaker | Salesforce Developer Series - 9 & 10| Controlling Statement - IF - ELSE - SWITCH

---

### Commenting

Both single and multi-line comments are supported in Apex code:
- // is used for single line comments
- /* This comment can wrap over multiple
  lines without getting interpreted by the
  compiler */

### Conditions

```apex
if(condition){
    do sometihng;
} else if(condition2) {
    do something else;
} else{
    do another something else;
}
```

### Switch

```apex
// the switch when statement (not 'case' for syntax reasons. SF already has a Case object)
switch on expression{
    when value1{    // when block 1
        // code block 1
    }
    when value2{    // when block 2
        // code block 2
    }
    when else{      // default block, optional
        // code block 3
    }
}
```

Apex `switch` statement expressions can be one of the following types:
- Integer
- Long
- sObject
- String
- Enum

