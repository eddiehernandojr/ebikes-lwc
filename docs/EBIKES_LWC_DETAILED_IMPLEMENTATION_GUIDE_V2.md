# E-Bikes LWC Detailed Implementation Guide

## 1. Purpose Of This Guide

This manual explains how to rebuild, deploy, test, and understand the Salesforce E-Bikes Lightning Web Components application as a beginner Salesforce developer.

The guide is based on the existing build and simulation guide at:

`docs/EBIKES_LWC_BUILD_SIMULATION_GUIDE.md`

It was also cross-verified against the actual repository metadata and tooling. When the repository is the source of truth, this guide follows the repository.

Use this guide in two ways:

1. To learn the app step by step by studying each metadata folder.
2. To install the app safely by deploying the existing repository metadata instead of recreating complex Experience Cloud and Lightning page XML by hand.

The safest full installation path is:

```bash
sf org login web -d -a devhub
sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 30
sf project deploy start -o scratch-org
sf org assign permset -n ebikes -o scratch-org
sf data import tree -p data/sample-data-plan.json -o scratch-org
sf community publish -n E-Bikes -o scratch-org
sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10
sf apex run test -o scratch-org -l RunLocalTests -w 20
```

## 2. Source Documents And Codebase Verification

### Verified Source Documents

Primary source document:

`docs/EBIKES_LWC_BUILD_SIMULATION_GUIDE.md`

Repository files verified:

| Area | Verified path |
| --- | --- |
| Salesforce objects | `force-app/main/default/objects` |
| Apex classes | `force-app/main/default/classes` |
| Lightning Web Components | `force-app/main/default/lwc` |
| Aura components | `force-app/main/default/aura` |
| Lightning pages | `force-app/main/default/flexipages` |
| Layouts | `force-app/main/default/layouts` |
| Lightning app | `force-app/main/default/applications` |
| Tabs | `force-app/main/default/tabs` |
| Permission sets | `force-app/main/default/permissionsets` |
| Message channels | `force-app/main/default/messageChannels` |
| Static resources | `force-app/main/default/staticresources` |
| Experience Bundle | `force-app/main/default/experiences` |
| Network metadata | `force-app/main/default/networks` |
| Site metadata | `force-app/main/default/sites` |
| Visualforce pages | `force-app/main/default/pages` |
| Navigation menus | `force-app/main/default/navigationMenus` |
| Network branding | `force-app/main/default/networkBranding` |
| CSP Trusted Sites | `force-app/main/default/cspTrustedSites` |
| Platform Event channel members | `force-app/main/default/platformEventChannelMembers` |
| Guest profile metadata | `guest-profile-metadata` |
| Sample data | `data` |
| Scratch org config | `config/project-scratch-def.json` |
| DX project config | `sfdx-project.json` |
| Node tooling | `package.json`, `package-lock.json` |
| Jest config | `jest.config.js`, `jest-sa11y-setup.js`, `force-app/test/jest-mocks` |
| UTAM and UI test config | `utam.config.js`, `wdio.conf.js`, `force-app/test/utam` |
| CI/CD | `.github/workflows/ci.yml`, `.github/workflows/ci-pr.yml` |

### Verification Notes And Conflicts

No blocking conflicts were found between the existing guide and the codebase.

Additional repository items that must be included beyond the main learning flow:

| Repository item | What the build guide says | What the codebase shows | Version followed here | Why |
| --- | --- | --- | --- | --- |
| Prompts | Not a major focus | `force-app/main/default/prompts` contains walkthrough prompt metadata | Codebase | It is deployable metadata and affects the guided app experience. |
| Content assets | Mentioned with Experience Cloud assets | `force-app/main/default/contentassets` contains `logo` and `ebikeslogo` | Codebase | Experience Cloud pages can reference these assets. |
| CI/CD | Existing guide includes current command patterns | Repo has actual GitHub Actions workflows | Codebase | The real pipeline names, jobs, and secrets should be documented. |
| Change Data Capture | Existing guide mentions CDC learning | Repo has `platformEventChannelMembers/ChangeEvents_Order_ChangeEvent...` | Codebase | This is CDC channel membership metadata, not a custom CDC trigger. |
| External integration/OAuth | Existing guide does not require one | No Named Credential, External Credential, Connected App, Auth Provider, Remote Site Setting, Apex HTTP callout, API key, token, or OAuth flow was found | Codebase | Do not invent integrations. The only external URL dependency is public S3-hosted media. |

## 3. Repository And Application Overview

The app is a Salesforce sample application for a fictional electric bike company.

It has two main user experiences:

| Audience | Experience |
| --- | --- |
| Internal users | A Lightning app named `EBikes` for browsing products, reseller orders, accounts, and product records. |
| Public or community users | An Experience Cloud site named `E-Bikes` for product browsing and case creation. |

The application teaches:

| Concept | Where to study it |
| --- | --- |
| Salesforce DX source format | `sfdx-project.json`, `force-app/main/default` |
| Custom objects and fields | `force-app/main/default/objects` |
| Lookup and master-detail relationships | object field XML files |
| Standard object extension | `force-app/main/default/objects/Case` |
| Permission sets and field security | `force-app/main/default/permissionsets/ebikes.permissionset-meta.xml` |
| Guest user access | `guest-profile-metadata` |
| Apex controllers | `force-app/main/default/classes` |
| LWC components | `force-app/main/default/lwc` |
| Lightning Message Service | `force-app/main/default/messageChannels` |
| Platform Events | `force-app/main/default/objects/Manufacturing_Event__e` |
| Change Data Capture channel membership | `force-app/main/default/platformEventChannelMembers` |
| Lightning App Builder pages | `force-app/main/default/flexipages` |
| Experience Cloud | `force-app/main/default/experiences`, `networks`, `sites` |
| Static media | `force-app/main/default/staticresources`, `contentassets`, `cspTrustedSites` |
| Jest tests | LWC `__tests__` folders and `jest.config.js` |
| UTAM UI tests | `force-app/test/utam`, `utam.config.js`, `wdio.conf.js` |
| CI/CD | `.github/workflows` |

### Application Dependency Map

```text
Account
  |
  +-- Order__c
        |
        +-- Order_Item__c ---- Product__c ---- Product_Family__c

Case ---- Product__c

Manufacturing_Event__e --> orderStatusPath LWC --> Order__c.Status__c
Order__ChangeEvent metadata --> ChangeEvents platform event channel
```

UI dependency map:

```text
ProductController
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

OrderController
  --> orderBuilder
       --> orderItemTile
       --> placeholder
       --> errorPanel

ProductRecordInfoController
  --> heroDetails
       --> hero

orderStatusPath
  --> lightning/empApi
  --> Manufacturing_Event__e
  --> lightning/uiRecordApi updateRecord
```

## 4. Prerequisites

### Local Tools

Install these on your computer:

| Tool | Why it is needed |
| --- | --- |
| Git | Clones the repository and manages branches. |
| VS Code | Edits Salesforce metadata and JavaScript files. |
| Salesforce Extension Pack for VS Code | Adds Salesforce commands and metadata support. |
| Salesforce CLI | Creates orgs, deploys metadata, imports data, and runs tests. |
| Node.js | Runs Jest, ESLint, Prettier, UTAM, and WebdriverIO. |
| Chrome | Runs UTAM/WebdriverIO UI tests. |

The repository declares Node version `22.18.0` under `package.json` `volta`. If you use Volta, it will select that version automatically.

### Salesforce Orgs

You need either:

| Org type | Use it when |
| --- | --- |
| Dev Hub plus scratch org | You want the cleanest Salesforce DX development flow and repeatable CI. |
| Developer Edition org | You want a persistent org and are not practicing scratch org lifecycle. |

For beginners, use scratch orgs first. They are temporary and easy to recreate when you make a mistake.

### Clone And Install

Where: Git and VS Code terminal.

Exact commands:

```bash
git clone https://github.com/trailheadapps/ebikes-lwc
cd ebikes-lwc
npm install
```

Why:

`npm install` installs local developer tools such as Jest, ESLint, Prettier, UTAM, and WebdriverIO.

Verify:

```bash
sf --version
node --version
npm --version
npm run prettier:verify
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| `sf` command not found | Install Salesforce CLI and restart the terminal. |
| Node version errors | Install Node LTS or use Volta with this repository. |
| `npm install` fails | Delete `node_modules`, keep `package-lock.json`, then run `npm install` again. |

## 5. Salesforce DX Project Setup

### Step 5.1: Understand The Project Root

Where: VS Code.

Important files:

| File or folder | Purpose |
| --- | --- |
| `sfdx-project.json` | Tells Salesforce CLI which folder contains source metadata and which API version to use. |
| `force-app/main/default` | Main Salesforce metadata package. |
| `config/project-scratch-def.json` | Defines the scratch org shape. |
| `data` | Sample data import plan and JSON records. |
| `guest-profile-metadata` | Guest profile and sharing rules deployed after the Experience Cloud site exists. |
| `package.json` | Node scripts for formatting, linting, Jest, UTAM, and WebdriverIO. |
| `.github/workflows` | GitHub Actions CI/CD pipelines. |

### Step 5.2: Study `sfdx-project.json`

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

What this teaches:

| Line | Meaning |
| --- | --- |
| `packageDirectories` | Salesforce metadata lives under `force-app`. |
| `"default": true` | New generated metadata goes into this package by default. |
| `"namespace": ""` | This is not a managed package namespace project. |
| `"sourceApiVersion": "65.0"` | Metadata deploys using Salesforce API version 65.0. |

Verify:

```bash
sf project deploy preview -o scratch-org
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| CLI says no project found | Run the command from the repository root where `sfdx-project.json` exists. |
| Metadata API version error | Update Salesforce CLI with `sf update`. |

## 6. Scratch Org And Developer Org Setup

### Step 6.1: Study The Scratch Org Definition

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

Why this is needed:

The app includes Experience Cloud metadata, walkthrough prompts, Lightning pages, and mobile-compatible Lightning Experience settings. A scratch org must enable those features before metadata deployment.

What Salesforce concept it teaches:

Scratch org definitions are source-controlled org shapes. They make org creation repeatable.

### Step 6.2: Authenticate A Dev Hub

Where: Salesforce CLI.

Exact command:

```bash
sf org login web -d -a devhub
```

What happens:

The browser opens. Log in to the org that has Dev Hub enabled.

Manual Setup UI path if Dev Hub is not enabled:

Setup > Dev Hub > Enable Dev Hub.

Verify:

```bash
sf org list
```

You should see an alias named `devhub`.

### Step 6.3: Create A Scratch Org

Where: Salesforce CLI.

Exact command:

```bash
sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 30
```

Why:

This creates a temporary Salesforce org with the features required by this app.

Verify:

```bash
sf org open -o scratch-org
```

### Step 6.4: Delete A Scratch Org

Where: Salesforce CLI.

Exact command:

```bash
sf org delete scratch -p -o scratch-org
```

Why:

Scratch orgs are disposable. Deleting and recreating them is normal in Salesforce DX.

### Step 6.5: Authenticate A Developer Org

Where: Salesforce CLI.

Exact command:

```bash
sf org login web -a dev-org
```

When to use a Developer Org:

Use a Developer Org when you want a persistent learning org and do not need source tracking.

When to use a scratch org:

Use a scratch org for day-to-day development, CI validation, and clean rebuild practice.

Important difference:

| Org type | Best for | Main tradeoff |
| --- | --- | --- |
| Scratch org | Repeatable development and CI | Temporary lifecycle |
| Developer Org | Persistent demo or learning org | Can drift from source over time |

## 7. Deployment Strategy

### Step 7.1: Deploy The Existing Metadata

Where: Salesforce CLI.

Exact command:

```bash
sf project deploy start -o scratch-org
```

Why:

The repository contains many metadata files with cross-references: objects, fields, FlexiPages, Experience Cloud routes, navigation menus, static assets, CSP settings, message channels, Apex, and LWCs. Recreating all of that manually is error-prone.

What it teaches:

Salesforce metadata deployments are dependency-aware, but your org must have required features enabled first.

Verify:

```bash
sf project deploy report -o scratch-org
```

Common errors and fixes:

| Error | Fix |
| --- | --- |
| ExperienceBundle deploy error | Confirm the scratch org definition has `Communities` and `enableExperienceBundleMetadata`. |
| Unknown user permission or feature | Confirm the scratch org was created from `config/project-scratch-def.json`. |
| Component reference not found | Deploy all of `force-app`, not one random file. |

### Step 7.2: Assign Permissions

Exact file:

`force-app/main/default/permissionsets/ebikes.permissionset-meta.xml`

Exact command:

```bash
sf org assign permset -n ebikes -o scratch-org
```

Why:

The default scratch org user needs access to custom objects, fields, tabs, and the `EBikes` app.

Verify:

Setup > Users > Permission Set Assignments.

### Step 7.3: Publish Experience Cloud

Exact command:

```bash
sf community publish -n E-Bikes -o scratch-org
```

Why:

Experience Cloud metadata can deploy before the public site is fully published. Publishing makes the site pages visible.

Verify:

Setup > Digital Experiences > All Sites > E-Bikes > Builder.

### Step 7.4: Deploy Guest Profile Metadata After The Site Exists

Exact folder:

`guest-profile-metadata`

Exact command:

```bash
sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10
```

Why:

Guest profile metadata depends on the Experience Cloud site and its generated guest user/profile context. Deploy this after the main metadata and after publishing the site.

## 8. Data Model Implementation

### Step 8.1: Object Inventory

Where: VS Code and Salesforce Setup UI.

Exact folder:

`force-app/main/default/objects`

| Object | Type | Purpose |
| --- | --- | --- |
| `Product_Family__c` | Custom Object | Groups products into families. |
| `Product__c` | Custom Object | Stores bike catalog records and product specs. |
| `Order__c` | Custom Object | Represents reseller orders. |
| `Order_Item__c` | Custom Object | Represents products and quantities inside an order. |
| `Manufacturing_Event__e` | Platform Event | Streams manufacturing status updates. |
| `Case` | Standard Object extension | Adds E-Bikes support fields to standard Case. |

Manual Setup UI path:

Setup > Object Manager.

Verify:

```bash
sf data query -o scratch-org --query "SELECT QualifiedApiName FROM EntityDefinition WHERE QualifiedApiName IN ('Product__c','Product_Family__c','Order__c','Order_Item__c','Manufacturing_Event__e')"
```

### Step 8.2: Relationships

| Child object | Field | Parent object | Relationship type | File |
| --- | --- | --- | --- | --- |
| `Product__c` | `Product_Family__c` | `Product_Family__c` | Lookup | `objects/Product__c/fields/Product_Family__c.field-meta.xml` |
| `Order__c` | `Account__c` | `Account` | Required lookup | `objects/Order__c/fields/Account__c.field-meta.xml` |
| `Order_Item__c` | `Order__c` | `Order__c` | Master-detail | `objects/Order_Item__c/fields/Order__c.field-meta.xml` |
| `Order_Item__c` | `Product__c` | `Product__c` | Lookup | `objects/Order_Item__c/fields/Product__c.field-meta.xml` |
| `Case` | `Product__c` | `Product__c` | Lookup | `objects/Case/fields/Product__c.field-meta.xml` |

What this teaches:

Lookup relationships connect records loosely. Master-detail relationships create ownership and sharing inheritance. In this app, `Order_Item__c` is controlled by its parent `Order__c`, so its object sharing model is `ControlledByParent`.

### Step 8.3: Object And Field Summary

#### `Product_Family__c`

Exact folder:

`force-app/main/default/objects/Product_Family__c`

Fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Category__c` | Picklist | Family category: `Commuter`, `Hybrid`, `Mountain`. |
| `Description__c` | TextArea | Family description. |

#### `Product__c`

Exact folder:

`force-app/main/default/objects/Product__c`

Fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Battery__c` | Text | Battery spec. |
| `Category__c` | Picklist | `Mountain`, `Commuter`. |
| `Charger__c` | Text | Charger spec. |
| `Description__c` | LongTextArea | Long product description. |
| `Fork__c` | Text | Fork spec. |
| `Frame_Color__c` | Picklist | `white`, `red`, `blue`, `green`. |
| `Front_Brakes__c` | Text | Brake spec. |
| `Handlebar_Color__c` | Picklist | `white`, `red`, `blue`, `green`. |
| `Level__c` | Picklist | `Beginner`, `Enthusiast`, `Racer`. |
| `MSRP__c` | Currency | Retail price. |
| `Material__c` | Picklist | `Aluminum`, `Carbon`. |
| `Motor__c` | Text | Motor spec. |
| `Picture_URL__c` | URL | Public product image URL. |
| `Product_Family__c` | Lookup | Product family relationship. |
| `Rear_Brakes__c` | Text | Brake spec. |
| `Seat_Color__c` | Picklist | `white`, `red`, `blue`, `green`. |
| `Waterbottle_Color__c` | Picklist | `white`, `red`, `blue`, `green`. |

#### `Order__c`

Exact folder:

`force-app/main/default/objects/Order__c`

Fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Account__c` | Required lookup to `Account` | The reseller account placing the order. |
| `Status__c` | Required picklist | `Draft`, `Submitted to Manufacturing`, `Approved by Manufacturing`, `In Production`. |

#### `Order_Item__c`

Exact folder:

`force-app/main/default/objects/Order_Item__c`

Fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Order__c` | Master-detail to `Order__c` | Parent order. |
| `Product__c` | Lookup to `Product__c` | Ordered product. |
| `Price__c` | Currency | Unit price. |
| `Qty_S__c` | Number | Small size quantity. |
| `Qty_M__c` | Number | Medium size quantity. |
| `Qty_L__c` | Number | Large size quantity. |

#### `Case` Standard Object Extension

Exact folder:

`force-app/main/default/objects/Case`

Fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `Case_Category__c` | Picklist | `Mechanical`, `Electrical`, `Electronic`, `Structural`, `Other`. |
| `Product__c` | Lookup to `Product__c` | Product involved in the support case. |

Why this is important:

Most Salesforce apps extend standard objects rather than replacing them. This app extends `Case` to reuse Salesforce Service functionality.

### Step 8.4: List Views

Exact files:

| File | Purpose |
| --- | --- |
| `objects/Product__c/listViews/ProductList.listView-meta.xml` | Product list view. |
| `objects/Product_Family__c/listViews/All_Product_Families.listView-meta.xml` | Product family list view. |
| `objects/Order__c/listViews/All_Orders.listView-meta.xml` | Reseller orders list view. |
| `objects/Case/listViews/My_Cases.listView-meta.xml` | Case list view. |

Manual Setup UI path:

Object Manager > Object > List View Controls.

Verify:

Open the `EBikes` app and check the Products, Product Families, Orders, and Cases tabs.

## 9. Security And Permissions Implementation

### Step 9.1: Permission Set

Exact file:

`force-app/main/default/permissionsets/ebikes.permissionset-meta.xml`

What to configure:

The repository already contains the permission set. Deploy it and assign it.

Exact command:

```bash
sf org assign permset -n ebikes -o scratch-org
```

What it grants:

| Area | Access |
| --- | --- |
| `EBikes` app | Visible |
| `Product__c` and `Product_Family__c` | Create, read, edit, delete, view all, modify all |
| `Order__c` and `Order_Item__c` | Create, read, edit, delete, view all, modify all |
| `Case` | Create, read, edit, delete, view all |
| `Account` | Read and view all |
| Product fields | Read and edit |
| Order item fields | Read and edit |
| Tabs | Visible for Products, Product Families, Orders, Product Explorer |

What Salesforce concept it teaches:

Permission sets grant access without changing a user's profile. This is the preferred modern approach.

Manual Setup UI path:

Setup > Permission Sets > ebikes > Manage Assignments.

Verify:

Open App Launcher > E-Bikes.

### Step 9.2: Field-Level Security

Where:

`force-app/main/default/permissionsets/ebikes.permissionset-meta.xml`

Why:

A user can have object access but still be blocked from individual fields. The permission set contains `<fieldPermissions>` entries for the custom fields used by the app.

Verify in UI:

Setup > Permission Sets > ebikes > Object Settings > Product > Field Permissions.

Common error:

| Error | Cause | Fix |
| --- | --- | --- |
| LWC shows blank values | User lacks field read access | Assign `ebikes` permission set or add missing field permissions. |

### Step 9.3: Guest Profile Metadata

Exact folder:

`guest-profile-metadata`

Files:

| File | Purpose |
| --- | --- |
| `package.xml` | Deployment package for guest metadata. |
| `profiles/E-Bikes Profile.profile` | Guest profile settings. |
| `sharingRules/Product__c.sharingRules` | Guest sharing for products. |
| `sharingRules/Product_Family__c.sharingRules` | Guest sharing for product families. |
| `profilePasswordPolicies/...profilePasswordPolicy` | Guest profile password policy metadata. |
| `profileSessionSettings/...profileSessionSetting` | Guest profile session setting metadata. |

Exact command:

```bash
sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10
```

Why:

Experience Cloud guest users need read access to public catalog records. Guest access is intentionally restrictive, so this metadata is separated and deployed after the site exists.

Manual Setup UI path:

Setup > Digital Experiences > All Sites > E-Bikes > Workspaces > Administration > Pages > Go to Force.com > Public Access Settings.

Verify:

Open the public Experience Cloud site URL in a private browser window and confirm product pages can load.

### Step 9.4: Sharing Rules

Exact files:

| File | Purpose |
| --- | --- |
| `guest-profile-metadata/sharingRules/Product__c.sharingRules` | Shares product records with the site guest user. |
| `guest-profile-metadata/sharingRules/Product_Family__c.sharingRules` | Shares product family records with the site guest user. |

What Salesforce concept it teaches:

Object permissions decide what a user can do. Sharing decides which records the user can see.

## 10. Apex Backend Implementation

### Step 10.1: Apex Inventory

Exact folder:

`force-app/main/default/classes`

| Class | Type | Purpose |
| --- | --- | --- |
| `ProductController.cls` | Apex controller | Queries products and similar products for LWCs. |
| `OrderController.cls` | Apex controller | Queries order items for the order builder. |
| `ProductRecordInfoController.cls` | Apex controller | Finds product or family records by name for hero navigation. |
| `PagedResult.cls` | DTO/wrapper | Wraps paged product query results for LWC. |
| `HeroDetailsPositionCustomPicklist.cls` | Dynamic picklist provider | Provides Experience Builder design options. |
| `CommunitiesLandingController.cls` | Visualforce controller | Redirects the landing page into the community. |
| `TestProductController.cls` | Apex test | Tests product query behavior. |
| `TestOrderController.cls` | Apex test | Tests order query behavior. |
| `CommunitiesLandingControllerTest.cls` | Apex test | Tests Visualforce landing controller. |

### Step 10.2: `with sharing`

Verified classes use `with sharing`, including:

| Class | Declaration |
| --- | --- |
| `ProductController` | `public with sharing class ProductController` |
| `OrderController` | `public with sharing class OrderController` |
| `ProductRecordInfoController` | `public with sharing class ProductRecordInfoController` |
| `PagedResult` | `public with sharing class PagedResult` |
| `CommunitiesLandingController` | `public with sharing class CommunitiesLandingController` |

Why:

`with sharing` respects record sharing rules for the running user.

### Step 10.3: `@AuraEnabled` And `@AuraEnabled(cacheable=true)`

Where:

`ProductController.cls`, `OrderController.cls`, `ProductRecordInfoController.cls`, `PagedResult.cls`

Purpose:

`@AuraEnabled` exposes Apex to Aura and LWC. `@AuraEnabled(Cacheable=true)` allows read-only Apex methods to be called with `@wire` and cached by the client.

Example pattern from this app:

```apex
public with sharing class OrderController {
    @AuraEnabled(Cacheable=true)
    public static List<Order_Item__c> getOrderItems(Id orderId) {
        return [
            SELECT Id, Price__c, Qty_S__c, Qty_M__c, Qty_L__c,
                   Product__r.Id, Product__r.Name, Product__r.MSRP__c,
                   Product__r.Picture_URL__c
            FROM Order_Item__c
            WHERE Order__c = :orderId
            WITH USER_MODE
        ];
    }
}
```

What it teaches:

LWC can call server-side Apex, but read-only wire methods should be cacheable.

### Step 10.4: SOQL And `WITH USER_MODE`

Verified code:

`ProductController`, `OrderController`, and `ProductRecordInfoController` use `WITH USER_MODE`.

Why:

`WITH USER_MODE` makes SOQL enforce object and field permissions for the running user.

Common error:

| Error | Cause | Fix |
| --- | --- | --- |
| Query exception about inaccessible fields | User lacks field or object access | Update permission set or remove the field from the user-mode query. |

### Step 10.5: Dynamic SOQL

Where:

`force-app/main/default/classes/ProductController.cls`

What it does:

`ProductController` builds a product query using filter inputs from `productFilter` and returns a `PagedResult`.

Why:

The Product Explorer supports filtering by search key, price, category, material, and level.

Verify:

```bash
sf apex run test -o scratch-org -n TestProductController -w 10 -r human
```

### Step 10.6: Apex Tests

Exact command:

```bash
sf apex run test -o scratch-org -l RunLocalTests -w 20 -r human
```

What this teaches:

Apex tests create test data, call Apex methods, and assert expected behavior. Salesforce requires Apex tests for production deployment.

Common errors and fixes:

| Error | Fix |
| --- | --- |
| Missing required field | Check object field metadata and create all required test data. |
| FLS error with `WITH USER_MODE` | Assign the right permissions to the test running context or adjust query fields. |

## 11. LWC Frontend Implementation

### Step 11.1: LWC Folder Structure

Exact folder:

`force-app/main/default/lwc`

Each component normally contains:

| File | Purpose |
| --- | --- |
| `componentName.html` | Template markup. |
| `componentName.js` | JavaScript controller. |
| `componentName.css` | Optional component styles. |
| `componentName.js-meta.xml` | Salesforce metadata: exposure, targets, design properties. |
| `__tests__` | Jest unit tests. |
| `__utam__` | UTAM page object definitions if present. |

### Step 11.2: Component Inventory

| Component | Purpose | Key concepts |
| --- | --- | --- |
| `accountMap` | Shows account billing address on a map. | `@api recordId`, `@wire(getRecord)`, `getFieldValue`. |
| `createCase` | Creates a Case in Experience Cloud. | Record form behavior, custom `refresh` event. |
| `errorPanel` | Displays errors. | `@api`, conditional templates. |
| `hero` | Experience Cloud hero media component. | `@api` design properties, static/external media. |
| `heroDetails` | Looks up product/family info for hero navigation. | Apex wire, child component composition. |
| `ldsUtils` | Shared error utility. | Reusable JavaScript module. |
| `orderBuilder` | Drag products into an order and edit quantities. | Apex wire, UI Record API, drag-and-drop, custom events. |
| `orderItemTile` | Child tile for order item editing. | `@api`, child-to-parent events. |
| `orderStatusPath` | Displays and updates order status. | `lightning/empApi`, UI Record API, platform events. |
| `paginator` | Previous/next controls. | Parent-to-child props, child-to-parent events. |
| `placeholder` | Empty state component. | Static resources. |
| `productCard` | Shows selected product details. | Lightning Message Service, NavigationMixin, LDS. |
| `productFilter` | Publishes product filter changes. | `@wire(getPicklistValues)`, LMS publish. |
| `productListItem` | Compact product row. | `@api`, NavigationMixin. |
| `productTile` | Product card with drag support. | `@api`, custom event, drag event. |
| `productTileList` | Product grid and product selection publisher. | Apex wire, LMS subscribe and publish. |
| `similarProducts` | Related product recommendations. | `@wire(getRecord)`, Apex wire. |

### Step 11.3: LWC HTML, JavaScript, And Meta XML

Study this pattern:

| File | What to look for |
| --- | --- |
| `productTile/productTile.html` | Template renders the product image and clickable tile. |
| `productTile/productTile.js` | Uses `@api`, dispatches `selected`, handles drag start. |
| `productTile/productTile.js-meta.xml` | Exposes the component for Salesforce targets if needed. |

Important concepts:

| Concept | In this repository |
| --- | --- |
| `@api` | Present in many components for public properties such as `recordId`, `product`, `draggable`, and Experience Builder design inputs. |
| `@track` | Not present in the current LWC JavaScript files. Modern LWC usually tracks plain object/array assignment without explicit `@track`. |
| `@wire` | Used for Apex, LDS, MessageContext, object info, picklist values, and records. |
| Lightning Data Service | Used through `lightning/uiRecordApi` and record-aware components. |
| UI Record API | `getRecord`, `getFieldValue`, `createRecord`, `updateRecord`. |
| NavigationMixin | Used by `productCard` and `productListItem`. |
| Custom events | Used by `paginator`, `productTile`, `orderItemTile`, and `createCase`. |

### Step 11.4: Parent-To-Child Communication

Example:

`productTileList` passes product data and drag settings into `productTile`.

Concept:

A parent passes data down through public `@api` properties.

Verify:

Open Product Explorer. Product tiles should display product names, images, and prices.

### Step 11.5: Child-To-Parent Communication

Examples:

| Child | Event | Parent |
| --- | --- | --- |
| `paginator` | `previous`, `next` | `productTileList` |
| `productTile` | `selected` | `productTileList` |
| `orderItemTile` | `orderitemchange`, `orderitemdelete` | `orderBuilder` |

Concept:

Children dispatch DOM events. Parents listen and update state.

### Step 11.6: Drag-And-Drop Behavior

Verified files:

| File | Behavior |
| --- | --- |
| `lwc/productTile/productTile.js` | Sets drag data for a product. |
| `lwc/orderBuilder/orderBuilder.js` | Handles drop and creates `Order_Item__c`. |
| `lwc/productTileList/productTileList.js-meta.xml` | Has a design property to make tiles draggable. |

What happens:

1. The Product Tile starts a drag operation.
2. The Order Builder receives the drop.
3. `orderBuilder` creates an `Order_Item__c` using `createRecord`.
4. The UI refreshes order line items.

Common errors:

| Error | Fix |
| --- | --- |
| Drop does nothing | Confirm the Product Tile List on the Order Record Page has draggable tiles enabled. |
| Create fails | Confirm the user has create access to `Order_Item__c` and read access to product fields. |

## 12. Lightning Message Service Implementation

### Step 12.1: Message Channels

Exact folder:

`force-app/main/default/messageChannels`

Files:

| File | Purpose |
| --- | --- |
| `ProductsFiltered.messageChannel-meta.xml` | Sends filter criteria from `productFilter` to `productTileList`. |
| `ProductSelected.messageChannel-meta.xml` | Sends selected product ID from `productTileList` to `productCard`. |

### Step 12.2: ProductsFiltered Flow

```text
productFilter
  publishes ProductsFiltered
productTileList
  subscribes ProductsFiltered
  calls ProductController.getProducts
```

Where:

| Component | File |
| --- | --- |
| Publisher | `lwc/productFilter/productFilter.js` |
| Subscriber | `lwc/productTileList/productTileList.js` |

### Step 12.3: ProductSelected Flow

```text
productTile
  dispatches selected event
productTileList
  publishes ProductSelected
productCard
  subscribes ProductSelected
```

Where:

| Component | File |
| --- | --- |
| DOM event child | `lwc/productTile/productTile.js` |
| LMS publisher | `lwc/productTileList/productTileList.js` |
| LMS subscriber | `lwc/productCard/productCard.js` |

Why this step is needed:

Lightning Message Service lets sibling components communicate without a direct parent-child relationship.

Verify:

Open Product Explorer. Select a tile. The Product Card should update.

Common errors:

| Error | Fix |
| --- | --- |
| Product Card does not update | Check that both components are on the same Lightning page and the message channel metadata deployed. |
| Jest message service import fails | Confirm `force-app/test/jest-mocks/lightning/messageService.js` exists and Jest config maps mocks correctly. |

## 13. Platform Event Implementation

### Step 13.1: Platform Event Object

Exact folder:

`force-app/main/default/objects/Manufacturing_Event__e`

Object metadata summary:

| Setting | Value |
| --- | --- |
| Label | Manufacturing Event |
| API name | `Manufacturing_Event__e` |
| Event type | HighVolume |
| Publish behavior | PublishAfterCommit |

Fields:

| Field | Type | Required | Purpose |
| --- | --- | --- | --- |
| `Order_Id__c` | Text | true | Identifies the order to update. |
| `Status__c` | Text | true | New manufacturing status. |

What it teaches:

Platform Events are event messages. They are not normal records for UI editing. They are used to publish and subscribe to event notifications.

### Step 13.2: LWC Subscription

Exact file:

`force-app/main/default/lwc/orderStatusPath/orderStatusPath.js`

Verified imports:

```js
import {
    subscribe,
    unsubscribe,
    onError
} from 'lightning/empApi';
```

What happens:

1. `orderStatusPath` subscribes to the Manufacturing Event channel.
2. When a message arrives, it checks whether the event is for the current order.
3. It updates `Order__c.Status__c` using `updateRecord`.

Verify manually:

Use Anonymous Apex:

```apex
Manufacturing_Event__e event = new Manufacturing_Event__e(
    Order_Id__c = 'REPLACE_WITH_ORDER_ID',
    Status__c = 'In Production'
);
EventBus.publish(event);
```

Then open the related order record and confirm the status updates.

### Step 13.3: Change Data Capture Metadata

Exact file:

`force-app/main/default/platformEventChannelMembers/ChangeEvents_Order_ChangeEvent.platformEventChannelMember-meta.xml`

Full content:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<PlatformEventChannelMember xmlns="http://soap.sforce.com/2006/04/metadata">
    <eventChannel>ChangeEvents</eventChannel>
    <selectedEntity>Order__ChangeEvent</selectedEntity>
</PlatformEventChannelMember>
```

What this means:

The repository includes metadata that adds `Order__ChangeEvent` to the standard `ChangeEvents` channel. This supports Change Data Capture-style event membership for `Order__c`.

Important accuracy note:

This repository does not contain an Apex trigger or LWC subscriber that directly consumes `Order__ChangeEvent`. The app's visible order status event behavior uses `Manufacturing_Event__e` and `lightning/empApi`.

## 14. Lightning App, Tabs, Layouts, And Pages

### Step 14.1: Lightning Application

Exact file:

`force-app/main/default/applications/EBikes.app-meta.xml`

Purpose:

Defines the internal Lightning app named `EBikes`.

Manual Setup UI path:

Setup > App Manager > E-Bikes.

Verify:

App Launcher > E-Bikes.

### Step 14.2: Tabs

Exact folder:

`force-app/main/default/tabs`

Files:

| File | Purpose |
| --- | --- |
| `Product__c.tab-meta.xml` | Product object tab. |
| `Product_Family__c.tab-meta.xml` | Product Family object tab. |
| `Order__c.tab-meta.xml` | Reseller Order object tab. |
| `Product_Explorer.tab-meta.xml` | Lightning page tab for Product Explorer. |

Manual Setup UI path:

Setup > Tabs.

### Step 14.3: Layouts

Exact folder:

`force-app/main/default/layouts`

Files:

| File | Purpose |
| --- | --- |
| `Product__c-Product Layout.layout-meta.xml` | Product page layout. |
| `Product_Family__c-Product Family Layout.layout-meta.xml` | Product Family page layout. |
| `Order__c-Order Layout.layout-meta.xml` | Reseller Order page layout. |
| `Order_Item__c-Order Item Layout.layout-meta.xml` | Order Item page layout. |
| `Case-Case Layout.layout-meta.xml` | Case page layout. |
| `CaseClose-Close Case Layout.layout-meta.xml` | Case close layout. |

Manual Setup UI path:

Object Manager > Object > Page Layouts.

### Step 14.4: Lightning App Builder Pages

Exact folder:

`force-app/main/default/flexipages`

| Page | Type | Important components |
| --- | --- | --- |
| `Product_Explorer.flexipage-meta.xml` | App Page | `productFilter`, `productTileList`, `productCard`. |
| `Product_Record_Page.flexipage-meta.xml` | Record Page for `Product__c` | record details, related lists, `similarProducts`. |
| `Order_Record_Page.flexipage-meta.xml` | Record Page for `Order__c` | `orderBuilder`, `orderStatusPath`, draggable `productTileList`. |
| `Account_Record_Page.flexipage-meta.xml` | Record Page for `Account` | Account detail tabs and `accountMap`. |

Manual Setup UI path:

Setup > Lightning App Builder.

Why:

FlexiPages connect LWC components to actual Salesforce pages.

Verify:

Open Product Explorer and an Order record from the E-Bikes app.

Common errors:

| Error | Fix |
| --- | --- |
| Component not visible in Lightning App Builder | Check `component.js-meta.xml` has `<isExposed>true</isExposed>` and the right target. |
| Record page not active | Activate the FlexiPage in Lightning App Builder or deploy the object action override metadata. |

## 15. Experience Cloud Implementation

### Step 15.1: Experience Cloud Metadata Inventory

| Area | Path |
| --- | --- |
| Experience Bundle | `force-app/main/default/experiences/E_Bikes1` |
| Network | `force-app/main/default/networks/E-Bikes.network-meta.xml` |
| Site | `force-app/main/default/sites/E_Bikes.site-meta.xml` |
| Navigation menus | `force-app/main/default/navigationMenus` |
| Network branding | `force-app/main/default/networkBranding` |
| Aura page template | `force-app/main/default/aura/pageTemplate_2_7_3` |
| Visualforce landing page | `force-app/main/default/pages/CommunitiesLanding.page` |
| Audience | `force-app/main/default/audience/Default_E-Bikes.audience-meta.xml` |

### Step 15.2: Deploy Experience Cloud Metadata

Where: Salesforce CLI.

Exact command:

```bash
sf project deploy start -o scratch-org
```

Then publish:

```bash
sf community publish -n E-Bikes -o scratch-org
```

Manual Setup UI path:

Setup > Digital Experiences > All Sites > E-Bikes > Builder.

Why:

Experience Cloud metadata is dependency-heavy. Beginners should deploy the repository metadata, then study it in Experience Builder.

### Step 15.3: Visualforce Page

Exact files:

| File | Purpose |
| --- | --- |
| `force-app/main/default/pages/CommunitiesLanding.page` | Visualforce landing page. |
| `force-app/main/default/pages/CommunitiesLanding.page-meta.xml` | Visualforce page metadata. |
| `force-app/main/default/classes/CommunitiesLandingController.cls` | Controller for landing redirect behavior. |
| `force-app/main/default/classes/CommunitiesLandingControllerTest.cls` | Apex test. |

What this teaches:

Older Salesforce UI technologies can coexist with LWC. This app uses Visualforce for a community landing page pattern.

### Step 15.4: Aura Component

Exact folder:

`force-app/main/default/aura/pageTemplate_2_7_3`

Files:

| File | Purpose |
| --- | --- |
| `pageTemplate_2_7_3.cmp` | Aura page template component. |
| `pageTemplate_2_7_3.design` | Design-time configuration. |
| `pageTemplate_2_7_3.svg` | Template icon. |
| `pageTemplate_2_7_3.cmp-meta.xml` | Aura bundle metadata. |

What this teaches:

Experience Cloud page templates can be Aura components even when the app's main custom UI is LWC.

## 16. Static Resources, Content Assets, And CSP Trusted Sites

### Step 16.1: Static Resources

Exact folder:

`force-app/main/default/staticresources`

Static resource:

| File | Purpose |
| --- | --- |
| `bike_assets.resource-meta.xml` | Metadata for the static resource folder. |
| `bike_assets/logo.svg` | E-Bikes logo. |
| `bike_assets/commuter.png` | Product/family image. |
| `bike_assets/enthusiast.png` | Product/family image. |
| `bike_assets/racer.png` | Product/family image. |
| `bike_assets/CyclingGrass.jpg` | Hero/background image. |

Why:

Static resources package images and assets with your Salesforce metadata.

### Step 16.2: Content Assets

Exact folder:

`force-app/main/default/contentassets`

Files:

| File | Purpose |
| --- | --- |
| `logo.asset` and `logo.asset-meta.xml` | Content asset for the site. |
| `ebikeslogo.asset` and `ebikeslogo.asset-meta.xml` | Content asset for the site. |

Why:

Content assets are used by Experience Cloud and branding features.

### Step 16.3: CSP Trusted Site

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

Sample product records use public Amazon S3 image URLs in `Picture_URL__c`, and the Experience Cloud home page references public S3 media. Salesforce CSP must trust that domain for images and media.

### Step 16.4: Integration And OAuth Verification

Codebase finding:

This app does not require an external integration, OAuth 2.0 flow, Named Credential, External Credential, Connected App, API key, bearer token, custom remote site setting, or Apex HTTP callout.

What exists:

| External dependency | Type | Authentication |
| --- | --- | --- |
| `https://s3-us-west-2.amazonaws.com` | Public image/video media host | No OAuth, no API key, no token. |

What to configure:

Deploy the CSP Trusted Site metadata. Do not create any Named Credential or Connected App for this app.

Manual Setup UI path:

Setup > CSP Trusted Sites.

Verify:

Open Product Explorer and the Experience Cloud home page. Product images and hero media should load.

## 17. Sample Data Import

### Step 17.1: Sample Data Files

Exact folder:

`data`

Files:

| File | Purpose |
| --- | --- |
| `Accounts.json` | Sample reseller accounts. |
| `Product_Family__cs.json` | Product family records. |
| `Product__cs.json` | Product records with references to families. |
| `sample-data-plan.json` | Import plan and dependency order. |

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

Products reference product families. The import plan saves product family references first, then resolves them when importing products.

### Step 17.2: Import Data

Exact command:

```bash
sf data import tree -p data/sample-data-plan.json -o scratch-org
```

Verify:

```bash
sf data query -o scratch-org --query "SELECT Count() FROM Product__c"
sf data query -o scratch-org --query "SELECT Count() FROM Product_Family__c"
sf data query -o scratch-org --query "SELECT Count() FROM Account"
```

Common errors:

| Error | Fix |
| --- | --- |
| Invalid field | Deploy metadata before importing data. |
| Reference not found | Use the plan file, not individual JSON files out of order. |
| Product images do not load | Confirm CSP Trusted Site metadata deployed. |

## 18. Testing Strategy

### Step 18.1: Apex Tests

Exact files:

| Test file | Tests |
| --- | --- |
| `classes/TestProductController.cls` | Product filtering and similar product Apex logic. |
| `classes/TestOrderController.cls` | Order item Apex query logic. |
| `classes/CommunitiesLandingControllerTest.cls` | Visualforce landing controller. |

Exact command:

```bash
sf apex run test -o scratch-org -l RunLocalTests -w 20 -r human
```

CI command used by repository:

```bash
sf apex test run -c -r human -d ./tests/apex -w 20
```

Note:

The current repository command writes test results to `./tests/apex`. The folder may be generated by the command.

### Step 18.2: LWC Jest Tests

Exact config files:

| File | Purpose |
| --- | --- |
| `jest.config.js` | Jest configuration. |
| `jest-sa11y-setup.js` | Accessibility testing setup. |
| `force-app/test/jest-mocks` | Custom Jest mocks for Apex, navigation, message service, and EMP API. |

Exact commands:

```bash
npm run test:unit
npm run test:unit:coverage
```

Jest tests are located in LWC `__tests__` folders.

Examples:

| Component | Test file |
| --- | --- |
| `productTileList` | `lwc/productTileList/__tests__/productTileList.test.js` |
| `productFilter` | `lwc/productFilter/__tests__/productFilter.test.js` |
| `orderBuilder` | `lwc/orderBuilder/__tests__/orderBuilder.test.js` |
| `orderStatusPath` | `lwc/orderStatusPath/__tests__/orderStatusPath.test.js` |
| `hero` | `lwc/hero/__tests__/hero.test.js` |

Why:

LWC Jest tests run locally without a Salesforce org. They verify JavaScript behavior, DOM rendering, and component events.

### Step 18.3: UTAM And UI Tests

Exact files:

| File | Purpose |
| --- | --- |
| `utam.config.js` | Compiles UTAM page objects from `__utam__` JSON. |
| `wdio.conf.js` | WebdriverIO browser test configuration. |
| `force-app/test/utam/page-explorer.spec.js` | UI test spec. |
| `force-app/test/utam/utam-helper.js` | Test helper. |

Commands:

```bash
npm run test:ui:compile
npm run test:ui:generate:login
npm run test:ui
```

Important:

UI tests need an org URL and browser session. Run these only after deployment and sample data import.

### Step 18.4: Formatting And Linting

Commands:

```bash
npm run prettier:verify
npm run lint
```

Fix formatting:

```bash
npm run prettier
```

## 19. CI/CD Pipeline Setup

### Step 19.1: Existing CI/CD Files

Exact folder:

`.github/workflows`

Files:

| File | Purpose |
| --- | --- |
| `ci.yml` | Runs on pushes to `main` and manual workflow dispatch. |
| `ci-pr.yml` | Runs on pull requests and manual workflow dispatch. |
| `auto-assign.yml` | Repository automation. |
| `codetour-watch.yml` | Repository automation. |
| `new-issue-welcome.yml` | Repository automation. |

### Step 19.2: Main CI Workflow

Exact file:

`.github/workflows/ci.yml`

What it does:

1. Checks out source.
2. Installs Volta.
3. Restores or installs Node dependencies.
4. Runs Prettier verification.
5. Runs ESLint.
6. Runs LWC Jest coverage.
7. Uploads LWC coverage to Codecov.
8. Installs Salesforce CLI.
9. Authenticates to Dev Hub using `DEVHUB_SFDX_URL`.
10. Creates a scratch org.
11. Deploys source.
12. Assigns `ebikes` permission set.
13. Imports sample data.
14. Waits for Experience Cloud activation.
15. Publishes the Experience Cloud site.
16. Deploys guest profile metadata.
17. Runs Apex tests.
18. Deletes the scratch org.

Important secrets:

| Secret | Required | Purpose |
| --- | --- | --- |
| `DEVHUB_SFDX_URL` | Yes for scratch org CI | Authenticates GitHub Actions to the Dev Hub. |
| `CODECOV_TOKEN` | Optional unless using Codecov upload | Uploads coverage reports. |

### Step 19.3: Pull Request CI Workflow

Exact file:

`.github/workflows/ci-pr.yml`

What it adds:

| Feature | Meaning |
| --- | --- |
| Pull request trigger | Runs validation before merging changes. |
| Prerelease input | Can create preview-release scratch orgs for prerelease branches. |
| Dependabot auto-merge job | Can approve and auto-merge certain safe dependency updates. |

### Step 19.4: How To Create `DEVHUB_SFDX_URL`

Where: local terminal.

Command:

```bash
sf org display -o devhub --verbose
```

Find the `Sfdx Auth Url` value.

GitHub UI path:

GitHub repository > Settings > Secrets and variables > Actions > New repository secret.

Secret name:

`DEVHUB_SFDX_URL`

Secret value:

Paste the full SFDX auth URL.

Security warning:

Treat the SFDX auth URL like a password. Do not commit it.

### Step 19.5: Beginner-Friendly CI Template

The repository already contains CI files. If you were creating a simpler starter workflow from scratch, create:

`.github/workflows/salesforce-ci.yml`

Sample content:

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

Do not add this sample file unless you want to replace or supplement the repository's existing workflows.

## 20. Git Workflow

### Step 20.1: Create A Branch

Where: Git terminal.

```bash
git checkout -b feature/my-ebikes-change
```

Why:

Branches isolate your work until it is reviewed.

### Step 20.2: Make A Change

Where: VS Code.

Example changes:

| Change | Typical path |
| --- | --- |
| New field | `force-app/main/default/objects/Object__c/fields` |
| LWC update | `force-app/main/default/lwc/componentName` |
| Apex update | `force-app/main/default/classes` |
| Permission update | `force-app/main/default/permissionsets` |

### Step 20.3: Validate Locally

Commands:

```bash
npm run prettier:verify
npm run lint
npm run test:unit
sf project deploy start -o scratch-org
sf apex run test -o scratch-org -l RunLocalTests -w 20 -r human
```

### Step 20.4: Commit And Push

```bash
git status
git add .
git commit -m "Describe the ebikes change"
git push origin feature/my-ebikes-change
```

### Step 20.5: Open A Pull Request

Where:

GitHub repository > Pull requests > New pull request.

What CI should do:

The `.github/workflows/ci-pr.yml` workflow should run formatting, linting, Jest tests, scratch org deployment, sample data import, site publishing, guest profile deployment, and Apex tests.

## 21. Full End-To-End Build Sequence

Use this sequence for a clean scratch org installation.

### Step 21.1: Install Dependencies

```bash
npm install
```

Verify:

```bash
npm run prettier:verify
```

### Step 21.2: Authenticate Dev Hub

```bash
sf org login web -d -a devhub
```

Verify:

```bash
sf org list
```

### Step 21.3: Create Scratch Org

```bash
sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 30
```

### Step 21.4: Deploy Metadata

```bash
sf project deploy start -o scratch-org
```

### Step 21.5: Assign Permission Set

```bash
sf org assign permset -n ebikes -o scratch-org
```

### Step 21.6: Import Sample Data

```bash
sf data import tree -p data/sample-data-plan.json -o scratch-org
```

### Step 21.7: Publish Experience Cloud

```bash
sf community publish -n E-Bikes -o scratch-org
```

If publishing immediately fails because the site is still activating, wait two minutes and retry.

### Step 21.8: Deploy Guest Profile Metadata

```bash
sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10
```

### Step 21.9: Open The Org

```bash
sf org open -o scratch-org
```

Manual checks:

1. App Launcher > E-Bikes.
2. Open Product Explorer.
3. Select a product tile and verify Product Card updates.
4. Open a Product record and verify Similar Products.
5. Open an Order record and verify Order Builder and Order Status Path.
6. Setup > Digital Experiences > All Sites > E-Bikes > Builder.

### Step 21.10: Run Tests

```bash
npm run test:unit
sf apex run test -o scratch-org -l RunLocalTests -w 20 -r human
```

Optional UI tests:

```bash
npm run test:ui:compile
npm run test:ui:generate:login
npm run test:ui
```

## 22. Troubleshooting

### Deployment Errors

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Experience Cloud metadata fails | Scratch org missing Communities or ExperienceBundle settings | Recreate org from `config/project-scratch-def.json`. |
| Guest profile deploy fails | Site not created or not published | Deploy `force-app`, publish site, then deploy `guest-profile-metadata`. |
| LWC target error | Component metadata target mismatch | Check `*.js-meta.xml` targets. |
| Field not found | Object metadata not deployed | Deploy full `force-app` before data or page metadata. |

### Runtime Errors

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Product images missing | CSP Trusted Site missing or S3 URL blocked | Deploy `cspTrustedSites` and check browser console. |
| Product Card does not change | LMS channel missing or components not on same page | Deploy message channels and check Product Explorer FlexiPage. |
| Order Builder cannot create items | Missing object or field permissions | Assign `ebikes` permission set. |
| Order status does not update from event | Event not published or wrong order ID | Publish `Manufacturing_Event__e` with the exact `Order__c` Id. |
| Public site cannot see products | Guest sharing/profile metadata missing | Deploy `guest-profile-metadata` after site publication. |

### Test Errors

| Symptom | Fix |
| --- | --- |
| Jest cannot resolve Lightning module | Check `jest.config.js` and `force-app/test/jest-mocks`. |
| UTAM cannot find browser | Install Chrome and confirm WebdriverIO can start it. |
| Apex test data fails | Review required fields in object XML. |

### CLI Errors

| Symptom | Fix |
| --- | --- |
| `No authorization information found` | Run `sf org login web -a devhub` or recreate the scratch org alias. |
| `The org cannot be found` | Run `sf org list` and use the correct alias. |
| Command syntax mismatch | Update Salesforce CLI with `sf update`. |

## 23. Learning Checklist

Use this checklist to confirm you understand the implementation.

| Topic | Can you explain it? |
| --- | --- |
| Salesforce DX project structure | |
| `sfdx-project.json` | |
| Scratch org definition JSON | |
| Scratch org create/open/delete commands | |
| Developer Org vs scratch org | |
| Custom objects | |
| Custom fields | |
| Lookup relationships | |
| Master-detail relationships | |
| Standard `Case` extension | |
| Platform Events | |
| Change Data Capture channel membership | |
| Permission sets | |
| Field-level security | |
| Guest profile metadata | |
| Sharing rules | |
| Apex classes | |
| Apex test classes | |
| SOQL queries | |
| `with sharing` | |
| `WITH USER_MODE` | |
| `@AuraEnabled` | |
| `@AuraEnabled(cacheable=true)` | |
| LWC folder structure | |
| LWC HTML, JavaScript, meta XML | |
| `@api` | |
| Why `@track` is not used here | |
| `@wire` | |
| Lightning Data Service | |
| UI Record API | |
| Lightning Message Service | |
| Custom events | |
| Parent-to-child communication | |
| Child-to-parent communication | |
| Drag-and-drop behavior | |
| NavigationMixin | |
| Experience Cloud metadata | |
| Visualforce pages | |
| Aura components | |
| Static resources | |
| Content assets | |
| CSP Trusted Sites | |
| Lightning App Builder pages | |
| Tabs | |
| Lightning applications | |
| Layouts | |
| List views | |
| Sample data import | |
| Data import plan JSON | |
| Apex tests | |
| LWC Jest tests | |
| UTAM/UI tests | |
| Git workflow | |
| CI/CD pipeline YAML | |
| Deployment order | |
| Troubleshooting | |

## 24. Final Validation Checklist

Before calling the build complete, verify all of this:

| Check | Command or UI path | Expected result |
| --- | --- | --- |
| Salesforce CLI works | `sf --version` | Version prints successfully. |
| Node dependencies installed | `npm install` | Completes successfully. |
| Formatting passes | `npm run prettier:verify` | No formatting errors. |
| Lint passes | `npm run lint` | No lint errors. |
| Jest passes | `npm run test:unit` | LWC tests pass. |
| Scratch org exists | `sf org list` | `scratch-org` appears. |
| Metadata deploys | `sf project deploy start -o scratch-org` | Deployment succeeds. |
| Permission set assigned | `sf org assign permset -n ebikes -o scratch-org` | Assignment succeeds. |
| Sample data imports | `sf data import tree -p data/sample-data-plan.json -o scratch-org` | Import succeeds. |
| Products exist | `sf data query -o scratch-org --query "SELECT Count() FROM Product__c"` | Count is greater than zero. |
| Product Explorer works | App Launcher > E-Bikes > Product Explorer | Products display and filters work. |
| LMS works | Select product tile | Product Card updates. |
| Product record page works | Open Product record | Similar Products displays. |
| Order page works | Open Order record | Order Builder and Order Status Path display. |
| Experience site publishes | `sf community publish -n E-Bikes -o scratch-org` | Publish succeeds. |
| Guest metadata deploys | `sf project deploy start --metadata-dir guest-profile-metadata -o scratch-org -w 10` | Deployment succeeds. |
| Apex tests pass | `sf apex run test -o scratch-org -l RunLocalTests -w 20 -r human` | Tests pass. |
| CI secrets configured | GitHub > Settings > Secrets and variables > Actions | `DEVHUB_SFDX_URL` exists. |

Final accuracy statement:

The repository does not contain OAuth setup, Named Credentials, External Credentials, Connected Apps, custom API keys, bearer tokens, or Apex callouts. The only external runtime dependency found is public Amazon S3 media, handled by the CSP Trusted Site metadata.
