# E-Bikes LWC Detailed Implementation Guide

## 1. Purpose Of This Guide

This guide explains how to rebuild, configure, deploy, test, and understand the Salesforce E-Bikes LWC sample application as a beginner Salesforce developer.

The safest way to install this repository is to deploy the existing metadata. The best way to learn it is to study and rebuild it in dependency order: org setup, data model, security, Apex, Lightning Web Components, Lightning pages, Experience Cloud, data, tests, and CI/CD.

Do not manually recreate every Experience Builder JSON file, FlexiPage activation, or permission set XML unless your goal is metadata authoring practice. Those files are dependency-heavy exports. In a real project, you study them, make careful edits in VS Code or Setup, retrieve/deploy with Salesforce DX, and verify in the org.

## 2. Source Documents And Codebase Verification

This manual was created from:

| Source | Verified finding |
|---|---|
| `docs/e-bikes-lwc-build-simulation-guide.md` | Existing learning guide and deployment sequence |
| `README.md` | Official scratch org and Developer Edition / Trailhead Playground install steps |
| `CONTRIBUTION.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md` | Repository workflow, contribution, and support context |
| `.github/workflows/*.yml` | Existing CI, PR CI, issue automation, and CodeTour workflows |
| `sfdx-project.json` | `force-app` is the only package directory; API version is `65.0` |
| `config/project-scratch-def.json` | Scratch org uses Developer edition with Communities, Walkthroughs, and API password setup features |
| `package.json` | Actual npm scripts for linting, Prettier, Jest, UTAM, and login generation |
| `force-app/main/default/**` | Salesforce metadata inventory |
| `guest-profile-metadata/**` | Guest profile, session/password policy, and guest sharing rule deployment package |
| `data/**` | Sample Accounts, Product Families, Products, and import plan |
| `scripts/generate-login-url.js` | UTAM helper that generates `.env` login data |

### Reconciled differences

| Topic | Existing guide / README says | Codebase shows | This guide follows |
|---|---|---|---|
| Walkthroughs permission set | README says assign `Walkthroughs` after assigning `ebikes` in scratch org installation. | `force-app/main/default/permissionsets` contains `ebikes.permissionset-meta.xml` and `sfdcInternalInt__sfdc_scrt2.permissionset-meta.xml`; no local `Walkthroughs.permissionset-meta.xml` exists. | Assign `ebikes`. Try `Walkthroughs` only if your org provides it; do not treat it as repository metadata. |
| Data import command | README uses `sf data tree import -p ./data/sample-data-plan.json`. | Existing guide uses modern `sf data import tree --plan data/sample-data-plan.json`. | Both are documented; prefer the modern command, and recognize the README command as the repo's historical style. |
| CI/CD files | Existing guide treats CI/CD mostly as a learning topic. | `.github/workflows/ci.yml` and `ci-pr.yml` already exist. | Explain the existing workflows and include a sample simplified YAML as documentation only. |
| Developer org site metadata | README requires editing `force-app/main/default/sites/E_Bikes.site-meta.xml`. | The checked-in file has sample `siteAdmin`, `siteGuestRecordDefaultOwner`, and `subdomain` values from another org. | Explain exactly why those values must match your target org for Developer Edition / Trailhead Playground installs. |

## 3. Repository And Application Overview

E-Bikes is a fictional electric bicycle manufacturer app. Internal users manage products and reseller orders in a Lightning app. Public users browse products and create support Cases in an Experience Cloud site.

| Area | Path | Purpose |
|---|---|---|
| Salesforce source | `force-app/main/default` | Metadata deployed to Salesforce |
| Data model | `force-app/main/default/objects` | Custom objects, fields, list views, Case extensions, platform event |
| Apex | `force-app/main/default/classes` | Controllers and tests |
| LWC | `force-app/main/default/lwc` | UI components and Jest tests |
| Aura | `force-app/main/default/aura/pageTemplate_2_7_3` | Custom three-region FlexiPage template |
| Lightning pages | `force-app/main/default/flexipages` | Product Explorer, Product, Order, and Account pages |
| App and tabs | `force-app/main/default/applications`, `tabs` | E-Bikes app shell and navigation |
| Experience Cloud | `experiences`, `networks`, `sites`, `navigationMenus`, `networkBranding` | Public site metadata |
| Guest access | `guest-profile-metadata` | Guest profile and sharing rules deployed after site publish |
| Sample data | `data` | Import plan plus Account, Product Family, Product records |
| Tooling | `package.json`, `jest.config.js`, `utam.config.js`, `wdio.conf.js` | Local JavaScript, unit, UI, and formatting tools |

The application dependency map is:

```text
Account
  -> Order__c
       -> Order_Item__c -> Product__c -> Product_Family__c

Case -> Product__c

Manufacturing_Event__e -> orderStatusPath LWC -> Order__c.Status__c
Order__ChangeEvent -> ChangeEvents platform event channel member
```

## 4. Prerequisites

Install these before starting:

| Tool | Where used | Why |
|---|---|---|
| Git | Git / VS Code terminal | Clone, branch, commit, and push the project |
| VS Code | Local development | Edit metadata and run Salesforce commands |
| Salesforce Extension Pack | VS Code | Salesforce project commands, Apex/LWC language support |
| Salesforce CLI `sf` | Terminal / CI | Authenticate, create orgs, deploy, import data, run tests |
| Node.js | Local tooling | Required for npm scripts; `package.json` pins Volta Node `22.18.0` |
| npm | Local tooling | Install dependencies and run Jest, ESLint, Prettier, UTAM |
| Chrome | UTAM UI tests | Browser automation target |
| A Dev Hub org | Scratch org path | Required to create scratch orgs |
| Fresh Developer Edition or Trailhead Playground | Persistent learning path | Useful when you want a longer-lived org |

Beginner setup in VS Code:

1. Open VS Code.
2. Install the Salesforce Extension Pack.
3. Open the folder containing this repository.
4. Open the integrated terminal.
5. Run `sf --version` and `git --version`.
6. Run `npm install` if you plan to run local tests and formatting.

## 5. Salesforce DX Project Setup

### What to build or configure

You do not need to invent a project layout. This repository is already a Salesforce DX project.

### Where to do it

VS Code and terminal.

### Exact file

`sfdx-project.json`

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

### Why this step is needed

Salesforce DX uses `sfdx-project.json` to know which folders contain deployable source and which Salesforce Metadata API version to use.

### Salesforce concept

A package directory is a source-format folder that can be deployed to an org. This repo has one default package directory: `force-app`.

### How it connects

Every deploy command in this guide points at `force-app` or a subfolder of it. The `sourceApiVersion` of `65.0` controls metadata behavior for the source files in this project.

### Verify

```powershell
sf project list
```

Expected result: the project is recognized, and `force-app` is the package directory.

### Common errors and fixes

| Error | Fix |
|---|---|
| `This directory does not contain a Salesforce DX project` | Run commands from the repository root where `sfdx-project.json` exists. |
| Deploy uses unexpected files | Check `packageDirectories`; this repo's deployable source is under `force-app`. |

## 6. Scratch Org And Developer Org Setup

### Scratch org definition

Exact file: `config/project-scratch-def.json`

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

| Setting | Meaning |
|---|---|
| `orgName: Ebikes` | Human-readable scratch org name |
| `edition: Developer` | Scratch org edition shaped like a Developer org |
| `hasSampleData: false` | Starts without Salesforce sample data |
| `Communities` | Allows Experience Cloud / Digital Experiences metadata |
| `Walkthroughs` | Supports in-app guidance prompt metadata in `force-app/main/default/prompts` |
| `EnableSetPasswordInApi` | Helpful for automated user/login flows |
| `enableNetworksEnabled` | Enables Experience Cloud networks |
| `enableExperienceBundleMetadata` | Allows deployment of `force-app/main/default/experiences/E_Bikes1` |
| `enableS1DesktopEnabled` | Enables Lightning Experience |
| `enableS1EncryptedStoragePref2: false` | Matches sample app mobile setting |
| `enableOrchestrationInSandbox` | Supports walkthrough/orchestration-related engagement settings |

### Scratch org path

Where: Salesforce CLI.

```powershell
sf org login web -d -a myhuborg
sf org create scratch -d -f config/project-scratch-def.json -a ebikes
sf org open -o ebikes
sf org delete scratch -o ebikes -p
```

Use a scratch org when:

| Use scratch org when | Why |
|---|---|
| You want a disposable environment | You can recreate from source |
| You are practicing CI/CD | CI creates, validates, and deletes orgs |
| You want source tracking | Scratch orgs are designed for DX workflows |

### Developer Edition / Trailhead Playground path

Where: Salesforce CLI and Salesforce Setup UI.

```powershell
sf org login web -s -a mydevorg
sf org open -o mydevorg
```

Use a Developer Edition or Trailhead Playground when:

| Use Developer org when | Why |
|---|---|
| You want a longer-lived learning org | Scratch orgs expire |
| You are not using Dev Hub yet | Developer org install does not require scratch org creation |
| You want to manually inspect Setup over time | The org persists |

### Digital Experiences setup for Developer Edition / Trailhead Playground

Where: Salesforce Setup UI.

1. Open Setup.
2. Search for **Digital Experiences Settings**.
3. Select **Settings** under Digital Experiences.
4. Check **Enable Digital Experiences**.
5. Click **Save**, then confirm.
6. Return to Setup.
7. Search for **Experience Management Settings**.
8. Enable **ExperienceBundle Metadata API**.
9. Click **Save**.
10. Confirm My Domain is configured and deployed.

### Site metadata values for Developer Edition / Trailhead Playground

Exact file: `force-app/main/default/sites/E_Bikes.site-meta.xml`

The checked-in file contains:

```xml
<siteAdmin>test-yypfaowt42yo@example.com</siteAdmin>
<siteGuestRecordDefaultOwner>test-yypfaowt42yo@example.com</siteGuestRecordDefaultOwner>
<subdomain>efficiency-connect-1801-dev--1701a0bb7eb</subdomain>
<urlPathPrefix>ebikes</urlPathPrefix>
```

For a Developer Edition or Trailhead Playground install, update those values in your working copy before deploying:

| Element | Replace with |
|---|---|
| `siteAdmin` | Your org username |
| `siteGuestRecordDefaultOwner` | Your org username |
| `subdomain` | Your Experience Cloud / My Domain subdomain, not the full URL |
| `urlPathPrefix` | Leave as `ebikes` unless you intentionally change the site URL path |

This repository file was not changed by this guide. The instruction above is what a learner does in their own install when targeting a persistent org.

### Case Product field conflict

README says to remove a `Product` picklist field from Case in fresh Developer Edition / Trailhead Playground orgs if it conflicts. The repo adds its own Case lookup field:

| File | Metadata |
|---|---|
| `force-app/main/default/objects/Case/fields/Product__c.field-meta.xml` | Lookup from Case to `Product__c` |

Use this only as a fresh learning-org troubleshooting step:

1. Setup -> Object Manager -> Case -> Fields & Relationships.
2. Search for **Product**.
3. If a deployment error specifically reports a conflict with an existing Product field, delete the conflicting custom Product field only in a fresh disposable learning org.
4. Do not delete fields in a business org without impact analysis and backup.

## 7. Deployment Strategy

### Recommended scratch org installation

Where: Salesforce CLI.

```powershell
sf org login web -d -a myhuborg
sf org create scratch -d -f config/project-scratch-def.json -a ebikes
sf project deploy start -o ebikes
sf org assign permset -n ebikes -o ebikes
sf data import tree --plan data/sample-data-plan.json -o ebikes
sf community publish -n E-Bikes -o ebikes
sf project deploy start --metadata-dir guest-profile-metadata -w 10 -o ebikes
sf org open -o ebikes
```

README's historical data command is also valid in many CLI versions:

```powershell
sf data tree import -p ./data/sample-data-plan.json
```

### Recommended Developer Edition / Trailhead Playground installation

Where: Salesforce Setup UI and Salesforce CLI.

1. Authenticate:

```powershell
sf org login web -s -a mydevorg
```

2. Enable Digital Experiences and ExperienceBundle Metadata API in Setup.
3. Update `force-app/main/default/sites/E_Bikes.site-meta.xml` with your org-specific site values.
4. Resolve the fresh-org Case Product field conflict only if deployment reports it.
5. Deploy and configure:

```powershell
sf project deploy start -d force-app -o mydevorg
sf org assign permset -n ebikes -o mydevorg
sf data import tree --plan data/sample-data-plan.json -o mydevorg
sf community publish -n E-Bikes -o mydevorg
sf project deploy start --metadata-dir guest-profile-metadata -w 10 -o mydevorg
sf org open -o mydevorg
```

6. Setup -> Themes and Branding -> activate **Lightning Lite** if it is available in the org, matching README guidance.
7. App Launcher -> **E-Bikes**.

### Why deploy all of `force-app`

For learning, you can study metadata in phases. For installation, deploy `force-app` together. The repo contains cross-references:

| Dependency | Example |
|---|---|
| Objects reference activated record pages | Product and Order pages are FlexiPages |
| FlexiPages reference LWCs and Aura template | `Product_Explorer.flexipage-meta.xml` uses Product Explorer components |
| App references tabs | `EBikes.app-meta.xml` uses Product Explorer and object tabs |
| ExperienceBundle references LWCs, routes, assets, network | `experiences/E_Bikes1/**` |
| Guest profile references published site/profile context | `guest-profile-metadata` deploys after site publish |

## 8. Data Model Implementation

### Metadata inventory

| Object | Label | Name field | Sharing | Purpose |
|---|---|---|---|---|
| `Product_Family__c` | Product Family | Text | ReadWrite | Groups bicycle models |
| `Product__c` | Product | Text | ReadWrite | Product catalog |
| `Order__c` | Reseller Order | AutoNumber | ReadWrite | Reseller order header |
| `Order_Item__c` | Order Item | AutoNumber | ControlledByParent | Products and quantities on an order |
| `Manufacturing_Event__e` | Manufacturing Event | Platform event | Event | Streams manufacturing status updates |
| `Case` extension | Case | Standard object | Standard | Adds product support fields |

### Relationships

| Child object | Field | Parent object | Type | Why |
|---|---|---|---|---|
| `Product__c` | `Product_Family__c` | `Product_Family__c` | Lookup | A product can belong to a family without strict ownership |
| `Order__c` | `Account__c` | `Account` | Required lookup | A reseller order belongs to an Account |
| `Order_Item__c` | `Order__c` | `Order__c` | Master-detail | Order items are owned by the order and deleted with it |
| `Order_Item__c` | `Product__c` | `Product__c` | Lookup | An item references the product being ordered |
| `Case` | `Product__c` | `Product__c` | Lookup | Support cases can identify a related product |

### Fields verified in repository

| Object | Fields |
|---|---|
| `Product_Family__c` | `Category__c` picklist: Commuter, Hybrid, Mountain; `Description__c` TextArea |
| `Product__c` | `Battery__c`, `Category__c`, `Charger__c`, `Description__c`, `Fork__c`, `Frame_Color__c`, `Front_Brakes__c`, `Handlebar_Color__c`, `Level__c`, `Material__c`, `Motor__c`, `MSRP__c`, `Picture_URL__c`, `Product_Family__c`, `Rear_Brakes__c`, `Seat_Color__c`, `Waterbottle_Color__c` |
| `Order__c` | `Account__c` required lookup, `Status__c` required picklist: Draft, Submitted to Manufacturing, Approved by Manufacturing, In Production |
| `Order_Item__c` | `Order__c`, `Product__c`, `Price__c`, `Qty_S__c`, `Qty_M__c`, `Qty_L__c` |
| `Case` | `Case_Category__c` picklist: Mechanical, Electrical, Electronic, Structural, Other; `Product__c` lookup |
| `Manufacturing_Event__e` | Required text fields `Order_Id__c`, `Status__c` |

### Where to study or deploy

| Step | Path |
|---|---|
| Custom object XML | `force-app/main/default/objects/<ObjectName>/<ObjectName>.object-meta.xml` |
| Fields | `force-app/main/default/objects/<ObjectName>/fields/*.field-meta.xml` |
| List views | `force-app/main/default/objects/<ObjectName>/listViews/*.listView-meta.xml` |
| Layouts | `force-app/main/default/layouts/*.layout-meta.xml` |

### Beginner manual setup concept

If rebuilding by hand in Setup:

1. Setup -> Object Manager -> Create -> Custom Object.
2. Create parent objects before child objects.
3. Add fields from Object Manager -> Object -> Fields & Relationships -> New.
4. Choose Lookup when records can exist independently.
5. Choose Master-Detail when the child record should be owned by and deleted with the parent.
6. Add list views after objects and fields exist.
7. Add layouts after fields exist.

### CLI deploy

For a faithful repo install:

```powershell
sf project deploy start --source-dir force-app/main/default/objects -o ebikes -w 20
```

If dependency errors occur because object metadata activates pages, deploy all of `force-app` instead:

```powershell
sf project deploy start --source-dir force-app -o ebikes -w 30
```

### Verify

```powershell
sf data query -q "SELECT Id, Name FROM Product_Family__c LIMIT 1" -o ebikes
sf data query -q "SELECT Id, Name, Product_Family__c FROM Product__c LIMIT 1" -o ebikes
sf data query -q "SELECT Id, Name, Status__c FROM Order__c LIMIT 1" -o ebikes
```

## 9. Security And Permissions Implementation

### Permission sets

Verified files:

| File | Label | Purpose |
|---|---|---|
| `force-app/main/default/permissionsets/ebikes.permissionset-meta.xml` | `ebikes` | Main internal user permission set for objects, fields, Apex, tabs, app, and custom permissions needed by the sample |
| `force-app/main/default/permissionsets/sfdcInternalInt__sfdc_scrt2.permissionset-meta.xml` | `SCRT2 Integration User` | Cloud Integration User permission set with Case access; included in repo metadata but not part of the normal learner assignment flow |

The README mentions a `Walkthroughs` permission set for scratch orgs. This repo does not contain a local `Walkthroughs` permission set file. If your scratch org provides it because of the `Walkthroughs` feature, you can run:

```powershell
sf org assign permset -n Walkthroughs -o ebikes
```

If that command fails with "not found", continue; the repository metadata does not provide it.

### Assign internal user permissions

```powershell
sf org assign permset -n ebikes -o ebikes
```

### What `ebikes` teaches

| Concept | Meaning |
|---|---|
| Object permissions | Grants read/create/edit/delete access to app objects |
| Field-level security | Grants field visibility/editability for custom fields |
| Apex class access | Allows LWC users to call Apex controllers |
| Tab visibility | Shows Product Explorer, Product, Product Family, and Order tabs |
| Least privilege | Users need explicit rights even if metadata exists |

### Guest profile metadata

Path: `guest-profile-metadata`

| File/folder | Purpose |
|---|---|
| `guest-profile-metadata/package.xml` | Deploy package manifest for guest metadata |
| `profiles/E-Bikes Profile.profile` | Public guest profile access |
| `sharingRules/Product__c.sharingRules` | Guest visibility for products |
| `sharingRules/Product_Family__c.sharingRules` | Guest visibility for product families |
| `profileSessionSettings/**` | Session controls for the guest profile |
| `profilePasswordPolicies/**` | Password policy metadata associated with the profile export |

Deploy it only after the site is published:

```powershell
sf community publish -n E-Bikes -o ebikes
sf project deploy start --metadata-dir guest-profile-metadata -w 10 -o ebikes
```

### Verify security

| User type | Check |
|---|---|
| Internal user | App Launcher -> E-Bikes; Product Explorer loads products |
| Internal user | Can create Reseller Order and edit Order Items |
| Guest user | Incognito browser -> Experience site; product catalog is visible |
| Guest user | Create Case page loads only fields allowed by guest profile/FLS |

## 10. Apex Backend Implementation

### Apex inventory

| Class | Type | Purpose | Important concepts |
|---|---|---|---|
| `PagedResult` | DTO | Returns paginated Apex data to LWC | `@AuraEnabled` properties |
| `ProductController` | Controller | `getProducts`, `getSimilarProducts` | `with sharing`, `@AuraEnabled(Cacheable=true scope='global')`, dynamic SOQL, `WITH USER_MODE` |
| `OrderController` | Controller | `getOrderItems` | Relationship SOQL, `WITH USER_MODE` |
| `ProductRecordInfoController` | Controller | Finds product/family record for hero link | Cacheable Apex for Experience Cloud |
| `HeroDetailsPositionCustomPicklist` | Design helper | Supplies Experience Builder design picklist values | `VisualEditor.DynamicPickList` |
| `CommunitiesLandingController` | Visualforce controller | Routes site landing page to community start URL | Apex page controller |
| `TestProductController` | Test | Tests product controller behavior | Test data setup |
| `TestOrderController` | Test | Tests order item retrieval | Test data setup |
| `CommunitiesLandingControllerTest` | Test | Tests landing controller | Visualforce controller test |

### Key code patterns verified

`ProductController` starts with:

```apex
public with sharing class ProductController {
    static Integer PAGE_SIZE = 9;
```

`with sharing` means Apex respects record sharing rules. The controller also uses `WITH USER_MODE` in SOQL, which enforces object and field permissions for the running user.

`@AuraEnabled(Cacheable=true)` means an LWC can call the method with `@wire`, and Salesforce can cache the read-only result.

`OrderController.getOrderItems` queries child records:

```apex
FROM Order_Item__c
WHERE Order__c = :orderId
WITH USER_MODE
```

### Where to build or study

Exact folder: `force-app/main/default/classes`

Each class has:

| File | Purpose |
|---|---|
| `<ClassName>.cls` | Apex code |
| `<ClassName>.cls-meta.xml` | API version and active status |

### Deploy Apex

```powershell
sf project deploy start --source-dir force-app/main/default/classes -o ebikes -w 20
```

### Run Apex tests

README workflow uses:

```powershell
sf apex test run -c -r human -d ./tests/apex -w 20
```

Modern equivalent with explicit target org:

```powershell
sf apex run test --test-level RunLocalTests --result-format human --code-coverage -o ebikes -w 20
```

### Verify

All local Apex tests should pass. If Apex calls fail from LWC, check permission set Apex class access and field-level security first.

## 11. LWC Frontend Implementation

### LWC folder structure

Every component lives under `force-app/main/default/lwc/<componentName>`.

| File | Purpose |
|---|---|
| `<component>.html` | Template |
| `<component>.js` | JavaScript controller |
| `<component>.css` | Component-scoped styles, when present |
| `<component>.js-meta.xml` | Targets and design properties |
| `__tests__/*.test.js` | Jest tests, when present |
| `__utam__/*.utam.json` | UTAM page object definitions, when present |

### Component inventory

| Component | Purpose | Key concepts |
|---|---|---|
| `accountMap` | Shows Account billing address on a map | `@api recordId`, `@wire(getRecord)` |
| `createCase` | Public Experience Cloud Case form | `lightning-record-edit-form`, toast, custom `refresh` event |
| `errorPanel` | Displays normalized errors | `@api`, reusable templates |
| `hero` | Configurable Experience Cloud hero | `@api` design properties, static/external media |
| `heroDetails` | Links hero button to Product or Product Family | Apex wire, record navigation |
| `ldsUtils` | Reduces LDS/Apex errors | Reusable JS module |
| `orderBuilder` | Drag/drop order item builder | `@api recordId`, Apex wire, `createRecord`, `updateRecord`, `deleteRecord` |
| `orderItemTile` | Editable order item child tile | `@api`, child-to-parent custom events |
| `orderStatusPath` | Order status path with platform event updates | UI Record API, UI Object Info API, EMP API |
| `paginator` | Previous/next controls | `@api`, custom events |
| `placeholder` | Empty-state display | Static resource |
| `productCard` | Shows selected product details | LMS subscribe, `NavigationMixin` |
| `productFilter` | Publishes product filters | `@wire(getPicklistValues)`, LMS publish |
| `productListItem` | Compact product row | `NavigationMixin` |
| `productTile` | Product card/tile | `@api`, custom `selected`, drag data |
| `productTileList` | Paged product grid | Apex wire, LMS subscribe/publish, paginator |
| `similarProducts` | Product record related products | `@api recordId`, LDS, Apex wire |

### LWC concepts verified

| Concept | Where found | Explanation |
|---|---|---|
| `@api` | Many components | Public property from parent, App Builder, or record page |
| `@track` | Not used in current repo | Modern LWC tracks most object/array state without explicit `@track`; mention this so learners do not search for it |
| `@wire` | `accountMap`, `productFilter`, `productTileList`, `similarProducts`, `orderBuilder`, `orderStatusPath`, `heroDetails` | Reactive data provisioning |
| Lightning Data Service | `getRecord`, `createRecord`, `updateRecord`, `deleteRecord` imports | Record access without custom Apex |
| UI Record API | `lightning/uiRecordApi` | Reads and writes records |
| UI Object Info API | `lightning/uiObjectInfoApi` | Reads picklists/object metadata |
| Custom events | `paginator`, `productTile`, `orderItemTile`, `createCase` | Child-to-parent communication |
| Parent-to-child communication | `@api` properties in child components | Parent sets data into child markup |
| NavigationMixin | `productCard`, `productListItem` | Navigates to record pages |
| Drag and drop | `productTile`, `orderBuilder` | Tile sets drag data; order builder handles drop and creates order item |

### Deploy LWC

```powershell
sf project deploy start --source-dir force-app/main/default/lwc -o ebikes -w 20
```

If LWC deploys but pages fail, deploy full `force-app` because pages reference components and targets.

### Verify

1. Open App Launcher -> E-Bikes.
2. Open Product Explorer.
3. Filter products.
4. Select a product tile.
5. Open a Product record and check Similar Products.
6. Create an Order and open Order Builder.

## 12. Lightning Message Service Implementation

### Metadata

| Message channel | File | Purpose |
|---|---|---|
| `ProductsFiltered__c` | `force-app/main/default/messageChannels/ProductsFiltered.messageChannel-meta.xml` | Sends filter criteria from `productFilter` to `productTileList` |
| `ProductSelected__c` | `force-app/main/default/messageChannels/ProductSelected.messageChannel-meta.xml` | Sends selected product ID from `productTileList` to `productCard` |

### Component flow

```text
productFilter
  publish ProductsFiltered__c
      -> productTileList receives filters and calls ProductController.getProducts

productTileList
  publish ProductSelected__c
      -> productCard receives selection and displays details
```

### Why LMS is needed

Product Filter, Product Tile List, and Product Card are placed in separate FlexiPage regions. They do not have a direct parent-child relationship. Lightning Message Service lets unrelated components on the same Lightning page communicate.

### Deploy and verify

```powershell
sf project deploy start --source-dir force-app/main/default/messageChannels -o ebikes -w 10
```

Open Product Explorer. Changing filters should change the product list. Clicking a product should update the card.

## 13. Platform Event Implementation

### Metadata

| Metadata | File | Purpose |
|---|---|---|
| Platform Event | `force-app/main/default/objects/Manufacturing_Event__e` | Custom event with `Order_Id__c` and `Status__c` |
| Channel member | `force-app/main/default/platformEventChannelMembers/ChangeEvents_Order_ChangeEvent.platformEventChannelMember-meta.xml` | Adds `Order__ChangeEvent` to the standard `ChangeEvents` channel for CDC |
| LWC subscriber | `force-app/main/default/lwc/orderStatusPath/orderStatusPath.js` | Subscribes to `/event/Manufacturing_Event__e` with EMP API |

### Important distinction

This repo contains both:

| Event type | What it does here |
|---|---|
| `Manufacturing_Event__e` Platform Event | Used directly by `orderStatusPath` to receive manufacturing status updates |
| `Order__ChangeEvent` channel member | Change Data Capture metadata present for Order changes, likely used by optional demo/event scenarios |

The README mentions an optional external `ebikes-manufacturing` demo repository for Pub/Sub API. That is optional and not required to install this app.

### Test publish event manually

Where: Developer Console or anonymous Apex.

```apex
EventBus.publish(new Manufacturing_Event__e(
    Order_Id__c = 'PUT_ORDER_RECORD_ID_HERE',
    Status__c = 'In Production'
));
```

Then open the matching Order record. The `orderStatusPath` component should update when the event arrives.

### Common errors

| Error | Fix |
|---|---|
| EMP API not enabled | Test inside Lightning Experience on a supported page |
| Status does not change | Confirm `Order_Id__c` equals the current order record ID |
| Invalid status | Use a value from `Order__c.Status__c` picklist |

## 14. Lightning App, Tabs, Layouts, And Pages

### Metadata inventory

| Type | Path | Files |
|---|---|---|
| Lightning app | `force-app/main/default/applications` | `EBikes.app-meta.xml` |
| Tabs | `force-app/main/default/tabs` | `Product_Explorer`, `Product__c`, `Product_Family__c`, `Order__c` |
| FlexiPages | `force-app/main/default/flexipages` | Account, Order, Product, Product Explorer pages |
| Layouts | `force-app/main/default/layouts` | Product, Product Family, Order, Order Item, Case layouts |
| List views | Object `listViews` folders | ProductList, All Product Families, All Orders, My Cases |
| Aura template | `force-app/main/default/aura/pageTemplate_2_7_3` | Three-region custom template |

### What each page teaches

| Page | File | Teaches |
|---|---|---|
| Product Explorer | `Product_Explorer.flexipage-meta.xml` | Composing unrelated LWCs with LMS |
| Product Record Page | `Product_Record_Page.flexipage-meta.xml` | Record context and related product component |
| Order Record Page | `Order_Record_Page.flexipage-meta.xml` | Record context, drag/drop, status path, event-driven updates |
| Account Record Page | `Account_Record_Page.flexipage-meta.xml` | Standard object extension with `accountMap` |

### Deploy

```powershell
sf project deploy start --source-dir force-app/main/default/aura --source-dir force-app/main/default/flexipages --source-dir force-app/main/default/tabs --source-dir force-app/main/default/applications --source-dir force-app/main/default/layouts -o ebikes -w 20
```

### Verify in Setup UI

1. Setup -> Lightning App Builder.
2. Open Product Explorer.
3. Confirm left/center/right regions.
4. Setup -> App Manager -> E-Bikes -> View.
5. App Launcher -> E-Bikes.
6. Confirm navigation items: Product Explorer, Products, Product Families, Reseller Orders, Accounts, Contacts/Home as configured.

## 15. Experience Cloud Implementation

### Metadata inventory

| Type | Path | Purpose |
|---|---|---|
| Site | `force-app/main/default/sites/E_Bikes.site-meta.xml` | Salesforce Site shell and URL path |
| Network | `force-app/main/default/networks/E-Bikes.network-meta.xml` | Experience Cloud network |
| ExperienceBundle | `force-app/main/default/experiences/E_Bikes1` | Routes, views, themes, config, branding |
| Navigation menus | `force-app/main/default/navigationMenus` | Public site navigation |
| Network branding | `force-app/main/default/networkBranding` | Branding metadata |
| Audience | `force-app/main/default/audience/Default_E-Bikes.audience-meta.xml` | Experience audience |
| Visualforce landing page | `force-app/main/default/pages/CommunitiesLanding.page` | Landing route controller |
| Content assets | `force-app/main/default/contentassets` | Logos for app/site |

### Manual Setup UI prerequisites

For Developer Edition / Trailhead Playground:

1. Setup -> Digital Experiences -> Settings.
2. Enable Digital Experiences.
3. Save.
4. Setup -> Experience Management Settings.
5. Enable ExperienceBundle Metadata API.
6. Save.
7. Ensure My Domain is deployed.

Scratch orgs get these from `config/project-scratch-def.json`.

### Site metadata values explained

| Element | Meaning |
|---|---|
| `siteAdmin` | User who administers the site; must exist in target org |
| `siteGuestRecordDefaultOwner` | Owner assigned to records created by guest user; must exist in target org |
| `subdomain` | Digital Experience subdomain for the target org |
| `urlPathPrefix` | Site path, `ebikes`, used in the public URL |
| `indexPage` | Uses Visualforce page `CommunitiesLanding` |
| `siteType` | `ChatterNetwork`, the metadata type used for Experience Cloud sites |

### Deploy and publish

```powershell
sf project deploy start --source-dir force-app -o ebikes -w 30
sf community publish -n E-Bikes -o ebikes
sf project deploy start --metadata-dir guest-profile-metadata -w 10 -o ebikes
```

### Verify

1. Setup -> Digital Experiences -> All Sites.
2. Confirm **E-Bikes** exists.
3. Click Builder to inspect pages.
4. Click Workspaces -> Administration if needed.
5. Open the public URL in an incognito window.

## 16. Static Resources, Content Assets, And CSP Trusted Sites

### Static resources and content assets

| Path | Purpose |
|---|---|
| `force-app/main/default/staticresources/bike_assets` | Local images: `racer.png`, `enthusiast.png`, `commuter.png`, `CyclingGrass.jpg`, `logo.svg` |
| `force-app/main/default/staticresources/bike_assets.resource-meta.xml` | Static resource metadata |
| `force-app/main/default/contentassets/logo.asset*` | Content asset logo |
| `force-app/main/default/contentassets/ebikeslogo.asset*` | Experience/site logo asset |

### CSP Trusted Site

Exact file: `force-app/main/default/cspTrustedSites/s3_us_west_2_amazonaws_com.cspTrustedSite-meta.xml`

It allows images and media from:

```text
https://s3-us-west-2.amazonaws.com
```

This is not an API integration. It is a browser security allowlist so Salesforce pages can load sample product images and the Experience hero video from Amazon S3 URLs found in `data/Product__cs.json` and `experiences/E_Bikes1/views/home.json`.

### External integration and OAuth verification

The codebase was checked for callouts, Named Credentials, External Credentials, Connected Apps, OAuth flows, API keys, tokens, and third-party service calls. No required external API integration or OAuth 2.0 flow exists in this app.

External URLs in this repo are:

| External resource | Role |
|---|---|
| Amazon S3 product images/video | Static media loaded by browser; allowed by CSP Trusted Site |
| Trailhead, Salesforce docs, YouTube, GitHub links in README/prompts | Documentation/demo links |
| Optional `ebikes-manufacturing` repository | Separate optional demo, not required |
| Codecov in GitHub Actions | CI coverage upload, not Salesforce app runtime |

No Salesforce Named Credential, External Credential, Remote Site Setting, Auth Provider, Connected App, client secret, or API token is required to run the app.

## 17. Sample Data Import

### Data files

| File | Records | First record | Purpose |
|---|---:|---|---|
| `data/Accounts.json` | 3 | Wheelworks | Reseller accounts |
| `data/Product_Family__cs.json` | 4 | Dynamo | Product families |
| `data/Product__cs.json` | 16 | FUSE X1 | Products shown in catalog |
| `data/sample-data-plan.json` | 3 plan entries | Account first | Import order and reference resolution |

Exact import plan:

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

### Why this order matters

Products reference Product Families. The plan saves references for `Product_Family__c`, then resolves them when inserting `Product__c`.

### Import commands

Modern command:

```powershell
sf data import tree --plan data/sample-data-plan.json -o ebikes
```

README command style:

```powershell
sf data tree import -p ./data/sample-data-plan.json
```

### Verify

```powershell
sf data query -q "SELECT COUNT() FROM Account" -o ebikes
sf data query -q "SELECT COUNT() FROM Product_Family__c" -o ebikes
sf data query -q "SELECT COUNT() FROM Product__c" -o ebikes
```

Expected sample-data counts: at least 3 Accounts from this plan, 4 Product Families, and 16 Products.

The data plan does not create `Order__c` or `Order_Item__c` records. Create a Reseller Order manually in the app, then drag products into it with Order Builder.

## 18. Testing Strategy

### npm scripts verified in `package.json`

| Script | Command | Runs where |
|---|---|---|
| `npm run lint` | `eslint .` | Local/CI |
| `npm test` | `npm run test:unit` | Local/CI |
| `npm run test:unit` | `sfdx-lwc-jest` | Local/CI |
| `npm run test:unit:watch` | `sfdx-lwc-jest --watch` | Local |
| `npm run test:unit:debug` | `sfdx-lwc-jest --debug` | Local |
| `npm run test:unit:coverage` | `sfdx-lwc-jest --coverage` | Local/CI |
| `npm run test:ui:compile` | `utam -c utam.config.js` | Local/CI-capable |
| `npm run test:ui:generate:login` | `node scripts/generate-login-url.js` | Local with authenticated org |
| `npm run test:ui` | `wdio` | Local/browser |
| `npm run prettier` | Prettier write | Local |
| `npm run prettier:verify` | Prettier check | Local/CI |

### Jest

Config file: `jest.config.js`

It uses `@salesforce/sfdx-lwc-jest`, adds `jest-sa11y-setup.js`, and maps mocks for Apex, navigation, message service, and EMP API from `force-app/test/jest-mocks`.

Run:

```powershell
npm install
npm run test:unit
npm run test:unit:coverage
```

### UTAM and WebdriverIO

| File | Purpose |
|---|---|
| `utam.config.js` | Compiles page objects from `force-app/**/__utam__/**/*.utam.json` |
| `wdio.conf.js` | Runs Chrome with Jasmine and `wdio-utam-service` |
| `force-app/test/utam/page-explorer.spec.js` | End-to-end Product Explorer test |
| `scripts/generate-login-url.js` | Writes `.env` with Salesforce login URL for UI tests |

Run:

```powershell
npm run test:ui:compile
npm run test:ui:generate:login
npm run test:ui
```

The UTAM test logs in, opens Product Explorer, verifies 16 products, filters for `fuse`, selects `FUSE X1`, and navigates to the Product record page.

### Apex tests

Run in an org:

```powershell
sf apex run test --test-level RunLocalTests --result-format human --code-coverage -o ebikes -w 20
```

### Common testing errors

| Error | Fix |
|---|---|
| Jest cannot find modules | Run `npm install` |
| Prettier Apex plugin fails | Install Java 11 or newer |
| UTAM login fails | Authenticate an org, then rerun `npm run test:ui:generate:login` |
| Chrome does not open | Check Chrome installation and WebdriverIO/chromedriver compatibility |
| Apex tests fail with access/query errors | Confirm metadata deployed and `ebikes` permission set assigned |

## 19. CI/CD Pipeline Setup

### Existing workflows

This repository already contains GitHub Actions workflows:

| Workflow | Path | Purpose |
|---|---|---|
| CI | `.github/workflows/ci.yml` | Runs on pushes to `main`; Prettier, lint, LWC coverage, scratch org deployment, sample data, site publish, guest profile deploy, Apex tests, delete scratch org |
| CI on PR | `.github/workflows/ci-pr.yml` | Runs on pull requests; includes normal and prerelease scratch org paths |
| CodeTour watch | `.github/workflows/codetour-watch.yml` | CodeTour-related automation |
| New issue greeting | `.github/workflows/new-issue-welcome.yml` | Issue greeting automation |
| Issue assignment | `.github/workflows/auto-assign.yml` | Issue assignment automation |

Important verified CI steps:

| CI step | Actual command |
|---|---|
| Install Salesforce CLI | `npm install @salesforce/cli --location=global` |
| Install Node dependencies | `HUSKY=0 npm ci` |
| Prettier check | `npm run prettier:verify` |
| Lint | `npm run lint` |
| LWC tests | `npm run test:unit:coverage` |
| Auth Dev Hub | `sf org login sfdx-url -f ./DEVHUB_SFDX_URL.txt -a devhub -d` |
| Create scratch org | `sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 1` |
| Deploy source | `sf project deploy start` |
| Assign permissions | `sf org assign permset -n ebikes` |
| Import data | `sf data tree import -p ./data/sample-data-plan.json` |
| Publish site | `sf community publish -n E-Bikes` |
| Deploy guest profile | `sf project deploy start --metadata-dir=guest-profile-metadata -w 10` |
| Apex tests | `sf apex test run -c -r human -d ./tests/apex -w 20` |
| Delete scratch org | `sf org delete scratch -p -o scratch-org` |

### Secrets used by existing CI

| Secret | Meaning |
|---|---|
| `DEVHUB_SFDX_URL` | Salesforce auth URL for Dev Hub, used to create scratch orgs |
| `CODECOV_TOKEN` | Token used by Codecov upload action |

### Sample beginner CI YAML, documentation only

Do not create this file unless you choose to add a new workflow later. Suggested path would be `.github/workflows/salesforce-ci.yml`.

```yaml
name: Salesforce Beginner CI

on:
    pull_request:
    workflow_dispatch:

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
              run: HUSKY=0 npm ci

            - name: Verify formatting
              run: npm run prettier:verify

            - name: Lint
              run: npm run lint

            - name: Run LWC Jest tests
              run: npm run test:unit:coverage

            - name: Install Salesforce CLI
              run: npm install @salesforce/cli --location=global

            - name: Authenticate Dev Hub
              run: |
                  echo "${{ secrets.DEVHUB_SFDX_URL }}" > DEVHUB_SFDX_URL.txt
                  sf org login sfdx-url -f DEVHUB_SFDX_URL.txt -a devhub -d

            - name: Create scratch org
              run: sf org create scratch -f config/project-scratch-def.json -a scratch-org -d -y 1

            - name: Deploy metadata
              run: sf project deploy start -o scratch-org -w 30

            - name: Assign permission set
              run: sf org assign permset -n ebikes -o scratch-org

            - name: Import sample data
              run: sf data import tree --plan data/sample-data-plan.json -o scratch-org

            - name: Publish Experience site
              run: sf community publish -n E-Bikes -o scratch-org

            - name: Deploy guest profile metadata
              run: sf project deploy start --metadata-dir guest-profile-metadata -w 10 -o scratch-org

            - name: Run Apex tests
              run: sf apex run test --test-level RunLocalTests --result-format human --code-coverage -o scratch-org -w 20

            - name: Delete scratch org
              if: always()
              run: sf org delete scratch -o scratch-org -p
```

### What each section means

| YAML section | Meaning |
|---|---|
| `on` | Defines when the pipeline runs |
| `jobs.validate` | One validation job |
| `actions/checkout` | Pulls repository files into the runner |
| Node setup and `npm ci` | Installs exact dependencies from `package-lock.json` |
| Prettier/lint/Jest | Fast local quality checks |
| Salesforce CLI install | Provides the `sf` command |
| Dev Hub auth | Uses secure GitHub secret, not a password in source |
| Scratch org | Creates disposable validation org |
| Deploy/data/site/guest profile | Recreates the app |
| Apex tests | Validates Salesforce server-side behavior |
| Delete scratch org | Cleans up even when earlier steps fail |

## 20. Git Workflow

### Beginner workflow

```powershell
git status
git checkout -b your-name/learn-ebikes
git add docs/EBikes-LWC-Detailed-Implementation-Guide.md
git commit -m "Add detailed E-Bikes implementation guide"
git push -u origin your-name/learn-ebikes
```

### Repository contribution guidance

`CONTRIBUTION.md` says work happens in topic branches based on `main`, pull requests are reviewed, and the project prefers squash merge.

### Good Salesforce Git habits

| Habit | Why |
|---|---|
| Commit metadata and tests together | Keeps deployable changes understandable |
| Run `npm run prettier:verify` before PR | Avoids formatting CI failures |
| Run Jest before PR | Catches LWC regressions quickly |
| Use scratch org validation | Confirms source can rebuild an org |
| Do not commit `.env` | `scripts/generate-login-url.js` writes personal login data |

## 21. Full End-To-End Build Sequence

### Scratch org

```powershell
git clone https://github.com/trailheadapps/ebikes-lwc
cd ebikes-lwc
npm install
sf org login web -d -a myhuborg
sf org create scratch -d -f config/project-scratch-def.json -a ebikes
sf project deploy start -o ebikes -w 30
sf org assign permset -n ebikes -o ebikes
sf data import tree --plan data/sample-data-plan.json -o ebikes
sf community publish -n E-Bikes -o ebikes
sf project deploy start --metadata-dir guest-profile-metadata -w 10 -o ebikes
sf apex run test --test-level RunLocalTests --result-format human --code-coverage -o ebikes -w 20
npm run prettier:verify
npm run lint
npm run test:unit
sf org open -o ebikes
```

Then:

1. App Launcher -> E-Bikes.
2. Open Product Explorer.
3. Confirm 16 sample products.
4. Create a Reseller Order.
5. Drag products into Order Builder.
6. Open Digital Experiences -> All Sites -> E-Bikes.
7. Publish again if you made Builder changes.
8. Test public site in an incognito browser.

### Developer Edition / Trailhead Playground

```powershell
git clone https://github.com/trailheadapps/ebikes-lwc
cd ebikes-lwc
npm install
sf org login web -s -a mydevorg
```

Manual Setup:

1. Enable Digital Experiences.
2. Enable ExperienceBundle Metadata API.
3. Update `force-app/main/default/sites/E_Bikes.site-meta.xml` with your username and subdomain.
4. Resolve Case Product conflict only if a fresh learning org deploy reports it.

Deploy:

```powershell
sf project deploy start -d force-app -o mydevorg -w 30
sf org assign permset -n ebikes -o mydevorg
sf data import tree --plan data/sample-data-plan.json -o mydevorg
sf community publish -n E-Bikes -o mydevorg
sf project deploy start --metadata-dir guest-profile-metadata -w 10 -o mydevorg
sf org open -o mydevorg
```

## 22. Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Site metadata fails | Digital Experiences not enabled | Enable Digital Experiences and ExperienceBundle Metadata API |
| Site deploy references old user | `siteAdmin` or `siteGuestRecordDefaultOwner` is from checked-in sample org | Replace with your target org username in your install copy |
| Subdomain error | `subdomain` does not match target org | Use your Digital Experiences/My Domain subdomain |
| Case Product field conflict | Fresh org already has a Product field on Case | Delete only the conflicting fresh-org field after confirming no data risk |
| `Walkthroughs` permset not found | Repo does not include it | Continue; assign only `ebikes` unless org provides `Walkthroughs` |
| Product Explorer empty | Data not imported or no permissions | Import data and assign `ebikes` |
| Product images/video blocked | CSP trusted site missing | Deploy `force-app/main/default/cspTrustedSites` |
| Filters do nothing | LMS channel or Apex missing | Deploy `messageChannels`, `classes`, and LWC |
| Similar Products blank | Product has no family or no related records | Check `Product_Family__c` data and sample import |
| Order Builder cannot create items | Missing `Order_Item__c` CRUD/FLS | Assign `ebikes`; check field permissions |
| Platform event does not update path | Event Order ID does not match current record | Publish event with exact Order record ID |
| Guest site cannot see products | Guest profile/sharing not deployed after publish | Publish site, then deploy `guest-profile-metadata` |
| Apex user-mode query error | Running user lacks object/FLS access | Assign permission set or fix profile/permission set access |
| Jest failures after clone | Dependencies missing | Run `npm install` |
| UTAM test fails at login | `.env` missing or expired | Run `npm run test:ui:generate:login` |
| CI cannot auth Dev Hub | Missing GitHub secret | Add `DEVHUB_SFDX_URL` repository secret |

## 23. Learning Checklist

- [ ] Explain why `force-app` is the default package directory.
- [ ] Explain why API version `65.0` matters.
- [ ] Create and delete a scratch org.
- [ ] Authenticate a Developer Edition org.
- [ ] Enable Digital Experiences and ExperienceBundle Metadata API.
- [ ] Explain `siteAdmin`, `siteGuestRecordDefaultOwner`, and `subdomain`.
- [ ] Explain lookup versus master-detail relationships.
- [ ] Explain how Case is extended with custom fields.
- [ ] Explain `Manufacturing_Event__e` and `Order__ChangeEvent`.
- [ ] Assign the `ebikes` permission set.
- [ ] Explain object permissions and field-level security.
- [ ] Explain guest profile metadata and guest sharing rules.
- [ ] Explain `with sharing`, `WITH USER_MODE`, `@AuraEnabled`, and `@AuraEnabled(Cacheable=true)`.
- [ ] Explain LWC HTML, JavaScript, CSS, and meta XML roles.
- [ ] Identify where `@api`, `@wire`, LDS, UI Record API, UI Object Info API, LMS, custom events, and NavigationMixin are used.
- [ ] Explain that `@track` is not used in this repo.
- [ ] Explain drag-and-drop from product tiles to order builder.
- [ ] Explain Aura template usage.
- [ ] Explain Visualforce landing page usage.
- [ ] Explain static resources, content assets, and CSP Trusted Sites.
- [ ] Import sample data from the plan.
- [ ] Run Apex tests, Jest tests, linting, Prettier verification, and optional UTAM tests.
- [ ] Explain the existing GitHub Actions workflows and required secrets.

## 24. Final Validation Checklist

- [ ] `sf org display -o ebikes` or `sf org display -o mydevorg` works.
- [ ] `sf project deploy start -d force-app -o <alias> -w 30` succeeds.
- [ ] `sf org assign permset -n ebikes -o <alias>` succeeds.
- [ ] `sf data import tree --plan data/sample-data-plan.json -o <alias>` succeeds.
- [ ] Product Family count is 4.
- [ ] Product count is 16.
- [ ] Experience site publishes with `sf community publish -n E-Bikes -o <alias>`.
- [ ] `guest-profile-metadata` deploys after site publish.
- [ ] App Launcher shows E-Bikes.
- [ ] Product Explorer filters and selection work.
- [ ] Product record page shows Similar Products when applicable.
- [ ] A Reseller Order can be created.
- [ ] Order Builder can add, update, and delete order items.
- [ ] Order Status Path displays status picklist values.
- [ ] Public Experience site loads in an incognito browser.
- [ ] Public Product Explorer can see products.
- [ ] `npm run prettier:verify` passes.
- [ ] `npm run lint` passes.
- [ ] `npm run test:unit` passes.
- [ ] Apex tests pass in the org.
- [ ] Optional `npm run test:ui` passes after UTAM compile and login generation.

The main lesson: Salesforce apps are layered. Build the data model first, secure it intentionally, expose it through Apex and Lightning Data Service, compose UI with LWCs, arrange those LWCs in Lightning pages and Experience Cloud, then validate everything with data, tests, and CI.
