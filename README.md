# phones

## Recommended commerce stack

| Component | Service | Why |
| --- | --- | --- |
| Storefront | Firebase Hosting (or Vercel) | Global CDN, free SSL, simple deploys |
| Database | Firestore | Product catalog, orders, user profiles |
| Auth | Firebase Authentication | Customer accounts, guest-to-account merging |
| Images | Cloud Storage | Product photos for phones/watches |
| Payments | Cloud Functions + Paystack | Secure server-side transaction handling |
| Search/Filter | Algolia (or Typesense) | Firestore cannot handle rich multi-attribute filtering well |

## Honest verdict

Firebase is a strong fit for this store. It removes most operational overhead so the team can focus on selling instead of managing servers.

Trade-offs:
- Search/filter quality still needs a dedicated search engine (Algolia/Typesense).
- The platform is more tightly coupled to Google services.

For a phones and watches store, this is generally a practical trade.

## Firestore collection sketch

```text
users/{userId}
  - email
  - displayName
  - createdAt
  - role ("customer" | "admin")

products/{productId}
  - sku
  - name
  - brand
  - category ("phone" | "watch")
  - price
  - currency
  - stock
  - imageUrls[]
  - isActive
  - attributes (map: color, storage, connectivity, etc.)
  - searchSyncedAt

orders/{orderId}
  - userId
  - items[] (productId, nameSnapshot, priceSnapshot, qty)
  - subtotal
  - shippingFee
  - total
  - status ("pending" | "paid" | "shipped" | "delivered" | "cancelled")
  - paymentProvider ("paystack")
  - paymentReference
  - createdAt
  - updatedAt

carts/{userId}
  - items[] (productId, qty)
  - updatedAt
```

## Firestore security rules starter

```firebase
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isSignedIn() {
      return request.auth != null;
    }

    function isAdmin() {
      return isSignedIn() && request.auth.token.role == 'admin';
    }

    match /products/{productId} {
      allow read: if resource.data.isActive == true || isAdmin();
      allow create, update, delete: if isAdmin();
    }

    match /users/{userId} {
      allow read, write: if isSignedIn() && request.auth.uid == userId;
    }

    match /carts/{userId} {
      allow read, write: if isSignedIn() && request.auth.uid == userId;
    }

    match /orders/{orderId} {
      allow create: if isSignedIn() && request.resource.data.userId == request.auth.uid;
      allow read: if isSignedIn() && resource.data.userId == request.auth.uid;
      allow update, delete: if isAdmin();
    }
  }
}
```

> Note: Payment verification and order state transitions should run in trusted Cloud Functions, not directly from clients.
