# My Global Bank - CTF Challenge

A deliberately vulnerable Android banking application for CTF competitions. Contains **10 real-world vulnerabilities**.

## 🚀 Quick Start

To participate in this CTF, you need to download the assets from the [Latest Release](https://github.com/AbasovFarasat/MyGlobalBank-CTF/releases/latest).

### 1. Installation
Install the main banking application on your Android emulator or physical device:

adb install MyGlobalBank_v1.0.0.apk
adb install Malicious_Exploit.apk
adb install CVE_2023_20963_Exploit.apk


## 1. Firestore Misconfiguration - Admin Credential Leak
Vulnerability Type

**CWE-284: Improper Access Control
Description**

Firestore Rules are misconfigured. The users/admin document is publicly readable without any authentication.

<img width="1920" height="1080" alt="Screenshot_2026-04-24_15_07_59" src="https://github.com/user-attachments/assets/7f08c08c-855b-413b-8419-4b19eb46ea5a" />

The application features a "Sign Up" button, but upon interaction, the user is met with a "Restricted Access" screen. This indicates that public registration is disabled through the UI, forcing the attacker to find alternative methods to obtain credentials.

<img width="1920" height="1080" alt="Screenshot_2026-04-24_15_13_18" src="https://github.com/user-attachments/assets/cd406ab3-3859-433b-b00b-dad862e3351b" />

To analyze the source code and resources, we decompile the APK using Apktool.

**Command**

apktool d MyGlobalBank_CTF.apk

<img width="1920" height="1080" alt="Screenshot_2026-04-21_12_59_47" src="https://github.com/user-attachments/assets/b080bcdd-6ea5-4b45-87de-3a3d53d90390" />

By inspecting the AndroidManifest.xml file, we identify that the application is integrated with Firebase services.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_13_02_49" src="https://github.com/user-attachments/assets/f595701c-d825-4e3e-bde2-fafbabdf754b" />

To access the database, we need two key pieces of information: the Project ID and the Collection Name. The Project ID is typically found in the strings.xml file.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_13_04_23" src="https://github.com/user-attachments/assets/b7a44e8c-735e-4566-8419-913d8ece34c0" />

Inside res/values/strings.xml, we successfully located the Firebase Project ID.

    Project ID: myglobalbank-b212d

<img width="1920" height="1080" alt="Screenshot_2026-04-21_13_05_08" src="https://github.com/user-attachments/assets/880b9654-39df-40b3-8700-4b0511489339" />

We search for the Firebase Firestore REST API documentation to construct a direct access URL. Using the Project ID found earlier, we can access the database via a simple HTTP request.

REST API Format

https://firestore.googleapis.com/v1/projects/{project_id}/databases/(default)/documents/

<img width="1920" height="1080" alt="Screenshot_2026-04-21_14_02_50" src="https://github.com/user-attachments/assets/3bbf40cb-9811-4c2c-88c2-77cfaed4de5d" />

After analyzing the Smali files, I suspected a collection named **users** might exist.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_14_28_08" src="https://github.com/user-attachments/assets/0301dfe2-28f1-42c3-ab3f-96fac81e9d7a" />

To confirm and find valid users, I intercepted the request and routed it through Burp Suite using --proxy 127.0.0.1:8080

<img width="1920" height="1080" alt="Screenshot_2026-04-21_14_30_06" src="https://github.com/user-attachments/assets/185e5110-99e2-46a9-b637-5ac53a2d0e02" />

I sent the request to Burp Intruder and performed a Sniper attack using a simple wordlist to fuzz for valid usernames within the Firestore path.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_14_32_24" src="https://github.com/user-attachments/assets/274bfb2a-79a0-4d9e-9af3-dcd49230049c" />

The Intruder attack was successful. One of the requests returned a 200 OK response containing the credentials for the admin account.

    Admin Email: farasat.abasov@mygoldbank.az
    Password: crucio123

<img width="1920" height="1080" alt="Screenshot_2026-04-21_14_33_58" src="https://github.com/user-attachments/assets/292bb5d0-032d-4f86-b6c3-dbeeadb5c84c" />

## 2. Dashboard - Insecure Logging (CWE-532)

### Vulnerability Type
**CWE-532:** Insertion of Sensitive Information into Log File

### Description
The application logs sensitive information including user balance and Admin Panel access token.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_14_35_39" src="https://github.com/user-attachments/assets/4e79d186-9dac-413a-b84d-ddc0cec9604c" />

I analyzed the Dashboard Activity to understand how it handles sensitive data. To speed up the process, I used AI-powered tools to decompile and interpret the Dalvik (Smali) code, which is an efficient way to map out application logic.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_14_55_48" src="https://github.com/user-attachments/assets/1a09eaa4-012e-43fb-9bf9-a21b1e324aa5" />

To understand the complex Dalvik (Smali) bytecode of the Dashboard activity, I utilized Grok AI.

The Prompt Used:

    "Analyze the provided Smali (Dalvik) code to identify logic flaws, security vulnerabilities, or potential bypasses."

<img width="1920" height="1080" alt="Screenshot_2026-04-21_15_05_49" src="https://github.com/user-attachments/assets/9f2b6ce2-7253-4aab-b91d-a46b8d2d6ee9" />

Grok identified a high-severity vulnerability in the checkDeepLink() method. The application blindly trusts any Intent with the custom scheme goldbank://admin.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_15_07_50" src="https://github.com/user-attachments/assets/e18faa44-9b78-432e-933a-49f31a70463e" />

I can bypass authentication and jump straight to the Admin Panel by triggering the deep link via ADB.

adb shell am start -a android.intent.action.VIEW -d "goldbank://admin"

<img width="1920" height="1080" alt="Screenshot_2026-04-21_15_08_22" src="https://github.com/user-attachments/assets/3f95a8ae-27fb-43fc-815b-27b556c604d6" />

## 3. Hidden 5-Click Admin Backdoor ##

**Analysis**

Another backdoor was found in the setupHiddenClick() method. By clicking the balanceAmountText (Balance TextView) exactly 5 times, the application triggers the AdminPanelActivity.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_15_10_24" src="https://github.com/user-attachments/assets/727136d2-bb40-46c7-956d-dd9565f0b522" />


## Exploitation Steps: ##

    Login to the User Dashboard.

    Locate the balance display.

    Click on the balance amount 5 times.

    The Admin Panel opens, bypassing all security checks.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_15_10_40" src="https://github.com/user-attachments/assets/83b6734e-a970-4121-8d46-5df8e6564621" />

## 4. Advanced Logic Bypass via Frida Hooking

**Vulnerability**

The application relies on client-side validation to prevent negative transfers or empty inputs. However, the transaction logic and balance storage (SharedPreferences) are accessible within the application's process memory.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_22_39_12" src="https://github.com/user-attachments/assets/93168aa0-bd2d-40bd-a7e0-94e1801bc215" />

## What is Frida?

Frida is a world-class dynamic instrumentation toolkit. It allows developers, analysts, and security researchers to inject custom scripts (written in JavaScript) into black-box processes. In the context of Android security, it is used to:

    Hook functions: Intercept and modify function calls in real-time.

    Modify variables: Change values (like balances or flags) stored in memory.

    Bypass checks: Skip SSL pinning, root detection, or—as in our case—client-side logic validations.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_22_43_10" src="https://github.com/user-attachments/assets/fe2f7c62-8497-49d2-be0d-b5352b1362cc" />

To begin the exploitation of the TransferActivity, we first need to set up the environment and execute the tool:

    Frida-Server Deployment:
    First, the frida-server binary (matching the device architecture) must be uploaded to the Android device and executed with root privileges.
    

adb push frida-server /data/local/tmp/
adb shell "chmod +x /data/local/tmp/frida-server"
adb shell "/data/local/tmp/frida-server &"

<img width="1920" height="1080" alt="Screenshot_2026-04-21_22_46_38" src="https://github.com/user-attachments/assets/d999bbde-d6f8-45c8-9075-5dea751779d0" />

## Exploitation Process

Identifying the Logic Flaw in Smali
By analyzing the TransferActivity.smali file, we can pinpoint exactly where the transaction logic resides.

    Logic Location: The method .method private processTransaction()V (Line 569) handles the entire transfer process.

    The Vulnerability: At Lines 572-585, the code retrieves input from cardNumberInput and amountInput, then proceeds to validate and calculate the balance locally. Because this logic is executed entirely on the client-side within the JVM/Dalvik environment, it is susceptible to Dynamic Instrumentation.
    
<img width="1920" height="1080" alt="Screenshot_2026-04-21_22_49_03" src="https://github.com/user-attachments/assets/c2155148-9790-4499-b037-a7cb3695e405" />


The Frida script (mgb_exploit.js) performs a "Method Hook" to hijack the application's execution flow in real-time:c

<img width="1920" height="1080" alt="Screenshot_2026-04-21_22_50_37" src="https://github.com/user-attachments/assets/c0e24ccd-c5ed-454e-b80b-47d931c71759" />


    Method Interception: Frida intercepts the call to processTransaction(). Instead of running the original Smali code, it executes our custom JavaScript function.

    Input Manipulation: The script checks the user's input. If the input is empty or invalid (isNaN), it programmatically forces the userAmount to 1000.0.

    Bypassing Validations: The original Smali checks (like ensuring the amount is greater than zero or less than the current balance) are completely skipped.

    Direct Memory Write: The script accesses the SharedPreferences API directly. It reads the current user_balance, adds the forced amount, and commits the change using prefs.edit().putFloat().apply().


<img width="1920" height="1080" alt="Screenshot_2026-04-21_22_52_29" src="https://github.com/user-attachments/assets/37b2abb4-9dfc-4bc7-b520-1b184f80f000" />

The attack is successful because the application lacks server-side verification. By hooking the local method, we can manipulate the financial state of the app without ever providing valid data through the UI

<img width="1920" height="1080" alt="Screenshot_2026-04-21_22_52_42" src="https://github.com/user-attachments/assets/583da2c3-d5a8-4f41-a534-08b255f9fbb2" />

## 5.TransferActivity - External Intent Vulnerability (CWE-927)

**Vulnerability Type**

CWE-927: Use of Implicit Intent for Sensitive Communication

**Description**

The TransferActivity is exported and contains a logic flaw in the checkExternalIntent() method. It listens for external intents containing specific extras (auto_transfer_amount and target_card). If these extras are present, the application automatically populates the transfer fields and initiates a transaction without any user confirmation or origin validation.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_22_58_12" src="https://github.com/user-attachments/assets/fcb43a3e-fa02-4326-bfc0-5190e811e4bc" />

**Static Analysis**

In the AndroidManifest.xml, we can see that TransferActivity is set to android:exported="true", making it accessible to any other application on the device.

<img width="1920" height="1080" alt="Screenshot_2026-04-21_23_00_30" src="https://github.com/user-attachments/assets/deff5371-f4ab-4a75-b95f-655571463458" />

In the source code (JADX), the checkExternalIntent() method processes these values and triggers processTransaction() after a short delay:

if (intent.hasExtra("auto_transfer_amount") && intent.hasExtra("target_card")) {
    // ... logic to auto-fill and process ...
    android.util.Log.d("CTF_EXTERNAL", "FLAG{3xt3rn4l_int3nt_4bus3}");
}

**Exploitation (POC)**

We can exploit this by sending a crafted Intent via ADB. This simulates a malicious app on the device trying to force a bank transfer.

adb shell am start -n com.global.mygoldbank/.TransferActivity \
  --es auto_transfer_amount "1000" \
  --es target_card "1234567891234567"

  <img width="1920" height="1080" alt="Screenshot_2026-04-21_22_59_04" src="https://github.com/user-attachments/assets/50a03dec-eb0a-4f26-97a5-07a084b6dcfe" />

## 6.Real-World Exploit Scenario: Malicious App Integration

**Concept**

To demonstrate the severity of the External Intent Vulnerability, I developed a proof-of-concept (PoC) malicious application disguised as a system utility named "System Update" (com.hacker.malicious).

**Attack Mechanism**

The malicious app runs a background service (SilentExploitService) that monitors the device state. Once triggered, it programmatically sends a crafted implicit intent to the MyGlobalBank application.

**The Code (Malicious App):**

The attacker's app uses the following logic to trigger the transfer without any user interaction:

Intent intent = new Intent();
intent.setComponent(new ComponentName("com.global.mygoldbank", "com.global.mygoldbank.TransferActivity"));
intent.putExtra("auto_transfer_amount", "5000");
intent.putExtra("target_card", "4444333322221111");
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);

<img width="1920" height="1080" alt="Screenshot_2026-04-21_23_24_49" src="https://github.com/user-attachments/assets/306ad325-7527-468e-ab7d-1cf1c7077d03" />

Background Trigger: As seen in the screenshots, the malicious app starts the exploit service.

Automated Transfer: The TransferActivity of the bank app is forced into the foreground, automatically processes a $5000 transfer to the attacker's card.


<img width="1920" height="1080" alt="Screenshot_2026-04-21_23_25_08" src="https://github.com/user-attachments/assets/cc85e9fd-01ed-4b5e-b1f4-62459b5c6064" />

## 7.RequestActivity - IDOR & Information Disclosure (CWE-639)

**Vulnerability Type**

CWE-639: Insecure Direct Object Reference (IDOR)

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_12_25" src="https://github.com/user-attachments/assets/4ee81327-0cc8-4ff9-8137-d7dabecbabdd" />

## Bypassing SSL Pinning & Traffic Inspection Setup

**The Challenge**

To perform an IDOR attack by intercepting backend requests, we must monitor the application's HTTPS traffic. However, modern Android versions do not trust user-installed CA certificates by default, which prevents tools like Charles Proxy from decrypting the traffic.

**Modifying Network Security Configuration**

To bypass this restriction, I de-compiled the APK and modified the network security settings to explicitly trust user-installed certificates.

**Step 1: Creating the Configuration**

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_14_16" src="https://github.com/user-attachments/assets/437416b0-0649-44eb-b841-31481f144b96" />

I created a new file at res/xml/network_security_config.xml with the following rules:

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_14_27" src="https://github.com/user-attachments/assets/734b7ccd-a91a-4876-81b0-89842801b50a" />

**Step 2: Updating the Android Manifest**

To apply this configuration, I modified the <application> tag in the AndroidManifest.xml to include the networkSecurityConfig attribute:

<application
    ...
    android:networkSecurityConfig="@xml/network_security_config"
    ...>

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_15_18" src="https://github.com/user-attachments/assets/16cefba7-b585-4a8f-978b-f53d472133db" />

After modifying the network security configurations, the APK must be recompiled and signed to be functional. I used Apktool to build the modified project, but I encountered several resource errors related to color definitions in res/values/colors.xml.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_30_55" src="https://github.com/user-attachments/assets/50e03d8b-8acd-41aa-a435-450989125f24" />

Since Android requires all applications to be digitally signed, I used keytool to generate a custom keystore named mybank.keystore and then applied the signature using jarsigner. This manual signing process is critical because it allows the device to install the patched APK as a trusted package.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_34_01" src="https://github.com/user-attachments/assets/9c174914-078a-4e68-80b2-247a221cd506" />

Next, we use a tool called Charles Proxy. Charles Proxy is an HTTP proxy / HTTP monitor / Reverse Proxy that enables a developer to view all of the HTTP and SSL/TLS traffic between their machine and the Internet. In our mobile penetration testing scenario, it acts as a "Man-in-the-Middle," allowing us to intercept, inspect, and even modify the data being sent and received by the application in real-time.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_36_44" src="https://github.com/user-attachments/assets/625af455-7320-4a5f-9eca-c8aef0a52de9" />

With the patched and signed APK now running on the device, we launch Charles Proxy to begin the monitoring process.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_37_40" src="https://github.com/user-attachments/assets/3ee60856-02ed-4de0-bf0a-d33915ba3ec2" />

In the Charles Proxy settings, we configure the port to 8888 

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_38_27" src="https://github.com/user-attachments/assets/85e105cd-817d-4d1d-ac49-c2b8369f633f" />

Enable SSL Proxying by adding *.* to the locations to intercept all incoming and outgoing traffic.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_38_37" src="https://github.com/user-attachments/assets/6af346ee-2789-4c93-9f71-b980f825fa57" />

To link the device, we identify the machine's local IP address from the wlan0 interface via the Help menu and enter this IP, along with port 8888, into the Android device's manual proxy settings.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_39_15" src="https://github.com/user-attachments/assets/72c67116-7e9a-47a7-92e0-3d27f6d9481b" />

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_42_00" src="https://github.com/user-attachments/assets/893a757a-e780-4abe-b08b-cb34c00e2367" />

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_42_50" src="https://github.com/user-attachments/assets/77602310-7fbb-48af-8b47-d634daa59908" />


Next, we navigate back to the SSL Proxying menu in Charles to save the Charles Root Certificate, which is exported as a .pem file. This certificate must be transferred to the Android device and installed via the "Install from storage" option within the system's Security/Encryption settings (Trusted Credentials) to allow the device to trust the proxy's intercepted traffic.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_41_49" src="https://github.com/user-attachments/assets/7d5a7e52-b18a-4dcd-8527-b866cfa66ac6" />

Once the setup is complete, we select Start SSL Proxying and Start Recording in Charles. We then perform a "Request Money" transaction to the victim's card number (4444333322221111).

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_43_55" src="https://github.com/user-attachments/assets/bf7f19bb-7707-4529-bfcb-c387008b1d19" />

By monitoring the live traffic, we can clearly see the request being routed to the Firestore backend at the /users/ endpoint.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_44_15" src="https://github.com/user-attachments/assets/742c0c45-ceb1-4513-b384-23f9359b1e7a" />

**Using the extracted REST API path, we execute a final cURL command:**

curl -X GET "https://firestore.googleapis.com/v1/projects/myglobalbank-b212d/databases/(default)/documents/users/victim"

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_46_55" src="https://github.com/user-attachments/assets/9c221e7f-1136-450e-9fe3-e7537f9b0035" />

## 8.NewsActivity - File Access via WebView (CWE-552)

The NewsActivity contains a critical security flaw involving the WebView component. Static analysis reveals that the WebView has allowFileAccess enabled and, more importantly, exposes a custom JavaScript interface named CTFFileReader.

This interface acts as a bridge between the web content and the Android native Java code, allowing JavaScript to execute sensitive file system operations.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_55_40" src="https://github.com/user-attachments/assets/5c252493-27a8-4287-be2c-f62458edb618" />

By injecting a stored XSS payload into the news database, I was able to force the application to execute JavaScript when a user views the news feed.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_56_24" src="https://github.com/user-attachments/assets/83f76d89-50a1-4fb9-a180-51f37cf4df9c" />

I used the following payload to interact with the exposed interface:

'); CTFFileReader.listFiles('/data/data/com.global.mygoldbank/databases'); //

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_57_08" src="https://github.com/user-attachments/assets/d00132be-bf51-46ec-a572-790d874c7e42" />

This exploit successfully bypassed the application's sandbox. By calling the listFiles method through the WebView, I was able to list the contents of the internal /databases directory.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_07_58_13" src="https://github.com/user-attachments/assets/7ea492a2-5973-46de-97da-7e1ed0578e34" />

## 9. VaultActivity - AccountManager Privilege Escalation (CVE-2023-20963)

**Vulnerability Type CVE-2023-20963: Android AccountManager Bundle Mishandling**

The VaultActivity introduces a sophisticated attack surface through its "Enable Cloud Backup" feature, which leverages the Android AccountManager system service. Static analysis of the AndroidManifest.xml and the VaultAuthenticatorService reveals that the service is exported and thus reachable by any external application on the device

<img width="1920" height="1080" alt="Screenshot_2026-04-22_08_06_23" src="https://github.com/user-attachments/assets/bfc54baa-d60d-45de-a8d8-d04c01cbe8ed" />

## The core vulnerability (CVE-2023-20963) is rooted in VaultSyncService.java, specifically within the addAccount method, where the system retrieves a Parcelable object from the options bundle using the sync_intent key

<img width="1920" height="1080" alt="Screenshot_2026-04-22_08_12_44" src="https://github.com/user-attachments/assets/f6cd84e4-133c-4df6-9986-a9a8373a749a" />

Because this intent is processed without any validation, it creates a direct pathway for a Privilege Escalation attack. To exploit this, I developed a custom exploit application that calls AccountManager.addAccount() and passes a hidden intent targeting the restricted AdminVaultActivity

<img width="1920" height="1080" alt="Screenshot_2026-04-22_08_12_52" src="https://github.com/user-attachments/assets/4c3c71e6-0e4b-4c81-9e03-60640165ccd2" />

Since this malicious intent is processed and launched by the system's own AccountManager, it completely bypasses the exported="false" security restriction.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_08_16_27" src="https://github.com/user-attachments/assets/d1dd4857-ed2e-4747-9a44-7a1cf185f708" />

Upon triggering the exploit, the unauthorized "ADMIN VAULT" screen was forced into the foreground, revealing an encrypted string: ZNFGER_ERPBVREL_XRL: 88-K99-0NAX-2026

<img width="1920" height="1080" alt="Screenshot_2026-04-24_14_39_42" src="https://github.com/user-attachments/assets/93480509-5b5e-4872-b7cb-4dee6f9e1a3e" />

By identifying the encryption as ROT-13 and passing it through a decoder, I successfully retrieved the final Master Recovery Key: MASTER_RECOVERY_KEY: 88-X99-BANK-2026

<img width="1920" height="1080" alt="Screenshot_2026-04-24_14_39_52" src="https://github.com/user-attachments/assets/31f0999b-29da-49aa-b725-f4dbdbdf144c" />

To finalize the decryption of the sensitive data found within the hidden Admin Vault, I used the rot13.com online decoder to process the encoded string ZNFGER_ERPBVREL_XRL: 88-K99-0NAX-2026. This simple substitution cipher shifted the characters to reveal the plain-text Master Recovery Key: 88-X99-BANK-2026

<img width="1920" height="1080" alt="Screenshot_2026-04-24_14_42_42" src="https://github.com/user-attachments/assets/cedd40f7-80de-45dc-bd06-4c3fca12da27" />

## The acquisition of this Master Key represents the ultimate breach of the application's security architecture. In a real-world banking environment, a Master Key is essentially a "god-mode" credential designed for emergency account recovery or administrative overrides. By possessing this key, an attacker can bypass every individual security layer, including PIN codes, biometric locks, and multi-factor authentication (MFA). It grants absolute control over the entire vault's encrypted assets and sensitive user records, effectively rendering the application's primary defense mechanisms completely obsolete.

## 10.Vault Login Analysis - Hardcoded Secrets & Broken MFA

The investigation into the VaultLoginActivity exposed a series of catastrophic security failures that completely undermine the integrity of the application's most sensitive area.

<img width="1920" height="1080" alt="Screenshot_2026-04-22_17_39_31" src="https://github.com/user-attachments/assets/f95ff8e5-5c0f-49b3-a41c-ff350e56f46f" />

My analysis began by reverse-engineering the application to examine the underlying logic, specifically targeting the VaultLoginActivity.smali file. Within this file, I discovered hardcoded administrative credentials stored as plain-text static fields, revealing the valid username as **vault_admin** and the password as **SecurePass123**

<img width="1920" height="1080" alt="Screenshot_2026-04-24_14_49_17" src="https://github.com/user-attachments/assets/80850c88-4419-4d8f-95cb-774be1d5b461" />

This presence of hardcoded secrets is a critical oversight, as it allows any attacker with access to the decompiled APK to identify administrative entry points without needing to compromise a legitimate user's account.

<img width="1920" height="1080" alt="Screenshot_2026-04-24_14_50_00" src="https://github.com/user-attachments/assets/bc8ce213-6176-4eb3-87e1-decf893469a0" />

Furthermore, the application's attempt at implementing Multi-Factor Authentication (MFA) is fundamentally broken. Although the "Vault Access" screen prompts for a One-Time Password (OTP) to finalize the login process, the system generates and displays the verification code—in this case, 7152—directly on the UI for the user to see. This implementation negates the entire purpose of out-of-band authentication, as the "secret" code is leaked at the very moment it is required. By simply entering the hardcoded credentials and the visible OTP, I bypassed all security layers and gained unrestricted access to the "Secure Vault.

<img width="1920" height="1080" alt="Screenshot_2026-04-24_14_50_09" src="https://github.com/user-attachments/assets/38810459-52a3-4a88-9013-485642aeb119" />





