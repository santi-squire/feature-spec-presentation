# Services Management — Jira Tasks

> **Generated from:** `docs/services_management_spec.md`
> **Date:** 2026-02-24
> **Epic:** Services Management for Mobile Commander

---

## TASK: Create Service

### Key Details

| Field | Value |
|:------|:------|
| **Type** | Story |
| **Priority** | P1 |
| **Status** | ✅ Done |
| **Sprint** | TBD |

### Description

Users can create a new service from the Services list screen. A service represents a bookable unit (e.g., "Haircut", "Beard Trim") with a name, description, duration, price, optional category, optional taxes, and visibility settings. After filling in the form and saving, the new service appears in the services list and becomes available for booking.

**Entry Point:** Services List Screen → "Add Service" button (top-right)
**Flow:** Tap "Add Service" → Fill in service details → Save → Success message → Return to services list

### Roles & Permissions

| User Type | Access | Restrictions |
|:----------|:-------|:-------------|
| Brand User | Can create services for any shop. Sees the Assignments step after creation to assign the service to specific shops and barbers. | None |
| Shop User (Owner/Manager) | Can create services for their shop. Sees the Assignments step to assign to barbers within the shop. | None |
| Barber User (Staff) | Can create services for themselves only. | Cannot see the Assignments step. Cannot see the "Kiosk Enabled" toggle. Commission barbers have limited editing ability after creation (name and description become read-only). |

The "Add Service" button is only visible to users with the **Modify Services** permission. Users without this permission do not see the button at all.

### Settings & Feature Flags

| Setting / Flag | Effect |
|:---------------|:-------|
| **Payment Deposits Enabled** | When enabled, the "Requires Prepaid" toggle appears on the form. When disabled, this toggle is hidden entirely. |
| **Kiosk Mode** | The "Kiosk Enabled" toggle controls whether the service appears on in-shop kiosks for walk-in customers. This toggle is hidden for barber users. |

### Acceptance Criteria

- [ ] The "Add Service" button is visible only for users with the Modify Services permission
- [ ] The form opens with all fields empty/at default values
- [ ] The user can enter a service name (required, at least 3 characters)
- [ ] The user can enter a description (optional, up to 300 characters)
- [ ] The user can set duration using hours (0-12) and minutes (0-55, in 5-minute increments)
- [ ] The user can set a price (required, must be 0 or greater)
- [ ] The user can optionally select one category from available categories
- [ ] The user can optionally select one or more taxes
- [ ] The "Enabled" toggle defaults to on (service visible for online booking)
- [ ] The "Requires Prepaid" toggle is only shown when the Payment Deposits setting is enabled
- [ ] The "Kiosk Enabled" toggle is hidden for barber users
- [ ] The Save button is disabled until all required fields are valid
- [ ] On save, a success message ("Service saved successfully") is shown
- [ ] After saving, the user is returned to the services list and the new service appears
- [ ] If an error occurs during save, an error message is shown and the form remains editable
- [ ] Multi-language support: service name and description can be entered in English, Spanish, and French

### Release Notes

Users can create new services with name, description, duration, pricing, and category from the Services Management screen.

### Recommended Integration Testing

| Test Case | Steps | Expected Result |
|:----------|:------|:----------------|
| Happy path — create service | Tap "Add Service" → enter name "Haircut", price $25, duration 30 min → Save | Service created, appears in list, success message shown |
| Validation — name too short | Enter name "Hi" (2 characters) → try to save | Validation error appears on the name field |
| Validation — missing price | Leave price empty → try to save | Validation error appears on the price field |
| Permission check — no access | Login as user without Modify Services permission → go to Services List | "Add Service" button is not visible |
| Barber — kiosk hidden | Login as barber → open Add Service form | "Kiosk Enabled" toggle is not shown |
| Prepaid flag disabled | With Payment Deposits setting disabled → open Add Service form | "Requires Prepaid" toggle is not shown |
| Category selection | Open Add Service → select a category → save | Service created with the selected category |
| Free service | Enter price as $0 → save | Service created with $0 price |
| Save error | Trigger a server error during save | Error message shown, form stays open and editable |
| Brand user — assignments | Login as brand user → create service → save | After saving, the Assignments step is available |
| Barber user — no assignments | Login as barber → create service → save | Redirected directly to services list (no Assignments step) |

### Testing Notes

- Test with all three user types: Brand, Shop, and Barber
- Barber users should NOT see the "Kiosk Enabled" toggle
- Toggle the Payment Deposits setting on/off and verify the "Requires Prepaid" toggle visibility changes
- If multi-language is supported, test entering service names in English, Spanish, and French
- Verify that a newly created service appears in the services list immediately after saving

### Release Info

- **Target Release:** TBD
- **Feature Flag:** Payment Deposits (for prepaid toggle only)
- **Rollout:** TBD

### Associations

- **Epic:** Services Management
- **Depends On:** None
- **Blocks:** None
- **Related:** Edit Service, Delete Service

---

## TASK: Edit Service

### Key Details

| Field | Value |
|:------|:------|
| **Type** | Story |
| **Priority** | P1 |
| **Status** | ✅ Done |
| **Sprint** | TBD |

### Description

Users can edit an existing service by tapping on it from the services list. The form opens pre-filled with the service's current values. The key behavior in this flow is **field-level access control**: depending on who created the service and the user's role, certain fields may be locked (read-only) while others remain editable. If no changes are made, saving does nothing.

**Entry Point:** Services List Screen → Tap on a service card
**Flow:** Tap service → Form loads with current values → Edit desired fields → Save → Success message → Return to services list

### Roles & Permissions

| User Type | Service Created By | Editable Fields | Locked Fields |
|:----------|:-------------------|:----------------|:--------------|
| Brand User | Anyone | All fields | None |
| Shop User | Their own shop | All fields | None |
| Shop User | Brand (multi-shop) | Price, Duration, Visibility | Name, Description |
| Barber (rental shop, creator) | Themselves | All fields | None |
| Barber (non-rental or non-creator) | Themselves | Price, Duration, Booking Visibility | Name, Description, Category, Taxes, Kiosk |
| Barber | Shop or Brand | Price, Duration only | Everything else |

Locked fields are displayed as read-only with a visual indicator (e.g., greyed out or lock icon). The user cannot interact with locked fields.

The "Delete" button is also visible in edit mode — see the Delete Service task for details.

### Settings & Feature Flags

| Setting / Flag | Effect |
|:---------------|:-------|
| **Payment Deposits Enabled** | Controls visibility of the "Requires Prepaid" toggle, same as in Create. |
| **Rental Shop Status** | Affects which fields barbers can edit. Barbers in rental shops who created the service have full edit access. |

### Acceptance Criteria

- [ ] Tapping a service from the list opens the edit form with all fields pre-filled
- [ ] The correct fields are locked/editable based on the user's role and who created the service (see Roles & Permissions table)
- [ ] Locked fields are visually distinct (read-only appearance)
- [ ] The Save button is disabled when no changes have been made
- [ ] After making changes and saving, a success message ("Service saved successfully") is shown
- [ ] The services list reflects the updated values after saving
- [ ] If no changes are made, tapping Save does not trigger a save operation
- [ ] The "Delete" / "Remove Service" button is visible at the bottom of the edit form
- [ ] When taxes are removed during edit, the service is updated correctly (removed taxes are no longer associated)
- [ ] Error messages are shown for validation failures or server errors
- [ ] If the service no longer exists (deleted by another user), an error message is shown and the user is redirected to the services list

### Release Notes

Users can edit existing service details including name, description, duration, pricing, and category. Access to fields depends on the user's role and who originally created the service.

### Recommended Integration Testing

| Test Case | Steps | Expected Result |
|:----------|:------|:----------------|
| Happy path — edit price | Tap existing service → change price from $25 to $30 → Save | Service updated, list shows new price |
| No changes — save | Open edit → make no changes → Save | Nothing happens, no save triggered |
| Field locking — barber, shop-created | Login as barber → tap a service created by the shop | Only price and duration fields are editable; all others are locked |
| Field locking — brand user | Login as brand user → tap any service | All fields are editable |
| Field locking — rental barber, own service | Login as rental barber → tap own service | All fields are editable |
| Field locking — non-rental barber, own service | Login as non-rental barber → tap own service | Only price, duration, and booking visibility are editable |
| Remove a tax | Edit service → remove a tax → Save | Service saved without the removed tax |
| Change category | Edit service → change category → Save | Service saved with new category |
| Service deleted by another user | Try to edit a service that was just deleted | Error message shown, redirected to services list |

### Testing Notes

Test field locking thoroughly with these combinations:
1. Brand user editing any service (all fields editable)
2. Shop user editing their own shop's service (all fields editable)
3. Shop user editing a brand-level service in a multi-shop brand (name/description locked)
4. Rental barber editing their own service (all fields editable)
5. Non-rental barber editing their own service (limited fields)
6. Barber editing a shop-created or brand-created service (only price and duration)

Each combination should show a different set of editable vs locked fields.

### Release Info

- **Target Release:** TBD
- **Feature Flag:** None
- **Rollout:** TBD

### Associations

- **Epic:** Services Management
- **Depends On:** Create Service (shared form)
- **Blocks:** None
- **Related:** Create Service, Delete Service

---

## TASK: Delete Service

### Key Details

| Field | Value |
|:------|:------|
| **Type** | Story |
| **Priority** | P2 |
| **Status** | ✅ Done |
| **Sprint** | TBD |

### Description

Users can delete a service from the edit form. A confirmation dialog appears before the service is permanently removed. The delete option is only available when editing an existing service — it is not shown on the create form. Deletion is permanent and cannot be undone.

**Entry Point:** Service Edit Form → "Remove Service" button (bottom of form, red/destructive style)
**Flow:** Tap "Remove Service" → Confirmation dialog appears with service name → Confirm deletion → Service removed → Success message → Return to services list

### Roles & Permissions

| User Type | Access | Restrictions |
|:----------|:-------|:-------------|
| Brand User | Can delete any service | None |
| Shop User (Owner/Manager) | Can delete services in their shop | None |
| Barber User (Staff) | Can delete their own services | Cannot delete services created by the shop or brand |

The delete button is only visible to users with the **Modify Services** permission.

### Settings & Feature Flags

None.

### Acceptance Criteria

- [ ] The "Remove Service" button is visible only when editing an existing service (not on the create form)
- [ ] The button has a red/destructive visual style
- [ ] Tapping the button shows a confirmation dialog that includes the service name
- [ ] The confirmation dialog has a "Delete" (destructive) button and a "Cancel" button
- [ ] Tapping "Cancel" dismisses the dialog without any changes
- [ ] Tapping "Delete" removes the service and shows a loading indicator during the operation
- [ ] After successful deletion, a success message ("Service is deleted") is shown
- [ ] After deletion, the user is navigated back to the services list
- [ ] The deleted service no longer appears in the services list
- [ ] If the deletion fails (server error), an error message is shown and the service remains unchanged
- [ ] If the service was already deleted by another user, the error is handled gracefully

### Release Notes

Users can delete services from the edit screen with a confirmation step. Deletion is permanent.

### Recommended Integration Testing

| Test Case | Steps | Expected Result |
|:----------|:------|:----------------|
| Happy path — delete service | Open edit form → tap "Remove Service" → Confirm | Service deleted, removed from list, success message shown |
| Cancel deletion | Open edit form → tap "Remove Service" → Cancel | Dialog dismissed, service unchanged |
| Button hidden on create | Open "Add Service" form | "Remove Service" button is not visible |
| Permission check | Login as user without Modify Services permission | Edit form inaccessible or delete button hidden |
| Barber — shop service | Login as barber → edit a shop-created service | Delete button should not be available (or behaves per permission rules) |
| Server error during delete | Trigger server error → confirm delete | Error message shown, service remains in list |
| Already deleted | Another user deletes the service → confirm delete | Error handled gracefully, redirected to services list |

### Testing Notes

- Test with all user types (Brand, Shop, Barber)
- Verify the confirmation dialog displays the correct service name
- After deletion, refresh the services list to confirm the service is truly gone (not just hidden locally)
- Test the scenario where two users try to delete the same service simultaneously

### Release Info

- **Target Release:** TBD
- **Feature Flag:** None
- **Rollout:** TBD

### Associations

- **Epic:** Services Management
- **Depends On:** Edit Service (delete is accessed from the edit form)
- **Blocks:** None
- **Related:** Edit Service

---

## TASK: Manage Categories

### Key Details

| Field | Value |
|:------|:------|
| **Type** | Story |
| **Priority** | P2 |
| **Status** | ⚠️ Partial |
| **Sprint** | TBD |

### Description

Service categories allow Brand users to organize services into groups (e.g., "Haircuts", "Coloring", "Treatments"). This task covers the full category management experience: viewing the list of categories, adding new categories, editing existing category names, and deleting categories.

Categories are **brand-scoped** — they are managed at the brand level and shared across all shops in the brand. Only Brand users can create, edit, or delete categories. Shop and Barber users can see and use categories (for filtering the services list and assigning categories to services), but they cannot modify the category list itself.

**Currently:** Categories can be viewed and used for filtering/assigning, but there is no dedicated screen for managing (adding, editing, deleting) categories.

**Entry Point:** Services List Screen → "Categories" management button (visible to Brand users only)
**Flow:** Tap "Categories" → View category list → Add / Edit / Delete categories → Changes reflected in services list and service form

### Roles & Permissions

| User Type | Access | What They Can Do |
|:----------|:-------|:-----------------|
| Brand User | Full access to category management | View list, add new categories, edit category names, delete categories |
| Shop User (Owner/Manager) | View only | Can see categories as filter pills on the services list and as options in the service form. Cannot add, edit, or delete categories. |
| Barber User (Staff) | View only | Same as Shop User — can use categories for filtering and assigning but cannot manage them. |

The "Categories" management button is **hidden** for Shop and Barber users.

### Settings & Feature Flags

None — category management is always available for Brand users.

### Acceptance Criteria

**Category List:**
- [ ] A "Categories" button/link is visible on the Services List screen for Brand users only
- [ ] Shop and Barber users do not see the "Categories" management button
- [ ] The category list screen displays all categories with their names
- [ ] When no categories exist, an empty state message is shown
- [ ] An "Add Category" button is available on the list screen

**Add Category:**
- [ ] Tapping "Add Category" opens a form with a single "Name" field
- [ ] The name field is required and must be at least 2 characters
- [ ] After saving, the new category appears in the category list
- [ ] After adding a category, it becomes available in the service form's category picker and as a filter on the services list

**Edit Category:**
- [ ] Each category in the list has an edit action
- [ ] Tapping edit opens the form pre-filled with the current category name
- [ ] After saving changes, the updated name is reflected in the category list, service form picker, and services list filters

**Delete Category:**
- [ ] Each category in the list has a delete action
- [ ] Tapping delete shows a confirmation dialog
- [ ] After confirming, the category is removed from the list
- [ ] Services that were assigned to a deleted category lose their category assignment

**General:**
- [ ] If more than 50 categories exist, additional categories are loaded automatically as the user scrolls
- [ ] Validation errors are shown when the category name is too short (less than 2 characters)

### Release Notes

Brand users can now manage service categories — add, rename, and delete categories used to organize services.

### Recommended Integration Testing

| Test Case | Steps | Expected Result |
|:----------|:------|:----------------|
| Happy path — add category | Open categories → "Add Category" → enter "Haircuts" → Save | Category created, appears in list |
| Happy path — edit category | Tap edit on a category → change name to "Hair Services" → Save | Category name updated in list |
| Happy path — delete category | Tap delete on a category → Confirm | Category removed from list |
| Validation — name too short | Add category with name "A" (1 character) → Save | Validation error: name must be at least 2 characters |
| Empty state | View category list when no categories exist | Empty state message displayed |
| Pagination | Have 60+ categories → scroll through list | All categories load across pages |
| Permission — shop user | Login as shop user → go to Services List | "Categories" management button is not visible |
| Permission — barber user | Login as barber → go to Services List | "Categories" management button is not visible |
| Category in service form | Add a new category → open the service create/edit form | New category appears in the category picker |
| Category in services list filter | Add a new category → go to services list | New category appears as a filter pill |
| Delete category with services | Delete a category that has services assigned → view those services | Services no longer show the deleted category |
| Rename category with services | Rename a category → view services assigned to it | Services show the updated category name |

### Testing Notes

- Test exclusively with Brand user accounts for management actions
- After every category change (add/edit/delete), verify the impact in two places: the services list filter pills and the service form category picker
- Test what happens to services when their assigned category is deleted — they should lose the category assignment gracefully
- Test with a large number of categories (50+) to verify pagination/infinite scroll works correctly
- Verify that Shop and Barber users cannot access the category management screen even by navigating directly

### Release Info

- **Target Release:** TBD
- **Feature Flag:** TBD
- **Rollout:** TBD

### Associations

- **Epic:** Services Management
- **Depends On:** None
- **Blocks:** None
- **Related:** Create Service, Edit Service (category picker)

---

## TASK: Optional Service Assignments

### Key Details

| Field | Value |
|:------|:------|
| **Type** | Story |
| **Priority** | P0 |
| **Status** | ❌ To Do |
| **Sprint** | TBD |

### Description

After creating or editing a service, Shop and Brand users are taken to an optional Assignments step where they can assign the service to specific shops and barbers. Each assignment can have its own overrides for price, duration, online visibility, prepayment requirement, and kiosk availability — allowing a single service to have different settings per location and per barber.

This step is automatically shown when a Shop or Brand user saves a service. Barber users never see this step — they go directly back to the services list.

The assignment step is **functionally optional**: the user can leave without making any assignments, or assign the service to as many shops and barbers as needed.

**Entry Point:** Automatic redirect after saving service details (for Shop and Brand users only). The "Save" button on the service form shows as "Next" for these users.
**Flow:**
1. User saves service details (create or edit) → automatically taken to Assignments step
2. **Brand users** see a list of all shops (none pre-selected for new services). They check shops to assign the service. When a shop is checked, all barbers in that shop are automatically assigned.
3. **Shop users** see their own shop (pre-selected for new services) with their barbers listed. They can assign/unassign individual barbers.
4. For each assigned shop or barber, the user can override: price, duration, online visibility, prepayment requirement, and kiosk mode (kiosk is shop-level only).
5. Changes are saved immediately as the user makes them (no need for a final "Save All" action).
6. User taps "Save" or navigates back to finish.

### Roles & Permissions

| User Type | Access | Restrictions |
|:----------|:-------|:-------------|
| Brand User | Full control — can assign/unassign any shop and any barber within those shops. Can override all fields (price, duration, visibility, prepaid, kiosk) for each entity. | None |
| Shop User (Owner/Manager) | Can assign/unassign barbers within their own shop. Can override all barber-level fields. | Cannot toggle their own shop's assignment (it is always assigned). Cannot modify shop-level overrides for their own shop. |
| Barber User (Staff) | **No access** — this step is completely hidden for barber users. | The service form shows "Save" (not "Next") and returns the barber directly to the services list. |

### Settings & Feature Flags

None — the Assignments step is always available for Shop and Brand users. No feature flag gates this flow.

### Acceptance Criteria

**Navigation & Visibility:**
- [ ] After saving a service (create or edit), Shop and Brand users are automatically taken to the Assignments step
- [ ] The service details form shows "Next" (not "Save") for Shop and Brand users
- [ ] Barber users see "Save" on the service form and are returned to the services list without seeing the Assignments step
- [ ] The user can navigate back from the Assignments step to re-edit service details

**Brand User — Shop Assignment:**
- [ ] Brand users see a list of all shops in their brand
- [ ] When creating a new service, no shops are pre-selected
- [ ] When editing an existing service, previously assigned shops are shown as selected
- [ ] Checking a shop assigns the service to that shop with the service's default values
- [ ] Checking a shop automatically assigns all barbers within that shop
- [ ] Unchecking a shop removes the assignment from that shop and all its barbers

**Shop User — Barber Assignment:**
- [ ] Shop users see their own shop as always-assigned (checkbox disabled / always checked)
- [ ] When creating a new service, their shop is pre-assigned with service defaults
- [ ] Shop users can expand their shop to see all barbers
- [ ] Individual barbers can be assigned or unassigned independently
- [ ] Each barber's fields can be overridden independently

**Per-Entity Overrides:**
- [ ] The user can override the **price** for each assigned shop or barber (currency input)
- [ ] The user can override the **duration** for each assigned shop or barber (hours + minutes dropdowns)
- [ ] The user can toggle **online visibility** per shop or barber
- [ ] The user can toggle **requires prepaid** per shop or barber
- [ ] The user can toggle **kiosk mode** at the shop level only (not shown for barber rows)
- [ ] Override values cascade: if a barber has no override, the shop's value is used; if the shop has no override, the service default is used

**Expandable Shop Rows:**
- [ ] Shops can be expanded to show their barbers
- [ ] Barbers are loaded when a shop is expanded (loading indicator shown during fetch)
- [ ] Each barber row shows the barber name and override fields

**Saving Behavior:**
- [ ] Each field change is saved immediately (no batch save needed)
- [ ] A loading indicator is shown on the affected row while the change is being saved
- [ ] If a save fails, an error message is shown and the row reverts to its previous value
- [ ] Tapping "Save" / navigating away completes the flow and returns to the services list

### Release Notes

Shop and Brand users can now assign services to specific shops and barbers with customized pricing, duration, and visibility settings per location and per barber.

### Recommended Integration Testing

| Test Case | Steps | Expected Result |
|:----------|:------|:----------------|
| Brand — create and assign | Brand user creates a service → reaches Assignments → checks two shops → saves | Service assigned to both shops and all their barbers |
| Brand — no assignment | Brand user creates a service → reaches Assignments → saves without checking any shops | Service created with no shop assignments |
| Brand — override price | Brand user assigns a shop → overrides barber price to $50 → saves | Barber's price is $50, shop retains service default |
| Shop — auto-assigned | Shop user creates a service → reaches Assignments | Own shop is pre-selected and cannot be unchecked |
| Shop — assign barbers | Shop user expands shop → assigns 2 of 3 barbers | Only selected barbers are assigned |
| Shop — override duration | Shop user overrides a barber's duration to 45 min | Barber shows 45 min, other barbers keep service default |
| Barber — no assignments step | Barber user creates a service → saves | Returns to services list directly, no Assignments step |
| Edit — existing assignments | Shop user edits service with existing assignments → Assignments step | Existing assignments are shown, can be modified |
| Expand shop — loading | Brand user clicks to expand a shop | Loading spinner shown while barbers load, then barber list appears |
| Toggle kiosk — shop only | Check that kiosk toggle appears for shop rows | Kiosk toggle visible on shop rows, hidden on barber rows |
| Error handling | Trigger API error while saving an assignment | Error message shown, row reverts to previous value |
| Shop user — cannot toggle own shop | Shop user tries to uncheck their own shop | Checkbox is disabled or not interactive |

### Testing Notes

- Test with **Brand** and **Shop** user accounts — each sees different behavior
- For Brand users, test with brands that have multiple shops and barbers per shop
- For Shop users, test with shops that have varying numbers of barbers (0, 1, many)
- Verify the cascade logic: if a shop has overridden price and a barber has no override, the barber should display the shop's price (not the service default)
- Test the flow from both **Create** (new service → Assignments) and **Edit** (existing service → Assignments) entry points
- Verify that changes made in the Assignments step persist: go back to the services list, then re-open the same service — assignments should still be there
- Test rapid field changes to verify immediate-save behavior handles concurrency correctly
- Barber users should be tested to confirm they **never** see the Assignments step under any circumstances

### Release Info

- **Target Release:** TBD
- **Feature Flag:** TBD
- **Rollout:** TBD

### Associations

- **Epic:** Services Management
- **Depends On:** Create Service, Edit Service (assignments step is reached from the service form)
- **Blocks:** None
- **Related:** Create Service, Edit Service, Manage Categories
