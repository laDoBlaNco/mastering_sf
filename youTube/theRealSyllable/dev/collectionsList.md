<!-- collectionsList.md -->
# The Trailmaker | Salesforce Developer Series - 12 | Collections - Lists

---

### Collections - Lists

A list is an ordered collection of elements that are distinguished by their indices. List elements can be any data type -- primitive types, collections, sObjects, user-defined types, and built-in Apex types.

This table is a visual representation of a list os Strings
|Index 0|Index 1|Index 2|Index 3|Index 4|Index 5|
|---|---|---|---|---|---|
|'Red'|'Orange'|'Yellow'|'Green'|'Blue'|'Purple'|

The index position of the first element always starts at 0

Lists can contain **any** collection and can be nested within one another and become multi-dimensional. For exaample, I can have a list of lists of sets of Integers. A list can contain ***up to seven levels of nested collections*** inside of it. This is 8 levels overall

```apex
// creat an empty list
List<String> myList = new List<String>();

// create a nested list (multi-dimensional)
List<List<Set<Integer>>>myList2 = new List<List<Set<Integer>>>();

```
