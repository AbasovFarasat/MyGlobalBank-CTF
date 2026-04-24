# My Global Bank - CTF Challenge

A deliberately vulnerable Android banking application for CTF competitions. Contains **10 real-world vulnerabilities**.

##  Quick Start

adb install MyGlobalBank_CTF.apk

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





















