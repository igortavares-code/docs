---
id: react-native-ory-auth
title: Adding Authentication with Ory to a React Native App
author: Igor Silva
sidebar_label: React Native Auth Integration
description: Step-by-step guide on how to integrate Ory Kratos authentication in a React Native app.
---

# Adding Authentication with Ory to a React Native App

This tutorial shows how to integrate **Ory Kratos** into a React Native app, enabling secure login and registration.  
You'll learn to run Ory locally, connect your app, and protect user data with authentication.

Authentication is a core part of most applications: it verifies who a user is before granting access to data or features.  
Before we start, let's clarify the difference between authentication and authorization.

:::tip
**Authentication** (authn) confirms who a user is.  
**Authorization** (authz) defines what a user can do.  
This tutorial focuses on **authentication**.
:::

### Prerequisites

Before starting this tutorial, make sure you have the following installed and ready:

- **Basic knowledge of React Native** – understanding of components, state management, and navigation.  
- **Node.js (≥18, LTS recommended)** installed. [Download Node.js](https://nodejs.org/)  
- **Expo CLI** installed (we’ll use Expo for simplicity). [Install Expo CLI](https://docs.expo.dev/get-started/installation/)  
- **Ory CLI** installed to run Ory Kratos locally. [Ory CLI installation guide](https://www.ory.sh/docs/guides/cli/installation)  
- **Docker Desktop** (or another container runtime) to run Ory services. [Docker Desktop](https://www.docker.com/products/docker-desktop/)  
- A code editor such as **VS Code**. [VS Code](https://code.visualstudio.com/)

**Optional but recommended:**  
- Familiarity with sessions, login flows, and JSON APIs.

### Step 1 — Run Ory Kratos Locally

In this step, you'll start Ory Kratos locally using Docker. This will provide the backend services needed for authentication.

#### 1. Start Ory Kratos

You can start Kratos in two ways:

1. **Quick test (development mode)** – runs in-memory with temporary data. Good for fast prototyping, but all identities are lost when the container stops.

```
docker run --rm -it \
    -p 4433:4433 \
    -p 4434:4434 \
    oryd/kratos:latest \
    serve --dev
```

2. **Full configuration** – runs with a custom configuration file (`kratos.yml`) and identity schema. Required if you want custom flows, schemas, or persistence.

```
docker run --rm -it \
  -p 4433:4433 \
  -p 4434:4434 \
  -v ${PWD}/path/to/config:/etc/config/kratos \
  oryd/kratos:latest \
  serve -c /etc/config/kratos/kratos.yml
```

:::note
Windows users:  
- In PowerShell, replace ' \ ' with ' ` ' at the end of each line.  
- In CMD, put the entire command on a single line.
- ```${PWD}``` variable: works on Linux/macOS. On Windows PowerShell, it needs to be replaced with / or an absolute path.
:::

**Example Configuration**
<!-- - You must provide a valid ```kratos.yml``` (see example below).
- For testing purposes, using ```oryd/kratos:latest``` is fine; for production, consider pinning a specific version for stability (e.g., ```oryd/kratos:v1.18.0```). -->

```kratos.yml```
```yaml
serve:
  public:
    base_url: http://<LAN_IP>:4433/   # LAN_IP allows access from physical devices
    host: 0.0.0.0                     # required for external access
    port: 4433
  admin:
    base_url: http://<LAN_IP>:4434/
    host: 0.0.0.0
    port: 4434

dsn: memory                           # in-memory storage (not persisted)

identity:
  default_schema_id: default
  schemas:
    - id: default
      url: file:///etc/config/kratos/identity.schema.json

selfservice:
  default_browser_return_url: http://localhost:4455/
  flows:
    login:
      ui_url: http://localhost:4455/login
    registration:
      ui_url: http://localhost:4455/registration

  methods:
    password:
      enabled: true                   # password-based login/registration enabled
    oidc:
      enabled: false

log:
  level: debug                        # verbose logs during development
```
```identity.schema.json```
```json
{
  "$id": "https://example.com/identity.schema.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Default Identity Schema",
  "type": "object",
  "properties": {
    "traits": {
      "type": "object",
      "properties": {
        "email": {
          "type": "string",
          "format": "email",
          "title": "Email",
          "ory.sh/kratos": {
            "credentials": {
              "password": {
                "identifier": true
              }
            }
          }
        }
      },
      "required": ["email"]
    }
  }
}
```

Place both files (```kratos.yml``` and ```identity.schema.json```) inside your config folder, e.g., ```./config/``` and mount it:

```-v ${PWD}/config:/etc/config/kratos```

This ensures Kratos can load both the config file and the identity schema.

### 🔧 Config Improvements

| Improvement | Why it matters |
|-------------|----------------|
| **`host: 0.0.0.0`** | Allows access from LAN devices (e.g. Expo on phone). |
| **`<LAN_IP>` in `base_url`** | Works for both PC and phone, avoids `localhost` issues. |
| **`ui_url` values** | Required so Kratos knows where to send users during login/registration flows. |
| **`methods.password.enabled: true`** | Enables quick password login/registration for testing. |
| **`log.level: debug`** | Easier debugging during development. |
| **`dsn: memory`** | Simplifies local setup, no DB needed (⚠️ not for production). |

> ⚠️ **Production note:**  
> Replace `dsn: memory` with a real database (e.g. Postgres) and lower log verbosity.  
> Example:
> ```yaml
> dsn: postgres://user:password@postgres:5432/kratos?sslmode=disable&max_conn_lifetime=1h
> ```

#### 2. Verify Kratos is running

Before testing the health endpoint, **make sure the Kratos container is running**. In PowerShell or a terminal, check:

```docker ps```

You should see a container with ```oryd/kratos:latest``` running.
If not, start it using your **full configuration** from last step again.

:::note
 "Keep this terminal open while testing; the container stops when you close it."
:::

Once the container is running, check the health endpoint **from your PC**:

**Linux/macOS**:

```bash 
curl http://localhost:4434/health/alive
```

**Windows PowerShell:**

```powershell
Invoke-WebRequest http://localhost:4434/health/alive
```

You should get a response like:

```{"status":"ok"}```

:::tip Mobile testing
The above check uses localhost and only works on the same machine as the Kratos container.
To test from a physical device or Expo on your phone, replace localhost with your PC's LAN IP, for example:

```curl http://<LAN_IP>:4434/health/alive```


:::

### Step 2 — Connect Your React Native App

Now that Ory Kratos is running locally, let's connect your React Native app to it.

#### 1. Install dependencies

In your project folder, install Axios (for API requests) and any other required libraries:

```bash
npm install axios
```
#### 2. Configure the API client

Create a new file ```oryClient.js``` to handle requests to Ory Kratos:

```javascript
import axios from "axios";
import { Platform } from "react-native";
import { getSessionTokenSync } from "./authToken";

// Define host depending on the environment
const HOST =
  Platform.OS === "android"
    ? "10.0.2.2"   // Android emulator
    : "<LAN_IP>";  // iOS simulator; replace <LAN_IP> with your machine's LAN IP for real devices

export const oryApi = axios.create({
  baseURL: `http://${HOST}:4433`, // Kratos public endpoint
  withCredentials: false, // mobile does not rely on cookies
  headers: { "Content-Type": "application/json", Accept: "application/json" },
  timeout: 10000,
});

// Interceptor: automatically adds X-Session-Token header if a token exists
oryApi.interceptors.request.use((config) => {
  const token = getSessionTokenSync();
  if (token) {
    config.headers = config.headers ?? {};
    (config.headers as any)["X-Session-Token"] = token;
  }
  return config;
});

// Initialize a login or registration flow
export async function initFlow(type) {
  try {
    const { data } = await oryApi.get(`/self-service/${type}/api`);
    return data; // contains flow data including ui.action and csrf_token
  } catch (err: unknown) {
    if (axios.isAxiosError(err)) {
      console.error(
        `Failed to initialize ${type} flow:`,
        err.message,
        err.code,
        err.response?.status
      );
      throw new Error(err.message);
    } else if (err instanceof Error) {
      throw err;
    } else {
      throw new Error("Unknown error in initFlow");
    }
  }
}
```

#### 3. Implement a login flow

Example of a simple login function:

```javascript
import { oryApi, initFlow } from "./oryClient";
import { setSessionToken } from "./authToken";
import axios from "axios";

export async function login(email: string, password: string) {
  // Step 1: Initialize login flow
  const flow = await initFlow("login");

  let actionUrl = flow.ui?.action;

  try {
    // Step 2: Submit login credentials
    const { data } = await oryApi.post(actionUrl, {
      method: "password",
      identifier: email, // ✅ login expects "identifier"
      password,
    });

    // Step 3: Save the session token (required for mobile, no cookies)
    if (data.session_token) {
      await setSessionToken(data.session_token);
    }

    return data;
  } catch (err) {
    if (axios.isAxiosError(err)) {
      throw new Error(
        err.response?.data?.ui?.messages?.map((m: any) => m.text).join("\n") ||
          err.message
      );
    }
    throw err;
  }
}
```
#### 4. Notes

Replace localhost with your Ory Kratos server URL when deploying.

This is a basic example; later steps will show handling errors and sessions.

### Step 3 — Create Login and Registration Screens

Now that we have the API client set up, let's create simple login and registration screens.

#### 1. Create Login Screen

Create a new file `LoginScreen.js`:

```javascript
import React, { useState } from "react";
import {
  View,
  TextInput,
  Button,
  Alert,
  ActivityIndicator,
} from "react-native";
import { login } from "../login";

export default function LoginScreen() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [loading, setLoading] = useState(false);

  const handleLogin = async () => {
    setLoading(true);
    try {
      const data = await login(email, password);

      if (data.session_token) {
        Alert.alert("✅ Logged in", "Session started successfully!");
        // TODO: Navigate to your main app screen here
      } else {
        Alert.alert("⚠️ No session", "Login succeeded but no session token found.");
      }
    } catch (err: any) {
      Alert.alert("Login failed", err.message || "Unknown error");
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={{ padding: 16, flex: 1, justifyContent: "center" }}>
      <TextInput
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
        style={{
          marginBottom: 12,
          padding: 12,
          borderWidth: 1,
          borderRadius: 8,
        }}
      />
      <TextInput
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
        style={{
          marginBottom: 12,
          padding: 12,
          borderWidth: 1,
          borderRadius: 8,
        }}
      />
      {loading ? (
        <ActivityIndicator size="large" color="#0000ff" />
      ) : (
        <Button title="Login" onPress={handleLogin} />
      )}
    </View>
  );
}
```

#### 2. Create Registration Screen

Create a new file ```RegisterScreen.js```:

```javascript
import React, { useState } from "react";
import {
  View,
  TextInput,
  Button,
  Alert,
  ActivityIndicator,
} from "react-native";
import { oryApi, initFlow } from "../oryClient";
import axios from "axios";

export default function RegisterScreen() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [loading, setLoading] = useState(false);

  const handleRegister = async () => {
    setLoading(true);

    try {
      // Step 1: initialize registration flow
      const flow = await initFlow("registration");
      let actionUrl = flow.ui?.action;
      if (!actionUrl) throw new Error("No action URL found");

      // Step 2: submit registration data (traits.email + password)
      const { data } = await oryApi.post(actionUrl, {
        method: "password",
        traits: { email },
        password,
      });

      Alert.alert("✅ Success", "Registration completed!");
      // TODO: Navigate back to login or main screen
    } catch (err: unknown) {
      if (axios.isAxiosError(err)) {
        const msg =
          err.response?.data?.ui?.messages?.map((m: any) => m.text).join("\n") ||
          err.response?.data?.error?.reason ||
          err.message;
        Alert.alert("Registration failed", msg);
      } else if (err instanceof Error) {
        Alert.alert("Registration failed", err.message);
      } else {
        Alert.alert("Registration failed", "Unknown error");
      }
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={{ padding: 16, flex: 1, justifyContent: "center" }}>
      <TextInput
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
        style={{
          marginBottom: 12,
          padding: 12,
          borderWidth: 1,
          borderRadius: 8,
        }}
      />
      <TextInput
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
        style={{
          marginBottom: 12,
          padding: 12,
          borderWidth: 1,
          borderRadius: 8,
        }}
      />
      {loading ? (
        <ActivityIndicator size="large" color="#0000ff" />
      ) : (
        <Button title="Register" onPress={handleRegister} />
      )}
    </View>
  );
}
```
#### 3. Notes

This example shows basic screens without styling, you can style later using your preferred approach.

Make sure your Kratos server is running locally before testing these screens.

In production, always handle errors, sessions, and security properly.

### Step 4 — Handle Sessions and Logout

After logging in, you need to manage user sessions and provide a way to log out.

#### 1. Check User Session

Create a new file `session.js`:

```javascript
import axios from "axios";
import { oryApi } from "./oryClient";

/**
 * Get the current authenticated session from Ory Kratos.
 * Uses the X-Session-Token header (set automatically by oryClient interceptor).
*/
export async function getSession() {
  try {
    const { data } = await oryApi.get("/sessions/whoami");
    return data; // contains identity, traits, etc.
  } catch (err) {
    if (axios.isAxiosError(err)) {
      console.log("whoami error:", err.response?.status, err.response?.data);
      return null;
    }
    console.error("Error in getSession:", err);
    return null;
  }
}
```

#### 2. Implement Logout

Add a logout function to oryClient.js or a separate file:

```javascript
import axios from "axios";
import { oryApi } from "./oryClient";
import { clearSessionToken, getSessionTokenSync } from "./authToken";

export async function logout() {
  try {
    const token = getSessionTokenSync();
    if (token) {
      // API logout: send session_token in the request body
      await oryApi.post("/self-service/logout/api", { session_token: token });
    }
  } catch (err) {
    if (axios.isAxiosError(err)) {
      console.error("Logout failed:", err.response?.data || err.message);
    } else {
      console.error("Logout failed:", err);
    }
  } finally {
    await clearSessionToken();
  }
}
```

#### 3. Integrate in Screens

You can call ```getSession()``` on app start or screen mount to check if the user is logged in.

Add a Logout button in your main app screen:

```
import { Button } from "react-native";
import { logout } from "./oryClient";

<Button title="Logout" onPress={logout} />
```

#### 4. Notes

These are basic examples to demonstrate session handling.

In production, consider secure storage for tokens, proper error handling, and refreshing sessions.

Make sure your Kratos server is running when testing session endpoints.

### Step 5 — Test the Authentication Flow

Now that you have implemented login, registration, session handling, and logout, let's test the full authentication flow.

#### 1. Start your app

Make sure your React Native app is running (Expo):

```bash
npm start
or
expo start
```

#### 2. Register a new user

1. Open the **Registration screen**.

2. Enter an email and password.

3. Submit the form.

4. You should see a success alert with the registration response.

#### 3. Login with the new user

1. Open the **Login screen**.

2. Enter the same credentials.

3. Submit the form.

4. You should see a success alert and session information.

#### 4. Check session

Call the ```getSession()``` function (or navigate to a protected screen) to verify the user is authenticated.

```
const session = await getSession();
console.log(session);
```

#### 5. Logout

Press the **Logout button**.
Call ```getSession()``` again, it should return null or indicate the user is logged out.

This step ensures your app works end-to-end with Ory Kratos.
Test each screen and flow to confirm everything is connected correctly.
Always keep your Kratos server running during testing.

### Step 6 — Optional Enhancements / Next Steps

After completing the basic authentication flow, consider adding the following enhancements:

1. **Form Validation**
   - Ensure that email and password fields have proper validation before sending requests.
   - Show inline error messages for a better UX.

2. **Secure Storage**
   - Store session tokens securely using libraries like `react-native-keychain` or `SecureStore`.

3. **Error Handling**
   - Handle API errors gracefully, including network failures and invalid credentials.
   - Display user-friendly messages instead of raw API errors.

4. **Styling and UX**
   - Style the login and registration screens with your preferred design system or CSS-in-JS library.
   - Add loading indicators while API requests are in progress.

5. **Deployment Considerations**
   - Replace `localhost` with your production Ory Kratos server URL.
   - Ensure HTTPS is used for all requests.
   - Configure CORS and secure headers if needed.

:::tip
These enhancements are optional but recommended for production applications.  
They improve security, reliability, and the overall developer and user experience.
:::
