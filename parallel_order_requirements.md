# Parallel Order Screen Requirements

This document tracks the exact user needs, logic specifications, and requirements for each field, panel, and button in the Parallel Order Screen. 

## 1. Global Order Fields (Header Section)

### Order No. Field
**User Intent & Requirements:**
- **Read-Only vs Editable:** The Order No. field must be **Read-Only** when *Adding* a new parallel order. However, it must be **Editable** when *Editing* an existing parallel order.
- **Manual Override:** If a user manually edits the Order No. during the Edit phase, the system must accept that manually entered number as the absolute truth and sync it exactly as typed.
- **Prefixes:** No special prefixes are required for different users/devices. Standard numeric sequencing is perfect.
- **Offline UI:** When a tablet is disconnected from the Admin server, the UI **must** clearly show the `OFFLINE-` prefix (e.g., `OFFLINE-1502`). This serves as a visual indicator to the salesman that the order is pending synchronization and has not yet received a permanent server-allocated ID.

## 2. Customer & Family Management

### Phone Number & Customer Selection
**User Intent & Requirements:**
- **Strictly Required:** An order cannot be submitted without a valid phone number. A phone number dictates who the order belongs to.
- **Updating vs Creating:** If a user selects an existing customer (e.g., Rahul) and changes the phone number, the system needs to distinguish between:
  - *Scenario A:* Rahul got a new phone number (update existing database row).
  - *Scenario B:* This is a completely different person (create a new database row).
  *(Note: Specific buttons exist for "Create New" vs "Edit" to handle this clearly).*

### Family Members (Sub-Customers) & Session Architecture
**User Intent & Requirements:**
- **The "Parallel" Concept:** A "Parallel Order" is NOT a single giant order with sub-sections. It is a "Session" that creates multiple, completely independent orders (one for each family member).
- **No Global Fields:** The ONLY global element on the screen is the tab switcher at the top (the session progress header). Everything else—Delivery Date, Advance Amount, Notes, Total Amount—is strictly per-tab (per-order). 
- **Live Reactivity:** If a sub-customer's name is updated, that name change must reflect immediately in the UI (e.g., automatically renaming their respective Tab).
- **Tab Deletion (Nuke):** If a salesman accidentally clicks to delete an entire populated tab (with products/measurements), the system must delete it instantly without a confirmation warning popup. It completely relies on the Global Undo button to save the user from a misclick.

## 3. Product Addition & Details (Per Tab)

### Measurement Auto-Fill & Updates
**User Intent & Requirements:**
- **Historical Consistency:** When a customer has past orders, their previous measurements are fetched and auto-filled. If the salesman changes a measurement (e.g., changes Waist from 30 to 32), this new measurement (32) becomes the default for *future* orders.
- **Immutability of Past Orders:** The old measurement (30) attached to the historical order MUST NOT be altered. Historical orders preserve the customer's measurements exactly as they were at that specific point in time.

### Component Isolation & Blank Fields
**User Intent & Requirements:**
- **Partial Measurements Allowed:** If a product contains multiple components (e.g., a "Suit" with a Shirt and Pant), the salesman is fully allowed to submit the order even if component measurement fields are partially or completely blank. The system must not enforce strict validation requiring every single measurement field to be filled.

### Product Type Changes & Deletion
**User Intent & Requirements:**
- **Dropdown Changes (Wipe Prevention):** If a user changes a product type via the dropdown (e.g., from Lehenga to Suit) to fix a mistake, the system MUST NOT clear the drawing canvas or the uploaded images. It must also retain the entered measurements and map them to the relevant fields of the newly selected product type.
- **Delete & Re-add:** If a user explicitly clicks the "Delete Product" (trash) icon and then clicks "Add Product", the new product must be completely fresh (no drawings, no images). However, it MUST automatically fetch the measurements from the last entered product (the one just deleted) to save the salesman time from re-typing them.

## 4. Drawing Canvas & Attachments
*To be documented based on discussion...*

## 5. Invoice & Final Submission (Footer Section)
**User Intent & Requirements:**
- **Tab Independence:** Each tab represents a distinct, independent order. The "Total Amount" calculated and displayed is strictly for that single tab/order.
- **No Aggregation:** Totals are **never** aggregated or mixed across multiple tabs (e.g., Tab A and Tab B do not combine into a single mega-invoice). They remain completely isolated financially.

## 6. Drafts & Auto-Save Mechanics
**User Intent & Requirements:**
- **Aggressive Overwrite:** There is intentionally NO "Discard Changes" feature. The system must aggressively auto-save and overwrite the current draft in the background at all times. 
- **Graceful Interruption Handling:** Because salesmen frequently back out of screens abruptly or force-close the app, continuous aggressive auto-saving is the primary strategy to ensure zero data loss. If a salesman makes a mistake, they must rely on the Undo system to fix it, rather than attempting to abort the draft session.
- **72-Hour Cleanup:** The 72-hour wipe applies blindly and aggressively to ALL drafts. Drafts are strictly ephemeral workspaces. If a customer is delaying payment for days, the salesman must submit the order (which can be cancelled later via the Order Details panel), rather than leaving it in Drafts indefinitely.

## 7. Undo/Redo System
**User Intent & Requirements:**
- **Canvas Protection (Recommendation):** The Global Undo button must **ignore** Canvas save events. Because canvas drawings require significant manual effort and have their own internal undo buttons, a global undo (used to fix a typo in a text field) should never accidentally wipe out a complex drawing that was just saved. Global Undo should strictly affect text, measurements, and numerical inputs.

## 8. Order Submission Pipeline (Background Processes)
**User Intent & Requirements:**
When a user clicks "Submit Order" for a specific tab, the exact execution pipeline must strictly follow this order:
1. **Validation Check:** The system verifies that a Customer/Phone Number is selected and that all entered products have a quantity > 0 and price > 0. (Blank component measurements are ignored).
2. **Order No. Allocation:** If the `order_no` is currently empty, the system generates an order number (either fetching the true sequential number if the admin server is reachable, or defaulting to an `OFFLINE-` prefix if unreachable).
3. **Local Database Write (Crucial Step):** The order and its products are instantly serialized and inserted into the tablet's local SQLite database (`orders` and `order_details` tables).
4. **Silent Offline Success:** If the device is offline, the UI **must not** throw any blocking popups or warnings about the network. The save to the local SQLite database guarantees the data is safe. The UI must silently succeed (e.g., showing a standard green checkmark) and immediately allow the user to continue working.
5. **Background Sync Trigger:** The app triggers the `deviceSyncClient.forceSync()` process in the background. If Wi-Fi is available, this background worker pushes the SQLite mutations to the Admin server and pulls any allocated true order numbers. If Wi-Fi is dead, it queues the sync silently.
6. **Session Advancement:** If the user is in a multi-member session (e.g., 3 tabs), the submitted tab is marked as "Done" and the UI seamlessly slides to the next tab without returning to the home screen. Only when the *final* tab is submitted does the screen exit.

## 9. Interactive Elements & Button Pipelines
**User Intent & Requirements:**

### The "Add Customer" Button (Person Add Icon)
- **Execution Pipeline:** Opens a dialog -> Validates name & phone -> Inserts the new customer directly into the `customers` SQLite table -> Inserts them into the `family_members` table -> Auto-selects this customer for the current order -> Triggers an explicit snapshot for Undo history.

### The "Add Product" Button
- **Execution Pipeline:** Triggers the dropdown -> Creates a blank product skeleton (`price: 0, qty: 1`) -> Associates the correct `identity_key` -> Fetches any historical measurements from past orders for that specific customer/product combo -> Refreshes the UI -> Triggers a Live Auto-Save to the SQLite Drafts.

### The "Delete Product" Button (Trash Icon)
- **Execution Pipeline:** Instantly removes the product array from memory without any blocking confirmation dialogues -> Recalculates the Tab's Total Amount -> Pushes the deletion to the Undo stack -> Triggers a Live Auto-Save.

### The "View Past Orders" Button
- **Execution Pipeline:** Opens a bottom sheet -> Queries the `orders` SQLite table for the selected customer ID -> If the Family Name is "Self", it filters orders where `family_name` is null. If it's a sub-customer (e.g., Sushila), it filters by her exact string name -> Renders historical read-only cards.

### The "Undo" Button
- **Execution Pipeline:** Forces an immediate commit of any currently typing text -> Pops the last saved JSON string from the `_undoStack` -> Pushes the current state to the `_redoStack` -> Completely replaces the UI's product map, text controllers, and global fields with the restored JSON -> Re-requests focus on the text field that was just restored -> Ignores Canvas drawings (per Section 7).

### The "Add Photo" Button
- **Execution Pipeline:** Requests OS permissions (handles denial gracefully without crashing) -> Triggers device ImagePicker -> Reads raw image bytes -> Encodes them directly into Base64 strings -> Attaches to the `_productImages` array in memory -> Triggers a Live Auto-Save to SQLite (without compression, supporting maximum resolution).

### The "UPI QR" Button (Invoice Footer)
- **Execution Pipeline:** Reads the current value of the "Advance Amount" field -> Dynamically generates a UPI QR code string including the `transactionRef` (Order No) and `transactionNote` -> Renders the QR code visually for the customer to scan.
