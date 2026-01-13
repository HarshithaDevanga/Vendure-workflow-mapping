## Approach to Reverse‑Engineering Vendure Workflows

This document explains how Vendure Admin UI workflows were mapped into `workflow_map.json`, and how the same method generalizes to other systems.

**Track A — Promotion Challenge**: Black Friday coupon **BF2024** workflow  
**Track B — Global Shipping Challenge**: Oceania shipping zone and flat rate method workflow

Where information was not directly observable from the Admin UI or official Vendure documentation, it is explicitly called out below as an **assumption** rather than a fact.

---

### 1. Mapping UI → Network Request → GraphQL → JSON

- **Identify the functional flow in the UI**
  - **Goal**: “Create a Black Friday coupon code `BF2024` with a 15% discount, only if order total > $100 and customer is in the `VIP` group.”
  - **UI path** (conceptual): `Marketing` → `Promotions` → `Create promotion`.

- **Observe the GraphQL operations used by the Admin UI**
  - In a real browser, this is done via DevTools → Network → filter for `graphql` / `XHR`.
  - For Vendure, the Admin UI calls the **Admin GraphQL API**. The relevant **field operations** (as defined in the official Admin API schema) for this workflow are:
    - `promotionConditions` (query field) to load available promotion conditions.
    - `promotionActions` (query field) to load available promotion actions.
    - `customerGroups` (query field) to resolve the `VIP` group name to an internal `CustomerGroup.id`.
    - `createPromotion` (mutation field) to persist the configured promotion.

- **Translate raw GraphQL into workflow steps**
  - Each **UI action** that materially affects data is mapped to one **step** in `workflow_map.json`.
  - For each step we record:
    - **ui_action**: human description of what the admin does.
    - **api_call.operation**: the real Vendure Admin API field being invoked (e.g. `promotionConditions`, `promotionActions`, `customerGroups`, `createPromotion`).
    - **api_call.type**: `"query"` or `"mutation"`, matching the GraphQL operation type.
    - **api_call.inputs**: JSON structure of the `variables` payload actually sent on the wire.
    - **api_call.outputs**: JSONPath selectors into the GraphQL response.
    - **dependencies**: list of prior steps whose outputs are required.

-- **Concrete mappings used in `workflow_map.json`**
  - **Loading promotion conditions**
    - UI: opening the create‑promotion screen, so that condition options become available.
    - API: Admin `Query` field `promotionConditions`.
    - This is represented as the `load_promotion_conditions` step with:
      - `api_call.operation = "promotionConditions"`
      - `api_call.type = "query"`.
    - The codes we rely on are taken from Vendure’s documented built‑in conditions:
      - `minimumOrderAmount`
      - `inCustomerGroup`
    - These are extracted via JSONPath filters over `$.data.promotionConditions[*].code`.
  - **Loading promotion actions**
    - UI: same screen, but loading available actions.
    - API: Admin `Query` field `promotionActions`.
    - This is represented as the `load_promotion_actions` step with:
      - `api_call.operation = "promotionActions"`
      - `api_call.type = "query"`.
    - We rely on the built‑in action code (from Vendure docs):
      - `orderPercentageDiscount`
    - Extracted via JSONPath over `$.data.promotionActions[*].code`.
  - **Resolving “VIP” → customerGroupId**
    - UI: when configuring the “Customer is in group” condition, the admin chooses group `VIP`.
    - API: Admin `Query` field `customerGroups(options: ...)`.
    - The workflow extracts:
      - `vipCustomerGroupId = $.data.customerGroups.items[0].id`
      - `vipCustomerGroupName = $.data.customerGroups.items[0].name`
    - Represented as step `fetch_vip_group`.
  - **Creating the promotion**
    - UI: filling the form and clicking **Save**.
    - API: Admin `Mutation` field `createPromotion(input: CreatePromotionInput!)`.
    - `CreatePromotionInput` (from Vendure Admin API schema) includes:
      - `enabled: Boolean`
      - `name: String!`
      - `couponCode: String`
      - `conditions: [ConfigurableOperationInput!]`
      - `actions: [ConfigurableOperationInput!]!`
    - `ConfigurableOperationInput`:
      - `code: String!`
      - `arguments: [ConfigArgInput!]!`
    - For this scenario (step `create_promotion`) we set (based on the documented Vendure schema and the Track A requirements):
      - `enabled: true`
      - `name: "Black Friday 2024"`
      - `couponCode: "BF2024"`
      - `conditions`:
        - `minimumOrderAmount` with `amount = "10000"` (100 dollars in cents).
        - `inCustomerGroup` with `customerGroupId = "{{fetch_vip_group.vipCustomerGroupId}}"`.
      - `actions`:
        - `orderPercentageDiscount` with `discount = "15"`.

---

### 2. Detecting Field Meanings, Condition Codes, and Action Codes

- **Field meanings (UI labels → API fields)**
  - **“Coupon code” (UI)** → **`couponCode` (API)**  
    Observed in the `CreatePromotionInput` type in the Admin API schema and the variables sent with `createPromotion`.
  - **“Enabled” toggle (UI)** → **`enabled` (API)**  
    Boolean field on the `Promotion` type and its input type.
  - **“Name” (UI)** → **`name` (API)**  
    Required field in `CreatePromotionInput`.

  - **Condition codes (semantic codes)**
  - Vendure exposes promotion conditions as **configurable operations**:
    - Query field: `promotionConditions` (Admin API).
    - Each condition provides a stable **`code`** and its argument metadata.
  - From Vendure’s **documented built‑in promotion conditions**:
    - `minimumOrderAmount`  
      - Argument: `amount` (integer threshold in smallest currency unit, passed as a string in `ConfigArgInput.value`).
    - `inCustomerGroup`  
      - Argument: `customerGroupId` (the `id` of a `CustomerGroup`).
  - In the workflow these are **not hard‑coded literals**; instead we:
    - Discover them via `promotionConditions`.
    - Store them as outputs of `load_promotion_config`.
    - Reference them as `{{load_promotion_config.minimumOrderAmountConditionCode}}` and `{{load_promotion_config.inCustomerGroupConditionCode}}`.

  - **Action codes**
  - Similar to conditions, Vendure’s actions are configurable operations returned by `promotionActions`.
  - For this scenario we use the built‑in action (per Vendure docs and default configuration):
    - `orderPercentageDiscount`
      - Argument: `discount` (percentage as a stringified integer).
  - In the workflow:
    - Code taken from `promotionActions` and referenced as `{{load_promotion_config.orderPercentageDiscountActionCode}}`.

  - **Data type semantics (money, IDs, strings)**
  - **Money values**
    - Vendure represents money in the smallest currency unit (e.g. cents) as an `Int`.
    - For a threshold of **$100**, the argument value is `10000`.
    - In GraphQL variables, `ConfigArgInput.value` is a **string**, so `"10000"` is used.
  - **Customer groups**
    - UI shows `"VIP"`; API requires the corresponding `CustomerGroup.id`.
    - We therefore always:
      - Query `customerGroups`.
      - Filter or select by `name == "VIP"`.
      - Pass the returned `id` into the condition argument.

---

### 3. Dependency Graph Extraction and Dynamic Data Flow

- **Why dependencies matter**
  - Promotion conditions and actions are configured using **codes** and **argument values** that may come from prior API calls:
    - Codes from `promotionConditions` / `promotionActions`.
    - IDs from `customerGroups`.
  - To ensure an automation engine can replay the workflow, we must explicitly express:
    - **Where each input comes from**.
    - **Which steps must run before others**.

- **How dependencies are modeled in `workflow_map.json`**
  - Each step has a **`dependencies`** array:
    - `load_promotion_config` has no dependencies.
    - `fetch_vip_group` depends on `load_promotion_config` (same screen / same logical phase).
    - `create_promotion` depends on both prior steps.
  - **Variable references**:
    - `{{fetch_vip_group.vipCustomerGroupId}}` is used for the `customerGroupId` argument.
    - Condition and action codes are referenced via:
      - `{{load_promotion_config.minimumOrderAmountConditionCode}}`
      - `{{load_promotion_config.inCustomerGroupConditionCode}}`
      - `{{load_promotion_config.orderPercentageDiscountActionCode}}`
  - An automation engine can:
    - Execute steps in topological order.
    - Materialize the JSONPath‑extracted outputs into a variable context.
    - Substitute `{{...}}` placeholders in later inputs before issuing API calls.

---

### 4. Semantic Normalization and Gap Recording

- **UI vs API semantics**
  - **Currencies**: UI shows “$100”; API expects `"10000"` cents inside a `ConfigArgInput.value` string.
  - **Groups by name vs ID**: UI shows group name `VIP`; API requires `customerGroupId`.
  - **Conditions & actions as “blocks” vs codes**:
    - UI presents readable labels like “Order total at least …”.
    - API uses opaque `code` strings and argument lists.
  - **Multiple conditions semantics**:
    - For this scenario, it is understood (from Vendure documentation) that multiple conditions on a promotion are combined with logical **AND**.
    - This means both `minimumOrderAmount` and `inCustomerGroup` must be satisfied for the action to apply.

  - **How gaps are documented**
  - The file `archaeology_log.md` records:
    - **UI Expectation → API Reality** pairs (e.g. “$100” → `"10000"`).
    - Behavioral nuances (e.g. use of codes, IDs, and logical AND between conditions).
  - Any place where the precise behavior cannot be inferred solely from the UI is either:
    - **Corroborated against Vendure documentation**, or
    - **Labeled explicitly as an assumption**.

- **Key assumptions (explicitly marked, to avoid “guessing”)**
  - The following items are taken from Vendure’s **published Admin API schema and built‑in promotion docs**, and are treated as **facts for an unmodified Vendure installation**:
    - Existence of query fields: `promotionConditions`, `promotionActions`, `customerGroups`.
    - Existence of mutation field: `createPromotion`.
    - Built‑in condition codes: `minimumOrderAmount`, `inCustomerGroup`.
    - Built‑in action code: `orderPercentageDiscount`.
    - Argument names: `amount`, `customerGroupId`, `discount`.
  - The following are **ASSUMPTIONS that must be checked against the live system** and are therefore documented, not silently relied upon:
    - That there is a `CustomerGroup` whose `name` is exactly `"VIP"` and that the admin selects this one.
    - That the currency for the shop uses a factor of 100 (e.g. dollars → cents) so that `$100` becomes `10000`.
    - That the Admin UI, when constructing `ConfigArgInput`, always stringifies numeric values (e.g. `"10000"`, `"15"`), which is how the default Admin UI behaves.
This is the same method used for Vendure in this challenge: capture the **actual network semantics**, map them carefully to UI concepts, expose dependencies explicitly, and document every non‑obvious transformation or assumption.


---

### 5. Universal Algorithm for Other Systems (Salesforce, Jira, HubSpot, …)

- **Network‑driven reverse‑engineering**
  - **Step 1: Identify the target user intent**
    - E.g. “Create a discount”, “Create a Jira issue”, “Add a Salesforce opportunity”.
  - **Step 2: Drive the UI manually while recording network traffic**
    - Use browser DevTools to capture all XHR / fetch / GraphQL / REST calls.
  - **Step 3: Cluster requests into logical steps**
    - Group network calls by UI interaction (button clicks, form submits, autocomplete searches).

- **Field matching heuristics**
  - **Label and value correlation**
    - Compare UI labels and entered values with fields in the request payload.
  - **Shape and type matching**
    - Recognize IDs (UUIDs, numeric IDs), enums, timestamps, money formats, etc.
  - **Cross‑checking with public schema/docs**
    - Use official API docs (e.g. Salesforce REST/GraphQL, Jira REST, HubSpot CRM API) to confirm field meanings.

- **Dependency graph extraction**
  - **Detect IDs and tokens flowing between calls**
    - For example, a “create contact” call returns `id` which is then used in “create deal” as `contactId`.
  - **Create a directed graph**
    - Nodes = API operations.
    - Edges = “this operation’s inputs reference that operation’s outputs”.
  - **Serialize to a workflow format**
    - As in `workflow_map.json`, with:
      - `steps[*].api_call.inputs` referencing `{{other_step.outputVar}}`.

- **Semantic normalization**
  - **Normalize units and formats**
    - Money (cents vs dollars), dates (ISO vs localized), booleans (checkbox vs true/false).
  - **Normalize entity references**
    - Human‑readable labels (e.g. “VIP”, “High priority”) → internal IDs or codes.
  - **Record all transformations**
    - Capture them as **UI → API** mappings in an archaeology log, ensuring an automation engine can reapply them deterministically.

This is the same method used for Vendure in this challenge: capture the **actual network semantics**, map them carefully to UI concepts, expose dependencies explicitly, and document every non‑obvious transformation or assumption.

---

## Track B — Global Shipping Challenge

### 1. Mapping UI → Network Request → GraphQL → JSON (Shipping Workflow)

- **Identify the functional flow in the UI**
  - **Goal**: "Set up a new shipping zone for 'Oceania'. Add 'Australia' and 'New Zealand' to this zone. Then, configure a 'Flat Rate' shipping method of $15 that only applies to this zone."
  - **UI path** (conceptual): `Settings` → `Shipping` → `Shipping Zones` → `Create zone`, then `Shipping Methods` → `Create method`.

- **Observe the GraphQL operations used by the Admin UI**
  - For Vendure shipping configuration, the Admin UI calls the **Admin GraphQL API**. The relevant **field operations** for this workflow are:
    - `shippingCalculators` (query field) to load available shipping calculator operations.
    - `shippingEligibilityCheckers` (query field) to load available eligibility checker operations.
    - `createShippingZone` (mutation field) to create the "Oceania" zone.
    - `updateShippingZone` (mutation field) to add countries to the zone.
    - `createShippingMethod` (mutation field) to create the flat rate shipping method.

- **Translate raw GraphQL into workflow steps**
  - Each **UI action** that materially affects data is mapped to one **step** in `workflow_map.json`.
  - For Track B, we record:
    - **Loading shipping calculators** (`load_shipping_calculators`)
      - UI: opening the create‑shipping‑method screen, so that calculator options become available.
      - API: Admin `Query` field `shippingCalculators`.
      - Extracts the built‑in calculator code: `default-shipping-calculator`.
    - **Loading shipping eligibility checkers** (`load_shipping_eligibility_checkers`)
      - UI: same screen, but loading available eligibility checkers.
      - API: Admin `Query` field `shippingEligibilityCheckers`.
      - Extracts the built‑in checker code: `default-shipping-eligibility-checker`.
    - **Creating the shipping zone** (`create_oceania_zone`)
      - UI: filling the zone form with name "Oceania" and clicking **Save**.
      - API: Admin `Mutation` field `createShippingZone(input: CreateShippingZoneInput!)`.
      - `CreateShippingZoneInput` includes:
        - `name: String!`
        - `members: [CreateShippingZoneMemberInput!]` (optional, can be added later)
      - Returns `ShippingZone.id` which is captured for later use.
    - **Updating the shipping zone** (`update_oceania_zone`)
      - UI: adding countries "Australia" and "New Zealand" to the zone via country picker.
      - API: Admin `Mutation` field `updateShippingZone(input: UpdateShippingZoneInput!)`.
      - `UpdateShippingZoneInput` includes:
        - `id: ID!`
        - `members: [CreateShippingZoneMemberInput!]` (array of country codes)
      - Each member is `{ code: "AU" }` or `{ code: "NZ" }` (ISO-2 codes).
    - **Creating the shipping method** (`create_flat_rate_method`)
      - UI: filling the method form (name, calculator with rate $15, zone assignment) and clicking **Save**.
      - API: Admin `Mutation` field `createShippingMethod(input: CreateShippingMethodInput!)`.
      - `CreateShippingMethodInput` includes:
        - `code: String!`
        - `translations: [ShippingMethodTranslationInput!]!`
        - `checker: ConfigurableOperationInput!`
        - `calculator: ConfigurableOperationInput!`
        - `fulfillmentHandler: String!`
        - `shippingZoneIds: [ID!]!`
      - The calculator uses:
        - `code = "{{load_shipping_calculators.flatRateCalculatorCode}}"`
        - `arguments = [{ name: "rate", value: "1500" }]`
      - The method is linked to the zone via:
        - `shippingZoneIds = ["{{create_oceania_zone.shippingZoneId}}"]`

---

### 2. Nested Configuration: Zone → Country → Method → Calculator

- **Hierarchical structure**
  - **Zone**: A geographical grouping that contains countries.
    - Created via `createShippingZone`.
    - Identified by `id` and `name`.
  - **Country**: Assigned to zones via ISO-2 codes.
    - Added to zones via `updateShippingZone` with `members` array.
    - Each member is `{ code: "AU" }` or `{ code: "NZ" }`.
  - **Method**: A shipping strategy associated with one or more zones.
    - Created via `createShippingMethod`.
    - Linked to zones via `shippingZoneIds` array.
  - **Calculator**: A configurable operation that computes shipping cost.
    - Discovered via `shippingCalculators` query.
    - Configured with arguments (e.g. `rate = "1500"` for flat rate).
    - Attached to methods via the `calculator` field in `CreateShippingMethodInput`.

- **Why this nesting matters**
  - **Separation of concerns**: Zones define geography, methods define pricing strategy, calculators define computation logic.
  - **Reusability**: A calculator can be used by multiple methods; a method can apply to multiple zones.
  - **Dependency chain**: Methods depend on zones existing; calculators are discovered before method creation.
  - **Flexibility**: Countries can be added/removed from zones without recreating methods; calculator arguments can be changed without recreating methods.

- **How dependencies are modeled in `workflow_map.json`**
  - `create_oceania_zone` has no dependencies (creates the zone).
  - `update_oceania_zone` depends on `create_oceania_zone` (needs zone ID).
  - `load_shipping_calculators` and `load_shipping_eligibility_checkers` have no dependencies (discovery queries).
  - `create_flat_rate_method` depends on:
    - `load_shipping_calculators` (needs calculator code).
    - `load_shipping_eligibility_checkers` (needs checker code).
    - `create_oceania_zone` (needs zone ID for `shippingZoneIds`).

---

### 3. Generalization to Other Platforms (Salesforce, HubSpot, etc.)

- **Salesforce**
  - **Zone → Country → Method → Calculator** maps to:
    - **Zone**: Custom object `Shipping_Zone__c` with fields `Name`, `Region__c`.
    - **Country**: Junction object `Zone_Country__c` linking `Shipping_Zone__c` to standard `Country` records (via ISO codes).
    - **Method**: Custom object `Shipping_Method__c` with:
      - `Zone__c` (lookup to `Shipping_Zone__c`).
      - `Rate__c` (currency field).
      - `Calculator_Type__c` (picklist: "Flat Rate", "Weight-Based", etc.).
    - **Calculator**: Apex class implementing `ShippingCalculator` interface, invoked via `Calculator_Type__c`.
  - **API pattern**: REST API calls to create/update custom objects, with dependencies expressed via lookup relationships.
  - **Workflow mapping**: Each Salesforce object creation becomes a step; lookups create dependencies.

- **HubSpot**
  - **Zone → Country → Method → Calculator** maps to:
    - **Zone**: Custom object type "Shipping Zone" with properties `name`, `region`.
    - **Country**: Association between "Shipping Zone" and standard "Country" objects (via ISO-2 codes stored as properties).
    - **Method**: Custom object type "Shipping Method" with:
      - `shipping_zone` (association to "Shipping Zone").
      - `rate` (number property).
      - `calculator_type` (single-select property: "flat_rate", "weight_based", etc.).
    - **Calculator**: Workflow or custom code that reads `calculator_type` and computes cost.
  - **API pattern**: HubSpot CRM API v3 calls to create/update objects and associations.
  - **Workflow mapping**: Object creation steps with association steps linking methods to zones.

- **Shopify**
  - **Zone → Country → Method → Calculator** maps to:
    - **Zone**: Shipping zone resource (Admin API) with `name` and `countries` array.
    - **Country**: Included directly in zone's `countries` array (ISO-2 codes).
    - **Method**: Shipping rate within a zone, with `price` and `name`.
    - **Calculator**: Implicit (Shopify handles calculation based on rate configuration).
  - **API pattern**: REST Admin API calls to `POST /admin/api/2024-01/shipping_zones.json` and `PUT /admin/api/2024-01/shipping_zones/{id}.json`.
  - **Workflow mapping**: Zone creation, zone update (add countries), rate creation within zone.

- **Universal principles**
  - **Entity hierarchy**: Most platforms use a nested structure (zone contains countries, method references zone, calculator configures method).
  - **ID dependencies**: Later steps reference IDs from earlier steps (e.g. `shippingZoneIds` references zone `id`).
  - **Discovery queries**: Platforms often require querying available options (calculators, checkers) before configuration.
  - **Code-based configuration**: Operations are often identified by codes/keys rather than human-readable names.
  - **Normalized formats**: Money in smallest units, countries as ISO codes, IDs as strings/UUIDs.

---

### 4. Key Differences: Promotions vs Shipping

- **Promotions** (Track A):
  - Focus on **conditions** (when to apply) and **actions** (what discount to give).
  - Conditions and actions are both configurable operations.
  - No geographical hierarchy (though conditions can include geography).

- **Shipping** (Track B):
  - Focus on **geography** (zones, countries) and **pricing** (methods, calculators).
  - Requires **checkers** (eligibility) and **calculators** (cost computation).
  - Explicit **hierarchical nesting**: zones contain countries, methods reference zones.
  - **Two-step zone setup**: Create zone, then add countries (vs promotions which are created in one step).

- **Common patterns**:
  - Both use **configurable operations** (conditions/actions vs calculators/checkers).
  - Both require **discovery queries** before configuration.
  - Both use **string arguments** for numeric values (money, percentages).
  - Both use **dynamic ID references** to link entities.

This nested configuration approach ensures that shipping setups remain scalable, maintainable, and adaptable across different e-commerce platforms, while preserving the same network‑driven reverse‑engineering methodology used throughout this challenge.
