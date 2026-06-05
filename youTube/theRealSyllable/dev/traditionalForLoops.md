<!-- traditionalForLoops.md -->
# The Trailmaker | Salesforce Developer Series - 11 | Traditional For Loops

---

### Traditional For Loops

```apex
for(init_stmt;exit_condition;increment_stmt){
    code_block;
}
```

When executing this type of for loop, the Apex runtime engine performs the following steps, in order:
>1. Execute the init_stmt compontent of the loop. Note that multiple variables can also be declared and initialized in this statement.
>2. Perform the exit_condition check. If true, the loop continues, if false, the loop exits.
>3. Execute the code_block.
>4. Execute the increment_stmt statement
>5. Return to step 2


