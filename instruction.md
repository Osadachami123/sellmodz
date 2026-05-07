# SellModzLK - Project Development Instructions

## 1. Project Overview
**Project Name:** SellModzLK  
**Description:** A marketplace for buying and selling Bus Simulator mods in Sri Lanka.  
**Key Features:**
- **Roles:** Admin (Super User), Seller (Approved via application), Buyer (Registered/ Guest).
- **Mod Types:** Public (Unlimited downloads) & Private (Sold out after one purchase).
- **Categories:** Bodykit, Leyland mod, Spoiler, Rim Modz, Other.
- **Game Versions:** V3.7.1, V4.3.4, V4.4, V4.4.1, V4.5 (Dynamic addition by Admin).
- **Payments:** Ez Cash, Bank Transfer, Reload (Dialog, Hutch, Mobitel, Airtel).
- **Storage:** Images on Cloudinary, Mod Files on Cloudflare R2.
- **Theme:** Professional Dark Mode, Fully Responsive (Mobile First).

---

## 2. Technology Stack

### Frontend
- **Framework:** Next.js 14+ (App Router)
- **Styling:** Tailwind CSS (Dark mode default)
- **State Management:** React Context or Zustand
- **Deployment:** Vercel

### Backend
- **Runtime:** Node.js
- **Framework:** Express.js
- **Validation:** Joi or Zod
- **Deployment:** Render

### Database
- **Type:** MongoDB (Atlas)
- **ODM:** Mongoose

### Storage & Media
- **Images (Slips, Thumbnails):** Cloudinary
- **Files (Mod Downloads):** Cloudflare R2 (S3 Compatible)

---

## 3. Database Schema Design (MongoDB)

*Note: Use unique collection names to avoid conflicts with existing DB data.*

### 3.1 Users Collection (`sellmodz_users`)
```javascript
{
  _id: ObjectId,
  username: String,
  email: String,
  password: HashedString,
  role: Enum['admin', 'seller', 'buyer'], // Default: 'buyer'
  isSellerApproved: Boolean, // Default: false
  sellerProfile: {
    youtubeChannel: String,
    whatsapp: String,
    proofDescription: String,
    paymentMethods: [{
      type: Enum['ezcash', 'bank', 'reload'],
      details: Object // Dynamic based on type
    }]
  },
  createdAt: Date
}
```

### 3.2 Mods Collection (`sellmodz_items`)
```javascript
{
  _id: ObjectId,
  sellerId: ObjectId (Ref: sellmodz_users),
  title: String,
  description: String,
  category: Enum['bodykit', 'leyland', 'spoiler', 'rim', 'other'],
  modType: Enum['public', 'private'], // Private = Sold out after 1 buy
  isFree: Boolean,
  price: Number,
  thumbnail: String (Cloudinary URL),
  versions: [{
    versionLabel: String (e.g., "V4.4"),
    fileUrl: String (Cloudflare R2 URL),
    isActive: Boolean
  }],
  downloadCount: Number,
  soldCount: Number, // For private mods
  status: Enum['pending', 'approved', 'rejected'], // Admin approval for listing
  createdAt: Date
}
```

### 3.3 Orders/Transactions Collection (`sellmodz_orders`)
```javascript
{
  _id: ObjectId,
  userId: ObjectId,
  modId: ObjectId,
  sellerId: ObjectId,
  amount: Number,
  paymentMethod: String,
  slipImage: String (Cloudinary URL),
  status: Enum['pending_verification', 'approved', 'rejected'],
  createdAt: Date
}
```

### 3.4 Game Versions Collection (`sellmodz_game_versions`)
```javascript
{
  _id: ObjectId,
  versionName: String (e.g., "V4.5"),
  isActive: Boolean
}
// Only Admin can add/edit this
```

---

## 4. Key Functional Requirements

### 4.1 Authentication & Authorization
- **Registration:** Open for Buyers. 
- **Seller Application:** Users can apply to become sellers by submitting YouTube links/proof.
- **Approval Flow:** 
  - New seller accounts have `isSellerApproved: false`.
  - Only the **Main Admin** can toggle this to `true`.
  - Until approved, users cannot upload mods.
- **Login:** JWT-based authentication.

### 4.2 Mod Management (Seller Dashboard)
- Sellers can upload mods with:
  - Title, Description, Category.
  - Type: Public vs Private.
  - Pricing: Free vs Paid.
  - **Versioning:** Select active game versions (fetched from DB) and upload specific files for each version to Cloudflare R2.
  - **Payment Details:** Sellers set their own EzCash/Bank/Reload numbers in their profile. These appear dynamically on the product page.

### 4.3 Buying Process
- **Free Mods:** Direct download link (no login required for viewing, optional login for tracking).
- **Paid Mods:**
  1. User selects version.
  2. System shows Seller's payment details.
  3. User uploads payment slip (Cloudinary).
  4. Order status becomes "Pending Verification".
  5. **Admin/Seller** verifies slip -> Status "Approved" -> Download link revealed.
- **Private Mods Logic:** If `modType === 'private'` and `soldCount >= 1`, mark as "Sold Out".

### 4.4 Admin Panel
- **User Management:** Approve/Reject Seller applications.
- **Content Management:** Add new Game Versions (e.g., V4.6).
- **Moderation:** Approve/Reject uploaded Mods before they go live (optional safety step).
- **Order Oversight:** View all transactions.

---

## 5. UI/UX Guidelines

- **Theme:** Dark Mode by default (`bg-slate-900`, `text-gray-100`).
- **Responsiveness:** Mobile-first design. Navigation should be a bottom bar or hamburger menu on mobile.
- **Clarity:** 
  - Clearly label "FREE" vs "PAID" badges on mod cards.
  - Clearly label "PUBLIC" (Unlimited) vs "PRIVATE" (One-time) tags.
- **Performance:** Lazy load images. Use Next.js Image optimization for Cloudinary.

---

## 6. Environment Variables (.env)

Configure these in both Backend (Render) and Frontend (Vercel).

```bash
# Database
MONGO_URI=mongodb+srv://osada:pukathama@cluster0.pf6zyl4.mongodb.net/?appName=Cluster0
DB_NAME=sellmodz_lk_db

# Server
PORT=5000
NODE_ENV=production
JWT_SECRET=your_super_secure_jwt_secret_key_change_this

# Cloudflare R2 (For Mod Files)
R2_ACCOUNT_ID=d769697a8a791c7d113491e7c2d2b1b0
R2_ACCESS_KEY_ID=2788bd904fd0f9228406bb36d7805016
R2_SECRET_ACCESS_KEY=ba36bc459b4361ca6e0b4f9a54f84351cd537f06cc27a62a7ebe695bf8b6d99f
R2_BUCKET_NAME=rdx-store-files
R2_PUBLIC_URL=https://pub-ee943591805f4bb5a18f244f59019894.r2.dev

# Cloudinary (For Images/Slips)
CLOUDINARY_CLOUD_NAME=dkknbfxlk
CLOUDINARY_API_KEY=227421668352571
CLOUDINARY_API_SECRET=VI8GUzWKeyxT47sUTdWJQuAsQMY

# Admin Contact
ADMIN_WHATSAPP=94750458513
```

---

## 7. Implementation Steps

### Phase 1: Setup & Backend
1. Initialize Node.js project on Render.
2. Connect to MongoDB using Mongoose.
3. Set up Cloudinary and Cloudflare R2 SDKs.
4. Create API endpoints:
   - `POST /api/auth/register` (Handle seller application fields)
   - `POST /api/mods/upload` (Protected: Seller only, handles R2 upload)
   - `POST /api/orders/create` (Handle slip upload to Cloudinary)
   - `PATCH /api/admin/approve-seller/:id` (Admin only)
   - `POST /api/admin/add-version` (Admin only)

### Phase 2: Frontend (Next.js)
1. Setup Next.js with Tailwind CSS (Dark theme config).
2. Create Pages:
   - `Home`: Featured mods, Search filter (Category, Version).
   - `ModDetails`: Display info, Version selector, Payment info, Buy/Download button.
   - `Dashboard/Seller`: Upload form, Sales history, Payment settings.
   - `Dashboard/Admin`: Seller approvals, Version management.
   - `Auth`: Login/Register/Application form.
3. Integrate APIs.

### Phase 3: Testing & Deployment
1. Test Seller Application flow (Apply -> Admin Approve -> Upload).
2. Test Private Mod logic (Buy once -> Sold out).
3. Test File Uploads (R2 for mods, Cloudinary for slips).
4. Deploy Frontend to Vercel, Backend to Render.

---

## 8. Special Logic Notes

- **Default Admin:** Seed the database with one initial Admin account during the first deployment.
- **Seller Payment Flexibility:** When a seller uploads a mod, the checkout page should fetch *that specific seller's* payment details from their profile, not a global site admin number.
- **Free Mods:** If `isFree === true`, skip the slip upload step and provide the R2 link directly (or after a simple click count).
- **Security:** Ensure R2 bucket policy allows public read only for objects if using public URLs, or use signed URLs for paid content security. (Recommended: Signed URLs for paid mods).
