---
title: "SAP CAP· CSV Uploader Fiori Elements v4"
date: 2025-10-14
author: GonzaloMB
tags: ["sap", "cap", "fiori-elements", "ui5"]
---

End‑to‑end example to bulk upload data from a CSV file into a CAP service and trigger it from a Fiori Elements v4 List Report via a custom action.

Reference repo (full code): https://github.com/GonzaloMB/cap-file-uploader-v2.git

---

## Requirements

- CAP (Node.js) `@sap/cds@^8`
- Dev DB: SQLite (`@cap-js/sqlite`) or HANA in prod
- FE v4 app (List Report + Object Page)
- UI5 1.134.x

---

## Data Model (db/schema.cds)

Simple entity to store uploaded items.

```cds
namespace my.bookshop;

entity Books {
  key ID : Integer;
  title  : String;
  stock  : Integer;
}
```

---

## Service + Action (srv/cat-service.cds)

Expose `Books` and add an action to receive CSV rows as typed input.

```cds
using my.bookshop as my from '../db/schema';

type BooksInput { title: String; stock: Integer; }

service CatalogService {
  entity Books as projection on my.Books;
  action UploadBooks(items: array of BooksInput) returns Integer;
}
```

Implementation (srv/cat-service.js): validation + bulk insert in a single tx.

```js
const cds = require("@sap/cds");

module.exports = cds.service.impl(async function () {
  const { Books } = this.entities;

  this.on("UploadBooks", async (req) => {
    const { items } = req.data;
    if (!Array.isArray(items) || items.length === 0)
      return req.error(400, "No records were provided");

    items.forEach((item, i) => {
      if (typeof item.title !== "string" || item.title.trim().length < 3)
        req.error(400, `Row ${i + 1}: invalid 'title'`);
      if (!Number.isInteger(item.stock) || item.stock < 0)
        req.error(400, `Row ${i + 1}: invalid 'stock'`);
    });

    const titles = items.map((i) => i.title.trim().toLowerCase());
    const dup = titles.filter((t, i) => titles.indexOf(t) !== i);
    if (dup.length)
      return req.error(
        400,
        `Duplicate titles found: ${[...new Set(dup)].join(", ")}`
      );

    const tx = cds.transaction(req);
    await tx.run(
      INSERT.into(Books).entries(
        items.map((i) => ({
          title: i.title.trim(),
          stock: i.stock,
          IsActiveEntity: true,
        }))
      )
    );
    return items.length;
  });
});
```

---

## FE v4 App (List Report)

Manifest adds a custom action on the List Report toolbar and wires a controller extension.

```json
"targets": {
  "BooksList": {
    "type": "Component",
    "id": "BooksList",
    "name": "sap.fe.templates.ListReport",
    "options": {
      "settings": {
        "contextPath": "/Books",
        "controlConfiguration": {
          "@com.sap.vocabularies.UI.v1.LineItem": {
            "tableSettings": { "type": "ResponsiveTable" },
            "actions": {
              "customAction": {
                "press": ".extension.bookstockcontrolv2.ext.controller.ListReportExt.onUploadCSV",
                "text": "Upload File",
                "requiresSelection": false
              }
            }
          }
        }
      }
    }
  }
}
```

Annotations (app/book_stock_control_v2/annotations.cds) define fields and facets; drafts are enabled with `@odata.draft.enabled`.

---

## UI: Dialog + Controller Extension

Dialog fragment with `FileUploader` and actions.

```xml
<core:FragmentDefinition xmlns="sap.m" xmlns:core="sap.ui.core" xmlns:u="sap.ui.unified">
  <Dialog title="{i18n>titleDialogCSV}">
    <endButton><Button text="{i18n>btnClose}" type="Negative" press=".onCloseDialog"/></endButton>
    <beginButton><Button type="Success" icon="sap-icon://upload" text="{i18n>btnUpload}" press=".onUploadData"/></beginButton>
    <VBox alignItems="Center">
      <u:FileUploader change="handleFiles" buttonText="{i18n>btnBrowse}" fileType="CSV" placeholder="{i18n>msgNonFileSelect}"/>
      <Button type="Neutral" icon="sap-icon://download" text="{i18n>btnDowloadTmpl}" press=".onDownloadTemplate"/>
      <MessageStrip id="messageStripId" visible="false" text="{i18n>msgStrip}" type="Success" showIcon="true"/>
    </VBox>
  </Dialog>
 </core:FragmentDefinition>
```

Controller extension: parse CSV (semicolon‑separated), call CAP action, show results.

```js
onUploadCSV() { Fragment.load({ id: this.base.getView().getId(), name: "bookstockcontrolv2.ext.view.UploadFileDialog", controller: this })
  .then(d => (this.pDialog = d, this.base.getView().addDependent(d), d.open())); },

handleFiles(oEvent) {
  const f = oEvent.getParameter("files")[0];
  const r = new FileReader();
  r.onload = (e) => {
    const rows = e.target.result.split(/\r\n|\n/).map(l => l.split(";")).filter(a => a.length > 1);
    const headers = rows.shift();
    this.oFileData = rows.map(cols => headers.reduce((o,h,i)=>(o[h]=cols[i],o),{}));
    this.base.byId("messageStripId")?.setVisible(true);
  };
  r.readAsText(f);
},

async onUploadData() {
  const oModel = this.base.getView().getModel();
  const items = (this.oFileData||[]).map(i => ({ title: i.title, stock: parseInt(i.stock,10) }));
  if (!items.length) return MessageToast.show(this._i18n("msgNonFileSelect"));
  const res = await fetch(`${oModel.sServiceUrl}UploadBooks`, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ items })});
  const data = await res.json();
  if (!res.ok) throw new Error(data?.error?.message || 'Upload failed');
  MessageToast.show(`Created ${data} books.`);
  this.onCloseDialog();
}
```

CSV template generation is also included:

```js
onDownloadTemplate() {
  const a = document.createElement('a');
  a.href = 'data:text/csv;charset=utf-8,' + encodeURI('title;stock');
  a.download = 'TemplateCsv.csv';
  a.click();
}
```

---

## CSV Contract

- Separator: semicolon `;`
- Header row required: `title;stock`
- Validation rules (enforced in service):
  - `title`: string, min length 3, non‑empty
  - `stock`: integer, >= 0
  - No duplicate titles in the same upload

---

## Run and Verify

1. `npm install` (first time)
2. `cds watch`
3. Open the FE app (List Report). Click "Upload File" → pick a CSV → Upload.
4. Success toast shows amount created; the list refreshes automatically.

---

## Troubleshooting

- 400 with backend message: fix the offending row (min title length, integer stock, duplicates).
- Empty upload: ensure the CSV has a header and `;` as delimiter.
- Dialog not opening: verify the controller extension path in `manifest.json`.
- Draft entity flag: inserts set `IsActiveEntity: true` to display immediately in FE v4.

---

## Links

- Source code: https://github.com/GonzaloMB/cap-file-uploader-v2.git
- CAP Docs: https://cap.cloud.sap/docs/

---
