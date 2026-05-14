# DATA MODEL

---
## Objects

- Introduction:
  - An object is very similar to a database table in sf platform.
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





