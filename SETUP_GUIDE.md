# CFS Tribal Vehicle Usage -- Setup and Deployment Guide

**System:** CFS Tribal Vehicle Usage Power Apps
**Version:** 1.1
**Organization:** Choctaw Nation EPS
**Dataverse Instance:** `org7c6cb60f.crm.dynamics.com`
**Publisher Prefix:** `cr63e_`
**Document Updated:** February 2026

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Dataverse Table Schema -- Existing Tables](#2-dataverse-table-schema--existing-tables)
3. [Dataverse Table Schema -- New Tables for Admin App](#3-dataverse-table-schema--new-tables-for-admin-app)
4. [Table Relationship Diagram](#4-table-relationship-diagram)
5. [Security Roles](#5-security-roles)
6. [Step-by-Step Deployment Instructions](#6-step-by-step-deployment-instructions)
7. [User Guide](#7-user-guide)
8. [Appendix](#appendix)

---

## 1. System Overview

The CFS Tribal Vehicle Usage system is a Microsoft Power Apps solution that manages a fleet of tribal vehicles across multiple office locations. It consists of two applications:

| Application | Audience | Platform | App Type |
|---|---|---|---|
| **Mobile App** | Field staff, associates | Phone / tablet (Power Apps Mobile) | Canvas App |
| **Admin App** | Fleet managers, administrators | Desktop / browser | Canvas App (Desktop) |

### Architecture

- **Data layer:** Microsoft Dataverse (7 existing custom tables, 2 recommended new tables)
- **Application layer:** Power Apps Canvas Apps (YAML source-controlled)
- **Authentication:** Microsoft Entra ID (Azure AD), matched to Associates table via `User().FullName`
- **Theme:** Custom centralized theme system (`varTheme`, `varStatusColors`) defined in `App.OnStart`

### Mobile App Screens (9 total)

| Screen | Internal Name | Purpose |
|---|---|---|
| Fleet Dashboard | `scrHome` | Vehicle status counts, KPIs, quick actions, upcoming reservations |
| Vehicle Fleet | `scrVehicleList` | Browse/search/filter all vehicles by status and location |
| Vehicle Detail | `scrVehicleDetail` | View vehicle info, check out/in, reserve, set maintenance |
| New Reservation | `scrReservation` | Create a vehicle reservation with date range and purpose |
| My Reservations | `scrMyReservations` | View, filter, and cancel personal reservations |
| Log Gas Purchase | `scrGasPurchase` | Log fuel transactions with card matching and receipt capture |
| Fuel History | `scrFuelHistory` | View fuel transaction history with receipt viewer |
| Report Issue | `scrMaintenanceRequest` | Submit maintenance/issue reports for vehicles |
| More / Settings | `scrSettings` | User profile, quick links, app version info |

### Reusable Components (3)

| Component | Internal Name | Purpose |
|---|---|---|
| Header | `cmpHeader` | Consistent header bar with title, subtitle, optional back button |
| Bottom Navigation | `cmpBottomNav` | 5-tab navigation: Home, Vehicles, Reserve, Gas, More |
| Vehicle Card | `cmpVehicleCard` | Reusable card displaying vehicle name, metadata, status indicator |

---

## 2. Dataverse Table Schema -- Existing Tables

All tables use the publisher prefix `cr63e_`. Dataverse system fields (`createdby`, `createdon`, `modifiedby`, `modifiedon`, `ownerid`, `owningbusinessunit`, `cr63e_<table>id`) are present on every table and are omitted below for brevity.

---

### Table 1: Associates

| Property | Value |
|---|---|
| **Display Name** | Associates |
| **Logical Name** | `cr63e_associates` |
| **Entity Set Name** | `cr63e_associateses` |
| **Primary Name Column** | `cr63e_associate` |
| **Description** | Staff members who use or manage fleet vehicles |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_associate` | Associate | Text (200) | **Yes** (Primary Name) | Associate's full name. Used for user matching via `User().FullName`. |
| `cr63e_location` | Location | Text (100) | Yes | Office/location assignment (e.g., "Durant", "McAlester"). Used for location-based vehicle filtering. |
| `cr63e_email` | Email | Text (200) | No | Email address. Optional if system email from Entra ID is used. |
| `cr63e_phone` | Phone | Text (20) | No | Phone number. |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

---

### Table 2: CFSVehicles

| Property | Value |
|---|---|
| **Display Name** | CFSVehicles |
| **Logical Name** | `cr63e_cfsvehicles` |
| **Entity Set Name** | `cr63e_cfsvehicleses` |
| **Primary Name Column** | `cr63e_vehiclename` |
| **Description** | Fleet vehicles managed by the organization |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_vehiclename` | Vehicle Name | Text (100) | **Yes** (Primary Name) | Vehicle display identifier (e.g., "CFS-001") |
| `cr63e_year` | Year | Text (4) | Yes | Vehicle model year |
| `cr63e_make` | Make | Text (100) | Yes | Vehicle manufacturer (e.g., "Ford", "Chevrolet") |
| `cr63e_model` | Model | Text (100) | Yes | Vehicle model (e.g., "F-150", "Tahoe") |
| `cr63e_tag` | Tag | Text (20) | Yes | License plate / tag number |
| `cr63e_vin` | VIN | Text (17) | No | Vehicle Identification Number (17 characters) |
| `cr63e_location` | Location | Text (100) | Yes | Assigned office/location. Must match values used in Associates table. |
| `cr63e_currentmileage` | Current Mileage | Whole Number | No | Current odometer reading in miles |
| `cr63e_status` | Status | Choice (Picklist) | Yes | Vehicle operational status. See option set below. |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

#### Choice: Status (`cr63e_status`)

| Value | Label | Color Usage in App |
|---|---|---|
| `360780000` | Available | Green (`RGBA(16, 124, 16, 1)`) |
| `360780001` | Reserved | Yellow (`RGBA(255, 185, 0, 1)`) |
| `360780002` | Checked Out | Red (`RGBA(209, 52, 56, 1)`) |
| `360780003` | Out Of Service | Dark Red (`RGBA(168, 0, 0, 1)`) |
| `360780004` | Maintenance | Gray (`RGBA(138, 136, 134, 1)`) |

---

### Table 3: Fuel Card Assignments

| Property | Value |
|---|---|
| **Display Name** | Fuel Card Assignments |
| **Logical Name** | `cr63e_fuelcardassignments` |
| **Entity Set Name** | `cr63e_fuelcardassignmentses` |
| **Primary Name Column** | `cr63e_name` |
| **Description** | Maps fuel cards to vehicles for automatic card-to-vehicle matching |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_name` | Name | Text (100) | **Yes** (Primary Name) | Card identifier / description |
| `cr63e_vehicle` | Vehicle | Lookup -> CFSVehicles | Yes | The vehicle this fuel card is assigned to |
| `cr63e_cardlast4` | CardLast4 | Text (4) | Yes | Last 4 digits of the fuel card number. Used for auto-matching on gas purchase screen. |
| `cr63e_isactive` | IsActive | Yes/No (Boolean) | Yes | Whether the card is currently active. `1` = Yes, `0` = No |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

---

### Table 4: Fuel Transactions

| Property | Value |
|---|---|
| **Display Name** | Fuel Transactions |
| **Logical Name** | `cr63e_fueltransactions` |
| **Entity Set Name** | `cr63e_fueltransactionses` |
| **Primary Name Column** | `cr63e_name` |
| **Description** | Records of fuel purchases linked to vehicles |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_name` | Name | Text (100) | **Yes** (Primary Name) | Transaction identifier (auto-generated or manual) |
| `cr63e_vehicle` | Vehicle | Lookup -> CFSVehicles | Yes | The vehicle that was fueled |
| `cr63e_transactiondatetime` | TransactionDateTime | Date and Time | Yes | Date and time of the fuel transaction |
| `cr63e_amount` | Amount | Currency / Decimal | Yes | Dollar amount of the purchase |
| `cr63e_gallons` | Gallons | Decimal | Yes | Number of gallons purchased |
| `cr63e_source` | Source | Choice (Picklist) | Yes | How the transaction was entered. See option set below. |
| `cr63e_notes` | Notes | Text (Multiline, 2000) | No | Optional notes about the transaction |
| `cr63e_submittedby` | SubmittedBy | Lookup -> Associates | No | The associate who submitted the transaction |
| `cr63e_receiptimage` | ReceiptImage | Image / File | No | Photo of the fuel receipt (captured via mobile camera) |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

#### Choice: Source (`cr63e_source`)

| Value | Label | Description |
|---|---|---|
| `360780000` | Manual | Entered manually by an associate via the mobile app |
| `360780001` | Import | Imported from external data (CSV, Excel, etc.) |
| `360780002` | VoyagerFeed | Automatically ingested from the Voyager fuel card system |

---

### Table 5: Vehicle Checkouts

| Property | Value |
|---|---|
| **Display Name** | Vehicle Checkouts |
| **Logical Name** | `cr63e_vehiclecheckouts` |
| **Entity Set Name** | `cr63e_vehiclecheckoutses` |
| **Primary Name Column** | `cr63e_name` |
| **Description** | Tracks when vehicles are physically checked out and returned |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_name` | Name | Text (100) | **Yes** (Primary Name) | Checkout identifier |
| `cr63e_vehicle` | Vehicle | Lookup -> CFSVehicles | Yes | The vehicle being checked out |
| `cr63e_associate` | Associate | Lookup -> Associates | Yes | The associate checking out the vehicle |
| `cr63e_checkedoutat` | CheckedOutAt | Date and Time | Yes | Timestamp when the vehicle was checked out |
| `cr63e_returnedat` | ReturnedAt | Date and Time | No | Timestamp when the vehicle was returned. **Blank indicates the vehicle is still checked out.** |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

> **Business Logic Note:** A checkout record with `ReturnedAt = Blank()` means the vehicle is currently out. The app uses `IsBlank(ReturnedAt)` to determine active checkouts and display "Checked out by: [name]" on the vehicle detail screen.

---

### Table 6: Vehicle Maintenances

| Property | Value |
|---|---|
| **Display Name** | Vehicle Maintenances |
| **Logical Name** | `cr63e_vehiclemaintenance` |
| **Entity Set Name** | `cr63e_vehiclemaintenances` |
| **Primary Name Column** | `cr63e_name` |
| **Description** | Maintenance requests and issue reports for vehicles |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_name` | Name | Text (100) | **Yes** (Primary Name) | Maintenance ticket identifier |
| `cr63e_vehicle` | Vehicle | Lookup -> CFSVehicles | Yes | The vehicle needing maintenance |
| `cr63e_type` | Type | Multi-Select Choice | Yes | One or more issue categories. See option set below. |
| `cr63e_description` | Description | Text (Multiline, 2000) | Yes | Detailed description of the issue |
| `cr63e_status` | Status | Choice (Picklist) | Yes | Current status of the maintenance ticket. See option set below. |
| `cr63e_outofservice` | OutOfService | Yes/No (Boolean) | Yes | Whether the vehicle should be pulled from service. `1` = Yes, `0` = No. When Yes, the app automatically sets the vehicle status to Maintenance. |
| `cr63e_reportedby` | ReportedBy | Lookup -> Associates | Yes | The associate who reported the issue |
| `cr63e_reporteddatetime` | ReportedDateTime | Date and Time | Yes | When the issue was reported |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

#### Multi-Select Choice: Type (`cr63e_type`)

| Value | Label |
|---|---|
| `360780000` | Oil change |
| `360780001` | Tire |
| `360780002` | Damage |
| `360780003` | Check engine light |
| `360780004` | Other |

#### Choice: Status (`cr63e_status`)

| Value | Label |
|---|---|
| `360780000` | Open |
| `360780001` | In Progress |
| `360780002` | Closed |

---

### Table 7: Vehicle Reservations

| Property | Value |
|---|---|
| **Display Name** | Vehicle Reservations |
| **Logical Name** | `cr63e_vehiclereservations` |
| **Entity Set Name** | `cr63e_vehiclereservationses` |
| **Primary Name Column** | `cr63e_name` |
| **Description** | Reservations for future vehicle usage |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_name` | Name | Text (100) | **Yes** (Primary Name) | Reservation identifier |
| `cr63e_vehicle` | Vehicle | Lookup -> CFSVehicles | Yes | The reserved vehicle |
| `cr63e_associate` | Associate | Lookup -> Associates | Yes | The associate requesting the reservation |
| `cr63e_startdatetime` | StartDateTime | Date and Time | Yes | Reservation start date/time |
| `cr63e_enddatetime` | EndDateTime | Date and Time | Yes | Reservation end date/time |
| `cr63e_purpose` | Purpose | Text (Multiline, 2000) | No | Trip purpose or notes |
| `cr63e_reservedby` | Reserved By | Text (200) | No | Name of the person who made the reservation (display text) |
| `cr63e_status` | Status | Choice (Picklist) | Yes | Current reservation status. See option set below. |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

#### Choice: Status (`cr63e_status`)

| Value | Label | Description |
|---|---|---|
| `360780000` | Draft | Reservation started but not submitted |
| `360780001` | Submitted | Reservation submitted, awaiting approval |
| `360780002` | Approved | Reservation approved and confirmed |
| `360780003` | CheckedOut | Vehicle has been physically checked out for this reservation |
| `360780004` | Returned | Vehicle has been returned after reservation use |
| `360780005` | Cancelled | Reservation cancelled by the associate or admin |
| `360780006` | NoShow | Associate did not pick up the vehicle |

---

## 3. Dataverse Table Schema -- New Tables for Admin App

The following tables are recommended additions to support admin app functionality. They follow the same `cr63e_` publisher prefix.

---

### Table 8: App Settings (NEW)

| Property | Value |
|---|---|
| **Display Name** | App Settings |
| **Logical Name** | `cr63e_appsettings` |
| **Entity Set Name** | `cr63e_appsettingses` |
| **Primary Name Column** | `cr63e_name` |
| **Description** | Configurable key-value settings for the application (locations, notification thresholds, defaults) |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_name` | Name | Text (200) | **Yes** (Primary Name) | Setting name / key (e.g., `DefaultLocation`, `MileageAlertThreshold`) |
| `cr63e_value` | Value | Text (2000) | Yes | Setting value (stored as text; parse as needed in the app) |
| `cr63e_category` | Category | Text (100) | No | Grouping category (e.g., "General", "Notifications", "Locations", "Thresholds") |
| `cr63e_description` | Description | Text (500) | No | Human-readable description of what this setting controls |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

#### Suggested Initial Records

| Name | Value | Category |
|---|---|---|
| `OfficeLoc_1` | Durant | Locations |
| `OfficeLoc_2` | McAlester | Locations |
| `OfficeLoc_3` | Poteau | Locations |
| `MileageAlertThreshold` | 100000 | Thresholds |
| `OilChangeIntervalMiles` | 5000 | Thresholds |
| `DefaultReservationStatus` | Approved | General |
| `RequireApproval` | false | General |
| `FuelCardProvider` | Voyager | General |

---

### Table 9: Activity Log (NEW -- Optional Enhancement)

| Property | Value |
|---|---|
| **Display Name** | Activity Log |
| **Logical Name** | `cr63e_activitylog` |
| **Entity Set Name** | `cr63e_activitylogs` |
| **Primary Name Column** | `cr63e_name` |
| **Description** | Audit trail for tracking admin actions and significant user activities |

#### Columns

| Column Name | Display Name | Data Type | Required | Description |
|---|---|---|---|---|
| `cr63e_name` | Name | Text (200) | **Yes** (Primary Name) | Auto-generated log entry identifier |
| `cr63e_action` | Action | Text (200) | Yes | Action performed (e.g., "Vehicle Status Changed", "Reservation Approved", "Fuel Import Completed") |
| `cr63e_performedby` | PerformedBy | Lookup -> Associates | Yes | The associate who performed the action |
| `cr63e_performedon` | PerformedOn | Date and Time | Yes | When the action occurred |
| `cr63e_details` | Details | Text (Multiline, 4000) | No | Additional context (e.g., "Changed CFS-012 from Available to Maintenance") |
| `cr63e_relatedrecord` | RelatedRecord | Text (200) | No | Identifier or GUID of the related record for cross-referencing |
| `cr63e_entityname` | EntityName | Text (100) | No | Which table was affected (e.g., "CFSVehicles", "Vehicle Reservations") |
| `cr63e_severity` | Severity | Choice (Picklist) | No | Log severity level. See option set below. |
| `statecode` | Status | State | Auto | `0` = Active, `1` = Inactive |
| `statuscode` | Status Reason | Status | Auto | `1` = Active, `2` = Inactive |

#### Choice: Severity (`cr63e_severity`) -- Suggested

| Value | Label |
|---|---|
| `360780000` | Info |
| `360780001` | Warning |
| `360780002` | Error |
| `360780003` | Critical |

---

## 4. Table Relationship Diagram

### Entity Relationship Map

```
                        +------------------+
                        |   Associates     |
                        |  (cr63e_         |
                        |   associates)    |
                        +--------+---------+
                                 |
              +------------------+-------------------+------------------+
              |                  |                   |                  |
              | cr63e_associate  | cr63e_submittedby | cr63e_reportedby |
              | (Lookup)         | (Lookup)          | (Lookup)         |
              |                  |                   |                  |
    +---------v-------+  +------v-----------+  +----v---------------+  |
    |   Vehicle       |  |  Fuel            |  |  Vehicle           |  |
    |   Checkouts     |  |  Transactions    |  |  Maintenances      |  |
    | (cr63e_vehicle  |  | (cr63e_fuel      |  | (cr63e_vehicle     |  |
    |  checkouts)     |  |  transactions)   |  |  maintenance)      |  |
    +---------+-------+  +------+-----------+  +----+---------------+  |
              |                 |                    |                  |
              | cr63e_vehicle   | cr63e_vehicle      | cr63e_vehicle   |
              | (Lookup)        | (Lookup)           | (Lookup)        |
              |                 |                    |                  |
              +--------+--------+--------------------+     cr63e_associate
                       |                                   | (Lookup)
                       v                                   |
              +------------------+                +--------v---------+
              |   CFSVehicles    |                |    Vehicle        |
              |  (cr63e_         |<------+        |    Reservations   |
              |   cfsvehicles)   |       |        |  (cr63e_vehicle   |
              +--------+---------+       |        |   reservations)   |
                       ^                 |        +------------------+
                       |                 |
                       | cr63e_vehicle   |
                       | (Lookup)        |
                       |                 |
              +--------+---------+       |
              | Fuel Card        |       |
              | Assignments      +-------+
              | (cr63e_fuelcard  |  cr63e_vehicle
              |  assignments)    |  (Lookup)
              +------------------+
```

### Lookup Relationships Summary

| Source Table | Lookup Column | Target Table | Relationship |
|---|---|---|---|
| Fuel Card Assignments | `cr63e_vehicle` | CFSVehicles | Many-to-One |
| Fuel Transactions | `cr63e_vehicle` | CFSVehicles | Many-to-One |
| Fuel Transactions | `cr63e_submittedby` | Associates | Many-to-One |
| Vehicle Checkouts | `cr63e_vehicle` | CFSVehicles | Many-to-One |
| Vehicle Checkouts | `cr63e_associate` | Associates | Many-to-One |
| Vehicle Maintenances | `cr63e_vehicle` | CFSVehicles | Many-to-One |
| Vehicle Maintenances | `cr63e_reportedby` | Associates | Many-to-One |
| Vehicle Reservations | `cr63e_vehicle` | CFSVehicles | Many-to-One |
| Vehicle Reservations | `cr63e_associate` | Associates | Many-to-One |
| Activity Log (NEW) | `cr63e_performedby` | Associates | Many-to-One |

### Key Business Relationships

- **CFSVehicles** is the central entity -- all transactional tables link back to it.
- **Associates** is the second hub -- all user-facing actions reference an associate.
- **Fuel Card Assignments** bridges fuel cards to vehicles, enabling automatic card-to-vehicle matching via last-4-digits lookup.
- **Vehicle Checkouts** use `ReturnedAt = Blank()` as the indicator for active checkouts (no separate status field).

---

## 5. Security Roles

Three security roles are recommended for the CFS Vehicle Usage system. All roles assume Organization-owned tables.

---

### Role 1: CFS Vehicle User

**Audience:** Field staff, associates, drivers

| Table | Create | Read | Update | Delete | Scope |
|---|---|---|---|---|---|
| Associates | -- | Org | -- | -- | Organization |
| CFSVehicles | -- | Org | User* | -- | *Own checkout/status changes only |
| Fuel Card Assignments | -- | Org | -- | -- | Organization |
| Fuel Transactions | User | Org | User | User | Own records |
| Vehicle Checkouts | User | Org | User | -- | Own records |
| Vehicle Maintenances | User | Org | User | -- | Own records |
| Vehicle Reservations | User | Org | User | User | Own records |
| App Settings | -- | Org | -- | -- | Organization (read-only) |
| Activity Log | -- | -- | -- | -- | No access |

**Permissions Notes:**
- Can view all vehicles and their status across all locations
- Can create and manage their own reservations, fuel transactions, and maintenance reports
- Can check out and check in vehicles (updates vehicle status to "Checked Out" / "Available")
- Can set a vehicle to "Maintenance" when reporting an out-of-service issue
- Cannot modify other users' records
- Cannot access admin settings or activity logs

---

### Role 2: CFS Vehicle Admin

**Audience:** Fleet managers, office managers, IT administrators

| Table | Create | Read | Update | Delete | Scope |
|---|---|---|---|---|---|
| Associates | Org | Org | Org | Org | Organization |
| CFSVehicles | Org | Org | Org | Org | Organization |
| Fuel Card Assignments | Org | Org | Org | Org | Organization |
| Fuel Transactions | Org | Org | Org | Org | Organization |
| Vehicle Checkouts | Org | Org | Org | Org | Organization |
| Vehicle Maintenances | Org | Org | Org | Org | Organization |
| Vehicle Reservations | Org | Org | Org | Org | Organization |
| App Settings | Org | Org | Org | Org | Organization |
| Activity Log | Org | Org | Org | Org | Organization |

**Permissions Notes:**
- Full CRUD on all tables
- Can manage associates (add, deactivate, update locations)
- Can approve, reject, or cancel any reservation
- Can import fuel transactions (bulk operations)
- Can manage fuel card assignments
- Can close maintenance tickets
- Can modify app settings
- Can view complete activity logs

---

### Role 3: CFS Vehicle Viewer

**Audience:** Supervisors, reporting users, auditors

| Table | Create | Read | Update | Delete | Scope |
|---|---|---|---|---|---|
| Associates | -- | Org | -- | -- | Organization |
| CFSVehicles | -- | Org | -- | -- | Organization |
| Fuel Card Assignments | -- | Org | -- | -- | Organization |
| Fuel Transactions | -- | Org | -- | -- | Organization |
| Vehicle Checkouts | -- | Org | -- | -- | Organization |
| Vehicle Maintenances | -- | Org | -- | -- | Organization |
| Vehicle Reservations | -- | Org | -- | -- | Organization |
| App Settings | -- | Org | -- | -- | Organization |
| Activity Log | -- | Org | -- | -- | Organization |

**Permissions Notes:**
- Read-only access to all tables
- Suitable for reporting dashboards, auditing, and oversight
- Cannot create, modify, or delete any records

---

## 6. Step-by-Step Deployment Instructions

---

### Phase 1: Dataverse Environment Setup

#### Step 1.1: Create or Select a Dataverse Environment

1. Navigate to the [Power Platform Admin Center](https://admin.powerplatform.microsoft.com).
2. Select **Environments** from the left navigation.
3. Either use an existing environment or click **+ New** to create one:
   - **Name:** `CFS Vehicle Usage` (or your preferred name)
   - **Type:** Production (or Sandbox for testing)
   - **Region:** United States (or your region)
   - **Create a database:** Yes
   - **Language:** English
   - **Currency:** USD
4. Wait for provisioning to complete (5-15 minutes).

#### Step 1.2: Create Tables

For each of the 7 existing tables (and optionally 2 new tables), follow this process:

1. Navigate to [make.powerapps.com](https://make.powerapps.com).
2. Select the correct environment from the environment picker (top right).
3. Go to **Tables** in the left navigation.
4. Click **+ New table** > **New table**.

**Create tables in this order** (to satisfy lookup dependencies):

1. **Associates** (no lookups -- create first)
2. **CFSVehicles** (no lookups)
3. **Fuel Card Assignments** (looks up CFSVehicles)
4. **Fuel Transactions** (looks up CFSVehicles and Associates)
5. **Vehicle Checkouts** (looks up CFSVehicles and Associates)
6. **Vehicle Maintenances** (looks up CFSVehicles and Associates)
7. **Vehicle Reservations** (looks up CFSVehicles and Associates)
8. **App Settings** (optional, no lookups)
9. **Activity Log** (optional, looks up Associates)

#### Step 1.3: Create Columns for Each Table

For each table, add columns as specified in Section 2 above. Key instructions:

**For Text columns:**
1. Click **+ New column** in the table designer.
2. Enter the Display Name.
3. Set Data Type to **Single line of text** (or **Multiple lines of text** for Multiline fields).
4. Set the maximum length as noted.
5. Set Required as noted.

**For Choice (Picklist) columns:**
1. Click **+ New column**.
2. Set Data Type to **Choice**.
3. Select **New choice** to create a local option set.
4. Add each option with the exact label. Dataverse will auto-assign values, but if you need specific values (for consistency with the app source), use the advanced option to set custom values matching the `360780xxx` pattern documented above.

**For Multi-Select Choice columns (e.g., Maintenance Type):**
1. Follow the same process as Choice, but select **Choices** (plural) instead of **Choice** to enable multi-select.

**For Lookup columns:**
1. Click **+ New column**.
2. Set Data Type to **Lookup**.
3. In the **Related table** dropdown, select the target table.
4. Save.

**For Yes/No (Boolean) columns:**
1. Click **+ New column**.
2. Set Data Type to **Yes/No**.
3. Set the default value as appropriate.

**For Date and Time columns:**
1. Click **+ New column**.
2. Set Data Type to **Date and Time**.
3. Choose **Date and time** (not Date only) for timestamp fields.

**For Whole Number columns (e.g., Current Mileage):**
1. Click **+ New column**.
2. Set Data Type to **Whole number**.
3. Set minimum (0) and maximum values as appropriate.

**For Currency/Decimal columns (e.g., Amount, Gallons):**
1. Click **+ New column**.
2. Set Data Type to **Currency** for dollar amounts or **Decimal** for gallons.
3. Set precision as needed (2 decimal places for currency, 3 for gallons).

**For Image/File columns (e.g., ReceiptImage):**
1. Click **+ New column**.
2. Set Data Type to **Image** or **File**.
3. For receipt photos, **Image** is recommended (provides thumbnail preview).

#### Step 1.4: Create Choice/Option Sets

If you prefer global (reusable) option sets instead of local ones, create them before adding columns:

1. Navigate to **Tables** > **Choices** (under Data section).
2. Click **+ New choice**.
3. Create the following global choices:

| Choice Name | Options |
|---|---|
| Vehicle Status | Available, Reserved, Checked Out, Out Of Service, Maintenance |
| Fuel Source | Manual, Import, VoyagerFeed |
| Maintenance Type | Oil change, Tire, Damage, Check engine light, Other |
| Maintenance Status | Open, In Progress, Closed |
| Reservation Status | Draft, Submitted, Approved, CheckedOut, Returned, Cancelled, NoShow |
| Log Severity | Info, Warning, Error, Critical |

#### Step 1.5: Configure Table Relationships

Relationships are automatically created when you add Lookup columns. Verify them:

1. Open each table in the table designer.
2. Click the **Relationships** tab.
3. Confirm all Many-to-One relationships match the diagram in Section 4.

#### Step 1.6: Create Security Roles

1. Go to the [Power Platform Admin Center](https://admin.powerplatform.microsoft.com).
2. Select your environment > **Settings** > **Users + permissions** > **Security roles**.
3. Click **+ New role** for each of the three roles defined in Section 5.
4. On the **Custom Entities** tab, configure Create/Read/Write/Delete permissions for each table as specified.
5. Save each role.
6. Assign roles to users via **Settings** > **Users + permissions** > **Users**.

#### Step 1.7: Add Sample Data

Populate the system with sample data for testing. At minimum:

**Associates (3-5 records):**

| Associate | Location | Email | Phone |
|---|---|---|---|
| John Smith | Durant | jsmith@cfs.org | 555-0101 |
| Jane Doe | McAlester | jdoe@cfs.org | 555-0102 |
| Bob Johnson | Durant | bjohnson@cfs.org | 555-0103 |
| Mary Williams | Poteau | mwilliams@cfs.org | 555-0104 |

**CFSVehicles (5-8 records):**

| Vehicle Name | Year | Make | Model | Tag | Location | Status | Current Mileage |
|---|---|---|---|---|---|---|---|
| CFS-001 | 2023 | Ford | F-150 | ABC-1234 | Durant | Available | 15230 |
| CFS-002 | 2022 | Chevrolet | Tahoe | DEF-5678 | Durant | Available | 28450 |
| CFS-003 | 2024 | Ford | Explorer | GHI-9012 | McAlester | Available | 8100 |
| CFS-004 | 2021 | Toyota | Camry | JKL-3456 | Poteau | Maintenance | 45600 |
| CFS-005 | 2023 | Chevrolet | Equinox | MNO-7890 | Durant | Available | 12300 |

**Fuel Card Assignments (3-5 records):**

| Name | Vehicle | CardLast4 | IsActive |
|---|---|---|---|
| Voyager Card - CFS-001 | CFS-001 | 4421 | Yes |
| Voyager Card - CFS-002 | CFS-002 | 7788 | Yes |
| Voyager Card - CFS-003 | CFS-003 | 3355 | Yes |

---

### Phase 2: Mobile App Deployment

#### Step 2.1: Import the Canvas App from Source

**Option A: Import from .msapp package (Recommended)**

1. If you have an exported `.msapp` file:
   - Go to [make.powerapps.com](https://make.powerapps.com).
   - Select the target environment.
   - Click **Apps** > **Import canvas app**.
   - Upload the `.msapp` file.
   - Follow the import wizard to map data connections.

**Option B: Import from Solution**

1. Package the app into a Dataverse solution:
   - Go to **Solutions** in the left navigation.
   - Open or create a solution (e.g., `CFSVehicleUsage`).
   - Click **Add existing** > **App** > **Canvas app**.
   - Select the CFS Tribal Vehicle Usage app.
   - Export the solution as managed or unmanaged.
2. In the target environment:
   - Go to **Solutions** > **Import solution**.
   - Upload the solution `.zip` file.
   - Follow the import wizard.

**Option C: Recreate from YAML Source (Advanced)**

1. Use the Power Apps CLI (`pac`) to unpack/pack the source:
   ```bash
   # Install Power Apps CLI
   dotnet tool install --global Microsoft.PowerApps.CLI.Tool

   # Authenticate to your environment
   pac auth create --environment https://org7c6cb60f.crm.dynamics.com

   # Pack the YAML source into an .msapp file
   pac canvas pack --sources "CFS Tribal Vehicle Usage (9)/Src" --msapp CFS_Tribal_Vehicle_Usage.msapp
   ```
2. Upload the resulting `.msapp` file via Power Apps Studio.

#### Step 2.2: Connect Data Sources

After import, verify data connections:

1. Open the app in **Power Apps Studio** (Edit mode).
2. Click **Data** in the left panel.
3. Verify all 7 Dataverse tables are connected:
   - Associates
   - CFSVehicles
   - Fuel Card Assignments
   - Fuel Transactions
   - Vehicle Checkouts
   - Vehicle Maintenances
   - Vehicle Reservations
4. If any are missing, click **+ Add data** > **Dataverse** and select the table.
5. If connecting to a different environment, you may need to update the connection reference.

#### Step 2.3: Verify App Configuration

1. Check `App.OnStart` to ensure the user-matching logic works:
   ```
   Set(varUserAssociate, LookUp(Associates, Associate = varUserDisplayName));
   ```
   - This matches `User().FullName` to the `Associate` column value.
   - Ensure associate names in Dataverse exactly match Entra ID display names.

2. Review the app layout settings (already configured):
   - **Layout:** Landscape (1366 x 768)
   - **Scale to fit:** Off
   - **Lock orientation:** Off
   - **Show status bar:** Off
   - **App type:** Desktop or Tablet

3. If deploying for phones specifically, consider adjusting to Portrait layout (Phone form factor).

#### Step 2.4: Publish the App

1. In Power Apps Studio, click **File** > **Save** (or Ctrl+S).
2. Click **Publish**.
3. Confirm the publish dialog.

#### Step 2.5: Share the App

1. Go to [make.powerapps.com](https://make.powerapps.com) > **Apps**.
2. Find the CFS Tribal Vehicle Usage app.
3. Click the **...** menu > **Share**.
4. Add users or security groups:
   - For all field staff: share with the appropriate Entra ID security group.
   - Assign the **CFS Vehicle User** security role.
5. Optionally send an email notification.

#### Step 2.6: Mobile App Distribution

For field staff to use on their phones/tablets:

1. Have users install **Power Apps** from:
   - [iOS App Store](https://apps.apple.com/app/power-apps/id702033961)
   - [Google Play Store](https://play.google.com/store/apps/details?id=com.microsoft.msapps)
2. Users sign in with their organizational Entra ID credentials.
3. The shared app will appear in their app list.
4. Users can also pin the app to their home screen for quick access.

---

### Phase 3: Admin App Deployment

#### Step 3.1: Create the Admin Canvas App

The admin app is a separate Canvas App optimized for desktop use.

1. Go to [make.powerapps.com](https://make.powerapps.com).
2. Click **+ Create** > **Blank app** > **Blank canvas app**.
3. Configure:
   - **Name:** CFS Vehicle Admin
   - **Format:** Tablet (landscape, 1366 x 768)
4. Click **Create**.

#### Step 3.2: Connect Data Sources

Add all Dataverse tables:

1. Click **Data** in the left panel.
2. Click **+ Add data** > **Dataverse**.
3. Add all 9 tables:
   - Associates
   - CFSVehicles
   - Fuel Card Assignments
   - Fuel Transactions
   - Vehicle Checkouts
   - Vehicle Maintenances
   - Vehicle Reservations
   - App Settings (new)
   - Activity Log (new)

#### Step 3.3: Recommended Admin App Screens

Build the following screens for the admin app:

| Screen | Purpose |
|---|---|
| **Dashboard** | Fleet overview with charts: vehicles by status, fuel spend trends, active checkouts, open maintenance tickets |
| **Vehicle Management** | CRUD for vehicles: add/edit/deactivate vehicles, manage status, view history |
| **Associate Management** | CRUD for associates: add/edit/deactivate, assign locations |
| **Reservation Management** | View all reservations, approve/reject, bulk operations, conflict detection |
| **Fuel Management** | View all fuel transactions, import fuel data (CSV), manage fuel card assignments |
| **Maintenance Management** | Triage open tickets, assign priority, close tickets, generate work orders |
| **Reports** | Fuel cost by vehicle/month, utilization rates, mileage tracking, maintenance history |
| **Settings** | Manage App Settings table, configure office locations, notification thresholds |
| **Activity Log Viewer** | Browse/search/filter activity log entries with date range and entity filters |

#### Step 3.4: Implement Admin-Specific Features

**Reservation Approval Workflow:**
```
// Approve a reservation
Patch(
    'Vehicle Reservations',
    selectedReservation,
    {'Status (cr63e_status)': 'Status (Vehicle Reservations)'.Approved}
);

// Log the action
Patch(
    'Activity Log',
    Defaults('Activity Log'),
    {
        Action: "Reservation Approved",
        PerformedBy: varAdminAssociate,
        PerformedOn: Now(),
        Details: "Approved reservation " & selectedReservation.Name & " for " & selectedReservation.Vehicle.'Vehicle Name',
        RelatedRecord: Text(selectedReservation.'Vehicle Reservations'),
        EntityName: "Vehicle Reservations"
    }
);
```

**Fuel Data Import Pattern:**
```
// After loading CSV data into a collection (colImportData):
ForAll(
    colImportData,
    Patch(
        'Fuel Transactions',
        Defaults('Fuel Transactions'),
        {
            Vehicle: LookUp(CFSVehicles, 'Vehicle Name' = ThisRecord.VehicleName),
            TransactionDateTime: DateTimeValue(ThisRecord.TransactionDate),
            Amount: Value(ThisRecord.Amount),
            Gallons: Value(ThisRecord.Gallons),
            Source: 'Source (Fuel Transactions)'.Import,
            Notes: "Imported on " & Text(Now(), "yyyy-mm-dd")
        }
    )
);
```

#### Step 3.5: Publish for Desktop Use

1. Save and publish the admin app.
2. Go to **Apps** > find the admin app > **...** > **Share**.
3. Share only with fleet managers and administrators.
4. Assign the **CFS Vehicle Admin** security role.

Desktop users can access the admin app via:
- Direct URL: `https://apps.powerapps.com/play/e/<app-id>`
- The Power Apps portal at [make.powerapps.com](https://make.powerapps.com)
- Bookmarking the app URL in their browser

---

## 7. User Guide

---

### 7.1 Mobile App -- Features and Navigation

#### Bottom Navigation Bar

The mobile app uses a persistent 5-tab bottom navigation bar:

| Tab | Screen | Description |
|---|---|---|
| **Home** | Fleet Dashboard | Vehicle status counts, quick actions, upcoming reservations, KPIs |
| **Vehicles** | Vehicle Fleet | Browse and search all fleet vehicles |
| **Reserve** | New Reservation | Quick-create a new vehicle reservation |
| **Gas** | Log Gas Purchase | Log a fuel purchase with receipt |
| **More** | Settings / Links | Profile, quick links, app version |

#### Home Screen (Fleet Dashboard)

The dashboard displays:

- **Status Cards (2x2 grid):** Available, On Road (Checked Out), Reserved, Maintenance -- each showing a count. Tap any card to filter the vehicle list.
- **Quick Actions:** Four buttons for Reserve, Gas, All Vehicles, and Report Issue.
- **My Upcoming Reservations:** The next 3 upcoming reservations with vehicle name, date, and countdown. Tap "See All" to view all reservations.
- **This Month's Snapshot:** Monthly fuel spend, active checkout count, and open maintenance ticket count.
- **Fleet Total:** Summary bar showing total fleet size and available count.

#### Vehicle Fleet Screen

- **Search bar:** Filter vehicles by name.
- **Location toggle:** Switch between "My Location" (filtered by user's assigned office) and "All Locations".
- **Status filter:** When navigating from a dashboard card, the list is pre-filtered by that status.
- **Vehicle rows:** Each row shows vehicle name, year/make/model, tag number, and color-coded status indicator.
- **Tap a vehicle:** Navigate to Vehicle Detail.

#### Vehicle Detail Screen

- **Status banner:** Color-coded banner showing current vehicle status.
- **Vehicle information card:** Tag, VIN, assigned office, fuel card (last 4), current mileage.
- **Checked-out-by indicator:** Shows who has the vehicle when status is "Checked Out".
- **Action buttons:**
  - **Reserve** (when Available): Create a reservation for this vehicle.
  - **Check Out** (when Available): Immediately check out the vehicle and set status to "Checked Out".
  - **Check In** (when Checked Out): Return the vehicle and set status to "Available".
  - **Set Maintenance** (when not already in Maintenance): Pull the vehicle from service.
  - **Clear Maintenance** (when in Maintenance): Return the vehicle to "Available".
- **Upcoming Reservations:** List of future reservations for this vehicle.

---

### 7.2 Admin App -- Features and Navigation (Planned)

The admin app provides fleet managers with comprehensive management capabilities:

| Feature | Description |
|---|---|
| **Fleet Dashboard** | Charts and metrics: utilization rates, fuel spend trends, maintenance backlogs |
| **Vehicle CRUD** | Add new vehicles, edit details, deactivate retired vehicles, manage status |
| **Associate Management** | Add/deactivate associates, assign office locations, view usage history |
| **Reservation Approval** | Review submitted reservations, approve/reject with notes, handle conflicts |
| **Fuel Import** | Bulk-import fuel transactions from Voyager feed or CSV files |
| **Fuel Card Management** | Assign/reassign fuel cards to vehicles, activate/deactivate cards |
| **Maintenance Triage** | Review open tickets, assign priority, update status, close with resolution notes |
| **Reporting** | Fuel cost per vehicle, monthly utilization, mileage trends, maintenance cost analysis |
| **Settings Management** | Configure office locations, thresholds, and app-wide settings |
| **Activity Audit** | Browse/search all logged admin actions with date and entity filters |

---

### 7.3 Common Workflows

---

#### Workflow 1: Reserve a Vehicle

**Actor:** Field staff (mobile app)

1. Open the app. From the Home screen, tap **+ Reserve** (quick action) or tap the **Reserve** tab.
2. On the New Reservation screen:
   - Select a vehicle from the dropdown (shows active vehicles).
   - Set the **Start Date** and **End Date** using the date pickers.
   - Enter a **Purpose** / trip notes (optional).
   - If the selected vehicle has overlapping reservations, a yellow warning appears.
3. Tap **Submit Reservation**.
4. If the start date is today, the vehicle status is automatically set to "Reserved".
5. The app navigates to My Reservations, showing the new reservation.

**To cancel a reservation:**
1. Go to **More** > **My Reservations** (or tap "See All" on the Home screen).
2. Filter to "Upcoming".
3. Tap **Cancel** on the reservation row.
4. Confirm in the dialog.

---

#### Workflow 2: Log a Gas Purchase

**Actor:** Field staff (mobile app)

1. Tap the **Gas** tab or the **Gas** quick action from Home.
2. On the Log Gas Purchase screen:
   - Enter the **Fuel Card Last 4 Digits**. The app automatically looks up the matching vehicle.
   - A green checkmark appears with the matched vehicle name, or a yellow warning if no match is found.
   - Set the **Transaction Date**.
   - Enter the **Amount ($)** and **Gallons**.
   - The price-per-gallon is calculated and displayed automatically.
   - Add optional **Notes**.
   - Tap the **Capture Receipt** button to take a photo of the receipt.
3. Tap **Submit Gas Purchase**.
4. The transaction is saved with source = "Manual" and the app navigates back to Home.

---

#### Workflow 3: Report a Maintenance Issue

**Actor:** Field staff (mobile app)

1. From the Home screen, tap **Issue** (quick action), or from a Vehicle Detail screen, the app pre-selects the vehicle.
2. On the Report Issue screen:
   - Select the **Vehicle** (or confirm the pre-selected one).
   - Select the **Issue Type** from the multi-select dropdown (Oil change, Tire, Damage, Check engine light, Other).
   - Set **Urgency** (Low, Medium, High, Critical).
   - Choose whether to **Pull from service** (Yes/No toggle).
   - Enter a **Description** of the issue.
3. Tap **Submit Issue Report**.
4. If "Pull from service" = Yes, the vehicle status is automatically changed to "Maintenance".
5. The maintenance ticket is created with status = "Open".

---

#### Workflow 4: Check Out and Return a Vehicle

**Actor:** Field staff (mobile app)

**Check Out:**
1. Navigate to the vehicle list. Tap a vehicle with "Available" status.
2. On the Vehicle Detail screen, tap **Check Out**.
3. A checkout record is created with the current timestamp.
4. The vehicle status changes to "Checked Out".

**Check In / Return:**
1. Navigate to the vehicle (now showing "Checked Out" status).
2. Tap **Check In Vehicle**.
3. The open checkout record is updated with `ReturnedAt = Now()`.
4. The vehicle status changes back to "Available".

---

#### Workflow 5: Admin Reservation Approval (Admin App)

**Actor:** Fleet manager (admin app)

1. Open the Admin App on desktop.
2. Navigate to **Reservation Management**.
3. Filter by status = "Submitted".
4. Review the reservation details: vehicle, associate, dates, purpose.
5. Check for scheduling conflicts with other reservations on the same vehicle.
6. Click **Approve** or **Reject**.
7. The reservation status updates and an activity log entry is created.
8. Optionally, a Power Automate flow can send an email notification to the associate.

---

#### Workflow 6: Fuel Data Import (Admin App)

**Actor:** Fleet manager (admin app)

1. Open the Admin App on desktop.
2. Navigate to **Fuel Management** > **Import**.
3. Upload a CSV file from the Voyager fuel card system.
4. Map CSV columns to Dataverse fields (Vehicle Name, Date, Amount, Gallons).
5. Preview the import data and resolve any vehicle-name mismatches.
6. Click **Import**.
7. All records are created with source = "Import" or "VoyagerFeed".
8. An activity log entry records the bulk import operation.

---

## Appendix

### A. Theme Configuration Reference

The mobile app uses a centralized theme defined in `App.OnStart`. To customize colors:

```
// Main theme object
varTheme = {
    Primary:        RGBA(0, 120, 212, 1),     // Microsoft Blue
    PrimaryDark:    RGBA(0, 90, 158, 1),
    PrimaryLight:   RGBA(199, 224, 244, 1),
    Success:        RGBA(16, 124, 16, 1),      // Green
    Warning:        RGBA(255, 185, 0, 1),      // Yellow/Amber
    Danger:         RGBA(209, 52, 56, 1),      // Red
    Background:     RGBA(243, 242, 241, 1),    // Light Gray
    Surface:        RGBA(255, 255, 255, 1),    // White
    TextPrimary:    RGBA(50, 49, 48, 1),       // Dark Gray
    TextSecondary:  RGBA(96, 94, 92, 1),       // Medium Gray
    TextOnPrimary:  RGBA(255, 255, 255, 1),    // White text on colored bg
    Border:         RGBA(237, 235, 233, 1)     // Light border
}

// Status-specific colors
varStatusColors = {
    Available:    RGBA(16, 124, 16, 1),   // Green
    CheckedOut:   RGBA(209, 52, 56, 1),   // Red
    Reserved:     RGBA(255, 185, 0, 1),   // Yellow
    Warning:      RGBA(255, 185, 0, 1),   // Yellow
    Maintenance:  RGBA(138, 136, 134, 1), // Gray
    Draft:        RGBA(0, 120, 212, 1),   // Blue
    Submitted:    RGBA(0, 120, 212, 1),   // Blue
    Approved:     RGBA(16, 124, 16, 1),   // Green
    Cancelled:    RGBA(209, 52, 56, 1),   // Red
    OutOfService: RGBA(168, 0, 0, 1)      // Dark Red
}
```

### B. Global Variables Reference

| Variable | Type | Set In | Description |
|---|---|---|---|
| `varCurrentUser` | User | App.OnStart | Current Power Apps user object |
| `varUserEmail` | Text | App.OnStart | User's email from Entra ID |
| `varUserDisplayName` | Text | App.OnStart | User's full name from Entra ID |
| `varUserAssociate` | Record | App.OnStart | Matched Associates table record |
| `varTheme` | Record | App.OnStart | Theme color palette |
| `varStatusColors` | Record | App.OnStart | Status-specific colors |
| `varSelectedVehicle` | Record | Various | Currently selected vehicle for detail view |
| `varFilterStatus` | Text | Various | Active vehicle list filter ("All", "Available", etc.) |
| `varShowAllLocations` | Boolean | scrVehicleList | Whether to show vehicles from all locations |
| `varIsSubmitting` | Boolean | Various | Loading state during form submissions |
| `varIsNewReservation` | Boolean | Various | Whether reservation form is in create mode |
| `varResStartDate` | Date | scrReservation | Reservation start date |
| `varResEndDate` | Date | scrReservation | Reservation end date |
| `varResPurpose` | Text | scrReservation | Reservation purpose text |
| `varResVehicleSelected` | Record | scrReservation | Vehicle selected for reservation |
| `varResFilter` | Text | scrMyReservations | Reservation list filter ("Upcoming", "Past", "All") |
| `varGasAmount` | Number | scrGasPurchase | Fuel transaction dollar amount |
| `varGasGallons` | Number | scrGasPurchase | Fuel transaction gallons |
| `varGasDate` | Date | scrGasPurchase | Fuel transaction date |
| `varGasCardLast4` | Text | scrGasPurchase | Entered fuel card last 4 digits |
| `varGasVehicle` | Record | scrGasPurchase | Matched vehicle from fuel card lookup |
| `varGasNotes` | Text | scrGasPurchase | Fuel transaction notes |
| `varMaintVehicle` | Record | scrMaintenanceRequest | Vehicle for maintenance report |
| `varMaintDescription` | Text | scrMaintenanceRequest | Maintenance issue description |
| `varMaintUrgency` | Text | scrMaintenanceRequest | Urgency level |
| `varMaintOutOfService` | Boolean | scrMaintenanceRequest | Whether to pull vehicle from service |
| `varFuelHistoryFilter` | Text | scrFuelHistory | Fuel history filter ("Mine", "All") |
| `varFuelSpendMonth` | Number | scrHome | Current month's total fuel spend |
| `varActiveCheckouts` | Number | scrHome | Count of currently checked-out vehicles |
| `varOpenMaintenance` | Number | scrHome | Count of open/in-progress maintenance tickets |
| `varConfirmCancelRes` | Record | scrMyReservations | Reservation pending cancellation confirmation |
| `varViewReceipt` | Record | scrFuelHistory | Fuel transaction selected for receipt viewing |
| `varVehicleFuelCard` | Record | scrVehicleDetail | Fuel card assigned to the selected vehicle |

### C. Collections Reference

| Collection | Screen | Source | Description |
|---|---|---|---|
| `colMyUpcoming` | scrHome | Vehicle Reservations | User's next 3 upcoming reservations |
| `colMyReservations` | scrMyReservations | Vehicle Reservations | All of the user's reservations (sorted by date descending) |
| `colVehicleReservations` | scrVehicleDetail | Vehicle Reservations | Upcoming reservations for the selected vehicle |
| `colFuelCards` | scrGasPurchase | Fuel Card Assignments | All fuel card assignments (for card-to-vehicle matching) |
| `colFuelHistory` | scrFuelHistory | Fuel Transactions | Fuel transactions (filtered by user or all, sorted by date descending) |

### D. Dataverse Views (Pre-configured)

Each table has standard Dataverse views created automatically:

| Table | Views |
|---|---|
| Associates | Active Associates, Inactive Associates, Lookup View, Advanced Find View, Associated View |
| CFSVehicles | Active CFSVehicles, Inactive CFSVehicles, Lookup View, Advanced Find View, Associated View |
| Fuel Card Assignments | Active, Inactive, Lookup View, Advanced Find View, Associated View |
| Fuel Transactions | Active, Inactive, Lookup View, Advanced Find View, Associated View |
| Vehicle Checkouts | Active, Inactive, Lookup View, Advanced Find View, Associated View |
| Vehicle Maintenances | Active, Inactive, Lookup View, Advanced Find View, Associated View |
| Vehicle Reservations | Active, Inactive, Lookup View, Advanced Find View, Associated View |

### E. Troubleshooting

| Issue | Cause | Resolution |
|---|---|---|
| User sees no data on Home screen | `varUserAssociate` is blank; user's Entra ID display name does not match any Associates record | Ensure the user's `FullName` in Entra ID exactly matches an `Associate` value in the Associates table |
| Vehicle list shows no vehicles | Location filter is active and user's location does not match any vehicles | Toggle "All Locations" on the Vehicle Fleet screen, or verify the user's Associate record has the correct Location value |
| Fuel card not matching | Card last 4 digits entered do not match any Fuel Card Assignment record | Verify the `CardLast4` value in Fuel Card Assignments matches exactly (case-sensitive, no spaces) |
| "Failed to save" error on form submission | Network connectivity issue or Dataverse permission denied | Check internet connection; verify the user has the CFS Vehicle User security role |
| Vehicle status not updating after checkout | Concurrent update or stale cache | Refresh the vehicle list (navigate away and back); check if another user modified the record |
| Receipt image not displaying | Image control not fully configured in Power Apps Studio | In Studio, add a proper Image control and set its `Image` property to `varViewReceipt.ReceiptImage` |

---

**End of Setup and Deployment Guide**
