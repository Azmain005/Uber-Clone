# Uber Backend API Documentation

## User Registration Endpoint

### POST /users/register

Register a new user account.

#### Request Body

The request must include a JSON body with the following structure:

```json
{
  "fullname": {
    "firstname": "string",
    "lastname": "string"
  },
  "email": "string",
  "password": "string"
}
```

#### Field Requirements

| Field                | Type   | Required | Validation                               |
| -------------------- | ------ | -------- | ---------------------------------------- |
| `fullname.firstname` | String | Yes      | Minimum 3 characters                     |
| `fullname.lastname`  | String | No       | Minimum 3 characters (if provided)       |
| `email`              | String | Yes      | Valid email format, minimum 5 characters |
| `password`           | String | Yes      | Minimum 6 characters                     |

#### Example Request

```json
{
  "fullname": {
    "firstname": "John",
    "lastname": "Doe"
  },
  "email": "john.doe@example.com",
  "password": "securepassword123"
}
```

#### Response

##### Success Response (201 Created)

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "_id": "507f1f77bcf86cd799439011",
    "fullname": {
      "firstname": "John",
      "lastname": "Doe"
    },
    "email": "john.doe@example.com"
  }
}
```

##### Error Response (400 Bad Request)

```json
{
  "errors": [
    {
      "msg": "Please provide a valid email address",
      "param": "email",
      "location": "body"
    },
    {
      "msg": "First name must be at least 3 characters long",
      "param": "fullname.firstname",
      "location": "body"
    },
    {
      "msg": "Password must be at least 6 characters long",
      "param": "password",
      "location": "body"
    }
  ]
}
```

#### Status Codes

| Status Code | Description                           |
| ----------- | ------------------------------------- |
| 201         | User successfully created             |
| 400         | Validation error - invalid input data |
| 500         | Server error                          |

#### Notes

- Password is automatically hashed before storage using bcrypt
- JWT token expires in 1 hour
- Email must be unique (duplicate emails will cause an error)
- The response does not include the password field for security reasons

## User Login Endpoint

### POST /users/login

Authenticate an existing user and receive an access token.

#### Request Body

The request must include a JSON body with the following structure:

```json
{
  "email": "string",
  "password": "string"
}
```

#### Field Requirements

| Field      | Type   | Required | Validation         |
| ---------- | ------ | -------- | ------------------ |
| `email`    | String | Yes      | Valid email format |
| `password` | String | Yes      | Cannot be empty    |

#### Example Request

```json
{
  "email": "john.doe@example.com",
  "password": "securepassword123"
}
```

#### Response

##### Success Response (200 OK)

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "_id": "507f1f77bcf86cd799439011",
    "fullname": {
      "firstname": "John",
      "lastname": "Doe"
    },
    "email": "john.doe@example.com"
  }
}
```

##### Error Response (400 Bad Request)

```json
{
  "errors": [
    {
      "msg": "Please provide a valid email address",
      "param": "email",
      "location": "body"
    },
    {
      "msg": "Password cannot be empty",
      "param": "password",
      "location": "body"
    }
  ]
}
```

##### Error Response (401 Unauthorized)

```json
{
  "message": "Invalid email or password"
}
```

#### Status Codes

| Status Code | Description                                 |
| ----------- | ------------------------------------------- |
| 200         | User successfully authenticated             |
| 400         | Validation error - invalid input data       |
| 401         | Authentication failed - invalid credentials |
| 500         | Server error                                |

#### Notes

- Returns the same JWT token format as the registration endpoint
- JWT token expires in 1 hour
- For security reasons, the same error message is returned whether the email doesn't exist or the password is incorrect
- The response does not include the password field for security reasons
- Token is also set as a cookie in the response

## User Profile Endpoint

### GET /users/profile

Retrieve the authenticated user's profile information.

#### Authentication

This endpoint requires authentication. Include the JWT token in one of the following ways:

- **Cookie**: Token automatically sent if stored in cookies
- **Authorization Header**: `Authorization: Bearer <token>`

#### Request Body

No request body required.

#### Response

##### Success Response (200 OK)

```json
{
  "user": {
    "_id": "507f1f77bcf86cd799439011",
    "fullname": {
      "firstname": "John",
      "lastname": "Doe"
    },
    "email": "john.doe@example.com"
  }
}
```

##### Error Response (401 Unauthorized)

```json
{
  "message": "Unauthorized"
}
```

#### Status Codes

| Status Code | Description                                       |
| ----------- | ------------------------------------------------- |
| 200         | Profile retrieved successfully                    |
| 401         | Unauthorized - missing, invalid, or expired token |
| 500         | Server error                                      |

#### Notes

- Requires a valid JWT token
- Returns the user object without the password field
- Token can be blacklisted after logout

## User Logout Endpoint

### GET /users/logout

Log out the authenticated user by invalidating their token.

#### Authentication

This endpoint requires authentication. Include the JWT token in one of the following ways:

- **Cookie**: Token automatically sent if stored in cookies
- **Authorization Header**: `Authorization: Bearer <token>`

#### Request Body

No request body required.

#### Response

##### Success Response (200 OK)

```json
{
  "message": "Logged out successfully"
}
```

##### Error Response (401 Unauthorized)

```json
{
  "message": "Unauthorized"
}
```

#### Status Codes

| Status Code | Description                                       |
| ----------- | ------------------------------------------------- |
| 200         | User logged out successfully                      |
| 401         | Unauthorized - missing, invalid, or expired token |
| 500         | Server error                                      |

#### Notes

- Clears the authentication token cookie
- Adds the token to a blacklist to prevent further use
- After logout, the token cannot be used for authentication
- User must login again to receive a new token

## Authentication Middleware

### Overview

The authentication middleware (`authUser`) protects routes that require user authentication. It verifies the JWT token and attaches the user object to the request.

### How It Works

1. **Token Extraction**: Retrieves token from cookies or Authorization header
2. **Token Validation**: Checks if token exists and is not blacklisted
3. **JWT Verification**: Verifies token signature and expiration
4. **User Attachment**: Fetches user from database and attaches to `req.user`
5. **Access Grant**: Calls `next()` to proceed to the route handler

### Token Sources

The middleware accepts tokens from two sources (in order of priority):

1. **Cookies**: `req.cookies.token`
2. **Authorization Header**: `Authorization: Bearer <token>`

### Error Responses

All authentication failures return:

```json
{
  "message": "Unauthorized"
}
```

**Status Code**: 401 Unauthorized

### Common Failure Scenarios

- Token is missing from request
- Token has been blacklisted (after logout)
- Token signature is invalid
- Token has expired
- User associated with token no longer exists

### Usage Example

Protected routes automatically have access to the authenticated user:

```javascript
router.get("/profile", authMiddleware.authUser, userController.getUserProfile);
```

The `req.user` object contains the full user document (excluding password) for use in route handlers.

## Captain Registration Endpoint

### POST /captains/register

Register a new captain account with vehicle information.

#### Request Body

The request must include a JSON body with the following structure:

```json
{
  "fullname": {
    "firstname": "string",
    "lastname": "string"
  },
  "email": "string",
  "password": "string",
  "vehicle": {
    "color": "string",
    "plate": "string",
    "capacity": number,
    "vehicleType": "string"
  }
}
```

#### Field Requirements

| Field                 | Type   | Required | Validation                           |
| --------------------- | ------ | -------- | ------------------------------------ |
| `fullname.firstname`  | String | Yes      | Minimum 3 characters                 |
| `fullname.lastname`   | String | No       | Minimum 3 characters (if provided)   |
| `email`               | String | Yes      | Valid email format                   |
| `password`            | String | Yes      | Minimum 6 characters                 |
| `vehicle.color`       | String | Yes      | Minimum 3 characters                 |
| `vehicle.plate`       | String | Yes      | Minimum 3 characters, must be unique |
| `vehicle.capacity`    | Number | Yes      | Minimum 1                            |
| `vehicle.vehicleType` | String | Yes      | Must be one of: 'bike', 'car', 'cng' |

#### Example Request

```json
{
  "fullname": {
    "firstname": "test_captain_firstname",
    "lastname": "test_captain_lastname"
  },
  "email": "test_email@gmail.com",
  "password": "test_captain",
  "vehicle": {
    "color": "red",
    "plate": "MP 04 XY 6204",
    "capacity": 3,
    "vehicleType": "car"
  }
}
```

#### Response

##### Success Response (201 Created)

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "captain": {
    "_id": "507f1f77bcf86cd799439011",
    "fullname": {
      "firstname": "test_captain_firstname",
      "lastname": "test_captain_lastname"
    },
    "email": "test_email@gmail.com",
    "vehicle": {
      "color": "red",
      "plate": "MP 04 XY 6204",
      "capacity": 3,
      "vehicleType": "car"
    },
    "status": "inactive"
  }
}
```

##### Error Response (400 Bad Request)

```json
{
  "errors": [
    {
      "msg": "First name must be at least 3 characters long",
      "param": "fullname.firstname",
      "location": "body"
    },
    {
      "msg": "Please enter a valid email address",
      "param": "email",
      "location": "body"
    },
    {
      "msg": "Password must be at least 6 characters long",
      "param": "password",
      "location": "body"
    },
    {
      "msg": "Color must be at least 3 characters long",
      "param": "vehicle.color",
      "location": "body"
    },
    {
      "msg": "Plate must be at least 3 characters long",
      "param": "vehicle.plate",
      "location": "body"
    },
    {
      "msg": "Capacity must be at least 1",
      "param": "vehicle.capacity",
      "location": "body"
    },
    {
      "msg": "Vehicle type must be bike, car, or cng",
      "param": "vehicle.vehicleType",
      "location": "body"
    }
  ]
}
```

##### Error Response (400 Bad Request - Duplicate Email)

```json
{
  "message": "Captain with this email already exists"
}
```

#### Status Codes

| Status Code | Description                           |
| ----------- | ------------------------------------- |
| 201         | Captain successfully created          |
| 400         | Validation error - invalid input data |
| 500         | Server error                          |

#### Notes

- Password is automatically hashed before storage using bcrypt
- JWT token expires in 24 hours
- Email must be unique (duplicate emails will return an error)
- Vehicle plate must be unique across all captains
- Captain status defaults to 'inactive'
- The response does not include the password field for security reasons
- Vehicle type is restricted to three options: 'bike', 'car', or 'cng'
