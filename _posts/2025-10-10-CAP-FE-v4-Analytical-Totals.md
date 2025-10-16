---
title: "SAP CAP - Totals Fiori Elements v4"
date: 2025-10-10
author: GonzaloMB
tags: ["sap", "cap", "fiori-elements", "ui5", "odata"]
---

This post walks through, step by step, how to enable totals (summations) in a Fiori Elements v4 Analytical Table using SAP CAP (Node.js) and HANA. The goal is to help you replicate it quickly in your own project.

Reference repo (full code): https://github.com/GonzaloMB/Totals_CAP_v4

---

## Requirements

- CAP in Node.js + HANA
- FE v4 app (List Report) with `AnalyticalTable`
- UI5 1.112.1 or newer
- Environment: SAP BAS (recommended) or local with `@sap/cds`

---

## 1) Data model (db/)

Define the entity and mark the numeric field you want to totalize (here, `stock`).

```abap
namespace my.bookshop;

entity Books {
  key ID       : Integer;
      title    : String  @Common.Label: 'Title';
      bookshop : String  @Common.Label: 'BookShop';
      stock    : Integer @Common.Label: 'Stock';
}
```

---

## 2) Service (srv/)

Project the entity in your service to expose it via OData.

```abap
using my.bookshop as my from '../db/data-model';

service CatalogService {
    entity Books as projection on my.Books;
}
```

---

## 3) UI: manifest.json (APP/)

Configure the List Report to use `AnalyticalTable` and ensure your UI5 version is set properly.

Change table type:

```json
"targets": {
  "BooksList": {
    "type": "Component",
    "id": "BooksList",
    "name": "sap.fe.templates.ListReport",
    "options": {
      "settings": {
        "initialLoad": "Enabled",
        "entitySet": "Books",
        "controlConfiguration": {
          "@com.sap.vocabularies.UI.v1.LineItem": {
            "tableSettings": {
              "type": "AnalyticalTable"
            }
          }
        }
      }
    }
  }
}
```

Pin the UI5 version (example 1.112.1):

```json
"sap.platform.cf": {
  "ui5VersionNumber": "1.112.1"
},
"sap.ui5": {
  "flexEnabled": true,
  "dependencies": {
    "minUI5Version": "1.112.1",
    "libs": {
      "sap.m": {},
      "sap.ui.core": {},
      "sap.ushell": {},
      "sap.fe.templates": {}
    }
  }
}
```

---

## 4) Annotations (APP/annotations.cds)

Define grouping, default totals, and the measure with its aggregation. This enables the total in the table footer and totals by group.

```abap
using CatalogService as service from '../../srv/cat-service';

annotate service.Books with @(
    Aggregation.ApplySupported        : { GroupableProperties: [
        title,
        bookshop
    ]},
    Aggregation.CustomAggregate #stock: 'Edm.Int32',
    UI.PresentationVariant            : {
        Total         : [stock],
        Visualizations: ['@UI.LineItem']
    },
    UI.SelectionFields                : [
        title,
        bookshop,
        stock
    ],
    UI.HeaderInfo                     : {
        $Type         : 'UI.HeaderInfoType',
        TypeName      : 'Book Stock Totals',
        TypeNamePlural: 'Book Stock Totals',
        Title         : { Value: 'Book Stock Totals' }
    },
    UI.LineItem                       : [
        { $Type: 'UI.DataField', Value: title },
        { $Type: 'UI.DataField', Value: bookshop },
        { $Type: 'UI.DataField', Value: stock }
    ]
);

annotate service.Books with @(
    UI.FieldGroup #GeneratedGroup1: {
        $Type: 'UI.FieldGroupType',
        Data : [
            { $Type: 'UI.DataField', Value: title },
            { $Type: 'UI.DataField', Value: bookshop },
            { $Type: 'UI.DataField', Value: stock }
        ]
    },
    UI.Facets: [{
        $Type : 'UI.ReferenceFacet',
        ID    : 'GeneratedFacet1',
        Label : 'General Information',
        Target: '@UI.FieldGroup#GeneratedGroup1'
    }]
) {
    stock @Analytics.Measure @Aggregation.default: #SUM;
    ID    @UI.Hidden: true;
};
```

Key points:

- `UI.PresentationVariant.Total`: shows the totals row in the table footer for `stock`.
- `Aggregation.ApplySupported.GroupableProperties`: which fields can be grouped.
- `Aggregation.CustomAggregate`: aggregate type for `stock` (integer).
- `@Analytics.Measure` + `@Aggregation.default: #SUM`: marks `stock` as a summable measure.

---

## 5) CAP Node.js: OData parser for `$apply`

In CAP (Node.js), certain `$apply` combinations (like `concat(...)`) may fail with the default parser. Enable the new (experimental) parser in `package.json` to support FE v4 AnalyticalTable queries with grouping.

```json
{
  "cds": {
    "requires": {
      "db": "hana"
    },
    "features": {
      "odata_new_parser": true
    }
  }
}
```

Without this option, requests like the following may fail in Node.js even if they work in CAP Java:

```http
GET /BooksAggregate?$apply=concat(groupby((ID,category,title))/aggregate($count%20as%20UI5__leaves),groupby((category))/concat(aggregate($count%20as%20UI5__count),%20top(72)))
```

---

## 6) Run and verify

1. Start the project: `cds watch`.
2. Open the FE app (List Report for `Books`).
3. Check:
   - Table footer shows total for `stock`.
   - Group totals appear when grouping by `title`/`bookshop`.
   - No `$apply` errors (if any, revisit the new parser and your annotation targets).

---

## Common pitfalls

- Totals row not visible: confirm `UI.PresentationVariant.Total` and that `stock` is `@Analytics.Measure` with `#SUM`.
- Grouping doesn’t sum: add the field to `GroupableProperties` and check `CustomAggregate`.
- `$apply` fails with `concat(...)`: enable `"odata_new_parser": true` in `package.json`.
- Wrong annotation target: `annotate` must point to `service.Books` (not the plain namespace entity).

---

## Links

- Reference code: https://github.com/GonzaloMB/Totals_CAP_v4
- CAP Docs (Get Started): https://cap.cloud.sap/docs/get-started/

---
