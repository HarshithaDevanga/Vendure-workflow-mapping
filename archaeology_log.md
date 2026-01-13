## Vendure Promotion & Shipping Archaeology Log

This log captures concrete **UI Expectation → API Reality** mappings based on Vendure's documented Admin GraphQL semantics.

**Track A — Promotion Challenge**: Black Friday coupon scenario  
**Track B — Global Shipping Challenge**: Oceania shipping zone and flat rate method

---

### 1. Money Representation

- **UI Expectation**:  
  “Order total above **$100**”.

- **API Reality**:  
  - The `minimumOrderAmount` promotion condition (built‑in in Vendure) expects the threshold as an integer amount in the smallest currency unit (e.g. cents).  
  - In the Admin API this is supplied via a `ConfigArgInput` where:
    - `name = "amount"`
    - `value = "10000"`  
  - The `value` is a **string**, but semantically represents the integer `10000` (100 dollars × 100 cents).

---

### 2. Customer Group vs CustomerGroupId

- **UI Expectation**:  
  “Customer is in **VIP** group” (chosen by visible name in a dropdown).

- **API Reality**:  
  - Promotions operate on **`CustomerGroup` IDs**, not names.
  - The relevant built‑in condition is `inCustomerGroup`, whose argument:
    - `name = "customerGroupId"`
    - `value` must be set to the `id` of the `CustomerGroup` whose `name = "VIP"`.
  - The Admin UI therefore must:
    - Query `customerGroups` from the Admin API.
    - Find the group where `name == "VIP"`.
    - Inject the returned `id` into the promotion condition’s argument.

---

### 3. Conditions as Coded Operations

- **UI Expectation**:  
  Readable condition editors such as:
  - “Order total at least …”
  - “Customer is in group …”

- **API Reality**:  
  - Conditions are expressed as **configurable operations**:
    - Discovered via the `promotionConditions` query.
    - Each condition has a stable `code` (e.g. `minimumOrderAmount`, `inCustomerGroup`) and a list of argument descriptors.
  - When creating a promotion, the Admin API expects:
    - `conditions: [ { code: "...", arguments: [ { name, value }, ... ] }, ... ]`
  - The human‑readable labels in the UI are derived from these codes and their metadata, but the automation layer must work with the **codes and arguments**, not the labels.

---

### 4. Actions as Coded Operations

- **UI Expectation**:  
  An action editor such as “Apply **15%** discount to order”.

- **API Reality**:  
  - Actions are also configurable operations, discovered via `promotionActions`.
  - For this scenario, we rely on the documented built‑in action:
    - `code = "orderPercentageDiscount"`
    - Takes a `discount` argument:
      - `name = "discount"`
      - `value = "15"` (string containing the integer percentage).
  - The Admin UI maps “15%” in the form to this string value in the mutation variables.

---

### 5. Promotions and Enabled State

- **UI Expectation**:  
  A simple toggle like “Enabled” on the promotion form.

- **API Reality**:  
  - The `Promotion` type and `CreatePromotionInput` include a boolean field:
    - `enabled: Boolean`
  - Whether the promotion is active is controlled by this flag, not by creation alone.
  - For the Black Friday coupon to be usable immediately, the Admin UI sets:
    - `enabled = true` in the `createPromotion` mutation input.

---

### 6. Multiple Conditions Composition

- **UI Expectation**:  
  A promotion that should apply only when:
  - Order total is above $100  
  **AND**  
  - Customer is in the `VIP` group.

- **API Reality**:  
  - Vendure’s promotion engine (as documented for built‑in promotions) combines multiple listed conditions with logical **AND** semantics.  
  - In the `createPromotion` input, both conditions are placed in the same `conditions` array:
    - One `minimumOrderAmount` operation.
    - One `inCustomerGroup` operation.
  - The discount action (`orderPercentageDiscount`) is only executed when **all** configured conditions evaluate to true.

---

### 7. Configurable Operation Arguments Are Strings

- **UI Expectation**:  
  - Numeric fields for money and percentages.
  - Dropdown for customer groups.

- **API Reality**:  
  - The `ConfigArgInput` type used for promotion conditions/actions in the Admin API has:
    - `name: String!`
    - `value: String!`
  - Even though arguments are conceptually integers (amount, discount) or IDs (customerGroupId), they are transmitted as **strings** and interpreted according to metadata attached to the operation definition.

---

### 8. Query vs Mutation Responsibilities

- **UI Expectation**:  
  - A single screen where the admin both discovers options (conditions/actions/groups) and saves the promotion.

- **API Reality**:  
  - **Queries**:
    - `promotionConditions` / `promotionActions` provide the available building blocks.
    - `customerGroups` resolves group names to IDs.
  - **Mutation**:
    - `createPromotion` takes the final, composed configuration:
      - Basic fields (name, couponCode, enabled).
      - Conditions (by code + arguments).
      - Actions (by code + arguments).
  - Automation must reproduce this split: lookups first, then a single mutation using the derived IDs and codes.

---

### 9. Assumptions Specific to This Reconstruction

These points are derived from Vendure's official schema and built‑in promotion documentation. A real deployment might customize or extend these behaviors (e.g. additional conditions or alternative currencies); in such cases, the same network‑driven process must be re‑run and this log and the workflow map updated to match the observed API reality.

---

## Track B — Global Shipping Challenge

### 10. Money Representation in Shipping Methods

- **UI Expectation**:  
  "Configure a **$15** flat rate shipping method".

- **API Reality**:  
  - Shipping method calculators (like promotions) expect monetary values in the smallest currency unit (e.g. cents).  
  - In the Admin API, the `default-shipping-calculator` (built‑in in Vendure) expects a `rate` argument where:
    - `name = "rate"`
    - `value = "1500"`  
  - The `value` is a **string**, but semantically represents the integer `1500` (15 dollars × 100 cents).

---

### 11. Country Names vs ISO-2 Country Codes

- **UI Expectation**:  
  "Add **Australia** and **New Zealand** to the shipping zone" (chosen by visible country names in a picker).

- **API Reality**:  
  - Shipping zones operate on **ISO 3166-1 alpha-2 country codes**, not country names.
  - The `updateShippingZone` mutation's `members` field expects an array of objects with:
    - `code: String!` (the ISO-2 code)
  - For this scenario:
    - **"Australia"** → **`"AU"`**
    - **"New Zealand"** → **`"NZ"`**
  - The Admin UI therefore must:
    - Map country names displayed in the picker to their corresponding ISO-2 codes.
    - Pass these codes in the `members` array when updating the shipping zone.

---

### 12. Shipping Methods Bind to Zones via shippingZoneIds

- **UI Expectation**:  
  "Configure a shipping method that only applies to the 'Oceania' zone" (selected via a zone picker or checkbox).

- **API Reality**:  
  - Shipping methods are associated with shipping zones via a **`shippingZoneIds`** array field in `CreateShippingMethodInput`.
  - This field takes an array of `ShippingZone.id` values (not names or codes).
  - To link a method to a zone, the Admin UI must:
    - Capture the `id` returned from `createShippingZone` (or query existing zones).
    - Include this `id` in the `shippingZoneIds` array when creating the shipping method.
  - In the workflow, this is expressed as:
    - `shippingZoneIds: ["{{create_oceania_zone.shippingZoneId}}"]`

---

### 13. Flat Rate as a Calculator with Arguments

- **UI Expectation**:  
  A shipping method editor with options like "Flat Rate" and a rate input field showing "$15".

- **API Reality**:  
  - Shipping methods use **configurable calculator operations**, similar to promotion conditions/actions.
  - Calculators are discovered via the `shippingCalculators` query field.
  - Each calculator has a stable `code` and argument metadata.
  - For flat rate shipping, Vendure provides the built‑in calculator:
    - `code = "default-shipping-calculator"`
    - Takes a `rate` argument:
      - `name = "rate"`
      - `value = "1500"` (string containing the integer amount in cents).
  - The Admin UI maps "Flat Rate" and "$15" in the form to:
    - The calculator `code` from `shippingCalculators`.
    - The `rate` argument value as a stringified integer in cents.

---

### 14. Shipping Eligibility Checkers

- **UI Expectation**:  
  A shipping method configuration that determines when the method is available (often implicit or default).

- **API Reality**:  
  - Shipping methods require a **checker** operation (similar to calculators) that determines eligibility.
  - Checkers are discovered via the `shippingEligibilityCheckers` query field.
  - The built‑in checker (per Vendure docs):
    - `code = "default-shipping-eligibility-checker"`
    - Typically takes no arguments (empty `arguments` array).
  - In the workflow, the checker code is discovered dynamically and referenced as:
    - `{{load_shipping_eligibility_checkers.defaultEligibilityCheckerCode}}`

---

### 15. Zone Creation vs Country Assignment Separation

- **UI Expectation**:  
  A single form where the admin creates a zone and adds countries in one step.

- **API Reality**:  
  - Vendure's Admin API separates zone creation from country assignment:
    - `createShippingZone` creates an empty zone (or optionally accepts `members`).
    - `updateShippingZone` adds or removes countries via the `members` field.
  - The workflow reflects this separation:
    - Step `create_oceania_zone` creates the zone.
    - Step `update_oceania_zone` adds countries `AU` and `NZ`.
  - This two-step approach allows for:
    - Creating zones before knowing all countries.
    - Updating zone membership independently of zone metadata.
    - Maintaining a clear dependency chain: method creation depends on zone existence.

---

### 16. Shipping Method Fulfillment Handlers

- **UI Expectation**:  
  A shipping method configuration that determines how orders are fulfilled (often implicit or default).

- **API Reality**:  
  - Shipping methods include a `fulfillmentHandler` field that specifies how fulfillment is processed.
  - For manual fulfillment (common default), the value is:
    - `fulfillmentHandler = "manual-fulfillment"`
  - This is a string identifier, not a configurable operation, and is set directly in `CreateShippingMethodInput`.

---

### 17. Shipping Method Translations

- **UI Expectation**:  
  A shipping method form with fields for "Name" and "Description".

- **API Reality**:  
  - Vendure uses a **translations** array for multi-language support.
  - `CreateShippingMethodInput` includes:
    - `translations: [ShippingMethodTranslationInput!]!`
  - Each translation object contains:
    - `languageCode: LanguageCode!` (e.g. `"en"`)
    - `name: String!`
    - `description: String`
  - For a single-language setup, one translation entry is provided (typically `languageCode: "en"`).

---

### 18. Assumptions Specific to Track B Reconstruction

These points are derived from Vendure's official Admin API schema and built‑in shipping documentation. A real deployment might customize calculators, checkers, or fulfillment handlers; in such cases, the same network‑driven process must be re‑run and this log and the workflow map updated to match the observed API reality.


