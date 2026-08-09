**CityDrop — Frontend / Backend API Contract**  
v1 — written so both sides can build in parallel against the same agreement  
This defines every endpoint the frontend skeleton needs, derived from citydrop-web-classic's utils.js (the USE\_MOCK toggle's mock implementations) and from CityDrop\_Interactive\_Prototype\_v5.html's behavior. Whoever owns the backend builds Controllers matching this exactly; whoever owns the frontend flips USE\_MOCK to false in utils.js once the real endpoints exist. If either side needs to deviate, flag it here and update this doc — don't let it drift silently.

Conventions

**Base URL (dev):** Frontend code calls relative paths like /api/orders. Set "proxy": "http://localhost:8080" in citydrop-web-classic's package.json, so the browser never makes a cross-origin request in development — this sidesteps CORS entirely for local dev.

**Base URL (prod / demo deploy):** Set the real backend URL via a Create React App environment variable (a REACT\_APP\_ prefixed variable in .env) once there is somewhere to deploy it; not needed yet.

**Auth:** Spring Security

**Error format:** Every error response is a JSON object shaped { "error": "\<human-readable message\>" }, with an appropriate HTTP status code (400/401/404/409). The frontend reads err.error directly for the modal/toast text — no parsing needed.

**Field naming:** camelCase everywhere in JSON, matching the frontend's existing JS conventions. Spring's Jackson serializer does this automatically from Java field names.

The Order object

Every endpoint below that returns an order uses this same shape. citydrop-web-classic's utils.js mock implementation already returns orders in exactly this shape (a single status enum, not separate statusIndex/cancelled/queued fields) — no adjustment needed there once the backend matches it.

{  
  "id": int,  
  "destination": String,  
  "weightLb": int,  
  "mode": "ROBOT" || "DRONE"  
  "price": double,  
  "stationId": int,  
  "status": “PENDING\_DROPOFF” || “AT\_STATION” || “BEFORE\_HALF\_WAY” || “HALF\_WAY” || “MORE\_THAN\_HALF\_WAY” || "DELIVERED"  
  "createdAt": String  
}

Auth endpoints

### **POST** **/register**

### **Purpose:** Create a new account

### Request body:

{  
  "username": String  
  "password": String  
}

### Success — 201

### Errors –– 409 this username is already taken

### 

### **POST /login**

### **Purpose:** Log in

### Request body:

{  
  "username": String  
  "password": String  
}

### Success — 204

### Errors –– 401 error on login

### 

### **POST /logout**

### **Purpose:** Log out

### Request body: none

### Success — 204

### 

Delivery Option endpoints

GET **/delivery-options**

### **Purpose**: get all six delivery options

### Query Params: \[destStreet: String, destCity: String, destState: String, destZip: string, packageWeight: double\]

### Success –– 200

### Error –– 422 address cannot be geocoded

### 

Order endpoints

### **POST /order**

### **Purpose:** submit order

### Request body:

{  
  "destination": String,  
  "weightLb": double,  
  "preferStationId": int,  
  "preferMode": “ROBOT” || “DRONE”,   
}

### Response body: Order Object

### Success — 201:

### Error — 409 not enough vehicles

### 

### **GET /order**

### **Purpose:** List the logged-in user’s order IDs

### Response body:

{  
  “active” : array\[{“orderId” : int}\]  
  “completed” : array\[{“orderId” : int}\]  
}

### Success — 200

### 

### **GET /order/{id}**

### **Purpose:** Fetch a single order’s detail

### Response body: an Order object

### Success — 200

### Error — 404 order not found