# DAY 02 - SF LEARNING BOOTCAMP 2023

## SF DATA MODEL | OBJECT, FIELD, RECORD, TAB & APPS

---
## Objects

- Introduction:
  - An object maps directly to th emental model of database table in sf platform.
  - The platform comes with a number of standard objects like Account,Contact,Case,Lead,Opportunity, etc.
  - The standard objects support default apps like sf sales and sf service clouds.
  - We can create Custom Objects in sf as per our requirements of the project.

## Standard vs Custom Objects

| |Standard|Custom|
|---|---|---|
|Label|Account|Student|
|Plural Label|Accounts|Students| <!-- Normally plural label is for tabs -->
|API Name|Account|Student__c|

**NOTE:** Plural Label is used for Tabs and API name is used in features like flow, apex, external tools so they can reference particular objects.

## Tabs

- Through tabs we can navigate around an app (basically a UI/Navigation thing). Without the tab we won't see or object on the UI.
- Every tab serves as the start point for viewing, editing and entering information for a specific object (table).
- When we click a tab, the corresponding home page of that object  appears.
  - For example, if we click the Accounts tab, the Accounts object home page appears. It gives us access to all of the account records. We can view details of a particular record by clicking on it.

## Fields

- A field maps directly to the mental model of a database column.
- There are various data types avaiable in sf to create fields (columns).
- By entering values in fields (just like in a database column), we create a record (row) in sf.

### Data Types for Fields

| | | |
|---|---|---|
|Auto Number|Date|Picklist (Multi-Select)|
|Formula|Date/Time|Text|
|Roll-up Summary|Email|Text Area|
|Lookup Relationship|GeoLocation|Text Area (Long)|
|Master-Detail Relationship|Number|Text Area (Rich)|
|External Lookup Relationship|Percent|Text (Encrypted)|
|Checkbox|Phone|Time|
|Currency|Picklist|URL|

## Records

- Records map directly to database rows (entries) in the object (table) which are uniquly identified by there ids.
  - The id can be either Auto Number or Text and is created with the object. this maps to the PK in a dbase. O sea one of the fields must be the primary key of the table or Name of the Object. In our example its Student ID
- We can create records by entering values in fields available in an object
- We can create, edit, view and delete a record in sf

## Apps

- An App is a container for all the objects, tabs and other functionality. There is no direct mapping of our mental model of a database, but we can think of the App as a database view. Since it can have multiple combos of the same tables and records in different views, just as we can have the same records, fields, and objects in the different apps
- It is similar to a programming project where we keep all our code in files.
- An sf app consists of simple a name, a logo, and an ordered set of tabs (with objects)










