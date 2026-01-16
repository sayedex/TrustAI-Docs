# Launchpad Backend API Documentation

## What's this?

This is the backend API that sits between our frontend and the smart contract. It handles all the off-chain data - things like project descriptions, logos, social media links, and admin management. Basically, anything that doesn't need to live on the blockchain goes through this API.

Built with Express.js and MongoDB.

---

## API Endpoints

### Adding Launchpad Data

**POST** `/addLaunchpadData`

When a project creates their launchpad on-chain, they also need to add metadata about their project. This is where that happens.

**Request Body:**
```json
{
  "tokenAddress": "0x123...",
  "name": "Project Name",
  "description": "What your project is about",
  "logo": "https://yoursite.com/logo.png",
  "website": "https://yourproject.com",
  "socialMedia": {
    "twitter": "https://twitter.com/yourproject",
    "telegram": "https://t.me/yourproject",
    "discord": "https://discord.gg/yourproject"
  }
}
```

**Response:**
```json
{
  "success": true,
  "message": "Launchpad data added successfully",
  "data": {
    "tokenAddress": "0x123...",
    "name": "Project Name",
    // ... rest of the data
  }
}
```

**Why this exists:**  
The smart contract only handles token sales. Storing project descriptions and images on-chain would be expensive and unnecessary. This API stores that metadata in a database so it's easy to fetch and display.

---

### Getting All Launchpads

**GET** `/getAllLaunchpads`

Fetches all launchpads in the system. Used for the main marketplace view where users browse available sales.

**Query Parameters:**
- `page` (optional) - Page number for pagination (default: 1)
- `limit` (optional) - Items per page (default: 10)
- `status` (optional) - Filter by status (pending, approved, rejected, ended)

**Example Request:**
```
GET /getAllLaunchpads?page=1&limit=20&status=approved
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "tokenAddress": "0x123...",
      "name": "Project A",
      "description": "...",
      "logo": "...",
      "status": "approved",
      "createdAt": "2024-01-15T10:30:00Z"
    },
    // ... more launchpads
  ],
  "pagination": {
    "currentPage": 1,
    "totalPages": 5,
    "totalItems": 50,
    "itemsPerPage": 10
  }
}
```

---

### Getting a Specific Launchpad

**GET** `/getLaunchpad/:tokenAddress`

Fetches detailed information about a single launchpad. Used when someone clicks into a specific project.

**Example Request:**
```
GET /getLaunchpad/0x123...
```

**Response:**
```json
{
  "success": true,
  "data": {
    "tokenAddress": "0x123...",
    "name": "Project Name",
    "description": "Detailed description...",
    "logo": "https://...",
    "website": "https://...",
    "socialMedia": {
      "twitter": "...",
      "telegram": "...",
      "discord": "..."
    },
    "status": "approved",
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

---

### Updating Launchpad Info

**PUT** `/updateLaunchpad/:tokenAddress`

Projects can update their information after launching. Useful if they want to change their description, update social links, or fix a logo.

**Request Body:**
```json
{
  "name": "Updated Project Name",
  "description": "New description",
  "logo": "https://newlogo.png",
  "website": "https://newsite.com",
  "socialMedia": {
    "twitter": "https://twitter.com/updated",
    "telegram": "https://t.me/updated"
  }
}
```

**Response:**
```json
{
  "success": true,
  "message": "Launchpad updated successfully",
  "data": {
    // updated launchpad data
  }
}
```

**Note:** You can update individual fields - no need to send everything. Just send what you want to change.

---

### Getting Admin Launchpads

**GET** `/getAllLaunchpadsByAdmin`

Shows all launchpads filtered for admin view. Includes pending, approved, and rejected sales. Used in the admin dashboard.

**Query Parameters:**
- `status` (optional) - Filter by status
- `page` (optional) - Page number
- `limit` (optional) - Items per page

**Example Request:**
```
GET /getAllLaunchpadsByAdmin?status=pending&page=1
```

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "tokenAddress": "0x123...",
      "name": "Project Name",
      "status": "pending",
      "submittedAt": "2024-01-15T10:30:00Z",
      // ... other details
    }
  ],
  "stats": {
    "pending": 5,
    "approved": 20,
    "rejected": 3
  }
}
```

---

### Launchpad Statistics

**GET** `/launchpad/stats`

Gets platform-wide statistics. Shows total launchpads, total raised, active sales, etc. Used for the homepage dashboard.

**Response:**
```json
{
  "success": true,
  "data": {
    "totalLaunchpads": 50,
    "activeSales": 12,
    "totalRaised": "5000000",
    "totalParticipants": 15000,
    "byStatus": {
      "pending": 5,
      "approved": 30,
      "rejected": 3,
      "ended": 12
    }
  }
}
```

---

### Getting Admin List

**GET** `/getAdminList`

Returns all admins and their permissions. Used for admin management in the dashboard.

**Response:**
```json
{
  "success": true,
  "data": [
    {
      "address": "0xAdmin1...",
      "displayName": "John Doe",
      "permissions": {
        "canApproveLaunchpad": true,
        "canRejectLaunchpad": true,
        "canPauseLaunchpad": true,
        "canEditTokenPrice": false,
        // ... other permissions
      },
      "isActive": true,
      "addedAt": "2024-01-10T08:00:00Z"
    }
  ]
}
```

---

## Error Handling

All endpoints return consistent error responses:

```json
{
  "success": false,
  "error": "Error message here",
  "code": "ERROR_CODE"
}
```

**Common error codes:**
- `LAUNCHPAD_NOT_FOUND` - Requested launchpad doesn't exist
- `INVALID_ADDRESS` - Token address is invalid
- `VALIDATION_ERROR` - Request body validation failed
- `UNAUTHORIZED` - Admin permission required
- `DATABASE_ERROR` - Something went wrong with the database

---

## How it integrates

**Frontend → API → Smart Contract flow:**

1. User creates launchpad on-chain via MetaMask
2. Smart contract emits `LaunchpadAdded` event
3. Frontend listens for event, gets token address
4. Frontend calls `/addLaunchpadData` with metadata
5. API stores metadata in database
6. Admin reviews in dashboard (fetches via `/getAllLaunchpadsByAdmin`)
7. Admin approves on-chain
8. Users see it in marketplace (via `/getAllLaunchpads`)

**Why this architecture?**

Keeps costs low. Project descriptions, logos, and social links don't need blockchain permanence. They can change frequently. Storing this stuff on-chain would cost hundreds of dollars per project in gas fees.

The smart contract handles the financial stuff (token sales, vesting, refunds). The API handles the presentational stuff (names, descriptions, images).

---

## Security considerations

**Input validation:**  
All endpoints validate input before touching the database. Token addresses are checked, URLs are validated, and text fields have length limits.

**Rate limiting:**  
Heavy endpoints like `getAllLaunchpads` are rate-limited to prevent abuse and database overload.

**Admin verification:**  
Admin-only endpoints verify the caller's address against the smart contract's admin list. Can't just claim to be an admin.

**CORS:**  
Configured to only accept requests from the frontend domain. Prevents random sites from hitting the API.

---
