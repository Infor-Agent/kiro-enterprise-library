---
inclusion: fileMatch
fileMatchPattern: "**/SmartOfficeScripts/**"
---

# Smart Office Script to M3 Cloud H5 Script Migration Guide

## Introduction

This steering file provides migration rules for converting M3 Smart Office scripts (JScript/TypeScript with .NET-style APIs) to M3 Cloud H5 V2 UI scripts (TypeScript with H5 framework APIs). Smart Office scripts use C#-like patterns, WPF UI controls, and Odin framework APIs that must be converted to web-based equivalents.

### Activation Keywords

The following patterns indicate Smart Office code requiring migration:

- `import MForms;`
- `import Lawson.M3.MI;`
- `import Mango.`
- `package MForms.JScript`
- `MIWorker.Run`
- `MIAccess.Execute`
- `new MIRecord()`
- `MIResponse`
- `BackgroundWorker`
- `UserContext.CurrentCompany`
- `ScriptUtil.FindChild`
- `.RenderEngine.Content`
- `.RenderEngine.ListViewControl`
- `.add_Requested(`
- `.add_Requesting(`
- `.remove_Requested(`
- `.remove_Requesting(`
- `ConfirmDialog.Show`
- `DashboardTaskService.Manager.LaunchTask`
- `new MFormsAutomation()`
- `Grid.SetColumn(`
- `Grid.SetRow(`
- `content.Children.Add(`

## Source Format

Smart Office scripts are JScript files with:
- C# import statements (`import System;`, `import MForms;`, etc.)
- Package declaration (`package MForms.JScript { ... }`)
- Class-based structure with typed variables
- .NET/WPF UI patterns (Grid layout, Button, TextBlock, etc.)
- Odin framework APIs for MI access

## Target Format

The output is TypeScript (`.ts`) compatible with H5 V2 UI framework using:
- No import statements (H5 provides globals)
- Class with `static Init(scriptArgs: IScriptArgs)` entry point
- `IInstanceController`, `IContentElement`, `IScriptLog` interfaces
- `MIService.executeV2()` for MI API calls
- H5 UI elements (`ButtonElement`, `TextBoxElement`, etc.) with `PositionElement`
- Event subscription via `.Requesting.On()` / `.Requested.On()`

## Transformation Phases

Apply in this order:

### Phase 1: Structural Transformation

1. **Remove imports** — Delete all `import` statements
2. **Remove package wrapper** — Remove `package MForms.JScript {` and its closing `}`
3. **Convert class structure** — Transform to H5 class pattern with `static Init(scriptArgs: IScriptArgs)`
4. **Convert Init function** — Map 4-arg Init to H5 scriptArgs pattern:
   - `element` → `scriptArgs.elem`
   - `args` → `scriptArgs.args`
   - `controller` → `scriptArgs.controller` (typed as `IInstanceController`)
   - `debug` → `scriptArgs.log` (typed as `IScriptLog`)
5. **Convert global vars** — `var` at class level → `private`; add types where identifiable
6. **Prefix global vars with `this.`** — Inside functions, global variables get `this.` prefix
7. **Prefix function calls with `this.`** — Calls to class methods get `this.` prefix

### Phase 2: API Migration (MIWorker/MIAccess → MIService)

| Smart Office | H5 V2 UI |
|---|---|
| `new MIRecord()` | `{}` (plain object) |
| `record["FIELD"] = value` | `record.FIELD = value` or `record["FIELD"] = value` |
| `MIAccess.Execute(program, transaction, record)` | `await (MIService as any).executeV2(program, transaction, record)` |
| `MIWorker.Run(program, transaction, record, callback)` | `(MIService as any).executeV2(program, transaction, record).then((response: IMIResponse) => { this.callback(response) }).catch((response: IMIResponse) => { this.callback(response) })` |
| `MIWorker.Run(prog, trans, record, callback, outputFields)` | `(MIService as any).executeV2(prog, trans, record, outputFields).then(...)` |
| `new MIParameters("A","B")` | `["A","B"]` |
| `response.HasError` | `response.hasError()` |
| `response.Item["FIELD"]` | `response.item["FIELD"]` |
| `response.Items` | `response.items` |
| `response.Error` | `response.error` |
| `MIResponse` (type) | `IMIResponse` |

### Phase 3: BackgroundWorker → Promise

**Before:**
```javascript
var worker = new BackgroundWorker();
worker.add_DoWork(OnDoWorkFunction);
worker.add_RunWorkerCompleted(OnCompletedFunction);
worker.RunWorkerAsync();
```

**After:**
```typescript
this.OnDoWorkFunction().then((result) => {
    this.OnCompletedFunction(false, null)
}).catch(error => {
    this.OnCompletedFunction(true, error)
});
```

The DoWork function should become an `async` function returning a Promise.

### Phase 4: Event Subscription Migration

| Smart Office | H5 V2 UI |
|---|---|
| `controller.add_Requested(OnRequested)` | `this.unsubscribeRequested = this.controller.Requested.On((e) => { this.OnRequested(this.controller, e); })` |
| `controller.add_Requesting(OnRequesting)` | `this.unsubscribeRequesting = this.controller.Requesting.On((e) => { this.OnRequesting(this.controller, e); })` |
| `controller.remove_Requested(OnRequested)` | `this.unsubscribeRequested()` |
| `controller.remove_Requesting(OnRequesting)` | `this.unsubscribeRequesting()` |

Note: Each `add_Requested`/`add_Requesting` creates a new `private unsubscribeRequested: Function` / `private unsubscribeRequesting: Function` global variable.

### Phase 5: UI Element Migration

| Smart Office | H5 V2 UI |
|---|---|
| `new Button()` | `new ButtonElement()` + `button.Position = new PositionElement()` |
| `button.Content = "text"` | `button.Value = "text"` |
| `button.Width = N` | `button.Position.Width = N` |
| `button.Height = N` | `button.Position.Height = N` |
| `Grid.SetColumn(el, N)` | `el.Position.Left = N` |
| `Grid.SetRow(el, N)` | `el.Position.Top = N` |
| `Grid.SetColumnSpan(el, N)` | `// Grid.SetColumnSpan(el, N)` (comment out) |
| `Grid.SetRowSpan(el, N)` | `// Grid.SetRowSpan(el, N)` (comment out) |
| `content.Children.Add(button)` | `var $button = this.content.AddElement(button)` |
| `button.add_Click(onClick)` | `$button.click({}, () => { this.onClick(this, null) })` |
| `button.remove_Click(onClick)` | `button.OnClick = null` |
| `button.add_Unloaded(onUnload)` | `//button.add_Unloaded(onUnload)` (comment out — not supported) |

### Phase 6: Content/Field Access Migration

| Smart Office | H5 V2 UI |
|---|---|
| `controller.RenderEngine.Content` | `controller.GetContentElement()` |
| `controller.RenderEngine.ListViewControl` | `controller.GetGrid()` |
| `controller.RenderEngine.ListControl` | `controller.GetGrid()` |
| `ScriptUtil.FindChild(content, "FIELD")` | `(document.getElementById("FIELD") as HTMLInputElement)` |
| `ScriptUtil.FindChild(content, "FIELD").Text` | `ScriptUtil.GetFieldValue("FIELD")` |
| `element.Text = value` (HTMLInputElement) | `element.value = value` |
| `element.Focus()` | `element.focus()` |

### Phase 7: Utility Replacements

| Smart Office | H5 V2 UI |
|---|---|
| `debug.WriteLine(text)` / `scriptLog.WriteLine(text)` | `this.log.Info(text)` |
| `UserContext.UserName` | `ScriptUtil.GetUserContext().USID` |
| `UserContext.CurrentCompany` | `ScriptUtil.GetUserContext().CurrentCompany` |
| `UserContext.CurrentDivision` | `ScriptUtil.GetUserContext().CurrentDivision` |
| `UserContext.Facility` | `ScriptUtil.GetUserContext().FACI` |
| `UserContext.Warehouse` | `ScriptUtil.GetUserContext().WHLO` |
| `.Trim()` | `.trim()` |
| `.Equals(x)` | `=== (x)` |
| `.Contains(x)` | `.includes(x)` |
| `.GetString("FIELD")` | `["FIELD"]` |
| `e.CommandType` | `e.commandType` |
| `e.CommandValue` | `e.commandValue` |
| `e.Cancel = true` | `e.cancel = true` |
| `for (var x in items)` | `for (var x of items)` |
| `DateTime.Now.ToString("yyyyMMdd")` | `new Date().toISOString().slice(0,10).replaceAll('-', '')` |
| `DashboardTaskService.Manager.LaunchTask(new Task(uri))` | `ScriptUtil.Launch(uri)` |

### Phase 8: MFormsAutomation Migration

| Smart Office | H5 V2 UI |
|---|---|
| `auto.AddStep(...)` | `auto.addStep(...)` |
| `auto.AddField(...)` | `auto.addField(...)` |
| `auto.ToUri()` | `auto.toEncodedUri()` |

### Phase 9: Dialog Migration

| Smart Office | H5 V2 UI |
|---|---|
| `ConfirmDialog.ShowAttentionDialog(header, message)` | `ConfirmDialog.Show({header: header, message: message, dialogType: "Attention", withCancelButton: false, canHide: false})` |
| `ConfirmDialog.ShowErrorDialog(header, message)` | `ConfirmDialog.Show({header: header, message: message, dialogType: "Error", withCancelButton: false, canHide: false})` |
| `ConfirmDialog.ShowInformationDialog(header, message)` | `ConfirmDialog.Show({header: header, message: message, dialogType: "Info", withCancelButton: false, canHide: true})` |
| `ConfirmDialog.ShowWarningDialog(header, message)` | `ConfirmDialog.Show({header: header, message: message, dialogType: "Warning", withCancelButton: false, canHide: false})` |
| `ConfirmDialog.ShowQuestionDialog(header, message)` | `await ConfirmDialogHelper.ShowQuestionDialog(header, message, false, false)` |

### Phase 10: Type Replacements

| Smart Office Type | H5 V2 UI Type |
|---|---|
| `int` | `number` |
| `double` | `number` |
| `String` (capital S) | `string` |
| `Button` | `ButtonElement` |
| `Object` | `any` |
| `MIResponse` | `IMIResponse` |
| `RoutedEventArgs` | `any` |
| `DateTime` | `Date` |
| `Exception` | `Error` |

### Phase 11: Grid Operations (SelectedItems/Columns)

| Smart Office | H5 V2 UI |
|---|---|
| `list.SelectedItems.Count` | `list.getSelectedGridRows().length` |
| `list.SelectedItems` | `list.getSelectedGridRows()` |
| `list.Columns.Count` | `list.getColumns().length` |
| `list.Columns[i]` | `list.getColumns()[i].name` |
| `list.Columns` | `list.getColumns()` |
| `list.GetColumnValue("COL")` | `list.getSelectedGridRows()[0].data[list.getColumns().filter(c => c.name === "COL")[0].fullName]` |

## Output Structure

The migrated TypeScript file should follow this structure:

```typescript
class ClassName {
    private controller: IInstanceController;
    private content: IContentElement;
    private log: IScriptLog;
    private unsubscribeRequested: Function;
    private unsubscribeRequesting: Function;
    // ... other private members

    static Init(scriptArgs: IScriptArgs): void {
        new ClassName().run(scriptArgs.elem, scriptArgs.args, scriptArgs.controller, scriptArgs.log);
    }

    private function run(element: any, args: string, controller: IInstanceController, log: IScriptLog): void {
        this.controller = controller;
        this.content = controller.GetContentElement();
        this.log = log;
        // ... initialization logic
    }

    // ... other methods
}
```

## MFormsAutomation Full Workflow

### Creating and Executing MFormsAutomation

**Before (Smart Office):**
```javascript
var auto = new MFormsAutomation();
auto.AddStep(ActionType.Run, "QMS600");
auto.AddField("WEFACI", psFACI);
auto.AddField("WEITNO", psITNO);
var uri = auto.ToUri();
DashboardTaskService.Manager.LaunchTask(new Task(uri));
```

**After (H5 V2):**
```typescript
var auto = new MFormsAutomation();
auto.addStep(ActionType.Run, "QMS600");
auto.addField("WEFACI", psFACI);
auto.addField("WEITNO", psITNO);
var uri = auto.toEncodedUri();
ScriptUtil.Launch(uri);
```

### Bookmark Pattern

**Before (Smart Office):**
```javascript
DashboardTaskService.Manager.LaunchTask(new Task("mforms://bookmark?program=CRS620&..."));
```

**After (H5 V2):**
```typescript
ScriptUtil.Launch("mforms://bookmark?program=CRS620&...");
```

## IonApiService for External API Calls

Smart Office scripts use `HttpWebRequest`/`WebRequest.Create()` for external HTTP calls. In H5 Cloud, use `IonApiService` to call external APIs routed through the Mingle/ION API gateway.

### H5 V1 Pattern (IonApiService.Current)

```javascript
// TODO [H5 Migration]: Configure the Mingle endpoint URL per environment
var mingleEndpoint = 'CustomerApi/mulesoft-api/postAddressValidation/address-validation';
var request = {
    url: "/" + mingleEndpoint,
    method: "POST",
    headers: {
        "Accept": "application/xml",
        "Content-Type": "application/xml"
    },
    data: messageContent
};

IonApiService.Current.execute(request).then(function(response) {
    console.log(response.status);
    // In V1, successful responses may arrive here
}).catch(function(Error) {
    // NOTE: In V1, the actual response body often arrives here via Error.error.text
    _this.parseResponse(Error.error.text);
});
```

### H5 V2 Pattern (IonApiService — no .Current)

```javascript
// TODO [H5 Migration]: Configure the Mingle endpoint URL per environment
var mingleEndpoint = 'CustomerApi/mulesoft-api/postAddressValidation/address-validation';
var request = {
    url: "/" + mingleEndpoint,
    method: "POST",
    headers: {
        "Accept": "application/xml",
        "Content-Type": "application/xml"
    },
    data: messageContent
};

IonApiService.execute(request).then(function(response) {
    // V2: success response body is here
    _this.parseResponse(response.data || response.body);
}).catch(function(error) {
    _this.log.Error("ION API error: " + (error.errorMessage || error.message || error));
});
```

### Dual-Compatible IonApiService

```javascript
var ionService;
if (ScriptUtil.version >= 2.0) {
    ionService = IonApiService;
} else {
    ionService = IonApiService.Current;
}

ionService.execute(request).then(function(response) {
    // handle response
}).catch(function(error) {
    // In V1, actual response body may be in error.error.text
    // In V2, this is a real error
});
```

### Key Points

- The URL is a **Mingle-relative path** (e.g., `CustomerApi/mulesoft-api/...`), NOT a full external URL
- Authentication is handled by the Mingle/ION gateway — no credentials in the script
- In V1 (`.Current`), the response body sometimes comes through `.catch()` via `Error.error.text`
- In V2 (no `.Current`), response routing follows standard Promise semantics
- The endpoint URL itself should be flagged as TODO (environment-specific):

```javascript
// TODO [H5 Migration]: The Mingle endpoint URL may differ per environment. Configure via ION API gateway.
```

### Additional Learned Patterns from Real Migrations

**MIService usage in H5 V1 (with MIRequest):**
```javascript
var request = new MIRequest();
request.program = "CRS610MI";
request.transaction = "GetOrderInfo";
request.record = { CONO: this.gCompany, CUNO: customerNum };
request.outputFields = ["CUCL"];

MIService.Current.executeRequest(request).then(function(response) {
    // V1: response.items[0].FIELDNAME
    var value = response.items[0].CUCL;
}).catch(function(response) {
    console.log(response.errorMessage);
});
```

**ScriptUtil.SetFieldValue for setting panel fields:**
```javascript
ScriptUtil.SetFieldValue("WRCUA1", addressLine1);
ScriptUtil.SetFieldValue("WRTOWN", city);
```

**ScriptUtil.GetFieldValue for reading panel fields:**
```javascript
var addr1 = ScriptUtil.GetFieldValue("WRCUA1");
```

**ComboBoxElement with ComboBoxItemElement for dropdowns:**
```javascript
this.comboBoxElem = new ComboBoxElement();
this.comboBoxElem.Name = "ComboBox";
this.comboBoxElem.Position = new PositionElement();
this.comboBoxElem.Position.Top = 8;
this.comboBoxElem.Position.Left = 80;
this.comboBoxElem.Position.Width = 50;

var cboxItem = new ComboBoxItemElement();
cboxItem.Value = "0";
cboxItem.Text = "Option text here";
cboxItem.IsSelected = true;
this.comboBoxElem.Items.push(cboxItem);
this.content.AddElement(this.comboBoxElem);
```

**Requesting event to intercept ENTER key for validation:**
```javascript
this.unsubscribeRequesting = this.instanceController.Requesting.On(function(e) {
    if (e.commandType === "KEY" && e.commandValue === "ENTER") {
        if (_this.allowEnter) {
            _this.allowEnter = false;
            return;
        }
        e.cancel = true;
        _this.validateAsync().then(function(isValid) {
            if (isValid) {
                _this.allowEnter = true;
                _this.instanceController.PressKey("ENTER");
            }
        });
    }
});
```

**DOMParser for XML parsing (replaces System.Xml):**
```javascript
var parser = new DOMParser();
var xmlDoc = parser.parseFromString(xmlPayload, 'text/xml');
var errorNode = xmlDoc.querySelector('parsererror');
if (errorNode) {
    throw new Error("XML Parsing Error: " + errorNode.textContent);
}
var nodes = xmlDoc.getElementsByTagName('ns0:CandidateAddress');
```

**Important:** If the external call is NOT routable through Mingle/ION (e.g., direct third-party URL with no gateway proxy), there is NO H5 equivalent. Flag with:
```javascript
// TODO [H5 Migration]: Direct HTTP to external URL has no H5 equivalent. Must route through ION API gateway.
```

## Multi-Record MI Response Iteration

When transactions return multiple records (e.g., `LstXxx`, `SelXxx`):

**Before (Smart Office):**
```javascript
function OnResponseList(response: MIResponse) {
    if (!response.HasError) {
        for (var item in response.Items) {
            var itno = item.GetString("ITNO");
            var itds = item.GetString("ITDS");
        }
    }
}
```

**After (H5 V2):**
```typescript
(MIService as any).executeV2("MMS200MI", "LstItmBasic", record).then((response: IMIResponse) => {
    if (!response.hasError()) {
        for (var item of response.items) {
            var itno = item["ITNO"];
            var itds = item["ITDS"];
        }
    }
}).catch((response: IMIResponse) => {
    this.log.Error("MI error: " + response.error);
});
```

**Key differences:**
- `response.Items` → `response.items`
- `for (var item in ...)` → `for (var item of ...)`
- `item.GetString("FIELD")` → `item["FIELD"]`
- `response.HasError` → `response.hasError()` (function call, not property)

## RequestCompleted Event

In addition to `Requesting` and `Requested`, Smart Office has `RequestCompleted`:

| Smart Office | H5 V2 UI |
|---|---|
| `controller.add_RequestCompleted(OnRequestCompleted)` | `this.unsubscribeRequestCompleted = this.controller.RequestCompleted.On((e) => { this.OnRequestCompleted(this.controller, e); })` |
| `controller.remove_RequestCompleted(OnRequestCompleted)` | `this.unsubscribeRequestCompleted()` |

This creates a new `private unsubscribeRequestCompleted: Function` global variable.

## Panel Navigation Detection

**Before (Smart Office):**
```javascript
if (controller.RenderEngine.PanelHeader == "OIS101/B") {
    // on list panel
}
```

**After (H5 V2):**
```typescript
if (this.controller.GetPanelName() == "OIS101/B") {
    // on list panel
}
```

| Smart Office | H5 V2 UI |
|---|---|
| `controller.RenderEngine.PanelHeader` | `controller.GetPanelName()` |
| `controller.RenderEngine.PanelName` | `controller.GetPanelName()` |
| `controller.RenderEngine != null` | `controller.GetPanelName() != null` |

## Important Notes

- Smart Office scripts use **synchronous** MI calls (`MIAccess.Execute`). H5 uses **async/Promise-based** calls. Functions containing MI calls should become `async`.
- `BackgroundWorker` patterns are replaced with Promise chains.
- WPF Grid positioning (`Grid.SetColumn/Row`) maps to `Position.Left/Top` on H5 UI elements.
- `.NET` specific APIs (System.IO, System.Net, System.Xml) have no direct H5 equivalent — flag these for manual review with TODO comments.
- The `ConfirmDialogHelper` TypeScript helper class should be included in the output for question dialogs that need await support.
- `ApplicationMessageService.Current` for inter-script messaging has no H5 equivalent — flag with TODO and suggest `SessionCache` or custom DOM events as alternative.

## Database / SQL → MDBREADMI API Migration

When a source script (Smart Office or H5 Classic UI) contains direct database connections or SQL queries, these MUST be replaced with calls to the **MDBREADMI** API in H5 Cloud. Direct database access is not available in H5 scripts.

### Patterns to Detect

| Pattern | Source Type |
|---|---|
| `SqlConnection` / `SqlCommand` / `SqlDataReader` | Smart Office (.NET) |
| `System.Data.SqlClient` | Smart Office (.NET) |
| `OdbcConnection` / `OleDbConnection` | Smart Office (.NET) |
| `SELECT ... FROM ...` (SQL query strings) | Both |
| `DataAccess` / `DatabaseConnection` | Both |
| `ExecuteReader` / `ExecuteScalar` / `ExecuteNonQuery` | Smart Office (.NET) |
| `$.ajax` with SQL endpoint | H5 Classic UI |

### H5 V1 Replacement (MIService.Current + MDBREADMI)

```javascript
var request = new MIRequest();
request.program = "MDBREADMI";
request.transaction = "SelMITMAS00";  // Table-specific transaction
request.record = {
    ITNO: itemNumber   // Filter fields
};
request.outputFields = ["ITNO", "ITDS", "STAT"];  // Fields to return
request.maxReturnedRecords = 100;

MIService.Current.executeRequest(request).then(function(response) {
    for (var i = 0; i < response.items.length; i++) {
        var item = response.items[i];
        var itno = item.ITNO;
        var desc = item.ITDS;
        // Process each record
    }
}).catch(function(error) {
    console.log("MDBREADMI error: " + error.errorMessage);
});
```

### H5 V2 Replacement (MIService + MDBREADMI)

```javascript
var request = new MIRequest();
request.program = "MDBREADMI";
request.transaction = "SelMITMAS00";
request.record = {
    ITNO: itemNumber
};
request.outputFields = ["ITNO", "ITDS", "STAT"];
request.maxReturnedRecords = 100;

MIService.executeRequestV2(request).then(function(response) {
    if (response.errorMessage) {
        console.log("MDBREADMI error: " + response.errorMessage);
        return;
    }
    for (var i = 0; i < response.items.length; i++) {
        var item = response.items[i];
        var itno = item.ITNO;
        var desc = item.ITDS;
        // Process each record
    }
}).catch(function(error) {
    console.log("Network error: " + (error.errorMessage || error.message));
});
```

### Dual-Compatible (V1+V2)

```javascript
var miService;
if (ScriptUtil.version >= 2.0) {
    miService = MIService;
} else {
    miService = MIService.Current;
}

var request = new MIRequest();
request.program = "MDBREADMI";
request.transaction = "SelMITMAS00";
request.record = { ITNO: itemNumber };
request.outputFields = ["ITNO", "ITDS", "STAT"];

miService.executeRequest(request).then(function(response) {
    // V2: check response.errorMessage first
    if (ScriptUtil.version >= 2.0 && response.errorMessage) {
        console.log("Error: " + response.errorMessage);
        return;
    }
    for (var i = 0; i < response.items.length; i++) {
        // Process records
    }
}).catch(function(error) {
    console.log("Error: " + (error.errorMessage || error));
});
```

### MDBREADMI Transaction Naming Convention

MDBREADMI transactions follow the pattern: `Sel{TABLE}{INDEX}`

| Table | Transaction Example | Description |
|---|---|---|
| MITMAS | `SelMITMAS00` | Item Master (index 00) |
| OCUSMA | `SelOCUSMA00` | Customer Master |
| CIDMAS | `SelCIDMAS00` | Supplier Master |
| OOLINE | `SelOOLINE00` | Order Lines |
| MPDHED | `SelMPDHED00` | Purchase Order Header |
| MITBAL | `SelMITBAL00` | Item Balance |

### SQL → MDBREADMI Mapping Rules

1. **Identify the table** from the SQL `FROM` clause → use as the table part of the transaction name
2. **Identify the WHERE clause fields** → map to `request.record` (filter inputs)
3. **Identify the SELECT fields** → map to `request.outputFields`
4. **Identify any LIMIT/TOP** → map to `request.maxReturnedRecords`
5. **Identify the index** used (if known from SQL hints or WHERE clause structure) → append to transaction name (00, 01, 02...)
6. If the exact index is unknown, default to `00` (primary key index) and add a TODO comment

### Example Migration

**Before (Smart Office with SQL):**
```javascript
import System.Data.SqlClient;

var conn = new SqlConnection(connectionString);
conn.Open();
var cmd = new SqlCommand("SELECT ITNO, ITDS, STAT FROM MITMAS WHERE ITNO = '" + itemNo + "'", conn);
var reader = cmd.ExecuteReader();
while (reader.Read()) {
    var itemDesc = reader["ITDS"];
}
conn.Close();
```

**After (H5 V1):**
```javascript
var request = new MIRequest();
request.program = "MDBREADMI";
request.transaction = "SelMITMAS00";
request.record = { ITNO: itemNo };
request.outputFields = ["ITNO", "ITDS", "STAT"];
request.maxReturnedRecords = 1;

MIService.Current.executeRequest(request).then(function(response) {
    if (response.items && response.items.length > 0) {
        var itemDesc = response.items[0].ITDS;
    }
}).catch(function(error) {
    console.log("MDBREADMI error: " + error.errorMessage);
});
```

### Important Notes

- **MDBREADMI is read-only** — it only supports SELECT operations. INSERT/UPDATE/DELETE have no MDBREADMI equivalent; these must use the appropriate MI program (e.g., `MMS200MI/AddItmBasic` for inserts)
- **The MDBREADMI API must be created by the developer first** — it does not exist by default. The migrated code should be wrapped in a TODO comment instructing the developer to create the API in M3
- **Index selection matters** — using the wrong index will return no results or incorrect data. Check M3 documentation for available indexes on each table
- **Performance** — set `maxReturnedRecords` appropriately. Unbounded queries can time out
- Direct database access patterns should NEVER be left as-is in H5 scripts — they will not work in cloud environments
- **Reference documentation:** [Infor M3 MDBREADMI Documentation](https://docs.infor.com/m3udi/16.x/en-us/m3beud/default.html?helpcontent=sns1599471544581.html)

### TODO Comment Format

When migrating SQL/database patterns, the generated code should call the MDBREADMI API using the **same pattern as any other MI call** (MIRequest + MIService), but wrap the entire section with a TODO comment instructing the developer to create the MDBREADMI API first:

```javascript
// TODO [H5 Migration]: Database/SQL access detected in source script.
// The developer must create an MDBREADMI API in M3 for table {TABLE_NAME} before this code will work.
// Steps: 1) Open MRS001 in M3  2) Create MDBREADMI transaction for the required table/index
// 3) Define input/output fields matching the SQL query below
// Reference: https://docs.infor.com/m3udi/16.x/en-us/m3beud/default.html?helpcontent=sns1599471544581.html
//
// Original SQL: SELECT {FIELDS} FROM {TABLE} WHERE {CONDITIONS}
var request = new MIRequest();
request.program = "MDBREADMI";
request.transaction = "Sel{TABLE}{INDEX}";  // Developer must verify correct transaction name
request.record = { /* WHERE clause fields */ };
request.outputFields = [ /* SELECT fields */ ];
request.maxReturnedRecords = 100;

MIService.Current.executeRequest(request).then(function(response) {
    // Process results
}).catch(function(error) {
    console.log("MDBREADMI error: " + error.errorMessage);
});
```

The key point is: **the call structure is identical to any other MI call** — the only difference is the developer must first create the MDBREADMI transaction in M3 via MRS001.

## V2-Only ListControl APIs

These APIs are only available in H5 V2 and are useful for grid/list operations:

```typescript
// Render a custom data grid in a container element
ListControl.RenderDataGrid($listElement, columns, data, options);

// Get column index by column name
var index = ListControl.GetColumnIndexByName("ITNO");

// Get all data for a specific column
var columnData = ListControl.GetListColumnData("ITNO", controller);

// Get the datagrid instance from the controller
var datagrid = ListControl.ListView.GetDatagrid(controller);

// Hide specific columns
controller.GetGrid().hideColumns(["COL1", "COL2"]);
```

### Smart Office → V2 ListControl Mappings

| Smart Office Pattern | H5 V2 Equivalent |
|---|---|
| `Configuration.Current.ListConfig(id, columns, data)` + `$el.inforDataGrid(options)` | `ListControl.RenderDataGrid($list, columns, data, options)` |
| Manual column index lookup via loop | `ListControl.GetColumnIndexByName(colName)` |
| `list.Columns[i].Name` / manual iteration | `ListControl.GetListColumnData(colName, controller)` |
| Getting grid reference from RenderEngine | `ListControl.ListView.GetDatagrid(controller)` |

### Dual-Compatible RenderDataGrid

```javascript
if (ScriptUtil.version >= 2.0) {
    var options = { forceFitColumns: true, autoHeight: true };
    ListControl.RenderDataGrid($list, columns, data, options);
} else {
    var options = Configuration.Current.ListConfig('id', columns, data);
    options["forceFitColumns"] = true;
    options["autoHeight"] = true;
    var grid = $("#myGrid").inforDataGrid(options);
    grid.render();
}
```

## ExportToExcel / ExportToGoogleSheets (V2-Only)

H5 V2 provides built-in export methods on `IInstanceController`:

```typescript
// Export current list data to Excel
controller.ExportToExcel();

// Export current list data to Google Sheets
controller.ExportToGoogleSheets();
```

### Smart Office → V2 Export Mappings

| Smart Office Pattern | H5 V2 Equivalent |
|---|---|
| Clipboard copy + paste to Excel | `controller.ExportToExcel()` |
| Manual CSV/data construction for export | `controller.ExportToExcel()` |
| No Google Sheets equivalent existed | `controller.ExportToGoogleSheets()` |

**Note:** These are V2-only. In dual-compatible mode, wrap with version check:
```javascript
if (ScriptUtil.version >= 2.0) {
    controller.ExportToExcel();
} else {
    // V1: no built-in export, leave as-is or flag
    // TODO [H5 Migration]: Manual export logic — V2 provides controller.ExportToExcel()
}
```

## RadioGroupElement (V2-Only)

Smart Office uses WPF `RadioButton` controls. H5 V2 provides `RadioGroupElement`:

**Before (Smart Office):**
```javascript
var radioBtn = new RadioButton();
radioBtn.Content = "Option A";
radioBtn.GroupName = "myGroup";
radioBtn.IsChecked = true;
Grid.SetRow(radioBtn, 5);
Grid.SetColumn(radioBtn, 10);
content.Children.Add(radioBtn);
```

**After (H5 V2):**
```typescript
var radioGroup = new RadioGroupElement();
radioGroup.Name = "myGroup";
radioGroup.Items = [
    { label: "Option A", value: "A", selected: true },
    { label: "Option B", value: "B" }
];
radioGroup.Position = new PositionElement();
radioGroup.Position.Top = 5;
radioGroup.Position.Left = 10;
this.content.AddElement(radioGroup);
```

| Smart Office | H5 V2 UI |
|---|---|
| Multiple `new RadioButton()` with same `GroupName` | Single `new RadioGroupElement()` with `Items` array |
| `radioBtn.Content = "text"` | `{ label: "text", value: "val" }` in Items |
| `radioBtn.IsChecked = true` | `{ selected: true }` in item |
| `radioBtn.GroupName = "name"` | `radioGroup.Name = "name"` |

## ContentElement.RemoveScriptComponents() (V2-Only)

**WARNING: This is V2-only. Do NOT use in V1 scripts.**

When a script adds UI elements (buttons, labels, etc.) to a panel, it should clean them up when the script unloads or the panel changes. H5 V2 provides:

```typescript
// V2 ONLY - Remove all UI elements that were added by this script
controller.GetContentElement().RemoveScriptComponents();
```

### V1 Alternative (no RemoveScriptComponents available)

In V1, there is no `RemoveScriptComponents()`. Instead:
- Hide elements: `document.getElementById("elemName").style.display = "none"`
- Clear values: `this.labelElement.Value = ""`
- Leave elements in DOM and overwrite their content on next use

```javascript
// V1: Manual UI reset (no RemoveScriptComponents)
OIS002_E_ValidateAddress.prototype.resetUI = function() {
    this.statusLabelElem.Value = "";
    this.confirmMsgLabelElem.Value = "";
    var btnEl = document.getElementById("confirmBtn");
    if (btnEl) { btnEl.style.display = "none"; }
    var comboEl = document.getElementById("addressSelectionCombo");
    if (comboEl) { comboEl.style.display = "none"; }
};
```

### Usage Pattern

```typescript
// In the RequestCompleted or panel-change handler:
MyScript.prototype.onRequestCompleted = function(args) {
    if (this.controller.GetPanelName() !== "MMS001/E") {
        // Leaving the panel — clean up
        this.controller.GetContentElement().RemoveScriptComponents();
        this.detachEvents();
    }
};
```

### Smart Office → V2 Cleanup Mapping

| Smart Office | H5 V2 UI |
|---|---|
| `content.Children.Remove(element)` (per element) | `controller.GetContentElement().RemoveScriptComponents()` (all at once) |
| `element.add_Unloaded(handler)` | Not needed — use event unsubscription + `RemoveScriptComponents()` |
| Manual removal of each added control | Single call removes all script-added components |

## Error Handling Patterns (Consistent Across Both Paths)

### H5 V2 MIService Error Handling Tree

Every MI call in V2 can produce three outcomes. Handle all three:

```javascript
MIService.executeRequest(miRequest).then(function(response) {
    // CASE 1: Functional MI error (record not found, validation failure)
    if (response.hasError()) {
        // The MI transaction returned an error
        _this.log.Error("MI Error: " + response.errorMessage);
        // response.errorCode — error code (e.g., "WMM0104")
        // response.errorField — field that caused the error
        return;
    }
    // CASE 2: Success — process results
    var items = response.items;  // array of records
    if (items && items.length > 0) {
        var firstItem = items[0];
        // access fields: firstItem["ITNO"], firstItem.ITDS, etc.
    }
}).catch(function(error) {
    // CASE 3: Technical/network error only (timeout, connection lost)
    _this.log.Error("Network error: " + (error.errorMessage || error.message || error));
});
```

**CRITICAL:** In V2, use `response.hasError()` (method call) to check for functional MI errors — NOT `response.errorMessage` (property check). The `hasError()` method is the correct V2 API.

### Smart Office → V2 Error Handling Migration

| Smart Office Pattern | H5 V2 Pattern |
|---|---|
| `response.HasError` (in .then or callback) | `response.errorMessage` (truthy check in .then) |
| `response.Error.ToString()` | `response.errorMessage` |
| `response.Error` | `response.error` |
| Catch = technical + functional errors | `.then()` = functional errors; `.catch()` = technical only |
| `MIResponse` type | `IMIResponse` type |

### Classic UI (V1) → V2 Error Restructuring

In Classic UI (V1), `MIService.Current` routes ALL errors (functional + technical) to `.catch()`. In V2, functional errors come to `.then()` via `response.hasError()`:

**Before (V1):**
```javascript
MIService.Current.executeRequest(request).then(function(response) {
    processItems(response.items);
}).catch(function(error) {
    showError(error.errorMessage);  // catches BOTH functional and technical
});
```

**After (V2):**
```javascript
MIService.executeRequest(request).then(function(response) {
    if (response.hasError()) {
        showError(response.errorMessage);  // functional error (was in .catch in V1)
    } else {
        processItems(response.items);      // success
    }
}).catch(function(error) {
    showError(error.errorMessage || error.message);  // technical/network only
});
```

### V2 Event Cleanup Pattern

In V2, use `widget.destroy()` in the Requested event handler to clean up jQuery widgets and event listeners:

```javascript
const unsubscribeRequesting = this.controller.Requesting.On((args) => {
    // Handle pre-request logic (e.g., cancel ENTER if widget is open)
});

const unsubscribeRequested = this.controller.Requested.On((args) => {
    unsubscribeRequesting();
    unsubscribeRequested();
    widget.destroy();  // Clean up jQuery widgets
});
```

### V2 Controller Properties

| Property/Method | Description |
|---|---|
| `controller.focusField` | Name of the currently focused field on the panel |
| `controller.GetPanelName()` | Returns current panel (e.g., "OIS101/B") |
| `controller.GetElement(fieldId)` | Returns jQuery element for the field |
| `controller.GetContent()` | Returns jQuery wrapper of panel content |
| `controller.GetContentElement()` | Returns IContentElement interface |
| `controller.GetGrid()` | Returns IActiveGrid interface |
| `controller.PressKey(keyName)` | Programmatically press a key (e.g., "ENTER") |

### V2 Grid Data Structure (Cloud Pattern)

In Cloud M3 H5, the grid has an `h5Data` property with column metadata:

```javascript
const grid = this.controller.GetGrid();
if (grid.h5Data && grid.h5Data.columns) {
    const column = grid.h5Data.columns.find((col) => {
        return col.hasPosField && col.positionField.name === fieldId;
    });
    if (column) {
        const $contentPanel = this.controller.GetContentElement().ContentPanel;
        const $column = $contentPanel.find(`[data-column-id='${column.fullName}']`);
        const $input = $column.find('input');
    }
}
```
