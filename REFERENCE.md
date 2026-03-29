# Verification Reference -- Detailed Checklists & Examples

Concrete examples and grep commands for every verification perspective.

## Plan Decomposition Example

Given a plan for an "Order Management" section:

```
Plan says:
- Orders list page with filters (status, date range, customer)
- Order detail page showing items, totals, customer info
- Create order flow (select customer -> add items -> review -> confirm)
- Edit order (only draft status)
- Cancel order (with reason)
- Order status workflow: draft -> pending -> processing -> shipped -> delivered
- Admin can manage all orders, staff can only view assigned orders
- Export orders to CSV
- Dashboard widget showing order stats

Verification matrix:
| # | Plan Item | Route | Controller | Component | Status |
|---|---|---|---|---|---|
| 1 | Orders list | GET /orders | OrderController@index | OrdersPage | ? |
| 2 | Order filters | GET /orders?status=X | OrderController@index | OrderFilters | ? |
| 3 | Order detail | GET /orders/{id} | OrderController@show | OrderDetail | ? |
| 4 | Create order | POST /orders | OrderController@store | CreateOrder | ? |
| 5 | Edit order | PUT /orders/{id} | OrderController@update | EditOrder | ? |
| 6 | Cancel order | POST /orders/{id}/cancel | OrderController@cancel | CancelModal | ? |
| 7 | Status workflow | - | OrderStatusService | StatusBadge | ? |
| 8 | Permissions | - | OrderPolicy | middleware | ? |
| 9 | CSV export | GET /orders/export | OrderController@export | ExportButton | ? |
| 10 | Dashboard widget | GET /dashboard/stats | DashboardController | OrderStats | ? |
```

---

## Data Contract Audit Example

**Backend controller returns:**
```php
// OrderController@show
return response()->json([
    'data' => [
        'id' => $order->id,
        'reference_number' => $order->reference_number,
        'status' => $order->status,
        'total_amount' => $order->total_amount,  // integer (cents)
        'customer' => [
            'id' => $order->customer->id,
            'full_name' => $order->customer->full_name,
            'email' => $order->customer->email,
        ],
        'items' => $order->items->map(fn ($item) => [
            'id' => $item->id,
            'product_name' => $item->product->name,
            'quantity' => $item->quantity,
            'unit_price' => $item->unit_price,
            'line_total' => $item->line_total,
        ]),
        'created_at' => $order->created_at->toISOString(),
    ],
]);
```

**Frontend component accesses:**
```tsx
// OrderDetail.tsx
<h1>Order #{order.referenceNumber}</h1>        // FAIL: referenceNumber vs reference_number
<StatusBadge status={order.status} />           // PASS
<p>Total: ${order.totalAmount / 100}</p>        // FAIL: totalAmount vs total_amount
<p>Customer: {order.customer.name}</p>          // FAIL: name vs full_name
<p>Email: {order.customer.email}</p>            // PASS
{order.items.map(item => (
  <div>
    <span>{item.productName}</span>             // FAIL: productName vs product_name
    <span>{item.qty} x ${item.price}</span>     // FAIL: qty vs quantity, price vs unit_price
    <span>${item.total}</span>                  // FAIL: total vs line_total
  </div>
))}
<time>{order.createdAt}</time>                  // FAIL: createdAt vs created_at
```

**Audit result: 2 PASS, 7 FAIL -- CRITICAL**

---

## Grep Commands for Common Issues

```bash
# Dummy data and placeholders
grep -rn "lorem" --include="*.{php,tsx,jsx,ts,js,vue,blade.php}" src/ app/ resources/
grep -rn "dummy" --include="*.{php,tsx,jsx,ts,js,vue,blade.php}" src/ app/ resources/
grep -rn "placeholder" --include="*.{php,tsx,jsx,ts,js,vue,blade.php}" src/ app/ resources/
grep -rn "fake\|sample\|mock" --include="*.{php,tsx,jsx,ts,js,vue,blade.php}" src/ app/ resources/
grep -rn "Coming Soon\|Under Construction\|WIP" --include="*.{php,tsx,jsx,ts,js,vue,blade.php}" src/ app/ resources/

# Debug statements
grep -rn "console\.log\|console\.warn\|console\.error\|console\.debug" --include="*.{tsx,jsx,ts,js,vue}" src/ resources/
grep -rn "dd(\|dump(\|var_dump(\|print_r(\|ray(" --include="*.php" app/ resources/
grep -rn "Log::debug\|logger(" --include="*.php" app/

# Unfinished work
grep -rn "TODO\|FIXME\|HACK\|XXX\|TEMP\|TEMPORARY" --include="*.{php,tsx,jsx,ts,js,vue,blade.php}" src/ app/ resources/

# Hardcoded values that should be config/env
grep -rn "localhost\|127\.0\.0\.1\|:3000\|:8000\|:8080" --include="*.{php,tsx,jsx,ts,js,vue}" src/ app/ resources/

# Unsafe query patterns
grep -rn "DB::raw\|whereRaw\|selectRaw" --include="*.php" app/

# SECURITY NOTE: When checking for credential safety, verify that:
# - .env is in .gitignore (check .gitignore file)
# - Config files use env() helper, not hardcoded values (check config/ directory)
# - No credentials are committed to source control
# NEVER output or display actual credential values in verification reports.

# Missing error handling
grep -rn "\.catch\s*(\s*)" --include="*.{tsx,jsx,ts,js}" src/ resources/
grep -rn "catch\s*(\s*\$e\s*)\s*{[\s]*}" --include="*.php" app/

# Unpinned dependencies
grep -n '"\^' package.json
grep -n '"~' package.json
```

---

## Validation Parity Check Example

**Backend validation (Laravel):**
```php
// CreateOrderRequest
public function rules(): array
{
    return [
        'customer_id' => ['required', 'exists:customers,id'],
        'items' => ['required', 'array', 'min:1'],
        'items.*.product_id' => ['required', 'exists:products,id'],
        'items.*.quantity' => ['required', 'integer', 'min:1', 'max:999'],
        'notes' => ['nullable', 'string', 'max:500'],
        'discount_code' => ['nullable', 'string', 'exists:discount_codes,code'],
    ];
}
```

**Frontend validation should match:**
```tsx
// Verify these rules exist in the frontend form:
// customer_id: required                    -- CHECK
// items: required, at least 1             -- CHECK
// items.*.product_id: required            -- CHECK
// items.*.quantity: required, min 1, max 999  -- CHECK (is max enforced?)
// notes: optional, max 500 chars          -- CHECK (is maxLength set?)
// discount_code: optional                 -- CHECK

// COMMON MISMATCHES TO LOOK FOR:
// - Frontend allows quantity 0 but backend requires min:1
// - Frontend has no max length on notes but backend limits to 500
// - Frontend doesn't validate items array is non-empty
// - Frontend allows submit with no customer selected
```

---

## Status Workflow Verification Example

**Plan says:** draft -> pending -> processing -> shipped -> delivered

**Verify:**
```
1. Can an order go from draft to pending? (expected: YES)
2. Can an order go from draft to shipped? (expected: NO -- skips steps)
3. Can an order go from delivered back to processing? (expected: NO -- no backward)
4. Can an order go from cancelled to any status? (expected: NO -- terminal state)
5. Does the UI show only valid next-status options?
6. Does the backend reject invalid transitions?
7. Does each transition trigger the correct events/notifications?
```

**How to find in code:**
```bash
# Find status constants/enum
grep -rn "status\|Status" --include="*.php" app/Enums/ app/Models/
# Find transition logic
grep -rn "transition\|canBe\|allowedTransitions\|status_flow" --include="*.php" app/
# Find status validation
grep -rn "in:draft,pending\|Rule::in" --include="*.php" app/
```

---

## Permission Verification Example

**Plan says:** Admin manages all orders. Staff views only assigned orders.

**Verify backend:**
```bash
# Find policy
grep -rn "OrderPolicy\|order.*policy\|Gate::define.*order" --include="*.php" app/
# Find middleware
grep -rn "middleware.*auth\|middleware.*role\|middleware.*permission" --include="*.php" routes/
# Find authorization in controller
grep -rn "authorize\|can(\|cannot(\|this->authorize" --include="*.php" app/Http/Controllers/
# Find scope filtering
grep -rn "scope\|where.*user_id\|where.*assigned" --include="*.php" app/Models/ app/Http/Controllers/
```

**Verify frontend:**
```bash
# Find permission checks in UI
grep -rn "can\|permission\|role\|isAdmin\|hasRole\|v-if.*admin\|v-if.*permission" --include="*.{tsx,jsx,ts,js,vue,blade.php}" src/ resources/
```

**Check both layers:**
```
1. Backend: Does OrderPolicy@viewAny scope by user for staff?
2. Backend: Does OrderPolicy@update check ownership for staff?
3. Frontend: Does the UI hide edit/delete buttons for staff on others' orders?
4. CRITICAL: Is backend enforcement present? (UI hiding alone is NOT security)
```

---

## N+1 Query Detection Example

```bash
# Find eager loading
grep -rn "with(\|load(" --include="*.php" app/Http/Controllers/ app/Actions/ app/Services/
# Find relationships used in loops (potential N+1)
grep -rn "->items\|->customer\|->user\|->product" --include="*.php" app/Http/Controllers/
# Find query in blade/component loops
grep -rn "@foreach.*->.*->" --include="*.blade.php" resources/views/
```

**Red flag pattern:**
```php
// N+1: loads customer for EACH order in the loop
$orders = Order::all();
foreach ($orders as $order) {
    echo $order->customer->name; // N+1 query!
}

// Fixed:
$orders = Order::with('customer')->get();
```

---

## Environment Check Example

```bash
# Verify .env is gitignored (NEVER read or output .env contents)
grep -n "\.env" .gitignore

# Check that config files use env() helper instead of hardcoded values
grep -rn "env(" --include="*.php" config/ | head -20

# Check .env.example has all required keys documented (keys only, no values)
grep -n "^[A-Z]" .env.example | cut -d= -f1

# Check for dev-only drivers in production config
grep -n "driver.*=.*file\|driver.*=.*array\|driver.*=.*log\|driver.*=.*sync" config/
```

---

## Verification Report Template

```markdown
## Verification Report -- [Project Name] -- [Date]

### Plan Coverage
- Total plan items: X
- Implemented: Y
- Not found: Z
- Coverage: Y/X (Z%)

### Results by Perspective

| Perspective | Pass | Fail | Partial | Critical | Score |
|---|---|---|---|---|---|
| QA Engineer | 15 | 3 | 2 | 1 | 0.75 |
| Product Manager | 10 | 1 | 0 | 0 | 0.91 |
| Project Manager | 8 | 2 | 1 | 0 | 0.73 |
| Tech Lead | 18 | 4 | 1 | 2 | 0.78 |
| Business Analyst | 11 | 0 | 1 | 0 | 0.92 |
| DevOps Engineer | 14 | 3 | 0 | 1 | 0.82 |
| Stakeholder | 9 | 1 | 0 | 0 | 0.90 |

### Overall Confidence: 0.83 / 1.0
### Verdict: NOT APPROVED (3 critical failures blocking release)

### Critical Failures
1. [Data Contract] OrderDetail.tsx accesses `order.referenceNumber` but backend returns `reference_number` -- app/Http/Controllers/OrderController.php:45 vs resources/js/Pages/Orders/Show.tsx:23
2. [Security] Admin route /admin/users has no auth middleware -- routes/web.php:87
3. [Config] Payment integration config references test/sandbox mode -- config/services.php:12

### High Priority
1. [QA] Order export CSV returns empty file -- app/Actions/ExportOrdersAction.php:34 returns empty collection
2. [Tech] N+1 query on orders index -- app/Http/Controllers/OrderController.php:12 missing eager load for items
3. [Business] Tax calculation uses 10% flat rate but spec says tiered rates -- app/Services/TaxCalculator.php:8

### Data Contract Mismatches
| Component | Field Accessed | Backend Field | Status |
|---|---|---|---|
| OrderDetail.tsx:23 | order.referenceNumber | reference_number | MISMATCH |
| OrderDetail.tsx:25 | order.totalAmount | total_amount | MISMATCH |
| OrderDetail.tsx:27 | order.customer.name | customer.full_name | MISMATCH |
| OrderDetail.tsx:31 | item.productName | product_name | MISMATCH |
| OrderDetail.tsx:32 | item.qty | quantity | MISMATCH |
| OrderDetail.tsx:32 | item.price | unit_price | MISMATCH |
| OrderDetail.tsx:33 | item.total | line_total | MISMATCH |
| OrderList.tsx:15 | order.status | status | MATCH |
| OrderList.tsx:16 | order.customer.email | customer.email | MATCH |

### Blocking Items (must fix before release)
1. Fix all data contract mismatches (7 fields)
2. Add auth middleware to admin routes
3. Switch payment integration from test/sandbox mode to production mode
```

---

## Browser Testing Detailed Checklists

### Authentication Flow

```
1. Navigate to login page
   - [ ] Page loads without errors
   - [ ] Login form visible and styled correctly
   - [ ] Form fields have proper labels and placeholders
   - [ ] Password field masks input

2. Submit with empty fields
   - [ ] Validation errors display correctly
   - [ ] Error messages readable and positioned properly
   - [ ] Form does not submit

3. Submit with wrong credentials
   - [ ] Error message displays (not generic 500)
   - [ ] Password field clears
   - [ ] No sensitive info leaked in error

4. Submit with correct credentials
   - [ ] Redirects to dashboard/home
   - [ ] User name/avatar in navigation
   - [ ] No flash of unauthenticated content

5. Test logout
   - [ ] Redirects to login page
   - [ ] Cannot access protected pages after logout
   - [ ] Back button does not show authenticated content
```

### Navigation & Layout

```
FOR EVERY PAGE:
1. Click every nav link
   - [ ] Correct page loads
   - [ ] Active state highlights current page
   - [ ] No 404s, blank pages, or error screens
   - [ ] Page title matches navigation label
   - [ ] Breadcrumbs correct (if applicable)

2. Layout consistency
   - [ ] Sidebar consistent across all pages
   - [ ] Header/navbar consistent
   - [ ] Spacing and alignment match
   - [ ] No content overflow or horizontal scrollbars

3. Responsive (375px mobile)
   - [ ] Navigation collapses to hamburger/drawer
   - [ ] Content stacks properly
   - [ ] No text overflow or cut-off elements
   - [ ] Touch targets large enough
```

### Data Display

```
FOR EVERY LIST/TABLE/GRID:
1. Real data
   - [ ] Data from database, NOT placeholder
   - [ ] Column headers match plan
   - [ ] Formatting correct (dates, currency, badges)
   - [ ] No "undefined", "null", "NaN", "[object Object]"
   - [ ] Empty state when no data

2. Pagination
   - [ ] Page numbers correct
   - [ ] Next/previous works
   - [ ] First/last page works
   - [ ] "Showing X of Y" accurate
   - [ ] URL updates with page number

3. Search/Filters
   - [ ] Search input functional
   - [ ] Results update on search
   - [ ] Clear search restores full list
   - [ ] Filter dropdowns have real options
   - [ ] Multiple filters combine correctly
   - [ ] No results shows appropriate message

4. Sorting
   - [ ] Column header click sorts
   - [ ] Sort indicator displays
   - [ ] Data actually reorders
   - [ ] Asc/desc toggle works
```

### Form Testing

```
FOR EVERY FORM:
1. Load form
   - [ ] All fields render correctly
   - [ ] Labels present and readable
   - [ ] Required fields marked
   - [ ] Defaults populated (edit forms)
   - [ ] Select options load from backend

2. Submit empty
   - [ ] Validation errors on correct fields
   - [ ] Readable error messages
   - [ ] Form does not submit
   - [ ] Page does not reload

3. Submit invalid data
   - [ ] Email rejects non-email
   - [ ] Numbers reject text
   - [ ] Max length enforced
   - [ ] Min/max values enforced
   - [ ] Specific error messages

4. Submit valid data
   - [ ] Success notification appears
   - [ ] Redirects to correct page
   - [ ] New record in list
   - [ ] All data saved correctly

5. Edit form
   - [ ] Existing data populates all fields
   - [ ] Changes save correctly
   - [ ] Unchanged fields retain values
   - [ ] Success notification

6. File uploads
   - [ ] File picker opens
   - [ ] Preview after selection
   - [ ] Upload succeeds on submit
   - [ ] File accessible after save
   - [ ] Invalid types rejected
```

### CRUD Operations

```
FOR EVERY RESOURCE:
CREATE:
- [ ] Create button navigates to form
- [ ] Form submits and creates record
- [ ] New record visible in list

READ:
- [ ] Detail page loads with all data
- [ ] All fields from plan displayed
- [ ] Related data correct
- [ ] Images/media load

UPDATE:
- [ ] Edit button navigates to form
- [ ] Form pre-fills with existing data
- [ ] Changes save and reflect
- [ ] Updated data visible in list

DELETE:
- [ ] Delete button exists
- [ ] Confirmation dialog appears
- [ ] Cancel returns without deleting
- [ ] Confirm deletes the record
- [ ] Record disappears from list
- [ ] Success notification
```

### Interactive Elements

```
Modals:
- [ ] Opens on trigger, content loads, close works (X, overlay, Escape)
- [ ] Form inside modal submits, background scroll locked

Dropdowns:
- [ ] Opens on click, options populated, selection works, closes on outside click

Tabs:
- [ ] All visible, switching works, active visually distinct, content loads per tab

Toggles:
- [ ] State changes, persists after save, immediate visual feedback

Notifications:
- [ ] Appear after actions, correct type (success/error/warning), auto-dismiss, dismissible
```

### Permission & Role Testing

```
FOR EACH ROLE IN THE PLAN:
1. Login as that role
2. Verify menu items match allowed pages
3. Verify CRUD actions match allowed operations
4. Try accessing restricted pages via URL directly
5. Verify edit/delete hidden on others' records (if applicable)
6. No over-permission, no under-permission
```

### Error States

```
- [ ] Non-existent URL shows 404 page (not white screen or stack trace)
- [ ] 404 has navigation back to app
- [ ] Server error handled gracefully (no stack trace to user)
- [ ] User can recover from errors (navigate away, retry)
- [ ] Loading spinners/skeletons appear during slow loads
```

### Visual Consistency

```
ACROSS ALL PAGES:
- [ ] Same fonts throughout
- [ ] Same color scheme
- [ ] Same button styles
- [ ] Same card/panel/table styles
- [ ] Same form field styles
- [ ] Same spacing patterns
- [ ] Same icon style (all outline or all solid)
- [ ] No misaligned elements
- [ ] Dark mode consistent (if applicable)
```

### Screenshot Evidence

```
Capture for the report:
- List pages with real data
- Detail pages with complete info
- Forms showing validation states
- Empty states
- Error states
- Mobile responsive views

NEVER screenshot pages with sensitive data (credentials, payment info).
Reference screenshots by page name in the report.
```
