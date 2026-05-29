# E-Bikes LWC Detailed Implementation Guide V3

## 1. Purpose

This manual explains how to rebuild, configure, deploy, test, and understand the Salesforce E-Bikes Lightning Web Components application as a beginner Salesforce developer.

It uses the existing guide as source material:

`docs/EBIKES_LWC_BUILD_SIMULATION_GUIDE.md`

The original build/simulation guide remains unchanged. This V3 file is the expanded implementation manual.

The app has two main experiences:

| Audience | Experience |
| --- | --- |
| Internal Salesforce users | A Lightning app named `EBikes` for browsing products, managing reseller orders, viewing accounts, and inspecting product records. |
| Public site users | An Experience Cloud site named `E-Bikes` for browsing products and creating support cases. |

By the end, you should understand:

| Topic | Where it appears |
| --- | --- |
| Salesforce DX project structure | `sfdx-project.json`, `force-app/main/default` |
| Scratch org setup | `config/project-scratch-def.json` |
| Custom objects and fields | `force-app/main/default/objects` |
| Apex and SOQL | `force-app/main/default/classes` |
| LWC HTML, JavaScript, CSS, and meta XML | `force-app/main/default/lwc` |
| Lightning Message Service | `force-app/main/default/messageChannels` |
| Lightning Data Service and UI API | LWC components such as `productCard`, `orderBuilder`, and `orderStatusPath` |
| Platform Events | `Manufacturing_Event__e` and `orderStatusPath` |
| Experience Cloud metadata | `experiences`, `networks`, `sites`, `navigationMenus`, `networkBranding` |
| Permission sets and guest access | `permissionsets`, `guest-profile-metadata` |
| Sample data import | `data/sample-data-plan.json` |
| Apex tests and Jest tests | `classes/*Test.cls`, LWC `__tests__` folders |
| CI/CD | `.github/workflows` or the sample YAML in this guide |

## 2. Important File Creation Rule

This guide is the only file created by this task:

`docs/EBIKES_LWC_DETAILED_IMPLEMENTATION_GUIDE_V3.md`

Do not modify, overwrite, rename, or delete:

`docs/EBIKES_LWC_BUILD_SIMULATION_GUIDE.md`

Both files should exist after generation.

## 3. Application Dependency Map

### Data Model

```text
Account
  |
  +-- Order__c
        |
        +-- Order_Item__c ---- Product__c ---- Product_Family__c

Case ---- Product__c

Manufacturing_Event__e --> orderStatusPath LWC --> Order__c.Status__c
```

### UI Model

```text
ProductController Apex
  --> productTileList
       --> productTile
       --> paginator
       --> placeholder
       --> errorPanel

productFilter
  --> ProductsFiltered message channel
       --> productTileList

productTileList
  --> ProductSelected message channel
       --> productCard

OrderController Apex
  --> orderBuilder
       --> orderItemTile
       --> placeholder
       --> errorPanel

ProductRecordInfoController Apex
  --> heroDetails
       --> hero
```

## 4. Recommended Build Strategy

There are two valid ways to learn this app.

### Learning Rebuild

Use this guide phase by phase. Study or create the metadata in dependency order:

1. Salesforce DX project setup
2. Scratch org setup
3. Data model
4. Security and permissions
5. Apex backend
6. LWC components
7. Lightning pages, tabs, app, and layouts
8. Experience Cloud site
9. Sample data
10. Tests and CI/CD

This teaches how the app is assembled.

### Faithful Repository Install

Deploy the existing metadata together after org prerequisites are ready:

```bash
sf project deploy start -o scratch-org
```

This is usually safer because exported Salesforce metadata contains cross-references. For example, object metadata can reference activated Lightning pages, pages can reference LWCs, and Experience Cloud pages can reference routes and components.

## 5. Prerequisites

### Local Tools

Install these before starting:

| Tool | Why it is needed |
| --- | --- |
| Git | Clones the repository and manages commits. |
| VS Code | Edits project files. |
| Salesforce Extension Pack for VS Code | Adds Salesforce commands and syntax help. |
| Salesforce CLI | Creates orgs, deploys metadata, imports data, and runs tests. |
| Node.js | Runs Jest, ESLint, Prettier, UTAM, and WebdriverIO. |
| Chrome | Runs optional UI tests. |

Verify the tools:

```bash
git --version
sf --version
node --version
npm --version
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| `sf` is not recognized | Install Salesforce CLI and restart VS Code terminal. |
| `npm` is not recognized | Install Node.js and restart the terminal. |
| Git command fails | Install Git and confirm it is on your PATH. |

## 6. Clone And Open The Project

Where: Git terminal and VS Code.

Commands:

```bash
git clone https://github.com/trailheadapps/ebikes-lwc
cd ebikes-lwc
code .
```

Why this step is needed:

Salesforce DX projects are source-controlled. You work with local files first, then deploy them to an org.

Salesforce concept:

Salesforce DX source format stores metadata as folders and files instead of relying only on changes made in Setup.

Verify:

In VS Code, confirm these paths exist:

| Path | Purpose |
| --- | --- |
| `force-app/main/default` | Main Salesforce metadata. |
| `config/project-scratch-def.json` | Scratch org shape. |
| `data` | Sample data files. |
| `guest-profile-metadata` | Guest profile deployment package. |
| `package.json` | Node scripts and test dependencies. |

## 7. Install Node Dependencies

Where: VS Code terminal.

Command:

```bash
npm install
```

Why:

This installs local developer tools used by the repository, including LWC Jest, ESLint, Prettier, UTAM, and WebdriverIO.

Verify:

```bash
npm run prettier:verify
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Peer dependency warning | Usually safe if install completes. |
| Install fails due to stale modules | Delete `node_modules`, keep `package-lock.json`, then run `npm install`. |
| Node version mismatch | Use Node 22 or install Volta so the repository can select `22.18.0`. |

## 8. Understand `sfdx-project.json`

Where: VS Code.

Exact file path:

`sfdx-project.json`

Full content:

```json
{
    "packageDirectories": [
        {
            "path": "force-app",
            "default": true
        }
    ],
    "namespace": "",
    "sourceApiVersion": "65.0"
}
```

Line by line:

| JSON property | Beginner meaning |
| --- | --- |
| `packageDirectories` | The Salesforce CLI should look inside these folders for metadata. |
| `"path": "force-app"` | The main package folder is named `force-app`. |
| `"default": true` | New generated metadata should go into this package by default. |
| `"namespace": ""` | This is not a namespaced managed package project. |
| `"sourceApiVersion": "65.0"` | Metadata is deployed with Salesforce API version 65.0. |

Why this step is needed:

The Salesforce CLI reads this file to know what to deploy and which API version to use.

Verify:

```bash
sf project deploy preview -o scratch-org
```

If you do not have `scratch-org` yet, this command will fail. That is expected until you create the org.

## 9. Understand The Scratch Org Definition

Where: VS Code.

Exact file path:

`config/project-scratch-def.json`

Full content:

```json
{
    "orgName": "Ebikes",
    "edition": "Developer",
    "hasSampleData": false,
    "features": ["Communities", "Walkthroughs", "EnableSetPasswordInApi"],
    "settings": {
        "communitiesSettings": {
            "enableNetworksEnabled": true
        },
        "experienceBundleSettings": {
            "enableExperienceBundleMetadata": true
        },
        "lightningExperienceSettings": {
            "enableS1DesktopEnabled": true
        },
        "mobileSettings": {
            "enableS1EncryptedStoragePref2": false
        },
        "userEngagementSettings": {
            "enableOrchestrationInSandbox": true
        }
    }
}
```

What each part teaches:

| JSON property | Beginner meaning |
| --- | --- |
| `orgName` | Display name of the scratch org. |
| `edition` | The Salesforce edition shape. |
| `hasSampleData` | `false` means Salesforce sample data is not loaded automatically. |
| `Communities` | Enables Experience Cloud features. |
| `Walkthroughs` | Enables in-app guidance prompt metadata. |
| `EnableSetPasswordInApi` | Allows CLI/API password setup flows used by some tooling. |
| `enableNetworksEnabled` | Turns on Experience Cloud networks. |
| `enableExperienceBundleMetadata` | Allows ExperienceBundle metadata deployment. |
| `enableS1DesktopEnabled` | Enables Lightning Experience. |

Why this step is needed:

The app contains Experience Cloud metadata. If the org is not created with the right features, deployment can fail.

## 10. Authenticate A Dev Hub

Where: Salesforce CLI.

Required for scratch orgs.

Command:

```bash
sf org login web -d -a devhub
```

What this does:

The browser opens. Log in to a Salesforce org with Dev Hub enabled. The alias `devhub` lets you refer to that org by name.

Manual UI path if Dev Hub is not enabled:

Setup > Dev Hub > Enable Dev Hub.

Verify:

```bash
sf org list
```

Expected result:

You should see an org with alias `devhub`.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Dev Hub not enabled | Enable Dev Hub in Setup, then log in again. |
| Browser login opens wrong account | Log out of Salesforce in the browser or use a private window. |

## 11. Create And Open A Scratch Org

Where: Salesforce CLI.

Command:

```bash
sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 30
```

What each flag means:

| Flag | Meaning |
| --- | --- |
| `-f config/project-scratch-def.json` | Use the scratch org definition file. |
| `-a scratch-org` | Give the org an alias named `scratch-org`. |
| `-d` | Make it the default target org. |
| `-y 30` | Keep it for 30 days. |

Open it:

```bash
sf org open -o scratch-org
```

Why:

A scratch org gives you a clean Salesforce environment for deployment and testing.

Verify:

```bash
sf org display -o scratch-org
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Scratch org creation fails for feature | Confirm `config/project-scratch-def.json` is unchanged and Dev Hub supports scratch orgs. |
| Alias already exists | Delete the old scratch org or use a different alias. |

## 12. Developer Org Alternative

If you are not using scratch orgs, authenticate a Developer Edition org:

```bash
sf org login web -a dev-org
```

Manual UI prerequisites for a Developer Org:

1. Setup > Digital Experiences > Settings.
2. Enable Digital Experiences.
3. Setup > Digital Experiences > Settings or Experience Management Settings.
4. Enable ExperienceBundle Metadata API.
5. Confirm My Domain is deployed.

Important:

Developer orgs can drift from source because they persist. Scratch orgs are cleaner for repeatable builds.

## 13. Deploy The Main Salesforce Metadata

Where: Salesforce CLI.

Command:

```bash
sf project deploy start -o scratch-org
```

Equivalent explicit source directory command:

```bash
sf project deploy start --source-dir force-app -o scratch-org
```

Why this step is needed:

This deploys objects, fields, Apex, LWCs, Lightning pages, layouts, tabs, app metadata, message channels, static resources, Experience Cloud metadata, CSP trusted sites, and more.

Salesforce concept:

Metadata deployment moves source-controlled configuration into an org.

Verify:

```bash
sf project deploy report -o scratch-org
```

Common errors and fixes:

| Error | Likely cause | Fix |
| --- | --- | --- |
| ExperienceBundle deployment fails | Experience Cloud not enabled | Recreate scratch org from `config/project-scratch-def.json` or enable settings in Developer Org. |
| Component reference not found | Partial deployment missed dependencies | Deploy all of `force-app`. |
| Unknown user permission | Org shape does not match project | Use the scratch org definition file. |

## 14. Assign The Internal Permission Set

Where: Salesforce CLI.

Exact metadata file:

`force-app/main/default/permissionsets/ebikes.permissionset-meta.xml`

Command:

```bash
sf org assign permset -n ebikes -o scratch-org
```

Why:

The default user needs access to custom objects, fields, tabs, Apex classes, and the E-Bikes Lightning app.

Salesforce concept:

A permission set adds access without changing the user's profile. This is the preferred way to grant app-specific permissions.

Verify:

Manual UI path:

Setup > Users > Users > open your user > Permission Set Assignments.

CLI verification:

```bash
sf data query -o scratch-org --query "SELECT PermissionSet.Name, Assignee.Username FROM PermissionSetAssignment WHERE PermissionSet.Name = 'ebikes'"
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Permission set not found | Deploy `force-app` first. |
| Missing tab/app permission in UI | Confirm assignment succeeded and refresh the browser. |

## 15. Data Model Implementation

### 15.1 Object Inventory

Where: VS Code.

Exact folder:

`force-app/main/default/objects`

| Object | Type | Purpose |
| --- | --- | --- |
| `Product_Family__c` | Custom object | Groups related bike products. |
| `Product__c` | Custom object | Stores bike catalog records. |
| `Order__c` | Custom object | Stores reseller orders. |
| `Order_Item__c` | Custom object | Stores products and quantities in an order. |
| `Manufacturing_Event__e` | Platform Event | Sends manufacturing status changes to the UI. |
| `Case` | Standard object extension | Adds product support fields to standard Case. |

Manual UI path:

Setup > Object Manager.

Verify:

```bash
sf data query -o scratch-org --query "SELECT QualifiedApiName FROM EntityDefinition WHERE QualifiedApiName IN ('Product_Family__c','Product__c','Order__c','Order_Item__c','Manufacturing_Event__e')"
```

### 15.2 Product Family Object

Where: VS Code or Salesforce Setup UI.

Exact object folder:

`force-app/main/default/objects/Product_Family__c`

Important files:

| File | Purpose |
| --- | --- |
| `Product_Family__c.object-meta.xml` | Defines the custom object. |
| `fields/Category__c.field-meta.xml` | Picklist field for product family category. |
| `fields/Description__c.field-meta.xml` | Text area field for description. |
| `listViews/All_Product_Families.listView-meta.xml` | Product family list view. |

What to create manually in Setup:

1. Setup > Object Manager > Create > Custom Object.
2. Label: `Product Family`.
3. Plural Label: `Product Families`.
4. Object Name: `Product_Family`.
5. Record Name: `Product Family Name`.
6. Data Type: Text.
7. Add field `Category__c` as a picklist with values `Commuter`, `Hybrid`, `Mountain`.
8. Add field `Description__c` as a text area.

Why:

Products can belong to a family. Parent objects should exist before child objects that reference them.

Concept:

Custom objects represent business data that standard Salesforce objects do not already cover.

Verify:

```bash
sf data query -o scratch-org --query "SELECT Id, Name, Category__c FROM Product_Family__c LIMIT 5"
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Object not found | Deploy object metadata or create it in Object Manager. |
| Picklist value missing | Check `Category__c` values in Object Manager. |

### 15.3 Product Object

Exact object folder:

`force-app/main/default/objects/Product__c`

Important fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Product_Family__c` | Lookup | Connects a product to a product family. |
| `Description__c` | Long text area | Product description. |
| `MSRP__c` | Currency | Retail price. |
| `Picture_URL__c` | URL | Product image URL. |
| `Category__c` | Picklist | Product category. |
| `Level__c` | Picklist | Beginner, Enthusiast, Racer. |
| `Material__c` | Picklist | Aluminum or Carbon. |
| `Battery__c` | Text | Battery spec. |
| `Charger__c` | Text | Charger spec. |
| `Motor__c` | Text | Motor spec. |
| `Fork__c` | Text | Fork spec. |
| `Front_Brakes__c` | Text | Front brake spec. |
| `Rear_Brakes__c` | Text | Rear brake spec. |
| `Frame_Color__c` | Picklist | Color value. |
| `Handlebar_Color__c` | Picklist | Color value. |
| `Seat_Color__c` | Picklist | Color value. |
| `Waterbottle_Color__c` | Picklist | Color value. |

What to create manually in Setup:

1. Setup > Object Manager > Create > Custom Object.
2. Label: `Product`.
3. Plural Label: `Products`.
4. Object Name: `Product`.
5. Record Name: `Product Name`.
6. Data Type: Text.
7. Add the fields listed above.

Why:

This is the central catalog object used by internal users and public site users.

Concept:

Lookup fields connect objects without making the child record fully dependent on the parent.

Verify:

```bash
sf data query -o scratch-org --query "SELECT Id, Name, Product_Family__c, MSRP__c FROM Product__c LIMIT 5"
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Lookup field cannot be created | Create `Product_Family__c` first. |
| Product Explorer has no filters | Confirm `Category__c`, `Level__c`, `Material__c`, and `MSRP__c` exist. |

### 15.4 Order Object

Exact object folder:

`force-app/main/default/objects/Order__c`

Important files:

| File | Purpose |
| --- | --- |
| `Order__c.object-meta.xml` | Defines the custom order object. |
| `fields/Account__c.field-meta.xml` | Required lookup to Account. |
| `fields/Status__c.field-meta.xml` | Picklist status field. |

Status values:

```text
Draft
Submitted to Manufacturing
Approved by Manufacturing
In Production
```

Why:

Reseller orders belong to Accounts and move through manufacturing statuses.

Concept:

This object extends the sales process with a custom order flow instead of using standard Order.

Verify:

```bash
sf data query -o scratch-org --query "SELECT Id, Name, Account__c, Status__c FROM Order__c LIMIT 5"
```

### 15.5 Order Item Object

Exact object folder:

`force-app/main/default/objects/Order_Item__c`

Important fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Order__c` | Master-detail to `Order__c` | Parent order. |
| `Product__c` | Lookup to `Product__c` | Product being ordered. |
| `Price__c` | Currency | Unit price. |
| `Qty_S__c` | Number | Small size quantity. |
| `Qty_M__c` | Number | Medium size quantity. |
| `Qty_L__c` | Number | Large size quantity. |

Why:

One order can contain many products. Each product line needs its own quantities and price.

Concept:

Master-detail means the detail record is tightly owned by the master. If an `Order__c` is deleted, its `Order_Item__c` records are deleted too.

Verify:

```bash
sf data query -o scratch-org --query "SELECT Id, Order__c, Product__c, Qty_S__c, Qty_M__c, Qty_L__c FROM Order_Item__c LIMIT 5"
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Master-detail field cannot be created | Create `Order__c` first. |
| Order Builder cannot save rows | Assign the `ebikes` permission set and confirm field permissions. |

### 15.6 Case Standard Object Extension

Exact folder:

`force-app/main/default/objects/Case`

Files:

| File | Purpose |
| --- | --- |
| `fields/Product__c.field-meta.xml` | Lookup from Case to Product. |
| `fields/Case_Category__c.field-meta.xml` | Support category picklist. |
| `listViews/My_Cases.listView-meta.xml` | Case list view. |

Picklist values for `Case_Category__c`:

```text
Mechanical
Electrical
Electronic
Structural
Other
```

Why:

Public site visitors can create support cases related to products.

Concept:

Standard objects can be extended with custom fields.

Common error:

Some orgs already contain a Case field called Product. If deployment reports a duplicate field conflict, inspect:

Setup > Object Manager > Case > Fields & Relationships.

Only remove a conflicting custom field if this is a fresh learning org and no data depends on it.

### 15.7 Platform Event Object

Exact folder:

`force-app/main/default/objects/Manufacturing_Event__e`

Fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Order_Id__c` | Text | Stores the affected `Order__c` record Id. |
| `Status__c` | Text | Stores the new manufacturing status. |

Why:

The `orderStatusPath` LWC listens for manufacturing events and updates the visible order status.

Concept:

Platform Events are event messages, not normal database records. They are used for event-driven communication.

Verify the object exists:

```bash
sf data query -o scratch-org --query "SELECT QualifiedApiName FROM EntityDefinition WHERE QualifiedApiName = 'Manufacturing_Event__e'"
```

Publish a test event in Anonymous Apex:

```apex
Manufacturing_Event__e eventMessage = new Manufacturing_Event__e(
    Order_Id__c = 'REPLACE_WITH_ORDER_ID',
    Status__c = 'In Production'
);
EventBus.publish(eventMessage);
```

Where to run it:

VS Code Command Palette > SFDX: Execute Anonymous Apex with Currently Selected Text.

## 16. Security And Permissions

### 16.1 Internal Permission Set

Exact file:

`force-app/main/default/permissionsets/ebikes.permissionset-meta.xml`

What it grants:

| Permission area | Purpose |
| --- | --- |
| Object permissions | Access to Product, Product Family, Order, and Order Item records. |
| Field permissions | Access to custom fields used by LWCs and pages. |
| Apex class access | Allows LWC calls to Apex controllers. |
| Tab settings | Makes custom tabs visible. |
| Application visibility | Makes the E-Bikes app available. |

Why:

Salesforce blocks access unless users have object, field, and class permissions.

Verify:

```bash
sf org assign permset -n ebikes -o scratch-org
```

Then open:

App Launcher > E-Bikes.

### 16.2 Guest Profile Metadata

Exact folder:

`guest-profile-metadata`

Important files:

| File | Purpose |
| --- | --- |
| `package.xml` | Tells Metadata API what is inside this deployment package. |
| `profiles/E-Bikes Profile.profile` | Guest profile settings. |
| `sharingRules/Product__c.sharingRules` | Public sharing rules for products. |
| `sharingRules/Product_Family__c.sharingRules` | Public sharing rules for product families. |
| `profilePasswordPolicies/*` | Guest profile password policy metadata. |
| `profileSessionSettings/*` | Guest profile session settings. |

When to deploy:

Deploy this after:

1. Main `force-app` metadata is deployed.
2. The Experience Cloud site exists.
3. The Experience Cloud site is published.

Command:

```bash
sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10
```

Why:

Guest users have separate security. Public site access does not automatically mean access to records.

Verify:

1. Setup > Digital Experiences > All Sites.
2. Open the E-Bikes public URL in an incognito browser.
3. Confirm Product Explorer shows products.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Guest profile deploy fails | Publish the site first, then deploy guest metadata. |
| Public site shows no products | Check guest sharing rules and guest profile object permissions. |
| Product detail page inaccessible | Check field-level security and sharing for `Product__c`. |

## 17. Apex Backend

Exact folder:

`force-app/main/default/classes`

### 17.1 Apex Class Inventory

| Class | Purpose |
| --- | --- |
| `PagedResult` | Generic wrapper for paginated Apex results. |
| `ProductController` | Queries products for Product Explorer and similar-products features. |
| `OrderController` | Queries order items and product details for Order Builder. |
| `ProductRecordInfoController` | Resolves a product or family name to a Salesforce record. |
| `HeroDetailsPositionCustomPicklist` | Supplies left/right design options for Experience Builder. |
| `CommunitiesLandingController` | Redirects Visualforce community landing pages. |
| `TestProductController` | Tests product query behavior. |
| `TestOrderController` | Tests order query behavior. |
| `CommunitiesLandingControllerTest` | Tests Visualforce landing behavior. |

### 17.2 Apex Concepts In This App

| Syntax | Beginner explanation |
| --- | --- |
| `public with sharing class` | The class respects Salesforce sharing rules for the running user. |
| `@AuraEnabled` | Exposes an Apex method or property to Lightning components. |
| `@AuraEnabled(cacheable=true)` | Allows LWC wire service to cache read-only Apex results. |
| `SOQL` | Salesforce Object Query Language, used to query records. |
| `WITH USER_MODE` | Runs queries with the user's object and field permissions. |
| `List<Product__c>` | A collection of Product records. |
| `Map<String, Object>` | A key-value structure often used for flexible filters. |

### 17.3 `PagedResult`

Exact files:

| File | Purpose |
| --- | --- |
| `force-app/main/default/classes/PagedResult.cls` | Apex wrapper class. |
| `force-app/main/default/classes/PagedResult.cls-meta.xml` | Apex metadata file. |

What it does:

`PagedResult` holds:

| Property | Meaning |
| --- | --- |
| `records` | The records for the current page. |
| `totalItemCount` | Total matching records. |
| `pageSize` | Number of records per page. |
| `pageNumber` | Current page number. |

Why:

Product Explorer needs pages of products instead of loading everything at once.

### 17.4 `ProductController`

Exact files:

| File | Purpose |
| --- | --- |
| `force-app/main/default/classes/ProductController.cls` | Product Apex controller. |
| `force-app/main/default/classes/ProductController.cls-meta.xml` | Metadata for the Apex class. |

Used by:

| LWC | Purpose |
| --- | --- |
| `productTileList` | Gets filtered and paginated products. |
| `similarProducts` | Gets related products from the same family. |

Important ideas:

| Idea | Explanation |
| --- | --- |
| Filter parameters | The LWC passes search text, price, category, material, and level. |
| SOQL query | Apex queries matching `Product__c` records. |
| Pagination | Apex returns only one page at a time. |
| `WITH USER_MODE` | The query respects user permissions. |

Verify:

```bash
sf apex run test -o scratch-org -n TestProductController -r human -w 20
```

Common errors:

| Error | Fix |
| --- | --- |
| LWC says Apex method failed | Confirm Apex deployed and permission set assigned. |
| Query returns no products | Import sample data. |
| Access error | Confirm object and field access in `ebikes` permission set. |

### 17.5 `OrderController`

Exact files:

| File | Purpose |
| --- | --- |
| `force-app/main/default/classes/OrderController.cls` | Order Apex controller. |
| `force-app/main/default/classes/OrderController.cls-meta.xml` | Metadata for the Apex class. |

Used by:

`force-app/main/default/lwc/orderBuilder`

What it teaches:

`OrderController` shows how Apex can query child records, parent fields, and related product details for a custom UI.

Verify:

```bash
sf apex run test -o scratch-org -n TestOrderController -r human -w 20
```

### 17.6 Experience Cloud Helper Apex

Exact classes:

| Class | Purpose |
| --- | --- |
| `ProductRecordInfoController` | Lets `heroDetails` find a product or family record by name. |
| `HeroDetailsPositionCustomPicklist` | Provides design-time values in Experience Builder. |
| `CommunitiesLandingController` | Supports Visualforce landing behavior. |

Why:

Experience Builder components can expose design properties and navigation behavior, but sometimes they need Apex to resolve Salesforce records.

## 18. Lightning Web Components

Exact folder:

`force-app/main/default/lwc`

### 18.1 LWC File Structure

Each component folder usually contains:

| File | Purpose |
| --- | --- |
| `componentName.html` | Template markup shown in the browser. |
| `componentName.js` | JavaScript logic. |
| `componentName.css` | Component-scoped styles. |
| `componentName.js-meta.xml` | Salesforce exposure and target configuration. |
| `__tests__` | Jest unit tests. |
| `__utam__` | Optional UI test page object definitions. |

Example:

```text
force-app/main/default/lwc/productTileList
  productTileList.html
  productTileList.js
  productTileList.css
  productTileList.js-meta.xml
  __tests__/productTileList.test.js
  __utam__/productTileList.utam.json
```

### 18.2 LWC JavaScript Concepts

| Syntax | Meaning |
| --- | --- |
| `import { LightningElement } from 'lwc'` | Imports the base class for LWC components. |
| `@api` | Public property or method that a parent component or Lightning page can set. |
| `@wire` | Reactive data connection to Apex, LDS, UI API, or Message Service. |
| `dispatchEvent(new CustomEvent(...))` | Sends a custom event from child to parent. |
| `NavigationMixin` | Lets a component navigate to records, pages, and URLs. |
| `getRecord` | Lightning Data Service wire adapter for reading records. |
| `updateRecord` | UI Record API function for saving record updates. |

### 18.3 LWC HTML Concepts

| Syntax | Meaning |
| --- | --- |
| `<template>` | Root element of an LWC HTML file. |
| `if:true={property}` | Render a block only when the property is true. |
| `for:each={items}` | Repeat markup for each item in an array. |
| `onclick={handleClick}` | Run JavaScript when the user clicks. |
| `<lightning-button>` | Standard Salesforce base component. |
| `<c-product-tile>` | Custom LWC named `productTile`. |

### 18.4 LWC Meta XML Concepts

A component's `*.js-meta.xml` file tells Salesforce where the component can be used.

Typical structure:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>65.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__AppPage</target>
        <target>lightning__RecordPage</target>
        <target>lightningCommunity__Page</target>
    </targets>
</LightningComponentBundle>
```

What it means:

| Element | Meaning |
| --- | --- |
| `apiVersion` | Salesforce API version for the component. |
| `isExposed` | Whether the component appears in builders. |
| `target` | Where the component can be placed. |

### 18.5 Foundation Components

| Component | Exact folder | Purpose |
| --- | --- | --- |
| `ldsUtils` | `lwc/ldsUtils` | Converts Salesforce errors into readable messages. |
| `errorPanel` | `lwc/errorPanel` | Displays errors in a friendly UI. |
| `placeholder` | `lwc/placeholder` | Displays an empty state with a logo. |
| `paginator` | `lwc/paginator` | Handles next and previous page controls. |
| `productTile` | `lwc/productTile` | Displays a product card and supports drag/drop. |
| `productListItem` | `lwc/productListItem` | Displays a compact product row. |
| `orderItemTile` | `lwc/orderItemTile` | Displays and edits one order item. |

Verify:

```bash
npm run test:unit -- --findRelatedTests force-app/main/default/lwc/paginator/__tests__/paginator.test.js
```

### 18.6 Product Explorer Components

| Component | Exact folder | Purpose |
| --- | --- | --- |
| `productFilter` | `lwc/productFilter` | Publishes filter values through Lightning Message Service. |
| `productTileList` | `lwc/productTileList` | Calls Apex and displays products. |
| `productCard` | `lwc/productCard` | Shows selected product details using LDS. |
| `similarProducts` | `lwc/similarProducts` | Shows products from the same product family. |

How the feature works:

1. User changes filters in `productFilter`.
2. `productFilter` publishes a message on `ProductsFiltered__c`.
3. `productTileList` receives the message.
4. `productTileList` calls `ProductController`.
5. User selects a product tile.
6. `productTileList` publishes selected product Id on `ProductSelected__c`.
7. `productCard` receives the selected Id and loads record fields.

Concept:

Lightning Message Service lets unrelated components on the same page communicate without a direct parent-child relationship.

### 18.7 Order Components

| Component | Exact folder | Purpose |
| --- | --- | --- |
| `orderBuilder` | `lwc/orderBuilder` | Lets users add products to an order and edit quantities. |
| `orderItemTile` | `lwc/orderItemTile` | Displays one item in the order. |
| `orderStatusPath` | `lwc/orderStatusPath` | Shows order status and listens for manufacturing events. |

Concepts taught:

| Concept | Where it appears |
| --- | --- |
| Drag and drop | Product tiles dragged into Order Builder. |
| UI Record API | `createRecord`, `updateRecord`, and `deleteRecord` style operations. |
| Parent-child events | `orderItemTile` sends changes to `orderBuilder`. |
| EMP API | `orderStatusPath` subscribes to Platform Events. |
| Picklist UI | `orderStatusPath` uses object and picklist metadata. |

### 18.8 Experience Cloud Components

| Component | Exact folder | Purpose |
| --- | --- | --- |
| `hero` | `lwc/hero` | Configurable site hero image or video. |
| `heroDetails` | `lwc/heroDetails` | Overlay button that links to product or family records. |
| `createCase` | `lwc/createCase` | Public case creation form. |

Why:

These components are built for the public Experience Cloud site.

Verify:

Open:

Setup > Digital Experiences > All Sites > E-Bikes > Builder.

Confirm the home page and Create Case page include these components.

## 19. Lightning Message Service

Exact folder:

`force-app/main/default/messageChannels`

Files:

| File | Purpose |
| --- | --- |
| `ProductsFiltered.messageChannel-meta.xml` | Message contract for product filters. |
| `ProductSelected.messageChannel-meta.xml` | Message contract for selected product Id. |

Concept:

A message channel is a named communication contract. Components can publish or subscribe to it.

Typical JavaScript pattern:

```js
import { publish, MessageContext } from 'lightning/messageService';
import PRODUCTS_FILTERED_MESSAGE from '@salesforce/messageChannel/ProductsFiltered__c';
```

Beginner explanation:

`MessageContext` connects the component to the Lightning page message bus. `publish` sends data to any subscribed component.

Verify:

1. Open E-Bikes app.
2. Open Product Explorer.
3. Select a product.
4. Confirm Product Card updates.
5. Change filters.
6. Confirm product tiles update.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Product card never updates | Confirm `ProductSelected.messageChannel-meta.xml` deployed. |
| Filters do nothing | Confirm `ProductsFiltered.messageChannel-meta.xml` deployed and components are on the same Lightning page. |

## 20. Lightning Data Service And UI API

Used in:

| Component | Purpose |
| --- | --- |
| `productCard` | Reads selected product records. |
| `accountMap` | Reads Account address fields. |
| `orderBuilder` | Creates, updates, and deletes order items. |
| `orderStatusPath` | Reads picklist values and updates `Order__c.Status__c`. |

Concept:

Lightning Data Service handles record access, caching, sharing, and field-level security. You often do not need custom Apex for simple record reads and writes.

Common imports:

```js
import { getRecord, updateRecord } from 'lightning/uiRecordApi';
import { getObjectInfo, getPicklistValues } from 'lightning/uiObjectInfoApi';
```

Beginner explanation:

| Import | Meaning |
| --- | --- |
| `getRecord` | Read a record and selected fields. |
| `updateRecord` | Save changed field values. |
| `getObjectInfo` | Read metadata about an object. |
| `getPicklistValues` | Read allowed values for a picklist field. |

Verify:

Open a Product record. If fields load without custom Apex errors, LDS is working.

## 21. Lightning Pages, Tabs, App, Layouts, And List Views

### 21.1 FlexiPages

Exact folder:

`force-app/main/default/flexipages`

Files:

| File | Purpose |
| --- | --- |
| `Product_Explorer.flexipage-meta.xml` | App page containing filters, product tiles, and product card. |
| `Product_Record_Page.flexipage-meta.xml` | Product record page with similar products. |
| `Order_Record_Page.flexipage-meta.xml` | Order record page with Order Builder and Status Path. |
| `Account_Record_Page.flexipage-meta.xml` | Account record page with account map. |

Manual UI path:

Setup > Lightning App Builder.

Why:

FlexiPages assemble components into real Salesforce pages.

Verify:

1. App Launcher > E-Bikes.
2. Open Product Explorer.
3. Open Product record.
4. Open Order record.

### 21.2 Aura Template

Exact folder:

`force-app/main/default/aura/pageTemplate_2_7_3`

Files:

| File | Purpose |
| --- | --- |
| `pageTemplate_2_7_3.cmp` | Aura page template component. |
| `pageTemplate_2_7_3.design` | Design-time settings. |
| `pageTemplate_2_7_3.svg` | Builder icon. |
| `pageTemplate_2_7_3.cmp-meta.xml` | Metadata exposure. |

Concept:

Aura and LWC can coexist. Experience Cloud page templates are often Aura-based even when page content is LWC.

### 21.3 Tabs

Exact folder:

`force-app/main/default/tabs`

Files:

| File | Purpose |
| --- | --- |
| `Product_Explorer.tab-meta.xml` | Tab for Product Explorer app page. |
| `Product__c.tab-meta.xml` | Product object tab. |
| `Product_Family__c.tab-meta.xml` | Product Family object tab. |
| `Order__c.tab-meta.xml` | Reseller Order object tab. |

Concept:

Tabs make pages and objects available in app navigation.

### 21.4 Lightning App

Exact file:

`force-app/main/default/applications/EBikes.app-meta.xml`

Purpose:

Defines the internal Lightning application named E-Bikes.

Verify:

App Launcher > E-Bikes.

Common error:

If the app is missing, confirm `EBikes.app-meta.xml` deployed and the `ebikes` permission set is assigned.

### 21.5 Layouts

Exact folder:

`force-app/main/default/layouts`

Files:

| File | Purpose |
| --- | --- |
| `Product__c-Product Layout.layout-meta.xml` | Product record field layout. |
| `Product_Family__c-Product Family Layout.layout-meta.xml` | Product family record layout. |
| `Order__c-Order Layout.layout-meta.xml` | Order record layout. |
| `Order_Item__c-Order Item Layout.layout-meta.xml` | Order item layout. |
| `Case-Case Layout.layout-meta.xml` | Case layout with E-Bikes fields. |
| `CaseClose-Close Case Layout.layout-meta.xml` | Case close layout. |

Concept:

Page layouts control field arrangement on standard record detail and edit pages.

### 21.6 List Views

List view folders are inside object folders:

| File | Purpose |
| --- | --- |
| `objects/Product__c/listViews/ProductList.listView-meta.xml` | Product list view. |
| `objects/Product_Family__c/listViews/All_Product_Families.listView-meta.xml` | Product family list. |
| `objects/Order__c/listViews/All_Orders.listView-meta.xml` | Order list. |
| `objects/Case/listViews/My_Cases.listView-meta.xml` | Case list. |

Concept:

List views are saved table filters for users.

## 22. Static Resources, Content Assets, And CSP

### 22.1 Static Resources

Exact folder:

`force-app/main/default/staticresources`

Files:

| File | Purpose |
| --- | --- |
| `bike_assets.resource-meta.xml` | Static resource metadata. |
| `bike_assets/logo.svg` | E-Bikes logo. |
| `bike_assets/commuter.png` | Image asset. |
| `bike_assets/enthusiast.png` | Image asset. |
| `bike_assets/racer.png` | Image asset. |
| `bike_assets/CyclingGrass.jpg` | Hero background image. |

Concept:

Static resources package files with Salesforce metadata so components can reference them.

### 22.2 Content Assets

Exact folder:

`force-app/main/default/contentassets`

Files:

| File | Purpose |
| --- | --- |
| `logo.asset` | Site content asset. |
| `logo.asset-meta.xml` | Metadata for logo asset. |
| `ebikeslogo.asset` | E-Bikes logo content asset. |
| `ebikeslogo.asset-meta.xml` | Metadata for E-Bikes logo asset. |

Concept:

Content assets are often used by Experience Cloud pages and branding.

### 22.3 CSP Trusted Site

Exact file:

`force-app/main/default/cspTrustedSites/s3_us_west_2_amazonaws_com.cspTrustedSite-meta.xml`

Full content:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<CspTrustedSite xmlns="http://soap.sforce.com/2006/04/metadata">
    <canAccessCamera>false</canAccessCamera>
    <canAccessMicrophone>false</canAccessMicrophone>
    <context>All</context>
    <description>Amazon S3 hosted sample app media</description>
    <endpointUrl>https://s3-us-west-2.amazonaws.com</endpointUrl>
    <isActive>true</isActive>
    <isApplicableToConnectSrc>false</isApplicableToConnectSrc>
    <isApplicableToFontSrc>false</isApplicableToFontSrc>
    <isApplicableToFrameSrc>false</isApplicableToFrameSrc>
    <isApplicableToImgSrc>true</isApplicableToImgSrc>
    <isApplicableToMediaSrc>true</isApplicableToMediaSrc>
    <isApplicableToStyleSrc>false</isApplicableToStyleSrc>
</CspTrustedSite>
```

Why:

Sample product records use public Amazon S3 image URLs. Salesforce blocks external images unless the domain is trusted.

Manual UI path:

Setup > CSP Trusted Sites.

Verify:

Product images display in Product Explorer and Experience Cloud.

## 23. Experience Cloud Implementation

### 23.1 Experience Cloud Metadata Inventory

| Folder | Purpose |
| --- | --- |
| `force-app/main/default/sites` | Salesforce Site metadata. |
| `force-app/main/default/networks` | Experience Cloud network metadata. |
| `force-app/main/default/experiences/E_Bikes1` | Experience Bundle pages, routes, branding, and config. |
| `force-app/main/default/navigationMenus` | Public site navigation menus. |
| `force-app/main/default/networkBranding` | Site branding metadata. |
| `force-app/main/default/audience` | Audience targeting metadata. |
| `force-app/main/default/pages` | Visualforce landing page. |

### 23.2 Required Manual UI Prerequisites For Developer Orgs

Skip this section for scratch orgs created from `config/project-scratch-def.json`.

Manual UI steps:

1. Setup > Digital Experiences > Settings.
2. Enable Digital Experiences.
3. Save.
4. Setup > Experience Management Settings.
5. Enable ExperienceBundle Metadata API.
6. Save.
7. Setup > My Domain.
8. Confirm My Domain is deployed.

Why:

Experience Cloud metadata cannot deploy correctly until these features are active.

### 23.3 Site Metadata

Exact file:

`force-app/main/default/sites/E_Bikes.site-meta.xml`

Important values:

| Element | Meaning |
| --- | --- |
| `siteAdmin` | Admin username for the site. |
| `siteGuestRecordDefaultOwner` | Owner for guest-created records. |
| `subdomain` | Site subdomain. |

Developer Org warning:

If deploying to a persistent Developer Org, update these values in your working copy to match that org. Do not edit the original build guide.

### 23.4 Deploy Experience Cloud

Main deployment command:

```bash
sf project deploy start -o scratch-org
```

Publish command:

```bash
sf community publish -n E-Bikes -o scratch-org
```

Why:

Deployment creates the site metadata. Publishing makes the site available.

Verify:

Setup > Digital Experiences > All Sites.

Open the E-Bikes site URL.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Site not visible after deploy | Publish it. |
| Publish command fails immediately after deploy | Wait two minutes, then retry. |
| Guest user cannot see data | Deploy `guest-profile-metadata` after publishing. |

## 24. Visualforce Page

Exact files:

| File | Purpose |
| --- | --- |
| `force-app/main/default/pages/CommunitiesLanding.page` | Visualforce page source. |
| `force-app/main/default/pages/CommunitiesLanding.page-meta.xml` | Metadata for the page. |

Related Apex:

`force-app/main/default/classes/CommunitiesLandingController.cls`

Concept:

Visualforce is Salesforce's older page framework. It can still be used alongside LWC, especially for site landing or redirect patterns.

Verify:

Run:

```bash
sf apex run test -o scratch-org -n CommunitiesLandingControllerTest -r human -w 20
```

## 25. External Integrations, OAuth, Tokens, And API Keys

Codebase finding:

This app does not require:

| Item | Required? |
| --- | --- |
| External integration registration | No |
| OAuth 2.0 flow | No |
| Connected App | No |
| Client ID or client secret | No |
| API key | No |
| Access token | No |
| Named Credential | No |
| External Credential | No |
| Apex HTTP callout | No |

The only external runtime dependency is public media hosted at:

```text
https://s3-us-west-2.amazonaws.com
```

This is handled by the CSP Trusted Site metadata described earlier.

Beginner explanation:

OAuth is needed when Salesforce must securely log in to another service or another service must securely access Salesforce. This app only displays public image and media URLs, so no OAuth flow is needed.

## 26. Sample Data

### 26.1 Data Files

Exact folder:

`data`

Files:

| File | Purpose |
| --- | --- |
| `Accounts.json` | Sample reseller accounts. |
| `Product_Family__cs.json` | Product family records. |
| `Product__cs.json` | Product records. |
| `sample-data-plan.json` | Import order and reference resolution. |

Full import plan:

```json
[
    {
        "sobject": "Account",
        "saveRefs": false,
        "files": ["Accounts.json"]
    },
    {
        "sobject": "Product_Family__c",
        "saveRefs": true,
        "files": ["Product_Family__cs.json"]
    },
    {
        "sobject": "Product__c",
        "resolveRefs": true,
        "files": ["Product__cs.json"]
    }
]
```

Why:

Products reference product families. The import plan saves family references first, then resolves them while importing products.

### 26.2 Import Data

Where: Salesforce CLI.

Command:

```bash
sf data import tree -p data/sample-data-plan.json -o scratch-org
```

Verify:

```bash
sf data query -o scratch-org --query "SELECT Count() FROM Account"
sf data query -o scratch-org --query "SELECT Count() FROM Product_Family__c"
sf data query -o scratch-org --query "SELECT Count() FROM Product__c"
```

Expected:

| Object | Expected count |
| --- | --- |
| Product families | 4 |
| Products | 16 |
| Included sample accounts | 3 |

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Invalid field | Deploy metadata before importing data. |
| Reference not found | Use `sample-data-plan.json`, not individual files out of order. |
| Image URLs show broken images | Confirm CSP Trusted Site deployed. |

## 27. Create A Sample Reseller Order

The sample data does not include orders or order items.

Manual UI steps:

1. Open the org:

```bash
sf org open -o scratch-org
```

2. App Launcher > E-Bikes.
3. Open Reseller Orders.
4. Click New.
5. Choose Account: `Wheelworks` or another sample account.
6. Set Status: `Draft`.
7. Save.
8. Open the order record.
9. Drag products into the Order Builder.
10. Adjust quantities and prices.

Why:

This demonstrates custom object creation, child records, drag/drop UI, and Lightning Data Service writes.

Verify:

```bash
sf data query -o scratch-org --query "SELECT Id, Name, Account__r.Name, Status__c FROM Order__c"
sf data query -o scratch-org --query "SELECT Id, Order__c, Product__r.Name, Qty_S__c, Qty_M__c, Qty_L__c FROM Order_Item__c"
```

## 28. Test Manufacturing Event Updates

Prerequisite:

Create at least one `Order__c` record.

Get an order Id:

```bash
sf data query -o scratch-org --query "SELECT Id, Name, Status__c FROM Order__c LIMIT 1"
```

Create a file in VS Code:

`scripts/apex/publishManufacturingEvent.apex`

Example content:

```apex
Manufacturing_Event__e eventMessage = new Manufacturing_Event__e(
    Order_Id__c = 'REPLACE_WITH_ORDER_ID',
    Status__c = 'In Production'
);

Database.SaveResult result = EventBus.publish(eventMessage);
System.debug('Published manufacturing event: ' + result.isSuccess());
```

Run it:

```bash
sf apex run -o scratch-org -f scripts/apex/publishManufacturingEvent.apex
```

Why:

This tests the event-driven UI behavior used by `orderStatusPath`.

Concept:

The EMP API lets Lightning components subscribe to event streams.

Verify:

Open the order record page and watch the Status Path update after the event is published.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Status does not update | Confirm the order Id in the event matches the open record. |
| EMP API subscription error | Test from a Lightning record page and check browser console. |
| Event publishes but UI unchanged | Confirm `orderStatusPath` is on the Order record page. |

## 29. Apex Tests

Exact test files:

| Test file | Purpose |
| --- | --- |
| `force-app/main/default/classes/TestProductController.cls` | Tests product filtering and similar product logic. |
| `force-app/main/default/classes/TestOrderController.cls` | Tests order item retrieval. |
| `force-app/main/default/classes/CommunitiesLandingControllerTest.cls` | Tests Visualforce landing controller. |

Run all local Apex tests:

```bash
sf apex run test -o scratch-org -l RunLocalTests -r human -w 20
```

Run with coverage:

```bash
sf apex run test -o scratch-org -l RunLocalTests -r human -c -w 20
```

Run one class:

```bash
sf apex run test -o scratch-org -n TestProductController -r human -w 20
```

Why:

Apex tests prove server-side logic works and are required for production deployments.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Test data insert fails | Check required fields in object metadata. |
| Query exception | Confirm test data creates related records before querying. |
| Permission issue during manual testing | Assign permission set; Apex tests run in their own test context. |

## 30. LWC Jest Tests

Exact config files:

| File | Purpose |
| --- | --- |
| `jest.config.js` | Jest configuration. |
| `jest-sa11y-setup.js` | Accessibility test setup. |
| `force-app/test/jest-mocks` | Mocks for Apex, navigation, LMS, and EMP API. |

Run all LWC tests:

```bash
npm run test:unit
```

Run coverage:

```bash
npm run test:unit:coverage
```

Run related test file:

```bash
npm run test:unit -- --findRelatedTests force-app/main/default/lwc/productFilter/__tests__/productFilter.test.js
```

Why:

Jest tests run locally and verify component JavaScript, DOM rendering, events, and mocked Salesforce APIs.

Concept:

LWC Jest does not require a Salesforce org. It is fast and useful during development.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Cannot find module `lightning/*` | Check Jest mocks and `jest.config.js`. |
| Snapshot or DOM expectation fails | Inspect rendered HTML in the test. |
| Tests fail after dependency install | Run `npm install` again and confirm Node version. |

## 31. Optional UTAM UI Tests

Exact files:

| File | Purpose |
| --- | --- |
| `utam.config.js` | Compiles UTAM page objects. |
| `wdio.conf.js` | WebdriverIO browser runner config. |
| `force-app/test/utam/page-explorer.spec.js` | Product Explorer UI test. |
| `force-app/test/utam/utam-helper.js` | Test helper. |

Commands:

```bash
npm run test:ui:compile
npm run test:ui:generate:login
npm run test:ui
```

Why:

UI tests verify real browser behavior after the app is deployed and sample data exists.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Browser cannot start | Install Chrome. |
| Login URL expired | Run `npm run test:ui:generate:login` again. |
| Product Explorer test fails | Confirm metadata deployed and sample data imported. |

## 32. Formatting And Linting

Commands:

```bash
npm run prettier:verify
npm run lint
```

Fix formatting:

```bash
npm run prettier
```

Why:

Consistent formatting and linting keep the codebase easier to review and maintain.

Common errors:

| Error | Fix |
| --- | --- |
| Prettier check fails | Run `npm run prettier`. |
| ESLint error | Read the line number and update the JavaScript. |

## 33. Git Workflow

### 33.1 Create A Branch

```bash
git checkout -b feature/ebikes-learning-change
```

Why:

A branch isolates your work from `main`.

### 33.2 Check Your Changes

```bash
git status
git diff
```

### 33.3 Validate Before Commit

```bash
npm run prettier:verify
npm run lint
npm run test:unit
sf project deploy start -o scratch-org
sf apex run test -o scratch-org -l RunLocalTests -r human -w 20
```

### 33.4 Commit

```bash
git add .
git commit -m "Add ebikes implementation guide"
```

### 33.5 Push

```bash
git push origin feature/ebikes-learning-change
```

Concept:

Git captures a versioned history of your Salesforce metadata and tests.

## 34. CI/CD Pipeline

### 34.1 Existing Pipeline Concept

A Salesforce CI pipeline usually does this:

1. Check out code.
2. Install Node dependencies.
3. Run formatting checks.
4. Run linting.
5. Run LWC Jest tests.
6. Install Salesforce CLI.
7. Authenticate to Dev Hub.
8. Create a scratch org.
9. Deploy metadata.
10. Assign permission sets.
11. Import data.
12. Publish Experience Cloud.
13. Deploy guest profile metadata.
14. Run Apex tests.
15. Delete scratch org.

### 34.2 GitHub Secret

Required secret:

`DEVHUB_SFDX_URL`

Create it locally:

```bash
sf org display -o devhub --verbose
```

Copy the `Sfdx Auth Url`.

Store it in GitHub:

GitHub repository > Settings > Secrets and variables > Actions > New repository secret.

Security rule:

Treat the SFDX auth URL like a password. Do not commit it.

### 34.3 Beginner GitHub Actions Workflow

If creating a simple workflow from scratch, create:

`.github/workflows/salesforce-ci.yml`

Full sample YAML:

```yaml
name: Salesforce CI

on:
  workflow_dispatch:
  pull_request:
  push:
    branches:
      - main

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install Node dependencies
        run: npm ci

      - name: Verify formatting
        run: npm run prettier:verify

      - name: Lint
        run: npm run lint

      - name: Run LWC Jest tests
        run: npm run test:unit:coverage

      - name: Install Salesforce CLI
        run: npm install @salesforce/cli --location=global

      - name: Authenticate Dev Hub
        shell: bash
        run: |
          echo "${{ secrets.DEVHUB_SFDX_URL }}" > DEVHUB_SFDX_URL.txt
          sf org login sfdx-url -f DEVHUB_SFDX_URL.txt -a devhub -d

      - name: Create scratch org
        run: sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 1

      - name: Deploy metadata
        run: sf project deploy start -o scratch-org

      - name: Assign permission set
        run: sf org assign permset -n ebikes -o scratch-org

      - name: Import sample data
        run: sf data import tree -p data/sample-data-plan.json -o scratch-org

      - name: Publish Experience Cloud site
        run: sf community publish -n E-Bikes -o scratch-org

      - name: Deploy guest profile metadata
        run: sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10

      - name: Run Apex tests
        run: sf apex run test -o scratch-org -l RunLocalTests -w 20 -r human

      - name: Delete scratch org
        if: always()
        run: sf org delete scratch -p -o scratch-org
```

YAML concept:

| YAML section | Meaning |
| --- | --- |
| `name` | Workflow display name. |
| `on` | Events that start the workflow. |
| `jobs` | Units of work. |
| `runs-on` | GitHub runner image. |
| `steps` | Ordered commands. |
| `uses` | Reusable GitHub Action. |
| `run` | Shell command. |
| `secrets.DEVHUB_SFDX_URL` | Secure secret value. |

## 35. Azure DevOps Pipeline Alternative

If your team uses Azure DevOps, create:

`azure-pipelines.yml`

Full sample YAML:

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  NODE_VERSION: '22.x'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: $(NODE_VERSION)
    displayName: Install Node.js

  - script: npm ci
    displayName: Install dependencies

  - script: npm run prettier:verify
    displayName: Verify formatting

  - script: npm run lint
    displayName: Lint

  - script: npm run test:unit:coverage
    displayName: Run LWC Jest tests

  - script: npm install @salesforce/cli --location=global
    displayName: Install Salesforce CLI

  - script: |
      echo "$(DEVHUB_SFDX_URL)" > DEVHUB_SFDX_URL.txt
      sf org login sfdx-url -f DEVHUB_SFDX_URL.txt -a devhub -d
    displayName: Authenticate Dev Hub

  - script: sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 1
    displayName: Create scratch org

  - script: sf project deploy start -o scratch-org
    displayName: Deploy metadata

  - script: sf org assign permset -n ebikes -o scratch-org
    displayName: Assign permission set

  - script: sf data import tree -p data/sample-data-plan.json -o scratch-org
    displayName: Import sample data

  - script: sf community publish -n E-Bikes -o scratch-org
    displayName: Publish Experience Cloud site

  - script: sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10
    displayName: Deploy guest profile metadata

  - script: sf apex run test -o scratch-org -l RunLocalTests -w 20 -r human
    displayName: Run Apex tests

  - script: sf org delete scratch -p -o scratch-org
    condition: always()
    displayName: Delete scratch org
```

Azure secret:

Create a pipeline variable named `DEVHUB_SFDX_URL` and mark it secret.

## 36. End-To-End Scratch Org Build

Use this command sequence for a clean install:

```bash
npm install
sf org login web -d -a devhub
sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 30
sf project deploy start -o scratch-org
sf org assign permset -n ebikes -o scratch-org
sf data import tree -p data/sample-data-plan.json -o scratch-org
sf community publish -n E-Bikes -o scratch-org
sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10
sf apex run test -o scratch-org -l RunLocalTests -r human -w 20
sf org open -o scratch-org
```

Manual validation:

1. App Launcher > E-Bikes.
2. Product Explorer displays products.
3. Product filters update product tiles.
4. Selecting a product updates Product Card.
5. Product record page displays Similar Products.
6. Create a Reseller Order.
7. Drag products into Order Builder.
8. Change order quantities.
9. Open Setup > Digital Experiences > All Sites.
10. Open the E-Bikes public site.
11. Product Explorer works for guest users.
12. Create Case page loads.

## 37. Manual UI Configuration Summary

Use these only when building manually or using a Developer Org.

| Area | UI path | Required? |
| --- | --- | --- |
| Dev Hub | Setup > Dev Hub | Required only for scratch org creation. |
| Digital Experiences | Setup > Digital Experiences > Settings | Required for Developer Org deployment. |
| ExperienceBundle | Setup > Experience Management Settings | Required for Developer Org deployment. |
| My Domain | Setup > My Domain | Required for Experience Cloud. |
| Objects | Setup > Object Manager | Optional if deploying metadata. |
| Fields | Setup > Object Manager > Object > Fields & Relationships | Optional if deploying metadata. |
| Permission sets | Setup > Permission Sets | Optional verification after deployment. |
| Lightning pages | Setup > Lightning App Builder | Optional verification after deployment. |
| Sites | Setup > Digital Experiences > All Sites | Required to publish and verify. |
| CSP Trusted Sites | Setup > CSP Trusted Sites | Optional verification after deployment. |

## 38. Troubleshooting

### 38.1 Deployment Problems

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Experience Cloud metadata fails | Org does not have Experiences enabled | Use scratch org definition or enable Digital Experiences manually. |
| Guest profile deployment fails | Site is not published yet | Deploy `force-app`, publish site, then deploy `guest-profile-metadata`. |
| Product object deploy fails due to page reference | Partial deployment missed FlexiPage | Deploy all of `force-app`. |
| Permission set deploy fails | Referenced tabs or objects missing | Deploy app, tabs, objects, and fields first. |
| Case field conflict | Org already has a Case Product field | Inspect Case fields and resolve only in a fresh learning org. |

### 38.2 Runtime Problems

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Product Explorer empty | Sample data missing | Run `sf data import tree -p data/sample-data-plan.json -o scratch-org`. |
| Product images missing | CSP Trusted Site missing | Deploy `cspTrustedSites` or check Setup > CSP Trusted Sites. |
| Product Card does not update | Message channel missing or page composition wrong | Confirm Product Explorer page has `productTileList` and `productCard`. |
| Order Builder cannot save | Missing object or field permissions | Assign `ebikes` permission set. |
| Public site cannot see products | Guest sharing/profile not deployed | Deploy `guest-profile-metadata` after publishing site. |
| Order Status Path does not react | Platform event not published or wrong order Id | Publish `Manufacturing_Event__e` with the correct `Order_Id__c`. |

### 38.3 CLI Problems

| Symptom | Fix |
| --- | --- |
| `No authorization information found` | Run `sf org login web -a devhub` or `sf org login web -a scratch-org`. |
| Alias not found | Run `sf org list` and use the correct alias. |
| Command syntax fails | Update Salesforce CLI with `sf update`. |
| Deployment appears stuck | Use `sf project deploy report -o scratch-org`. |

### 38.4 Test Problems

| Symptom | Fix |
| --- | --- |
| Jest cannot resolve Lightning module | Confirm Jest mocks exist under `force-app/test/jest-mocks`. |
| Apex tests fail with required field missing | Check object field metadata and test data setup. |
| UTAM cannot log in | Regenerate login URL. |
| UI test sees no products | Deploy metadata and import sample data first. |

## 39. Learning Checklist

Use this checklist to confirm understanding.

| Topic | Complete |
| --- | --- |
| Explain what `sfdx-project.json` does | |
| Explain what `project-scratch-def.json` does | |
| Create and open a scratch org | |
| Deploy `force-app` | |
| Assign the `ebikes` permission set | |
| Import sample data | |
| Explain custom objects and custom fields | |
| Explain lookup relationships | |
| Explain master-detail relationships | |
| Explain standard object extension on Case | |
| Explain Platform Events | |
| Explain Apex classes and test classes | |
| Explain SOQL | |
| Explain `with sharing` | |
| Explain `WITH USER_MODE` | |
| Explain `@AuraEnabled(cacheable=true)` | |
| Explain LWC HTML templates | |
| Explain LWC JavaScript imports | |
| Explain LWC meta XML targets | |
| Explain Lightning Message Service | |
| Explain Lightning Data Service | |
| Explain UI Record API | |
| Explain Experience Cloud metadata | |
| Explain guest profile metadata | |
| Run Apex tests | |
| Run LWC Jest tests | |
| Read CI/CD YAML | |

## 40. Final Validation Checklist

| Check | Command or UI path | Expected result |
| --- | --- | --- |
| CLI works | `sf --version` | Version prints. |
| Node works | `node --version` | Version prints. |
| Dependencies installed | `npm install` | Completes successfully. |
| Formatting passes | `npm run prettier:verify` | No formatting errors. |
| Lint passes | `npm run lint` | No lint errors. |
| Jest passes | `npm run test:unit` | LWC tests pass. |
| Scratch org exists | `sf org list` | `scratch-org` appears. |
| Metadata deploys | `sf project deploy start -o scratch-org` | Deployment succeeds. |
| Permission assigned | `sf org assign permset -n ebikes -o scratch-org` | Assignment succeeds. |
| Data imported | `sf data import tree -p data/sample-data-plan.json -o scratch-org` | Import succeeds. |
| Products exist | `sf data query -o scratch-org --query "SELECT Count() FROM Product__c"` | Count is 16. |
| Product Explorer works | App Launcher > E-Bikes > Product Explorer | Products display. |
| LMS works | Select product tile | Product Card updates. |
| Order page works | Open an order record | Order Builder and Status Path load. |
| Experience site exists | Setup > Digital Experiences > All Sites | E-Bikes appears. |
| Site publishes | `sf community publish -n E-Bikes -o scratch-org` | Publish succeeds. |
| Guest metadata deploys | `sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10` | Deployment succeeds. |
| Public products visible | Incognito browser > public site URL | Products display. |
| Apex tests pass | `sf apex run test -o scratch-org -l RunLocalTests -r human -w 20` | Tests pass. |

## 41. Final Notes

The E-Bikes LWC application is a layered Salesforce app:

1. The data model defines the business records.
2. Permission sets and guest profile metadata secure access.
3. Apex provides custom server-side queries.
4. Lightning Data Service handles standard record reads and writes.
5. Lightning Message Service connects unrelated components.
6. Platform Events demonstrate event-driven UI updates.
7. Lightning pages assemble the internal app.
8. Experience Cloud metadata assembles the public site.
9. Tests and CI/CD keep the project repeatable.

There is no OAuth, Connected App, Named Credential, External Credential, API key, token, or third-party registration required for this app. The only external dependency is public Amazon S3 media, configured through CSP Trusted Sites.
