### Request => call flow
### Response => data flow

## 🔵 **Given URL

```
https://www.app.com/users?name=soliman
```

---
# ✅ **1. Base URL**

This is the main domain of the API:

```
https://www.app.com
```

---
# ✅ **2. Endpoint

This is the API path you are calling:

```
/users
```

---
# ✅ **3. Headers**
Headers are **metadata** sent with the request. Common examples:

```
Accept: application/json
Content-Type: application/json![[request-response.png]]
Accept-Language: en-US
Authorization: Bearer <token>
User-Agent: ...
```

In your example, you mentioned:
- `Accept-Language`
- Any other headers can be added depending on the API.
    
---
# ✅ **4. Body**

Only used in **POST / PUT / PATCH** requests.  
For a **GET** request like your URL **there is no body**, but if it existed, it would be JSON:

Example (POST):

```json
{
  "user": "soliman",
  "age": 22
}
```

---
# ✅ **5. Query Parameters**

Everything after `?` in the URL.
For your URL:

```
?name=soliman
```

Query parameters key/value:

|Key|Value|
|---|---|
|name|soliman|

If multiple params:
```
?name=soliman&age=22&limit=10
```

---
## ⭐ Final Summary (Your URL)

- **Base URL:** `https://www.app.com`
- **Endpoint:** `/users`
- **Headers:** optional (`accept-language`, `authorization`, etc.)
- **Body:** none (GET request)
- **Query params:** `{ name: "soliman" }`

---
### **1️⃣ Client Sends Request**

Includes:
- Base URL
- Endpoint
- Headers
- Body (if POST)
- Query parameters
### **2️⃣ Server Processes**

The backend:
- Validates parameters
- Authenticates user
- Queries database
- Prepares JSON response
### **3️⃣ Server Sends Response**

Contains:
- Status code
- Response headers
- Response body (data)
--- 


