Step 1. In this exercise, you will create a new Azure AD web application registration using the Azure Active Directory admin center.
https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/Overview

    Open a browser and navigate to the Azure Active Directory admin center. Login using a personal account (aka: Microsoft Account) or Work or School Account.

    Select Azure Active Directory in the left-hand navigation, then select App registrations under Manage.
Step 2. Select New registration. On the Register an application page, set the values as follows.

    Set Name to Node.js Graph Tutorial.
    Set Supported account types to Accounts in any organizational directory and personal Microsoft accounts.
    Under Redirect URI, set the first drop-down to Web and set the value to http://localhost:3000/auth/callback.
    
Step 3. Select Register. On the Node.js Graph Tutorial page, copy the value of the Application (client) ID and save it, you will need it in the next step.

Step 4. Select Authentication under Manage. Locate the Implicit grant section and enable ID tokens. Select Save.

Step 5. Select Certificates & secrets under Manage. Select the New client secret button. Enter a value in Description and select one of the options for Expires and select Add.

Step 6. Copy the client secret value before you leave this page. You will need it in the next step.
