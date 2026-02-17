# CFS Tribal Vehicle Usage - Dataverse Setup Guide

## Overview

This guide documents the complete Dataverse table schema required for both the **Mobile User App** and the **Admin Desktop App**. All tables use the solution publisher prefix `cr63e_`.

The environment is: `org7c6cb60f.crm.dynamics.com`

---

## EXISTING TABLES (Already Created)

These tables should already exist in your Dataverse environment. Verify they match the schemas below.

### 1. Associates (`cr63e_associates`)

| Column Display Name | Schema Name | Data Type | Required | Notes |
|---|---|---|---|---|
| Associate | `cr63e_associate` | Single Line of Text | Yes | Primary name column (person's full name) |
| Location | `cr63e_location` | Single Line of Text | No | Office/site location |
| Email | `cr63e_email` | Single Line of Text (Email) | No | Work email address |

**Option Sets:**
- Status (statecode): Active (0), Inactive (1)
- Status Reason (statuscode): Active (1), Inactive (2)

---

### 2. CFS Vehicles (`cr63e_cfsvehicles`)

| Column Display Name | Schema Name | Data Type | Required | Notes |
|---|---|---|---|---|
| Vehicle Name | `cr63e_vehiclename` | Single Line of Text | Yes | Primary name (e.g., "Unit 101 - Tahoe") |
| Year | `cr63e_year` | Single Line of Text | No | Model year |
| Make | `cr63e_make` | Single Line of Text | No | Manufacturer (Chevrolet, Ford, etc.) |
| Model | `cr63e_model` | Single Line of Text | No | Model name (Tahoe, F-150, etc.) |
| Tag | `cr63e_tag` | Single Line of Text | No | License plate / fleet tag number |
| VIN | `cr63e_vin` | Single Line of Text | No | 17-character Vehicle Identification Number |
| Location | `cr63e_location` | Single Line of Text | No | Assigned office/site |
| Current Mileage | `cr63e_currentmileage` | Whole Number | No | Odometer reading |
| Status | `cr63e_status` | Choice (Option Set) | Yes | Vehicle availability status |

**Status Option Set (`cr63e_status`):**
| Value | Label |
|---|---|
| 360780000 | Available |
| 360780001 | Reserved |
| 360780002 | Checked Out |
| 360780003 | Out Of Service |
| 360780004 | Maintenance |

---

### 3. Fuel Card Assignments (`cr63e_fuelcardassignments`)

| Column Display Name | Schema Name | Data Type | Required | Notes |
|---|---|---|---|---|
| Card Last 4 | `cr63e_cardlast4` | Single Line of Text | Yes | Last 4 digits of the fuel card |
| Vehicle | `cr63e_vehicle` | Lookup (CFS Vehicles) | No | Vehicle this card is assigned to |
| IsActive | `cr63e_isactive` | Two Options (Boolean) | No | Yes (1) / No (0) |

---

### 4. Fuel Transactions (`cr63e_fueltransactions`)

| Column Display Name | Schema Name | Data Type | Required | Notes |
|---|---|---|---|---|
| Vehicle | `cr63e_vehicle` | Lookup (CFS Vehicles) | Yes | Vehicle fueled |
| Transaction Date Time | `cr63e_transactiondatetime` | Date and Time | Yes | When the purchase occurred |
| Amount | `cr63e_amount` | Currency / Decimal | Yes | Dollar amount |
| Gallons | `cr63e_gallons` | Decimal Number | Yes | Gallons purchased |
| Source | `cr63e_source` | Choice (Option Set) | No | How the transaction was entered |
| Notes | `cr63e_notes` | Multiple Lines of Text | No | Additional notes |
| Submitted By | `cr63e_submittedby` | Lookup (Associates) | No | Person who entered the record |
| Receipt Image | `cr63e_receiptimage` | Image / File | No | Photo of receipt |

**Source Option Set (`cr63e_source`):**
| Value | Label |
|---|---|
| 360780000 | Manual |
| 360780001 | Import |
| 360780002 | VoyagerFeed |

---

### 5. Vehicle Checkouts (`cr63e_vehiclecheckouts`)

| Column Display Name | Schema Name | Data Type | Required | Notes |
|---|---|---|---|---|
| Vehicle | `cr63e_vehicle` | Lookup (CFS Vehicles) | Yes | Vehicle being checked out |
| Associate | `cr63e_associate` | Lookup (Associates) | Yes | Person checking out |
| Checked Out At | `cr63e_checkedoutat` | Date and Time | Yes | Checkout timestamp |
| Returned At | `cr63e_returnedat` | Date and Time | No | Return timestamp (blank = still out) |

---

### 6. Vehicle Maintenances (`cr63e_vehiclemaintenance`)

| Column Display Name | Schema Name | Data Type | Required | Notes |
|---|---|---|---|---|
| Vehicle | `cr63e_vehicle` | Lookup (CFS Vehicles) | Yes | Vehicle with the issue |
| Type | `cr63e_type` | Choices (Multi-Select) | Yes | Category of maintenance issue |
| Description | `cr63e_description` | Multiple Lines of Text | Yes | Detailed description of the issue |
| Status | `cr63e_status` | Choice (Option Set) | Yes | Current maintenance status |
| Out Of Service | `cr63e_outofservice` | Two Options (Boolean) | No | Yes (1) = vehicle pulled from service |
| Reported By | `cr63e_reportedby` | Lookup (Associates) | No | Person who reported the issue |
| Reported Date Time | `cr63e_reporteddatetime` | Date and Time | No | When the issue was reported |

**Status Option Set (`cr63e_status`):**
| Value | Label |
|---|---|
| 360780000 | Open |
| 360780001 | In Progress |
| 360780002 | Closed |

**Type Option Set (`cr63e_type`) - Multi-Select:**
| Value | Label |
|---|---|
| 360780000 | Oil change |
| 360780001 | Tire |
| 360780002 | Damage |
| 360780003 | Check engine light |
| 360780004 | Other |

---

### 7. Vehicle Reservations (`cr63e_vehiclereservations`)

| Column Display Name | Schema Name | Data Type | Required | Notes |
|---|---|---|---|---|
| Vehicle | `cr63e_vehicle` | Lookup (CFS Vehicles) | Yes | Reserved vehicle |
| Associate | `cr63e_associate` | Lookup (Associates) | Yes | Person making reservation |
| Start Date Time | `cr63e_startdatetime` | Date and Time | Yes | Reservation start |
| End Date Time | `cr63e_enddatetime` | Date and Time | Yes | Reservation end |
| Purpose | `cr63e_purpose` | Multiple Lines of Text | No | Reason for the trip |
| Reserved By | `cr63e_reservedby` | Single Line of Text | No | Display name of the reserver |
| Status | `cr63e_status` | Choice (Option Set) | Yes | Reservation lifecycle status |

**Status Option Set (`cr63e_status`):**
| Value | Label |
|---|---|
| 360780000 | Draft |
| 360780001 | Submitted |
| 360780002 | Approved |
| 360780003 | CheckedOut |
| 360780004 | Returned |
| 360780005 | Cancelled |
| 360780006 | NoShow |

---

## NEW TABLE REQUIRED FOR ADMIN APP

### 8. Audit Log (`cr63e_auditlog`) - NEW - Must Be Created

This table tracks all administrative actions for accountability and compliance.

**Steps to Create:**
1. Go to **make.powerapps.com** > **Tables** > **+ New table**
2. Display name: `Audit Log`
3. Schema name: `cr63e_auditlog`
4. Primary column: `Title` (auto-created)

| Column Display Name | Schema Name | Data Type | Required | Notes |
|---|---|---|---|---|
| Title | `cr63e_title` | Single Line of Text | Yes | Auto-generated: e.g., "Vehicle Updated" |
| Action | `cr63e_action` | Single Line of Text | Yes | Action type (Created, Updated, Deleted, StatusChanged) |
| Entity | `cr63e_entity` | Single Line of Text | Yes | Table affected (Vehicle, Reservation, etc.) |
| Record Name | `cr63e_recordname` | Single Line of Text | No | Display name of the affected record |
| Changed By | `cr63e_changedby` | Lookup (Associates) | No | Admin who performed the action |
| Changed Date Time | `cr63e_changeddatetime` | Date and Time | Yes | When the action occurred |
| Details | `cr63e_details` | Multiple Lines of Text | No | JSON or text description of what changed |

**After creating the table:**
- Set the table to **Organization-owned** (not User-owned) for read access
- Grant **Read** access to all app users via security roles
- Grant **Create/Write** access only to Admin roles

---

## DATAVERSE VIEWS TO CREATE

For the Admin app to work efficiently, create these custom views:

### CFS Vehicles Views
1. **Active CFS Vehicles** (default) - Filter: statecode = Active
2. **Available Vehicles** - Filter: cr63e_status = Available AND statecode = Active
3. **Vehicles In Maintenance** - Filter: cr63e_status = Maintenance AND statecode = Active
4. **Out Of Service Vehicles** - Filter: cr63e_status = Out Of Service AND statecode = Active

### Vehicle Reservations Views
1. **Active Reservations** - Filter: cr63e_status IN (Approved, CheckedOut) AND statecode = Active
2. **Upcoming Reservations** - Filter: cr63e_enddatetime >= Today AND cr63e_status <> Cancelled
3. **All Reservations** - No filter, sorted by cr63e_startdatetime DESC

### Vehicle Maintenances Views
1. **Open Maintenance** - Filter: cr63e_status IN (Open, In Progress) AND statecode = Active
2. **All Maintenance** - No filter, sorted by cr63e_reporteddatetime DESC

### Fuel Transactions Views
1. **This Month's Transactions** - Filter: cr63e_transactiondatetime >= First day of current month
2. **All Transactions** - No filter, sorted by cr63e_transactiondatetime DESC

### Vehicle Checkouts Views
1. **Active Checkouts** - Filter: cr63e_returnedat IS NULL AND statecode = Active
2. **All Checkouts** - No filter, sorted by cr63e_checkedoutat DESC

---

## SECURITY ROLES

### Role: CFS Vehicle User
- **Associates**: Read (Organization)
- **CFS Vehicles**: Read (Organization)
- **Fuel Card Assignments**: Read (Organization)
- **Fuel Transactions**: Create (User), Read (Organization), Write (User)
- **Vehicle Checkouts**: Create (User), Read (Organization), Write (User)
- **Vehicle Maintenances**: Create (User), Read (Organization)
- **Vehicle Reservations**: Create (User), Read (Organization), Write (User), Delete (User)

### Role: CFS Fleet Admin
- **All tables above**: Full CRUD (Organization-level)
- **Audit Log**: Create, Read (Organization)

---

## POWER APPS SETUP INSTRUCTIONS

### Mobile User App (CFS Tribal Vehicle Usage)
1. Import the `.msapp` file via **make.powerapps.com** > **Apps** > **Import canvas app**
2. Connect to Dataverse data sources when prompted
3. Verify all 7 data connections resolve (Associates, CFSVehicles, Fuel Card Assignments, Fuel Transactions, Vehicle Checkouts, Vehicle Maintenances, Vehicle Reservations)
4. Set app to **Phone** layout for optimal mobile experience
5. Share with users assigned the **CFS Vehicle User** security role

### Admin Desktop App (CFS Fleet Admin)
1. Import the admin `.msapp` file
2. Connect to the same Dataverse environment
3. Also connect the **Audit Log** table (create it first per instructions above)
4. Set app to **Tablet/Desktop** layout (1366x768)
5. Share only with users assigned the **CFS Fleet Admin** security role

---

## RELATIONSHIP DIAGRAM

```
Associates (cr63e_associates)
    |
    |--< Vehicle Reservations (cr63e_vehiclereservations)
    |       |
    |       |--> CFS Vehicles (cr63e_cfsvehicles)
    |
    |--< Vehicle Checkouts (cr63e_vehiclecheckouts)
    |       |
    |       |--> CFS Vehicles
    |
    |--< Fuel Transactions (cr63e_fueltransactions)
    |       |
    |       |--> CFS Vehicles
    |
    |--< Vehicle Maintenances (cr63e_vehiclemaintenance)
    |       |
    |       |--> CFS Vehicles
    |
    |--< Audit Log (cr63e_auditlog)  [NEW]

CFS Vehicles (cr63e_cfsvehicles)
    |
    |--< Fuel Card Assignments (cr63e_fuelcardassignments)
```

---

## NOTES

- All option set values use the `360780xxx` numbering convention per your Dataverse publisher settings
- The mobile app is optimized for phone-first (portrait) usage
- The admin app is optimized for desktop (1366x768 landscape)
- Both apps share the same Dataverse backend - changes in one appear in the other
- The `varUserAssociate` lookup in the mobile app matches the current user's display name against the Associates table
