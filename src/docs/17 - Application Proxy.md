---
id: appproxy
title: Publishing on-premises apps using Application Proxy 
sidebar_label: Application proxy
slug: /appproxy
---



Application Proxy enables users to access on-premises web applications from the internet without requiring a VPN into the corporate network. It works like a traditional reverse proxy solution, but unlike a reverse proxy there is no inbound ports that needs to be open and exposed to the internet. Access works via the App Proxy cloud service, and the Application Proxy connector which runs on an on-premises server. Azure AD, the Application Proxy service, and the Application Proxy connector work together to securely pass the user sign-on token and traffic from Azure AD to the web application. The added benefit here is that conditional access can be used to protect the on-premises application, such as requiring MFA, or a hybrid/compliant device.

![](img/appproxy.001.png)

## Lab success exit criteria
An App Proxy Demo Installer utility will be used to deploy two websites on your Domain Controller. One website is configured for Kerberos, and the other is using Forms Based Authentication. You will then install the Application Proxy Connector on the Domain Controller and publish the two websites using the Application Proxy service so that users can access these same sites externally and where you can compare the different sign-on experience between the two.
##
## Step 1. Configure the on-premises websites
1. From <https://portal.azure.com> log into your **DC VM** using Azure Bastion.
1. From within the VM, download the AppProxyDemo.zip file from <https://1drv.ms/u/s!AjWydXh6TnNyi5onkQ4q7i5n41dSFg?e=i2dTUR> 
1. Once downloaded, extract the zip file to any location of your choosing.
1. In the extracted AppProxyDemo folder, right click on the file called **“Bootstrap.ps1”** and choose **Properties**.
1. On the **General** tab check the **Unblock** box at the bottom and then click **Apply** and **OK**.

![](img/appproxy.002.png)

1. Right click on the **Bootstrap.ps1** file again and choose the **Run with PowerShell** option to start the App Proxy Demo Installer.
1. A Windows PowerShell window will open, and it will first ask you to input the NetBIOS name of your VM. The default name should be **adVM**. If you chose a different name when creating the VM then enter that instead. If you are unsure of the NetBIOS name, then go to a cmd prompt and run “hostname” to confirm.

   ![Graphical user interface, text Description automatically generated](img/appproxy.003.png)

   Hit enter to begin the installation.

![](img/appproxy.004.png)

1. The installer will install IIS, and then configure the websites appropriately for you. You can confirm the installation was successful by going to Start/Run and typing **inetmgr.** This will open the Internet Information Services (IIS) Manager. In the left-hand pane, expand adVM > Sites > Default Web Site and you should see the following websites appear.

![](img/appproxy.005.png)

1. To validate the websites themselves are functioning, open Edge on the DC VM and browse to these URL’s which should load the pages as seen in the below screenshots.

<http://localhost/FormsSample> (presents a Forms Based Authentication page)

<http://localhost/WIASample> (users Kerberos which will log you straight in as admin)

![](img/appproxy.006.png)

## Step 2. Configure Kerberos Constrained Delegation
The WIASample website uses Kerberos authentication which provides SSO to the website when the user has (or can obtain) a Kerberos ticket from the DC. A user can only do this when they have line of site to the DC when on the domain. We want to provide the same SSO experience through App Proxy, but to do this we have to setup [Kerberos Constrained Delegation](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-constrained-delegation-overview) (KDC). KDC will allow the Azure AD token that is sent to the App Proxy connector to be exchanged for a Kerberos ticket by the DC, which the connector then sends to the website for consumption. The WIASample website is running under a user account called KFYyn-AppPool, this user account was created by the App Proxy Demo installer and it has a servicePrincipalName (SPN) assigned to it of http/adVM. We will configure the DC to allow delegation of Kerberos to this SPN so the App Proxy Connector can authenticate to it on the users behalf using Kerberos.

1. On the **DC VM**, go to **Start/Run** and type dsa.msc to open the **Active Directory Users & Computers** snap-in.
1. Go to the **Domain Controllers** OU and **right click** on the adVM object and select **Properties**.
1. On the **Properties** page select the **Delegation** tab.
1. On the **Delegation** tab 
   1. Select **Trust this computer for delegation to specified services only**
   1. Select **Use any authentication protocol**
   1. Click **Add**, click **Users or Computers** and type in **KFYyn-AppPool** and then click **Check Names** to resolve it and then click **OK**.
   1. In the **Add Services** page select the adVM object that appears under Available Services and then click **OK**.

      ![Graphical user interface, text, application Description automatically generated](img/appproxy.007.png)
   1. Back on the **Delegation** tab, confirm that the http adVM service is now listed, and verify the radio buttons are set correctly per the screenshot below.

![Graphical user interface, text, application, email Description automatically generated](img/appproxy.008.png)

Ensure to click **Apply** and then click **OK**.


Kerberos Constrained Delegation is now setup.



## Step 3. Deploy the Application Proxy Connector
1. On the DC VM, open Edge and sign in to <https://aad.portal.azure.com> as your global admin
1. Select *Azure Active Directory*, then browse to **Security** > **Enterprise Applications** > **Application Proxy**
1. Click on the **Download connector service** at the top of the page and click the **Accept terms & Download** button to download the AADApplicaitonProxyConnectorInstaller.exe file.

![](img/appproxy.009.png)

1. Open the file to begin the installation. In the installation wizard…
   1. Agree to the terms and conditions and click **Install**
   1. When prompted, sign-in as your Global Admin
   1. Once the installation completes successfully click **Close** to exit the wizard.

      ![](img/appproxy.010.png)
1. Return to the **Application Proxy** blade in the portal where you should now see the connector appear and the status should show as **Active**. 

![](img/appproxy.011.png)

1. Click the **Enable Application Proxy** button at the top of the page to enable the service for your tenant.

![Graphical user interface, text, application Description automatically generated](img/appproxy.012.png)

## Step 4. Publish the WIASample on-premises website
1. In the Application Proxy blade, click the **+ Configure an app** button.
1. In the **Add your own on-premises application** page complete the following details:
   Under **Basic Settings**
   1. **Name:** Intranet
   1. **Internal Url:** <http://advm/WIASample/>
   1. **External Url:** (leave as default)
   1. **Pre Authentication:** Azure Active Directory
   1. **Connector Group:** (leave as default)

Under **Additional Settings** leave everything as default

Click the **+ Add** button to publish the app.

![Graphical user interface, application Description automatically generated](img/appproxy.013.png)

1. In the left-hand menu under **Manage**, click **All Applications** where you should see the Intranet app you just added. 

![Graphical user interface, application Description automatically generated](img/appproxy.014.png)

1. Click the Intranet app, and in the next window under **Manage**, click **Users and Groups**
   1. Click the **+Add user/group button** and then add one of your on-premises synchronised users, e.g. Bobby Smith, and then click **Assign**.

1. Back in the Intranet app settings, under **Manage** click **Single sign-on** and then click the **Windows Integrated Authentication** tile.


![Graphical user interface, text Description automatically generated](img/appproxy.015.png)

1. On the next screen, enter in the following details
   1. **Internal Application SPN:** http/adVM
   1. **Delegated Login Identity:**  On-premises SAM account name

![](img/appproxy.016.png)

## Step 5. Publish the FormsSample website
1. In the Application Proxy blade, click the **+ Configure an app** button.
1. In the **Add your own on-premises application** page complete the following details:
   Under **Basic Settings**
   1. **Name:** Intranet Forms
   1. **Internal Url:** http://advm/formssample/
   1. **External Url:** (leave as default)
   1. **Pre Authentication:** Azure Active Directory
   1. **Connector Group:** (leave as default)

Under **Additional Settings** leave everything as default

Click the **+ Add** button to publish the app.

1. In the left-hand menu under **Manage**, click **All Applications** where you should see the **Intranet Forms** app you just added.
1. Click the **Intranet Forms** app, and in the next window under **Manage**, click **Users and Groups**
   1. Click the **+Add user/group button** and then add the same on-premises synchronised user that you added to the first app in the previous step, e.g. Bobby Smith, and then click **Assign**.



## Step 6. Testing App Proxy as a user
We will now test accessing both published Intranet sites through App Proxy as the test user that you assigned in the previous step. Perform these actions on your host machine and not on the DC itself.

1. Open a In-Private Edge session and browse to <https://myapps.microsoft.com>
1. Login as your on-premises test user (Bobby Smith if you are following along)
1. In the MyApps portal you should see both Intranet apps.

![Graphical user interface, application Description automatically generated](img/appproxy.017.png)

1. Click on the **Intranet** app. You will be redirected to the App Proxy Url and logged in seamlessly to the internal website using Kerberos.

![](img/appproxy.018.png)

1. Return to MyApps and now click on the **Intranet Forms** app. You will be redirected to the Forms Based logon page. To log into this website use these details.

**UseName:** test

**Password**: test

![](img/appproxy.019.png)


