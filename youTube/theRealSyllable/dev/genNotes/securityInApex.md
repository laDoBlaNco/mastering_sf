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

#### WITH USER_MODE and WITH SYSTEM_MODE in SOQL
