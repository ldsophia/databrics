# Informatica Master Hierarchy Implementation Guide

## Scenario

You currently have two source-system hierarchies in Informatica MDM:

- `CRM_HIERARCHY`
- `REDSHIFT_HIERARCHY`

The goal is to create:

- `MASTER_HIERARCHY`

The recommended design is **not** to simply copy or combine the two source hierarchies. Instead, create a mastered relationship layer where CRM and Redshift hierarchy relationships are resolved to the same mastered entities, matched, merged, governed, and then exposed as the Master Hierarchy.

---

# 1. Target Architecture

Assume the current source hierarchies are:

```text
CRM_HIERARCHY

CRM A
 └── CRM B
      └── CRM C
```

```text
REDSHIFT_HIERARCHY

RS 101
 └── RS 102
      └── RS 103
```

The entities have already been mastered by Informatica MDM:

```text
CRM A   ─┐
         ├── Master Entity M001
RS 101  ─┘

CRM B   ─┐
         ├── Master Entity M002
RS 102  ─┘

CRM C   ─┐
         ├── Master Entity M003
RS 103  ─┘
```

The target Master Hierarchy becomes:

```text
MASTER_HIERARCHY

M001
 └── M002
      └── M003
```

The relationship between `M001` and `M002` should retain source lineage:

```text
Master Relationship MR001

M001 → M002

Contributors:
- CRM: CRM_A → CRM_B
- REDSHIFT: RS_101 → RS_102
```

A useful way to think about the architecture is:

```text
                 ENTITY MASTERING

CRM Entity ───────────────┐
                          ├────→ Master Entity
Redshift Entity ──────────┘


              RELATIONSHIP MASTERING

CRM Hierarchy ────────────────┐
                              ├────→ Master Relationship
Redshift Hierarchy ───────────┘
                                      │
                                      ▼
                               MASTER HIERARCHY
```

In short:

> **Master Hierarchy = Master Entities + Master Relationships**

---

# 2. Informatica Objects Required

Assume the following objects already exist:

```text
C_ORGANIZATION

CRM_REL
REDSHIFT_REL

CRM_HIERARCHY
REDSHIFT_HIERARCHY
```

Add the following:

```text
STG_MASTER_REL_CANDIDATE
MASTER_REL
MASTER_PARENT_OF
MASTER_ORG_HIERARCHY
HIER_REL_CONFLICT
```

Recommended responsibilities:

| Object | Purpose |
|---|---|
| `C_ORGANIZATION` | Mastered organization/entity BO |
| `CRM_REL` | CRM source relationships |
| `REDSHIFT_REL` | Redshift source relationships |
| `STG_MASTER_REL_CANDIDATE` | Canonicalized candidate master relationships |
| `MASTER_REL` | Master Relationship Base Object |
| `MASTER_PARENT_OF` | Relationship Type |
| `MASTER_ORG_HIERARCHY` | Master Hierarchy definition |
| `HIER_REL_CONFLICT` | Relationship conflicts requiring rule-based resolution or steward review |

---

# 3. Step 1 — Master the Entities First

Before creating a Master Hierarchy, CRM and Redshift entities must already resolve to the same Informatica master entity.

Example Base Object:

## `C_ORGANIZATION`

| ROWID_OBJECT | Organization Name |
|---|---|
| M001 | ABC Global |
| M002 | ABC Japan |
| M003 | ABC Tokyo |

Example XREF:

| ROWID_OBJECT | SOURCE_SYSTEM | SOURCE_PKEY |
|---|---|---|
| M001 | CRM | CRM_A |
| M001 | REDSHIFT | 101 |
| M002 | CRM | CRM_B |
| M002 | REDSHIFT | 102 |
| M003 | CRM | CRM_C |
| M003 | REDSHIFT | 103 |

This gives the mapping:

```text
CRM_A   → M001
RS_101  → M001

CRM_B   → M002
RS_102  → M002

CRM_C   → M003
RS_103  → M003
```

Do not create the Master Hierarchy until this entity mastering layer is stable.

---

# 4. Step 2 — Read CRM Relationships

Example CRM relationship data:

## `CRM_REL`

| CRM_REL_ID | PARENT_ID | CHILD_ID |
|---|---|---|
| CR01 | CRM_A | CRM_B |
| CR02 | CRM_B | CRM_C |

Source structure:

```text
CRM_A
  ↓
CRM_B
  ↓
CRM_C
```

Resolve each endpoint through the entity XREF:

```text
CRM_A → M001
CRM_B → M002
CRM_C → M003
```

The canonicalized relationships become:

```text
CRM_A → CRM_B
  ↓        ↓
M001  →  M002
```

and:

```text
CRM_B → CRM_C
  ↓        ↓
M002  →  M003
```

---

# 5. Step 3 — Read Redshift Relationships

Example Redshift relationship data:

## `REDSHIFT_REL`

| RS_REL_ID | PARENT_ID | CHILD_ID |
|---|---|---|
| RR01 | 101 | 102 |
| RR02 | 102 | 103 |

Resolve through the same entity XREF:

```text
101 → M001
102 → M002
103 → M003
```

The canonicalized relationships become:

```text
101  → 102
 ↓      ↓
M001 → M002
```

and:

```text
102  → 103
 ↓      ↓
M002 → M003
```

At this point both source systems describe the same canonical relationships:

```text
CRM:
M001 → M002
M002 → M003

REDSHIFT:
M001 → M002
M002 → M003
```

---

# 6. Step 4 — Build `STG_MASTER_REL_CANDIDATE`

Create an intermediate staging table.

Recommended structure:

```text
SOURCE_SYSTEM
SOURCE_REL_ID

SOURCE_PARENT_ID
SOURCE_CHILD_ID

MASTER_PARENT_ROWID
MASTER_CHILD_ROWID

HIERARCHY_CODE
RELATIONSHIP_TYPE

PERIOD_START_DATE
PERIOD_END_DATE

SOURCE_PRIORITY

RESOLUTION_STATUS
CONFLICT_REASON

LAST_UPDATE_DATE
```

Example data:

| Source | Source Rel | Master Parent | Master Child | Hierarchy | Rel Type |
|---|---|---|---|---|---|
| CRM | CR01 | M001 | M002 | MASTER | PARENT_OF |
| REDSHIFT | RR01 | M001 | M002 | MASTER | PARENT_OF |
| CRM | CR02 | M002 | M003 | MASTER | PARENT_OF |
| REDSHIFT | RR02 | M002 | M003 | MASTER | PARENT_OF |

This table becomes the input to duplicate detection, conflict detection, and Master Relationship loading.

---

# 7. Step 5 — Create `MASTER_REL`

Create a new Relationship Base Object:

```text
MASTER_REL
```

Recommended business columns include:

```text
ROWID_OBJECT

MASTER_PARENT_ROWID
MASTER_CHILD_ROWID

HIERARCHY_TYPE
RELATIONSHIP_TYPE

PERIOD_START_DATE
PERIOD_END_DATE

RELATIONSHIP_STATUS

SOURCE_PRIORITY
CONFIDENCE_SCORE
MANUAL_OVERRIDE_IND
```

Source-specific keys should remain in XREF/source lineage rather than becoming the business identity of the Master Relationship.

---

# 8. Step 6 — Create the Relationship Type

Create:

```text
MASTER_PARENT_OF
```

Configuration:

```text
Parent Entity Type:
    Organization

Child Entity Type:
    Organization

Direction:
    Parent → Child
```

Conceptually:

```text
Organization
     │
     │ MASTER_PARENT_OF
     ▼
Organization
```

---

# 9. Step 7 — Create the Master Hierarchy

Create:

```text
Hierarchy Code:
MASTER_ORG_HIER

Display Name:
Master Organization Hierarchy
```

Add:

```text
MASTER_PARENT_OF
```

to:

```text
MASTER_ORG_HIER
```

Logical structure:

```text
MASTER_ORG_HIER
      │
      └── MASTER_PARENT_OF
              │
              └── MASTER_REL
```

At this stage you have created the hierarchy metadata, but the Master Relationship records still need to be generated and loaded.

---

# 10. Step 8 — Populate Master Relationship Candidates

Create two mappings.

## Mapping 1 — CRM

```text
CRM_REL
   ↓
Lookup Organization XREF for Parent
   ↓
Lookup Organization XREF for Child
   ↓
STG_MASTER_REL_CANDIDATE
```

## Mapping 2 — Redshift

```text
REDSHIFT_REL
   ↓
Lookup Organization XREF for Parent
   ↓
Lookup Organization XREF for Child
   ↓
STG_MASTER_REL_CANDIDATE
```

Keep the original source identity:

CRM:

```text
SOURCE_SYSTEM = CRM
SOURCE_PKEY   = CRM_REL_ID
```

Redshift:

```text
SOURCE_SYSTEM = REDSHIFT
SOURCE_PKEY   = RS_REL_ID
```

Example:

```text
CRM / CR01
REDSHIFT / RR01
```

---

# 11. Step 9 — Detect Duplicate Relationships

A duplicate relationship occurs when multiple source systems identify the same mastered parent-child relationship.

Example:

```text
CRM:
M001 → M002

Redshift:
M001 → M002
```

This should become one Master Relationship:

```text
MR001
M001 → M002
```

with lineage:

```text
MR001
├── CRM / CR01
└── REDSHIFT / RR01
```

---

# 12. Step 10 — Configure Relationship Match/Merge

For hierarchy relationships, use exact matching rather than fuzzy matching.

Recommended Match Rule:

```text
MASTER_PARENT_ROWID
AND
MASTER_CHILD_ROWID
AND
RELATIONSHIP_TYPE
AND
HIERARCHY_TYPE
```

Example canonical relationship key:

```text
M001|M002|PARENT_OF|MASTER
```

CRM generates:

```text
M001|M002|PARENT_OF|MASTER
```

Redshift generates:

```text
M001|M002|PARENT_OF|MASTER
```

Therefore they match and merge.

Do **not** include the source relationship ID in the match key.

For example:

```text
CRM: CR01
Redshift: RR01
```

These source identifiers will never match. They should be retained only for lineage/XREF.

Recommended configuration:

| Match Column | Match Type |
|---|---|
| Parent Master ROWID | Exact |
| Child Master ROWID | Exact |
| Relationship Type | Exact |
| Hierarchy Type | Exact |

Auto Merge can be enabled when these fields match exactly.

---

# 13. Step 11 — Handle Effective Dates

Example:

CRM:

```text
M001 → M002
2024-01-01 → 9999-12-31
```

Redshift:

```text
M001 → M002
2025-01-01 → 9999-12-31
```

Avoid using the exact start/end dates as part of the core relationship identity unless the business requires each effective period to represent a separate relationship.

Recommended identity:

```text
Parent
Child
Relationship Type
Hierarchy Type
```

Treat effective periods as relationship timeline/version attributes.

---

# 14. Step 12 — Detect Structural Conflicts

A conflict is different from a duplicate.

Example:

CRM:

```text
M001 → M003
```

Redshift:

```text
M002 → M003
```

These relationships should **not** match.

The question becomes:

> Which parent should survive in the Master Hierarchy?

This requires business governance.

---

# 15. Step 13 — Define Source Priority

Example for a Sales Hierarchy:

```text
CRM        = 100
REDSHIFT   = 50
```

Then:

```text
CRM:
M001 → M003
Priority 100

Redshift:
M002 → M003
Priority 50
```

Master result:

```text
M001 → M003
```

The Redshift candidate can be retained in a conflict table for lineage or review.

Example:

## `HIER_REL_CONFLICT`

| Child | Parent | Source | Priority | Result |
|---|---|---|---:|---|
| M003 | M001 | CRM | 100 | WIN |
| M003 | M002 | REDSHIFT | 50 | REJECT/REVIEW |

---

# 16. Source Priority Can Vary by Hierarchy Type

Do not assume that CRM always wins.

Example:

## Sales Hierarchy

```text
CRM        100
REDSHIFT    50
```

## Legal Hierarchy

```text
REDSHIFT   100
CRM         50
```

## Finance Hierarchy

```text
ERP        100
REDSHIFT    80
CRM         20
```

This allows separate enterprise hierarchies such as:

```text
MASTER_SALES_HIERARCHY
MASTER_LEGAL_HIERARCHY
MASTER_FINANCE_HIERARCHY
```

rather than forcing every business use case into one universal hierarchy.

---

# 17. Step 14 — Resolve Conflicts Before Loading `MASTER_REL`

Recommended processing flow:

```text
CRM_REL ───────────────┐
                       │
                       ├──→ Resolve Master Entity IDs
                       │
REDSHIFT_REL ──────────┘
                               │
                               ▼
                    STG_MASTER_REL_CANDIDATE
                               │
                               ▼
                       Duplicate Detection
                               │
                               ▼
                        Conflict Detection
                               │
                               ▼
                       Source Priority Rules
                               │
                  ┌────────────┴────────────┐
                  ▼                         ▼
             WINNER REL               CONFLICT REL
                  │                         │
                  ▼                         ▼
            MASTER_REL              HIER_REL_CONFLICT
                  │
                  ▼
             Match / Merge
                  │
                  ▼
           MASTER_HIERARCHY
```

---

# 18. Example Conflict Resolution SQL

If the business rule says:

> One child can have only one active parent within a specific hierarchy and relationship type.

Then a staging-layer rule can use:

```sql
ROW_NUMBER() OVER (
    PARTITION BY
        MASTER_CHILD_ROWID,
        HIERARCHY_TYPE,
        RELATIONSHIP_TYPE
    ORDER BY
        SOURCE_PRIORITY DESC,
        LAST_UPDATE_DATE DESC
) AS RN
```

Then:

```text
RN = 1
    → load to MASTER_REL

RN > 1
    → conflict / alternate relationship / steward review
```

Example:

| Child | Parent | Source | Priority | RN |
|---|---|---|---:|---:|
| M003 | M001 | CRM | 100 | 1 |
| M003 | M002 | REDSHIFT | 50 | 2 |

Only:

```text
M001 → M003
```

is loaded as the surviving Master Relationship.

---

# 19. Duplicate vs Conflict

These cases must be treated differently.

## Case A — Duplicate

CRM:

```text
M001 → M002
```

Redshift:

```text
M001 → M002
```

Result:

```text
MATCH + MERGE
```

Final:

```text
MR001
M001 → M002
```

with two source contributors.

---

## Case B — Conflict

CRM:

```text
M001 → M003
```

Redshift:

```text
M002 → M003
```

Result:

```text
SOURCE PRIORITY
or
DATA STEWARD REVIEW
```

Do not match/merge them into the same relationship.

---

## Case C — Relationship Exists in Only One Source

CRM:

```text
M001 → M004
```

Redshift:

```text
No relationship
```

Normally:

```text
Accept CRM relationship
```

unless the business requires confirmation from multiple sources.

---

# 20. Recommended Decision Matrix

| CRM | Redshift | Master Decision |
|---|---|---|
| A→B | A→B | A→B, merge lineage |
| A→B | null | A→B |
| null | A→B | A→B |
| A→B | C→B | Apply source priority or stewardship |
| A→B | A→C | Two independent relationships |
| A→B expired | A→B active | Apply effective-date policy |
| A→B | A→B | Match and merge relationship |

This matrix should be formally agreed with business/data governance teams.

---

# 21. Step 15 — Validation Before Loading Master Relationships

At minimum validate the following.

## 1. Orphan Parent

```text
Source Parent ID
→ No matching Master Entity
```

Do not load.

## 2. Orphan Child

```text
Source Child ID
→ No matching Master Entity
```

Do not load.

## 3. Self Relationship

```text
M001 → M001
```

Reject.

## 4. Multiple Parent Conflict

For a strict tree hierarchy:

```text
M001 → M003
M002 → M003
```

Send through conflict resolution.

## 5. Cycle Detection

Example:

```text
M001 → M002
M002 → M003
M003 → M001
```

Reject or route for steward review.

---

# 22. Production Processing Order

Recommended batch order:

```text
1. Load CRM Entities

2. Load Redshift Entities

3. Run Entity Match

4. Run Entity Merge

5. Confirm Entity XREF → Master ROWID_OBJECT

6. Load CRM Source Hierarchy

7. Load Redshift Source Hierarchy

8. Generate STG_MASTER_REL_CANDIDATE
   - Resolve Parent Master ROWID
   - Resolve Child Master ROWID

9. Validate Candidate Relationships
   - Unresolved parent
   - Unresolved child
   - Self relationship
   - Circular relationship
   - Multiple-parent conflict
   - Invalid effective dates

10. Detect duplicate relationships

11. Detect hierarchy conflicts

12. Apply source-priority and governance rules

13. Load surviving relationships into MASTER_REL

14. Tokenize MASTER_REL if required by Hub processing

15. Run MASTER_REL Match

16. Run MASTER_REL Merge

17. Publish/display MASTER_ORG_HIERARCHY
```

Entity mastering must happen before hierarchy mastering because source parent/child IDs must first resolve to stable master entity IDs.

---

# 23. End-to-End Example

Source CRM hierarchy:

```text
CRM001
 └── CRM002
      └── CRM003
```

Source Redshift hierarchy:

```text
RS100
 └── RS200
      └── RS300
```

Entity mastering:

```text
CRM001 = RS100 = M001
CRM002 = RS200 = M002
CRM003 = RS300 = M003
```

Candidate relationships:

```text
CRM:
M001 → M002
M002 → M003

REDSHIFT:
M001 → M002
M002 → M003
```

Relationship matching:

```text
CRM M001 → M002
+
Redshift M001 → M002
=
MR001
```

and:

```text
CRM M002 → M003
+
Redshift M002 → M003
=
MR002
```

Final hierarchy:

```text
MASTER_ORG_HIERARCHY

M001
 │
 │ MR001
 ▼
M002
 │
 │ MR002
 ▼
M003
```

Relationship lineage:

```text
MR001
├── CRM / CR01
└── REDSHIFT / RR01

MR002
├── CRM / CR02
└── REDSHIFT / RR02
```

---

# 24. Minimum Implementation Checklist

1. Confirm CRM and Redshift entities already resolve to the same Master Entity BO.
2. Keep `CRM_HIERARCHY` and `REDSHIFT_HIERARCHY` unchanged for source lineage.
3. Create `STG_MASTER_REL_CANDIDATE`.
4. Create `MASTER_REL` Relationship Base Object.
5. Create `MASTER_PARENT_OF` Relationship Type.
6. Create `MASTER_ORG_HIERARCHY`.
7. Map CRM parent/child IDs to master entity ROWIDs.
8. Map Redshift parent/child IDs to master entity ROWIDs.
9. Load both sets into `STG_MASTER_REL_CANDIDATE`.
10. Validate orphan, self, duplicate, multiple-parent, effective-date, and cycle conditions.
11. Define source-priority rules for CRM vs Redshift.
12. Route unresolved structural conflicts to `HIER_REL_CONFLICT` or steward review.
13. Load winning candidate relationships into `MASTER_REL`.
14. Configure exact relationship matching using:
    - Parent Master ID
    - Child Master ID
    - Relationship Type
    - Hierarchy Type
15. Run Match/Merge for `MASTER_REL`.
16. Expose the surviving relationships in `MASTER_ORG_HIERARCHY`.
17. Validate final hierarchy and source lineage in Hierarchy Manager / IDD.

---

# 25. Recommended Design Principle

Do not directly merge two already-built source hierarchies as opaque hierarchy structures.

Instead:

```text
Source Hierarchy
      ↓
Resolve Source Nodes to Master Entities
      ↓
Canonical Relationship Candidates
      ↓
Duplicate Detection
      ↓
Conflict Resolution
      ↓
Relationship Match/Merge
      ↓
Master Relationships
      ↓
Master Hierarchy
```

This preserves:

- source lineage;
- auditability;
- entity mastering;
- relationship mastering;
- source-specific hierarchies;
- enterprise hierarchy governance;
- conflict handling;
- support for multiple hierarchy types.

The key architectural principle is:

> **Master the nodes first, then master the relationships.**
