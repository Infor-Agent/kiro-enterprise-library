---
description: "Guides conversion of M3 H5 Classic UI scripts to V2 UI (Modern UI) scripts"
fileTriggers:
  - pattern: "**/*.js"
  - pattern: "**/*.ts"
activationKeywords:
  - "ScriptUtil.ApiRequest"
  - "MIService.Current"
  - "IonApiService.Current"
  - ".inforMessageDialog"
  - ".inforDataGrid"
  - ".inforBusyIndicator"
  - ".getSelectedRows()"
  - ".getDataItem("
  - ".readOnly()"
  - "ListControl.ListView"
  - "Configuration.Current.ListConfig"
---

# H5 Classic UI to V2 UI Script Migration

## How to Use

Open any `.js` or `.ts` H5 script containing Classic UI patterns and ask Kiro to "migrate this script to V2 UI". The skill activates automatically when it detects any of the activation keywords above.

### IMPORTANT: Ask the user which target version before migrating

Before applying any transformations, **ask the user**:

> "Which target version do you want to migrate to?"
> - **Version 2 only** (Angular/IDS framework — simpler code, uses V2-only APIs)
> - **Dual-compatible V1+V2** (works on both jQuery and Angular frameworks — uses `ScriptUtil.version` checks)

The key differences between targets:

| Pattern | V2-Only | Dual-Compatible (V1+V2) |
|---|---|---|
| MIService | `MIService` directly | `if (ScriptUtil.version >= 2.0) { MIService } else { MIService.Current }` |
| Grid data length | `list.getData().length` | `if (ScriptUtil.version >= 2.0) { list.getData().length } else { list.getData().getLength() }` |
| Grid data update | `list.setData(dataset)` | `if (ScriptUtil.version >= 2.0) { list.setData(dataset) } else { $.extend(list.getData().getItem(i), newData); list.setColumns(list.getColumns()); }` |
| Dialog creation | `H5ControlUtil.H5Dialog.CreateDialogElement(content[0], opts)` | `if (ScriptUtil.version >= 2.0) { H5ControlUtil.H5Dialog.CreateDialogElement(...) } else { content.inforMessageDialog(opts) }` |
| Dialog close | `model.close(true)` | `if (ScriptUtil.version >= 2.0) { model.close(true) } else { $(this).inforDialog("close") }` |
| Busy indicator | `controller.ShowBusyIndicator()` / `controller.HideBusyIndicator()` | `if (ScriptUtil.version >= 2.0) { controller.ShowBusyIndicator() } else { $("body").inforBusyIndicator({...}) }` |
| DatePicker | `new DatePickerElement()` | `if (ScriptUtil.version >= 2.0) { new DatePickerElement() } else { new TextBoxElement() + .inforDateField() }` |
| Checkbox toggle | `ScriptUtil.SetFieldValue(id, !value, controller)` | `if (ScriptUtil.version >= 2.0) { ScriptUtil.SetFieldValue(...) } else { $cbox.toggleChecked() }` |
| IonApiService | `IonApiService` directly | `if (ScriptUtil.version >= 2.0) { IonApiService } else { IonApiService.Current }` |
| RenderDataGrid | `ListControl.RenderDataGrid($list, columns, data, options)` | `if (ScriptUtil.version >= 2.0) { ListControl.RenderDataGrid(...) } else { Configuration.Current.ListConfig(...); $el.inforDataGrid(options) }` |

**If the user chooses "V2 only"** — apply the migration rules as documented below (current behavior).

**If the user chooses "Dual-compatible"** — wrap version-dependent patterns in `ScriptUtil.version >= 2.0` checks, keeping both V1 and V2 code paths. Use the H5SampleCustomColumns.ts pattern from the official Infor H5 Development Guide as the reference.

### Transformation Order (per function)

1. API Transaction Migration
2. Grid Statement Migration
3. Message Dialog Migration
4. Custom Dialog Migration
5. Function Content Migration
6. Flag any remaining Classic UI patterns as incompatible (add TODO comments and add "_incompatible" to the script file name)

---

## 1. V2 UI Script Structure Reference

All H5 V2 UI scripts follow this class pattern:

```javascript
var MyScript = /** @class */ (function () {
    function MyScript(scriptArgs) {
        this.controller = scriptArgs.controller;  // IInstanceController
        this.content = scriptArgs.controller.GetContentElement();
        this.log = scriptArgs.log;               // IScriptLog
        this.elem = scriptArgs.elem;
        this.args = scriptArgs.args;
        this.listView = ListControl.ListView;
        this.miService = MIService;
    }

    MyScript.Init = function (args) {
        new MyScript(args).run();
    };

    MyScript.prototype.run = function () {
        // Script logic here
    };

    return MyScript;
}());
```

### ES6 Class Pattern (Preferred for V2-only scripts)

For V2-only scripts (ScriptUtil.version >= 2.0), use the modern ES6 class syntax:

```javascript
var MyScript = class {
    constructor(scriptArgs) {
        this.controller = scriptArgs.controller;
        this.content = scriptArgs.controller.GetContentElement();
        this.log = scriptArgs.log;
        this.args = scriptArgs.args;
        this.element = scriptArgs.elem;
    }

    static Init(scriptArgs) {
        if (ScriptUtil.version >= 2.0) {
            new MyScript(scriptArgs).run();
        } else {
            console.error('[MyScript] Wrong H5 version, requires >= 2.0');
        }
    }

    run() {
        // Script logic here
    }
};
```

**IMPORTANT:** When migrating to V2-only, prefer the ES6 class pattern. It uses arrow functions for proper `this` binding and is the standard for Cloud M3 H5 scripts.

### Key V2 UI APIs

#### MIRequest (for MI API calls)

```javascript
var miRequest = new MIRequest();
miRequest.program = "CRS610MI";
miRequest.transaction = "GetBasicData";
miRequest.record = { CUNO: "CUSTOMER01" };
miRequest.outputFields = ["CUNM", "CUA1"];
miRequest.maxReturnedRecords = 1;  // optional
miRequest.includeMetadata = false; // optional
```

#### MIService (promise-based, no .Current)

```javascript
MIService.executeRequest(miRequest).then(function(response) {
    if (response.hasError()) {
        // Functional MI error (record not found, validation failure)
        // Access: response.errorMessage, response.errorCode, response.errorField
    } else {
        // Success — process response.items
    }
}).catch(function(error) {
    // Technical/network error only
});
```

**CRITICAL:** Use `response.hasError()` (method call) to check for errors — NOT `response.errorMessage` (property check).

#### IInstanceController Methods

`GetElement`, `GetGrid`, `GetValue`, `SetValue`, `GetProgramName`, `GetPanelName`, `GetView`, `PressKey`, `ListOption`, `ShowMessage`, `ShowMessageInStatusBar`

Events: `Requesting`, `Requested`, `RequestCompleted` (use `.On(handler)` / `.Off(handler)`)

#### IActiveGrid (from controller.GetGrid())

`getColumns()`, `setColumns(cols)`, `getData()`, `setData(data)`, `getSelectedGridRows()`, `setSelectedRows(rows)`

#### ConfirmDialog

```javascript
ConfirmDialog.ShowMessageDialog({
    dialogType: "Alert",  // or "Question", "Error"
    header: "Title",
    message: "Body text",
    closed: function(choice) { if (choice.ok) { /* OK clicked */ } }
});
```

#### H5ControlUtil.H5Dialog

`CreateDialogElement(content: HTMLElement, dialogOptions)` — creates V2 dialog
Close with `model.close(true)` (not DestroyDialog)

#### Other Utilities

- `ScriptUtil.GetUserContext()` — user context (company, division)
- `ScriptUtil.getController().showBusyIndicator()` / `.hideBusyIndicator()`
- `IonApiService.execute(request)` — ION API (no `.Current`)
- `InstanceCache.Add/Get/Remove/ContainsKey(controller, key, val)`
- `SessionCache.Add/Get/Remove/ContainsKey(key, val)`
- `ListControl.GetColumnIndexByName(colName)`, `ListControl.ListView.GetValueByColumnName(colName)`

#### UI Element Classes

`ButtonElement`, `TextBoxElement`, `CheckBoxElement`, `ComboBoxElement`, `LabelElement`, `ListElement`, `TextAreaElement`, `DatePickerElement` (V2-specific, replaces TextBoxElement for date fields)

---

## 2. API Transaction Migration

### 2.1 ScriptUtil.ApiRequest → MIService.executeRequestV2

**Before:**
```javascript
var url = "/execute/MMS200MI/Get;returncols=ITDS,FUDS;metadata=false;maxrecs=1?ITNO=" + ITNO;
ScriptUtil.ApiRequest(url, this.onSuccess.bind(this), this.onError.bind(this));
```

**After:**
```javascript
var miRequest = new MIRequest();
miRequest.program = "MMS200MI";
miRequest.transaction = "Get";
miRequest.record = { ITNO: ITNO };
miRequest.outputFields = ["ITDS", "FUDS"];
miRequest.includeMetadata = false;
miRequest.maxReturnedRecords = 1;

MIService.executeRequestV2(miRequest)
    .then((result) => { this.onSuccess.bind(this)(result); })
    .catch((error) => { this.onError.bind(this)('Error', error.error?.terminationReason ?? error.message ?? error); })
```

#### URL Parsing Rules

Parse `/execute/{Program}/{Transaction}[;params]*[?fields]` into MIRequest:
- Program/Transaction → `miRequest.program`, `miRequest.transaction`
- `;returncols=A,B` → `miRequest.outputFields = ["A", "B"]`
- `;metadata=false` → `miRequest.includeMetadata = false`
- `;maxrecs=1` → `miRequest.maxReturnedRecords = 1`
- `;excludeEmpty=true` → `miRequest.excludeEmpty = true`
- `?FIELD=value&FIELD2=value2` → `miRequest.record = { FIELD: value, FIELD2: value2 }`

#### Error Callback Rules

- **Simple function reference** (e.g., `onErrorFn`): `.catch((error) => { onErrorFn('Error', error.error?.terminationReason ?? error.message ?? error); })`
- **Inline function with 2 params** (error, header): Remove header param, insert `var header = 'Error';` as first line in catch body
- **Inline function with 1 param**: Place body directly in `.catch()`
- **`.bind(this)` references**: Preserve as-is

#### Dynamic URL (cannot parse)

If the URL is a variable without an inline `/execute/` literal, emit:
```javascript
// TODO [H5 Migration]: Replace ScriptUtil.ApiRequest with MIService.executeV2(program, transaction, inputParameters) - URL could not be parsed automatically
```

### 2.2 MIService.Current Error Handling Restructuring

Classic UI routes functional MI errors to `.catch()`. V2 UI routes them to `.then()` via `response.hasError()`. Restructure accordingly.

**Before:**
```javascript
MIService.Current.executeRequest(myRequest).then(function(response) {
    processResults(response.items);
}).catch(function(error) {
    _this.log.Error(error.errorMessage);
});
```

**After:**
```javascript
MIService.executeRequest(myRequest).then(function(response) {
    if(response.hasError()) {
        _this.log.Error(response.errorMessage);
    } else {
        processResults(response.items);
    }
}).catch(function(error) {
    _this.log.Error(error.errorMessage);
});
```

**Rules:**
1. Remove `.Current` from `MIService.Current` and `IonApiService.Current`
2. Wrap `.then()` body in `if(response.hasError()) { catch-logic } else { original-then-logic }`
3. Replace `error.` with `response.` in the moved catch logic (dot-property access only)
4. Preserve original `.catch()` unchanged (for technical/network errors)

### 2.3 Response Property Replacements

| Classic UI | V2 UI |
|---|---|
| `result.MIRecord` | `result.items` |
| `result.Message` | `result.errorMessage` |
| `error.responseJSON.Message` | `error.error.terminationReason ?? error.message ?? error` |
| `MIService.Current` | `MIService` |
| `IonApiService.Current` | `IonApiService` |

### 2.4 MIRecord NameValue Loop → Compatibility Shim

**Before:**
```javascript
var nameValue = miRecords[i].NameValue;
```

**After:**
```javascript
var nameValue = [];
for (const property in miRecords[i]) {
    nameValue.push({Name: property, Value: miRecords[i][property]});
}
```

This preserves downstream `nameValue[j].Name` / `nameValue[j].Value` access patterns.

---

## 3. Grid Operations Migration

### Rules (apply in order)

| Classic UI | V2 UI |
|---|---|
| `.getSelectedRows()` | `.getSelectedGridRows()` |
| `selectedRows[0]` (after getSelectedGridRows) | `selectedRows[0].idx` |
| `.getDataItem(row)["C1"]` | `.getData()[row][grid.getColumns().find(c => c.colId == "C1").fullName]` |
| `.getDataItem(rowIndex)` (assignment) | `.getData()[rowIndex]` |
| `controller.GetGrid().getDataItem(row)["col"]` | `controller.GetGrid().getData()[row][controller.GetGrid().getColumns().find(c => c.colId == "col").fullName]` |
| `.getData().getLength()` | `.getData().length` |
| `.getColumnIndex(grid.getColumns().find(c => c.colFld == 'F').id)` | `.getColumns().find(c => c.fullName == 'F').index` |
| `row["C1"]` (after row = getData()[i]) | `row[grid.getColumns().find(c => c.colId == "C1").fullName]` |
| `.colFld` | `.fullName` |
| `$.extend(grid.getData().getItem(i), newData)` | `grid.setData(grid.getData())` |
| `newData = {}` (paired with $.extend above) | `newData = grid.getData()[i]` |
| `.getData().deleteItem(rowId)` | `.removeRow(rowId)` |
| `.reinit()` | *(remove — not needed in V2)* |

### Complete Example

**Before:**
```javascript
var list = this.controller.GetGrid();
var selectedRows = list.getSelectedRows();
var rowIndex = selectedRows[0];
var row = list.getDataItem(rowIndex);
var ITNO = row["C1"];
var count = list.getData().getLength();
var newData = {};
newData.STAT = "20";
$.extend(list.getData().getItem(rowIndex), newData);
list.reinit();
```

**After:**
```javascript
var list = this.controller.GetGrid();
var selectedRows = list.getSelectedGridRows();
var rowIndex = selectedRows[0].idx;
var row = list.getData()[rowIndex];
var ITNO = row[list.getColumns().find(c => c.colId == "C1").fullName];
var count = list.getData().length;
var newData = list.getData()[rowIndex];
newData.STAT = "20";
list.setData(list.getData());
```

### Graceful Degradation

If a grid variable cannot be determined (e.g., grid passed as parameter without prior assignment), leave the statement unchanged and flag it for manual review.

---

## 4. Message Dialog Migration

### ShowDialog → ConfirmDialog.ShowMessageDialog

| Scenario | Replacement |
|---|---|
| `ShowDialog(header, detail, ...)` where detail ≠ null | `ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: header, message: detail })` |
| `ShowDialog(message, null, ...)` | `ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: "", message: message })` |
| `ShowDialog(singleArg)` (< 2 args) | Flag as incompatible — add TODO comment |

3rd and 4th arguments are always discarded.

**Before:**
```javascript
this.controller.Controller.ShowDialog("Failed", result["Message"], null, true);
```

**After:**
```javascript
ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: "Failed", message: result["Message"]});
```

---

## 5. Custom Dialog Migration

Applies only to functions containing `.inforMessageDialog(`. Apply in order:

### 5.1 Insert `this`-capturing variable

If no `var _this = this` / `const self = this` exists, insert `var self = this;` (JS) or `const self = this;` (TS) at function start.

### 5.2 Relocate `open:` handler

Extract `open: function(){body}` content, move it to execute after the `CreateDialogElement` call, remove the `open` property from options.

### 5.3 Replace patterns

| Classic UI | V2 UI |
|---|---|
| `content.inforMessageDialog(opts)` | `H5ControlUtil.H5Dialog.CreateDialogElement(content[0], opts)` |
| `click: function()` | `click: function(event, model)` |
| `H5ControlUtil.H5Dialog.DestroyDialog(...)` | `model.close(true)` |
| `Configuration.Current.ListConfig(id, cols, data)` | *(remove statement — store cols/data)* |
| `.inforDataGrid(options)` | `.datagrid({columns: cols, dataset: data})` |

### 5.4 Replace `this.method()` in click handlers

Inside click handler bodies, replace `this.classMethod()` with `self.classMethod()` (or `_this.classMethod()` if that variable already existed). Only replace calls to methods that exist on the same class.

### Complete Example

**Before:**
```javascript
MyScript.prototype.showDialog = function() {
    var options = Configuration.Current.ListConfig('id', columns, data);
    var dialogContent = $('<div></div>');
    var dialogOptions = {
        open: function() { $('#grid').inforDataGrid(options); },
        buttons: [{ text: 'OK', click: function() {
            this.save();
            H5ControlUtil.H5Dialog.DestroyDialog($(dialogContent));
        }}]
    };
    dialogContent.inforMessageDialog(dialogOptions);
};
```

**After:**
```javascript
MyScript.prototype.showDialog = function() {
    var self = this;
    var dialogContent = $('<div></div>');
    var dialogOptions = {
        buttons: [{ text: 'OK', click: function(event, model) {
            self.save();
            model.close(true);
        }}]
    };
    H5ControlUtil.H5Dialog.CreateDialogElement(dialogContent[0], dialogOptions);
    $('#grid').datagrid({columns: columns, dataset: data});
};
```

---

## 6. Function Content Migration

Apply these replacements in order:

| # | Classic UI | V2 UI |
|---|---|---|
| 1 | `userContext['prop']` | `ScriptUtil.GetUserContext()['prop']` |
| 2 | `userContext.prop` | `ScriptUtil.GetUserContext().prop` |
| 3 | `ScriptDebugConsole.WriteLine(text)` | `console.log(text)` |
| 4 | `.GetContentElement().GetElement(f).readOnly()` | `.GetElement(f).readonly()` |
| 5 | `.readOnly()` (standalone) | `.readonly()` |
| 6 | `$("body").inforBusyIndicator({...})` | `ScriptUtil.getController().showBusyIndicator()` |
| 7 | `$("body").inforBusyIndicator("close")` | `ScriptUtil.getController().hideBusyIndicator()` |
| 8 | `.GetElement(f).prop("disabled", true)` | `.GetElement(f).readonly()` |
| 9 | `.GetElement(f).prop("disabled", false)` | `.GetElement(f).enable()` |
| 10 | `.GetContentElement().GetElement(f)` (standalone) | `.GetElement(f)` |
| 11 | `element.inforDateField(...)` | *(remove — DatePickerElement handles this)* |
| 12-13 | `new TextBoxElement()` (when linked to inforDateField) | `new DatePickerElement()` |

**Important:** Rule 4 must run before Rule 5 (combined GetContentElement+readOnly before standalone readOnly). Rule 8/9 only apply when the `.prop()` argument is literally `"disabled"`.

---

## 7. Incompatible Statement Detection

After all transformations, flag any remaining Classic UI patterns with a TODO comment above the statement:

```javascript
// TODO [H5 Migration]: <description of required manual action>
```

### Patterns to Flag

| Pattern | TODO Description |
|---|---|
| `ScriptUtil.ApiRequest(` | Replace with MIService.executeV2(program, transaction, inputParameters) |
| `.getSelectedRows()` | Replace with .getSelectedGridRows() |
| `.getDataItem(` | Replace with .getData()[] |
| `.getLength()` | Replace with .length |
| `.getColumnIndex(` | Replace with .getColumns() lookup |
| `.deleteItem(` | Replace with .removeRow() |
| `.inforMessageDialog(` | Replace with H5ControlUtil.H5Dialog.CreateDialogElement() |
| `H5ControlUtil.H5Dialog.DestroyDialog(...)` | Replace with model.close(true) |
| `.ListConfig(` | Remove and pass cols/data to .datagrid() |
| `.inforDataGrid(` | Replace with .datagrid({columns, dataset}) |
| `.readOnly(` (capital O) | Replace with .readonly() |
| `.inforBusyIndicator(` | Replace with ScriptUtil.getController().showBusyIndicator() or hideBusyIndicator() |

**Rules:**
- Each incompatible statement gets its own TODO comment
- Original code is preserved unchanged (never modify incompatible statements)
- Continue processing even if individual flag/comment actions fail


---

## 8. Dual-Compatible Pattern Reference (V1+V2)

When the user requests dual-compatible output, use `ScriptUtil.version >= 2.0` checks. Based on the official Infor M3 H5 Development Guide (Version 10.4.1, November 2023).

### MIService Access

```javascript
// Dual-compatible
if (ScriptUtil.version >= 2.0) {
    this.miService = MIService;
} else {
    this.miService = MIService.Current;
}
// Then use this.miService.executeRequest(...) or this.miService.execute(...)
```

### Grid Data Population (Custom Columns)

```javascript
// Dual-compatible
if (ScriptUtil.version >= 2.0) {
    const dataset = list.getData();
    for (let i = 0; i < dataset.length; i++) {
        dataset[i][columnId] = "Data" + i;
    }
    list.setData(dataset);
} else {
    for (let i = 0; i < list.getData().getLength(); i++) {
        let newData = {};
        newData[columnId] = "Data" + i;
        newData["id_" + columnId] = "R" + (i + 1) + columnId;
        $.extend(list.getData().getItem(i), newData);
    }
    list.setColumns(list.getColumns());
}
```

### Dialog Creation

```javascript
// Dual-compatible
if (ScriptUtil.version >= 2.0) {
    H5ControlUtil.H5Dialog.CreateDialogElement(dialogContent[0], dialogOptions);
} else {
    dialogContent.inforMessageDialog(dialogOptions);
}
```

### Dialog Close (inside click handler)

```javascript
// Dual-compatible click handler
click: function(event, model) {
    // ... logic ...
    if (ScriptUtil.version >= 2.0) {
        model.close(true);
    } else {
        $(this).inforDialog("close");
    }
}
```

### DatePicker Element

```javascript
// Dual-compatible
if (ScriptUtil.version >= 2.0) {
    const datepickerElement = new DatePickerElement();
    datepickerElement.Name = name;
    datepickerElement.Position = position;
    datepickerElement.DateFormat = "DDMMYY";
    controller.GetContentElement().AddElement(datepickerElement);
} else {
    const textElement = new TextBoxElement();
    textElement.Name = name;
    textElement.DateFormat = "ddMMyy";
    textElement.Position = position;
    const datepicker = controller.GetContentElement().AddElement(textElement);
    datepicker.inforDateField({ hasInitialValue: false, openOnEnter: false, dateFormat: "ddMMyy" });
}
```

### RenderDataGrid (Custom List)

```javascript
// Dual-compatible
if (ScriptUtil.version >= 2.0) {
    const options = { forceFitColumns: true, autoHeight: true };
    ListControl.RenderDataGrid($list, columns, data, options);
} else {
    const options = Configuration.Current.ListConfig('id', columns, data);
    options["forceFitColumns"] = true;
    options["autoHeight"] = true;
    const grid = $("#myGrid").inforDataGrid(options);
    grid.render();
}
```

### Busy Indicator

```javascript
// Dual-compatible — show
if (ScriptUtil.version >= 2.0) {
    controller.ShowBusyIndicator();
} else {
    $("body").inforBusyIndicator({ delay: 100, modal: true });
}

// Dual-compatible — hide
if (ScriptUtil.version >= 2.0) {
    controller.HideBusyIndicator();
} else {
    $("body").inforBusyIndicator("close");
}
```

### Key V2-Only APIs (not available in V1)

These APIs only exist when `ScriptUtil.version >= 2.0`:
- `controller.ShowBusyIndicator()` / `controller.HideBusyIndicator()`
- `controller.ExportToExcel()` / `controller.ExportToGoogleSheets()`
- `controller.OpenFieldHelp(id)`
- `H5ControlUtil.H5Dialog.CreateDialogElement(content, options)`
- `ListControl.RenderDataGrid($list, columns, rows, options)`
- `ListControl.ListView.GetDatagrid(controller)`
- `ListControl.GetListColumnData(columnName, controller)`
- `ContentElement.RemoveScriptComponents()`
- `MIService.createMiRequest(program, transaction, record, maxRecords)`
- `ScriptUtil.UnloadScript(url)`
- `new DatePickerElement()` / `new RadioGroupElement()`
- `IActiveGrid.hideColumns(columnNames[])`

### Official Reference

Based on: Infor M3 H5 Development Guide, Version 10.4.1, Published November 2023.
