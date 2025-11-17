# FRONT-END-

## JWT Token Investigation

### What are JWT Tokens?

**JWT (JSON Web Token)** is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed.

### JWT Structure

A JWT consists of three parts separated by dots (`.`):

```
header.payload.signature
```

#### 1. Header
The header typically consists of two parts:
- **typ**: The type of token (JWT)
- **alg**: The signing algorithm being used (e.g., HMAC SHA256, RSA)

Example:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

#### 2. Payload
The payload contains the claims, which are statements about an entity (typically, the user) and additional data. There are three types of claims:
- **Registered claims**: Predefined claims like `iss` (issuer), `exp` (expiration time), `sub` (subject), `aud` (audience)
- **Public claims**: Custom claims that can be defined at will
- **Private claims**: Custom claims created to share information between parties

Example:
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622
}
```

#### 3. Signature
The signature is used to verify that the sender of the JWT is who it says it is and to ensure that the message wasn't changed along the way. It's created by:
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### Main Characteristics of JWT

1. **Self-Contained**: JWTs contain all the necessary information about the user, reducing the need to query the database multiple times.

2. **Compact**: Due to their small size, JWTs can be sent through URL parameters, POST parameters, or inside HTTP headers, making transmission fast.

3. **Stateless**: The server doesn't need to keep a record of tokens. Each token contains all the information needed to verify its validity.

4. **Secure**: JWTs are digitally signed using either a secret (HMAC) or a public/private key pair (RSA or ECDSA), ensuring data integrity.

5. **Cross-Domain/CORS**: JWTs work well with Cross-Origin Resource Sharing (CORS) since they are transmitted in headers rather than cookies.

6. **Standardized**: Being an open standard (RFC 7519), JWTs are supported by multiple programming languages and platforms.

### Why JWT is Needed in the Front End

#### 1. **Authentication**
- After a user logs in, the server generates a JWT and sends it to the frontend
- The frontend stores the token and includes it in subsequent requests to authenticate the user
- This eliminates the need for the server to maintain session state

#### 2. **Authorization**
- JWTs contain user permissions and roles in the payload
- The frontend can decode the token (without verifying the signature) to check user permissions
- This enables client-side route protection and conditional UI rendering before making API calls

#### 3. **Single Sign-On (SSO)**
- JWTs enable seamless authentication across multiple applications/domains
- Once authenticated, users can access multiple services without re-logging in

#### 4. **Information Exchange**
- JWTs can securely transmit data between the frontend and backend
- The signature ensures that the data hasn't been tampered with

#### 5. **Stateless Architecture**
- Supports RESTful API principles by being stateless
- Reduces server memory usage as session information isn't stored server-side
- Improves scalability as any server can validate the token

### JWT Storage Options in Frontend

#### 1. **LocalStorage**
```javascript
// Storing token
localStorage.setItem('token', jwtToken);

// Retrieving token
const token = localStorage.getItem('token');

// Removing token
localStorage.removeItem('token');
```

**Pros:**
- Persists across browser sessions
- Simple API
- Large storage capacity

**Cons:**
- Vulnerable to XSS (Cross-Site Scripting) attacks
- Accessible by any JavaScript code on the page

#### 2. **SessionStorage**
```javascript
// Storing token
sessionStorage.setItem('token', jwtToken);

// Retrieving token
const token = sessionStorage.getItem('token');
```

**Pros:**
- Cleared when tab/browser closes
- Not vulnerable to CSRF attacks

**Cons:**
- Still vulnerable to XSS attacks
- Doesn't persist across browser sessions

#### 3. **Memory (JavaScript Variables)**
```javascript
// Storing in a module-scoped variable
let authToken = null;

export const setToken = (token) => { authToken = token; };
export const getToken = () => authToken;
```

**Pros:**
- More secure against XSS if implemented correctly
- Token doesn't persist beyond page refresh

**Cons:**
- User needs to re-authenticate on page refresh
- More complex to implement

#### 4. **HttpOnly Cookies** (Requires backend support)
```javascript
// Token is set by server with HttpOnly flag
// Frontend automatically sends it with requests
```

**Pros:**
- Not accessible via JavaScript (immune to XSS)
- Automatically sent with requests

**Cons:**
- Vulnerable to CSRF attacks (requires CSRF tokens)
- Requires backend configuration

### Security Best Practices

1. **Use HTTPS**: Always transmit JWTs over HTTPS to prevent interception

2. **Set Expiration Times**: Use short expiration times (`exp` claim) to limit token validity
   ```json
   {
     "exp": 1516242622  // Unix timestamp
   }
   ```

3. **Implement Token Refresh**: Use refresh tokens to obtain new access tokens without requiring re-authentication
   ```javascript
   // Access token: short-lived (15 minutes)
   // Refresh token: long-lived (7 days), stored more securely
   ```

4. **Validate on Server**: Always validate JWTs on the server side; never trust client-side validation alone

5. **Don't Store Sensitive Data**: Keep minimal information in the payload as it can be decoded easily
   ```javascript
   // Bad: Storing sensitive data
   { "password": "secret123" }
   
   // Good: Storing only necessary identifiers
   { "userId": "123", "role": "user" }
   ```

6. **Implement Logout**: Maintain a token blacklist on the server or use short expiration times

7. **Sanitize Inputs**: Prevent XSS attacks by sanitizing all user inputs

8. **Use Strong Signing Keys**: Use cryptographically strong keys for signing tokens

### Practical Implementation Example

```javascript
// Authentication Service
class AuthService {
  async login(email, password) {
    const response = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    
    const { token } = await response.json();
    localStorage.setItem('token', token);
    return token;
  }
  
  logout() {
    localStorage.removeItem('token');
  }
  
  getToken() {
    return localStorage.getItem('token');
  }
  
  isAuthenticated() {
    const token = this.getToken();
    if (!token) return false;
    
    // Decode token to check expiration
    try {
      const payload = JSON.parse(atob(token.split('.')[1]));
      return payload.exp * 1000 > Date.now();
    } catch (e) {
      return false;
    }
  }
}

// API Request with JWT
async function fetchProtectedData() {
  const token = localStorage.getItem('token');
  
  const response = await fetch('/api/protected', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  
  return await response.json();
}

// Axios Interceptor Example
import axios from 'axios';

axios.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Handle token expiration
axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Token expired - redirect to login or refresh token
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### Common Use Cases in Frontend

1. **Protected Routes**: Restrict access to certain pages based on authentication
   ```javascript
   // React Router example
   function ProtectedRoute({ children }) {
     const isAuthenticated = authService.isAuthenticated();
     return isAuthenticated ? children : <Navigate to="/login" />;
   }
   ```

2. **Conditional Rendering**: Show/hide UI elements based on user roles
   ```javascript
   function AdminPanel() {
     const token = getToken();
     const payload = parseJWT(token);
     
     if (payload.role !== 'admin') {
       return <div>Access Denied</div>;
     }
     
     return <div>Admin Content</div>;
   }
   ```

3. **API Authorization**: Include tokens in API requests
   ```javascript
   fetch('/api/data', {
     headers: {
       'Authorization': `Bearer ${token}`
     }
   })
   ```

### Advantages of JWT in Frontend Development

1. **Reduced Server Load**: No need to store session data on the server
2. **Better Scalability**: Tokens can be validated by any server in a distributed system
3. **Mobile Friendly**: Works well with mobile applications and native apps
4. **Microservices Architecture**: Easy to share authentication across multiple services
5. **Developer Experience**: Simple to implement and debug
6. **Cross-Platform**: Works across web, mobile, and desktop applications

### Disadvantages and Considerations

1. **Token Size**: JWTs can be larger than session IDs, increasing bandwidth usage
2. **Cannot Revoke Easily**: Once issued, tokens are valid until expiration (unless using blacklist)
3. **Payload Not Encrypted**: Information in the payload is only base64 encoded, not encrypted
4. **Storage Security**: Requires careful consideration of where to store tokens in the frontend

### Conclusion

JWT tokens are essential for modern frontend development, especially for Single Page Applications (SPAs) and mobile apps. They provide a stateless, scalable, and secure way to handle authentication and authorization. However, developers must implement proper security measures and understand the trade-offs between different storage mechanisms to ensure the safety of user data.

### Additional Resources

- [JWT.io](https://jwt.io/) - Official JWT website with debugger and library list
- [RFC 7519](https://tools.ietf.org/html/rfc7519) - JWT specification
- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html) - Security best practices