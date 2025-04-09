# Understanding Raincode IMSQL: Schema Mapping, Data Translation, and DL/I Emulation

Raincode IMSQL enables legacy IMS COBOL or PL/I applications to run on modern infrastructure without source changes. This document summarizes how it emulates IMS subsystems (IMS/DB and IMS/DC), focusing on:

- Schema mapping and data translation  
- `REDEFINES` handling  
- `GNP` behavior across mixed child segment types  
- PCB/PSB/DBD resolution

---

## Table of Contents

1. Overview of Raincode IMSQL  
2. Schema Mapping  
3. DL/I Call Translation  
4. Data Type Translation  
5. Data Migration  
6. COBOL `REDEFINES` Handling  
7. GNP Across Mixed Segment Types  
8. PCB/PSB/DBD Resolution  
9. Traversal Using Tagged Blob Storage  
10. Conclusion  

---

## 1. Overview of Raincode IMSQL

Raincode IMSQL is a modernization platform that replaces the IMS subsystem with:

- A transaction manager for IMS/DC emulation  
- A DL/I emulation layer that redirects IMS database calls to a relational database (e.g., SQL Server or PostgreSQL)  

The source COBOL or PL/I code is preserved entirely. DL/I logic is emulated using a metadata-driven engine that mimics IMS behavior.

---

## 2. Schema Mapping

In IMS, data is organized hierarchically:

```
CUSTOMER  
 └── ORDER  
      └── LINEITEM
```

In IMSQL, segments become tables:

```
TABLE CUSTOMER (  
  CUSTOMER_ID INT PRIMARY KEY,  
  NAME VARCHAR  
);  

TABLE ORDER (  
  ORDER_ID INT PRIMARY KEY,  
  CUSTOMER_ID INT REFERENCES CUSTOMER  
);  

TABLE LINEITEM (  
  LINEITEM_ID INT PRIMARY KEY,  
  ORDER_ID INT REFERENCES ORDER  
);
```

Parent-child relationships become foreign keys. Segment keys and access paths are preserved based on the DBD metadata.

---

## 3. DL/I Call Translation

Typical DL/I calls like `GU`, `GN`, `GNP`, `REPL`, `DLET` are translated into SQL operations behind the scenes.

Example COBOL call:

`CALL 'CBLTDLI' USING 'GN', pcb, segment, ioarea.  `

Equivalent SQL logic (internally):  

`SELECT * FROM SEGMENT WHERE KEY = ?;  `

Translation is stateful and obeys IMS rules for key sensitivity, hierarchical path traversal, and segment selection.

---

## 4. Data Type Translation

Raincode maps COBOL/IMS data types to SQL types:

| IMS / COBOL Type | SQL Type              |
|------------------|-----------------------|
| `PIC X(n)`       | `VARCHAR(n)`          |
| `PIC 9(n)`       | `INTEGER` / `NUMERIC` |
| `COMP-3`         | `DECIMAL`             |
| Packed Decimal   | `NUMERIC` with scale  |
| Binary data      | `BYTEA` / `VARBINARY` |
| EBCDIC Text      | Converted to UTF-8    |

Field mapping is customizable via configuration.

---

## 5. Data Migration

Raincode provides tooling to:

- Extract IMS database content  
- Convert it to relational rows  
- Optionally preserve binary layouts (e.g., for `REDEFINES`)  
- Maintain logical database definitions from DBDs  

This allows segment data to be bulk-loaded or migrated incrementally.

---

## 6. COBOL `REDEFINES` Handling

Many IMS applications use `REDEFINES` to overlay multiple logical views on the same segment.

Raincode IMSQL preserves this behavior by:

- Storing the raw segment content as a binary blob  
- Decoding it at runtime based on the program’s working storage layout  

Optional decoded views may expose logical fields, but transactional behavior always relies on the original binary content.

---

## 7. GNP Across Mixed Segment Types

In IMS, `GNP` retrieves the next child segment under a parent, regardless of type.

Example hierarchy:

```
CUSTOMER  
 ├── ORDER  
 └── INVOICE
```

COBOL logic:

```
CALL 'CBLTDLI' USING 'GU', pcb, cust-seg.  
PERFORM UNTIL DONE  
   CALL 'CBLTDLI' USING 'GNP', pcb, child-seg  
   EVALUATE child-seg-type  
     WHEN 'ORDER'  
       PERFORM process-order  
     WHEN 'INVOICE'  
       PERFORM process-invoice  
     WHEN OTHER  
       SET DONE TO TRUE  
   END-EVALUATE  
END-PERFORM
```

Raincode must replicate the exact traversal behavior, including segment type switching and segment order defined in the DBD.

---

## 8. PCB/PSB/DBD Resolution

Raincode emulates IMS control blocks:

- **DBD**: Describes segment hierarchy, keys, and layout  
- **PSB**: Defines the set of PCBs a program can use  
- **PCB**: Describes segment sensitivity, access options  

Raincode tools parse these and build metadata used at runtime to resolve:

- Which SQL tables correspond to segments  
- What access logic to apply (e.g., GU, GN, GNP)  
- What keys or foreign keys to use  

DBDs and PSBs may be defined in classic macros or Raincode-specific formats (YAML, JSON, etc.).

---

## 9. Traversal Using Tagged Blob Storage

### Problem

In a fully normalized model, mixed child segment traversal (e.g., `ORDER`, `INVOICE`) requires:

- A separate table per segment  
- A separate cursor per table  
- Logic to merge rows into IMS-style sequence  

This is error-prone and hard to manage.

### Raincode Solution

Instead, Raincode stores all segment instances in a **tagged binary format** within a single table:

```
CREATE TABLE SEGMENTS (  
  PARENT_ID     VARCHAR,  
  SEQ_NO        INTEGER,  
  SEG_TYPE      VARCHAR,  
  SEG_DATA      BYTEA  
);
```

Traversal works by:

1. Selecting all children for a parent  
2. Sorting by `SEQ_NO`  
3. Streaming results one by one  
4. Interpreting `SEG_TYPE` to decode the buffer  

This enables `GNP` traversal across mixed types without managing multiple cursors or joins.

---

## 10. Conclusion

Raincode IMSQL enables faithful emulation of IMS DB/DC behavior:

- Segment hierarchies become relational structures with minimal change  
- `REDEFINES` and data layouts are preserved using binary blobs  
- `GNP` and other DL/I traversal patterns are maintained using tagged, ordered data  
- IMS control blocks (PCB/PSB/DBD) are preserved and emulated internally  

This architecture makes it possible to modernize IMS applications with high fidelity and minimal disruption to legacy code.
