<!-- currencyManagment.md -->
# CSCPAC | 2: CONFIGURATION AND SETUP | CURRENCY MANAGEMENT

---

Company Settings => Company Information

- Turning on multiple currencies introduces ***permanent*** changes to your org.
- This feature **cannot** be turned off
- Implications of Enabling Multiple Currencies:
    - After I turn on multiple currencies for my sf org, I can't turn it off.
    - If I turn on multiple currencies, field-to-field filters in reports don't support currency fields, like amount
    - Upon enaablement, existing records are stamped with a default currency code that I provide in the enablement request. For example, if my org contains records that use USD and EUR, I'll need to switch them all to the same default currency before enablement. Support for this type of conversion is also available as a SF paid implementation service.
    - Standard and custom objects, such as Account, Lead, Case, Opportunities, Opportunity Products, Opportunity Product Schedules, and Campaign Opportunities, have currency fields that support multiple currencies. By default, page layouts for these objects have multi-currency-compatible fields in which I can specify the currency for the record. Typically, those fields are available only when creating a record or editing an existing record. The selected currency is used for the primary amount field.
    - when I create an test a package, the temporary build org has only USD  configured. Using a currency code other than USD for testing results in an error.
    - After enablement, the number of decimal places defined in a custom currency field is ignored. Instead set decimal places per currency through `Manage Currencies` in Setup
    - After enablement, the primary currency shows as usual, and optionally, a secondary currency amount appears in ()s. The primary currency is typically the default corporate currency, unless it's overridden at the record level. The amount shown in ()s is the user's personal default currency. SF calculates the converted amount using a two-step sequence:
        - The record currency converts to the corporate currency first,
        - then converts again to the user's personal currency

        Because conversion happens in two steps, outdated conversion rates can compound variation across both conversions. To minimize variation in converted amounts, keep your conversion rates current. To control whether the converted currency amount appears, turn on or off parenthetical currency conversion from the `Manage Currencies` page
    - In reports, the primary currency reflects either the default corporate currence or the currency selected for the record. The secondary currency reflects the personal default currency of the user running the report, or the currency specified in the report criteria.
    - Users can specify a personal default currency on their personal information page. If parenthetical currency conversion is turned on, the personal default currency is shown as the secondary currency amount (converted amount). Changing the personal default currency then updates the converted amount in real time.
    - After a currency is added to an org's list of supported currencies, ti can't be deleted from the admin's list of currencies, even when it's deactivated. The presence of inactive currencies in the admin's list is a cosmetic issue that doesn't affect end users. A deactivated currency isn't visible to end users, but remains visible to admins. SF recommends that I keep this issue in mind during testing and use only those currencies that I eventually plan on using.
    - After enablement, all currency fields show the ISO code of the currency before the amount. For example, $100 displays as USD 100.
    **NOTE**: If I have only one currency in my multi-currency org, I can set a preference to see the currency symbols instead of the ISO codes. To show currency symbols, search Setup for `User Interface`, and then select `Show currency symbols instead of ISO codes` in the Currency Display settings section of the UI settings page. If I later enable more currencies in my organization, ISO codes display, and this preference is no longer available. This preference applies ony in the standard SF UI. When I activate multiple currencies, I specify the number of decimal places to show, however, when Show currency symbols instead of ISO codes is enabled, the specified decimal places for multiple currencies is ignored.
    - By default, all converted amounts rely on the current conversion rates defined for my org. Conversion rates **must be set and updated manually**.  Changing the exchange rate will automatically update converted amounts on all records, including on closed opportunities. In **advanced currency management** I can't opt to use dated exchange rates.
    **NOTE**: Dated exchange rates aren't sued in forecasting, currency fields in other objects, or currency fields in other types of reports.   
