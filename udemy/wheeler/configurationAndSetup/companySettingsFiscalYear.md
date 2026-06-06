# CSCPAC | 2: CONFIGURATION AND SETUP | COMPANY SETTINGS FISCAL YEAR

---

Setup => Company Settings => Fiscal Year

- To specify the fiscal year type for your organization, choose one of two options:
    - Standard Fiscal Year
        - Follows the gregorian calendar year but can start on the 1st of any month we choose
        - Your organization can change the fisal year start month
        - Specify whether the fiscal year name is set to the starting or ending year
            - Example: Starting April 2026 and ending March 2027, could  be either Fiscal Year 2026 or 2027
        - NOTE: ***Changing the fiscal year shifts periods and impacts opportuntiies and forecasts across the org. If forecast periods are set to quarterly then a shift of the year start month will erase exisiting forecast adjustments and quotas.***
    - Custom Fiscal Year
        - Follows the custom structure that you define. 
            - For example: 4 quarters with 13 weeks per quarter in a 4-4-5 pattern, or 13 periods per year.
        - You can define custom fiscal years that match the company's financial planning requirments. 
        - Salesforce provides flexible templates that you can customize to meet your needs. 
        - NOTE: ***Before enabling custom fiscal years, consider the following***:
            - After enabling custom fiscal years, your orgs fiscal years will no longer automatically be defined by salesforce. The only fiscal years available for reports, quotas, forecasts, and so on will be those defined by the org.
            - Enabling custom fixcal years is not reversible. After enabling custom fiscal years, you **cannot** revert to standard fiscal years. *However you can define your custom fiscal year to match standard fiscal years.*
            - If you have forecasting features enabled, creating the initial custom fiscal, deletes all quotes, adjustments from that year forward.

[Setting the Fiscal Year](https://help.salesforce.com/s/articleView?id=xcloud.setting_the_fiscal_year.htm&type=5)