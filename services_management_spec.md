# Services Management — Feature Specification

> **Source of Truth:** Web Commander (`squire-app/frontend/js/app/main/services/`)
> **iOS Reference:** iOS Commander (`ios-commander/Commander/Packages/NE/`)
> **Target:** Mobile Commander (React Native / Expo)
> **Date:** 2026-02-24

---

## 1. Executive Summary

Services Management enables shop owners, barbers, and brand administrators to create, edit, delete, and categorize services within the Squire platform. Services are the core bookable and chargeable units — each has a name, description, duration, pricing, tax configuration, category assignment, and visibility/payment settings. The feature supports multi-language localization (English, Spanish, French), granular per-barber/per-shop assignment with individualized pricing and duration overrides, and brand-scoped service category management.

### Flows Covered

1. **Create Service** — Add a new service with details and optional assignment
2. **Edit Service** — Modify an existing service's details
3. **Delete Service** — Remove a service with confirmation
4. **Manage Categories** — Full CRUD for service categories (brand-scoped)
5. **Service Assignments (Optional)** — Assign services to shops and barbers with per-entity overrides for price, duration, and visibility (Shop/Brand users only)

### User Types

| User Type | Create | Edit | Delete | Categories |
|-----------|--------|------|--------|------------|
| **Brand User** | Full access | Full access | Full access | Full CRUD |
| **Shop User** (Owner/Manager) | Full access | Limited (depends on service source) | Full access | Read-only (filter/assign) |
| **Barber User** (Staff) | Full access (own services) | Limited (depends on rental status and creator) | Own services only | Read-only (filter/assign) |

---

## 2. Feature Inventory & Traceability Table

| # | Flow | Web Commander Files | iOS Commander Files | Mobile Commander Status | User Types | Data Scope |
|---|------|--------------------|--------------------|----------------------|------------|------------|
| 1 | Create Service | `services/actions.js`, `services/containers/editServiceProvider.jsx`, `services/components/ServiceDetailsForm.jsx` | `CoreExperience/.../ServiceModification/Create/ServiceCreatePresenter.swift` | ✅ Done | Brand / Shop / Barber | Per-user |
| 2 | Edit Service | `services/actions.js`, `services/containers/editServiceProvider.jsx`, `services/components/ServiceDetailsForm.jsx` | `CoreExperience/.../ServiceModification/Edit/ServiceEditPresenter.swift` | ✅ Done | Brand / Shop / Barber | Per-user |
| 3 | Delete Service | `services/actions.js`, `services/containers/Actions.js` | `CoreExperience/.../ServiceModification/Delete/DeleteServicePresenter.swift` | ✅ Done | Brand / Shop / Barber | Per-user |
| 4 | Manage Categories | `services/categories/List.js`, `services/categories/Add.js`, `services/categories/Edit.js`, `services/categories/Form.js` | N/A (picker only) | ⚠️ Partial | Brand only (CRUD) / Shop & Barber (read) | Brand-scoped |
| 5 | Service Assignments (Optional) | `services/components/Assignments.jsx`, `services/containers/Assignments.js`, `services/hoc/assignments/*` | N/A | ❌ Missing | Brand / Shop (not Barber) | Per-shop, per-barber |

---

## 3. Detailed Tickets

### TICKET: Create Service

**Type:** Story
**Description:** Create a new service with name, description, duration, pricing, tax configuration, category, and visibility settings. Optionally assign to shops and barbers with per-entity overrides.
**Preconditions:** User has `services.modify` permission. User is authenticated and has selected a location.

#### Section 1 — User Triggers & Entry Points

- **Navigation path:** Services List Screen → "Add Service" button (top-right)
- **Route (Mobile Commander):** `/profile/services/new` via `routes.auth.profile.serviceForm()`
- **Web routes:** `/services/add` (Shop), `/barber/services/add` (Barber), `/brand/services/add` (Brand)
- **Button visibility:** Only visible when `permissions.services.modify` is granted

#### Section 2 — Acceptance Criteria (Happy Path)

- GIVEN a user with `services.modify` permission, WHEN they tap "Add Service", THEN the service form screen opens in "add" mode with empty fields
- GIVEN the form is open, WHEN the user enters a valid service name (min 3 chars), price (min 0), and duration, THEN the "Save" button becomes enabled
- GIVEN all required fields are valid, WHEN the user taps "Save", THEN a POST request is sent and the service is created
- GIVEN the service is created successfully, WHEN the API responds with the new service object, THEN the user is redirected back to the services list and the new service appears in the list
- GIVEN the user is a Shop or Brand user, WHEN the service is created, THEN the Assignments step is available to assign the service to specific barbers with overrides

#### Section 3 — UI State & Component Logic

**Form Fields:**

| Field | Type | Required | Default | Validation | Notes |
|-------|------|----------|---------|------------|-------|
| Name | Text input | Yes | Empty | Min 3 chars | Supports localization (EN, FR, ES) via `localizedName` |
| Description | Text area | No | Empty | Max 300 chars | Supports localization via `localizedDesc` |
| Duration (Hours) | Number picker | Yes | 0 | 0-12 | Combined: `duration = (hours * 60) + minutes` |
| Duration (Minutes) | Number picker | Yes | 0 | 0-55 (excludes 5 if hour < 1) | Step of 5 |
| Price | Currency input | Yes | 0 | Min 0 | Stored in cents (multiply display value by 100) |
| Category | Single-select picker | No | None | Max 1 category | From `useServiceCategoriesQuery()` |
| Tax | Multi-select | No | None | N/A | Array of tax IDs |
| Enabled (Online Visibility) | Toggle | No | `true` | N/A | Controls if service is bookable online |
| Requires Prepaid | Toggle | No | `false` | N/A | Gated by `paymentDepositsEnabled` feature flag |
| Kiosk Enabled | Toggle | No | `true` | N/A | Not shown for barber users |

**Localization:**
- Web Commander supports multi-language name/description via `localizedName` and `localizedDesc` objects
- EN translation is stored in the `name`/`desc` root fields (backend convention)
- Other languages stored in `localizedName.{lang}` and `localizedDesc.{lang}`

**Conditional UI:**
- "Requires Prepaid" toggle: Only visible when `paymentDepositsEnabled` feature flag is active
- "Kiosk Enabled" toggle: Hidden for barber users
- "Assignments" step: Only available for Shop and Brand users (not barbers)

#### Section 4 — Business Rules, Calculations & API Interactions

**API — Create Service:**

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /brand/{brandId}/service` (Brand), `POST /shop/{shopId}/service` (Shop), `POST /shop/{shopId}/barber/{barberId}/service` (Barber) |
| **Method** | POST |
| **Auth** | JWT token |

**Request Body:**
```
{
  name: string,                      // EN name (required, min 3 chars)
  desc: string,                      // EN description (optional)
  localizedName: { fr?: string, es?: string },  // Other language names
  localizedDesc: { fr?: string, es?: string },  // Other language descriptions
  serviceCategoriesId: [string],     // Array with single category ID
  duration: number,                  // Total minutes (hours*60 + minutes)
  cost: number,                      // Price in cents
  enabled: boolean,                  // Online visibility
  requiresPrepaid: boolean,          // Requires prepayment
  kioskEnabled: boolean,             // Kiosk visibility
  tax: [{ id: string }]             // Array of tax objects
}
```

**Response:** Full service object with ID, normalized into Redux/cache.

**Client-Side Validation (Joi schema in Web, Zod in Mobile):**
- Service name: required, min 3 characters per language
- Description: max 300 characters
- Duration: min 0 hours and minutes
- Cost: min 0, required
- Tax: array of valid IDs

#### Section 5 — States & Variants

| State | Description |
|-------|-------------|
| Empty form | All fields at default values, Save disabled |
| Valid form | All required fields populated, Save enabled |
| Submitting | Save button shows loading indicator, form fields disabled |
| Success | Toast "Service saved successfully", navigate back to list |
| API error | Error toast displayed, form remains editable |
| Permission denied | "Add Service" button hidden entirely |

#### Section 6 — Validations & Error Handling

**Client-Side:**
- Name: required, min 3 chars → "Service name must be at least 3 characters"
- Description: max 300 chars → "Description cannot exceed 300 characters"
- Cost: min 0 → "Cost must be a positive number"
- Duration: at least 1 minute total → "Duration is required"

**Server-Side:**
- 422: Validation errors with field-level messages
- 400: Bad request (malformed payload)
- 403: Permission denied
- 500: Server error

#### Section 7 — Analytics & Logging

- `SERVICE_CREATION_STARTED` — fired on form mount
- `SERVICE_CREATION_COMPLETED` — fired on successful save

#### Section 8 — Security & Privacy

- Permission guard: `services.modify` required
- Brand users: full access across all shops
- Shop users: full access within their shop
- Barber users: can only create services for themselves
- Commission barbers: cannot edit name/description after creation

#### Section 9 — Performance / UX Budgets

- Category list pre-fetched with 5-min stale time
- Tax list loaded on form mount
- Form submission should show loading state within 100ms

#### Section 10 — Dependencies / Remote Config / Feature Flags

- `paymentDepositsEnabled` — gates "Requires Prepaid" toggle visibility

#### Section 11 — Open Questions & Assumptions

- **Localization support in Mobile:** Web Commander supports multi-language (EN/FR/ES) for service name and description. Mobile Commander's current form does not appear to have multi-language input fields. Need to confirm if localization is required for MVP.
- **Assignments step:** ✅ RESOLVED — Documented as a separate ticket "Service Assignments (Optional)". The flow is: after saving service details, Shop/Brand users are auto-redirected to the Assignments page. Barbers skip it. See Jira ticket SQR-3179.

---

### TICKET: Edit Service

**Type:** Story
**Description:** Edit an existing service's details including name, description, duration, pricing, tax, category, and visibility settings. Supports field-level locking based on who created the service and the user's role.
**Preconditions:** User has `services.modify` permission. Service exists and is accessible to the user.

#### Section 1 — User Triggers & Entry Points

- **Navigation path:** Services List Screen → Tap on a service card
- **Route (Mobile Commander):** `/profile/services/{id}` via `routes.auth.profile.serviceForm(id)`
- **Web routes:** `/services/{serviceId}` (Shop), `/barber/services/{serviceId}` (Barber), `/brand/services/{serviceId}` (Brand)

#### Section 2 — Acceptance Criteria (Happy Path)

- GIVEN a user taps on a service, WHEN the edit form loads, THEN all fields are pre-populated with the service's current values
- GIVEN the user modifies a field, WHEN the form detects changes (isDirty), THEN the Save button becomes enabled
- GIVEN no changes were made, WHEN the user taps Save, THEN the form returns the existing service without making an API call (pristine check)
- GIVEN valid changes, WHEN the user saves, THEN a PUT request updates the service and the user is redirected back to the list
- GIVEN the user is a barber who didn't create the service, WHEN the form loads, THEN name and description fields are read-only

#### Section 3 — UI State & Component Logic

**Field Locking Rules (Critical — differs by user type and service source):**

| User Type | Service Source | Editable Fields | Locked Fields |
|-----------|---------------|-----------------|---------------|
| Brand User | Any | All fields | None |
| Shop User | Own shop | All fields | None |
| Shop User | Multi-shop brand | Limited | Name, Description |
| Barber (Rental, Creator) | Own | All fields | None |
| Barber (Non-rental or Non-creator) | Own | Price, Duration, Booking Visibility | Name, Description, Category, Taxes, Kiosk |
| Barber | Shop/Brand-created | Price, Duration only | Everything else |

**iOS Editability Status Enum:**
- `allFieldsEditable` — Rental barber who created the service
- `partlyEditable` — Barber who created the service in non-rental shop
- `onlyDurationAndPrice` — Service was created by shop or brand

**Initial Values Hydration:**
- `localizedName.en` ← `service.name`
- `localizedDesc.en` ← `service.desc`
- `serviceCategoriesId` ← `service.categories[0].id` (first category only)
- `durationHour` / `durationMinute` ← parsed from `service.duration` (minutes)
- `cost`, `enabled`, `requiresPrepaid`, `kioskEnabled` ← copied as-is
- `tax` ← array of tax IDs from `service.taxes[].id`

**Tax Handling on Edit (Web-specific pattern):**
- Detect removed taxes by comparing initial values with current values
- Removed taxes sent as `{ id: "taxId", deleted: true }` in the request
- This allows atomic add/update/delete in a single PUT

#### Section 4 — Business Rules, Calculations & API Interactions

**API — Fetch Service:**

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /shop/{shopId}/service/{serviceId}` (Shop), `GET /shop/{shopId}/barber/{barberId}/service/{serviceId}` (Barber), `GET /brand/{brandId}/service/{serviceId}` (Brand) |
| **Method** | GET |

**API — Update Service:**

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /shop/{shopId}/service/{serviceId}` (Shop), `PUT /shop/{shopId}/barber/{barberId}/service/{serviceId}` (Barber), `PUT /brand/{brandId}/service/{serviceId}` (Brand) |
| **Method** | PUT |

**Request Body (same structure as Create, only changed fields sent):**
```
{
  name: string,
  desc: string,
  localizedName: { fr?: string | null, es?: string | null },
  localizedDesc: { fr?: string | null, es?: string | null },
  serviceCategoriesId: [string],
  duration: number,
  cost: number,
  enabled: boolean,
  requiresPrepaid: boolean,
  kioskEnabled: boolean,
  tax: [{ id: string, deleted?: boolean }]   // deleted: true for removed taxes
}
```

#### Section 5 — States & Variants

| State | Description |
|-------|-------------|
| Loading | Service data being fetched, skeleton/spinner shown |
| Loaded (pristine) | All fields populated, Save disabled (no changes) |
| Loaded (dirty) | User has made changes, Save enabled |
| Submitting | Save button loading, fields disabled |
| Success | Toast "Service saved successfully", navigate to assignments or back to list |
| Error | Error toast, form remains editable |
| Locked fields | Read-only inputs with visual indicator (lock icon in iOS) |

#### Section 6 — Validations & Error Handling

Same as Create Service validation rules. Additionally:
- Pristine check: If no changes detected, skip API call and return existing service
- Field-level locking prevents invalid edits based on user permissions

#### Section 7 — Analytics & Logging

No specific edit analytics events discovered in source code.

#### Section 8 — Security & Privacy

- Same permission requirements as Create (`services.modify`)
- Field-level access control based on service source (who created it)
- Commission barbers restricted from modifying name/description

---

### TICKET: Delete Service

**Type:** Story
**Description:** Delete an existing service with a confirmation dialog. Only available in edit mode.
**Preconditions:** User has `services.modify` permission. Service exists. User is in the edit form.

#### Section 1 — User Triggers & Entry Points

- **Navigation path:** Service Edit Form → "Delete" / "Remove Service" button (bottom of form, red text)
- **Only available in edit mode** — the delete button is hidden when creating a new service
- In Web Commander: red "Delete" button in the Details step
- In Mobile Commander: "Remove Service" button at form bottom with `ConfirmationBottomSheet`

#### Section 2 — Acceptance Criteria (Happy Path)

- GIVEN the user is editing a service, WHEN they tap "Remove Service", THEN a confirmation dialog appears
- GIVEN the confirmation dialog is shown, WHEN the user confirms deletion, THEN a DELETE request is sent
- GIVEN the DELETE succeeds, WHEN the API responds, THEN the service is removed from the list, a success toast is shown, and the user is navigated back to the services list
- GIVEN the user cancels the confirmation, WHEN they tap "Cancel", THEN the dialog is dismissed and no changes are made

#### Section 3 — UI State & Component Logic

**Confirmation Dialog:**

| Platform | Component | Title | Message | Confirm | Cancel |
|----------|-----------|-------|---------|---------|--------|
| Web | ConfirmModal (danger) | "Delete service" | "Are you sure you want to delete a service?" | "Delete" (red) | "Cancel" |
| iOS | ActionDialogViewController | "Delete {serviceName}?" | N/A | "Confirm" (destructive) | "Cancel" |
| Mobile | ConfirmationBottomSheet | Includes service name | Confirmation message | "Delete" | "Cancel" |

#### Section 4 — Business Rules, Calculations & API Interactions

**API — Delete Service:**

| Field | Value |
|-------|-------|
| **Endpoint** | `DELETE /shop/{shopId}/service/{serviceId}` (Shop), `DELETE /shop/{shopId}/barber/{barberId}/service/{serviceId}` (Barber), `DELETE /brand/{brandId}/service/{serviceId}` (Brand) |
| **Method** | DELETE |
| **Request Body** | None |
| **Response** | Empty/204 No Content (Web), `{ isSuccessful: boolean }` (iOS) |

**Post-Delete:**
- Remove service from local cache/state
- Invalidate services list query
- Navigate back to services list

#### Section 5 — States & Variants

| State | Description |
|-------|-------------|
| Idle | Delete button visible at bottom of edit form |
| Confirmation shown | Bottom sheet/modal visible with confirm/cancel |
| Deleting | Loading indicator in confirmation dialog |
| Success | Toast "Service is deleted", navigate to services list |
| Error | Error toast, dialog dismissed, service remains |

#### Section 6 — Validations & Error Handling

- No client-side validation needed
- Server errors (400/500): show error toast, keep user on edit form
- 404 (service already deleted): show info message, navigate to list
- 403 (permission denied): show error message

#### Section 7 — Analytics & Logging

No specific delete analytics events discovered in source code.

#### Section 8 — Security & Privacy

- Same permission requirements: `services.modify`
- Deletion is permanent — no soft-delete or undo

---

### TICKET: Manage Categories

**Type:** Story
**Description:** Full CRUD interface for service categories. Categories are brand-scoped — only Brand users can create, edit, and delete categories. Shop and Barber users can view categories for filtering and assigning to services.
**Preconditions:** User is a Brand user with `services.modify` permission (for CRUD). Any user can view categories.

#### Section 1 — User Triggers & Entry Points

- **Web Navigation:** Services List Screen → "Categories" button → `/services/categories`
- **Button visibility (Web):** Shown via `RoleAwareLink` — accessible to Brand users only
- **Mobile Commander current state:** No dedicated categories management screen exists. Categories are used as filter pills in the services list and as a picker in the service form, but there is no UI for adding, editing, or deleting categories.

#### Section 2 — Acceptance Criteria (Happy Path)

**List Categories:**
- GIVEN a Brand user navigates to Categories, WHEN the screen loads, THEN all service categories are displayed in a list with name and optional description
- GIVEN categories exist, WHEN the user views the list, THEN each category shows an edit action and a delete action
- GIVEN no categories exist, WHEN the screen loads, THEN an empty state message is shown
- GIVEN there are more than 50 categories, WHEN the first page loads, THEN the next page is auto-fetched (pagination: limit=50, skip-based)

**Add Category:**
- GIVEN a Brand user is on the Categories list, WHEN they tap "Add Category", THEN a form with a "Name" field appears
- GIVEN a valid name (min 2 chars), WHEN the user saves, THEN a POST request creates the category and the list updates
- GIVEN an invalid name (less than 2 chars), WHEN the user tries to save, THEN a validation error is shown

**Edit Category:**
- GIVEN a Brand user taps edit on a category, WHEN the edit form loads, THEN the name field is pre-populated
- GIVEN the user changes the name, WHEN they save, THEN a PUT request updates the category

**Delete Category:**
- GIVEN a Brand user taps delete on a category, WHEN a confirmation dialog appears and the user confirms, THEN a DELETE request removes the category from the list

#### Section 3 — UI State & Component Logic

**Category List Screen (Web Commander):**
- Header: "Categories"
- Navigation buttons: "Services" (back to service list), "Add category" (link to add form)
- Table columns: Name, Description (if present), Actions (edit/delete buttons)
- Empty state: message shown when no categories exist

**Category Form (Web Commander — shared for Add and Edit):**
- Single field: `name` (text input)
- Validation: min 2 characters (Joi schema)
- Redux Form: `"categoriesForm"`
- Delete button shown on edit only (red, with confirmation modal)

**Category Form Fields:**

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| Name | Text input | Yes | Min 2 characters |

**Confirmation Dialog (Delete Category):**
- Title: "Delete item"
- Message: "Are you sure you want to delete item?"
- Actions: Confirm (destructive), Cancel

#### Section 4 — Business Rules, Calculations & API Interactions

**API — List Categories:**

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /brand/{brandId}/service-category` |
| **Method** | GET |
| **Query Params** | `limit=50`, `skip=0` |
| **Response** | `{ rows: [{ id, name, description? }], count: number }` |
| **Pagination** | Auto-fetch next page if `rows.length === 50` |

**API — Create Category:**

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /brand/{brandId}/service-category` |
| **Method** | POST |
| **Request Body** | `{ name: string }` |
| **Response** | `{ id, name }` |

**API — Update Category:**

| Field | Value |
|-------|-------|
| **Endpoint** | `PUT /brand/{brandId}/service-category/{categoryId}` |
| **Method** | PUT |
| **Request Body** | `{ name: string }` |
| **Response** | `{ id, name }` |

**API — Delete Category:**

| Field | Value |
|-------|-------|
| **Endpoint** | `DELETE /brand/{brandId}/service-category/{categoryId}` |
| **Method** | DELETE |
| **Request Body** | None |
| **Response** | 204 No Content |

**State Management (Web Commander Redux):**
- Categories stored in `services.categories = { items: [...], isFetched: boolean }`
- `ADD_ITEM`: appends to items array
- `UPDATE_ITEM`: finds by ID and replaces
- `DELETE_ITEM`: filters by ID
- State resets on `CHANGE_SHOP`, `RESET_SHOP`, `SET_BRAND` events

**Mobile Commander Current State:**
- `useServiceCategoriesQuery()` exists — fetches categories from brand ID
- `fetchServiceCategories()` API function exists
- `createServiceCategory()` API function exists (but no mutation hook)
- **Missing:** `updateServiceCategory()` API function, `deleteServiceCategory()` API function
- **Missing:** `useCreateServiceCategoryMutation` hook
- **Missing:** `useUpdateServiceCategoryMutation` hook
- **Missing:** `useDeleteServiceCategoryMutation` hook
- **Missing:** Dedicated category management screen

#### Section 5 — States & Variants

| State | Description |
|-------|-------------|
| Loading | Category list fetching, skeleton/spinner shown |
| Empty | No categories exist, empty state message |
| Populated | Categories listed with actions |
| Add form | Empty name field, Save disabled |
| Edit form | Name pre-populated, Save enabled on change |
| Deleting | Confirmation dialog with loading state |
| Error | API error toast, retry option |

#### Section 6 — Validations & Error Handling

**Client-Side:**
- Name: required, min 2 characters → "Category name must be at least 2 characters"

**Server-Side:**
- 422: Validation errors
- 400: Bad request
- 403: Permission denied (non-brand user trying to modify)
- 404: Category not found (edit/delete)

#### Section 7 — Analytics & Logging

No analytics events discovered for category management in source code.

#### Section 8 — Security & Privacy

- **Brand users only** can create, edit, and delete categories
- Shop and Barber users can only view/use categories (for filtering and assigning to services)
- The "Categories" button/link is hidden for non-Brand users in the Web Commander UI

#### Section 9 — Performance / UX Budgets

- Categories query: 5-min stale time, 30-min garbage collection (already configured in Mobile Commander)
- Pagination: auto-fetch next page at limit boundary (50 items per page)

#### Section 10 — Dependencies / Remote Config / Feature Flags

No feature flags gate category management.

#### Section 11 — Open Questions & Assumptions

- **Screen design:** No Figma designs referenced for the category management screen in Mobile Commander. The Web Commander uses a simple table layout. Mobile may need a different UX pattern (e.g., list with swipe actions, bottom sheet for add/edit).
- **Category descriptions:** Web Commander supports an optional description field in the category list display, but the create/edit form only has a name field. The description field source is unclear — it may come from the backend but not be editable via the UI.
- **Impact on services when deleting a category:** What happens to services assigned to a deleted category? The source code does not handle this explicitly — services likely lose their category assignment silently. Needs verification with backend/PM.

---

### TICKET: Service Assignments (Optional)

**Type:** Story
**Description:** After creating or editing a service, Shop and Brand users are taken to an Assignments step where they can assign the service to specific shops and barbers with per-entity overrides for price, duration, and visibility settings. This step is automatically shown for Shop/Brand users but is functionally optional — the user can leave without making any assignments. Barber users never see this step.
**Preconditions:** User is a Shop or Brand user. A service has just been created or is being edited.

#### Section 1 — User Triggers & Entry Points

- **Automatic redirect after saving service details:** When a Shop or Brand user saves a service (create or edit), they are automatically navigated to the Assignments step. The button on the details form says "Next" (not "Save") for these users.
- **Direct navigation (edit flow):** When editing an existing service, Shop/Brand users can navigate to the Assignments tab after the details step.
- **Web routes:** `/services/{serviceId}/assign` (Shop), `/brand/services/{serviceId}/assign` (Brand)
- **Not available for Barber users:** Barbers see a "Save" button that returns them directly to the services list.

#### Section 2 — Acceptance Criteria (Happy Path)

- GIVEN a Shop or Brand user saves service details, WHEN the save succeeds, THEN they are automatically redirected to the Assignments step
- GIVEN the Assignments step loads for a Brand user creating a NEW service, WHEN the page loads, THEN no shops are pre-assigned (empty table with available shops)
- GIVEN the Assignments step loads for a Shop user creating a NEW service, WHEN the page loads, THEN the user's own shop is pre-assigned
- GIVEN the Assignments page is shown, WHEN a Brand user checks a shop row, THEN the service is assigned to that shop AND all barbers in that shop are automatically assigned
- GIVEN a shop is expanded showing barbers, WHEN the user modifies a barber's price/duration/visibility, THEN only that barber's override is updated (other barbers keep their values)
- GIVEN the user has made assignments, WHEN they tap "Save", THEN they are navigated back to the services list
- GIVEN the user wants to skip assignments, WHEN they navigate away or tap "Save" with no changes, THEN the service is kept without assignments

#### Section 3 — UI State & Component Logic

**Assignment Hierarchy (3-level tree):**

```
Brand Level (default values from service)
└── Shop Level (can override: price, duration, visibility, prepaid, kiosk)
    └── Barber Level (can override: price, duration, visibility, prepaid)
```

Values cascade: Barber → Shop → Brand. If a barber has no override, the shop's value is used. If the shop has no override, the brand/service default is used.

**Table Columns:**

| Column | Type | Notes |
|--------|------|-------|
| Entity Name | Text | Shop name or barber name (with avatar for barbers) |
| Checkbox | Toggle | Enable/disable assignment for this entity |
| Duration (Hours) | Dropdown | 0-12, overrides service default |
| Duration (Minutes) | Dropdown | 0-55 in 5-min steps, overrides service default |
| Price | Currency input | Overrides service default |
| Online Visibility | Toggle | Show/hide service for this entity's online booking |
| Requires Prepaid | Toggle | Override prepayment requirement |
| Kiosk Mode | Toggle | Shop-level only (hidden for barber rows) |

**Expandable Rows:**
- Brand users see all shops as collapsible rows
- Clicking a shop row expands it to show all barbers within that shop
- Barbers are loaded asynchronously when a shop is expanded (loading spinner shown)

**Auto-Assignment Behavior:**
- When a shop checkbox is checked: all barbers in that shop are automatically assigned
- When a shop checkbox is unchecked: all barber assignments within that shop are removed
- Individual barber assignments can be toggled independently after the shop is assigned

**Initial State by User Type:**

| User Type | Creating New Service | Editing Existing Service |
|-----------|---------------------|------------------------|
| Brand | No shops pre-assigned | Existing assignments shown |
| Shop | Own shop pre-assigned | Existing assignments shown |
| Barber | N/A (no assignments step) | N/A |

**Immediate Save Behavior:**
- Each field change triggers an immediate API call (no batch "Save All" pattern)
- Loading spinner shown on the affected row during the API call
- Fields are disabled while their row is saving

**Permission-Based Row Controls:**

| User Type | Shop Rows | Barber Rows |
|-----------|-----------|-------------|
| Brand | Full control (check/uncheck, edit all fields) | Full control |
| Shop | Cannot toggle own shop checkbox (read-only) | Can toggle and edit barber fields |

#### Section 4 — Business Rules, Calculations & API Interactions

**API — Save Brand→Shop Assignment:**

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /brand/{brandId}/service/{serviceId}/save-assignment` |
| **Method** | POST |
| **Payload** | `{ assignmentType: "shop", assigneeId: shopId, cost, duration, enabled, requiresPrepaid, kioskEnabled }` |

**API — Save Shop→Barber Assignment:**

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /shop/{shopId}/service/{serviceId}/save-assignment` |
| **Method** | POST |
| **Payload** | `{ assignmentType: "barber", assigneeId: barberId, cost, duration, enabled, requiresPrepaid }` |

**API — Bulk Assign Barbers:**

| Field | Value |
|-------|-------|
| **Endpoint** | `POST /shop/{shopId}/service/{serviceId}/assign` |
| **Method** | POST |
| **Payload** | `{ barbers: [barberId1, barberId2, ...] }` |

**API — Remove Assignment:**

| Field | Value |
|-------|-------|
| **Endpoint** | `DELETE /shop/{shopId}/service/{serviceId}` |
| **Method** | DELETE |

**Fetch Service with Assignments (Brand):**

| Field | Value |
|-------|-------|
| **Endpoint** | `GET /brand/{brandId}/service?include=shops,category` |
| **Include** | `shops` returns assignment data per shop |

**Field Value Cascade Logic:**
```
getFieldValue(service, assignments, row, fieldName):
  if barber row → check barber override → check shop override → use service default
  if shop row → check shop override → use service default
```

#### Section 5 — States & Variants

| State | Description |
|-------|-------------|
| Empty (Brand, new service) | Table shows all available shops, none checked |
| Pre-assigned (Shop, new service) | Own shop row is checked with service defaults |
| Loaded (edit) | Existing assignments pre-populated from API |
| Expanding shop | Loading spinner while barbers are fetched |
| Saving row | Row fields disabled, spinner on checkbox |
| Error | API error toast, row reverts to previous state |
| No barbers | Shop expanded but has no barbers — empty nested section |

#### Section 6 — Validations & Error Handling

- Duration and price inputs accept the same ranges as the service form (hours 0-12, minutes 0-55, price ≥ 0)
- Cost input only saves on blur (prevents unnecessary API calls during typing)
- If a barber is deassigned while the shop is still assigned, the barber row becomes unchecked
- API errors on individual row saves show a toast and revert the row to its previous state

#### Section 7 — Analytics & Logging

No specific analytics events discovered for the assignments flow in source code.

#### Section 8 — Security & Privacy

- Shop users cannot toggle their own shop's assignment checkbox (always assigned)
- Barber users cannot access the assignments step at all
- Brand users have full control over all shops and barbers
- Each API call is individually authenticated with JWT

#### Section 9 — Performance / UX Budgets

- Barbers within a shop are loaded lazily (only when the shop row is expanded)
- Each field change triggers an immediate API call (no debouncing except for cost which saves on blur)
- Loading state shown per-row to indicate which entities are being saved

#### Section 10 — Dependencies / Remote Config / Feature Flags

No feature flags gate the assignments feature. It is always available for Shop and Brand users.

#### Section 11 — Open Questions & Assumptions

- **Mobile UX for assignments table:** Web Commander uses a complex expandable table with inline editing. Mobile may need a different UX pattern (e.g., list with drill-down into shop → barber, or separate screens per shop). Need design input.
- **Immediate save vs. batch save:** Web saves each row change immediately via API. Mobile may prefer a "Save All" pattern with optimistic updates for better UX on mobile networks. Need to decide.
- **Offline behavior:** If the user loses connectivity during assignment changes, some rows may save while others fail. Recovery strategy needs definition.

---

## 4. Backend API Reference

| Endpoint | Method | Used In | Include Parameter | Request Shape | Response Shape | Error Codes |
|----------|--------|---------|-------------------|---------------|----------------|-------------|
| `POST /brand/{brandId}/service` | POST | Create Service (Brand) | N/A | name, desc, localizedName, localizedDesc, serviceCategoriesId, duration, cost, enabled, requiresPrepaid, kioskEnabled, tax | Full service object | 400, 403, 422 |
| `POST /shop/{shopId}/service` | POST | Create Service (Shop) | N/A | Same as above | Full service object | 400, 403, 422 |
| `POST /shop/{shopId}/barber/{barberId}/service` | POST | Create Service (Barber) | N/A | Same as above | Full service object | 400, 403, 422 |
| `GET /shop/{shopId}/service/{serviceId}` | GET | Edit Service (Shop) | N/A | N/A | Full service object | 404 |
| `PUT /shop/{shopId}/service/{serviceId}` | PUT | Edit Service (Shop) | N/A | Same as Create + `tax[].deleted` | Updated service object | 400, 403, 404, 422 |
| `PUT /shop/{shopId}/barber/{barberId}/service/{serviceId}` | PUT | Edit Service (Barber) | N/A | Same as above | Updated service object | 400, 403, 404, 422 |
| `PUT /brand/{brandId}/service/{serviceId}` | PUT | Edit Service (Brand) | N/A | Same as above | Updated service object | 400, 403, 404, 422 |
| `DELETE /shop/{shopId}/service/{serviceId}` | DELETE | Delete Service (Shop) | N/A | None | 204 / `{ isSuccessful }` | 400, 403, 404 |
| `DELETE /shop/{shopId}/barber/{barberId}/service/{serviceId}` | DELETE | Delete Service (Barber) | N/A | None | 204 / `{ isSuccessful }` | 400, 403, 404 |
| `DELETE /brand/{brandId}/service/{serviceId}` | DELETE | Delete Service (Brand) | N/A | None | 204 / `{ isSuccessful }` | 400, 403, 404 |
| `GET /brand/{brandId}/service-category` | GET | Manage Categories (List) | N/A | limit, skip | `{ rows: [...], count }` | 403 |
| `POST /brand/{brandId}/service-category` | POST | Manage Categories (Add) | N/A | `{ name }` | `{ id, name }` | 400, 403, 422 |
| `PUT /brand/{brandId}/service-category/{id}` | PUT | Manage Categories (Edit) | N/A | `{ name }` | `{ id, name }` | 400, 403, 404, 422 |
| `DELETE /brand/{brandId}/service-category/{id}` | DELETE | Manage Categories (Delete) | N/A | None | 204 | 403, 404 |
| `POST /brand/{brandId}/service/{serviceId}/save-assignment` | POST | Service Assignments (Brand→Shop) | N/A | assignmentType, assigneeId, cost, duration, enabled, requiresPrepaid, kioskEnabled | Assignment object | 400, 403, 422 |
| `POST /shop/{shopId}/service/{serviceId}/save-assignment` | POST | Service Assignments (Shop→Barber) | N/A | assignmentType, assigneeId, cost, duration, enabled, requiresPrepaid | Assignment object | 400, 403, 422 |
| `POST /shop/{shopId}/service/{serviceId}/assign` | POST | Service Assignments (Bulk Assign Barbers) | N/A | `{ barbers: [id, ...] }` | Assignment result | 400, 403 |
| `DELETE /shop/{shopId}/service/{serviceId}` | DELETE | Service Assignments (Remove) | N/A | None | 204 | 403, 404 |
| `GET /brand/{brandId}/service` | GET | Service List (Brand) | `shops,category` | limit, skip, sortBy | `{ rows: [...], count }` | 403 |

---

## 5. Cross-Flow Decision Tables

| Condition | Create Service | Edit Service | Delete Service | Manage Categories | Service Assignments |
|-----------|---------------|--------------|----------------|-------------------|---------------------|
| `services.modify` permission missing | "Add Service" button hidden | Form loads read-only or hidden | Delete button hidden | Categories CRUD hidden (can still view) | Not accessible |
| User is Brand | Full form → "Next" → Assignments | Full edit access → "Next" → Assignments | Full delete access | Full CRUD access | Full control: assign to any shop/barber with overrides |
| User is Shop | Full form → "Next" → Assignments | Field locking based on service source → "Next" → Assignments | Full delete access | Read-only (filter/assign) | Own shop pre-assigned; can assign/override barbers only |
| User is Barber (rental, creator) | Full form → "Save" (no Assignments) | All fields editable → "Save" (no Assignments) | Can delete own services | Read-only (filter/assign) | **Not available** |
| User is Barber (non-rental or non-creator) | Full form → "Save" (no Assignments) | Only price, duration, visibility | Limited | Read-only (filter/assign) | **Not available** |
| User is Barber, service from Shop/Brand | N/A | Only price and duration | Cannot delete | Read-only | **Not available** |
| Service has no changes (edit) | N/A | Skip API call (pristine check) | N/A | N/A | N/A |
| Category deleted while assigned to services | N/A | Category field cleared or shows "None" | N/A | Services lose category (verify with backend) | N/A |

---

## 6. Data Scope & Visibility

| Data Entity | Brand User | Shop User | Barber User |
|-------------|------------|-----------|-------------|
| Services | All services across all shops | All services in their shop | Own services + shop/brand services assigned to them |
| Service Categories | Full CRUD (brand-scoped) | Read-only (filter & assign) | Read-only (filter & assign) |
| Service Assignments | Can assign to any shop/barber | Can assign to barbers in their shop | Cannot assign |
| Service Taxes | Can configure taxes | Can configure taxes | Can configure taxes on own services |

---

## 7. Edge-Case Inventory

- **Offline behavior:** Service creation/edit/delete should queue or fail gracefully with retry option when offline
- **Permission denied mid-flow:** If permissions change while user is on the form, the save should fail with a 403 and the user should be informed
- **Duplicate service names:** Web Commander does not enforce unique service names — duplicates are allowed
- **Zero-duration service:** Duration of 0 minutes is technically allowed by validation (min 0). Confirm if this is intentional or a bug
- **Zero-cost service:** A cost of 0 (free service) is valid
- **Category with assigned services deleted:** Behavior undefined — services may silently lose their category. Needs verification
- **Concurrent edits:** If two users edit the same service simultaneously, the last save wins. No conflict detection
- **Very long service name:** Min 3 chars enforced, but no max length found in Web Commander validation. Mobile should consider field length limits
- **Localization edge case:** If EN name is empty on edit, Web fills from first non-empty language. Mobile should handle this

---

## 8. Non-Functional Requirements (NFRs)

- **Reliability:** Standard REST API with retry on transient errors. No explicit retry policy found in source
- **Caching:** Services list query cached by React Query. Service categories: 5-min stale time, 30-min GC
- **Latency:** No explicit performance budgets defined in source. Standard expectation: form submission < 3s
- **Privacy:** No PII in service data. Service pricing may be commercially sensitive — access controlled by permissions

---

## 9. Migration Notes for Mobile Commander

- **Create/Edit/Delete Service:** Already implemented (✅ Done). See:
  - Screen: `src/screens/services/ServiceFormScreen/`
  - Mutations: `src/hooks/mutations/services/useCreateServiceMutation.ts`, `useUpdateServiceMutation.ts`, `useDeleteServiceMutation.ts`
  - API: `src/api/services.ts`
- **Manage Categories (⚠️ Partial):** Needs implementation:
  - **Existing:** `useServiceCategoriesQuery()`, `fetchServiceCategories()`, `createServiceCategory()` API function
  - **Missing:** Category management screen, update/delete API functions, mutation hooks for create/update/delete
  - **Pattern to follow:** React Query hooks in `src/hooks/queries/services/` and `src/hooks/mutations/services/`
  - **Component pattern:** 4-file structure (index.ts, Component.tsx, styles.ts, types.ts)
  - **Validation:** Zod schema with React Hook Form
  - **Styling:** react-native-unistyles with semantic theme tokens
  - **Navigation:** Add route to profile stack for category management

---

## 10. Glossary & Domain Vocabulary

| Term | Definition |
|------|-----------|
| Service | A bookable/chargeable unit (e.g., "Haircut", "Beard Trim") with name, duration, price |
| Service Category | A brand-scoped grouping for services (e.g., "Haircuts", "Coloring", "Treatments") |
| Assignment | Linking a service to specific shops and barbers with optional per-entity overrides for price and duration |
| Enabled / Online Visibility | Whether a service appears in online booking for customers |
| Kiosk Enabled | Whether a service appears on the in-shop kiosk for walk-in customers |
| Requires Prepaid | Whether a service requires prepayment at booking time |
| Rental Barber | A barber who rents a chair (independent contractor) — has more control over their own services |
| Commission Barber | A barber employed by the shop on commission — has limited control over services |
| Service Source | Who created the service: `.barber`, `.shop`, or `.brand` — determines edit permissions |
| Localized Name/Desc | Multi-language support for service name and description (EN, FR, ES) |

---

## 11. Coverage Summary

- [x] All user-specified flows documented (Create, Edit, Delete, Manage Categories, Service Assignments)
- [x] Related sub-flows identified (Assignments fully documented)
- [x] All three user types covered (Brand, Shop, Barber)
- [x] All API endpoints catalogued
- [x] All validation rules captured (client and server)
- [x] Edge cases inventoried
- [x] Open questions listed with verification steps
- [x] iOS differences noted where applicable
- [x] Mobile Commander implementation status classified per flow
