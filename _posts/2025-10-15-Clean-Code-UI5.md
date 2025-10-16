---
title: "Clean Code UI5"
date: 2025-10-15
author: GonzaloMB
tags: ["ui5", "fiori", "sap"]
---

Practical guidelines to write maintainable, testable, and scalable SAP UI5 apps. This post covers structure, naming, async patterns, bindings, fragments, formatters, error handling, performance tips, and examples in JS/TS.

---

## Core Principles

- Small, focused modules: keep controllers thin; move logic to helpers/services.
- Clear naming: nouns for models/controls; verbs for handlers; no abbreviations.
- Consistent structure: `onInit`, `onExit`, public handlers, then private helpers (`_helper`).
- No magic strings: centralize IDs, model names, event/channel names, routes.
- Prefer declarative bindings in XML over imperative DOM changes.
- Single data flow: one source of truth per concern (view model vs. OData model).
- Fail fast with useful messages; use MessageManager for user feedback.

---

## Project Structure (Example)

```
webapp/
  component.js
  manifest.json
  model/
    models.js           # factory for JSONModel, i18n, device
    constants.js        # model names, routes, ids
    formatters.js       # pure formatters
    services.js         # fetch/XHR wrappers
  controller/
    Main.controller.js
  view/
    Main.view.xml
  fragments/
    UploadDialog.fragment.xml
```

---

## Constants (No Magic Strings)

```js
// webapp/model/constants.js
export const MODELS = Object.freeze({
  I18N: "i18n",
  VIEW: "view",
  ODATA: "",
});

export const IDS = Object.freeze({
  TABLE: "ordersTable",
  DIALOG_UPLOAD: "uploadDialog",
});

export const ROUTES = Object.freeze({
  LIST: "OrdersList",
  OBJECT: "OrdersObjectPage",
});
```

---

## Model Factory

```js
// webapp/model/models.js
import JSONModel from "sap/ui/model/json/JSONModel";

export function createViewModel() {
  return new JSONModel({
    busy: false,
    search: "",
    selectedCount: 0,
  });
}
```

Register in `component.js` or `onInit`:

```js
// in Component or controller onInit
import { MODELS } from "../model/constants";
import { createViewModel } from "../model/models";

this.getView().setModel(createViewModel(), MODELS.VIEW);
```

---

## Controller Layout (Async/Await, Helpers)

```js
// webapp/controller/Main.controller.js
sap.ui.define(
  [
    "sap/ui/core/mvc/Controller",
    "sap/ui/core/Fragment",
    "sap/ui/core/message/MessageManager",
    "sap/ui/core/BusyIndicator",
    "../model/constants",
    "../model/services",
  ],
  function (
    Controller,
    Fragment,
    MessageManager,
    BusyIndicator,
    constants,
    services
  ) {
    "use strict";

    return Controller.extend("demo.controller.Main", {
      onInit() {
        this._mm = sap.ui.getCore().getMessageManager();
        this.getView().addDependent(
          this._mm.registerObject(this.getView(), true)
        );
      },

      async onRefresh() {
        await this._withBusy(async () => {
          await this._reloadList();
        });
      },

      onOpenUpload() {
        return Fragment.load({
          name: "demo.fragments.UploadDialog",
          controller: this,
          id: this.getView().getId(),
        }).then((d) => {
          this._upload = d;
          this.getView().addDependent(d);
          d.open();
        });
      },

      onCloseUpload() {
        this._upload?.close();
        this._upload?.destroy(true);
        this._upload = null;
      },

      // Private helpers
      async _reloadList() {
        const table = this.byId(constants.IDS.TABLE);
        await table?.getBinding("items")?.refresh();
      },

      async _withBusy(fn) {
        BusyIndicator.show(0);
        try {
          return await fn();
        } finally {
          BusyIndicator.hide();
        }
      },
    });
  }
);
```

Notes:

- Use `async/await` instead of nested callbacks.
- Encapsulate BusyIndicator and reload logic.
- Keep view state in the view model, not as globals.

---

## XML View: Declarative Bindings

```xml
<!-- webapp/view/Main.view.xml -->
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m" controllerName="demo.controller.Main">
  <VBox width="100%">
    <Toolbar>
      <Title text="Orders" level="H2"/>
      <ToolbarSpacer/>
      <Button text="Refresh" press="onRefresh" type="Emphasized"/>
      <Button text="Upload" press="onOpenUpload"/>
    </Toolbar>
    <Table id="ordersTable" items="{ path: '/Orders' }">
      <columns>
        <Column><Text text="ID"/></Column>
        <Column><Text text="Customer"/></Column>
        <Column><Text text="Total"/></Column>
      </columns>
      <items>
        <ColumnListItem>
          <cells>
            <Text text="{ID}"/>
            <Text text="{CustomerName}"/>
            <ObjectNumber number="{path: 'Total', type: 'sap.ui.model.type.Currency', formatOptions: { showMeasure: false }}"/>
          </cells>
        </ColumnListItem>
      </items>
    </Table>
  </VBox>
  <dependents>
    <core:Fragment id="uploadFrag" type="XML" fragmentName="demo.fragments.UploadDialog" xmlns:core="sap.ui.core"/>
  </dependents>
</mvc:View>
```

Tips:

- Bind visibility/enabled to model properties (`{view>/busy}`), not via JS.
- Use type/formatOptions in bindings; avoid manual string formatting.

---

## Formatters (Pure and Reusable)

````js
// webapp/model/formatters.js
export function currency(value, currency = "EUR") {
  if (value == null) return "";
  return new Intl.NumberFormat(undefined, { style: "currency", currency }).format(value);
}
``+

Use in XML:

```xml
<Text text="{ parts: ['Total'], formatter: '.formatters.currency' }"/>
````

Keep formatters pure (no side effects) to simplify testing.

---

## Services (HTTP/NW RFC/Backend Wrappers)

```js
// webapp/model/services.js
export async function requestJSON(url, options = {}) {
  const res = await fetch(url, {
    headers: { "Content-Type": "application/json" },
    ...options,
  });
  const data = await res.json().catch(() => null);
  if (!res.ok) throw new Error(data?.error?.message || res.statusText);
  return data;
}
```

Avoid placing fetch logic within controllers; centralize it to standardize errors and headers.

---

## Error Handling with MessageManager

```js
try {
  await this._withBusy(() => services.requestJSON("/api/orders"));
} catch (e) {
  const msg = new sap.ui.core.message.Message({
    message: e.message,
    type: sap.ui.core.MessageType.Error,
  });
  sap.ui.getCore().getMessageManager().addMessages(msg);
}
```

Benefits: consistent UX for errors, support for message popover, i18n.

---

## Fragments (Compose and Reuse)

```xml
<!-- webapp/fragments/UploadDialog.fragment.xml -->
<core:FragmentDefinition xmlns="sap.m" xmlns:core="sap.ui.core">
  <Dialog title="Upload">
    <content>
      <FileUploader width="100%"/>
    </content>
    <endButton><Button text="Close" press="onCloseUpload"/></endButton>
  </Dialog>
  </core:FragmentDefinition>
```

Rules:

- Fragments should be dumb (no heavy logic); controllers handle events.
- Always `destroy(true)` fragments on close to prevent leaks.

---

## Testing‑Friendly Patterns

- Extract pure functions (formatters, mappers, validators) for easy unit tests.
- Use DI (pass services into controllers where possible) to mock dependencies.
- Avoid static singletons for state; prefer models.

---

## Performance Tips

- Prefer `ListBinding` with growing/infinite scroll instead of manual pagination.
- Use `autoExpandSelect` and `$select` to avoid overfetching in OData.
- Throttle search liveChange, debounce expensive operations.
- Avoid calling `byId` in loops; cache references when needed.
- Don’t overuse `JSONModel.setData` (replaces object) — prefer `setProperty` for small updates.

---

## TypeScript in UI5 (Optional)

```ts
// tsconfig.json (snippet)
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "strict": true,
    "types": ["@types/openui5"],
    "moduleResolution": "bundler"
  }
}
```

Benefits: stronger contracts for handlers/services, better editor tooling, safer refactors.

---

## Checklist (TL;DR)

- Controllers thin; logic in helpers/services.
- Declarative bindings, pure formatters, reusable fragments.
- No magic strings; centralize IDs/models/routes.
- Async/await + busy wrappers; consistent error handling (MessageManager).
- Test pure functions; DI for services.
- Tune bindings/requests for performance.

---
