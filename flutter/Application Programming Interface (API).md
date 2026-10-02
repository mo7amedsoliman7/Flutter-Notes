
# HTTP Request & Response Anatomy

> [!NOTE]
> **Request → Call flow** (client to server) | **Response → Data flow** (server back to client)

---

## 🔵 Given URL

```
https://www.app.com/users?name=soliman
```

Let's break every part of this down.

---

## 🧩 URL Anatomy

| Part | Value | Role |
|---|---|---|
| **Protocol** | `https://` | Secure transport layer |
| **Base URL** | `www.app.com` | The API's main domain |
| **Endpoint** | `/users` | The specific resource being called |
| **Query params** | `?name=soliman` | Filters / options passed in the URL |

---

## ✅ 1. Base URL — *"Where to send the request"*

```
https://www.app.com
```

The root domain of the API. All endpoints are appended to this. Usually stored as a constant so it can be swapped between environments (dev / staging / production).

---

## ✅ 2. Endpoint — *"Which resource to call"*

```
/users
```

The path that identifies the specific resource or action on the server.

| Endpoint | Typical meaning |
|---|---|
| `GET /users` | List users |
| `GET /users/42` | Get user with ID 42 |
| `POST /users` | Create a new user |
| `PATCH /users/42` | Partially update user 42 |
| `DELETE /users/42` | Delete user 42 |

---

## ✅ 3. Headers — *"Metadata about the request"*

Not part of the URL — sent alongside the request to tell the server *how* to handle it.

```http
Accept: application/json
Content-Type: application/json
Accept-Language: en-US
Authorization: Bearer <token>
User-Agent: MyApp/1.0 (iOS 17)
```

| Header | Purpose |
|---|---|
| `Content-Type` | Format of the **request** body |
| `Accept` | Format the client expects in the **response** |
| `Authorization` | Auth token — identifies the caller |
| `Accept-Language` | Preferred response language |
| `User-Agent` | Client platform / version info |

---

## ✅ 4. Body — *"Data sent with the request"*

Only used in **POST / PUT / PATCH** requests. A `GET` request has **no body**.

```json
{
  "name": "soliman",
  "age": 22
}
```

> [!IMPORTANT]
> When sending a body, always set `Content-Type: application/json` so the server knows how to parse it.

---

## ✅ 5. Query Parameters — *"Filters passed in the URL"*

Everything after `?`. Used to filter, sort, or paginate without changing the endpoint.

```
?name=soliman&age=22&limit=10
```

| Key | Value | Purpose |
|---|---|---|
| `name` | `soliman` | Filter by name |
| `age` | `22` | Filter by age |
| `limit` | `10` | Limit result count |

Multiple params are chained with `&`. The first param starts with `?`.

---

## 🔄 Call Flow: Request → Response

### 1️⃣ Client Sends Request

The client assembles and sends:

```
GET https://www.app.com/users?name=soliman
Headers: { Authorization: Bearer <token>, Accept: application/json }
Body:     none (GET request)
```

### 2️⃣ Server Processes

```
1. Parse URL & query params
2. Validate headers & auth token
3. Authenticate / authorize the caller
4. Query the database
5. Serialize result to JSON
6. Attach status code + response headers
```

### 3️⃣ Server Sends Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [
    { "id": 1, "name": "soliman", "age": 22 }
  ]
}
```

| Response part | Example | Purpose |
|---|---|---|
| **Status code** | `200 OK` | Was the request successful? |
| **Response headers** | `Content-Type: application/json` | Metadata about the response |
| **Body** | `{ "data": [...] }` | The actual returned data |

---

## ⭐ Full Example Summary

| Component | Value |
|---|---|
| **Base URL** | `https://www.app.com` |
| **Endpoint** | `/users` |
| **Method** | `GET` |
| **Headers** | `Authorization`, `Accept`, `Accept-Language` |
| **Body** | none (GET request) |
| **Query params** | `name=soliman` |

---

## 🎯 Key Points

1. **Base URL + Endpoint = full URL** — keep the base URL as a constant, swap it per environment.
2. **Headers = metadata** — they describe the request, they are not the data.
3. **Body only on POST / PUT / PATCH** — always pair with `Content-Type`.
4. **Query params = URL filters** — visible in the URL, suitable for GET; don't put secrets here.
5. **Status code is the first thing to check** — `2xx` success, `4xx` client error, `5xx` server error.
