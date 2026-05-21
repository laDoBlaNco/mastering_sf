# DAY 03 | SF LEARNING BOOTCAMP 2023

## TEXT, PICKLIST, GLOBAL VALUE SET, FIELD DEPENDENCY, SCHEMA BUILDER

---
**Text**
- Text - Allows users to enter any combination of letters and numbers.
- Text Area - Allows users to enter up to 255 characters on separate lines.
- Text Area Long - Allows users to enter up to 131,072 characters on separate lines
- Text Area Rich - Allows users to enter formated text, add images and links. Up to 131,072 characters on separate lines. 

**Picklist Fields**
- Picklist - Allows users to select a value from a list you define.
- Multi-Select Picklist - Allows users to select multiple values from a list you define.

**Global Picklist Value Set**
- Generic picklist used with many object's picklest fields
- Global picklist values sets let you share the values across objects
  - The value set is restricted so users can't add unapproved values through the API and can't be configured through any individual object

**Field Dependency**
- Dependency between two picklist fields
- Create a dependent relationship that causes the values in a picklist or multi-select  picklist to be dynamically filtered based on the value selected by the user in another field
  - The field that drives filtering is called the *controlling field*. Standard and custom checkboxes and picklists with at least one and less than 300 values can be controlling fields (**no multi-select picklists**). 
  - The field that has its values filtered is called the *dependent field*. Custom picklists and  multi-select picklists can be dependent fields (**not checkboxes or standard picklists**)

**Schema Builder**
- Graphical representation of objects and their relationships 
  - Great for presenting our data model to stakeholders
  - To be able to see our relationships graphically
    - Lookup (*one to many*)
    - Master-Detail (*one to many*)
    - Junction Objects (*many to many with MD*)
  - We can also create objects and fields directly, so we can build our data model graphically as well, if easier.
