---
title: "Print SmartChart (UI5 Analytical List Page)"
date: 2025-10-11
author: GonzaloMB
tags: ["ui5", "fiori-elements", "extension"]
---

Create and extend an Analytical List Page (Fiori Elements) to add a "Print Chart to PDF" feature using UI5 and lightweight client-side libraries.

Reference repo (full code): https://github.com/GonzaloMB/smartchart_pdf_print_ui5.git

---

## Requirements

- ABAP CDS on an on‑prem system (OData V2 published via `@OData.publish: true`).
- SAP BAS or VS Code for FE development.
- SAP Logon / Eclipse (for ABAP CDS authoring, if applicable).

---

## Backend (ABAP CDS)

1. Basic view: consume table `sflight` from on‑prem.

```abap
@AbapCatalog.sqlViewName: 'ZISFLIGHTDEMOCDS'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl.authorizationCheck: #NOT_REQUIRED
@VDM.viewType: #BASIC
@EndUserText.label: 'Basic CDS to capability demo'
define view ZI_SFLIGHT_DEMO_CDS
  as select from sflight
{
key carrid as Carrid,
key connid as Connid,
key fldate as Fldate,
price as Price,
currency as Currency,
planetype as Planetype,
seatsmax as Seatsmax,
seatsocc as Seatsocc,
paymentsum as Paymentsum,
seatsmax_b as SeatsmaxB,
seatsocc_b as SeatsoccB,
seatsmax_f as SeatsmaxF,
seatsocc_f as SeatsoccF

}
```

2. Consumption view: publish OData V2.

```abap
@AbapCatalog.sqlViewName: 'ZCSFLIGHTCAP'
@AbapCatalog.compiler.compareFilter: true
@AbapCatalog.preserveKey: true
@AccessControl.authorizationCheck: #NOT_REQUIRED
@VDM.viewType: #CONSUMPTION
@Metadata.allowExtensions: true
@OData.publish: true
@EndUserText.label: 'Consumption CDS to capability demo'
define view ZC_SFLIGHT_CAPABILITY_CDS
  as select from ZI_SFLIGHT_DEMO_CDS
{
  key Carrid,
  key Connid,
  key Fldate,
      Price,
      Currency,
      Planetype,
      Seatsmax,
      Seatsocc,
      Paymentsum,
      SeatsmaxB,
      SeatsoccB,
      SeatsmaxF,
      SeatsoccF
}
```

3. Metadata extension: keep annotations separate from business logic.

```abap
@Metadata.layer: #(sflight)
@UI.chart: [
  {
    chartType: #COLUMN,
    dimensions: [ 'Planetype' ],
    measures: [ 'Seatsmax' ]
  }
]
@UI: {
  headerInfo: {
    typeName: 'Flight',
    typeNamePlural: 'Flights',
    title: { type: #STANDARD, value: 'Carrid' },
    description: { value: 'Planetype' }
  }
}

annotate entity ZC_SFLIGHT_CAPABILITY_CDS with
{
  @UI.facet: [
    { id: 'Fldateheaderid', purpose: #HEADER, type: #DATAPOINT_REFERENCE, position: 10, targetQualifier: 'Fldateheader' },
    { id: 'Currencyheaderid', purpose: #HEADER, type: #DATAPOINT_REFERENCE, position: 20, targetQualifier: 'Priceheader' },
    { label: 'General Information', id: 'GeneralInfo', type: #COLLECTION, position: 10 },
    { id: 'Price', purpose: #STANDARD, type: #IDENTIFICATION_REFERENCE, parentId: 'GeneralInfo', label: 'Price', position: 10 }
  ]
  @UI.lineItem: [{position: 10 }]
  @UI.selectionField: [{position: 10 }]
  Carrid;
  @UI.lineItem: [{position: 20 }]
  @UI.selectionField: [{position: 20 }]
  Connid;
  @UI.lineItem: [{position: 30 }]
  @UI.selectionField: [{position: 30 }]
  @UI.dataPoint: { qualifier: 'Fldateheader', title: 'Flight Date'}
  Fldate;
  @UI.dataPoint: { qualifier: 'Priceheader', title: 'Airfare'}
  @UI.lineItem: [{position: 40 }]
  @UI.identification: [{ position: 10 }]
  Price;
  @UI.dataPoint: { qualifier: 'Currencyheader', title: 'Airline Currency'}
  @UI.lineItem: [{position: 50 }]
  @UI.identification: [{ position: 20 }]
  Currency;
  @UI.lineItem: [{position: 60 }]
  Planetype;
  @UI.lineItem: [{position: 70 }]
  Seatsmax;
  @UI.lineItem: [{position: 80 }]
  Seatsocc;
  @UI.lineItem: [{position: 90 }]
  Paymentsum;
  @UI.lineItem: [{position: 100 }]
  SeatsmaxB;
  @UI.lineItem: [{position: 110 }]
  SeatsoccB;
  @UI.lineItem: [{position: 120 }]
  SeatsmaxF;
  @UI.lineItem: [{position: 130 }]
  SeatsoccF;
}
```

---

## Frontend (Fiori Elements · Analytical List Page)

1. Generate an ALP app (template: Analytical List Page). Fill project details and main OData service.

2. Validate data sources in `manifest.json`:

```json
"dataSources": {
  "mainService": {
    "uri": "/sap/opu/odata/sap/ZC_SFLIGHT_CAPABILITY_CDS_CDS/",
    "type": "OData",
    "settings": {
      "annotations": ["ZC_SFLIGHT_CAPABILITY_CDS_CD_VAN", "annotation"],
      "localUri": "localService/metadata.xml",
      "odataVersion": "2.0"
    }
  },
  "ZC_SFLIGHT_CAPABILITY_CDS_CD_VAN": {
    "uri": "/sap/opu/odata/IWFND/CATALOGSERVICE;v=2/Annotations(TechnicalName='ZC_SFLIGHT_CAPABILITY_CDS_CD_VAN',Version='0001')/$value/",
    "type": "ODataAnnotation",
    "settings": { "localUri": "localService/ZC_SFLIGHT_CAPABILITY_CDS_CD_VAN.xml" }
  },
  "annotation": {
    "type": "ODataAnnotation",
    "uri": "annotations/annotation.xml",
    "settings": { "localUri": "annotations/annotation.xml" }
  }
}
```

3. Controller extension: create `webapp/ext/controller/AnalyticalListPageExt.controller.js` and reference it in `manifest.json`:

```json
"extends": {
  "extensions": {
    "sap.ui.controllerExtensions": {
      "sap.suite.ui.generic.template.AnalyticalListPage.view.AnalyticalListPage": {
        "controllerName": "pdfprintui5.ext.controller.AnalyticalListPageExt"
      }
    }
  }
}
```

4. Custom action (button) wired to the controller method:

```json
"extends": {
  "extensions": {
    "sap.ui.controllerExtensions": {
      "sap.suite.ui.generic.template.AnalyticalListPage.view.AnalyticalListPage": {
        "controllerName": "pdfprintui5.ext.controller.AnalyticalListPageExt",
        "sap.ui.generic.app": {
          "ZC_SFLIGHT_CAPABILITY_CDS": {
            "EntitySet": "ZC_SFLIGHT_CAPABILITY_CDS",
            "Actions": {
              "btnPrintChart": {
                "id": "btnPrintChart",
                "text": "Print Chart to PDF",
                "press": "onPrintChart"
              }
            }
          }
        }
      }
    }
  }
}
```

5. Third‑party libraries and optional CSS in `manifest.json` (place assets under `webapp/libs` and `webapp/css`):

```json
"resources": {
  "css": [ { "uri": "css/style.css" } ],
  "js": [
    { "uri": "libs/rgbcolor.js" },
    { "uri": "libs/StackBlur.js" },
    { "uri": "libs/canvg.js" },
    { "uri": "libs/jspdf.js" }
  ]
}
```

6. Controller logic: capture SmartChart SVG, render to canvas, export to PDF via jsPDF, and show with `sap.m.PDFViewer`.

```javascript
sap.ui.define(
  ["sap/ui/core/BusyIndicator", "sap/m/PDFViewer"],
  function (BusyIndicator, PDFViewer) {
    "use strict";
    return sap.ui.controller(
      "pdfprintui5.ext.controller.AnalyticalListPageExt",
      {
        onPrintChart: function () {
          BusyIndicator.show(0);
          this.smartChartId =
            "pdfprintui5::sap.suite.ui.generic.template.AnalyticalListPage.view.AnalyticalListPage::ZC_SFLIGHT_CAPABILITY_CDS--chart";
          this.chart = this.getView()
            .byId(this.smartChartId)
            .getItems()[2]
            .getAggregation("_vizFrame");
          this.chart.setWidth("1100px");
          this.chart.setHeight("700px");
          setTimeout(
            function () {
              this.processChartPDF();
            }.bind(this),
            100
          );
        },
        processChartPDF: function () {
          var svg = this.chart.getDomRef().getElementsByTagName("svg")[0];
          var canvas = document.createElement("canvas");
          var bBox = svg.getBBox();
          canvas.width = bBox.width;
          canvas.height = bBox.height;
          var context = canvas.getContext("2d");
          var imageObj = new Image();
          imageObj.src =
            "data:image/svg+xml," +
            jQuery.sap.encodeURL(
              svg.outerHTML.replace(
                /^<svg/,
                '<svg xmlns="http://www.w3.org/2000/svg" version="1.1"'
              )
            );
          imageObj.onload = function () {
            context.drawImage(imageObj, 0, 0, canvas.width, canvas.height);
            var dataURL = canvas.toDataURL("base64");
            const doc = new jsPDF("l", "px", "a4");
            doc.addImage(dataURL, "jpg", 10, 20);
            this.chart.setWidth("100%");
            this.chart.setHeight("100%");
            var blob = doc.output("blob", "chart_in_pdf.pdf");
            var _pdfurl = URL.createObjectURL(blob);
            this._pdfViewer = new PDFViewer();
            this.getView().addDependent(this._pdfViewer);
            this._PDFViewer = new sap.m.PDFViewer({
              width: "auto",
              source: _pdfurl,
              showDownloadButton: true,
            });
            jQuery.sap.addUrlWhitelist("blob");
            this._PDFViewer.open();
          }.bind(this);
          setTimeout(function () {
            BusyIndicator.hide();
          }, 100);
        },
      }
    );
  }
);
```

---

## Troubleshooting

- Blob URL blocked: add `jQuery.sap.addUrlWhitelist("blob")` before opening the `PDFViewer`.
- Chart not found: verify the exact SmartChart control id and the `getItems()[2]` path to the internal `_vizFrame` in your ALP version.
- Empty PDF: ensure the SVG is fully rendered before capture (delay) and avoid cross‑origin images in the SVG (breaks `toDataURL`).
- Missing libs: check that `rgbcolor.js`, `StackBlur.js`, `canvg.js`, and `jspdf.js` are loaded from `webapp/libs`.

---

## Acknowledgement

- ABAP CDS
- JavaScript / UI5
- Fiori Elements

---
