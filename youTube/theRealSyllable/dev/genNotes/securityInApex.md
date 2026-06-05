## Security in Apex: User Mode vs System Mode

---

### How Apex Runs by Default

Salesforce Apex runs in **system mode** by default for API 66 and below. ***For API 67 and greater the default mode is security, but mroe on that later***. This means for earlier versions, the executing code can see and modify **every** record in the org, regardless of the current user's sharing rules, FLS, or object-level CRUD permissions. The platform behaves as if an all-powerful system admin is making **every** query and DML call. But that's not always the case.

The design exists for a legit reason. Integration code, data migration utilities, and admin-triggered batch jobs often need to operate on records that the running user may not normally see. If a nightly batch job that consolidates billing data had to respect every sales rep's sharing model, ti would silently miss half the records it needs to do it's job.

The danger appears the moment you call that same code from a Lightning component, a public-facing Experience Cloud page, or any context where an *unpriveleged* user triggers it. Without explicit security enforement, a community user or low-permission internal user will be able to read and write data far beyond their access level simply by invoking my Apex. 🤯🤯🤯

### With Sharing, Without Sharing, and Inherited Sharing

Apex controls ***record-level*** visibility through these class-level keywords. These keywords determine which records are visible to SOQL queries inside the class -- they are about *row access*, not field access.

#### with sharing

Declaring a class `with sharing` tells Apex to enforce the current user's sharing rules when executing SOQL. If a user can't see an Account record normally because no sharing rule grants them access, a query inside this class will not return that record either.

```apex
public with sharing class AccountService{
    public List<Account> getMyAccounts(){
        // only returns accounts the running user has access to via 
        //ownership, sharing rules, manual shares, or role hierarchy.
        return [SELECT Id, Name, Industry FROM Account];
    }
}
```

I should use `with sharing` for every class that handles user-facing data operations. This is the **safe default** for any code triggered by a user interaction
- ***and now its the safe default for any class where no keyword is used as well, as of API 67***

#### without sharing

Declaring a class `without sharing` explicitly *bypasses* the current user's sharing model. Every record the class queries or modifies is accessible regardless of who triggered the code.

```apex
public without sharing class BillingAggregator{
    public static Map<Id,Decimal> computeOrgWideTotals(){
        Map<Id,Decimal> totals = new Map<Id,Decimal>();
        for(Opportunity opp:[SELECT AccountId,Amount FROM Opportunity]){ // inline SOQL
            Decimal current = totals.containsKey(opp.AccountId) ? totals.get(opp.AccountId) : 0;
            totals.put(opp.AccountId, current + (opp.Amount ?? 0));
        }
        return totals;
    }
}
```

I need to use `without sharing` only for ***back-end utility*** code: scheduled jobs, batch classes, integration adapters, and admin tools that must see all data regardless of user context. **I should never explose a `without sharing` method directly to a user-facing controller**

#### inherited sharing

Declaring a class `inherited sharing` tells it to adopt the sharing mode of whatever called it. If a `with sharing` controller calls an `inherited sharing` service, the service enforces sharing. If a `without sharing` batch class calls the same service, the service does not enforce the sharing.

```apex
public inherited sharing class RecordFetcher{
    public static List<Case> getOpenCase(){
        return [SELECT Id, Subject, Status FROM Case WHERE Status != 'Closed'];
    }
}
```

This is the right keyword for shared utility or service classes that don't own the security decision -- the decision belongs to the caller. A controller that knows it is serving a user applies `with sharing`; it calls `RecordFetcher`, which inherits that restriction automaticall.

#### The hidden danger: no keyword declared (for versions <= API 66)

The class with no sharing keyword doesn't inherit anything -- **it runs in system mode** for record visibilty. This surprises many devs since it looks like `inherited sharing` but it behaves very differently.

```apex
// DANGEROUS for API 66 and below: no sharing keyword means SYSTEM MODE for record visibility
public class LeadService{
    public static List<Lead> getAllLeads(){
        // Returns all leads in the org, ignoring the user's sharing model.
        // If this is called from a Lightning component, any user can see
        // every lead regardless of their territory or role.
        return [SELECT Id, Name, Email, Company FROM Lead];
    }
}
```

The fix is to ***always declare an explicit keyword***. For new user-facing classes: `with sharing`. For classes whose behavior should adapt to their context: `inherited sharing`.

### WITH USER_MODE and WITH SYSTEM_MODE in SOQL

This was introduced in API 50.0 (`WITH USER_MODE` and `WITH SYSTEM_MODE`). These clauses apply security enforcement at the individual query level, independent of the class's sharing declaration. This is a more granular and explicit mechanism than class-level keywords.

`WITH USER_MODE` enforces 3 things simultaneously for that query:
- The current user's sharing rules (row-level visibility)
- Field-level security (FLS) - fields the user can't read are excluded from results
- Object-level CRUD - if the user can't query the object, the query throws an exception.

```apex
public with sharing class ContactController{
    public static List<Contact> getUserContacts(){
        // enforces sharing, FLS, and CRUD at query level
        // fields the running user can't see are stripped out automatically
        return [SELECT Id,FirstName,LastName,Email,Phone
                FROM Contact
                WITH USER_MODE
        ];
    }
}
```

`WITH SYSTEM_MODE` does the opposite -- it explicitly bypasses all 3 checks for that query alone, even inside a `with sharing` class. I should use it when one specific query in an otherwise secure class needs to read data the can't directly see, such as a configuration record or a cross-object lookup.

```apex
public with sharing class DashboardController{
    public static List<Account> getUserAccounts(){
        // normal user-mode query: respects sharing + FLS and CRUD
        return [SELECT Id, Name FROM Account WITH USER_MODE];
    }

    public static List<OrgConfig__c> getConfiguration(){
        // Configuration records are admin-managed; bypass sharing this query only
        return [SELECT Id, FeatureFlag__c, MaxBatchSize__c
                FROM OrgConfig_c
                WITH SYSTEM_MODE];
    }
}
```

#### Query-level `USER_MODE` vs class-level `with sharing`

These two mechanisms are complementary, **not interchangeable**. `with sharing` on a class enforces sharing rules for ***all*** SOQL in taht class but ***does not enforce FLS or CRUD***. `WITH USER_MODE` on a query enforces all 3 -- sharing, FLS, and CRUD -- for that **specific** query only.

For complete security, I need both: the class declared `with sharing` and queries using `WITH USER_MODE`, or I can rely on `WITH USER_MODE` alone (which is actually sufficient at the query level (*for each query*))

### Database.query() with AccessLevel

Dynamic SOQL through `Database.query()` (*very similar to some PHP stuff I learned with PDO queries; another very good article detailing the use of `Database.query` and SOQL injection: [Dynamic SOQL in Apex: Runtime Queries Done Right](https://realsyllabus.com/articles/dynamic-soql-apex-salesforce.html)) supports the same access levels (`USER_MODE` and `SYSTEM_MODE`) via the `AccessLevel` enum, available from API v57.0 with `Database.queryWithBinds()` or passed as a 3rd argument to `Database.query()`.

```apex
public with sharing class DynamicSearchService{
    public static List<SObject> searchRecords(String objectApiName, String searchTerm){
        String soql = 'SELECT Id, Name, FROM ' + String.escapeSingleQuotes(objectApiName) + ' WHERE Name LIKE :searchTerm';
        Map<String,Object> binds = new Map<String,Object>{'searchTerm' => '%' + searchTerm + '%'};

        // USER_MODE enforces sharing, FLS, and CRUD for this dynamic query
        return Database.queryWithBinds(soql,binds,AccessLevel.USER_MODE);
    }

    public static List<SObject> adminSearchRecords(String objectApiName, String searchTerm){
        String soql = 'SELECT Id, Name FROM ' + String.escapeSingleQuotes(objectApiName) + ' WHERE Name LIKE :searchTerm';
        Map<String,Object> binds = new Map<String,Object>{'searchTerm' => '%' + searchTerm + '%'};

        // SYSTEM_MODE for admin-facing tools that must see all records
        return Database.queryWithBinds(soql,binds,AccessLevel.SYSTEM_MODE);
    }
}
```
<div align='center'>More details around Database.query() in the link above and here: </div>

[Dynamic SOQL in Apex: Runtime Queries Done Right](https://realsyllabus.com/articles/dynamic-soql-apex-salesforce.html)

---

#### As a small aside, some details around using sObject and Object as generic return types:

In Salesforce Apex, `Object` is the root data type for everything in the system, whereas `sObject` is a **specific** generic type restricted ***strictly to Salesforce database records***.

Choosing between them as a generic return type determines *how flexible my code will be versus how much built-in database functionality I'll retain*

##### Key Differences At A Glance

|Feature|Object Return Type|sObject Return Type|
|---|---|---|
|**Scope**|Represents any Apex type (Primitives, Collections, Custom Classes, sObjects)|Represents only database tables (Account, Contact, Custom Objects)|
|**DML Support**|Cannot be used directly in DML operations (`insert` `update`)|Fully supports native DML operations and SOQL queries|
|**Field Access**|Cannot dynamically access database fields out-of-the-box|**Can** dynamically read/write fields using .get() and .put()|
|**Typical Use**|JSON deserialization, generic utility frameworks, HTTP callouts|Generic data layers, trigger frameworks, custom query utilities|

##### When to Return `sObject`

I should use `sObject` when my method is strictly dealing with Salesforce records but needs to be dynamic enough to handle different tables (e.g., handling both `Account` and `Custom_Object__c`)

Advantages:
- I can execute database operations like `insert`, `update`, or `upsert` directly only the returned value. 
- I can bypass hardcoded fields and use generic methods like `record.get('Name')` or `record.put('Status__c', 'Active')`.
- I can extract metadata tokens using `.getSObjectType()`.

Example:
```apex
public sObject getDatabaseRecord(Id recordId){
    // dynamically queries any record type based on ID prefix
    String objectName = recordId.getSObjectType().getDescribe().getName();
    String query = 'SELECT Id, Name FROM ' + objectName + ' WHERE Id = :recordId LIMIT 1';
    return Database.query(query); // returns the generic sObject
    // NOTE: remember .query just uses just the soql string and .queryWithBinds gets the 3 args (soql string,map,access enum)
}
```

##### When to Return `Object`

I need to use `Object` when my method needs to be completely agnostic about data structures, or when it returns non-database types like data primitives, API wrappers, or complex custom classes.

Advantages:
- Ultimate abstraction; it can hold a `String`, an `Integer`, a `Map<String,Object>`, or a custom class instance.
- Standard choice for JSON parsing utilities and integration engines

Disadvantages:
- To do anything useful with the returned data, the receiving code must explicitly cast it back to its specific type
- It cannot be inserted or updated into the database without casting it first

Example:
```apex
public Object parseApiResponse(String jsonString){
    // could return a Map, a primitive, or a Custom Class instance
    Map<String, Object> untypedMap = (Map<String,Object>) JSON.deserializeUntyped(jsonString);
    return untypedMap.get('data');
}
```

#### Summary Rule of Thumb

- If the method returns **Salesforce database records** that I plan to modify or save, then I should use **sObject**
- If the method returns **calculated values, API payloads, or varied system types**, then I should use **Object**

If I'm designing a framework, I need to know **what kind of data my methods will process** (e.g., triggers, integrations, or calculations), so that I can then determine the exact method structure.

---

### Manual FLS Enforcement

There will be times when I **cannot use** query-level `WITH USER_MODE` -- for instance, in code targeting API versions below 50.0 or when working with DML operations -- I'll need to check FLS manually using the `Schema.DescribeFieldResult` methods.

#### Checking field accessibility

```apex
public with sharing class AccountFieldChecker{
    public static void verifyReadAccess(){
        // check individual fields before reading them and throw early exception
        if(!Schema.SObjectType.Account.fields.Phone.isAccessible()){
            throw new System.NoAccessException();
        }
        if(!Schema.SObjectType.Account.fields.AnnualRevenue.isAcessible()){
            throw new System.NoAccessException();
        }

        // If we get his far, then its safe to query now
        List<Account> accounts = [SELECT Id, Phone, AnnualRevenue FROM Account];
    }

    public static void verifyWriteAccess(){
        // check before update DML
        if(!Schema.SObjectType.Account.fields.Phone.isUpdateable()){
            throw new System.NoAccessException();
        }
        // check before insert DML
        if(!Schema.SObjectType.Account.fields.Phone.isCreatable()){
            throw new System.NoAccessException();
        }
    }
}
```

#### `stripInaccessible()` -- the practical alternative

Manually field-by-field checking is verbose and easy to error. `Security.stripInaccessible()` automates all of this: it takes a list of records and removes any field values the current user is not allowed to access, based on FLS. The method returns an `SObjectAccessDecision` object containing the cleaned records.

```apex
public with sharing class SafeAccountReader{
    public static List<Account> getAccountStripped(){
        // we first query all fields of interest (some may not be accessible to all users)
        List<Account> rawAccounts = [
            SELECT Id, Name, Phone, AnnualRevenue, Rating, Fax
            FROM Account
            LIMIT 200
        ];

        // then we can strip any fields the current user can't see
        // AccessType.READABLE checks the read FLS
        SObjectAccessDecision decision = Security.stripInaccessible(
            AccessType.READABLE,
            rawAccounts
        );

        // then we can getRemovedFields() to return a map of object name->set of stripped field names,
        // useful for logging what was removed, if needed
        Map<String, Set<String>> removed = decision.getRemovedFields();
        if(!removed.isEmpty()){
            System.debug('Stripped inaccessible fields: ' + removed);
        }

        return decision.getRecords();
    }
}
```

There are four `AccessType` values corresponding to DML operations
- `AccessType.READABLE` -- strips fields the user cannot read
- `AccessType.CREATEABLE` -- strips fields the user cannot set on insert
- `AccessType.UPDATEABLE` -- strips fields the user cannot modify
- `AccessType.UPSERTABLE` -- strips fields not createable or updateable

### Manual CRUD Enforcement

So FLS governs individual fields, CRUD governs the entire object. A user might have read access to every field on Opportunity but no permission to create new Opportunity records. I must check object-level permissions separately.

```apex
public with sharing class CrudGuard{
    public static void assertCreateable(SObjectType objType){
        if(!objType.getDescribe().isCreatable()){
            throw new System.NoAccessException();
        }
    }

    public static void assertUpdateable(SObjectType objType){
        if(!objType.getDescribe().isUpdateable()){
            throw new System.NoAccessException();
        }
    }

    public static void assertDeletable(SObjectType objType){
        if(!objType.getDescribe().isDeleteable()){
            throw new System.NoAccessException();
        }
    }

    public staic void assertQueryable(SObjectType objType){
        if(!objType.getDescribe().isQueryable()){
            throw new System.NoAccessException();
        }
    }
}
```

I call these guards immediately before DML in any user-facing operation:

```apex
public with sharing class OpportunityWriter{
    public static void createOpportunity(String name, Id accountId, Date closedDate, String stage){
        // CRUD check before upsert
        if(!Schema.SObjectType.Opportunity.isCreateable()){
            throw new AuraHandledException('You do not have permission to create Opportunities.');
        }
        // FLS checks  for the fields being set
        if(!Schema.SObjectType.Opportunity.fields.StageName.isCreateable()){
            throw new AuraHandledException('You do not have permission to set the Stage field.');
        }

        Opportunity opp = new Opportunity(
            Name = name,
            AccountId = accountId,
            CloseDate = closeDate,
            stageName = stage
        );
        insert opp;  // if it got this far, all is good.
    }

    public static void deleteOpportunity(Id oppId){
        // CRUD check before delete
        if(!Schema.SObjectType.Opportunity.isDeleteable){
            throw new AuraHandledException('You do not have permission to delete Opportunities.');
        }
        delete new Opportunity(Id = oppId); // 🤔 interesting method of deletion. Creating a new opp with specific Id rather than retrieving it with SOQL to delete
    }
}
```

## Real-World Pattern: Secure User-Facing Utility Class

The following class combines everything I've noted above, every security layer discussed into a single, production-ready pattern. It handles a realist scenario: a Lightning component calls a method to fetch an update Account records on behalf of the current user.

```apex

public with sharing class AccountManager{

    /*
    Returns accounts visible to the current user, with inaccessible  fields stripped.
    Throws if the user can't query Accounts at all
    */

    public static List<Account> getAccessibleAccounts(String industryFilter){
        // 1. CRUD: can this user even query Accounts?
        if(!Schema.SObjectType.Account.isQueryable()){
            throw new AuraHandledException('You do not have permission to view Accounts.');
        }

        // 2. Query with USER_MODE: enforces sharing + FLS + CRUD at the query level.
        //    The CRUD check above is redundant here when using USER_MODE but is kept here
        //    as a fast-fail with a user-friendly message before the query fires.
        List<Account> accounts;
        if(String.isNotBlank(industryFilter)){
            accounts = Database.queryWithBinds(
                'SELECT id, Name, Phone, AnnualRevenue, Rating, Industry FROM Account WHERE Industry = :ind',
                new Map<String,Object>{'ind'=>industryFilter},
                AccessLevel.USER_MODE
            );
        }else{
            accounts = [SELECT Id, Name, Phone, AnnualRevenue, Rating, Industry
                        FROM Account
                        WITH USER_MODE];
        }

        // 3. stripInaccessible as a defense-in-depth layer for any fields
        //    not already excluded by USER_MODE (edge cases with legacy permissions)
        SObjectAccessDecision decision = Security.stripInaccessible(
            AccessType.READABLE,
            accounts
        );
        return decision.getRecords;
    }

    /**
        Updates the Phone and Rating fields of an Account
        Enforcing CRUD and FLS manually before any DML
    **/
    public static void updateAccountContact(Id accountId, String phone, String rating){
        // 1. CRUD check: Can this user update Accounts?
        if(!Schema.SObjectType.Account.isUpdateable()){
            throw new AuraHandledException('You do not have permissions to update Accounts.');
        }

        // 2. FLS check: Can this user write to these specific fields?
        List<String> blockedFields = new List<String>();
        if(!Schema.SObjectType.Account.fields.Phone.isUpdateable()){
            blockedFields.add('Phone');
        }
        if(!Schema.SObjectType.Account.fields.Rating.isUpdateable()){
            blockedFields.add('Rating');
        }
        if(!blockedFields.isEmpty()){
            throw new AuraHandledException('You don't have permission to edit: ' + String.join(blockedFields,', '));
        }

        // 3. Verify the record is accessible to this user before updating it
        //    USER_MODE on the read ensures we don't blindly update a record the
        //    user couldn't see -- which would be a privilege-escalation vector 
        List<Account> existing = [
            SELECT id FROM Account WHERE Id = :accountId WITH USER_MODE LIMIT 1
        ];
        if(existing.isEmpty()){
            throw new AuraHandledException('Account not found or you do not have access.');
        }

        // 4. Build a minimal update record - never query-then-update full object
        //    to avoid accidntally overwriting fields with stale values
        Account toUpdate = new Account(
            Id = accountId,
            Phone = phone,
            Rating = rating
        );

        // 5.  stripInaccessible on UPDATEABLE before the DML as a final safety net
        SObjectAccessDecision decision = Security.stripInaccessible(
            AccessType.UPDATEABLE,
            new List<Account>{toUpdate}
        );

        update decision.getRecords();
    }

    /**
        Inserts a new Account. Enforcing CRUD and FLS on every field being set.
    **/
    public static Account createAccount(String name, String industry, String phone){
        // 1. CRUD
        if(!Schema.SObjectType.Account.isCreateable()){
            throw new AuraHandledException('You do not have permission to create Accounts.');
        }

        // 2. FLS check on fields being written
        Map<String,Schema.SObjectField> fieldMap = Schema.SObjectType.Account.fields.getMap();
        List<String> fieldsToWrite = new List<String>{'Name','Industry','Phone'};
        for(String fieldName:fieldsToWrite){
            Schema.DescribeFieldResult dfr = fieldMap.get(fieldName.toLowerCase()).getDescribe();
            if(!dfr.isCreateable()){
                throw new AuraHandledException('You do not have permission to set: ' + fieldName);
            }
        }

        Account newAccount = new Account(
            Name = name,
            Industry = industry,
            Phone = phone
        );

        // 3. stripInaccessible on CREATEABLE before insert DML
        SObjectAccessDecision decision = Security.stripIaccessible(
            AccessType.CREATEABLE,
            new List<Account>{newAccount}
        );

        List<SObject> cleanRecords = decision.getRecords();
        insert cleanRecords;
        return (Account) cleanRecords[0];
    }
}

```

### Decision Guide: Which Security Mechanism to Use

- New user-facing class, all queries, all DML: declare `with sharing` on the class and add `WITH USER_MODE` to every SOQL statement. This is the most complete and concise approach
- Shared service or utility called by both user-facing and system code: declare `inherited sharing` on the class so the caller decides
- Back-end batch, scheduled job, or integration: declare `without sharing` explicitly on the class so the intent is documented
- In API v66 and below, no keyword: treat it like a bug. Audit  and add a keyword.
- DML without `USER_MODE` available (API verions < 50.0): use `Schema.SObjectType` CRUD checks before DML and `Security.stripInaccessible()` before insert/update specifically.
- Need to verify a field before reading or writing without full strip: use `.isAccessible()`, `.isCreateable()`, `.isUpdateable()` on the `DescribeFieldResult`.

Apex security is not one mechanism but a layered system:
    - Class-level sharing keywords govern ***row visibility***
    - `WITH USER_MODE` adds FLS and CRUD enforcement at the ***query level***
    - Manual describe checks and `stripAccessible()` cover DML

Use all 3 layers together in user-facing code:
    - a class declared `with sharing`, 
    - queries using `WITH USER_MODE` or `Database.queryWithBinds(..., AccessLevl.USER_MODE)`, and `Security.stripInaccessible()` as the final gate for DML. 

The most dangerous pattern in Salesforce development (***before API v67***) is a class with no sharing keyword called from a user-facing controller -- it silently exposes every record in the org. I must always declare sharing intent explicitly.

