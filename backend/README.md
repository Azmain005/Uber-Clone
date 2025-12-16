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
