# BDLS System - Environment Configuration & Third-Party Services Setup Guide

This document provides a complete, step-by-step technical guide for configuring the environment variables (`.env`) and integrating all required third-party services for the **Barangay Doña Lucia Services (BDLS)** platform.

---

## Table of Contents
1. [Quick Start & Local Development Modes](#1-quick-start--local-development-modes)
2. [Google Cloud Platform: Cloud Vision OCR & Service Account](#2-google-cloud-platform-cloud-vision-ocr--service-account)
3. [SMS Gateway 1: PinasSMS Setup](#3-sms-gateway-1-pinassms-setup)
4. [SMS Gateway 2: Fortmed SMS (FMCSMS) Setup](#4-sms-gateway-2-fortmed-sms-fmcsms-setup)
5. [Transactional Email Setup (Gmail SMTP & EmailJS)](#5-transactional-email-setup-gmail-smtp--emailjs)
6. [Real-time WebSockets (Laravel Reverb)](#6-real-time-websockets-laravel-reverb)
7. [Full `.env` Template with Placeholders](#7-full-env-template-with-placeholders)

---

## 1. Quick Start & Local Development Modes

The BDLS platform includes **built-in zero-cost development/mock modes**. Evaluators and developers do not need active API subscriptions or external billing accounts to test all primary features:

| Feature | Mock/Local Mode (Recommended for Grading) | Live Cloud Mode |
| :--- | :--- | :--- |
| **KYC / Valid ID OCR** | `OCR_DRIVER=mock`<br>*(Instantly validates Resident Masterlist name without using Cloud Vision credits)* | `OCR_DRIVER=live`<br>*(Requires Google Cloud Service Account JSON)* |
| **SMS Notifications & OTP** | `SMS_DRIVER=log`<br>*(OTPs & messages are written to `storage/logs/laravel.log`)* | `SMS_DRIVER=hybrid`<br>*(Dispatches to PinasSMS & FMCSMS gateways)* |
| **Transactional Emails** | `MAIL_MAILER=log`<br>*(Logs sent emails to `storage/logs/laravel.log`)* | `MAIL_MAILER=smtp`<br>*(Uses Gmail App Password or custom SMTP)* |

---

## 2. Google Cloud Platform: Cloud Vision OCR & Service Account

The Automated KYC module uses the Google Cloud Vision API to extract text from resident-submitted ID cards in memory and matches it against the official Barangay Masterlist.

### Step 2.1: Create a Google Cloud Account & Project
1. Navigate to the [Google Cloud Console](https://console.cloud.google.com/).
2. Sign in with your Google Account.
3. In the top project selector bar, click **Select a Project** > **New Project**.
4. Name the project (e.g., `bdls-kyc-vision`) and click **Create**.

### Step 2.2: Enable Cloud Vision API
1. In the console search bar at the top, type `Cloud Vision API`.
2. Select **Cloud Vision API** from the Marketplace/APIs search results.
3. Click **Enable**. *(Note: Google offers free monthly tier quotas for Vision API requests).*

### Step 2.3: Create a Dedicated Service Account
1. From the navigation menu (top-left burger icon), go to **IAM & Admin** > **Service Accounts**.
2. Click **+ Create Service Account** at the top.
3. Enter the details:
   - **Service account name:** `bdls-vision-runner`
   - **Service account ID:** `bdls-vision-runner` (auto-generated)
   - **Description:** `Service account for BDLS in-memory OCR text detection`
4. Click **Create and Continue**.
5. Under **Grant this service account access to project**, choose:
   - **Role:** `Cloud Vision API User` (or `Service Usage Consumer`)
6. Click **Done**.

### Step 2.4: Generate & Download the JSON Key File
1. In the **Service Accounts** list, find your newly created account (`bdls-vision-runner@...`).
2. Click on the account name or the three vertical dots (Actions menu) on the right, and select **Manage keys**.
3. Click **Add Key** > **Create new key**.
4. Select **JSON** as the key type and click **Create**.
5. Your browser will download a file named `<project-id>-<unique-id>.json`.

### Step 2.5: Place Key in BDLS Project
1. Rename the downloaded file to:
   ```text
   google-credentials.json
   ```
2. Place it inside the private storage directory of the BDLS project:
   ```text
   storage/app/private/google-credentials.json
   ```
   *(If the directory `storage/app/private/` does not exist, create it).*

3. Update your `.env` configuration:
   ```env
   OCR_DRIVER=live
   GOOGLE_CREDENTIALS_PATH="app/private/google-credentials.json"
   GOOGLE_VISION_API_URL="https://vision.googleapis.com/v1/images:annotate"
   GOOGLE_AUTH_SCOPE="https://www.googleapis.com/auth/cloud-platform"
   ```

*(Alternative for Cloud/Server Deployments: You can base64-encode the JSON and store it in `GOOGLE_CREDENTIALS_BASE64` or stringify it into `GOOGLE_CREDENTIALS_JSON`).*

---

## 3. SMS Gateway 1: PinasSMS Setup

PinasSMS serves as Gateway 1 in the BDLS Hybrid SMS Engine (covering Globe, TM, DITO, and Smart fallback).

### Step 3.1: Register on PinasSMS
1. Go to the [PinasSMS Registration Portal](https://pinassms.com/).
2. Sign up with your mobile number and organization details.
3. Verify your mobile number via OTP.

### Step 3.2: Obtain the API Key
1. Log in to your [PinasSMS Dashboard](https://pinassms.com/login).
2. Go to the **API / Developers** or **Settings** menu.
3. Generate or copy your **API Key / Token**.
4. Check your account balance to ensure active SMS credits are loaded for sending live messages.

### Step 3.3: Configure `.env`
Update the following lines in your `.env`:
```env
# Change driver to 'hybrid' or 'pinassms' for live sending
SMS_DRIVER=hybrid
SMS_MAX_LENGTH=150
SMS_CURFEW_START=21
SMS_CURFEW_END=7
SMS_QUEUE_DELAY_MS=200000

PINASSMS_API_KEY="YOUR_ACTUAL_PINASSMS_API_KEY"
PINASSMS_BASE_URL="https://pinassms.com/api"
PINASSMS_TIMEOUT=15
```

---

## 4. SMS Gateway 2: Fortmed SMS (FMCSMS) Setup

Fortmed SMS serves as Gateway 2 in the BDLS Hybrid SMS Engine (optimized specifically for Smart / TNT prefixes).

### Step 4.1: Access Fortmed SMS
1. Visit the [Fortmed SMS Portal](https://fortmed.org/web/FMCSMS/).
2. Log in with your registered agency / institutional credentials.
3. If registering a new account, register your official Sender ID (e.g., `BDLS`).

### Step 4.2: Retrieve API Key & Endpoint
1. Navigate to the **API Integration** tab in the Fortmed SMS panel.
2. Locate the **API Key** associated with your Sender ID.
3. Confirm the endpoint URL:
   `https://fortmed.org/web/FMCSMS/api/messages.php`

### Step 4.3: Configure `.env`
```env
FORTMED_SMS_API_KEY="YOUR_ACTUAL_FORTMED_API_KEY"
FORTMED_SMS_API_URL="https://fortmed.org/web/FMCSMS/api/messages.php"
FORTMED_SMS_FROM_NUMBER="+639189876543"
FORTMED_SMS_SENDER_NAME="BDLS"
FORTMED_SMS_TIMEOUT=15
```

---

## 5. Transactional Email Setup (Gmail SMTP & EmailJS)

The system supports standard Laravel SMTP mail as well as EmailJS REST API dispatch.

### Step 5.1: Generate a Gmail App Password
1. Log in to your [Google Account Security Settings](https://myaccount.google.com/security).
2. Enable **2-Step Verification** if not already enabled.
3. In the search box at the top of your Google Account page, search for **App passwords**.
4. Enter an app name (e.g., `BDLS-System`) and click **Create**.
5. Google will display a 16-character passcode (e.g., `abcd efgh ijkl mnop`).
6. Copy this password (without spaces).

### Step 5.2: Configure Email in `.env`
```env
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=465
MAIL_USERNAME="your-email@gmail.com"
MAIL_PASSWORD="your-16-character-app-password"
MAIL_ENCRYPTION=smtps
MAIL_FROM_ADDRESS="no-reply@bdls.gov.ph"
MAIL_FROM_NAME="Barangay Dona Lucia Services"
```

*(If using EmailJS REST API):*
```env
EMAILJS_SERVICE_ID="service_bdls_app_26"
EMAILJS_TEMPLATE_ID="template_yys6oze"
EMAILJS_PUBLIC_KEY="YOUR_EMAILJS_PUBLIC_KEY"
EMAILJS_PRIVATE_KEY="YOUR_EMAILJS_PRIVATE_KEY"
```

---

## 6. Real-time WebSockets (Laravel Reverb)

BDLS uses **Laravel Reverb** for real-time updates (notifications, queue progression, status updates) without third-party fees (no Pusher subscription required).

Default local Reverb keys (these do not incur charges and can remain as-is for local evaluation):
```env
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=432512
REVERB_APP_KEY=ig0pbozt1sbf29dp61uu
REVERB_APP_SECRET=vwkntqiqh7y07p7qxi25
REVERB_HOST=127.0.0.1
REVERB_PORT=8080
REVERB_SCHEME=http

VITE_REVERB_APP_KEY="ig0pbozt1sbf29dp61uu"
VITE_REVERB_HOST="127.0.0.1"
VITE_REVERB_PORT="8080"
VITE_REVERB_SCHEME="http"
```

To run the WebSocket server locally:
```bash
php artisan reverb:start
```

---

## 7. Full `.env` Template with Placeholders

Below is the clean `.env` template safe for distribution:

```env
# ==============================================================================
# 🏢 BDLS APPLICATION CORE CONFIGURATION
# ==============================================================================
APP_NAME="BDLS"
APP_ENV=local
APP_KEY=base64:LKcYHmh3RnOz6MQaio56ASrZWKGqijqYCfcV1+glvw8=
APP_DEBUG=true
APP_URL=http://localhost:8000
APP_LOCALE=en
APP_FALLBACK_LOCALE=en
APP_FAKER_LOCALE=en_US
APP_MAINTENANCE_DRIVER=file

# ==============================================================================
# 🔐 SECURITY & LOGGING
# ==============================================================================
BCRYPT_ROUNDS=12
LOG_CHANNEL=stack
LOG_STACK=single
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

# ==============================================================================
# 🗄️ LOCAL MYSQL DATABASE
# ==============================================================================
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=bdls_database
DB_USERNAME=root
DB_PASSWORD=""

# ==============================================================================
# 🍪 SESSION SETTINGS
# ==============================================================================
SESSION_DRIVER=database
SESSION_LIFETIME=120
SESSION_ENCRYPT=false
SESSION_PATH=/
SESSION_DOMAIN=null
SESSION_SECURE_COOKIE=false
SESSION_SAME_SITE=lax

# ==============================================================================
# ⚡ CACHE, QUEUE & STORAGE ENGINE
# ==============================================================================
CACHE_STORE=database
QUEUE_CONNECTION=database
FILESYSTEM_DISK=local

# ==============================================================================
# ⚡ REAL-TIME WEBSOCKET (LARAVEL REVERB)
# ==============================================================================
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=432512
REVERB_APP_KEY=ig0pbozt1sbf29dp61uu
REVERB_APP_SECRET=vwkntqiqh7y07p7qxi25
REVERB_HOST=127.0.0.1
REVERB_PORT=8080
REVERB_SCHEME=http

# ==============================================================================
# ⚛️ VITE FRONTEND RUNTIME ASSETS
# ==============================================================================
VITE_APP_NAME="BDLS"
VITE_HMR_HOST="localhost"
VITE_REVERB_APP_KEY="ig0pbozt1sbf29dp61uu"
VITE_REVERB_HOST="127.0.0.1"
VITE_REVERB_PORT="8080"
VITE_REVERB_SCHEME="http"

# ==============================================================================
# 📱 SMS GATEWAY CONFIGURATION
# ==============================================================================
# Drivers: 'log' (for testing without API credits), 'hybrid', 'pinassms', 'fortmed'
SMS_DRIVER=log
SMS_MAX_LENGTH=150
SMS_CURFEW_START=21
SMS_CURFEW_END=7
SMS_QUEUE_DELAY_MS=200000

# Gateway 1: PinasSMS (Globe, TM, DITO + Smart Fallback)
PINASSMS_API_KEY="YOUR_PINASSMS_API_KEY"
PINASSMS_BASE_URL="https://pinassms.com/api"
PINASSMS_TIMEOUT=15

# Gateway 2: Fortmed (Smart/TNT Primary)
FORTMED_SMS_API_KEY="YOUR_FORTMED_API_KEY"
FORTMED_SMS_API_URL="https://fortmed.org/web/FMCSMS/api/messages.php"
FORTMED_SMS_FROM_NUMBER="+639189876543"
FORTMED_SMS_SENDER_NAME="BDLS"
FORTMED_SMS_TIMEOUT=15

# ==============================================================================
# 📧 EMAIL CONFIGURATION
# ==============================================================================
EMAILJS_SERVICE_ID="service_bdls_app_26"
EMAILJS_TEMPLATE_ID="template_yys6oze"
EMAILJS_PUBLIC_KEY="YOUR_EMAILJS_PUBLIC_KEY"
EMAILJS_PRIVATE_KEY="YOUR_EMAILJS_PRIVATE_KEY"
MAIL_FROM_NAME="Barangay Dona Lucia Services"

MAIL_MAILER=log
MAIL_HOST=smtp.gmail.com
MAIL_PORT=465
MAIL_USERNAME="your-email@gmail.com"
MAIL_PASSWORD="your-google-app-password"
MAIL_ENCRYPTION=smtps
MAIL_FROM_ADDRESS="no-reply@bdls.gov.ph"

# ==============================================================================
# 🛡️ AUTOMATED KYC & GOOGLE CLOUD VISION OCR
# ==============================================================================
# Drivers: 'mock' (Recommended for testing without Google Cloud setup), 'live'
OCR_DRIVER=mock
GOOGLE_CREDENTIALS_PATH="app/private/google-credentials.json"
GOOGLE_VISION_API_URL="https://vision.googleapis.com/v1/images:annotate"
GOOGLE_AUTH_SCOPE="https://www.googleapis.com/auth/cloud-platform"
```
