---
title: Google Play Store In App Billing Issue
date: 2023-11-11 20:22:52
categories:
- [Essay]
tags: Google Play Store, In App Billing, Found null or empty SkuDetails. Check to see if the SKUs you requested are correctly published in the Google Play Console, onSkuDetailsResponse 3 Billing Unavailable
---

# Found null or empty SkuDetails.

> TrivialDrive:BillingDataSource: onSkuDetailsResponse: Found null or empty SkuDetails. Check to see if the SKUs you requested are correctly published in the Google Play Console.

Root cause: Some of following prerequisites are not satisfied.
1. Upload the application to Closing Testing, no need to publish at this stage. (Internal Testing may be sufficient, but not verified).
2. Create SKUs, note that the ID should match the ID of the application request.
3. Add the tester's mailbox, which is the Google account login to the device to test the app SKU.
4. Switch License Respond to LICENSED (Path: play console homepage -> Setup -> License testing).

Notes:
* The Application ID for uploading Closing Testing needs to be the same as the Application ID for local debugging.
* Copy the link in the "Testers" tab, send it to testers to invite them join the test.

# 3 Billing Unavailable

> TrivialDrive:BillingDataSource: onSkuDetailsResponse: 3 Billing Unavailable

Your "Premium" tab should show empty rather the paid applications list in your play store if you encounter this issue.
<img src="https://s3.uuu.ovh/imgs/2023/11/11/22c2efeaf37dabf6.jpeg" style="zoom:25%;" />
Root cause: The current Play Store account of the test device does not support the Billing feature, for example, my reason is that the account is registered as China region.
Solution: Change it to US, and the address can use a US physical address, or fill in a random one.

[Change your Google Play country](https://support.google.com/googleplay/answer/7431675?sjid=11201606240048920500-EU)

![](https://storage.googleapis.com/support-kms-prod/nZJ8elMyFeFOLGEpXwigaB6zmHvq46xYOfd1)
1. On your Android device, open the Google Play Store app Google Play.
2. At the top right, tap the profile icon.
3. Tap Settings and then General and then Account and device preferences and then Country and profiles.
4. Tap the country where you want to add an account.
5. Follow the on-screen instructions to add a payment method for that country.
Tips: Your profile can take up to 48 hours to update.

# Other checkpoints and actions to try
1. Enable VPN and use American server.
2. Register the test account in the settings and play store, remove other accounts.
3. Clear the storage of "Google Play services" and "Google Play Store".