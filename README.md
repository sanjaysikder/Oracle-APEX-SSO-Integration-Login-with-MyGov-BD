# 🔐 Oracle APEX SSO Integration: Login with MyGov BD

## 📌 Overview

This guide explains how to integrate **MyGov Bangladesh SSO (OAuth 2.0)** with an Oracle APEX application.

You will learn how to:

* Add **"Login with MyGov"** button
* Handle OAuth **Authorization Code Flow**
* Exchange code for **Access Token**
* Fetch user profile তথ্য
* Automatically login user into APEX
* Store user তথ্য in database

---

## 🌐 Step 1: Create Login Button

### 🔘 Button Label:

```
Login with MyGov
```

### 🔗 Button URL:

```text
https://beta-idp.stage.mygov.bd/oauth/authorize?response_type=code&client_id=97e9ed94-e0f8-43d0-9570-79c3e5b6d36f&scope=openid&state=10XmMlsbgdtvSqq18zn3PT7L1flS32H2rEHmJSI9&redirect_uri=https://pcc-test.police.gov.bd/ords/pcc2/r/pcc/callback
```

---

## 📄 Step 2: Create Callback Page

Create a **Public Page** in APEX:

* Page Name: `CALLBACK`
* Page Mode: Public
* Page Number: e.g., `37`

### 📥 Create Page Item:

```
Item Name: CODE
Type: Hidden
```

This item will capture the `authorization code` from MyGov redirect.

---

## ⚙️ Step 3: Add "Before Header" Process

Add the following PL/SQL code in:

```
Processing → Before Header
```

---

## 💻 PL/SQL Code

```sql id="mygov_sso_code"
DECLARE
    l_response_clob   CLOB;
    l_request_body    CLOB;
    l_access_token    VARCHAR2(10000);

    l_mobile          VARCHAR2(20);
    l_name            VARCHAR2(200);
    v_login_mobile    VARCHAR2(20);
    l_nation_id       VARCHAR2(50);
    l_token           CLOB;

    l_result_clob2    VARCHAR2(32767);

BEGIN
    -- Step 1: Exchange Authorization Code for Token
    l_request_body :=
          'grant_type=authorization_code'
       || '&client_id=97e9ed94-e0f8-43d0-9570-79c3e5b6d36f'
       || '&client_secret=j70lUgnFUhHAukpEoAYAwAqgiz0jtub67qAhwd2J'
       || '&redirect_uri=https://pcc-test.police.gov.bd/ords/pcc2/r/pcc/callback'
       || '&code=' || :CODE;

    apex_web_service.g_request_headers.delete;

    apex_web_service.g_request_headers(1).name  := 'Content-Type';
    apex_web_service.g_request_headers(1).value := 'application/x-www-form-urlencoded';

    l_response_clob := apex_web_service.make_rest_request(
        p_url         => 'https://beta-idp.stage.mygov.bd/oauth/token',
        p_http_method => 'POST',
        p_body        => l_request_body
    );

    -- Step 2: Parse Access Token
    apex_json.parse(l_response_clob);
    l_token := apex_json.get_varchar2('access_token');

    -- Step 3: Call Profile API
    apex_web_service.g_request_headers.delete;

    apex_web_service.g_request_headers(1).name  := 'Authorization';
    apex_web_service.g_request_headers(1).value := 'Bearer ' || l_token;

    l_result_clob2 := apex_web_service.make_rest_request(
        p_url         => 'https://beta-idp.stage.mygov.bd/api/profile',
        p_http_method => 'GET'
    );

    -- Step 4: Parse User Info
    apex_json.parse(l_result_clob2);

    l_mobile    := apex_json.get_varchar2('data.mobile');
    l_name      := apex_json.get_varchar2('data.name_en');
    l_nation_id := apex_json.get_varchar2('data.nationality');

    v_login_mobile := '0' || l_mobile;

    -- Step 5: Insert User (if not exists)
    BEGIN
        INSERT INTO PCC_REGISTRATION (
            MOBILE_NO,
            NAME,
            NID,
            USER_TYPE
        )
        VALUES (
            v_login_mobile,
            l_name,
            l_nation_id,
            'MyGov'
        );
    EXCEPTION
        WHEN OTHERS THEN NULL;
    END;

    -- Step 6: Set Session Items
    :P37_MOBILE    := v_login_mobile;
    :P37_USER_INFO := l_name;

    -- Step 7: Create Cookie
    owa_util.mime_header('text/html', FALSE);
    owa_cookie.send(
        name  => 'LOGIN_USERNAME_COOKIE',
        value => LOWER(v_login_mobile)
    );

    -- Step 8: Get User ID
    BEGIN
        SELECT PID
        INTO :G_PID_REG
        FROM PC_REGISTRATION
        WHERE UPPER(MOBILE_NO) = UPPER(v_login_mobile)
        AND ROWNUM = 1;
    EXCEPTION
        WHEN OTHERS THEN NULL;
    END;

    -- Step 9: Login into APEX
    wwv_flow_custom_auth_std.login(
        p_uname      => v_login_mobile,
        p_password   => '97e9ed94e0f843d0957079c3e5b6d36f',
        p_session_id => v('APP_SESSION'),
        p_flow_page  => :APP_ID || ':1'
    );

EXCEPTION
    WHEN OTHERS THEN
        NULL;
END;
/
```

---

## 🔄 Authentication Flow

```
User Clicks Button
        ↓
Redirect to MyGov Login
        ↓
User লগইন সম্পন্ন
        ↓
Redirect to CALLBACK Page (?code=XYZ)
        ↓
Exchange Code → Access Token
        ↓
Call Profile API
        ↓
Insert / Fetch User
        ↓
Auto Login into APEX
```

---

## 🧠 Key Concepts

| Component          | Description                    |
| ------------------ | ------------------------------ |
| OAuth 2.0          | Secure authentication protocol |
| Authorization Code | Temporary code from MyGov      |
| Access Token       | Used to access user data       |
| APEX_WEB_SERVICE   | REST API call                  |
| APEX_JSON          | JSON parsing                   |

---

## ⚠️ Important Notes

* ✅ Always clear headers using:

```sql
apex_web_service.g_request_headers.delete;
```

* 🔐 Keep `client_secret` secure (do NOT expose in frontend)
* 🔄 Use HTTPS only
* ❗ Handle exceptions properly in production
* 🚫 Avoid `WHEN OTHERS THEN NULL` (log errors instead)

---

## 🚀 Best Practices

* Store token in session নিরাপদভাবে
* Add logging table for API response
* Validate `CODE` before using
* Implement refresh token logic (if supported)

---

## 📂 Use Case

* Government SSO Integration
* Citizen Authentication System
* Police Clearance / PCC System
* Secure Login without password

---


 ## Thank you
 ## Sanjay Sikder

You can connect with me on <a href="https://www.linkedin.com/in/sanjay-sikder/" target="_blank">LinkedIn</a>!

---



---




