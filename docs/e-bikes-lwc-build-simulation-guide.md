# E-Bikes LWC Rebuild And Learning Guide

This guide is based on the repository contents and is aimed at rebuilding the application in a fresh Salesforce Developer Org using VS Code and Salesforce CLI.

No repository files were modified during this analysis.

Important distinction:

- **Learning rebuild:** Create metadata gradually in dependency order so you understand each Salesforce concept.
- **Faithful repository installation:** The original metadata contains page activations and cross-references, so the safest final deployment is deploying `force-app` together after the org prerequisites are complete.

The project uses Salesforce API version `65.0`. The installed CLI in this workspace supports the current command forms used below, such as `sf data import tree` and `sf apex run test`.

---

## 1. What The Application Does

E-Bikes is a fictional electric bicycle manufacturer application with two audiences:

| Audience | Experience |
|---|---|
| Internal staff | Lightning app for product browsing, reseller orders, Accounts, and product details |
| Site visitors | Experience Cloud site for browsing products and creating Cases |

The application teaches:

- Custom objects, fields, lookups, and master-detail relationships
- Lightning Web Components
- Apex controllers called from LWCs
- Lightning Data Service and UI Record API
- Lightning Message Service
- Lightning App Builder pages
- Experience Cloud configuration
- Guest-user security
- Platform Events and EMP API
- Jest and UTAM testing

---

# 2. Repository Structure

| Folder/File | Purpose |
|---|---|
| [force-app/main/default](<../force-app/main/default>) | Salesforce source metadata |
| [objects](<../force-app/main/default/objects>) | Data model: objects, fields, list views |
| [classes](<../force-app/main/default/classes>) | Apex backend and Apex tests |
| [lwc](<../force-app/main/default/lwc>) | Lightning Web Components and Jest tests |
| [flexipages](<../force-app/main/default/flexipages>) | Lightning App Builder pages |
| [layouts](<../force-app/main/default/layouts>) | Record page layouts |
| [applications](<../force-app/main/default/applications>) | E-Bikes Lightning application |
| [tabs](<../force-app/main/default/tabs>) | Navigation tabs |
| [permissionsets](<../force-app/main/default/permissionsets>) | User permissions |
| [messageChannels](<../force-app/main/default/messageChannels>) | LWC cross-component messages |
| [experiences/E_Bikes1](<../force-app/main/default/experiences/E_Bikes1>) | Experience Builder bundle |
| [networks](<../force-app/main/default/networks>) | Experience Cloud network |
| [sites](<../force-app/main/default/sites>) | Salesforce Site metadata |
| [data](<../data>) | Accounts, product families, products, import plan |
| [guest-profile-metadata](<../guest-profile-metadata>) | Guest profile and sharing rules deployed after site publication |
| [config/project-scratch-def.json](<../config/project-scratch-def.json>) | Scratch-org configuration; useful reference for Developer Org setup |
| [package.json](<../package.json>) | Jest, ESLint, Prettier, and UTAM tooling |

---

# 3. Application Dependency Map

```text
Account
  |
  +-- Order__c
        |
        +-- Order_Item__c ---- Product__c ---- Product_Family__c

Case ---- Product__c

Manufacturing_Event__e --> orderStatusPath LWC --> Order__c.Status__c
```

UI dependencies:

```text
ProductController Apex
  --> productTileList
       --> productTile
       --> paginator
       --> placeholder
       --> errorPanel

ProductsFiltered message channel
  --> productFilter publishes filters
  --> productTileList receives filters

ProductSelected message channel
  --> productTileList publishes selected product
  --> productCard receives selected product

OrderController Apex
  --> orderBuilder
       --> orderItemTile
       --> placeholder
       --> errorPanel

ProductRecordInfoController Apex
  --> heroDetails
       --> hero
```

---

# 4. Salesforce Metadata Inventory

## 4.1 Data Model

### Custom Objects And Platform Event

| Object | Label | Purpose |
|---|---|---|
| `Product_Family__c` | Product Family | Groups related bicycle models |
| `Product__c` | Product | Stores individual bicycle models and specifications |
| `Order__c` | Reseller Order | Stores an order placed by a reseller Account |
| `Order_Item__c` | Order Item | Stores products and quantities within an order |
| `Manufacturing_Event__e` | Manufacturing Event | Sends order status updates to the UI |
| `Case` extension | Standard Case | Supports product-related customer support cases |

### Relationships

| Child | Field | Parent | Type | Meaning |
|---|---|---|---|---|
| `Product__c` | `Product_Family__c` | `Product_Family__c` | Lookup | A product optionally belongs to a family |
| `Order__c` | `Account__c` | `Account` | Required Lookup | An order belongs to a reseller |
| `Order_Item__c` | `Order__c` | `Order__c` | Master-Detail | Items belong to an order |
| `Order_Item__c` | `Product__c` | `Product__c` | Lookup | An item references a bicycle |
| `Case` | `Product__c` | `Product__c` | Lookup | A support case can refer to a bicycle |

### Fields

| Object | Important Fields | Purpose |
|---|---|---|
| `Product_Family__c` | `Category__c`, `Description__c` | Classify and describe families |
| `Product__c` | `MSRP__c`, `Picture_URL__c`, `Category__c`, `Level__c`, `Material__c` | Browse/filter product catalog |
| `Product__c` | `Battery__c`, `Charger__c`, `Motor__c`, `Fork__c`, brake and color fields | Product specifications |
| `Order__c` | `Account__c`, `Status__c` | Customer and manufacturing lifecycle |
| `Order_Item__c` | `Order__c`, `Product__c`, `Price__c`, `Qty_S__c`, `Qty_M__c`, `Qty_L__c` | Ordering bicycle sizes and prices |
| `Case` | `Product__c`, `Case_Category__c` | Product support categorization |
| `Manufacturing_Event__e` | `Order_Id__c`, `Status__c` | Identify order and new status |

### Picklist Values

| Field | Values |
|---|---|
| `Product_Family__c.Category__c` | Commuter, Hybrid, Mountain |
| `Product__c.Category__c` | Mountain, Commuter |
| `Product__c.Level__c` | Beginner, Enthusiast, Racer |
| `Product__c.Material__c` | Aluminum, Carbon |
| Product color fields | white, red, blue, green |
| `Order__c.Status__c` | Draft, Submitted to Manufacturing, Approved by Manufacturing, In Production |
| `Case.Case_Category__c` | Mechanical, Electrical, Electronic, Structural, Other |

Relevant files:

- [Product_Family__c](<../force-app/main/default/objects/Product_Family__c>)
- [Product__c](<../force-app/main/default/objects/Product__c>)
- [Order__c](<../force-app/main/default/objects/Order__c>)
- [Order_Item__c](<../force-app/main/default/objects/Order_Item__c>)
- [Manufacturing_Event__e](<../force-app/main/default/objects/Manufacturing_Event__e>)
- [Case extensions](<../force-app/main/default/objects/Case>)

---

## 4.2 Apex Backend

| Apex Class | Purpose | Used By |
|---|---|---|
| `PagedResult` | Generic paginated response wrapper | `ProductController` |
| `ProductController` | Queries filtered products and similar products | `productTileList`, `similarProducts` |
| `OrderController` | Retrieves order items and related product details | `orderBuilder` |
| `ProductRecordInfoController` | Resolves a product or family name to a record ID and type | `heroDetails` |
| `HeroDetailsPositionCustomPicklist` | Supplies left/right design choices in Experience Builder | `hero` design configuration |
| `CommunitiesLandingController` | Redirects the site landing page into the community | Visualforce landing page |
| `TestProductController` | Tests product filters and similar-product queries | Apex verification |
| `TestOrderController` | Tests order-item retrieval | Apex verification |
| `CommunitiesLandingControllerTest` | Tests site landing controller | Apex verification |

Important learning points:

- Controllers use `with sharing`.
- SOQL queries use `WITH USER_MODE`.
- `@AuraEnabled(Cacheable=true)` allows LWC wire calls.
- `ProductController` builds dynamic query criteria and paginates nine records per page.

Files: [classes](<../force-app/main/default/classes>)

---

## 4.3 Lightning Web Components

### Foundation Components

| Component | Purpose | Teaches |
|---|---|---|
| `ldsUtils` | Normalizes LDS/Apex error messages | Reusable JavaScript utilities |
| `errorPanel` | Displays friendly and detailed errors | Conditional templates and template switching |
| `placeholder` | Empty-state message with logo | Static resources |
| `paginator` | Next/previous page controls | Parent-child events |
| `productTile` | Product thumbnail card | `@api`, custom events, drag-and-drop |
| `productListItem` | Compact product row | NavigationMixin |
| `orderItemTile` | Editable order item card | Child-to-parent custom events |

### Product Explorer Components

| Component | Purpose | Depends On |
|---|---|---|
| `productFilter` | Filters by text, price, category, material, and level | Product fields, `ProductsFiltered__c` message channel |
| `productTileList` | Retrieves paged products and displays tiles | `ProductController`, both message channels |
| `productCard` | Shows selected product details | `ProductSelected__c`, LDS |
| `similarProducts` | Shows products in the same family | `ProductController`, Product record page |

### Order Components

| Component | Purpose | Depends On |
|---|---|---|
| `orderBuilder` | Builds an order using drag/drop and CRUD operations | `OrderController`, LDS, `orderItemTile` |
| `orderStatusPath` | Displays and edits order status; listens for manufacturing events | `Order__c`, `Manufacturing_Event__e`, EMP API |

### Experience Cloud Components

| Component | Purpose | Depends On |
|---|---|---|
| `hero` | Configurable image/video home-page banner | Static resources, `heroDetails` |
| `heroDetails` | Button overlay that resolves and links to product/family records | `ProductRecordInfoController` |
| `createCase` | Public-facing support Case form | Case fields and Experience site |

Files: [lwc](<../force-app/main/default/lwc>)

---

## 4.4 Pages, Tabs, App, And Supporting Metadata

| Metadata | Purpose |
|---|---|
| `Product_Explorer.flexipage-meta.xml` | App page containing filter, product list, and product card |
| `Product_Record_Page.flexipage-meta.xml` | Product record page with similar products |
| `Order_Record_Page.flexipage-meta.xml` | Order page with order builder, draggable products, and status path |
| `Account_Record_Page.flexipage-meta.xml` | Account page with map component |
| `EBikes.app-meta.xml` | Lightning app navigation and branding |
| `Product_Explorer.tab-meta.xml` | Navigation tab for the custom app page |
| Object tabs | Tabs for Products, Product Families, Orders |
| Layout files | Display/edit arrangements for products, families, orders, order items, and cases |
| List views | Product list, families, orders, and cases |
| `ProductsFiltered.messageChannel-meta.xml` | Filter message contract |
| `ProductSelected.messageChannel-meta.xml` | Product-selection message contract |
| `bike_assets` static resource | Logo and hero imagery |
| CSP trusted site | Allows externally hosted bicycle images |
| Prompts | In-app guidance walkthroughs |
| `ChangeEvents_Order_ChangeEvent` | Change Data Capture channel configuration |

---

## 4.5 Experience Cloud Metadata

| Metadata | Purpose |
|---|---|
| `sites/E_Bikes.site-meta.xml` | Salesforce Site configuration |
| `networks/E-Bikes.network-meta.xml` | Experience Cloud network |
| `experiences/E_Bikes1` | Page routes, views, theme, branding, and configuration |
| `navigationMenus` | Public site navigation |
| `networkBranding` | Community colors |
| `audience/Default_E-Bikes` | Default audience |
| `pages/CommunitiesLanding.page` | Visualforce community entry page |
| `contentassets` | Experience site logos |
| `guest-profile-metadata` | Public profile permissions and guest sharing rules |

Public site pages include:

| Page | Purpose |
|---|---|
| Home | Two configurable hero panels |
| Product Explorer | Filters, product tiles, selected product card |
| Product Detail | Standard product record detail experience |
| Product Families | Product family listing |
| Create Case | Custom LWC Case form |
| Cases | Case list navigation item |
| Login/Register/Error pages | Standard Experience Cloud pages |

---

# 5. Recommended Build Order

Build in this order because each layer depends on the earlier one:

| Phase | Build |
|---|---|
| 1 | Org setup and Experience Cloud prerequisites |
| 2 | Local Salesforce project and CLI authorization |
| 3 | Product family and product data model |
| 4 | Order, order item, Case, and platform event model |
| 5 | Security design for internal and guest access |
| 6 | Apex backend and Apex tests |
| 7 | LWC foundation components |
| 8 | Product Explorer LWC feature |
| 9 | Product record and order-building features |
| 10 | Internal Lightning pages, tabs, app, and layouts |
| 11 | Experience Cloud pages and guest access |
| 12 | Sample data |
| 13 | Testing and troubleshooting |

A practical note: while this is the ideal **learning order**, a final metadata deployment from the unchanged repository should deploy the entire `force-app` directory together because the object metadata activates Lightning pages that themselves reference LWCs and Apex.

---

# Phase 1: Org Setup

## Step 1. Create And Authorize A Fresh Developer Org

### What To Build

Obtain a new Salesforce Developer Org and connect it to Salesforce CLI.

### Why First

All metadata deployment, permission assignment, data loading, and testing require an authenticated target org.

### Concept Learned

- Salesforce org types
- CLI authentication
- Org aliases
- Default target orgs

### Commands

```powershell
sf org login web --set-default --alias mydevorg
sf org display --target-org mydevorg
```

### Verify

```powershell
sf org open --target-org mydevorg
```

Confirm that Setup opens in the correct fresh org.

---

## Step 2. Enable Digital Experiences

### What To Build

Enable Experience Cloud features required by the E-Bikes public site.

### Why Now

Site, Network, and Experience Bundle metadata cannot deploy correctly until Experiences are enabled.

### Manual Setup

In Setup:

1. Search for **Digital Experiences Settings**.
2. Enable **Digital Experiences**.
3. Save.
4. Open **Experience Management Settings**.
5. Enable **ExperienceBundle Metadata API**.
6. Save.
7. Confirm that your org has a My Domain configured and deployed.

### Files That Depend On This

- [sites/E_Bikes.site-meta.xml](<../force-app/main/default/sites/E_Bikes.site-meta.xml>)
- [networks/E-Bikes.network-meta.xml](<../force-app/main/default/networks/E-Bikes.network-meta.xml>)
- [experiences/E_Bikes1](<../force-app/main/default/experiences/E_Bikes1>)

### Verify

In Setup, confirm that **All Sites** or **Digital Experiences** administration pages are available.

---

## Step 3. Prepare Developer-Org-Specific Site Values

### What To Build

When reproducing the repository in your own working copy, update the site metadata values for your org.

### Why Now

The checked-in site file contains a username and subdomain from the org where the sample was originally exported. Those values are not valid in your new org.

### File

- [E_Bikes.site-meta.xml](<../force-app/main/default/sites/E_Bikes.site-meta.xml>)

### Values To Replace In Your Build Copy

| Element | Replace With |
|---|---|
| `<siteAdmin>` | Your Developer Org username |
| `<siteGuestRecordDefaultOwner>` | Your Developer Org username |
| `<subdomain>` | Your org’s Digital Experiences/My Domain subdomain |

### Verify

Before deployment, confirm the values match the user and domain in your org.

---

## Step 4. Check For The Case Product Field Conflict

### What To Build

Confirm whether the fresh org already contains a Case field labeled Product that conflicts with the repository’s `Case.Product__c` metadata.

### Why Now

The repository adds its own lookup field:

```text
Case.Product__c -> Product__c
```

Some org configurations already expose a Product field on Case, which can cause deployment conflicts.

### Relevant File

- [Case/Product__c.field-meta.xml](<../force-app/main/default/objects/Case/fields/Product__c.field-meta.xml>)

### Verify In Setup

1. Open **Object Manager > Case > Fields & Relationships**.
2. Search for Product.
3. If the sample deployment reports a duplicate/conflict involving Product, remove the conflicting custom field only if this is truly a fresh learning org and no data depends on it.

---

# Phase 2: Repository Setup

## Step 5. Understand The Salesforce Project Container

### What To Build

Open the project in VS Code and study its Salesforce configuration.

### Relevant Files

| File | Purpose |
|---|---|
| [sfdx-project.json](<../sfdx-project.json>) | Defines `force-app` as the package directory and API `65.0` |
| [config/project-scratch-def.json](<../config/project-scratch-def.json>) | Shows features used by the app |
| [package.json](<../package.json>) | JavaScript tooling and tests |

### Concept Learned

Salesforce DX source format stores metadata as source files rather than as changes made only through Setup.

### Optional Tool Setup

The Salesforce application itself can deploy without Node tooling, but local LWC tests need dependencies:

```powershell
npm install
```

### Verify

```powershell
sf project list
npm run test:unit -- --passWithNoTests
```

The second command tests local JavaScript configuration, not the Salesforce org.

---

# Phase 3: Data Model

For learning, create the model manually or as small metadata deployments. For a faithful final repository install, deploy all of `force-app` together later because the exported object definitions reference activated Lightning pages.

## Step 6. Build Product Families

### What To Build

Create `Product_Family__c`.

| Property | Value |
|---|---|
| Label | Product Family |
| Plural Label | Product Families |
| Name Field | Text: Product Family Name |

Add fields:

| Field | Type | Values |
|---|---|---|
| `Description__c` | Text Area | Free text |
| `Category__c` | Picklist | Commuter, Hybrid, Mountain |

### Why First

Products reference product families. Parent records and parent objects should exist before their children.

### Files

- [Product_Family__c object folder](<../force-app/main/default/objects/Product_Family__c>)

### Salesforce Concept

Custom objects, custom fields, picklists, and parent-first modeling.

### CLI Deployment

This object has no final record-page override dependency, so it can be deployed early:

```powershell
sf project deploy start --source-dir force-app/main/default/objects/Product_Family__c --target-org mydevorg --wait 10
```

### Verify

```powershell
sf data query --query "SELECT Id, Name, Category__c FROM Product_Family__c" --target-org mydevorg
```

At this point no records are expected, but the query should run without an object error.

---

## Step 7. Build Products

### What To Build

Create `Product__c`.

| Property | Value |
|---|---|
| Label | Product |
| Plural Label | Products |
| Name Field | Text: Product Name |

Add fields:

| Group | Fields |
|---|---|
| Catalog | `Description__c`, `MSRP__c`, `Picture_URL__c` |
| Classification | `Category__c`, `Level__c`, `Material__c` |
| Product family | `Product_Family__c` lookup |
| Electric components | `Battery__c`, `Charger__c`, `Motor__c` |
| Mechanical components | `Fork__c`, `Front_Brakes__c`, `Rear_Brakes__c` |
| Configurable colors | `Frame_Color__c`, `Handlebar_Color__c`, `Seat_Color__c`, `Waterbottle_Color__c` |

### Why Now

Products are the main catalog records and are required by Product Explorer, similar-products functionality, Cases, and order items.

### Files

- [Product__c object folder](<../force-app/main/default/objects/Product__c>)

### Salesforce Concept

Lookups, product-style schemas, field types, and picklists used by UI filters.

### CLI Deployment Guidance

The exported `Product__c.object-meta.xml` activates `Product_Record_Page`, which is built later. During a manual learning rebuild, create the base object in Object Manager, then deploy its fields:

```powershell
sf project deploy start --source-dir force-app/main/default/objects/Product__c/fields --target-org mydevorg --wait 10
```

For the unchanged repository’s faithful installation, include the entire `force-app` package in the final integrated deployment.

### Verify

In Object Manager, confirm the Product object fields exist. Then query:

```powershell
sf data query --query "SELECT Id, Name, MSRP__c, Product_Family__c FROM Product__c" --target-org mydevorg
```

---

## Step 8. Build Reseller Orders

### What To Build

Create `Order__c`.

| Property | Value |
|---|---|
| Label | Reseller Order |
| Plural Label | Reseller Orders |
| Name Field | Auto Number: `O-{00000}` |

Add fields:

| Field | Type | Details |
|---|---|---|
| `Account__c` | Required Lookup to Account | Order owner/reseller |
| `Status__c` | Required Picklist | Draft through In Production |

### Why Now

The order parent must exist before creating order items.

### Files

- [Order__c object folder](<../force-app/main/default/objects/Order__c>)

### Salesforce Concept

Standard-object relationships, autonumber keys, and status lifecycle modeling.

### CLI Deployment Guidance

Because the exported object activates `Order_Record_Page`, create the base object manually during a staged learning build and deploy the fields:

```powershell
sf project deploy start --source-dir force-app/main/default/objects/Order__c/fields --target-org mydevorg --wait 10
```

### Verify

Create an Account in the UI, then create a Reseller Order related to it. Confirm that `Status__c` defaults to `Draft`.

---

## Step 9. Build Order Items

### What To Build

Create `Order_Item__c`.

| Property | Value |
|---|---|
| Label | Order Item |
| Name Field | Auto Number: `OI-{000000}` |

Add fields:

| Field | Type |
|---|---|
| `Order__c` | Master-Detail to `Order__c` |
| `Product__c` | Lookup to `Product__c` |
| `Price__c` | Currency |
| `Qty_S__c` | Number, default 1 |
| `Qty_M__c` | Number, default 1 |
| `Qty_L__c` | Number, default 1 |

### Why Now

The order builder manipulates line items, not the parent order directly.

### Files

- [Order_Item__c object folder](<../force-app/main/default/objects/Order_Item__c>)

### Salesforce Concept

Master-detail relationships, child ownership, roll-up-style thinking, and line-item design.

### CLI Deployment

After manually creating the base object:

```powershell
sf project deploy start --source-dir force-app/main/default/objects/Order_Item__c/fields --target-org mydevorg --wait 10
```

### Verify

Create an order item tied to an order and a product. Confirm it appears as a related record on the order.

---

## Step 10. Extend Case For Product Support

### What To Build

Add two fields to standard `Case`:

| Field | Type | Purpose |
|---|---|---|
| `Product__c` | Lookup to `Product__c` | Product being discussed |
| `Case_Category__c` | Picklist | Support issue classification |

### Why Now

The public Experience Cloud Case form depends on these fields.

### Files

- [Case object extensions](<../force-app/main/default/objects/Case>)

### Salesforce Concept

Extending standard Salesforce objects.

### CLI Deployment

```powershell
sf project deploy start --source-dir force-app/main/default/objects/Case/fields --target-org mydevorg --wait 10
```

### Verify

Open a Case record and confirm both fields can be added to a layout and populated.

---

## Step 11. Build The Manufacturing Event

### What To Build

Create high-volume platform event `Manufacturing_Event__e`.

| Field | Type | Required |
|---|---|---|
| `Order_Id__c` | Text(18) | Yes |
| `Status__c` | Text | Yes |

### Why Now

`orderStatusPath` listens for these events and updates order status.

### Files

- [Manufacturing_Event__e](<../force-app/main/default/objects/Manufacturing_Event__e>)
- [ChangeEvents_Order_ChangeEvent.platformEventChannelMember-meta.xml](<../force-app/main/default/platformEventChannelMembers/ChangeEvents_Order_ChangeEvent.platformEventChannelMember-meta.xml>)

### Salesforce Concept

Platform Events, real-time UI updates, EMP API, and optionally Change Data Capture.

### CLI Deployment

```powershell
sf project deploy start --source-dir force-app/main/default/objects/Manufacturing_Event__e --target-org mydevorg --wait 10
```

### Verify

In Setup, open **Platform Events** and confirm `Manufacturing Event` exists.

Note: the repository contains the subscriber UI, but no Apex publisher for manufacturing events. The optional companion manufacturing application described in the README is intended to produce that fuller demonstration.

---

# Phase 4: Security And Permissions

Security is designed now, but some original metadata is best deployed after tabs, apps, and the Experience site exist.

## Step 12. Understand Internal User Permissions

### What To Build

The `ebikes` permission set grants internal users:

| Access Type | Access |
|---|---|
| App visibility | E-Bikes application |
| Tabs | Products, Product Families, Orders, Product Explorer |
| Product and family objects | Create, read, edit, delete, view all, modify all |
| Order and order-item objects | Create, read, edit, delete, view all, modify all |
| Accounts | Read/view all |
| Cases | Create, read, edit, delete/view all |
| Product and order-item fields | Read/edit access |

### File

- [ebikes.permissionset-meta.xml](<../force-app/main/default/permissionsets/ebikes.permissionset-meta.xml>)

### Why It Is Deployed Later

The permission set references tabs and the `EBikes` app, which should already exist.

### Salesforce Concept

Object permissions, field-level security, application visibility, and tab visibility.

### Deployment And Assignment

After the app and tabs are deployed:

```powershell
sf project deploy start --source-dir force-app/main/default/permissionsets/ebikes.permissionset-meta.xml --target-org mydevorg --wait 10
sf org assign permset --name ebikes --target-org mydevorg
```

### Verify

Log in as the assigned user or use the current user, open App Launcher, and confirm the E-Bikes app and tabs are visible.

---

## Step 13. Understand Guest Access

### What To Build

The public site guest security is deployed separately after the site exists.

| Guest Access | Configuration |
|---|---|
| Product read access | Enabled through profile and guest sharing rule |
| Product family read access | Enabled through profile and guest sharing rule |
| Product Apex queries | `ProductController`, `ProductRecordInfoController`, and `PagedResult` allowed |
| Site entry page | `CommunitiesLanding` Visualforce page allowed |
| Case create | Profile grants Case create permission |
| Orders | Hidden from guest navigation and access |

### Files

- [guest-profile-metadata](<../guest-profile-metadata>)

### Important Learning Checkpoint

The repository’s guest profile marks `Case.Product__c` and `Case.Case_Category__c` unreadable, although `createCase` references those fields. If public guest Case submission does not display or accept those fields, inspect guest field-level security. This is an instructive example of UI metadata being limited by security metadata.

### Why Deployed After Publishing

The guest user/profile is generated in connection with the Experience Cloud site. Deploying guest-profile metadata before the site exists can fail because its named guest profile and sharing target do not yet exist.

### Deployment

Run after publishing the site:

```powershell
sf project deploy start --metadata-dir guest-profile-metadata --target-org mydevorg --wait 10
```

### Verify

Open the public site in a private browser window. Confirm that products can be browsed without login, then test the Create Case page.

---

# Phase 5: Apex Backend

## Step 14. Build Product Query Services

### What To Build

Implement:

| Class | Role |
|---|---|
| `PagedResult` | Returns page number, page size, total record count, and results |
| `ProductController` | Retrieves catalog results and similar products |

### Why Now

The Product Explorer LWCs need a server-side query layer after objects and fields exist.

### Core Behavior

`ProductController.getProducts`:

- Accepts text, maximum price, category, level, and material filters.
- Uses a page size of `9`.
- Returns product tiles ordered by product name.

`ProductController.getSimilarProducts`:

- Finds other products in the same family.
- Excludes the current product.

### Files

- [PagedResult.cls](<../force-app/main/default/classes/PagedResult.cls>)
- [ProductController.cls](<../force-app/main/default/classes/ProductController.cls>)
- [TestProductController.cls](<../force-app/main/default/classes/TestProductController.cls>)

### Salesforce Concept

Apex controllers, wire-compatible methods, cacheable reads, dynamic SOQL, paging, sharing, and user-mode security.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/classes --target-org mydevorg --wait 10
```

### Test

```powershell
sf apex run test --class-names TestProductController --result-format human --code-coverage --target-org mydevorg --wait 10
```

### Verify

Confirm both test methods pass:

- Product filtering returns the expected product.
- Similar products returns another item from the family.

---

## Step 15. Build Order Query Services

### What To Build

Implement `OrderController.getOrderItems`.

### Why Now

The order builder needs line items and related product image/price information.

### File

- [OrderController.cls](<../force-app/main/default/classes/OrderController.cls>)
- [TestOrderController.cls](<../force-app/main/default/classes/TestOrderController.cls>)

### Salesforce Concept

Relationship queries in SOQL and Apex controllers consumed by LWCs.

### Test

```powershell
sf apex run test --class-names TestOrderController --result-format human --code-coverage --target-org mydevorg --wait 10
```

### Verify

The test creates an Account, order, product, and order item, then confirms one item is returned.

---

## Step 16. Build Experience Cloud Apex Helpers

### What To Build

Implement:

| Class | Role |
|---|---|
| `ProductRecordInfoController` | Converts a hero button’s product/family name to a target record |
| `HeroDetailsPositionCustomPicklist` | Supplies left/right Experience Builder design values |
| `CommunitiesLandingController` | Handles site landing redirection |

### Why Now

Experience Cloud pages are built later, but their Apex dependencies must exist before deployment.

### Files

- [ProductRecordInfoController.cls](<../force-app/main/default/classes/ProductRecordInfoController.cls>)
- [HeroDetailsPositionCustomPicklist.cls](<../force-app/main/default/classes/HeroDetailsPositionCustomPicklist.cls>)
- [CommunitiesLandingController.cls](<../force-app/main/default/classes/CommunitiesLandingController.cls>)

### Test

```powershell
sf apex run test --class-names CommunitiesLandingControllerTest --result-format human --target-org mydevorg --wait 10
```

---

# Phase 6: LWC Frontend

## Step 17. Build Shared UI Components First

### What To Build

Create these reusable components before feature containers:

| Component | Why First |
|---|---|
| `ldsUtils` | Shared error formatting helper |
| `errorPanel` | Used when Apex or LDS returns errors |
| `placeholder` | Used when lists are empty |
| `paginator` | Used by product lists |
| `productTile` | Reusable product presentation |
| `productListItem` | Reusable compact product presentation |
| `orderItemTile` | Child UI used by order builder |

### Files

- [lwc shared bundles](<../force-app/main/default/lwc>)

### Salesforce Concept

Component composition, public properties, custom events, slots, static resources, and utility modules.

### Deploy

Before deploying `placeholder`, deploy the static resource it imports:

```powershell
sf project deploy start --source-dir force-app/main/default/staticresources --target-org mydevorg --wait 10
sf project deploy start --source-dir force-app/main/default/lwc/ldsUtils --source-dir force-app/main/default/lwc/errorPanel --source-dir force-app/main/default/lwc/placeholder --source-dir force-app/main/default/lwc/paginator --source-dir force-app/main/default/lwc/productTile --source-dir force-app/main/default/lwc/productListItem --source-dir force-app/main/default/lwc/orderItemTile --target-org mydevorg --wait 10
```

### Local Test

```powershell
npm run test:unit -- --runTestsByPath force-app/main/default/lwc/ldsUtils/__tests__/ldsUtils.test.js force-app/main/default/lwc/paginator/__tests__/paginator.test.js force-app/main/default/lwc/placeholder/__tests__/placeholder.test.js
```

---

## Step 18. Build Lightning Message Channels

### What To Build

Create two message channels:

| Channel | Payload | Publisher | Subscriber |
|---|---|---|---|
| `ProductsFiltered__c` | `filters` | `productFilter` | `productTileList` |
| `ProductSelected__c` | `productId` | `productTileList` | `productCard` |

### Why Now

The Product Explorer components communicate without requiring direct parent-child nesting.

### Files

- [messageChannels](<../force-app/main/default/messageChannels>)

### Salesforce Concept

Lightning Message Service enables loosely coupled communication across components on a page.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/messageChannels --target-org mydevorg --wait 10
```

---

## Step 19. Build Product Explorer

### What To Build

Create the feature in this order:

| Order | Component | Purpose |
|---|---|---|
| 1 | `productFilter` | Publishes filters from search and picklists |
| 2 | `productTileList` | Calls Apex and renders matching products |
| 3 | `productCard` | Reacts to product selections and displays details |

### Why This Order

This mirrors the data flow:

```text
User changes filter
  -> productFilter publishes message
  -> productTileList retrieves records
  -> user selects tile
  -> productTileList publishes selection
  -> productCard shows details
```

### Files

- [productFilter](<../force-app/main/default/lwc/productFilter>)
- [productTileList](<../force-app/main/default/lwc/productTileList>)
- [productCard](<../force-app/main/default/lwc/productCard>)

### Salesforce Concept

- `@wire` to Apex
- Lightning Message Service
- UI Record API
- Reactive parameters
- Debounced filtering
- Component orchestration

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/lwc/productFilter --source-dir force-app/main/default/lwc/productTileList --source-dir force-app/main/default/lwc/productCard --target-org mydevorg --wait 10
```

### Local Test

```powershell
npm run test:unit -- --runTestsByPath force-app/main/default/lwc/productFilter/__tests__/productFilter.test.js force-app/main/default/lwc/productTileList/__tests__/productTileList.test.js
```

### Verify In Org

After the Product Explorer page is built:

1. Open Product Explorer.
2. Confirm products appear nine per page.
3. Search for `fuse`.
4. Confirm four Fuse products remain.
5. Select a tile.
6. Confirm the product card updates.

---

## Step 20. Build Product Record Features

### What To Build

Create `similarProducts`.

### Why Now

This feature requires Products, Product Families, and `ProductController.getSimilarProducts`.

### File

- [similarProducts](<../force-app/main/default/lwc/similarProducts>)

### Salesforce Concept

Record-page context with `@api recordId`, wired LDS data, wired Apex data, and navigation.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/lwc/similarProducts --target-org mydevorg --wait 10
```

### Local Test

```powershell
npm run test:unit -- --runTestsByPath force-app/main/default/lwc/similarProducts/__tests__/similarProducts.test.js
```

### Verify In Org

Open a product in a family containing multiple products. Confirm similar family products appear in the side panel.

---

## Step 21. Build The Order Builder

### What To Build

Create `orderBuilder`.

### How It Works

1. `productTileList` displays draggable products.
2. A user drags a product onto `orderBuilder`.
3. `orderBuilder` creates `Order_Item__c` using Lightning Data Service.
4. A 40% reseller discount is applied by multiplying MSRP by `0.6`.
5. `orderItemTile` allows editing quantities and price.
6. Totals recalculate in the browser.

### Files

- [orderBuilder](<../force-app/main/default/lwc/orderBuilder>)
- [orderItemTile](<../force-app/main/default/lwc/orderItemTile>)

### Salesforce Concept

Drag-and-drop, Lightning Data Service CRUD, optimistic UI updates, custom events, and Apex refresh.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/lwc/orderBuilder --target-org mydevorg --wait 10
```

### Local Test

```powershell
npm run test:unit -- --runTestsByPath force-app/main/default/lwc/orderBuilder/__tests__/orderBuilder.test.js
```

### Verify In Org

1. Create a Reseller Order related to an Account.
2. Open the order record.
3. Drag a product into the order area.
4. Confirm an Order Item is created.
5. Change quantity or price.
6. Confirm totals update.

---

## Step 22. Build The Order Status Path

### What To Build

Create `orderStatusPath`.

### Why After Orders And Platform Events

It needs:

- `Order__c.Status__c`
- Current order record context
- EMP API subscription to `/event/Manufacturing_Event__e`

### File

- [orderStatusPath](<../force-app/main/default/lwc/orderStatusPath>)

### Salesforce Concept

Custom status path UIs, record updates through UI API, and event-driven user interfaces.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/lwc/orderStatusPath --target-org mydevorg --wait 10
```

### Local Test

```powershell
npm run test:unit -- --runTestsByPath force-app/main/default/lwc/orderStatusPath/__tests__/orderStatusPath.test.js
```

### Verify In Org

1. Open an order record.
2. Click another path value.
3. Confirm `Status__c` changes.
4. For the event scenario, publish a matching `Manufacturing_Event__e` record and confirm the displayed status updates.

---

## Step 23. Build Account And Community Components

### What To Build

| Component | Purpose |
|---|---|
| `accountMap` | Displays reseller Account billing address on a map |
| `hero` | Experience Cloud home-page banner |
| `heroDetails` | Hero button navigation |
| `createCase` | Customer support Case submission |

### Files

- [accountMap](<../force-app/main/default/lwc/accountMap>)
- [hero](<../force-app/main/default/lwc/hero>)
- [heroDetails](<../force-app/main/default/lwc/heroDetails>)
- [createCase](<../force-app/main/default/lwc/createCase>)

### Salesforce Concept

Record-page components, Experience Builder design properties, dynamic design picklists, standard record-edit forms, and toast events.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/lwc/accountMap --source-dir force-app/main/default/lwc/hero --source-dir force-app/main/default/lwc/heroDetails --source-dir force-app/main/default/lwc/createCase --target-org mydevorg --wait 10
```

### Verify

- On an Account record with a billing address, confirm a map appears after its page is activated.
- On the Experience site, confirm hero banners load and their buttons navigate.
- Confirm the Case page renders according to guest/user security settings.

---

# Phase 7: App And Page Setup

## Step 24. Build Layouts And List Views

### What To Build

| Metadata | Purpose |
|---|---|
| Product Layout | Product specifications |
| Product Family Layout | Family information and related products |
| Order Layout | Account, status, and related items |
| Order Item Layout | Product, quantities, and price |
| Case Layouts | Product support fields |
| ProductList | Product browsing columns |
| All Product Families | Family list |
| All Orders | Order list |
| My Cases | User-owned Case list |

### Files

- [layouts](<../force-app/main/default/layouts>)
- Object `listViews` directories under [objects](<../force-app/main/default/objects>)

### Salesforce Concept

Page layouts control record editing/detail arrangement; list views control tabular record browsing.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/layouts --source-dir force-app/main/default/objects/Product__c/listViews --source-dir force-app/main/default/objects/Product_Family__c/listViews --source-dir force-app/main/default/objects/Order__c/listViews --source-dir force-app/main/default/objects/Case/listViews --target-org mydevorg --wait 10
```

### Verify

Open each object tab or Object Manager layout assignment and confirm the expected fields are present.

---

## Step 25. Build Lightning App Pages And Record Pages

### What To Build

| Page | Components |
|---|---|
| Product Explorer app page | `productFilter`, `productTileList`, `productCard` |
| Product record page | Standard record content and `similarProducts` |
| Order record page | `orderBuilder`, `orderStatusPath`, draggable `productTileList` |
| Account record page | Standard account details and `accountMap` |

### Files

- [flexipages](<../force-app/main/default/flexipages>)
- [Aura three-region template](<../force-app/main/default/aura/pageTemplate_2_7_3>)

### Why Now

Pages require all referenced LWCs and fields to exist first.

### Salesforce Concept

Lightning App Builder composition and page activation.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/aura --source-dir force-app/main/default/flexipages --target-org mydevorg --wait 20
```

### Verify

In Lightning App Builder:

1. Open Product Explorer and confirm three regions.
2. Open Product Record Page and confirm Similar Products is present.
3. Open Order Record Page and confirm Order Builder and Status Path are present.
4. Open Account Record Page and confirm Account Map is present.

---

## Step 26. Build Tabs And The E-Bikes Lightning App

### What To Build

Create tabs:

| Tab | Purpose |
|---|---|
| `Product_Explorer` | Opens the custom Product Explorer app page |
| `Product__c` | Product object tab |
| `Product_Family__c` | Product Family tab |
| `Order__c` | Reseller Order tab |

Create the `EBikes` Lightning app containing:

```text
Home
Product Explorer
Accounts
Contacts
Products
Reseller Orders
Product Families
```

### Files

- [tabs](<../force-app/main/default/tabs>)
- [EBikes.app-meta.xml](<../force-app/main/default/applications/EBikes.app-meta.xml>)
- [contentassets/logo.asset-meta.xml](<../force-app/main/default/contentassets/logo.asset-meta.xml>)

### Salesforce Concept

Lightning app navigation and branded application shells.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/contentassets --source-dir force-app/main/default/tabs --source-dir force-app/main/default/applications --target-org mydevorg --wait 20
sf project deploy start --source-dir force-app/main/default/permissionsets/ebikes.permissionset-meta.xml --target-org mydevorg --wait 10
sf org assign permset --name ebikes --target-org mydevorg
```

### Verify

Open App Launcher and select **E-Bikes**. Confirm all navigation items appear.

---

## Step 27. Add Optional Walkthrough Prompts

### What To Build

The repository provides in-app guidance for:

| Prompt | Page |
|---|---|
| Product Explorer | Product Explorer app page |
| Product Detail | Product record page |
| Order Builder | Order record page |

### Files

- [prompts](<../force-app/main/default/prompts>)

### Salesforce Concept

In-App Guidance teaches users directly in the application.

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/prompts --target-org mydevorg --wait 10
```

### Verify

Open the associated app pages. Published walkthrough steps should become available according to Salesforce in-app guidance behavior.

---

# Phase 8: Experience Cloud Setup

## Step 28. Deploy Experience Site Dependencies

### What To Build

Deploy:

| Metadata | Purpose |
|---|---|
| Visualforce landing page | Community entry routing |
| Network and Site | Site identity and URL |
| Experience Bundle | Page routes and page contents |
| Navigation menus | Public navigation |
| Branding and content assets | Public visual appearance |
| CSP trusted site | Permits external bicycle images |

### Files

- [pages](<../force-app/main/default/pages>)
- [sites](<../force-app/main/default/sites>)
- [networks](<../force-app/main/default/networks>)
- [experiences](<../force-app/main/default/experiences>)
- [navigationMenus](<../force-app/main/default/navigationMenus>)
- [networkBranding](<../force-app/main/default/networkBranding>)
- [cspTrustedSites](<../force-app/main/default/cspTrustedSites>)

### Salesforce Concept

Experience Cloud stores a public application as routes, views, branding, navigation, network configuration, and security.

### Recommended Faithful Deployment

At this point, with org prerequisites completed and site values prepared in your learning copy, deploy the complete application package:

```powershell
sf project deploy start --source-dir force-app --target-org mydevorg --wait 30
```

This resolves exported dependencies together, including:

- Object action overrides pointing to FlexiPages
- FlexiPages pointing to LWCs
- Apps pointing to tabs and record pages
- Experience pages pointing to community LWCs and Apex

### Verify

In Setup, open **All Sites** and confirm an E-Bikes site exists.

---

## Step 29. Publish The Experience Site

### What To Build

Make the E-Bikes site accessible.

### Why Now

Publishing occurs only after its metadata has deployed successfully.

### Command

```powershell
sf community publish --name E-Bikes --target-org mydevorg
```

### Verify

From Setup > All Sites, open the site URL. Confirm the home page loads.

---

## Step 30. Deploy Guest Profile And Sharing Rules

### What To Build

Grant public users access to view product catalog data.

### Command

```powershell
sf project deploy start --metadata-dir guest-profile-metadata --target-org mydevorg --wait 10
```

### Verify

In a private/incognito browser:

1. Open the E-Bikes public site URL.
2. Open Product Explorer.
3. Confirm products display.
4. Open a product detail page.
5. Check whether the Create Case fields permitted by guest security appear.

---

# Phase 9: Sample Data

## Step 31. Load Sample Catalog Data

### What To Build

The sample data includes:

| Object | Record Count | Purpose |
|---|---:|---|
| Account | 3 | Resellers used with orders and account maps |
| Product Family | 4 | Dynamo, Fuse, Electra, Volt |
| Product | 16 | Bicycle models displayed in Product Explorer |

### Files

- [Accounts.json](<../data/Accounts.json>)
- [Product_Family__cs.json](<../data/Product_Family__cs.json>)
- [Product__cs.json](<../data/Product__cs.json>)
- [sample-data-plan.json](<../data/sample-data-plan.json>)

### Why This Order

Products contain references to product families, so families must be inserted first. Accounts are independent but are needed when you create orders.

### Salesforce Concept

Data import plans and reference IDs for relational test/demo data.

### Command

```powershell
sf data import tree --plan data/sample-data-plan.json --target-org mydevorg
```

### Verify

```powershell
sf data query --query "SELECT COUNT() FROM Product_Family__c" --target-org mydevorg
sf data query --query "SELECT COUNT() FROM Product__c" --target-org mydevorg
sf data query --query "SELECT COUNT() FROM Account WHERE Name IN ('Wheelworks','Northern Trail Cycling','Trailblazers')" --target-org mydevorg
```

Expected results:

| Query | Expected Count |
|---|---:|
| Product Families | 4 |
| Products | 16 |
| Included Accounts | 3 |

---

## Step 32. Create An Order For Simulation

The provided data plan does not insert `Order__c` or `Order_Item__c` records. Create those interactively to learn the order experience.

### Steps

1. Open the E-Bikes app.
2. Open **Reseller Orders**.
3. Create an order for `Wheelworks`.
4. Leave status as `Draft`.
5. Open the order record.
6. Drag products into its Order Builder.
7. Adjust quantities and prices.

### Concepts Learned

- Record creation
- Master-detail child records
- LWC-driven CRUD
- Reactive UI totals

---

# Phase 10: Testing

## Step 33. Run Apex Tests

### Tests Included

| Test Class | Verifies |
|---|---|
| `TestProductController` | Filtering and similar-products queries |
| `TestOrderController` | Order item retrieval |
| `CommunitiesLandingControllerTest` | Community landing controller |

### Command

```powershell
sf apex run test --test-level RunLocalTests --result-format human --code-coverage --target-org mydevorg --wait 20
```

### Verify

Confirm all local Apex tests pass and code coverage results are reported.

---

## Step 34. Run LWC Unit Tests Locally

### What They Cover

The repository includes Jest tests for components including:

- Product filters
- Product list/paging
- Similar products
- Hero and hero details
- Order builder
- Order status path
- Account map
- Error display
- Accessibility assertions

### Command

```powershell
npm run test:unit
```

Optional coverage:

```powershell
npm run test:unit:coverage
```

### Concept Learned

LWC JavaScript tests run locally with mocked LDS, Apex, Message Service, and EMP API adapters. They are separate from org-side Apex tests.

---

## Step 35. Run UI Tests With UTAM

### What It Tests

The UTAM test opens Product Explorer and verifies:

1. Sixteen total products are available.
2. The product card starts empty.
3. Filtering for `fuse` leaves four products.
4. Selecting `FUSE X1` populates the card.
5. Opening the record navigates to a Product record page.

### Files

- [page-explorer.spec.js](<../force-app/test/utam/page-explorer.spec.js>)
- LWC `__utam__` page object definitions
- [wdio.conf.js](<../wdio.conf.js>)

### Commands

```powershell
npm run test:ui:compile
npm run test:ui:generate:login
npm run test:ui
```

### Verify

A browser session opens and the Product Explorer interaction test passes.

---

# 11. Full Installation Sequence For A Fresh Developer Org

Once you understand the phases, this is the concise deployment path for a fresh Developer Org.

## Before Commands

Complete these manual prerequisites:

1. Enable Digital Experiences.
2. Enable ExperienceBundle Metadata API.
3. Configure My Domain.
4. In your working rebuild copy, update `sites/E_Bikes.site-meta.xml` with your username and subdomain.
5. Resolve any Case Product field conflict in the fresh org.

## Commands

```powershell
sf org login web --set-default --alias mydevorg

sf project deploy start --source-dir force-app --target-org mydevorg --wait 30

sf org assign permset --name ebikes --target-org mydevorg

sf data import tree --plan data/sample-data-plan.json --target-org mydevorg

sf community publish --name E-Bikes --target-org mydevorg

sf project deploy start --metadata-dir guest-profile-metadata --target-org mydevorg --wait 10

sf apex run test --test-level RunLocalTests --result-format human --code-coverage --target-org mydevorg --wait 20

sf org open --target-org mydevorg
```

Then:

1. In Setup, activate the desired Lightning theme if required.
2. Open App Launcher.
3. Select **E-Bikes**.
4. Create a Reseller Order manually to exercise the order builder.

---

# 12. Common Mistakes And Troubleshooting

| Problem | Likely Cause | Resolution |
|---|---|---|
| Site or Network metadata deployment fails | Digital Experiences not enabled | Enable Digital Experiences and ExperienceBundle Metadata API first |
| Site deployment references another user | Checked-in `siteAdmin` and owner belong to original org | Replace them with your Developer Org username in your rebuild copy |
| Site deployment fails on domain/subdomain | Checked-in site subdomain is not yours | Replace `<subdomain>` with your org’s domain value |
| Product object staged deployment complains about missing FlexiPage | Exported object metadata activates `Product_Record_Page` | Build manually in phases or deploy the complete `force-app` package together |
| Order object staged deployment complains about missing page/components | `Order__c` activates `Order_Record_Page` | Deploy all integrated metadata together after dependencies exist |
| Permission set deployment fails | It references tabs or app not yet deployed | Deploy tabs/app before the `ebikes` permission set |
| Products are invisible in the public site | Guest profile/sharing rules were not deployed | Publish site, then deploy `guest-profile-metadata` |
| Product images do not display | External S3 URL is blocked | Deploy the CSP trusted site metadata |
| Product Explorer is empty | Sample products not imported or user lacks access | Import data and assign `ebikes` permission set |
| Filters do nothing | Message channels or ProductController missing | Confirm LMS metadata and Apex deployed |
| Order Builder does not load products | No products imported or no order record context | Import sample products and open an actual Order record |
| Drag-and-drop does not create items | User lacks CRUD/FLS on `Order_Item__c` | Assign `ebikes` permission set and check field permissions |
| Order Status Path reports EMP API error | Event subscription is not available in the context/org | Test on a supported Lightning record page and inspect event permissions/configuration |
| Public Create Case fields are missing | Guest profile FLS denies the custom Case fields | Review guest-profile field-level security as part of your learning build |
| Apex query returns access errors | Controllers execute in user mode | Confirm object and field permissions for the running user |
| UTAM login fails | `.env` URL expired or was not generated | Run `npm run test:ui:generate:login` again |
| LWC tests fail before execution | Node dependencies missing | Run `npm install` |

---

# 13. What Each User Journey Teaches

| Journey | Walkthrough | Main Concepts |
|---|---|---|
| Browse catalog | Open Product Explorer, filter products, select one | Apex wire methods, LMS, pagination, LDS |
| Inspect product | Open product record and similar products | Record context, related queries, NavigationMixin |
| Build reseller order | Create order and drag products into it | Master-detail, LDS CRUD, child events, optimistic UI |
| Watch status | Change order status or receive platform event | UI API updates, EMP API, event-driven UI |
| View reseller | Open Account with billing address | Standard object data with `lightning-map` |
| Browse public site | Visit Experience Cloud Product Explorer | Site routes, guest data access, branding |
| Submit support request | Open Create Case page | Standard object forms and guest security |

---

# 14. Build Checklist

## Org Setup

- [ ] Create a fresh Developer Org.
- [ ] Authenticate with CLI using alias `mydevorg`.
- [ ] Enable Digital Experiences.
- [ ] Enable ExperienceBundle Metadata API.
- [ ] Confirm My Domain is configured.
- [ ] Prepare your site metadata values for your org.
- [ ] Check Case for conflicting Product field metadata.

## Data Model

- [ ] Create `Product_Family__c`.
- [ ] Create `Product__c`.
- [ ] Create Product-to-Family lookup.
- [ ] Create `Order__c`.
- [ ] Create Account-to-Order lookup.
- [ ] Create `Order_Item__c`.
- [ ] Create Order-to-Order Item master-detail relationship.
- [ ] Create Order Item-to-Product lookup.
- [ ] Extend Case with Product and Case Category.
- [ ] Create `Manufacturing_Event__e`.

## Backend

- [ ] Build `PagedResult`.
- [ ] Build `ProductController`.
- [ ] Build `OrderController`.
- [ ] Build Experience Cloud helper Apex.
- [ ] Run Apex tests.

## Frontend

- [ ] Build utility and empty/error components.
- [ ] Build product presentation components.
- [ ] Build message channels.
- [ ] Build Product Explorer components.
- [ ] Build Similar Products.
- [ ] Build Order Builder.
- [ ] Build Order Status Path.
- [ ] Build Account Map.
- [ ] Build Experience Cloud Hero and Create Case components.
- [ ] Run Jest tests.

## Pages And Experience

- [ ] Create layouts and list views.
- [ ] Create Product Explorer app page.
- [ ] Create Product, Order, and Account record pages.
- [ ] Create tabs.
- [ ] Create E-Bikes Lightning app.
- [ ] Assign internal permission set.
- [ ] Deploy Experience Cloud metadata.
- [ ] Publish the site.
- [ ] Deploy guest profile metadata.

## Data And Validation

- [ ] Import Accounts.
- [ ] Import Product Families.
- [ ] Import Products.
- [ ] Create a reseller order manually.
- [ ] Drag products into the order.
- [ ] Test the public product browser.
- [ ] Test product details and Cases.
- [ ] Run Apex, Jest, and optional UTAM tests.

---

# 15. Deployment Checklist

- [ ] Target org alias resolves correctly: `sf org display --target-org mydevorg`.
- [ ] Digital Experiences prerequisites are enabled.
- [ ] Site admin, guest owner, and subdomain values match the target org.
- [ ] Full application metadata deploy succeeds:

```powershell
sf project deploy start --source-dir force-app --target-org mydevorg --wait 30
```

- [ ] Permission set assignment succeeds:

```powershell
sf org assign permset --name ebikes --target-org mydevorg
```

- [ ] Sample data import succeeds:

```powershell
sf data import tree --plan data/sample-data-plan.json --target-org mydevorg
```

- [ ] Experience site publishes successfully:

```powershell
sf community publish --name E-Bikes --target-org mydevorg
```

- [ ] Guest profile metadata deploys after publication:

```powershell
sf project deploy start --metadata-dir guest-profile-metadata --target-org mydevorg --wait 10
```

- [ ] Apex tests pass:

```powershell
sf apex run test --test-level RunLocalTests --result-format human --code-coverage --target-org mydevorg --wait 20
```

- [ ] Product Explorer displays 16 products.
- [ ] Product selection updates product details.
- [ ] Order Builder creates and edits order items.
- [ ] Public Experience site displays products.

---

# 16. Learning Checklist

By the end of this rebuild, you should be able to explain:

- [ ] Why Product Families are created before Products.
- [ ] The difference between lookup and master-detail relationships.
- [ ] Why Orders reference Accounts.
- [ ] Why Cases reuse Products instead of duplicating product information.
- [ ] How a Platform Event differs from a normal custom object.
- [ ] Why Apex controller methods are `@AuraEnabled(Cacheable=true)`.
- [ ] Why the Apex queries use `WITH USER_MODE`.
- [ ] How Lightning Message Service connects unrelated LWCs.
- [ ] How Lightning Data Service performs record CRUD without custom Apex.
- [ ] How `@api recordId` supplies record-page context.
- [ ] How App Builder pages assemble LWCs into user journeys.
- [ ] Why permission sets and guest profiles serve different users.
- [ ] Why an Experience site must be published before deploying its guest profile configuration.
- [ ] Why sample data import order matters.
- [ ] The difference between Apex tests, LWC Jest tests, and UTAM browser tests.
- [ ] Why a full integrated deployment is safer than partial deployment for exported page-activation metadata.

The central lesson of this application is that Salesforce UI development is layered: build a sound data model first, secure it deliberately, expose it through Apex and LDS, compose it into LWCs, and only then arrange those components into applications and public experiences.