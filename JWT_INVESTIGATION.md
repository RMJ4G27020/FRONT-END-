# JWT Tokens Investigation

## Table of Contents
1. [Introduction](#introduction)
2. [What is JWT?](#what-is-jwt)
3. [JWT Structure](#jwt-structure)
4. [Main Characteristics](#main-characteristics)
5. [Why JWT is Needed in Front-End](#why-jwt-is-needed-in-front-end)
6. [Implementation in Front-End](#implementation-in-front-end)
7. [Best Practices](#best-practices)
8. [Security Considerations](#security-considerations)
9. [Common Use Cases](#common-use-cases)
10. [Conclusion](#conclusion)

---

## Introduction

JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed.

---

## What is JWT?

JWT is a string that consists of three parts separated by dots (`.`), which are:
- **Header**
- **Payload**
- **Signature**

A JWT typically looks like: `xxxxx.yyyyy.zzzzz`

---

## JWT Structure

### 1. Header
The header typically consists of two parts:
- The type of token (JWT)
- The signing algorithm being used (e.g., HMAC SHA256 or RSA)

**Example:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### 2. Payload
The payload contains the claims. Claims are statements about an entity (typically, the user) and additional data. There are three types of claims:

- **Registered claims**: Predefined claims like `iss` (issuer), `exp` (expiration time), `sub` (subject), `aud` (audience)
- **Public claims**: Custom claims that can be defined at will
- **Private claims**: Custom claims created to share information between parties

**Example:**
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "email": "john.doe@example.com",
  "role": "admin",
  "iat": 1516239022,
  "exp": 1516242622
}
```

### 3. Signature
The signature is created by taking the encoded header, encoded payload, a secret, and the algorithm specified in the header.

**Example:**
```javascript
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

---

## Main Characteristics

### 1. **Compact**
- JWTs are small in size, making them easy to send through URLs, POST parameters, or HTTP headers
- Fast transmission due to their compact nature

### 2. **Self-Contained**
- The payload contains all the required information about the user
- No need to query the database multiple times
- Reduces database load

### 3. **Stateless**
- Server doesn't need to keep track of which tokens are active
- Enables horizontal scaling
- No session storage required on the server

### 4. **Secure**
- Digitally signed using a secret (HMAC) or public/private key pair (RSA/ECDSA)
- Can be encrypted for additional security
- Prevents tampering with the token data

### 5. **Cross-Domain / CORS Compatible**
- Works across different domains
- Can be used for Single Sign-On (SSO)
- Ideal for microservices architecture

### 6. **Standardized**
- Based on RFC 7519 standard
- Widely supported across different platforms and languages
- Interoperable between different systems

### 7. **Expirable**
- Tokens can have expiration times (`exp` claim)
- Automatic invalidation after expiration
- Supports refresh token mechanism

---

## Why JWT is Needed in Front-End

### 1. **Authentication & Authorization**

**Problem Without JWT:**
- Traditional session-based authentication requires server-side session storage
- Difficult to scale across multiple servers
- CORS issues with cookies across domains

**Solution With JWT:**
- Token-based authentication eliminates server-side session storage
- Easy to scale horizontally
- Works seamlessly across different domains

**Front-End Benefits:**
```javascript
// Store token after login
localStorage.setItem('token', jwtToken);

// Use token for subsequent requests
const token = localStorage.getItem('token');
fetch('/api/protected-route', {
  headers: {
    'Authorization': `Bearer ${token}`
  }
});
```

### 2. **Single Page Applications (SPAs)**

**Why JWTs are Essential:**
- SPAs need to maintain authentication state across page navigations
- Traditional session cookies may not work well with modern SPAs
- JWTs provide a clean way to handle authentication in client-side routing

**Example:**
```javascript
// React Router with JWT
const PrivateRoute = ({ component: Component, ...rest }) => {
  const token = localStorage.getItem('token');
  
  return (
    <Route
      {...rest}
      render={props =>
        token ? (
          <Component {...props} />
        ) : (
          <Redirect to="/login" />
        )
      }
    />
  );
};
```

### 3. **API Communication**

**Why Important:**
- Modern front-ends communicate with RESTful APIs
- JWTs provide a standardized way to authenticate API requests
- Stateless nature makes it perfect for API authentication

**Example:**
```javascript
// Axios interceptor to add JWT to all requests
axios.interceptors.request.use(
  config => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers['Authorization'] = `Bearer ${token}`;
    }
    return config;
  },
  error => {
    return Promise.reject(error);
  }
);
```

### 4. **Mobile & Desktop Applications**

**Why JWTs are Ideal:**
- Same token can be used across web, mobile, and desktop applications
- Consistent authentication mechanism across platforms
- No dependency on cookies (which don't work well in mobile apps)

### 5. **Microservices Architecture**

**Front-End Perspective:**
- Single token can authenticate across multiple backend services
- No need to manage multiple authentication mechanisms
- Simplifies front-end authentication logic

### 6. **Offline Functionality**

**Benefits:**
- JWT can be stored locally and used when app comes back online
- Contains user information that can be accessed offline
- Enables Progressive Web Apps (PWAs) functionality

### 7. **Third-Party API Integration**

**Use Case:**
- Front-end can use JWT to authenticate with third-party services
- OAuth 2.0 implementations often use JWT
- Simplifies integration with external services

---

## Implementation in Front-End

### Basic Login Flow

```javascript
// 1. Login Function
async function login(email, password) {
  try {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email, password }),
    });
    
    const data = await response.json();
    
    if (data.token) {
      // Store JWT token
      localStorage.setItem('token', data.token);
      
      // Optionally decode and store user info
      const userInfo = parseJwt(data.token);
      localStorage.setItem('user', JSON.stringify(userInfo));
      
      return true;
    }
  } catch (error) {
    console.error('Login failed:', error);
    return false;
  }
}

// 2. Parse JWT Token (decode payload)
function parseJwt(token) {
  try {
    const base64Url = token.split('.')[1];
    const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
    const jsonPayload = decodeURIComponent(
      atob(base64)
        .split('')
        .map(c => '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2))
        .join('')
    );
    return JSON.parse(jsonPayload);
  } catch (error) {
    return null;
  }
}

// 3. Check Token Expiration
function isTokenExpired(token) {
  const decoded = parseJwt(token);
  if (!decoded || !decoded.exp) return true;
  
  const currentTime = Date.now() / 1000;
  return decoded.exp < currentTime;
}

// 4. Logout Function
function logout() {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
  window.location.href = '/login';
}

// 5. Get Current User
function getCurrentUser() {
  const token = localStorage.getItem('token');
  if (!token || isTokenExpired(token)) {
    logout();
    return null;
  }
  
  return parseJwt(token);
}

// 6. Protected API Call
async function makeAuthenticatedRequest(url, options = {}) {
  const token = localStorage.getItem('token');
  
  if (!token || isTokenExpired(token)) {
    logout();
    throw new Error('No valid token');
  }
  
  const headers = {
    ...options.headers,
    'Authorization': `Bearer ${token}`,
  };
  
  try {
    const response = await fetch(url, { ...options, headers });
    
    if (response.status === 401) {
      // Token invalid or expired
      logout();
      throw new Error('Unauthorized');
    }
    
    return response;
  } catch (error) {
    throw error;
  }
}
```

### React Example with Context

```javascript
// AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Check for existing token on mount
    const token = localStorage.getItem('token');
    if (token && !isTokenExpired(token)) {
      setUser(parseJwt(token));
    }
    setLoading(false);
  }, []);

  const login = async (email, password) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });
    
    const data = await response.json();
    
    if (data.token) {
      localStorage.setItem('token', data.token);
      setUser(parseJwt(data.token));
      return true;
    }
    return false;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Token Refresh Implementation

```javascript
// Token refresh logic
let isRefreshing = false;
let refreshSubscribers = [];

function subscribeTokenRefresh(callback) {
  refreshSubscribers.push(callback);
}

function onTokenRefreshed(newToken) {
  refreshSubscribers.forEach(callback => callback(newToken));
  refreshSubscribers = [];
}

async function refreshToken() {
  if (isRefreshing) {
    return new Promise(resolve => {
      subscribeTokenRefresh(token => {
        resolve(token);
      });
    });
  }

  isRefreshing = true;

  try {
    const refreshToken = localStorage.getItem('refreshToken');
    const response = await fetch('/api/auth/refresh', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken }),
    });

    const data = await response.json();

    if (data.token) {
      localStorage.setItem('token', data.token);
      onTokenRefreshed(data.token);
      isRefreshing = false;
      return data.token;
    }
  } catch (error) {
    isRefreshing = false;
    logout();
    throw error;
  }
}

// Axios interceptor with refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;

    if (error.response.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const newToken = await refreshToken();
        originalRequest.headers['Authorization'] = `Bearer ${newToken}`;
        return axios(originalRequest);
      } catch (refreshError) {
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

---

## Best Practices

### 1. **Token Storage**

**Recommended:**
- Use `localStorage` for web applications (simple but has XSS risks)
- Use `sessionStorage` for added security (cleared on tab close)
- Use secure, httpOnly cookies for maximum security (requires backend support)
- Use memory storage for highly sensitive applications (lost on refresh)

**Example of secure storage approach:**
```javascript
// Memory storage (more secure but lost on refresh)
class TokenStorage {
  constructor() {
    this.token = null;
  }
  
  setToken(token) {
    this.token = token;
  }
  
  getToken() {
    return this.token;
  }
  
  clearToken() {
    this.token = null;
  }
}

const tokenStorage = new TokenStorage();
```

### 2. **Token Validation**

Always validate tokens on the front-end:
```javascript
function validateToken(token) {
  if (!token) return false;
  
  try {
    const decoded = parseJwt(token);
    
    // Check expiration
    if (decoded.exp && decoded.exp < Date.now() / 1000) {
      return false;
    }
    
    // Check issuer if needed
    if (decoded.iss && decoded.iss !== 'your-expected-issuer') {
      return false;
    }
    
    return true;
  } catch (error) {
    return false;
  }
}
```

### 3. **Token Refresh Strategy**

Implement automatic token refresh before expiration:
```javascript
function scheduleTokenRefresh(token) {
  const decoded = parseJwt(token);
  if (!decoded || !decoded.exp) return;
  
  const currentTime = Date.now() / 1000;
  const expiresIn = decoded.exp - currentTime;
  
  // Refresh 5 minutes before expiration
  const refreshTime = (expiresIn - 300) * 1000;
  
  if (refreshTime > 0) {
    setTimeout(() => {
      refreshToken();
    }, refreshTime);
  }
}
```

### 4. **Error Handling**

Implement proper error handling for JWT operations:
```javascript
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // Token expired or invalid
      logout();
      window.location.href = '/login';
    } else if (error.response?.status === 403) {
      // Insufficient permissions
      alert('You do not have permission to access this resource');
    }
    return Promise.reject(error);
  }
);
```

### 5. **Don't Store Sensitive Data**

Never put sensitive information in JWT payload:
```javascript
// ❌ BAD - Don't do this
const payload = {
  userId: 123,
  password: 'user-password',      // Never!
  creditCard: '1234-5678-9012',   // Never!
  ssn: '123-45-6789'              // Never!
};

// ✅ GOOD - Do this
const payload = {
  userId: 123,
  email: 'user@example.com',
  role: 'admin',
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + (60 * 60) // 1 hour
};
```

### 6. **Use HTTPS**

Always use HTTPS to transmit JWT tokens:
- Prevents man-in-the-middle attacks
- Protects token from being intercepted
- Essential for production environments

### 7. **Set Appropriate Expiration Times**

```javascript
// Short-lived access tokens
const accessTokenExpiry = 15 * 60; // 15 minutes

// Long-lived refresh tokens
const refreshTokenExpiry = 7 * 24 * 60 * 60; // 7 days
```

---

## Security Considerations

### 1. **XSS (Cross-Site Scripting) Attacks**

**Risk:**
- If JWT is stored in localStorage, it can be accessed by malicious scripts
- XSS attacks can steal the token

**Mitigation:**
```javascript
// Sanitize user inputs
function sanitizeInput(input) {
  const temp = document.createElement('div');
  temp.textContent = input;
  return temp.innerHTML;
}

// Use Content Security Policy (CSP)
// Add to HTML:
// <meta http-equiv="Content-Security-Policy" content="default-src 'self'">
```

### 2. **CSRF (Cross-Site Request Forgery)**

**Risk:**
- Less of a concern with JWT compared to cookies
- Still possible if JWT is stored in cookies

**Mitigation:**
```javascript
// Use custom header for JWT (not vulnerable to CSRF)
fetch('/api/data', {
  headers: {
    'Authorization': `Bearer ${token}`,
    'X-Requested-With': 'XMLHttpRequest'
  }
});
```

### 3. **Token Leakage**

**Prevention:**
```javascript
// Don't log tokens
console.log('Token:', token); // ❌ Never do this

// Don't pass tokens in URL
const url = `/api/data?token=${token}`; // ❌ Never do this

// Use Authorization header
fetch('/api/data', {
  headers: {
    'Authorization': `Bearer ${token}` // ✅ Correct way
  }
});
```

### 4. **Token Revocation**

**Strategy:**
```javascript
// Maintain a token blacklist on the server
// Front-end should handle revocation gracefully
function handleTokenRevocation() {
  // Clear local storage
  localStorage.removeItem('token');
  localStorage.removeItem('refreshToken');
  
  // Redirect to login
  window.location.href = '/login';
}
```

### 5. **JWT Library Usage**

Always use well-maintained libraries for JWT handling:
```javascript
// For decoding (client-side verification)
// Use: jsonwebtoken, jwt-decode

import jwtDecode from 'jwt-decode';

const decoded = jwtDecode(token);
console.log(decoded);
```

**Note:** Never verify JWT signatures on the client-side with the secret key. Signature verification should only happen on the server.

---

## Common Use Cases

### 1. **User Authentication Flow**

```javascript
// Complete authentication flow
class AuthService {
  async login(credentials) {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    
    const { token, refreshToken } = await response.json();
    
    localStorage.setItem('token', token);
    localStorage.setItem('refreshToken', refreshToken);
    
    return this.getUserFromToken(token);
  }
  
  getUserFromToken(token) {
    return parseJwt(token);
  }
  
  isAuthenticated() {
    const token = localStorage.getItem('token');
    return token && !isTokenExpired(token);
  }
  
  logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('refreshToken');
  }
}
```

### 2. **Role-Based Access Control**

```javascript
// Check user permissions from JWT
function hasPermission(requiredRole) {
  const token = localStorage.getItem('token');
  if (!token) return false;
  
  const user = parseJwt(token);
  return user.role === requiredRole;
}

// React component with role check
const AdminPanel = () => {
  if (!hasPermission('admin')) {
    return <div>Access Denied</div>;
  }
  
  return <div>Admin Content</div>;
};
```

### 3. **Multi-Tab Synchronization**

```javascript
// Sync authentication across tabs
window.addEventListener('storage', (event) => {
  if (event.key === 'token') {
    if (event.newValue === null) {
      // Token removed in another tab - logout
      window.location.href = '/login';
    } else {
      // Token updated in another tab - reload user
      const user = parseJwt(event.newValue);
      updateUserState(user);
    }
  }
});
```

### 4. **WebSocket Authentication**

```javascript
// Authenticate WebSocket connection with JWT
const token = localStorage.getItem('token');
const socket = new WebSocket(`wss://api.example.com?token=${token}`);

// Or send token after connection
socket.onopen = () => {
  socket.send(JSON.stringify({
    type: 'auth',
    token: localStorage.getItem('token')
  }));
};
```

### 5. **File Download with Authentication**

```javascript
// Download protected file
async function downloadFile(fileId) {
  const token = localStorage.getItem('token');
  
  const response = await fetch(`/api/files/${fileId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  
  const blob = await response.blob();
  const url = window.URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'file.pdf';
  a.click();
}
```

---

## Conclusion

### Key Takeaways

1. **JWT is Essential for Modern Front-End Development**
   - Enables stateless authentication
   - Perfect for SPAs and mobile applications
   - Scales well with microservices architecture

2. **Main Characteristics That Matter**
   - Compact and efficient
   - Self-contained with all necessary information
   - Secure through digital signatures
   - Cross-platform compatibility

3. **Front-End Benefits**
   - Simplified authentication logic
   - Better user experience with seamless authentication
   - Enables offline functionality
   - Works across different domains

4. **Critical Considerations**
   - Security is paramount (use HTTPS, sanitize inputs)
   - Proper token storage strategy is essential
   - Implement token refresh mechanism
   - Never store sensitive data in JWT payload

5. **Best Practices to Follow**
   - Validate tokens before use
   - Set appropriate expiration times
   - Implement proper error handling
   - Use established JWT libraries
   - Keep tokens secure and never expose them

### Why Front-End Developers Need to Understand JWT

- **Authentication is a Core Feature**: Almost every modern application requires user authentication
- **API-First Development**: Modern front-ends consume APIs that use JWT for authentication
- **Better User Experience**: JWT enables seamless authentication without page reloads
- **Career Essential**: Understanding JWT is a fundamental skill for front-end developers
- **Security Awareness**: Knowing how JWT works helps build secure applications

### Next Steps for Implementation

1. Choose appropriate JWT library for your framework
2. Implement secure token storage strategy
3. Create authentication context/service
4. Add automatic token refresh mechanism
5. Implement proper error handling
6. Add role-based access control if needed
7. Test authentication flow thoroughly
8. Monitor and log authentication events

---

## Additional Resources

### Libraries
- **jwt-decode**: Decode JWT tokens on the client-side
- **jsonwebtoken**: Full JWT implementation (primarily for Node.js)
- **axios**: HTTP client with interceptor support for JWT

### Tools
- **JWT.io**: Debug and decode JWT tokens
- **Postman**: Test API endpoints with JWT authentication
- **Chrome DevTools**: Monitor network requests with JWT headers

### Further Reading
- [RFC 7519 - JWT Specification](https://tools.ietf.org/html/rfc7519)
- [OAuth 2.0 with JWT](https://oauth.net/2/jwt/)
- [OWASP JWT Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)

---

**Document Version**: 1.0  
**Last Updated**: November 2025  
**Author**: Front-End Development Team
