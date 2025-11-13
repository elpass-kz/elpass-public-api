# Elpass Card Management API - Integration Guide for Apartment Service Providers

## Overview

This guide explains how to integrate your apartment management application with the Elpass Access Control system. The API allows you to upload resident information and face photos, which are then synchronized with physical access control terminals (Hikvision and Dahua face recognition devices).

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Authentication](#authentication)
3. [Card Group Naming Convention](#card-group-naming-convention)
4. [Card Data Model](#card-data-model)
5. [Common Integration Scenarios](#common-integration-scenarios)
6. [Error Handling](#error-handling)
7. [FAQ](#faq)

---

## Getting Started

### Prerequisites

- JWT authentication token (provided by Elpass support)
- Your apartment service provider identifier (used for `group` naming)
- Base API URL: `https://api.elpass.io` (or development: `http://localhost:3000`)

### API Endpoint

The API backend is built with **PostgREST**, which provides a RESTful interface directly to the PostgreSQL database.

**Base Endpoint**: `/el_tcards`

Example:
```
POST   https://api.elpass.io/el_tcards          # Create card
GET    https://api.elpass.io/el_tcards          # List cards
GET    https://api.elpass.io/el_tcards?...      # Filter cards
PATCH  https://api.elpass.io/el_tcards?no=eq.{uuid}  # Update card
DELETE https://api.elpass.io/el_tcards?no=eq.{uuid}  # Delete card
```

---

## Authentication

All API requests require a JWT Bearer token in the `Authorization` header.

### Example:
```bash
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Adding token to requests:

**cURL**:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" https://api.elpass.io/el_tcards
```

**Python (requests)**:
```python
import requests

headers = {
    "Authorization": f"Bearer {your_token}",
    "Content-Type": "application/json"
}
response = requests.get("https://api.elpass.io/el_tcards", headers=headers)
```

**Node.js (fetch)**:
```javascript
const headers = {
    'Authorization': `Bearer ${yourToken}`,
    'Content-Type': 'application/json'
};
fetch('https://api.elpass.io/el_tcards', { headers })
```

---

## Card Group Naming Convention

The **`group`** field is used to organize cards by physical location and assign access permissions.

### Format

```
{SITE_ID}+{APARTMENT_NUMBER}
```

### Components

- **SITE_ID**: Unique identifier for your property/complex (uppercase, alphanumeric + underscore/hyphen)
- **+**: Literal plus sign separator (required)
- **APARTMENT_NUMBER**: Unit/apartment identifier (alphanumeric + underscore/hyphen)

### Examples

```
SITE001+A101                    # Site 1, Apartment A101
BUILDING_A+202                  # Building A, Unit 202
COMPLEX_1+UNIT_501              # Complex 1, Unit 501
MY_PROPERTY_KZ+FLAT_1205        # Your property, Flat 1205
```

### Important Rules

1. ✅ Must contain exactly one `+` separator
2. ✅ Both parts must be non-empty
3. ✅ Allowed characters: `a-z`, `A-Z`, `0-9`, `_`, `-`
4. ❌ No spaces, special characters, or multiple `+` signs
5. ❌ Must be a valid pattern: `^[a-zA-Z0-9_-]+\+[a-zA-Z0-9_-]+$`

### Recommended Format

```
{PROVIDER_CODE}_{PROPERTY_ID}+{APARTMENT_ID}
```

Example: If your company code is `APT` and you manage property `PROP001`:
- `APT_PROP001+A101`
- `APT_PROP001+A102`
- `APT_PROP001+B201`

---

## Card Data Model

### Card Create Request

When registering a new resident card, send these required fields:

```json
{
  "no": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Ivan Petrov",
  "group": "SITE001+A101",
  "begin_at": "2025-01-15T00:00:00Z",
  "end_at": "2026-01-15T23:59:59Z",
  "photo": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABg..."
}
```

### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| **no** | UUID | ✅ Yes | Card number - **must be UUID from your resident database** (unique across system) |
| **name** | String | ✅ Yes | Resident's full name (1-100 characters) |
| **group** | String | ✅ Yes | Card group - format: `SITE_ID+APARTMENT_NO` |
| **begin_at** | ISO 8601 DateTime | ✅ Yes | Card activation date (must be before end_at) |
| **end_at** | ISO 8601 DateTime | ✅ Yes | Card expiration date (must be after begin_at) |
| **photo** | Base64 String | ❌ Optional | Face photo for access terminals (see below) |

### Card Response

When a card is created or retrieved, you receive:

```json
{
  "uuid": "a1b2c3d4e5f6g7h8",
  "no": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Ivan Petrov",
  "group": "SITE001+A101",
  "photo": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABg...",
  "begin_at": "2025-01-15T00:00:00Z",
  "end_at": "2026-01-15T23:59:59Z",
  "isOK": true,
  "isDisabled": false,
  "created_at": "2025-01-10T14:30:00Z",
  "updated_at": "2025-01-10T14:30:00Z",
  "deleted_at": null
}
```

### Important Fields

- **`uuid`**: Internal system identifier (16 characters) - set by system, don't modify
- **`isOK`** (readonly): `true` if card is valid and active; `false` if disabled, expired, or deleted
- **`isDisabled`**: Set to `true` to block card access without deletion
- **`deleted_at`**: Soft delete timestamp (null = not deleted)

### Photo Format

Photos must be **Base64 encoded** with MIME type prefix:

**Format**:
```
data:image/{format};base64,{base64_string}
```

**Supported formats**:
- `data:image/jpeg;base64,...`
- `data:image/png;base64,...`

**Requirements**:
- Recommended resolution: 640x480 or larger
- Max decoded size: 5MB
- Clear frontal face image for better recognition

**Example Python code to encode photo**:
```python
import base64

with open('resident_photo.jpg', 'rb') as f:
    photo_base64 = base64.b64encode(f.read()).decode()
    photo_uri = f"data:image/jpeg;base64,{photo_base64}"
    # Use photo_uri in API request
```

---

## Common Integration Scenarios

### Scenario 1: Register New Resident

When a resident moves in and needs access card:

```bash
curl -X POST https://api.elpass.io/el_tcards \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "no": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Ivan Petrov",
    "group": "SITE001+A101",
    "begin_at": "2025-02-01T00:00:00Z",
    "end_at": "2026-02-01T23:59:59Z",
    "photo": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABg..."
  }'
```

**Response (201 Created)**: Card registered and synchronized to access terminals

### Scenario 2: Update Resident Information

When resident changes name or photo:

```bash
curl -X PATCH "https://api.elpass.io/el_tcards?no=eq.550e8400-e29b-41d4-a716-446655440000" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "Ivan Petrov Jr.",
    "photo": "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABg..."
  }'
```

### Scenario 3: Extend Card Validity

When resident renews lease:

```bash
curl -X PATCH "https://api.elpass.io/el_tcards?no=eq.550e8400-e29b-41d4-a716-446655440000" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "end_at": "2027-02-01T23:59:59Z"
  }'
```

### Scenario 4: Temporarily Block Resident

When rent is unpaid or access needs suspension:

```bash
curl -X PATCH "https://api.elpass.io/el_tcards?no=eq.550e8400-e29b-41d4-a716-446655440000" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "isDisabled": true
  }'
```

**Note**: Card is not deleted, just deactivated. Re-enable by sending `"isDisabled": false`

### Scenario 5: Remove Resident Access

When resident moves out:

```bash
curl -X DELETE "https://api.elpass.io/el_tcards?no=eq.550e8400-e29b-41d4-a716-446655440000" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

**Response (204 No Content)**: Card is soft-deleted (preserved for audit trail)

### Scenario 6: List All Residents in Apartment

Get all cards for a specific apartment:

```bash
curl -X GET "https://api.elpass.io/el_tcards?group=eq.SITE001%2BA101" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Scenario 7: List Expired Cards

Find cards that need renewal:

```bash
curl -X GET "https://api.elpass.io/el_tcards?end_at=lt.2025-02-01T00:00:00Z&order=end_at.asc" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Scenario 8: Batch Upload Multiple Residents

When importing residents from your database, make individual POST requests in sequence or implement parallel requests with rate limiting.

**Python example**:
```python
import requests
import json

residents = [
    {
        "no": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Ivan Petrov",
        "group": "SITE001+A101",
        "begin_at": "2025-02-01T00:00:00Z",
        "end_at": "2026-02-01T23:59:59Z"
    },
    {
        "no": "550e8400-e29b-41d4-a716-446655440001",
        "name": "Maria Smirnova",
        "group": "SITE001+A202",
        "begin_at": "2025-02-01T00:00:00Z",
        "end_at": "2026-02-01T23:59:59Z"
    }
]

headers = {
    "Authorization": f"Bearer {your_token}",
    "Content-Type": "application/json"
}

for resident in residents:
    response = requests.post(
        "https://api.elpass.io/el_tcards",
        headers=headers,
        json=resident
    )
    if response.status_code == 201:
        print(f"✓ Created: {resident['name']}")
    else:
        print(f"✗ Failed: {resident['name']} - {response.json()}")
```

---

## Error Handling

### Common Error Responses

#### 400 Bad Request - Invalid group format
```json
{
  "message": "Invalid group format",
  "code": "400",
  "details": "Expected format: SITE_ID+APARTMENT_NO"
}
```

**Fix**: Ensure group follows pattern: `^[a-zA-Z0-9_-]+\+[a-zA-Z0-9_-]+$`

#### 409 Conflict - Duplicate card number
```json
{
  "message": "Duplicate key value violates unique constraint \"uk_no\"",
  "code": "23505",
  "details": "Key (no)=(550e8400-e29b-41d4-a716-446655440000) already exists."
}
```

**Fix**: Ensure `no` (UUID) is unique in your system

#### 401 Unauthorized - Invalid token
```json
{
  "message": "JWT invalid",
  "code": "401"
}
```

**Fix**: Check token is valid and hasn't expired; request new token from Elpass support

#### 422 Unprocessable Entity - Invalid data
```json
{
  "message": "Invalid timestamp format",
  "code": "22007",
  "details": "invalid input syntax for type timestamp"
}
```

**Fix**: Ensure dates are ISO 8601 format: `YYYY-MM-DDTHH:MM:SSZ`

### Retry Strategy

For production integrations, implement exponential backoff:

```python
import time

def call_api_with_retry(url, method="GET", data=None, max_retries=3):
    for attempt in range(max_retries):
        try:
            if method == "GET":
                response = requests.get(url, headers=headers)
            else:
                response = requests.post(url, json=data, headers=headers)
            
            if response.status_code < 500:
                return response
            
        except requests.RequestException:
            pass
        
        if attempt < max_retries - 1:
            wait_time = 2 ** attempt  # 1s, 2s, 4s
            time.sleep(wait_time)
    
    return None
```

---

## FAQ

### Q: What if I don't have a photo for the resident?
**A**: The `photo` field is optional. Create the card without it, and add photo later via PATCH update.

### Q: Can I change the card number (no) after creation?
**A**: No, `no` is immutable. It's the primary identifier linking to your resident database.

### Q: What's the maximum number of cards I can create?
**A**: No hard limit, but implement rate limiting (suggest 100 requests/second max). Contact support for bulk operations.

### Q: What happens when a card expires?
**A**: When current date passes `end_at`, the card becomes inactive (`isOK = false`). Access terminals will deny entry. Update `end_at` to extend.

### Q: Can multiple residents live in the same apartment?
**A**: Yes! Multiple cards can have the same `group`. Query by group to see all residents in that apartment.

### Q: How do I know if synchronization to terminals succeeded?
**A**: The API returns 201 on successful creation. The system automatically synchronizes to all registered terminals. Check terminal logs for confirmation.

### Q: What's the difference between `isDisabled` and deletion?
**A**: 
- **Delete**: Soft-delete (sets `deleted_at`), card removed from active use, preserved for audit
- **Disable**: Sets `isDisabled=true`, card not deleted, can be re-enabled later

### Q: Can I filter cards by multiple criteria?
**A**: Yes, use PostgREST `and` operator:
```
?and=(group.eq.SITE001+A101,isDisabled.eq.false,deleted_at.is.null)
```

### Q: What's the pagination limit?
**A**: Default 20 items per page. Max is 100 items per request via `limit` parameter.

### Q: How do I handle timezone-aware dates?
**A**: Always use UTC (Z suffix): `2025-02-01T00:00:00Z`

### Q: Can I update only specific fields?
**A**: Yes, PATCH requests only update fields you provide. Others remain unchanged.

---

## Support

For technical issues or questions:

- Email: support@elpass.io
- Documentation: https://docs.elpass.io
- API Status: https://status.elpass.io

---

## Appendix: Complete Integration Example

**Python integration script**:

```python
import requests
import base64
from datetime import datetime, timedelta

class ElpassCardManager:
    def __init__(self, api_url, token):
        self.api_url = api_url
        self.headers = {
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        }
    
    def create_card(self, resident_uuid, name, site_id, apartment_no, photo_path=None):
        """Create new resident access card"""
        
        photo_data = None
        if photo_path:
            with open(photo_path, 'rb') as f:
                photo_b64 = base64.b64encode(f.read()).decode()
                photo_data = f"data:image/jpeg;base64,{photo_b64}"
        
        card = {
            "no": resident_uuid,
            "name": name,
            "group": f"{site_id}+{apartment_no}",
            "begin_at": datetime.utcnow().isoformat() + "Z",
            "end_at": (datetime.utcnow() + timedelta(days=365)).isoformat() + "Z",
            "photo": photo_data
        }
        
        response = requests.post(
            f"{self.api_url}/el_tcards",
            headers=self.headers,
            json=card
        )
        
        if response.status_code == 201:
            return response.json()[0]
        else:
            raise Exception(f"Failed to create card: {response.json()}")
    
    def list_residents_in_apartment(self, site_id, apartment_no):
        """Get all cards in an apartment"""
        
        group = f"{site_id}+{apartment_no}"
        response = requests.get(
            f"{self.api_url}/el_tcards?group=eq.{group}",
            headers=self.headers
        )
        
        if response.status_code == 200:
            return response.json()
        else:
            raise Exception(f"Failed to list cards: {response.json()}")
    
    def disable_card(self, resident_uuid):
        """Temporarily disable access"""
        
        response = requests.patch(
            f"{self.api_url}/el_tcards?no=eq.{resident_uuid}",
            headers=self.headers,
            json={"isDisabled": True}
        )
        
        if response.status_code == 200:
            return response.json()[0]
        else:
            raise Exception(f"Failed to disable card: {response.json()}")
    
    def delete_card(self, resident_uuid):
        """Remove resident access"""
        
        response = requests.delete(
            f"{self.api_url}/el_tcards?no=eq.{resident_uuid}",
            headers=self.headers
        )
        
        if response.status_code == 204:
            return True
        else:
            raise Exception(f"Failed to delete card: {response.text}")

# Usage example
manager = ElpassCardManager(
    api_url="https://api.elpass.io",
    token="YOUR_JWT_TOKEN"
)

# Create card
card = manager.create_card(
    resident_uuid="550e8400-e29b-41d4-a716-446655440000",
    name="Ivan Petrov",
    site_id="SITE001",
    apartment_no="A101",
    photo_path="resident_photo.jpg"
)
print(f"✓ Card created: {card['uuid']}")

# List residents in apartment
residents = manager.list_residents_in_apartment("SITE001", "A101")
print(f"✓ Found {len(residents)} residents in SITE001+A101")

# Disable access (unpaid rent)
manager.disable_card("550e8400-e29b-41d4-a716-446655440000")
print("✓ Card disabled")

# Delete on move-out
manager.delete_card("550e8400-e29b-41d4-a716-446655440000")
print("✓ Card deleted")
```

---

**Document Version**: 1.0  
**Last Updated**: January 2025  
**API Version**: PostgREST 1.0
