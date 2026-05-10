# LAB Excel → Schema Conversion Rules

_Learned from `Advertising licence (new).xlsx` ↔ `Advertising licence (new).txt`, then refined across Amend / Renew / Reduce._
_Use this as the recipe when a new licence Excel is uploaded._

## 0. Master template (canonical layout, as of 24 Apr 2026)

`Form Master Template.xlsx` in the workspace root is the canonical shape all new form Excels should follow. It is an 11-column layout:

| Col | Header                    | Role |
|-----|---------------------------|------|
| A   | Section name              | Section boundary. Empty = continuation of previous section. "Addable section…" text below the header signals a duplicable section (see §5). |
| B   | Pre-condition             | Natural-language show-if / require-if. Parsed per §6. |
| C   | Field type                | Field type token (see §3). |
| D   | Field name                | `attr.title`; `key` = camelCase of this. |
| E   | Field Description         | `attr.description`. |
| F   | Field values/options      | Bullet list of options. Also carries "Auto-populated" marker (pairs with Col G). |
| G   | **Retrieval**             | Where the value comes from. Values: `NA`, `MyInfo`, `Corppass`, `From AAPI "Retrieve"` / `From AAPI "<button>"`, `Postal code will populate from ONEMAP`, or free text describing the retrieval. Drives whether a field is part of `generalInfo` boilerplate (MyInfo/Corppass), an AAPI-populated field (`nonEditable: true`), or user-entered. |
| H   | Validation                | Drives `customValidation`, `limitNumber`, `startDateValidationConfig`, `emailConfirmation`, address postal-code flags, etc. |
| I   | Required / Optional       | `attr.required = true/false`. "All fields required" on Address also controls `withPostalCodeRequiredConfig.unit/floor/buildingName`. |
| J   | **Editable/Non-editable** | Distinct flag. "Non-editable" → `attr.nonEditable = true` (or `attr.readonly = true` on `licence_number`). "Editable" → no flag. |
| K   | Remarks                   | `attr.tooltip` when it says "Require tooltips"; otherwise a build-time note. |

**General Info template rows** (Master Template rows 2-24) describe the reusable boilerplate — Profile / Applicant Detail / Company Detail / Filer Detail — that every LAB form shares. These rows **do not** generate new `applicationDetail` entries; they confirm the standard `generalInfo` / `generalInfoLogics` blocks which are reused verbatim (see §2, §10).

**"Application Details" marker row** (Master Template row 25, Col A = "Application Details") signals the start of form-specific sections. Everything below becomes `applicationDetail` entries.

### 0a. Legacy BCA column layout (pre-master-template)

The earlier BCA Excels used a 9-column "field-only" layout followed by dated review columns (J…T). Map them as:

| Legacy Col | Master equivalent |
|-----------|-------------------|
| A Section name | A Section name |
| B Section/Field Logic | B Pre-condition |
| C Field type | C Field type |
| D Field name | D Field name |
| E Field Description | E Field Description |
| F Field values/options (also carries Auto-populated) | F Field values/options **+** G Retrieval (split: Auto-populated/AAPI tokens migrate to G) |
| G Validation (also carries "Non-editable") | H Validation **+** J Editable/Non-editable (split: Non-editable moves to J) |
| H Required / Optional | I Required / Optional |
| I Remarks | K Remarks |
| J…T Dated comment columns | **Not in master template.** Collapse each row to its latest resolved state before converting. |

**Rows to skip entirely**: rows whose Section column contains meta-instructions such as "Clicks on 'Review Form'", "END OF APPLICATION DETAILS", "START OF REVIEW FORM", payment instructions without a field, etc. These describe flow, not fields.

### 0b. Retrieval (Col G) → schema behaviour

| Retrieval value | Effect on the emitted field |
|-----------------|----------------------------|
| `NA` / empty    | User-entered field. Keep Col C's type as-is. |
| `MyInfo`        | Boilerplate field — already present in `generalInfo` (Applicant Detail / Filer Detail). Skip in `applicationDetail`. |
| `Corppass`      | Boilerplate field — already present in `generalInfo` (Company Detail). Skip in `applicationDetail`. |
| `From AAPI "<button>"` | AAPI-populated. Set `attr.nonEditable = true` on the field; the `<button>` name maps to an `api` button field elsewhere in the section. If the field is inside an addable section, use the AAPI-populated addable variant (§5B). |
| `Postal code will populate from ONEMAP` | For `address` fields — no extra flag, but `withPostalCodeRequiredConfig.buildingName = true` (the ONEMAP lookup drives auto-fill). |
| `EDH` (legacy) | Use the `edh_registered_address` type for a full address, or mark a scalar field `nonEditable: true`. |

### 0c. Editable (Col J) → schema behaviour

- `Non-editable` → `attr.nonEditable = true` (textfield/email/date/number/radiobutton/…), except on `licence_number` which uses `attr.readonly = true`.
- `Editable` / empty → no extra flag.
- Discrepancy between Col G (AAPI-populated) and Col J (Editable) → honour Col J. A field can be AAPI-prepopulated **and** user-editable (e.g. the Reduce form's per-row "Number of signs").

## 2. Top-level schema shape (always produced)

```json
{
  "declaration":         { …fixed boilerplate… },
  "generalInfo":         [ …fixed login/applicant/company/filer blocks… ],
  "applicationDetail":   [ …one entry per Excel section… ],
  "generalInfoLogics":   { …fixed show rules for login/profile… },
  "applicationDetailLogics": {
    "setIfLogics":    [],
    "showIfLogics":   [ …derived from column B… ],
    "requireIfLogics":[ …derived when required-state is conditional… ]
  }
}
```

The `declaration`, `generalInfo`, and `generalInfoLogics` blocks are **template boilerplate** — reused verbatim unless the Excel explicitly changes login/profile/applicant/filer/company fields.

## 3. Field-type mapping (column C → schema `type`)

| Excel Field type      | Schema `type`  | Notes on `attr` |
|-----------------------|----------------|-------------------------------------------------------------|
| Dropdown              | `dropdown`     | `options`, `hasOthers` (true only if "Others" listed + spec'd) |
| Radio / radiobutton   | `radiobutton`  | `options`, `hasOthers` |
| Short Text            | `textfield`    | +`customValidation` when Validation has max length |
| Long Text             | `textarea`     | Distinct type in LAB. Same `customValidation` pattern for max length. |
| Yes/No                | `yes_no`       | `metaTagYes`, `metaTagNo` (empty strings by default) |
| Checkbox              | `checkbox`     | `options` single-item → one checkbox; multi → list |
| Email                 | `email`        | `emailNotification`/`emailConfirmation` default `false` |
| Contact Number        | `contactnumber`| `allowIntContactNumber` true if Options says "international" |
| Address               | `address`      | `withPostalCodeRequiredConfig.{unit,floor,buildingName}` from Required rules. Native widget: user enters **Postal Code**, clicks **"Retrieve Address"**, the widget calls ONEMAP and auto-fills Block/House No., Street Name, Building Name (always non-editable). Floor/Level and Unit stay user-editable. See §9b for the master-template xlsx row layout. |
| ID Type               | `id_type`      | Combines ID Type + ID Number; title usually `""` |
| NRIC (standalone)     | `textfield`    | +`customValidation.type = "NRIC"` (or handled inside `id_type`) |
| Date                  | `date`         | `startDateValidationConfig.type = "Today"` if "must be Today or more" |
| Number                | `number`       | |
| Decimal               | `decimal`      | `limitNumber` = decimal places (e.g. "1") |
| Attachment            | `attachment`   | `size` = max MB (7 by default), `description` = allowed formats |
| Statement             | `statement`    | `description` = HTML of the instruction text + any links |
| Payment Mode          | `payment_mode` | `options` = payment modes |
| AAPI                  | `api`          | "Action API" button. `attr.title` = button label ("Retrieve", "Verify PTU", "Add"); `attr.description` = tooltip text; `attr.additionalApi` = API handle (placeholder, e.g. `"GOVTECH_TEST_ONE"`); `attr.requestElements` = map of input param → upstream field id; `attr.ableToEditApiWording = false`. Field has no `key`. |
| EDH-Address           | `edh_registered_address` | Non-user-editable address auto-populated from EDH. Uses the same `withPostalCodeRequiredConfig` (unit, floor, blockNo, postalCode, streetName, buildingName) defaulting to all `false`; typically `nonEditable: true`. |
| GoBiz-Licence No.     | `licence_number` | Dedicated LAB type for the licence number GoBiz prepopulates on Renew/Amend/etc. Uses `attr.readonly = true` and `required = true`. Title "Licence No.", key `licenceNo`. **Do not** use `textfield` + `nonEditable` for this — it's a distinct type with its own widget. |

## 4. IDs and keys

- **Section `id`** and **field `id`** use 9-char nanoid-style strings (lowercase alphanum), e.g. `a5cj9cd5d`. Generate fresh per field.
- Exception: the fixed `generalInfo` and `declaration` blocks use uppercase snake-case IDs (`GI_APPLICANT_DETAIL_ADDRESS`, `DECLARATION_FIELD`). Preserve those verbatim.
- **`key`** = lower-camelCase of the header/title, punctuation stripped (`"Is the Sign Owner the same as Event Owner?"` → `isTheSignOwnerTheSameAsEventOwner`).
- Addable sections include a system field `{ "id":"rowId", "key":"rowId", "type":"rowId", "attr":{"title":""} }` as the first field. AAPI-populated addable sections additionally include `{ "id":"rowStatus", "key":"rowStatus", "type":"rowStatus", "attr":{"title":""} }` immediately after `rowId`.

## 5. Addable sections

When Col A says something like:
```
<Header>
Addable section
- Min # of rows: 1
- Max # of rows: 100
```
produce on the section object:
```json
"enableEdit": true,
"enableDelete": true,
"addableRowIndex": true,
"minimumDuplication": 1,
"maximumDuplication": 100
```
Then pick **one of two variants** based on how rows are populated:

**A. User-entry addable** (rows the filer adds manually):
```json
"enableAdd": true,
"receiveSectionData": false
```
Include a system `rowId` field at the top of `fields[]`.

**B. AAPI-populated addable** (rows come back from a Retrieve action; filer can only edit/delete):
```json
"enableAdd": false,
"receiveSectionData": true
```
Include system `rowId` **and** `rowStatus` at the top of `fields[]`. Any field in Col F marked "value to be populated from AAPI …" gets `attr.nonEditable = true` (see §9 below). For these fields, Decimal from Col C usually becomes `textfield` (not `decimal`) — because the value is display-only strings retrieved from the API.

Also move any show-if / require-if that references fields **inside the same addable section** to the section's own `showIfLogics` / `requireIfLogics` (rather than the top-level `applicationDetailLogics`). Cross-section rules stay at the top level.

> **Note for reverse conversion (schema → xlsx):** When walking a LAB schema to produce a master-template xlsx, you MUST read **both** `applicationDetailLogics.showIfLogics` (top-level, cross-section rules) **and** each section's nested `showIfLogics` array (addable-internal rules). LAB's logic editor surfaces addable-internal rules under the section, not at the form level — so a converter that only reads top-level logics will silently drop dozens of rules. Same applies to `requireIfLogics` and `setIfLogics`. See §11 for the full three-layer logic model.

## 11. LAB's three logic layers (form / section / api)

Anthony, 25 Apr 2026: LAB stores **three** independent kinds of logic. A converter that only reads the form layer will silently miss the other two.

| Layer        | Where it lives in the schema                                              | What it controls                                              |
|--------------|---------------------------------------------------------------------------|---------------------------------------------------------------|
| **Form-logic**    | `applicationDetailLogics.{showIfLogics,requireIfLogics,setIfLogics}` (top-level) | Cross-section rules — e.g. "show Section X if Field Y in another section = Z" |
| **Section-logic** | Each section object's own `showIfLogics` / `requireIfLogics` / `setIfLogics` arrays (e.g. `applicationDetail[i].showIfLogics`) | Field-to-field rules **scoped inside that section**, especially addable sections. LAB's logic editor surfaces these under the section header, not at form level. |
| **API-logic**     | Each `api` button field's `attr.populateLogics` array                    | What the AAPI button does when fired: which API response field maps to which form field, and whether the populated value remains editable. |

A `populateLogics` entry shape:
```json
{
  "id": "<nanoid>",
  "editable": true,
  "apiFieldId": "<this api button's own field id>",
  "conditions": [{ "field": { "name": "<api response field name>", "type": "String" } }],
  "actionTargetIds": [ "<form field id to populate>" ]
}
```

**Reverse-conversion rule:** when emitting a populated form field's row in the master-template xlsx:
- Col G (Retrieval) → `From AAPI "<button title>"` (use the button title from the api field that owns the matching `populateLogics` entry).
- Col J (Editable) → `Editable` if `editable: true`, else `Non-editable`. The populate-level flag overrides the field's own `nonEditable` attr.
- Col K (Remarks) → `USE API field: <conditions[].field.name>` so the agency reviewer can see which response key drives the value.

On the api button's own row, add Col K → `Populates N field(s)` for quick scanning.

**Forward-conversion rule (Excel → schema):** if an Excel row's Col G says `From AAPI "<button>"`, generate a `populateLogics` entry on that button referencing this row's field id, with `editable` derived from Col J.

## 11a. Section-level multi-field validators (addable uniqueness)

Beyond the three logic layers, addable sections can also carry a **section-level multi-field validator** at `section.multiFieldsValidator` (with `section.multiFieldsCheck: true`). It's an array describing every field in the section plus per-field flags:

```json
{
  "id": "<field id>",
  "type": "dropdown" / "api" / "address" / ...,
  "label": "Teacher Names and Key",
  "isUnique": true,
  "isIdentical": false,
  "addressComp": [ { "id": "postalCode", "label": "Postal Code", "isUnique": false, "isIdentical": false }, ... ]
}
```

- `isUnique: true` → that field's value must be **unique across all rows** of the addable (e.g. you can't pick the same teacher twice).
- `isIdentical: true` → that field's value must be **identical across all rows** (rare, mostly for shared parent values).
- `addressComp` nests the same flags per address component (postalCode, blockHouseNo, streetName, buildingName, etc.) when the validated field is an `address`.

**Reverse-conversion rendering:** in the section header banner of the master-template xlsx, append:
```
- Unique across rows: <comma-separated labels>
- Identical across rows: <comma-separated labels>
```
so the agency reviewer can see the cross-row rules at a glance.

Other small metadata to surface when present:
- `section.headerComment` (seen in `generalInfo` only) — short note explaining when the section appears. Carries through to the LAB UI as helper text under the section header.
- Top-level `declaration` block has `attr.acknowledgement`, `attr.generalDeclaration`, `attr.additionalDeclaration`, `attr.hasAdditional` — boilerplate, reused verbatim.

## 6. Parsing column B "logic" into rules

Column B text looks like:
```
If "<trigger field>" = "<value>"
AND "<trigger field 2>" = "<value>"
SHOW this field / section
```
Convert to:
```json
{
  "id": "<nanoid>",
  "conditions": [
    { "state": "is equal to", "value": "<value>",  "fieldId": "<id of trigger field>", "fieldType": "<type of trigger field>" },
    …
  ],
  "actionTargetIds": [ "<id of this field or section>" ]
}
```
- Multiple lines joined with **AND** → multiple entries in the same `conditions` array.
- "Either … or …" / comma-separated values → use `"state": "is either"` with `value` as an array.
- "is NOT" / "is false" → `"state": "is not equal to"` (e.g. for unchecked).
- `actionTargetIds` is an array: group all fields that share the exact same condition set into one rule.
- A rule whose target is a whole section (no specific field) sets the `actionTargetIds` to that section's `id`.
- **Section + field belt-and-suspenders**: when a section-level condition controls whether a whole section shows, emit **two** rules with the same conditions — one with the section `id` as target, and a second with the explicit child field ids as targets (excluding any field that has its own nested show-if). This is how the existing LAB schemas wire it, and matching the pattern avoids render glitches when the engine re-evaluates logic.
- **Shared fields across mutually-exclusive branches**: if the Excel lists the same field (same title) under two different branches of a parent radio (e.g. "Licensee Name" under both "Individual" and "Company"), emit it **once** and list its id in **both** branches' `actionTargetIds`. Don't create two separate fields.

### require-if vs show-if
- A field whose _existence_ depends on a condition → `showIfLogics`.
- A field that is always shown but becomes mandatory conditionally (Col H reads e.g. "Required / Optional" with the "Required" branch tied to a condition) → `requireIfLogics`.

## 7. Options formatting

Column F bullets like:
```
- Event
- Non-event
```
→ `[{ "label": "Event" }, { "label": "Non-event", "metaTag": "" }]`

The first option has no `metaTag`; subsequent options carry `metaTag: ""` (empty placeholder) to match what LAB emits. Keep the order from Excel.

## 8. What I honour from the dated comment columns (J…T)

These columns are a _review history_. I only use them to **resolve the final state** of a field:
- "Add this field" / "Remove this field" that ends with "Done" → the field exists/does-not-exist accordingly.
- "Rename this field" / "Update field logic" resolved with "Done" → apply the rename/logic update.
- Unresolved/pending comments → leave the field in its current state and flag them to you (don't silently drop).

## 9. Small field-level conventions to remember

- `description` on a checkbox/radio with a long note → put it on `attr.description`, not on title.
- An email marked as the applicant's primary email uses `"replyTo": false` and `"emailConfirmation": false` by default.
- "Postal code" validation on an Address → `withPostalCodeRequiredConfig.buildingName = true`.
- A decimal with "1 decimal point" validation (user-entry) → `"limitNumber": "1"` (string, not number).
- An attachment row with no explicit size → default `"size": 7` (MB).
- **`nonEditable: true`** applies when any of these appear for a field:
  - Col F contains "value to be populated from AAPI …"
  - Col G (Validation) is "Non-editable"
  - Col I (Remarks) says "Uneditable" / "readonly field"
- For non-editable display-only numeric fields (e.g. retrieved Length/Breadth), use `textfield` with `nonEditable: true`, not `decimal`. User-entry decimals keep `decimal` + `limitNumber`.
- **Any non-editable AAPI-populated field with no user choice** (Col G "Non-editable" + Col I "Populated from AAPI") → emit as `textfield` with `nonEditable: true`, regardless of what Col C says. This applies to Dropdown/Radio/Decimal/Number rows that are purely display-only — the value is a string returned by the API and the user cannot select anything, so a plain text display is correct. (User-editable fields keep their original type, even if AAPI-prepopulated.)
- **Cross-field / cross-row validations that LAB cannot express** (e.g. "Number of signs must equal the sum of Sign Location rows", "Expiry must be less than 1 year from Start") → emit only `required: true` on the field and note that the agency's validation API (`validationAPI`) is expected to enforce it server-side. Do not invent a `customValidation` or `endDateValidationConfig` shape that LAB doesn't ship with.
- A Yes/No field inside an AAPI-populated section comes back as a `dropdown` with explicit `{label: "Yes"}, {label: "No"}` options (not `yes_no`), because the API returns a string. Honour whichever shape the row indicates.
- **ID Type + ID Number collapse**: when Excel lists an "ID Type" (Radio with NRIC/FIN/PASSPORT etc.) row immediately followed by an "ID. No." / "UEN/NRIC/FIN" Short Text row in the same logical group, collapse the two rows into a single `id_type` field with `attr.title = ""`, `required`, `nonEditable` (if the source says so), and `foreignIdEnabled: false`. Do **not** emit a separate radiobutton + textfield pair — LAB renders the whole thing from one `id_type` widget.
- **`readonly` vs `nonEditable`**: the `licence_number` type uses `attr.readonly = true` (its own flag). All other prepopulated/non-editable fields (textfield, email, textarea, date, radiobutton, etc.) use `attr.nonEditable = true`. Keep them distinct.

## 9b. User-entered Address — canonical multi-row layout

Anthony, 24 Apr 2026: in Application Details, every user-entered `address` field follows the same ONEMAP-retrieve pattern. The LAB `address` widget handles this natively (one JSON field), but the **master-template xlsx documents it as a multi-row group** — mirror that layout exactly:

| Row | Col D (Field name)                                   | Col G (Retrieval)                            | Col I (Required)                              | Col J (Editable)   |
|-----|------------------------------------------------------|----------------------------------------------|-----------------------------------------------|--------------------|
| 1   | `Postal Code`                                        | `Postal code will populate from ONEMAP`      | `Required` (unless "Enable without Postal Code") | `Editable`         |
| 2   | Bullet list: `- Block/House No.` `- Street Name` `- Building Name` | `Postal code will populate from ONEMAP` | All three typically Required (Building Name may be Optional) | **`Non-editable`** (always) |
| 3   | `Floor/Level`                                        | `Nil` / `NA`                                 | `Optional` (default)                          | `Editable`         |
| 4   | `Unit`                                               | `Nil` / `NA`                                 | `Optional` (default)                          | `Editable`         |

Rules:
- Collapse the three retrievable fields into **one** Col D row with a bullet list (`- Block/House No.\n- Street Name\n- Building Name`). Do not emit them on separate rows.
- Block/House No., Street Name, Building Name are **always Non-editable** regardless of source — their values come from ONEMAP after the user retrieves.
- The "Retrieve Address" button UX is implicit in the `address` widget — do not emit it as a separate `AAPI` row in the xlsx.
- In the LAB schema this all becomes a single `{ "type": "address" }` field with `withPostalCodeRequiredConfig` set per the Required/Optional flags.
- Applies equally to Licensed Premises Address, Registered Business Address (when user-entered, not Corppass-populated), Local branch of Residential Address, and any other user-entered address block.

Exception: address blocks that are **fully Corppass/AAPI-populated** (like Company Detail's address in General Info) keep all rows including Postal Code as Non-editable — no retrieve button is needed because the whole block comes from the API.

## 9c. Agency-specific address-as-textfields override

Anthony, 25 Apr 2026: some agencies deliberately model an address block as **separate `textfield` fragments** (Postal Code, Building Name, Block/House No, Street Name, Main Unit, Additional Unit, etc.) **instead of** a single `address` field. This is intentional — the agency wants stricter per-fragment validation, addable-row friendliness, or custom storage shape.

Examples seen: SSG PEI's "Particular of Premises List" addable section.

Rules:
- **Do not auto-collapse** these into the §9b 4-row address layout. Leave each Short Text / Number row as-is in the master-template xlsx.
- The schema stays as individual `textfield` (or `number`) fields — do not rewrite to a single `address` field.
- Treat any agency-modeled address-as-textfields the same way: faithfully mirror the schema. If unsure, ask Anthony.

## 9d. Numeric `maxRange` as char-limit hack

Anthony, 25 Apr 2026: agencies sometimes encode a "max N characters" requirement on a `number` / `decimal` field by setting `maxRange` to a string of `9`s of the desired length (e.g. `maxRange: "999999999999999999"` = max 18 chars; `maxRange: "9999999999999999999999999999999999.99"` = max 37 chars including the decimal point and 2 decimal places).

This isn't a true numeric range — it's a backdoor character-count cap because LAB's `decimal`/`number` widgets don't ship a native `customValidation: charLimit`.

Rules:
- **Detection**: `maxRange` is a string consisting only of `9`s, optionally containing one `.` (e.g. `"999"`, `"9999.99"`, `"99999999999999999999"`).
- **Render in the master-template xlsx**: translate to **`Max <len(maxRange)> characters`** in Col H (where `<len>` is the literal character count of the maxRange string, decimal point and digits included). Suppress the raw range value.
- Real numeric ranges (e.g. `maxRange: "7.0"`, `maxRange: "100"`) keep their numeric rendering.
- `minRange: "0"` is the default and can be suppressed when paired with a real `maxRange` value (don't render "Min 0").
- The schema field itself stays as `decimal` / `number` with the `maxRange` value untouched — the translation is for the readable xlsx only.

## 9a. Cross-form reuse via Q/R resolution

When an agency Q asks "can we remove this section if we already have it elsewhere?" and Anthony's R reply points at another licence form (e.g. "change this to according to the 'Sign Agent Information' as reflected in the 'Advertising licence (new)' form"), **replace** the Excel's section content with the referenced form's section structure verbatim (fresh IDs, same field shapes and keys). The Excel rows for the original section are superseded — do not emit them as individual fields.

## 10. Confirmed defaults (from Anthony, 21 Apr 2026)

- **Long Text** → schema type `textarea` (separate from Short Text).
- **Boilerplate** (`declaration`, `generalInfo`, `generalInfoLogics`): reused verbatim across licences unless the Excel explicitly changes login / applicant / filer / company fields.
- **IDs**: fabricate fresh 9-char lowercase nanoid-style strings for sections and fields; keep the uppercase snake-case IDs for the reused boilerplate (`GI_*`, `DECLARATION_*`).
- **Pending review comments** (no "Done" yet in cols J–T): build the schema from the current row state, and surface any unresolved comments in a separate note alongside the output so you can decide.
- **GoBiz-Licence No.** → `licence_number` (with `readonly: true`), not `textfield`.
- **ID Type + ID No.** rows collapse into a single `id_type` field (title `""`, `foreignIdEnabled: false`).
- **Shared fields across mutually-exclusive radio branches** (e.g. "Licensee Name" under both Individual and Company) = one field, listed in both branches' `actionTargetIds`.
- **Section show-if = section target + child field targets**: emit both rules with the same conditions when a section is conditionally shown.
- **Non-editable + AAPI-populated** fields — emit as `textfield` + `nonEditable: true`, overriding Col C (Dropdown/Radio/Decimal/Number). LAB shows these as plain read-only text.
- **Cross-field / cross-row arithmetic validations** are not expressible in LAB schema — emit `required` only and rely on the agency's validation API.

## 12. LAB schema authoring style (after-action review, NParks COPA Farm 25 Apr 2026)

Lessons baked in after Anthony's revision pass on my generated NParks COPA schema:

### 12a. Use dedicated types, not generic + flag

- **Registered addresses prepopulated by EDH/Corppass** → use `edh_registered_address`, **not** `address` + `nonEditable: true`. The dedicated type renders correctly in LAB and signals provenance. Same principle as `licence_number` (don't fake it with `textfield` + `readonly`).
- Reserve `address` (with `withPostalCodeRequiredConfig`) for user-entered addresses with ONEMAP retrieve.

### 12b. Prefer `radiobutton` over `yes_no` when the field is referenced in show-if logic

- `yes_no` is a binary widget. `radiobutton` with explicit `options: [{"label": "Yes"}, {"label": "No", "metaTag": ""}]` is more flexible: carries `hasOthers`, supports later option additions, and references cleanly in show-if `value: "Yes"` / `"No"`.
- Default to `radiobutton` for any Yes/No question that gates other fields. Reserve `yes_no` for cases where the binary is purely informational (no downstream rules).

### 12c. Don't create wrapper sections for single toggles

- A toggle (e.g. "Is billing address same as mailing?") that controls the visibility of a sibling section should be **the first field of that sibling section**, not its own one-field section.
- Two-field section: `[radiobutton: "Same as mailing?"]`, then `[address: ...]` with show-if `Same as mailing? = No`.
- Avoid the pattern of a 1-field "Billing Address Toggle" section preceding a 1-field "Billing Address" section.

### 12d. No `Declarations` section in `applicationDetail`

- Declarations live at schema root in the dedicated `declaration` block (`DECLARATION_SECTION` / `DECLARATION_FIELD`). That block is the canonical home for general declarations.
- Do **not** add a parallel "Declarations" section under `applicationDetail` — it duplicates the root block.
- Exception: agency-specific declarations tied to a particular flow (e.g. Muis Halal's "Other Declaration - Trading Items") that are **substantively different** from the standard general declaration. Those may live in `applicationDetail` as named sections.

### 12e. Positive show-if (`is equal to "Yes"`) over negative (`is not equal to "No"`)

- For gating logic where downstream fields should only appear after the user explicitly answers Yes, write the rule as `IF "<field>" is equal to "Yes" → SHOW [targets]`.
- The negative form (`is not equal to "No"`) treats unanswered/null state as "show", which is wrong for progressive disclosure.
- Use negative show-if only when the default-visible behaviour is what you want (rare).

### 12f. Section-level showIfLogics are for addable sections only

- For non-addable sections, **always put cross-section gating at the form level** (`applicationDetailLogics.showIfLogics`).
- Form-level rules can target individual field IDs (not just section IDs) in `actionTargetIds`. One rule like `IF landowner = "Yes" → SHOW [field1, field2, field3, field4, ...]` across multiple sections is cleaner than 4 section-wrapped rules.
- Section-level `showIfLogics` is reserved for rules **scoped inside an addable section's row template** (per §5). Putting non-addable logic there clutters the section object and obscures the form-level control flow.

### 12g. Section-level logic targeting individual fields → may belong at form level

- If a section's nested `showIfLogics` only targets the section's own fields (not nested addable rows), and the section isn't addable, lift those rules to form-level `applicationDetailLogics.showIfLogics`. The behaviour is identical and the schema is easier to scan.

## 13. Output format

Final deliverable is a single `.txt` (pretty-printed JSON, 4-space indent) matching the layout of `Advertising licence (new).txt`:
- Keys in the order: `declaration`, `generalInfo`, `applicationDetail`, `generalInfoLogics`, `applicationDetailLogics`.
- Inside each section: `id`, `key`, `fields`, `header`, then any addable flags, then any section-level logics, then `menuLabel`.
- Inside each field: `id`, `key`, `attr{}`, `type`.

---

## 14. Agency Review Workflow (Excel columns L–N)

_This section governs how Ah Seng processes agency feedback in the master-template Excel before generating the schema `.txt`. It applies to every licence form across all 200+ LAB forms._

### 14a. Column definitions

The master template has three agency-facing columns appended to the right of the standard 11-column layout:

| Col | Header | Owner | Purpose |
|-----|--------|-------|---------|
| L | **Agency Comment** | Agency | Plain-English description of what the agency wants changed on this row. |
| M | **Change Type** | Agency | Structured tag picked from the dropdown. Tells Ah Seng _what kind_ of change to apply. |
| N | **Status** | Ah Seng | Resolution state. Ah Seng sets this after processing each comment. |

**Conversion rule**: Ah Seng reads A–K to generate the schema. Columns L–N are the change-instruction layer. Before generating the `.txt`, Ah Seng must:
1. Scan every row for a non-empty col L or col M.
2. Apply the change to cols A–K of that row (or neighbouring rows if adding/removing rows).
3. Set col N = `Done` when successfully applied, `Rejected (reason)` if the request is invalid or contradictory.
4. Generate the schema from the updated A–K state.
5. Deliver the `.txt` plus a **Change Log** (see §14g) listing every L–N row it processed.

Rows where L and M are both empty are treated as confirmed — no action needed.

---

### 14b. Change Type vocabulary and Ah Seng's response per type

#### `ADD_OPTION`
**What agency writes in L:** `"Add option: Permanent"` / `"Add: Others (please specify)"` / `"Include 'N/A' as a choice"`

**Ah Seng does:**
- In col F, append the new option to the existing bullet list: `- <new option>`.
- If the option includes "others" / "please specify", also verify `hasOthers: true` will be emitted in the schema.
- If the field type (col C) is Short Text or something other than Dropdown/Radio/Checkbox, flag as `Rejected` — options only apply to those types.

---

#### `REMOVE_OPTION`
**What agency writes in L:** `"Remove 'Foreign Military Units'"` / `"Delete the 'N/A' option"`

**Ah Seng does:**
- In col F, delete the matching bullet line.
- If the removed option is currently referenced in col B (logic / pre-condition) of this or another row, flag the dependency explicitly in the Change Log and ask for clarification before removing.

---

#### `RENAME_OPTION`
**What agency writes in L:** `"Rename 'Non-event' to 'Permanent Sign'"` / `"Change label: Event → Temporary Event"`

**Ah Seng does:**
- In col F, find the matching bullet and update its label.
- Check col B of all rows — if the old label appears in any logic rule, update it there too and note it in the Change Log.

---

#### `CHANGE_REQUIRED`
**What agency writes in L:** `"Change to Optional"` / `"Make Required"` / `"Required only if [field X] = Yes"`

**Ah Seng does:**
- Simple flip (Required ↔ Optional): update col I.
- Conditional required: update col I to `Optional` (base state) and add a `requireIfLogics` note in col B.
- If the field is in `generalInfo` (rows 2–23 of the standard template), reject — generalInfo required-states are fixed boilerplate (see §2).

---

#### `CHANGE_EDITABLE`
**What agency writes in L:** `"Make Non-editable"` / `"Should be editable — remove lock"` / `"Pre-populated from AAPI, non-editable"`

**Ah Seng does:**
- Update col J to `Non-editable` or `Editable`.
- If switching to Non-editable and col G (Retrieval) is `NA` / empty (i.e. no retrieval source), flag in Change Log: _"Non-editable field has no retrieval source — confirm where the value comes from."_
- If the change conflicts with col G (e.g. col G says `MyInfo` but agency requests `Editable`), apply col J as instructed but note the conflict.

---

#### `RENAME_FIELD`
**What agency writes in L:** `"Rename to: Event Organiser Name"` / `"Change field label to 'UEN / NRIC / FIN'"`

**Ah Seng does:**
- Update col D with the new name.
- Derive the new `key` (camelCase of new name) and note it in the Change Log — the schema `key` will change, which may affect any downstream system referencing it by key.

---

#### `RENAME_DESCRIPTION`
**What agency writes in L:** `"Change description to: 'Enter the full legal name of the event organiser'"`

**Ah Seng does:**
- Update col E with the new description text.

---

#### `RENAME_SECTION`
**What agency writes in L:** `"Rename section to: Sign Owner & Agent Information"` / `"Change header: 'Event Details' → 'Event Information'"`

**Ah Seng does:**
- Find the row where col A contains the current section name.
- Update col A to the new name.
- Note in Change Log: the schema `header` and `menuLabel` will change accordingly.

---

#### `ADD_FIELD`
**What agency does:**
- Inserts a blank row at the exact position they want the new field to appear.
- Fills in cols A–K themselves using their own words (not required to follow strict format).
- Puts `ADD_FIELD` in col M. Col L is optional — used for any extra context or notes.

**Ah Seng does — Step 1: Normalise col C (Field Type)**

Agencies will write field types inconsistently. Map any of the following to the canonical schema type token:

| Agency may write | Normalise to (col C token) | Schema type |
|---|---|---|
| text, free text, short text, textbox, input, string, single line | `Short Text` | `textfield` |
| long text, paragraph, multi-line, textarea, text area, comments box | `Long Text` | `textarea` |
| dropdown, drop down, drop-down, select, list, picklist | `Dropdown` | `dropdown` |
| radio, radio button, single select, choose one, options | `Radio` | `radiobutton` |
| yes/no, yes no, boolean, toggle, y/n | `Yes/No` | `yes_no` |
| checkbox, check box, tick box, multi-select, multiple select | `Checkbox` | `checkbox` |
| email, email address, e-mail | `Email` | `email` |
| contact, contact no, contact number, phone, phone number, mobile, tel | `Contact No.` | `contactnumber` |
| date, date picker, calendar | `Date` | `date` |
| number, integer, numeric, whole number | `Number` | `number` |
| decimal, float, amount, measurement | `Decimal` | `decimal` |
| attachment, file, upload, document, supporting doc | `Attachment` | `attachment` |
| statement, info, notice, instruction, read-only text | `Statement` | `statement` |
| address, postal, location | `Address` | `address` |

If col C is blank or cannot be mapped to any of the above, set Status = `Rejected (field type unclear — agency wrote: "<raw text>". Anthony to clarify.)` and skip this row.

**Ah Seng does — Step 2: Normalise col D (Field Name)**

- Strip trailing punctuation, excessive spaces, hashtags (#), and asterisks (*).
- Preserve the agency's intended name otherwise — do not rename fields.
- If col D is blank, set Status = `Rejected (field name missing)`.

**Ah Seng does — Step 3: Normalise col I (Required / Optional)**

Map any of the following:

| Agency may write | Normalise to |
|---|---|
| yes, y, required, mandatory, compulsory, must, needed, not optional | `Required` |
| no, n, optional, not required, not mandatory, can be empty, leave blank | `Optional` |

If col I is blank, default to `Required` and note in Change Log: _"Required/Optional not specified — defaulted to Required. Anthony to confirm."_

If col I cannot be interpreted, set Status = `Rejected (Required/Optional unclear — agency wrote: "<raw text>".)`.

**Ah Seng does — Step 4: Normalise col J (Editable / Non-editable)**

| Agency may write | Normalise to |
|---|---|
| yes, y, editable, can edit, user input, user entry, blank/empty | `Editable` |
| no, n, non-editable, non editable, read only, readonly, locked, prepopulated, auto, auto-populated | `Non-editable` |

If col J is blank, default to `Editable`.

**Ah Seng does — Step 5: Normalise col F (Options)**

Only applies if col C normalised to Dropdown / Radio / Checkbox.

- Agency may write options as a bullet list, numbered list, comma-separated, or line-separated. Normalise all to the bullet format: `- Option 1\n- Option 2`.
- If col C requires options but col F is blank, set Status = `Rejected (field type is Dropdown/Radio/Checkbox but no options provided. Agency to fill in col F.)`.

**Ah Seng does — Step 6: Normalise col B (Logic / Pre-condition)**

Agency may write logic in any of these forms:
- `"Show if X = Yes"` / `"Only show when Type = Event"` / `"Display if field A is B"`
- `"Required if..."` / `"Mandatory when..."`
- `"Hide if..."` / `"Don't show unless..."`

Normalise to the standard col B format:
```
IF "<field name>" = "<value>" SHOW this field
IF "<field name>" = "<value>" REQUIRE this field
```

If the referenced trigger field name does not exist anywhere in col D of the sheet, flag in Change Log: _"Logic references field '<name>' which does not exist in this form. Anthony to confirm field name."_ Do not reject — include the field in the schema but flag the unresolved logic reference.

If col B is blank, no logic rule is emitted for this field.

**Ah Seng does — Step 7: Validate and finalise**

Run this checklist on the normalised row before accepting:

- [ ] Col C maps to a valid type token ✓
- [ ] Col D has a non-empty field name ✓
- [ ] Col I is `Required` or `Optional` ✓
- [ ] Col J is `Editable` or `Non-editable` ✓
- [ ] Col F has options if type is Dropdown / Radio / Checkbox ✓
- [ ] Col B logic references existing field names ✓

If all pass → set col N = `Done`. Include the new field in the schema with a fresh 9-char nanoid.

If any fail → set col N = `Rejected (<specific issue>)`. List every failed check, not just the first one, so the agency can fix everything in one round.

**Always include every ADD_FIELD row in the Change Log**, even if accepted, noting:
- What the agency wrote (raw)
- What Ah Seng normalised it to (after cols A–K cleanup)
- Any assumptions made (e.g. defaulted Required, defaulted Editable)

This gives Anthony a single place to spot-check all agency-added fields before sign-off.

---

#### `REMOVE_FIELD`
**What agency writes in L:** `"Remove this field"` / `"Delete — not applicable to our licence type"`

**Ah Seng does:**
- Mark the row for deletion: set col A–K values to a consistent sentinel (`[REMOVED]` in col D) so the row is skipped during schema generation, OR physically delete the row.
- **Preferred approach**: physically delete the Excel row so the template stays clean.
- Check if this field is referenced in any col B logic rule on another row. If yes: flag in Change Log and ask whether to remove that logic rule too, or leave it pointing to a now-deleted field (which would be a schema error).

---

#### `CHANGE_TYPE`
**What agency writes in L:** `"Change from Short Text to Dropdown"` / `"Should be a Yes/No, not a Radio"` / `"Convert to Long Text (textarea)"`

**Ah Seng does:**
- Update col C to the new field type token (using the vocabulary from §3).
- If switching to Dropdown/Radio/Checkbox and col F (options) is empty, flag: _"Options not provided — agency must specify the available choices."_
- If switching from Dropdown/Radio to Short Text, clear col F (options no longer applicable).
- Apply any type-specific attr rules from §3 for the new type.

---

#### `CHANGE_VALIDATION`
**What agency writes in L:** `"Max 200 characters"` / `"Must be a future date"` / `"International contact numbers allowed"` / `"Remove max length restriction"`

**Ah Seng does:**
- Update col H with the new validation rule in the same natural-language format that §3/§8/§9 already use.
- Examples of how to write col H:
  - `Max 200 characters`
  - `Must be today or later`
  - `International allowed`
  - `Max 7 MB; PDF, JPG, PNG only`

---

#### `ADD_LOGIC`
**What agency writes in L:**
```
Show this field only if 'Is the Sign Owner the same as Event Owner?' = No
```
or
```
Required if 'Sign Duration' = 'Permanent'
```

**Ah Seng does:**
- Update col B with the logic rule in the standard format used across the template:
  ```
  IF "<trigger field>" = "<value>" SHOW / REQUIRE this field
  ```
- If there is already a logic rule in col B, append the new condition (AND / OR as specified).
- The schema logic entry will be emitted under `showIfLogics` or `requireIfLogics` at generation time per §6.

---

#### `REMOVE_LOGIC`
**What agency writes in L:** `"Remove the show-if condition"` / `"This field should always be visible"` / `"Remove the required-if rule"`

**Ah Seng does:**
- Clear col B on the specified row (or the specific condition within col B if multiple conditions exist and the agency has specified which one).

---

#### `CHANGE_LOGIC`
**What agency writes in L:**
```
Change trigger: was 'Type = Event', now 'Type = Temporary Sign OR Event'
```

**Ah Seng does:**
- Update col B with the revised condition.
- Note in Change Log what the old condition was and what it was changed to.

---

#### `OTHER`
**What agency writes in L:** Anything not covered above — freeform description.

**Ah Seng does:**
- Attempt to interpret the intent and apply the closest applicable change.
- If the intent is clear, apply it and document the interpretation in the Change Log.
- If ambiguous, set Status = `Rejected (clarification needed: <question>)` and surface to Anthony for decision.
- Do not silently guess on `OTHER` requests that touch logic or field type — always flag these.

---

### 14c. Conflict resolution rules

When agency comments create contradictions, apply these tie-breakers in order:

1. **Later row wins for same-row edits** — if two rows in the same section request conflicting changes (e.g. one says Required, another for the same field says Optional), honour the lower row number (i.e. first occurrence) and flag the conflict.
2. **Explicit beats implicit** — if col M = `CHANGE_REQUIRED` and col L says "Make Required", that overrides any implied required-state from an `ADD_LOGIC` comment on another row.
3. **Reject + flag, never silently guess** — if a conflict cannot be resolved by rules 1–2, set both rows to `Rejected (conflict with row <N>: <description>)` and surface to Anthony.

---

### 14d. generalInfo and declaration are protected

Rows in the General Info section (rows 2–23 of the standard template — Profile, Applicant Detail, Company Detail, Filer Detail) and the Declaration block are **boilerplate shared across all LAB forms**. Agency comments on these rows must be **rejected** with the message:

> `Rejected — generalInfo / declaration is shared boilerplate. Changes here affect all LAB forms. Escalate to Anthony.`

The one exception: if the agency explicitly notes that their form omits the "For Companies" login type (i.e. individuals-only form), Ah Seng may note this in the Change Log but should not modify the boilerplate rows — that is a LAB platform configuration, not a schema-field change.

---

### 14e. Multi-row changes (ADD_FIELD and REMOVE_FIELD)

**Adding a field** inserts a new row. If the agency's comment is on an existing row (the row they want the new field placed after), the new row goes directly below that row. All subsequent rows shift down by one.

**Removing a field** deletes the row entirely. After deletion, check every remaining col B across the sheet for references to the deleted field's name — if found, flag in Change Log.

**Address blocks** (the 4-row ONEMAP layout from §9b) are treated as a unit. Removing "Postal Code" removes all four address rows. Adding an address field inserts all four rows. Never partially remove an address block.

---

### 14f. Addable section comments

If the commented row is inside an addable section (col A contains "Addable section"), all the same rules apply with one addition: any `ADD_LOGIC` or `CHANGE_LOGIC` that references fields inside the same addable section must be emitted as section-level logic (not form-level), per §5 and §12f.

---

### 14g. Change Log format

After processing all comments, Ah Seng must output a **Change Log** alongside the `.txt`. Format:

```
CHANGE LOG — <Licence Name> — <Date>
============================================================

ROW <N> | <Field Name> | <Change Type> | Status: Done / Rejected
  Requested: <verbatim col L text>
  Applied:   <what was changed in cols A–K>
  Notes:     <any dependency flags, assumptions, or rejection reason>

ROW <N> | ...
```

Rows where L and M were empty are omitted from the Change Log (nothing to report).

If there are zero comments, output:
```
CHANGE LOG — <Licence Name> — <Date>
No agency comments found. Schema generated from current Excel state.
```

---

### 14h. Applying this workflow to a new licence (the 200-form scale)

For any new licence form — not just Advertising:

1. **Start from the canonical master template** (`Form Master Template.xlsx`). Do not start from a previous licence's file.
2. Populate cols A–K for the new form's fields (this is the "Ah Seng generates Excel from scratch" or "agency provides their own layout" step).
3. Distribute the populated Excel to the agency. Agency fills cols L and M only.
4. Ah Seng receives the Excel back, applies §14b rules, generates the `.txt` and the Change Log.
5. Anthony reviews the Change Log and the `.txt`. If everything is `Done`, the schema is final.
6. If any rows are `Rejected`, Anthony resolves them, updates L accordingly, and Ah Seng re-runs.

This cycle replaces all manual back-and-forth edits to the `.txt` file directly.

---

### 14i. What Ah Seng must never do without flagging

- **Never** silently drop an unresolved comment (Status blank or `Pending`). If a comment exists, it must end up as `Done` or `Rejected`.
- **Never** modify the `generalInfo`, `declaration`, or `generalInfoLogics` blocks based on agency comments (see §14d).
- **Never** invent a new field ID — always use fresh nanoid-style 9-char strings generated at conversion time (§4).
- **Never** change a field `key` without noting it in the Change Log — keys are used by downstream systems.
- **Never** silently ignore an `ADD_FIELD` comment because the description is incomplete — always set `Rejected` with a specific list of missing information.
