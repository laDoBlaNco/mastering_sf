<!-- businessHours.md -->
# CSCPAC | 2: CONFIGURATION AND SETUP | BUSINESS HOURS

---

Company Settings => Business Hours (=> Holidays)

- Used to specify the hours when your support team is available to serve customers.
- Helps make your department's processes, such as escalations and milestones, more accurate
- Setting business hours lets me apply specifi time zones and locations to
    - Milestones in entitlement processes
    - Entitlement processes
    - Cases
    - Case escalation rules

I can make the Business Hours field available on the Case Layout so that my service reps can set the times of support the team is available to work on the case. By default, business hours are set to 24 hours, seven days a week in the default time zone specified by the Org's profile (normally Pacific since SF HQ is in Seattle)

If I have the *Customize Application* permission I can add business hours to escalation rules so that when the details of a case match the criteria of an escalation rule, the case is automatically updated and escalated with the times and location on the rule. For example, a case updated with LA business hours escalates only when a support team in LA is available.

To set business hours:
>1. From Setup, enter `Business Hours` in the Quick Find box, then **Business Hours**.
>2. Click **New Business Hours**
>3. Type a name for the business hours. Its recommended using a name that will remind users of a location or time zone when they view business hours on a case, entitlement, or milestone. For example, if my business hours are for support in San Francisco, I could use the name `San Francisco Business Hours`.
>4. Then I click `Active` to allow users to associate the business hours with cases, escalation rule, milestones, and entitlement processes.
>5. Optionally, I can click `Use these business hours as the default` to set the business hours as the default business hours on all new cases. Default business hours on cases can be updated with business hours on escalation rules if the cases match escalation rule criteria and the rule is set to override business hours.
>6. I need to choose a time zone to associate with the business hours in the `Time Zone` dropdown list.
>7. I then set my business hours for each day of the week:
>       - If my support team is avaiable during the entire day of the week, I select `24 hours` checkbox
>       - I choose the start and end times for the business hours. If the tiem I want isn't avaiable, I click the field and type it in.
>       - Then Leave the business hours start and end times blank and the 24 hours checkox deselected to indicate that the  support team is not available *at all* that day.
>8. Click **Save**.

After I've set business hours, I can associate them with:
- Escalation rules, so that when the details of a case match the criteria of an escalation rule, the case is updated and escalated with the business hours on the rule.
- Holidays, so that business hours and any escalation rules associated with business hours are suspended during the dates and times specified in holidays.
- Milestones, in entitlement processes so that business hours can change with the severity of a case.
- Entitlement processes, so that I can use the same entitlement process for cases with different business hours.

NOTE: ***All users***, even those without the *"View Setup and Configuration"* user permission, can view business ours via API.

To make my support processes more accurate, define when my support team is available to help customers. There are a few guidelines to keep in mind as I set business hours

- After you set business hours, add the `Business Hours` lookup field to the case layouts and set field-level security on the `Business Hours` field. This let's users view and update the business hours on a case
- Business hours on a case are automatically set to our organization's default business hours, unless the case matches the criteria on an escalation rule associated with different business hours.
- Salesforce automatically calcualtes daylight savings times for the time zones available for business hours, so you don't have to configure rules to account for time zones
- Business hours on a case include hours, minutse, and seconds. However, if business hours are less than 24 hours, the system ignores the seconds for the last minute before hours end.
    - For example, suppose if it 4:30PM now and business hours end at 5:00PM. If I have a milestone with a 30-minute target, it's more common to say that the the target is 5:00PM, not 4:59PM. To accommodate this, the system stops counting seconds after 5:00. If seconds were counted from 5:00:00 - 5:00:59, the 30-minute target would occur after the 5:00PM target  cut-off and would roll over to the next day.
- Escalation rules only run during the business hours they're associated with
- I **can** update cases associated with business hours that are no longer active.
- I **can't** include the `Business Hours` field in list views or reports.