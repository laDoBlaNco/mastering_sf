# DAY 32 - SALESFORCE BOOTCAMP 2023 | INTRODUCTION TO APEX PROGRAMMING

---

###  Introduction

---
- Apex is an Object Oriented Programming (OOP) Language
    - Supports *classes*, *interfaces*, and *inheritance*
    - Uses Java-like syntax
- Strongly typed
    - Validates references (types) to objects at compile time
    - At the time of compilation all code will be converted to machine language and checked for errors
- Integrated with the database
    - Provides direct access (through DML and SOQL) to records and their fields
- Enables developers to add business logic to system events, including button clicks, related record updates, Visualforce pages and Lightning Components
- I can call Apex code through Web Service request and Triggers on Objects (tables)

---

### Apex is:

---
- **Integrated**
    - Provides built-in support for common Lightning Platform idioms
- **Easy to Use**
    - Uses syntax and semantics  which are easy to use and understand 🤔🤔🤔
    - Apex is based on familiar Java idioms
- **Data Focused**
    - Provides transactional access to th database (C.U.D with DML and R. with SOQL)
    - Allows me to roll back operations as well (*or in DML's case, never commit them in the first place*)
- **Rigorous**
    - Strongly typed language that uses direct references to schema objects (tables) such as Object and Field APIs (*the unique name we've been creating for each custom object*)
- **Hosted**
    - It is executed, and controlled entirely by Lightning Platform
- **Multi-tenant Aware**
    - Apex runs in multi-tenant environment like the rest of the Lightning Platform
- **Easy to Test**
    - Apex provides built-in support for unit test creation and execution
    - Test results indicate how much code is covered
- **Versioned**
    - I can save Apex code against different API versions (*currently API version 67*)

---

### Apex is included in:

---

- Performance Edition
- Unlimited Edition
- Developer Edition
- Enterprise Edition
- Database.com

---

### I use Apex when I want to: 

(Note the theme here: ***complexity*** and ***customization***. So in general, anywhere we ***can't*** use declaritive point-and-click solutions)

---

- Create Web and Email services
- Perform complex validation over more than one object
- Create complex business logic that cannot be implemented in declaritive Flows
- Create custom logic that occurs over the entire transaction
- Attach custom logic to another operator, such as Create/Update/Save a record, so that it occurs whenever the operator is executed (*Triggers?* 🤔), regardless if it originates in the UI, VF pages, or from API

---

### Apex Supports:

---
- Classes, Interfaces, Collections (List, Set, Map)
- Objects, Array notation, Expressions, Variables & Constants
- Conditional Statements (if-else), Control Statements (for, while loops)
- Cloud dev as it is stored, compiled and executed in the cloud
- Triggers to call methods
- Database statements to query and search data
- Transactions and rollbacks
- The global access modifier which is more permissive than public, and allows access across namespaces and applications
- Versioning of custom code

---

### Development Tools:

---
- Developer Console
- Web Console
- Agentforce Vibes IDE
- salesforce extension for VS Code

---

### Apex is Case-Insensitive

---

- Similar to HTML or VB.Net
- I need to follow the ***case-sensitive*** style as Lightning Web Components are in JavaScript and JS is case-sensitive. So for consistency.

---

### Object Oriented Programming

---
- Apex is an Object Oriented Programming language
- An Object is a real world entity or problem
    - Technically in salesforce this is an ***sObject*** as an ***object*** in salesforce speak is the actual pseudo-table in the database we've  been dealing with when working with standard and custom objects
- To represent objects logically we implement classes (blueprints)
- Class forms the basis of Object Oriented Programming
- A class is a collection of variables and methods that are used in the instantiation of the object
- Variables are attributes/properties of an object, whereas methods are its behaviors
- Once a class is created when we can use it as a datatype to create an instance. So ***Types are just classes***.

A High-Level Example:
|Rectangle</ul>|Class Name --> Rectangle|
|---|---|
|Attributes/Properties<br>&emsp;• length<br>&emsp;• width<br>Behaviors<br>&emsp;• Area<br>&emsp;• Perimeter|Variables<br>&emsp;• length<br>&emsp;• width<br>Methods<br>&emsp;• area()<br>&emsp;• perimeter()|


A Programming Example:
```apex
public class Rectangle{
    // variables
    Integer length, width;

    // methods
    public void area(param1, param2, ... paramn){
        // we put our business logic here
    }

    public void perimeter(){
        // more business logic here
    }
}

```

---

### Class as Data Type to Create Instance

---
- Syntax
    - `ClassName instanceName = new ClassName();`
- Example
    - `Rectangle rec = new Rectangle();`

---

### Constructors

---

- It always has the same name as the class/type
- It is called and executed automatically when a class is instantiated
- Types:
    - Default Constructor
        - typically the one we use if the initial values never change
    - Parameterized Constructor
        - the one we use if we want to enter the values for each instance

---

### Static vs Non-Static Methods

---

- We use `static` methods frequently in salesforce
- They are called differently
    - `ClassName.methodName();`
    - Static methods are not instantiated but called directly off of the class
    - ***for Triggers and Test classes we only use this static method approach***
    - (*according to Sanjay*) 90% of the time I'll be using ***static*** methods

---

### Summary

---

- Familiarity with Developer Console (*I did learn some new stuff here*)
- Familiarity with the Anonymous Window
- Creation of Apex classes in Developer console
- Creation of Apex classes with Constructors
- Creation of Apex classes with multiple methods
- Static vs non-static methods
- Examples:
    - Calculating simple interest
    - Calculating Area & Perimeter of a Rectangle



