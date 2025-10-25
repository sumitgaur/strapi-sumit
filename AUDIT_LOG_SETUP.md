# **Strapi Content API Audit Log System**

This system automatically generates audit log entries whenever records are created, updated, or deleted through the Content API.

---

## **Key Features**

- **Automatic Logging**: Captures all Content API operations (create, update, delete).
- **Detailed Records**: Logs content type, record ID, action, timestamp, user, and changed fields.
- **Optimized Queries**: Includes database indexes for fast retrieval.
- **Customizable**: Allows exclusion of specific content types and fields.
- **User Association**: Links actions to authenticated users.
- **Request Metadata**: Stores IP address, user agent, and request ID.

---

## **Installation Steps**

1. **Copy audit log files** into your Strapi project:
  