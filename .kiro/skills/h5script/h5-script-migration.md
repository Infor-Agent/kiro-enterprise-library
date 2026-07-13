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

## 1. Introduction

This skill guides the conversion of M3 H5 Classic UI (on-premise) scripts to V2 UI (Modern UI / Cloud) scripts. 

### Scope

- JavaScript (`.js`) and TypeScript (`.ts`) H5 script files
- Scripts containing Classic UI patterns identified by the activation keywords
- Transformation follows a fixed dependency order across 6 migration phases


## 2. Preprocessing Pipeline

Before applying any transformation rules, protect string literals and comments from accidental modification by replacing them with unique placeholders. This prevents regex-based replacement rules from matching content inside strings or comments.

### 2.1 Protection Order

Placeholders are applied in this specific sequence:

1. **Quoted strings** (first, because strings can contain `//` or `/* */` patterns)
2. **Block comments** (`/* ... */`)
3. **Line comments** (`// ...`)

### 2.2 String Literal Protection

Replace all single-quoted and double-quoted string literals with sequential placeholders.

**Regex pattern:** `(["'])(?:\\.|[^\\])*?\1`

This matches:
- Double-quoted strings: `"hello world"`, `"contains // slashes"`
- Single-quoted strings: `'field name'`, `'mforms://something'`
- Escaped quotes within: `"say \"hi\""`, `'it\'s fine'`

**Placeholder format:** `ScriptPlaceHolderQuotedString` + zero-padded 10-digit counter

**Examples:**
| Original | Placeholder |
|---|---|
| `"hello"` | `ScriptPlaceHolderQuotedString0000000000` |
| `'/execute/MMS200MI/Get'` | `ScriptPlaceHolderQuotedString0000000001` |
| `"she said \"hi\""` | `ScriptPlaceHolderQuotedString0000000002` |

Each matched string is stored in a lookup table (keyed by placeholder) for later restoration.

### 2.3 Block Comment Protection

After string placeholders are in place, replace all block comments (`/* ... */`) with sequential placeholders.

**Regex pattern:** `/\*(?:[\w\W]*?)\*/`

This matches multi-line comments spanning any number of lines.

**Placeholder format:** `ScriptPlaceHolderCommentBlock` + zero-padded 10-digit counter

**Example:**
```javascript
/* This function calls ScriptUtil.ApiRequest
   to fetch data from /execute/MMS200MI/Get */
```
→ `ScriptPlaceHolderCommentBlock0000000000`

### 2.4 Line Comment Protection

After block comments are replaced, replace all line comments (`// ...`) with sequential placeholders.

**Regex pattern:** `//.*[^\r\n]`

This matches everything from `//` to the end of the line (excluding the newline character itself).

**Placeholder format:** `ScriptPlaceHolderCommentLine` + zero-padded 10-digit counter

**Example:**
```javascript
// TODO: replace ScriptUtil.ApiRequest with MIService
```
→ `ScriptPlaceHolderCommentLine0000000000`

Line comments are matched iteratively (one at a time from the last match position forward) to avoid issues with overlapping matches.

### 2.5 Restoration Order (Post-processing)

After all transformation rules have been applied, restore placeholders back to original content in **reverse protection order**:

1. **Comment lines** first — restore `ScriptPlaceHolderCommentLine*` → original `// ...` content
2. **Comment blocks** second — restore `ScriptPlaceHolderCommentBlock*` → original `/* ... */` content
3. **Quoted strings** last — restore `ScriptPlaceHolderQuotedString*` → original quoted content

This reverse order ensures that if a restored comment happens to contain text resembling another placeholder format, it does not interfere with subsequent restorations.

### 2.6 Why Placeholders Prevent False Pattern Matches

Without preprocessing, transformation rules would corrupt string and comment content. Consider:

**Problem scenario (without protection):**
```javascript
var url = "/execute/MMS200MI/Get?ITNO=ABC123";
// Old call: ScriptUtil.ApiRequest(url, success, error)
ScriptUtil.ApiRequest(url, this.onSuccess.bind(this), this.onError.bind(this));
```

If the API URL parsing rule ran without protection, it would incorrectly try to parse the `/execute/` pattern inside the string literal and the comment, potentially corrupting both.

**With placeholder protection:**
```javascript
var url = ScriptPlaceHolderQuotedString0000000000;
ScriptPlaceHolderCommentLine0000000000
ScriptUtil.ApiRequest(url, this.onSuccess.bind(this), this.onError.bind(this));
```

Now the transformation rules only match the actual `ScriptUtil.ApiRequest` call statement. The string literal and comment are opaque placeholders that no regex rule will match. After transformation completes, restoration brings back the original string and comment content unchanged.

## 3. Script Structure Reference

This section documents the V2 UI script architecture, interfaces, and classes available for H5 scripts. Use this as the target reference when migrating Classic UI scripts.

### 3.1 V2 UI Script Class Pattern

All H5 V2 UI scripts follow a class-based pattern with three structural components:

1. **Constructor** — accepts an `IScriptArgs` parameter and assigns instance properties
2. **Static `Init` method** — entry point called by the H5 framework; instantiates the class and invokes the main method
3. **Prototype methods** — contain the script business logic

```javascript
var MyScript = /** @class */ (function () {
    /**
     * Constructor - receives IScriptArgs from the H5 framework
     */
    function MyScript(scriptArgs) {
        this.controller = scriptArgs.controller;  // IInstanceController
        this.content = scriptArgs.controller.GetContentElement();  // IContentElement
        this.log = scriptArgs.log;               // IScriptLog
        this.elem = scriptArgs.elem;             // Element or null
        this.args = scriptArgs.args;             // Script argument string or null
        this.listView = ListControl.ListView;
        this.miService = MIService;
    }

    /**
     * Static Init method - H5 framework entry point
     */
    MyScript.Init = function (args) {
        new MyScript(args).run();
    };

    /**
     * Prototype method - main script logic
     */
    MyScript.prototype.run = function () {
        // Script logic here
    };

    return MyScript;
}());
```

#### `IScriptArgs` Interface

The `IScriptArgs` object is passed to the constructor by the H5 framework:

| Property | Type | Description |
|----------|------|-------------|
| `controller` | `IInstanceController` | Instance controller for the current program |
| `elem` | `any` | The element, or `null` if the script is not connected to an element |
| `args` | `string` | Script argument string, or `null` if no arguments were specified |
| `log` | `IScriptLog` | Log object for logging to the browser console |
| `debug` | `IScriptDebugConsole` | **Deprecated** — use `log` instead |

#### `IScriptLog` Interface

| Method | Description |
|--------|-------------|
| `Error(message: string, ex?: Error)` | Log an error message |
| `Warning(message: string, ex?: Error)` | Log a warning message |
| `Info(message: string, ex?: Error)` | Log an informational message |
| `Debug(message: string, ex?: Error)` | Log a debug message |
| `Trace(message: string, ex?: Error)` | Log a trace message |
| `SetDefault()` | Reset log level to default |
| `SetDebug()` | Set log level to debug |
| `SetTrace()` | Set log level to trace |

### 3.2 `IInstanceController` Interface

The `IInstanceController` provides access to the program context, content elements, grid, field values, and request lifecycle events.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `ParentWindow` | `JQuery` | The parent window jQuery element |
| `RenderEngine` | `IRenderEngine` | The render engine for the instance |
| `Response` | `IResponseElement` | The current response element |

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetElement` | `GetElement(name: string): any` | Get a UI element by name from the panel |
| `GetGrid` | `GetGrid(): IActiveGrid` | Get the active grid/list control |
| `GetValue` | `GetValue(name: string): any` | Get a field value by name |
| `SetValue` | `SetValue(name: string, val: any): void` | Set a field value by name |
| `GetProgramName` | `GetProgramName(): string` | Get the current M3 program name |
| `GetPanelName` | `GetPanelName(): string` | Get the current panel name |
| `GetView` | `GetView(): string` | Get the current view |
| `PressKey` | `PressKey(key: string): void` | Simulate a key press |
| `ListOption` | `ListOption(option: string): void` | Execute a list option |
| `ShowMessage` | `ShowMessage(message: string): void` | Show a message to the user |
| `ShowMessageInStatusBar` | `ShowMessageInStatusBar(message: string): void` | Show a message in the status bar |
| `GetContent` | `GetContent(): JQuery` | Get the content jQuery element |
| `GetContentElement` | `GetContentElement(): IContentElement` | Get the content element interface |
| `GetInstanceId` | `GetInstanceId(): string` | Get the instance identifier |
| `GetMode` | `GetMode(): string` | Get the current mode |
| `GetSortingOrder` | `GetSortingOrder(): string` | Get the current sorting order |
| `PageDown` | `PageDown(): void` | Simulate page down |

#### Request Lifecycle Events

These are `IInstanceEvent` objects supporting `On(handler)`, `Off(handler)`, and `Clear()`:

| Event | Description |
|-------|-------------|
| `Requesting` | Fires before a request is sent. Handler receives `CancelRequestEventArgs` (with `cancel` boolean to prevent the request, plus `controller`, `commandType`, `commandValue`) |
| `Requested` | Fires after a request is sent but before the response is processed. Handler receives `RequestEventArgs` (with `controller`, `commandType`, `commandValue`) |
| `RequestCompleted` | Fires after the response has been fully processed. Handler receives `RequestEventArgs` |

Usage example:
```javascript
// Subscribe to event
var handler = this.controller.Requesting.On(function(args) {
    args.cancel = true; // Cancel the request
});

// Unsubscribe from event
this.controller.Requesting.Off(handler);
```

### 3.3 UI Element Classes

All UI element classes share a common base (`IBaseElement`) and add type-specific properties.

#### Shared Base Properties (`IBaseElement`)

| Property | Type | Description |
|----------|------|-------------|
| `Name` | `string` | Element name identifier |
| `Value` | `string` | Current value of the element |
| `IsEnabled` | `boolean` | Whether the element is enabled |
| `IsVisible` | `boolean` | Whether the element is visible |
| `IsReadDisabled` | `boolean` | Whether read is disabled |
| `TabIndex` | `number` | Tab order index |
| `FieldHelp` | `string` | Field help identifier |
| `ReferenceFile` | `string` | Reference file name |
| `ReferenceField` | `string` | Reference field name |
| `Position` | `IPositionElement` | Position with `Top`, `Left`, `Width`, `Height` |
| `Constraint` | `IConstraintElement` | Constraints: `IsNumeric`, `IsUpper`, `MaxLength`, `MaxDecimals`, `MaxRow`, `MaxColumn` |

#### Element Classes

| Class | Additional Properties | Notes |
|-------|----------------------|-------|
| `ButtonElement` | `Command`, `CommandValue` | Button with command binding |
| `TextBoxElement` | `IsReverse`, `IsHighIntensity`, `IsRightJustified`, `IsBrowsable`, `IsFixedFont`, `IsPosition`, `DateFormat` | Standard text input |
| `CheckBoxElement` | `IsChecked` | Boolean checkbox |
| `ComboBoxElement` | `Items` (array), `Command`, `CommandValue`, `IsEditable` | Dropdown selection |
| `LabelElement` | `Id`, `ToolTip`, `IsFixed`, `IsAdditionalInfo`, `IsEmphasized`, `IsColon`, `CssClass` | Read-only label |
| `ListElement` | `Columns` (array), `Rows` (array), `SelectedRows` (array) | List/grid element |
| `TextAreaElement` | `IsReverse`, `IsHighIntensity`, `IsRightJustified`, `IsBrowsable`, `IsFixedFont`, `IsPosition` | Multi-line text |
| `DatePickerElement` | *(same as TextBoxElement)* | **V2-specific** — replaces `TextBoxElement` when the field uses date formatting. The `DatePickerElement` handles date formatting natively, removing the need for `.inforDateField()` calls. |

### 3.4 `MIRequest` Structure

The `MIRequest` class is used with `MIService.executeRequestV2()` for MI API calls.

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `program` | `string` | MI program name (e.g., `"MMS200MI"`) |
| `transaction` | `string` | MI transaction name (e.g., `"Get"`, `"Lst"`) |
| `record` | `object` | Input field key-value pairs (e.g., `{ CONO: "100", ITNO: "ITEM001" }`) |
| `outputFields` | `string[]` | Requested output column names (e.g., `["ITDS", "STAT"]`) |

#### Optional Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `company` | `string` | *(current)* | Override company number |
| `division` | `string` | *(current)* | Override division |
| `excludeEmptyValues` | `boolean` | `false` | Exclude empty values from response |
| `maxReturnedRecords` | `number` | *(unlimited)* | Maximum number of records to return |
| `includeMetadata` | `boolean` | `false` | Include field metadata in response |
| `typedOutput` | `boolean` | `false` | Return typed output values |
| `timeout` | `number` | *(default)* | Request timeout in milliseconds |

#### Usage Example

```javascript
var miRequest = new MIRequest();
miRequest.program = "CRS610MI";
miRequest.transaction = "GetBasicData";
miRequest.record = { CUNO: "CUSTOMER01" };
miRequest.outputFields = ["CUNM", "CUA1", "PONO"];
miRequest.maxReturnedRecords = 1;

MIService.executeRequestV2(miRequest).then(function(response) {
    if (response.errorMessage) {
        // Handle functional MI error
    } else {
        // Process response.items
        for (var i = 0; i < response.items.length; i++) {
            var item = response.items[i];
            // Access fields: item.CUNM, item.CUA1, etc.
        }
    }
}).catch(function(error) {
    // Handle technical/network error only
});
```

#### `IMIResponse` Interface

The response from MI calls contains:

| Property | Type | Description |
|----------|------|-------------|
| `program` | `string` | Program name |
| `transaction` | `string` | Transaction name |
| `item` | `any` | Single item result |
| `items` | `any[]` | Array of result items |
| `metadata` | `any` | Field metadata (if requested) |
| `tag` | `any` | Custom tag passed through |
| `errorField` | `string` | Field that caused the error |
| `errorType` | `MIErrorType` | Error type: `Http`, `MI`, or `Parse` |
| `error` | `any` | Raw error object |
| `errorMessage` | `string` | Error message text |
| `errorCode` | `string` | Error code |
| `hasError()` | `boolean` | Returns true if an error exists |

### 3.5 `ListControl` and `ListView`

#### `ListControl` Static Class

Provides grid/list utility methods:

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetColumnIndexByName` | `GetColumnIndexByName(colName: string, controller?: IInstanceController): number` | Get column index by field name |
| `Columns` | `Columns(controller?: IInstanceController): any[]` | Get all column definitions |
| `GetPositionFieldValue` | `GetPositionFieldValue(colName: string, controller?: IInstanceController): string` | Get position field value |
| `Headers` | `Headers(controller?: IInstanceController): string[]` | Get column headers |
| `GetListColumnData` | `GetListColumnData(columnName: string, controller: IInstanceController)` | Get all data for a column |
| `RenderDataGrid` | `RenderDataGrid(list: IList, columns: any[], rows: any[], options: object)` | Render a data grid |

#### `ListControl.ListView` Static Property

An instance of `ListView` providing row-level grid data access:

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetValueByColumnName` | `GetValueByColumnName(colName: string, controller?: IInstanceController): string[]` | Get values by column name for selected row(s) |
| `GetValueByColumnIndex` | `GetValueByColumnIndex(colIndex: number, controller?: IInstanceController): string[]` | Get values by column index for selected row(s) |
| `SelectedItem` | `SelectedItem(controller?: IInstanceController): number[]` | Get selected item indices |
| `SetValueByColumnName` | `SetValueByColumnName(colName, value, controller?: IInstanceController): void` | Set value by column name |
| `SetValueByColumnIndex` | `SetValueByColumnIndex(colIndex, value, controller?: IInstanceController): void` | Set value by column index |
| `GetDatagrid` | `GetDatagrid(controller: IInstanceController): IActiveGrid` | Get the underlying active grid |

#### `IActiveGrid` Interface

Returned by `controller.GetGrid()`:

| Method/Property | Description |
|-----------------|-------------|
| `getColumns(): any[]` | Get column definitions |
| `setColumns(columns: any[]): void` | Set column definitions |
| `getData(): any` | Get grid data array |
| `setData(data: any[]): void` | Set grid data |
| `getSelectedGridRows(): any[]` | Get selected rows (V2 — returns objects with `.idx` property) |
| `setSelectedRows(rows: any[]): void` | Set selected rows |
| `onSelectedRowsChanged` | Event for selection changes |

### 3.6 Cache Classes

Two cache classes are available for persisting state between script invocations:

#### `InstanceCache` — Scoped to a Single Program Instance

Data persists as long as the program instance is open. Keyed by both `controller` and `key`.

| Method | Signature | Description |
|--------|-----------|-------------|
| `Add` | `InstanceCache.Add(controller: IInstanceController, key: string, val: any): void` | Store a value |
| `Get` | `InstanceCache.Get<T>(controller: IInstanceController, key: string): T` | Retrieve a value |
| `Remove` | `InstanceCache.Remove(controller: IInstanceController, key: string): boolean` | Remove a value |
| `ContainsKey` | `InstanceCache.ContainsKey(controller: IInstanceController, key: string): boolean` | Check if key exists |

Usage example:
```javascript
// Store a value scoped to this program instance
InstanceCache.Add(this.controller, "customerName", "Acme Corp");

// Retrieve it later (same instance)
var name = InstanceCache.Get(this.controller, "customerName");

// Check existence
if (InstanceCache.ContainsKey(this.controller, "customerName")) {
    InstanceCache.Remove(this.controller, "customerName");
}
```

#### `SessionCache` — Scoped to the User Session

Data persists for the entire user session across program instances. Keyed by `key` only.

| Method | Signature | Description |
|--------|-----------|-------------|
| `Add` | `SessionCache.Add(key: string, val: any): void` | Store a value |
| `Get` | `SessionCache.Get<T>(key: string): T` | Retrieve a value |
| `Remove` | `SessionCache.Remove(key: string): boolean` | Remove a value |
| `ContainsKey` | `SessionCache.ContainsKey(key: string): boolean` | Check if key exists |

Usage example:
```javascript
// Store a value for the session (accessible from any program instance)
SessionCache.Add("userPreference", "dark");

// Retrieve it from another script/instance
var pref = SessionCache.Get("userPreference");
```

### 3.7 `ConfirmDialog.ShowMessageDialog` Interface

Used to display message dialogs in V2 UI (replaces `ShowDialog` and jQuery-based message dialogs).

#### `IConfirmDialogOptions`

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `dialogType` | `string` | No | Dialog type: `"Alert"`, `"Question"`, or `"Error"` |
| `header` | `string` | Yes | Dialog header/title text |
| `message` | `string` | Yes | Dialog body message text |
| `withCancelButton` | `boolean` | No | Whether to show a Cancel button |
| `canHide` | `boolean` | No | Whether the dialog can be hidden |
| `closed` | `(choice: { ok: boolean, cancel: boolean }) => void` | No | Callback when dialog closes; `choice.ok` is `true` if OK was clicked |
| `isCancelDefault` | `boolean` | No | Whether Cancel is the default button |
| `buttons` | `IConfirmDialogButton[]` | No | Custom button definitions |

#### `IConfirmDialogButton`

| Property | Type | Description |
|----------|------|-------------|
| `text` | `string` | Button label |
| `click` | `Function` | Click handler |
| `isDefault` | `boolean` | Whether this is the default button |

#### Usage Examples

```javascript
// Simple alert
ConfirmDialog.ShowMessageDialog({
    dialogType: "Alert",
    header: "Warning",
    message: "Record not found."
});

// Question with callback
ConfirmDialog.ShowMessageDialog({
    dialogType: "Question",
    header: "Confirm Action",
    message: "Are you sure you want to delete this record?",
    withCancelButton: true,
    canHide: false,
    closed: function(choice) {
        if (choice.ok) {
            // User clicked OK
        }
    }
});

// Error dialog
ConfirmDialog.ShowMessageDialog({
    dialogType: "Error",
    header: "Error",
    message: "An unexpected error occurred."
});
```

### 3.8 Additional Utility Classes

#### `ScriptUtil` Static Methods

| Method | Description |
|--------|-------------|
| `ScriptUtil.GetUserContext()` | Returns the user context object (company, division, etc.) |
| `ScriptUtil.GetFieldValue(fieldName, controller?)` | Get a field value |
| `ScriptUtil.SetFieldValue(fieldName, value, controller?)` | Set a field value |
| `ScriptUtil.FindChild(parent, elementName)` | Find a child jQuery element |
| `ScriptUtil.Launch(task)` | Launch a task |
| `ScriptUtil.AddEventHandler(element, eventType, callback, paramData?)` | Add event handler |
| `ScriptUtil.RemoveEventHandler(element, eventType)` | Remove event handler |
| `ScriptUtil.OpenMenu(name, controller?)` | Open a menu |
| `ScriptUtil.LoadScript(url, callback)` | Load an external script |
| `ScriptUtil.DoEnterpriseSearch(query, controller?)` | Perform enterprise search |

#### `H5ControlUtil.H5Dialog`

| Method | Description |
|--------|-------------|
| `CreateDialogElement(content: HTMLElement, dialogOptions: any)` | Create a V2 dialog |
| `DestroyDialog(dialogContent: JQuery, isRemoveInstance?: boolean)` | Destroy a dialog (Classic — use `model.close(true)` in V2) |
| `HasDialogInstance(dialogId: string): boolean` | Check if dialog instance exists |
| `AddDialogInstance(dialogId: string): boolean` | Register a dialog instance |
| `RemoveDialogInstance(dialogId: string): any` | Remove dialog instance registration |

#### `IonApiService`

| Method | Description |
|--------|-------------|
| `IonApiService.execute(request: IonApiRequest): Promise<IonApiResponse>` | Execute an ION API request |
| `IonApiService.getBaseUrl(): string` | Get the ION API base URL |
| `IonApiService.setToken(token: string): void` | Set the authentication token |

Note: In V2 UI, use `IonApiService` directly (not `IonApiService.Current`).

## 4. API Transaction Migration

This section covers all transformations related to MI (Messaging Interface) API calls. The migration converts Classic UI `ScriptUtil.ApiRequest` calls (which use URL-encoded request strings) into V2 UI `MIService.executeRequestV2` calls (which use structured `MIRequest` objects).

### 4.1 URL Parsing Rules

Classic UI scripts encode MI API calls as URL strings in the format:

```
"/execute/{Program}/{Transaction}[;paramKey=paramValue]*[?Field1=Value1&Field2=Value2...]"
```

The migration engine parses this URL into a structured `MIRequest` object. This sub-section documents the parsing rules.

#### URL Format Breakdown

| Segment | Description | MIRequest Mapping |
|---------|-------------|-------------------|
| `/execute/` | Fixed prefix (identifies MI API URLs) | *(detection marker)* |
| `{Program}` | MI program name (uppercase alphanumeric, e.g., `MMS200MI`) | `miRequest.program` |
| `{Transaction}` | MI transaction name (e.g., `Get`, `Lst`, `DelFieldValue`) | `miRequest.transaction` |
| `;paramKey=paramValue` | Semicolon-delimited request parameters (zero or more) | Various optional fields |
| `?Field1=Value1&Field2=Value2` | Query-string input fields (ampersand-delimited) | `miRequest.record` |

#### Detection Regex

The URL parsing is triggered when a variable assignment matches this pattern:

```regex
^(const|var|let){0,1}[\s]*[\w]+[\s]*=[\s]*"/execute/(?<MIProgram>[A-Z0-9]+)/(?<MITransaction>[A-z0-9]+)(?<apiRequestParameters>;[\w=,;]*){0,1}(?<inputFields>\?.*){0,1}
```

This matches statements like:
- `var url = "/execute/MMS200MI/Get;returncols=ITDS,FUDS;metadata=false?ITNO=" + ITNO;`
- `url = "/execute/CUSEXTMI/DelFieldValue?FILE=M3OUTPUT&PK01=" + APProp + "&PK02=" + DIVI;`

#### Request Parameter Mapping (`;`-delimited segment)

Parameters appear between the transaction name and the `?` query string, separated by semicolons:

| URL Parameter | MIRequest Property | Type | Example |
|---|---|---|---|
| `;returncols=COL1,COL2,...` | `miRequest.outputFields` | `string[]` | `;returncols=ITDS,FUDS` → `["ITDS", "FUDS"]` |
| `;metadata=true\|false` | `miRequest.includeMetadata` | `boolean` | `;metadata=false` → `false` |
| `;maxrecs=N` | `miRequest.maxReturnedRecords` | `number` | `;maxrecs=1` → `1` |
| `;excludeEmpty=true\|false` | `miRequest.excludeEmpty` | `boolean` | `;excludeEmpty=true` → `true` |

Multiple parameters are chained: `;returncols=ITDS,FUDS;metadata=false;maxrecs=1;excludeEmpty=true`

Parsing logic:
1. Strip the leading `;`
2. Split on `;` to get individual `key=value` pairs
3. Map each key to the corresponding `MIRequest` property
4. For `returncols`, split the value on `,` to produce a string array

#### Input Field Mapping (`?`-delimited query parameters)

Query parameters after `?` represent the input record fields sent to the MI transaction:

| URL Query Format | MIRequest Mapping |
|---|---|
| `?FIELD1=value1&FIELD2=value2` | `miRequest.record = { FIELD1: value1, FIELD2: value2 }` |

Parsing logic:
1. Strip the leading `?`
2. Split on `&` to get individual `FieldName=Value` pairs
3. For each pair, split on `=` to separate field name from value
4. If the value portion contains a string concatenation (e.g., `ITNO=" + itemNumber + "`), extract the script variable name as the value
5. If the value is a static string literal (no concatenation), wrap it in quotes
6. Build the `record` object preserving the original field order

#### Complete Before/After Example

**Before (Classic UI URL assignment):**

```javascript
var url = "/execute/MMS200MI/Get;returncols=ITDS,FUDS;metadata=false;maxrecs=1?ITNO=" + ITNO;
```

**After (V2 UI MIRequest object):**

```javascript
var miRequest = new MIRequest();
miRequest.program = "MMS200MI";
miRequest.transaction = "Get";
miRequest.record = {
    ITNO: ITNO
}
miRequest.outputFields = ["ITDS", "FUDS"];
miRequest.includeMetadata = false;
miRequest.maxReturnedRecords = 1;
```

#### Multi-Field Input Example

**Before:**

```javascript
url = "/execute/CUSEXTMI/DelFieldValue?FILE=M3OUTPUT&PK01=" + APProp + "&PK02=" + DIVI;
```

**After:**

```javascript
var miRequest = new MIRequest();
miRequest.program = "CUSEXTMI";
miRequest.transaction = "DelFieldValue";
miRequest.record = {
    FILE: "M3OUTPUT",
    PK01: APProp,
    PK02: DIVI
}
```

#### Parameters-Only Example (no input fields)

**Before:**

```javascript
var url = "/execute/MMS200MI/Lst;returncols=ITNO,ITDS,NEPR,DIA1,DIA2,DIA3,DIP1,SAPR,ORQT";
```

**After:**

```javascript
var miRequest = new MIRequest();
miRequest.program = "MMS200MI";
miRequest.transaction = "Lst";
miRequest.outputFields = ["ITNO", "ITDS", "NEPR", "DIA1", "DIA2", "DIA3", "DIP1", "SAPR", "ORQT"];
```

#### TypeScript Variant

For TypeScript target files, use `const` instead of `var`:

```typescript
const miRequest = new MIRequest();
miRequest.program = "MMS200MI";
miRequest.transaction = "Get";
miRequest.record = {
    ITNO: ITNO
}
miRequest.outputFields = ["ITDS", "FUDS"];
miRequest.includeMetadata = false;
miRequest.maxReturnedRecords = 1;
```

#### Edge Cases

- **URL as variable reference only** (e.g., `ScriptUtil.ApiRequest(url, ...)` where `url` was assigned elsewhere): The URL parsing is applied at the assignment site. If the URL variable was assigned inline as a string literal containing `/execute/`, it is parsed and replaced. If the URL is constructed dynamically without an inline `/execute/` literal, the statement cannot be parsed — see section 4.2 for the fallback behavior.
- **Concatenated URL strings**: When the URL is built via string concatenation (e.g., `"/execute/..." + variable + "..."`), the preprocessing placeholder mechanism replaces quoted segments with `ScriptPlaceHolderQuotedString*` tokens. The parser reassembles the original URL by restoring placeholders before applying the regex, then extracts variable names from the concatenation boundaries.
- **Trailing characters**: The parser strips any non-word characters (quotes, semicolons) from the end of the input fields string before splitting.

<!-- Sub-sections 4.2, 4.3, 4.4 follow below -->

### 4.2 Execute Statement Migration Rules

This sub-section documents the transformation of `ScriptUtil.ApiRequest(url, successCallback, errorCallback)` calls into V2 UI promise-based `MIService.executeRequestV2(miRequest).then(...).catch(...)` statements.

#### Overview

The Classic UI uses `ScriptUtil.ApiRequest` with three arguments:
1. **url** — the MI API URL (variable name or inline string)
2. **successCallback** — function or reference called on success
3. **errorCallback** — function or reference called on error

The V2 UI replaces this with:
```javascript
MIService.executeRequestV2(miRequest)
    .then((result) => { /* success logic */ })
    .catch((error) => { /* error logic */ })
```

The `miRequest` object is constructed from the URL parsing (see section 4.1). The `.then()` and `.catch()` handlers are built from the success and error callback arguments.

#### Detection Pattern

The migration engine finds all `ScriptUtil.ApiRequest(` calls using:

```regex
(?<prefixSpaces>[\s]*)ScriptUtil\.[Aa][Pp][Ii]Request[\s]*\(
```

For each match, the three arguments (url, responseStatement, errorStatement) are extracted by counting balanced parentheses and splitting on top-level commas.

#### Success Callback Transformation (`.then()`)

The success callback argument determines the `.then()` handler content. Two cases:

**Case A: Simple function reference** (matches pattern `^([\w]+\.)*[\w]+(\.bind\([\w]+\)){0,1}$`)

This covers named functions and bound method references:
- `MMS200MIGetResult`
- `this.MMS200MIGetResult.bind(this)`
- `CheckprocessItems`

The `.then()` wraps a direct invocation:
```javascript
.then((result) => { this.MMS200MIGetResult.bind(this)(result); })
```

**Case B: Inline function with body**

When the callback is an inline `function(param) { ... }` or arrow function, the body is extracted and placed inside the `.then()`:
```javascript
.then((result) => {
    // extracted function body here
})
```

The result variable name is preserved from the original function parameter (e.g., if the original uses `function(response)`, the `.then()` uses `(response) =>`).

#### Error Callback Transformation (`.catch()`)

The error callback argument determines the `.catch()` handler content. Three cases:

**Case A: Simple function reference** (matches pattern `^[\w]+$`)

For a plain function name like `MMS200MIGetFailed`, the `.catch()` invokes it with the error message:
```javascript
.catch((error) => { MMS200MIGetFailed('Error', error.error?.terminationReason ?? error.message ?? error); })
```

This maps the V2 error structure into the two-parameter pattern the original error handler expects.

**Case B: Inline function with 2 parameters (error, header)**

When the error callback has two parameters (e.g., `function(error3, header3) { ... }`), the V2 `.catch()` only uses the first parameter. The second parameter (header) is declared as a local variable set to `'Error'`:

```javascript
.catch((error3) => {
    var header3 = 'Error';
    // original error handler body unchanged
})
```

This preserves the header variable name used in the original body without breaking references to it.

**Case C: Inline function with 1 parameter (error only)**

When the error callback has a single parameter, the body is placed directly into `.catch()` without the header variable insertion:
```javascript
.catch((error) => {
    // original error handler body
})
```

#### `.bind(this)` Preservation

When callback references use `.bind(this)`, the bound reference is preserved as-is in the generated `.then()` or `.catch()` handler. The migration does not strip or restructure `.bind(this)` — it becomes a direct call within the arrow function:

```javascript
// Original bound reference preserved:
.then((result) => { this.MMS200MIGetResult.bind(this)(result); })
```

This ensures the callback executes with the correct `this` context, matching the Classic UI behavior.

#### Dynamic URL Fallback (TODO Emission)

When the URL argument is a variable name that cannot be resolved to an inline `/execute/` URL literal (i.e., the URL is constructed dynamically elsewhere or passed as a parameter), the migration engine cannot construct a structured `MIRequest`. In this case:

1. The statement is logged as a warning (script name, function name, statement content)
2. A `// TODO` comment is emitted suggesting manual replacement with `MIService.executeV2`:

```javascript
// TODO [H5 Migration]: Replace ScriptUtil.ApiRequest with MIService.executeV2(program, transaction, inputParameters) - URL could not be parsed automatically
```

`MIService.executeV2(program, transaction, inputParameters)` is the suggested alternative because it accepts program/transaction/record as separate parameters without requiring URL parsing.

#### Final Statement Assembly

The complete migrated statement is assembled in this order:
1. The `miRequest` construction (from URL parsing in section 4.1, if applicable)
2. `MIService.executeRequestV2(miRequest)` call
3. `.then(...)` handler
4. `.catch(...)` handler

All parts use arrow function syntax and maintain the original indentation level.

#### Concrete Examples

##### Example 1: TypeScript — Bound Method References

**Before (Classic UI):**

```typescript
var url = "/execute/MMS200MI/Get;returncols=ITDS,FUDS;metadata=false;maxrecs=1?ITNO=" + ITNO;
ScriptUtil.ApiRequest(url, this.MMS200MIGetResult.bind(this), this.MMS200MIGetFailed.bind(this));
```

**After (V2 UI):**

```typescript
const miRequest = new MIRequest();
miRequest.program = "MMS200MI";
miRequest.transaction = "Get";
miRequest.record = {
    ITNO: ITNO
}
miRequest.outputFields = ["ITDS", "FUDS"];
miRequest.includeMetadata = false;
miRequest.maxReturnedRecords = 1;

MIService.executeRequestV2(miRequest)
    .then((result) => { this.MMS200MIGetResult.bind(this)(result); })
    .catch((error) => { this.MMS200MIGetFailed.bind(this)('Error', error.error?.terminationReason ?? error.message ?? error); })
```

##### Example 2: JavaScript — Simple Function References

**Before (Classic UI):**

```javascript
var url = "/execute/CUSEXTMI/DelFieldValue?FILE=M3OUTPUT&PK01=" + APProp + "&PK02=" + DIVI;
ScriptUtil.ApiRequest(url, MMS200MIGetResult, MMS200MIGetFailed);
```

**After (V2 UI):**

```javascript
var miRequest = new MIRequest();
miRequest.program = "CUSEXTMI";
miRequest.transaction = "DelFieldValue";
miRequest.record = {
    FILE: "M3OUTPUT",
    PK01: APProp,
    PK02: DIVI
}

MIService.executeRequestV2(miRequest)
    .then((result) => { MMS200MIGetResult(result); })
    .catch((error) => { MMS200MIGetFailed('Error', error.error?.terminationReason ?? error.message ?? error); })
```

##### Example 3: Inline Functions with Error Header Parameter

**Before (Classic UI):**

```javascript
ScriptUtil.ApiRequest("/execute/MMS200MI/Get?ITNO=" + ITNO, function(response) {
    var items = response.MIRecord;
    processItems(items);
}, function(error3, header3) {
    ConfirmDialog.ShowMessageDialog({ dialogType: "Error", header: header3, message: error3.responseJSON.Message });
});
```

**After (V2 UI):**

```javascript
var miRequest = new MIRequest();
miRequest.program = "MMS200MI";
miRequest.transaction = "Get";
miRequest.record = {
    ITNO: ITNO
}

MIService.executeRequestV2(miRequest)
    .then((response) => {
        var items = response.MIRecord;
        processItems(items);
    })
    .catch((error3) => {
        var header3 = 'Error';
        ConfirmDialog.ShowMessageDialog({ dialogType: "Error", header: header3, message: error3.responseJSON.Message });
    })
```

> **Note:** The `response.MIRecord` and `error3.responseJSON.Message` references in this example would subsequently be transformed by the response reading rules (section 4.4) into `response.items` and `error3.error?.terminationReason ?? error3.message ?? error3` respectively. The execute statement migration applies first; other API replacements follow.

##### Example 4: Inline Function with Single Error Parameter

**Before (Classic UI):**

```javascript
ScriptUtil.ApiRequest(url, function(result) {
    CheckprocessItems(result, asset, subno);
}, CheckFlowonFail);
```

**After (V2 UI):**

```javascript
MIService.executeRequestV2(miRequest)
    .then((result) => {
        CheckprocessItems(result, asset, subno);
    })
    .catch((error) => { CheckFlowonFail('Error', error.error?.terminationReason ?? error.message ?? error); })
```

> **Note:** In this example, the `url` variable was previously transformed into the `miRequest` object by the URL parsing rules (section 4.1). The execute statement migration references the already-constructed `miRequest`.

##### Example 5: Dynamic URL — Cannot Parse (TODO Fallback)

**Before (Classic UI):**

```javascript
var apiUrl = buildDynamicUrl(program, transaction);
ScriptUtil.ApiRequest(apiUrl, this.onSuccess.bind(this), this.onError.bind(this));
```

**After (V2 UI):**

```javascript
var apiUrl = buildDynamicUrl(program, transaction);
// TODO [H5 Migration]: Replace ScriptUtil.ApiRequest with MIService.executeV2(program, transaction, inputParameters) - URL could not be parsed automatically
ScriptUtil.ApiRequest(apiUrl, this.onSuccess.bind(this), this.onError.bind(this));
```

When the URL is a variable that does not contain an inline `/execute/` literal, the original statement is preserved unchanged and a TODO comment is inserted above it. The developer must manually construct the appropriate `MIService.executeV2(program, transaction, inputParameters)` call.

#### Transformation Order Note

The execute statement migration (this section) runs **after** the URL parsing migration (section 4.1). This means:
1. Section 4.1 first converts URL string assignments into `miRequest` object construction
2. Section 4.2 then converts the `ScriptUtil.ApiRequest(url, ...)` call into `MIService.executeRequestV2(miRequest).then(...).catch(...)`

Both transformations work together to produce the complete V2 UI MI call pattern.

### 4.3 MI Request Execution Error Handling Restructuring

When a Classic UI script already uses `MIService.Current.executeRequest(...)` or `MIService.Current.execute(...)` with `.then()` and `.catch()` handlers, the error handling model must be restructured for V2 UI. In Classic UI, functional MI errors (e.g., "Record does not exist") trigger the `.catch()` handler. In V2 UI, functional MI errors are returned in the `.then()` handler via `response.errorMessage` — the `.catch()` handler only fires for technical/network errors.

The migration restructures the `.then()` body to check `response.errorMessage` first, inserting the catch logic (with variable substitution) in the if-branch and the original then logic in the else-branch. The `.catch()` handler is preserved unchanged for technical errors. Additionally, `.Current` is removed from `MIService.Current`.

#### Detection Regex

The following regex identifies MI execution statements with a chained `.then()`:

```regex
\.execute(Request){0,1}[\s]*\([\s]*(?<miRequestParameters>[\w\s,]+)[\s]*\)[\s]*\.then[\s]*\([\s]*(function){0,1}[\s]*\([\s]*(?<responseVarName>[\w]+)[\s]*\)
```

This matches:
- `MIService.Current.executeRequest(myRequest).then(function (response) {`
- `MIService.Current.execute(program, transaction, record, outputFields).then(function (response) {`
- `this.miService.Current.executeRequest(myRequest).then((response) => {` (arrow function syntax)

#### Catch Variable Name Extraction

The error variable name is extracted from the `.catch()` handler using:

```regex
\.catch[\s]*\((function[\s]*){0,1}[\s]*\((?<errorVarName>[\w]+)
```

This captures the parameter name (e.g., `error`, `response`, `err`) regardless of whether the handler uses `function` or arrow syntax.

#### Transformation Algorithm

1. **Locate** the `.execute`/`.executeRequest` call with a `.then()` handler
2. **Extract** the `.then()` body content (`thenContent`) and the response variable name (e.g., `response`)
3. **Extract** the `.catch()` body content (`catchContent`) and the error variable name (e.g., `error`)
4. **Substitute variables**: In a copy of the catch content destined for the then block, replace all occurrences of `<errorVarName>.` with `<responseVarName>.` (e.g., `error.` → `response.`)
5. **Build new `.then()` body**: Wrap in an `if/else` structure:
   ```
   if(<responseVarName>.errorMessage) {
       <modified catch content with variable substitution>
   } else {
       <original then content>
   }
   ```
6. **Preserve** the original `.catch()` handler unchanged (it handles technical/network errors)
7. **Remove `.Current`**: Replace `MIService.Current` with `MIService` (applied by the general API replacement rules in section 4.4)

#### Variable Substitution Detail

The substitution replaces the error variable prefix with the response variable prefix **only where followed by a dot** (property access). This ensures:
- `error.errorMessage` → `response.errorMessage`
- `error.message` → `response.message`
- `response` (standalone, no dot) → unchanged

The substitution uses simple string replacement: `"<errorVarName>."` → `"<responseVarName>."`. This works because the error variable is always accessed via dot notation in the catch body.

#### Complete Before/After Example 1: `executeRequest` with `function` syntax

**Before (Classic UI):**

```javascript
MIService.Current.executeRequest(myRequest).then(function (response) {
    //Read results here
    for (var _i = 0, _a = response.items; _i < _a.length; _i++) {
        var item = _a[_i];
        _this.ShowDialog("retrieved ITDS for selected ITNO by request: " + item.ITDS);
    }
}).catch(function (error) {
    //Handle errors here
    _this.log.Error(error.errorMessage);
});
```

**After (V2 UI):**

```javascript
MIService.executeRequest(myRequest).then(function (response) {
    if(response.errorMessage) {
        //Handle errors here
        _this.log.Error(response.errorMessage);
    } else {
        //Read results here
        for (var _i = 0, _a = response.items; _i < _a.length; _i++) {
            var item = _a[_i];
            _this.ShowDialog("retrieved ITDS for selected ITNO by request: " + item.ITDS);
        }
    }
}).catch(function (error) {
    //Handle errors here
    _this.log.Error(error.errorMessage);
});
```

**Changes applied:**
1. `.Current` removed from `MIService.Current`
2. `.then()` body restructured with `if(response.errorMessage)` check
3. Catch body content copied into the if-branch with `error.` → `response.` substitution
4. Original then content moved into the else-branch
5. Original `.catch()` handler preserved unchanged

#### Complete Before/After Example 2: `execute` with inline parameters

**Before (Classic UI):**

```javascript
this.miService.Current.execute(program, transaction, record, outputFields).then(function (response) {
    //Read results here
    for (var _i = 0, _a = response.items; _i < _a.length; _i++) {
        var item = _a[_i];
        _this.gDebug.Info("GetBatchHeadWarehouse: get warehouse: ".concat(item.WHLO));
        _this.FindKeyAccountItemStops(iCUNO, item.WHLO);
    }
}).catch(function (response) {
    //Handle errors here
    _this.gDebug.Error(response.errorMessage);
});
```

**After (V2 UI):**

```javascript
this.miService.execute(program, transaction, record, outputFields).then(function (response) {
    if(response.errorMessage) {
        //Handle errors here
        _this.gDebug.Error(response.errorMessage);
    } else {
        //Read results here
        for (var _i = 0, _a = response.items; _i < _a.length; _i++) {
            var item = _a[_i];
            _this.gDebug.Info("GetBatchHeadWarehouse: get warehouse: ".concat(item.WHLO));
            _this.FindKeyAccountItemStops(iCUNO, item.WHLO);
        }
    }
}).catch(function (response) {
    //Handle errors here
    _this.gDebug.Error(response.errorMessage);
});
```

**Note:** When the catch handler already uses `response` as the parameter name (same as the then handler), the variable substitution `response.` → `response.` is a no-op — the content is copied unchanged.

#### Complete Before/After Example 3: `executeRequest` with conditional logic in catch

**Before (Classic UI):**

```javascript
MIService.Current.executeRequest(myRequest)
    .then(function (response) {
    _this.postToModalBox(config.title, _this.processResponse(config.PK02, response));
})
    .catch(function (response) {
    _this.log.Error(response.errorMessage);
    if ("Record does not exist" == response.errorMessage) {
        _this.postToModalBox(config.title, "No records found");
    }
});
```

**After (V2 UI):**

```javascript
MIService.executeRequest(myRequest)
    .then(function (response) {
    if(response.errorMessage) {
        _this.log.Error(response.errorMessage);
        if ("Record does not exist" == response.errorMessage) {
            _this.postToModalBox(config.title, "No records found");
        }
    } else {
        _this.postToModalBox(config.title, _this.processResponse(config.PK02, response));
    }
})
    .catch(function (response) {
    _this.log.Error(response.errorMessage);
    if ("Record does not exist" == response.errorMessage) {
        _this.postToModalBox(config.title, "No records found");
    }
});
```

#### Arrow Function Syntax

The same transformation applies when arrow functions are used:

**Before:**

```javascript
MIService.Current.executeRequest(myRequest).then((response) => {
    processResults(response.items);
}).catch((err) => {
    console.log(err.errorMessage);
});
```

**After:**

```javascript
MIService.executeRequest(myRequest).then((response) => {
    if(response.errorMessage) {
        console.log(response.errorMessage);
    } else {
        processResults(response.items);
    }
}).catch((err) => {
    console.log(err.errorMessage);
});
```

Here `err.` is replaced with `response.` in the if-branch content.

#### Key Rules Summary

| Rule | Description |
|------|-------------|
| Applies to | `MIService.Current.executeRequest(...)` and `MIService.Current.execute(...)` |
| Trigger | Presence of `.then()` chained after `.execute`/`.executeRequest` |
| Variable substitution | `<errorVar>.` → `<responseVar>.` in copied catch content only |
| New `.then()` structure | `if(response.errorMessage) { catch-logic } else { original-then-logic }` |
| `.catch()` handler | Preserved unchanged (handles technical/network errors) |
| `.Current` removal | Applied separately (see section 4.4) |
| Syntax support | Both `function` keyword and arrow `=>` syntax |
| Multiple occurrences | Each `.execute`/`.executeRequest` call in the function is transformed independently |

#### Why the `.catch()` Is Preserved

In V2 UI, the `.catch()` handler serves a different purpose than in Classic UI:
- **Classic UI**: `.catch()` handles both functional MI errors AND technical errors
- **V2 UI**: `.catch()` handles ONLY technical/network errors (e.g., timeout, server unreachable)

Functional MI errors (like "Record does not exist") are now delivered to the `.then()` handler with `response.errorMessage` populated. By preserving the `.catch()` unchanged, the script retains a safety net for genuine connection failures while the new `if(response.errorMessage)` check handles business-level errors within the success path.

### 4.4 Response Reading and General API Replacement Rules

This sub-section covers the regex-driven replacements defined in `ReplaceList_APIStatements.csv` and the structural `MIRecord[i].NameValue` loop migration handled by `MigrateAPITransactionRequestReadResponseStatements`. These rules run **after** the URL parsing (4.1), execute statement migration (4.2), and MI request execution restructuring (4.3).

---

#### 4.4.1 `result.MIRecord` → `result.items`

The Classic UI MI response object uses `MIRecord` to hold the array of returned records. In V2 UI, this is simply `items`.

**CSV Rule** (from `ReplaceList_APIStatements.csv`):
- Search: `(?<Store_resultVarName>[\w]+)\.MIRecord`
- Replace: `{Replace_resultVarName}.items`

The `Store_resultVarName` named capture **stores** the variable name so subsequent rules can reference it (e.g., the `.Message` rule below). This means if the code uses `response4.MIRecord`, the stored variable name is `response4`.

**Before (Classic UI):**
```javascript
var miRecords = result.MIRecord;
for (var i = 0; i < miRecords.length; i++) {
    // process miRecords[i]
}
```

**After (V2 UI):**
```javascript
var miRecords = result.items;
for (var i = 0; i < miRecords.length; i++) {
    // process miRecords[i]
}
```

---

#### 4.4.2 `result.Message` → `result.errorMessage`

The Classic UI uses `.Message` on the response object to access error messages. V2 UI uses `.errorMessage`.

**CSV Rule**:
- Search: `{Replace_resultVarName}\.Message`
- Replace: `{Replace_resultVarName}.errorMessage`

This rule uses the variable name stored by the `MIRecord` rule (4.4.1). If `response4.MIRecord` was matched earlier, then `response4.Message` becomes `response4.errorMessage`.

**Before (Classic UI):**
```javascript
if (response4.Message) {
    console.log("Error: " + response4.Message);
}
```

**After (V2 UI):**
```javascript
if (response4.errorMessage) {
    console.log("Error: " + response4.errorMessage);
}
```

---

#### 4.4.3 `error.responseJSON.Message` → `error.error.terminationReason ?? error.message ?? error`

The Classic UI accesses error details via `.responseJSON.Message`. V2 UI uses a nullish coalescing chain to extract the most specific error information available.

**CSV Rule**:
- Search: `(?<requestAPIErrorVarName>[\w]+)\.responseJSON\.Message`
- Replace: `{Replace_requestAPIErrorVarName}.error.terminationReason ?? {Replace_requestAPIErrorVarName}.message ?? {Replace_requestAPIErrorVarName}`

The named capture `requestAPIErrorVarName` extracts the error variable name and reuses it in the replacement.

**Before (Classic UI):**
```javascript
.catch(function(error) {
    ConfirmDialog.ShowMessageDialog({ dialogType: "Error", header: header3, message: error.responseJSON.Message });
});
```

**After (V2 UI):**
```javascript
.catch(function(error) {
    ConfirmDialog.ShowMessageDialog({ dialogType: "Error", header: header3, message: error.error.terminationReason ?? error.message ?? error });
});
```

**Another example:**
```javascript
// Before
var msg = err.responseJSON.Message;

// After
var msg = err.error.terminationReason ?? err.message ?? err;
```

---

#### 4.4.4 `MIService.Current` → `MIService` (Remove `.Current` Singleton)

Classic UI accessed MI service through a singleton pattern (`MIService.Current`). In V2 UI, `MIService` is used directly without `.Current`.

**CSV Rule**:
- Search: `MIService\.Current`
- Replace: `MIService`

**Before (Classic UI):**
```javascript
MIService.Current.executeRequest(miRequest).then(function(response) {
    // handle response
}).catch(function(error) {
    // handle error
});
```

**After (V2 UI):**
```javascript
MIService.executeRequest(miRequest).then(function(response) {
    // handle response
}).catch(function(error) {
    // handle error
});
```

---

#### 4.4.5 `IonApiService.Current` → `IonApiService` (Remove `.Current` Singleton)

Same singleton removal pattern applies to `IonApiService`.

**CSV Rule**:
- Search: `IonApiService\.Current`
- Replace: `IonApiService`

**Before (Classic UI):**
```javascript
IonApiService.Current.execute(requestConfig).then(function(response) {
    // handle ION API response
});
```

**After (V2 UI):**
```javascript
IonApiService.execute(requestConfig).then(function(response) {
    // handle ION API response
});
```

---

#### 4.4.6 MIRecord NameValue Loop Pattern → Compatibility Shim

The Classic UI API returns each record as an object with a `NameValue` array containing `{Name, Value}` pairs. V2 UI returns records as plain key-value objects (e.g., `{ITNO: "ABC123", ITDS: "Item Desc"}`). To preserve downstream code that iterates `nameValue[j].Name` / `nameValue[j].Value`, a compatibility shim is inserted.

This transformation is handled by the `MigrateAPITransactionRequestReadResponseStatements` function in the PowerShell tool.

**Detection regex**: `(var|const|let)?\s+<nameValueVar>\s*=\s*<miRecordsVar>[<indexVar>]\.NameValue\s*;`

**Before (Classic UI):**
```javascript
var miRecords = result.MIRecord;
for (var i = 0; i < miRecords.length; i++) {
    var nameValue = miRecords[i].NameValue;
    for (var j = 0; j < nameValue.length; j++) {
        if ("ITDS" == nameValue[j].Name) {
            itemDescription = nameValue[j].Value;
        }
        if ("ITNO" == nameValue[j].Name) {
            itemNumber = nameValue[j].Value;
        }
    }
}
```

**After (V2 UI):**
```javascript
var miRecords = result.items;
for (var i = 0; i < miRecords.length; i++) {
    var nameValue = [];
    for (const property in miRecords[i]) {
        nameValue.push({Name: property, Value: miRecords[i][property]});
    }
    for (var j = 0; j < nameValue.length; j++) {
        if ("ITDS" == nameValue[j].Name) {
            itemDescription = nameValue[j].Value;
        }
        if ("ITNO" == nameValue[j].Name) {
            itemNumber = nameValue[j].Value;
        }
    }
}
```

**How the shim works:**
1. The original `var nameValue = miRecords[i].NameValue;` is replaced with `var nameValue = [];`
2. A `for...in` loop iterates over the properties of the V2 response object (`miRecords[i]`, which is now `result.items[i]`)
3. Each property is pushed as a `{Name: property, Value: miRecords[i][property]}` entry
4. The downstream `nameValue[j].Name` and `nameValue[j].Value` access patterns continue to work unchanged

**Variable preservation:**
- The `var`/`const`/`let` keyword (or absence) from the original statement is preserved
- The `nameValue` variable name (whatever the original name is) is preserved
- The `miRecords[i]` array access expression (whatever the original variable and index are) is preserved

**Note:** The `result.MIRecord` → `result.items` replacement (4.4.1) runs first via the CSV rules, so by the time this shim is inserted, `miRecords` already references `result.items`.

---

#### Processing Order

The rules in `ReplaceList_APIStatements.csv` are processed **sequentially** in file order:

1. `result.MIRecord` → `result.items` (stores the result variable name)
2. `result.Message` → `result.errorMessage` (uses the stored variable name)
3. `error.responseJSON.Message` → nullish coalescing chain
4. `MIService.Current` → `MIService`
5. `IonApiService.Current` → `IonApiService`

The variable name stored by rule 1 is reused by rule 2. This means the `.Message` replacement is only applied to the same variable that had `.MIRecord` — preventing false replacements on unrelated `.Message` properties.

The `MigrateAPITransactionRequestReadResponseStatements` function (NameValue shim, 4.4.6) runs **before** the CSV-based `MigrateAPIStatements` function in the pipeline. The execution order within `MigrateAPITransactionStatements` is:
1. `MigrateAPITransactionRequestInitStatements` (URL parsing)
2. `MigrateAPITransactionRequestExecuteStatements` (ScriptUtil.ApiRequest replacement)
3. `MigrateAPITransactionRequestReadResponseStatements` (NameValue shim — 4.4.6)
4. `MigrateMIRequestExecutionStatements` (error handling restructure)
5. `MigrateAPIStatements` (CSV-based rules — 4.4.1 through 4.4.5)

## 5. Grid Operations Migration

This section covers all transformations related to SlickGrid-based Classic UI grid operations. The migration converts legacy grid accessor patterns (`.getDataItem()`, `.getSelectedRows()`, `.getData().getLength()`, etc.) into their V2 UI equivalents using direct array access and updated method names.

Grid rules rely on **variable tracking** — a mechanism that captures variable names from earlier statements and uses them in subsequent replacement patterns. Rules are applied sequentially within each function, and the variable tracking state persists across rules within the same function scope.

### 5.1 Variable Tracking Mechanism

Before understanding individual rules, it is essential to understand how variable tracking works:

1. **Store rules** — Some rules exist solely to capture a variable name into a named store (e.g., `gridVarName`, `selectedRowsVar`, `rowVarName`). These rules do NOT modify the source code; they only populate the tracking state.
2. **Replace rules** — Subsequent rules reference stored variables using `{Replace_<varName>}` tokens in both their search patterns and replacement strings. If a required variable has not been captured, the replacement is **skipped** (graceful degradation — the original code is left unchanged).
3. **Scope** — Variable tracking state is reset per function. Each function processed starts with a clean tracking state.
4. **Order dependency** — Rules must be applied in the exact order documented below. Earlier rules capture variables that later rules depend on.

#### Store-Only Rule Examples

The first three rules in the grid replacement pipeline are store-only:

```
Rule 1: (?<Store_gridVarName>[\w]+)\.getData().getLength()
Rule 2: (?<Store_gridVarName>[\w]+)\.getDataItem(
Rule 3: (?<Store_gridVarName>[\w]+)\s*=\s*<any>.GetGrid()
```

When the engine encounters `list.getData().getLength()`, it stores `gridVarName = "list"` but does not modify the statement (the actual `.getLength()` replacement happens in a later rule).

When it encounters `var grid = this.controller.GetGrid();`, it stores `gridVarName = "grid"`.

This allows all subsequent rules to reference `{Replace_gridVarName}` knowing which variable holds the grid reference.

### 5.2 Rule Reference (Applied in Order)

Below are all grid replacement rules in the exact order they are applied. Each rule documents its search pattern, whether it modifies code, and the replacement output.

---

#### Rule 1: Store grid variable from `.getData().getLength()` (Store-Only)

**Purpose:** Capture the grid variable name from a `.getData().getLength()` call.

**Search pattern:** `(?<Store_gridVarName>[\w]+)\.getData\(\)\.getLength\(\)`

**Replaces:** No (store-only)

**Example input:**
```javascript
var count = list.getData().getLength();
```

**Effect:** Stores `gridVarName = "list"`. No code change.

---

#### Rule 2: Store grid variable from `.getDataItem(` (Store-Only)

**Purpose:** Capture the grid variable name from any `.getDataItem()` call.

**Search pattern:** `(?<Store_gridVarName>[\w]+)\.getDataItem\(`

**Replaces:** No (store-only)

**Example input:**
```javascript
var row = this.grid.getDataItem(rowIndex);
```

**Effect:** Stores `gridVarName = "this.grid"`. No code change. (Note: the actual regex `[\w]+` matches single-word identifiers; for `this.grid`, the controller-prefixed rules handle the case.)

---

#### Rule 3: Store grid variable from `= <obj>.GetGrid()` assignment (Store-Only)

**Purpose:** Capture the grid variable name when a variable is assigned from a `.GetGrid()` call.

**Search pattern:** `(?<Store_gridVarName>[\w]+)[\s]*=[\s]*(?<controllerVarName>[\w.]+)\.GetGrid[\s]*\([\s]*\)[\s;]*$`

**Replaces:** No (store-only)

**Example input:**
```javascript
var list = this.controller.GetGrid();
```

**Effect:** Stores `gridVarName = "list"`. No code change.

---

#### Rule 4: `.getSelectedRows()` → `.getSelectedGridRows()`

**Purpose:** Replace the Classic UI row selection method with the V2 equivalent. Also captures the variable name receiving the result (`selectedRowsVar`) and re-captures `gridVarName`.

**Search pattern:** `(?<Store_selectedRowsVar>[\w]+)[\s]*=[\s]*(?<Store_gridVarName>[\w]+)\.getSelectedRows[\s]*\([\s]*\)`

**Replaces:** Yes

**Replacement:** `{Replace_selectedRowsVar} = {Replace_gridVarName}.getSelectedGridRows()`

**Before:**
```javascript
var selectedRows = this.grid.getSelectedRows();
```

**After:**
```javascript
var selectedRows = this.grid.getSelectedGridRows();
```

**Side effect:** Stores `selectedRowsVar = "selectedRows"` and `gridVarName = "this"` (or whichever word precedes `.getSelectedRows()`).

---

#### Rule 5: `selectedRows[index]` → `selectedRows[index].idx`

**Purpose:** In V2 UI, `.getSelectedGridRows()` returns objects (not numeric indices). To get the numeric row index, append `.idx`.

**Search pattern:** `{Replace_selectedRowsVar}[\s]*\[(?<selectedRowIndexVar>[\w]+)\]`

**Replaces:** Yes

**Replacement:** `{Replace_selectedRowsVar}[{Replace_selectedRowIndexVar}].idx`

**Prerequisite:** `selectedRowsVar` must have been captured by Rule 4.

**Before:**
```javascript
var rowIndex = selectedRows[0];
```

**After:**
```javascript
var rowIndex = selectedRows[0].idx;
```

**Note:** If `selectedRowsVar` was not captured (e.g., the selected rows variable was assigned in a different function), this rule is skipped and the code is left unchanged.

---

#### Rule 6: `gridVar.getDataItem(row)["column"]` → full column resolution

**Purpose:** Replace direct column access via `.getDataItem()` with V2 array access plus column ID resolution.

**Search pattern:** `{Replace_gridVarName}\.getDataItem[\s]*\([\s]*(?<rowIndexVarName>[\w]+)[\s]*\)[\s]*\[(?<columnIndexStatement>[\w"'` + "`" + `$+{}\s]+?)\]`

**Replaces:** Yes

**Replacement:** `{Replace_gridVarName}.getData()[{Replace_rowIndexVarName}][{Replace_gridVarName}.getColumns().find(c => c.colId == {Replace_columnIndexStatement}).fullName]`

**Prerequisite:** `gridVarName` must have been captured by Rules 1, 2, or 3.

**Before:**
```javascript
var ITNO = this.grid.getDataItem(row)["C1"];
```

**After:**
```javascript
var ITNO = this.grid.getData()[row][this.grid.getColumns().find(c => c.colId == "C1").fullName];
```

**Template literal example — Before:**
```javascript
var value = this.grid.getDataItem(row)[`C${OBORNO_index + 1}`];
```

**After:**
```javascript
var value = this.grid.getData()[row][this.grid.getColumns().find(c => c.colId == `C${OBORNO_index + 1}`).fullName];
```

---

#### Rule 7: `rowVar = <any>.getDataItem(rowIndex)` → array access

**Purpose:** Replace `.getDataItem()` row retrieval with direct array access on `.getData()`. Also captures the row variable name for subsequent column access rules.

**Search pattern:** `(?<Store_rowVarName>[\w]+)[\s]*=[\s]*[\w.]+\.getDataItem[\s]*\([\s]*(?<rowIndexVarName>[\w]+)[\s]*\)`

**Replaces:** Yes

**Replacement:** `{Replace_rowVarName} = {Replace_gridVarName}.getData()[{Replace_rowIndexVarName}]`

**Prerequisite:** `gridVarName` must have been captured.

**Before:**
```javascript
var row = this.grid.getDataItem(rowIndex);
```

**After:**
```javascript
var row = this.grid.getData()[rowIndex];
```

**Side effect:** Stores `rowVarName = "row"`.

---

#### Rule 8: Controller-prefixed `.GetGrid().getDataItem(row)["col"]` → full resolution

**Purpose:** Handle cases where the grid is accessed through a controller variable (e.g., `_this.controller.GetGrid().getDataItem(row)["C1"]`) without a stored grid variable.

**Search pattern:** `(?<controllerVarName>[\w_.]+)\.[Gg]etGrid\(\)\.getDataItem[\s]*\([\s]*(?<rowIndexVarName>[\w]+)[\s]*\)[\s]*\[(?<columnIndexStatement>[\w"'` + "`" + `$+{}\s]+?)\]`

**Replaces:** Yes

**Replacement:** `{Replace_controllerVarName}.GetGrid().getData()[{Replace_rowIndexVarName}][{Replace_controllerVarName}.GetGrid().getColumns().find(c => c.colId == {Replace_columnIndexStatement}).fullName]`

**Before:**
```javascript
var ITNO = _this.controller.GetGrid().getDataItem(selectedItem)["C1"];
```

**After:**
```javascript
var ITNO = _this.controller.GetGrid().getData()[selectedItem][_this.controller.GetGrid().getColumns().find(c => c.colId == "C1").fullName];
```

---

#### Rule 9: Generic `.GetGrid().getDataItem(row)` → `.GetGrid().getData()[row]`

**Purpose:** Catch remaining `.GetGrid().getDataItem()` calls not matched by Rule 8 (i.e., without a column access suffix).

**Search pattern:** `\.[Gg]etGrid\(\)\.getDataItem[\s]*\([\s]*(?<rowIndexVarName>[\w]+)[\s]*\)`

**Replaces:** Yes

**Replacement:** `.GetGrid().getData()[{Replace_rowIndexVarName}]`

**Before:**
```javascript
var rowData = _this.controller.GetGrid().getDataItem(selectedItem);
```

**After:**
```javascript
var rowData = _this.controller.GetGrid().getData()[selectedItem];
```

---

#### Rule 10: `.getData().getLength()` → `.getData().length`

**Purpose:** Replace the SlickGrid data-length method with the standard JavaScript array `.length` property.

**Search pattern:** `\.getData\(\)\.getLength\(\)`

**Replaces:** Yes

**Replacement:** `.getData().length`

**Before:**
```javascript
var count = list.getData().getLength();
```

**After:**
```javascript
var count = list.getData().length;
```

**Note:** This rule applies globally (not grid-variable-specific) to catch all remaining `.getLength()` patterns after the store-only Rule 1 has already captured the grid variable.

---

#### Rule 11: `.getColumnIndex(...)` → `.getColumns().find(...).index`

**Purpose:** Replace the SlickGrid column index lookup (which takes a column ID obtained from a `.find()`) with the V2 approach that uses `.fullName` and returns `.index` directly.

**Search pattern:** `{Replace_gridVarName}.getColumnIndex[\s]*\((?<columnStatement>.*\))\.id[\s]*\)`

**Replaces:** Yes

**Replacement:** `{Replace_columnStatement}.index`

**Prerequisite:** `gridVarName` must have been captured.

**Before:**
```javascript
var colIdx = this.grid.getColumnIndex(this.grid.getColumns().find(c => c.colFld == 'MMITDS').id);
```

**After:**
```javascript
var colIdx = this.grid.getColumns().find(c => c.fullName == 'MMITDS').index;
```

**Explanation:** In Classic UI, getting a column index required: (1) find the column by `colFld`, (2) get its `.id`, (3) pass that to `.getColumnIndex()`. In V2 UI, the column object directly has an `.index` property, and `colFld` is replaced with `fullName`.

---

#### Rule 12: `rowVar["columnIndex"]` → full column resolution via row variable

**Purpose:** When a row variable (captured from Rule 7) is used with bracket notation for column access, resolve via `.getColumns().find()`.

**Search pattern:** `{Replace_rowVarName}[\s]*\[(?<columnIndexVarName>[\w'` + "`" + `$+{}\s\[\]]+?)\]`

**Replaces:** Yes

**Replacement:** `{Replace_rowVarName}[{Replace_gridVarName}.getColumns().find(c => c.colId == {Replace_columnIndexVarName}).fullName]`

**Prerequisites:** Both `rowVarName` and `gridVarName` must have been captured.

**Before:**
```javascript
var ITNO = row["C1"];
```

**After:**
```javascript
var ITNO = row[this.grid.getColumns().find(c => c.colId == "C1").fullName];
```

---

#### Rule 13: `.colFld` → `.fullName`

**Purpose:** Replace the Classic UI column field accessor with the V2 equivalent.

**Search pattern:** `\.colFld`

**Replaces:** Yes

**Replacement:** `.fullName`

**Before:**
```javascript
var fieldName = this.grid.getColumns().find(c => c.colFld == 'MMITDS');
```

**After:**
```javascript
var fieldName = this.grid.getColumns().find(c => c.fullName == 'MMITDS');
```

---

#### Rule 14: `$.extend(grid.getData().getItem(i), newData)` → `grid.setData(grid.getData())`

**Purpose:** Replace the jQuery `$.extend` pattern for in-place grid row updates with the V2 `setData` approach. Also captures the row index variable and the data variable name for Rule 15.

**Search pattern:** `\$\.extend\({Replace_gridVarName}\.getData\(\)\.getItem\((?<Store_listRowIndexVarName>[\w]+)\),[\s]*(?<Store_newColumnDataVarName>[\w]+)[\s]*\)`

**Replaces:** Yes

**Replacement:** `{Replace_gridVarName}.setData({Replace_gridVarName}.getData())`

**Prerequisite:** `gridVarName` must have been captured.

**Before:**
```javascript
$.extend(list.getData().getItem(i), newData);
```

**After:**
```javascript
list.setData(list.getData());
```

**Side effects:** Stores `listRowIndexVarName = "i"` and `newColumnDataVarName = "newData"`.

---

#### Rule 15: `newData = {}` → `newData = grid.getData()[i]`

**Purpose:** Replace the empty object initialization of the update data variable with a reference to the actual grid row data, so that property assignments modify the row in place before `setData` is called.

**Search pattern:** `{Replace_newColumnDataVarName}[\s]*=[\s]*{}`

**Replaces:** Yes

**Replacement:** `{Replace_newColumnDataVarName} = {Replace_gridVarName}.getData()[{Replace_listRowIndexVarName}]`

**Prerequisites:** `newColumnDataVarName`, `gridVarName`, and `listRowIndexVarName` must have been captured by Rule 14.

**Before:**
```javascript
var newData = {};
```

**After:**
```javascript
var newData = list.getData()[i];
```

**Combined transformation context (Rules 14 + 15):**

The Classic UI pattern for updating grid data:
```javascript
var newData = {};
newData.FIELD1 = "value1";
newData.FIELD2 = "value2";
$.extend(list.getData().getItem(i), newData);
```

Becomes the V2 pattern:
```javascript
var newData = list.getData()[i];
newData.FIELD1 = "value1";
newData.FIELD2 = "value2";
list.setData(list.getData());
```

The key difference: in V2, `newData` is a direct reference to the row object in the data array, so assigning properties modifies the actual row data. Then `setData` triggers the grid to refresh with the updated data.

---

#### Rule 16: `.getData().deleteItem(` → `.removeRow(`

**Purpose:** Replace the SlickGrid delete pattern with the V2 grid method.

**Search pattern:** `{Replace_gridVarName}\.getData\(\)\.deleteItem\(`

**Replaces:** Yes

**Replacement:** `{Replace_gridVarName}.removeRow(`

**Prerequisite:** `gridVarName` must have been captured.

**Before:**
```javascript
list.getData().deleteItem(rowId);
```

**After:**
```javascript
list.removeRow(rowId);
```

---

#### Rule 17: `.reinit()` → Remove statement

**Purpose:** Remove the `.reinit()` call entirely. In V2 UI, grid reinitialization is not available or necessary.

**Search pattern:** `{Replace_gridVarName}\.reinit\(\);{0,1}`

**Replaces:** Yes (replaces with empty string — effectively removes the statement)

**Replacement:** *(empty)*

**Prerequisite:** `gridVarName` must have been captured.

**Before:**
```javascript
list.reinit();
```

**After:**
```javascript
// (statement removed)
```

**Note:** The semicolon is included in the match pattern (optionally) so the entire statement including its terminator is removed cleanly.

### 5.3 Complete Transformation Example

Below is a full example showing how the variable tracking and sequential rule application work together on a realistic script function:

**Before (Classic UI):**
```javascript
MyScript.prototype.processSelectedRow = function() {
    var list = this.controller.GetGrid();
    var selectedRows = list.getSelectedRows();
    var rowIndex = selectedRows[0];
    var row = list.getDataItem(rowIndex);
    var ITNO = row["C1"];
    var ITDS = row["C2"];
    var count = list.getData().getLength();

    // Update the row
    var newData = {};
    newData.STAT = "20";
    $.extend(list.getData().getItem(rowIndex), newData);
    list.reinit();
};
```

**After (V2 UI):**
```javascript
MyScript.prototype.processSelectedRow = function() {
    var list = this.controller.GetGrid();
    var selectedRows = list.getSelectedGridRows();
    var rowIndex = selectedRows[0].idx;
    var row = list.getData()[rowIndex];
    var ITNO = row[list.getColumns().find(c => c.colId == "C1").fullName];
    var ITDS = row[list.getColumns().find(c => c.colId == "C2").fullName];
    var count = list.getData().length;

    // Update the row
    var newData = list.getData()[rowIndex];
    newData.STAT = "20";
    list.setData(list.getData());

};
```

**Step-by-step variable tracking trace:**

| Step | Rule | Match | Store | Output |
|------|------|-------|-------|--------|
| 1 | Rule 1 (store) | `list.getData().getLength()` | `gridVarName = "list"` | *(no change)* |
| 2 | Rule 2 (store) | `list.getDataItem(` | `gridVarName = "list"` | *(no change)* |
| 3 | Rule 3 (store) | `list = this.controller.GetGrid()` | `gridVarName = "list"` | *(no change)* |
| 4 | Rule 4 | `selectedRows = list.getSelectedRows()` | `selectedRowsVar = "selectedRows"`, `gridVarName = "list"` | `selectedRows = list.getSelectedGridRows()` |
| 5 | Rule 5 | `selectedRows[0]` | — | `selectedRows[0].idx` |
| 6 | Rule 6 | *(no match — no inline column access after `.getDataItem()`)* | — | — |
| 7 | Rule 7 | `row = list.getDataItem(rowIndex)` | `rowVarName = "row"` | `row = list.getData()[rowIndex]` |
| 8–9 | Rules 8–9 | *(no match — no `.GetGrid().getDataItem()` pattern)* | — | — |
| 10 | Rule 10 | `.getData().getLength()` | — | `.getData().length` |
| 11 | Rule 11 | *(no match)* | — | — |
| 12 | Rule 12 | `row["C1"]`, `row["C2"]` | — | `row[list.getColumns().find(c => c.colId == "C1").fullName]`, etc. |
| 13 | Rule 13 | *(no match)* | — | — |
| 14 | Rule 14 | `$.extend(list.getData().getItem(rowIndex), newData)` | `listRowIndexVarName = "rowIndex"`, `newColumnDataVarName = "newData"` | `list.setData(list.getData())` |
| 15 | Rule 15 | `newData = {}` | — | `newData = list.getData()[rowIndex]` |
| 16 | Rule 16 | *(no match)* | — | — |
| 17 | Rule 17 | `list.reinit()` | — | *(removed)* |

### 5.4 Controller-Prefixed Access Pattern

When a script accesses the grid through a controller variable without first assigning it to a local variable, Rule 8 handles the full resolution:

**Before:**
```javascript
var ITNO = _this.controller.GetGrid().getDataItem(selectedItem)["C1"];
```

**After:**
```javascript
var ITNO = _this.controller.GetGrid().getData()[selectedItem][_this.controller.GetGrid().getColumns().find(c => c.colId == "C1").fullName];
```

The controller variable name (`_this.controller`) is captured from the match and replicated in the replacement to maintain the correct reference chain.

### 5.5 Graceful Degradation

If a replacement rule requires a stored variable that has not been captured (e.g., `gridVarName` is needed but no store-only rule matched in the current function):

1. The `{Replace_gridVarName}` token remains unresolved in the replacement string
2. The engine detects unresolved tokens (`{Replace_` still present in the output)
3. The replacement is **skipped** — the original code is left unchanged
4. A log entry is emitted: `"Could not migrate statement: <original>, ToString still contains statements that need to be replaced: <attempted output>"`

This ensures that partially-resolvable patterns never produce corrupted output.

### 5.6 Incompatible Grid Statement Detection

After all replacement rules have been applied, the engine scans for remaining grid patterns that were not transformed. These are flagged as potentially incompatible statements requiring manual review.

#### Detected Patterns

| Pattern | Level | Suggested V2 Equivalent |
|---------|-------|------------------------|
| `.getLength(` | WARN | `.length` |
| `.getDataItem(` | WARN | `.getData()[]` |
| `.getSelectedRows()` | WARN | `.getSelectedGridRows()` |
| `.getColumnIndex(` | WARN | `.getColumns()` |
| `.deleteItem(` | WARN | `.removeRow()` |

#### Flagging Behavior

When an incompatible pattern is detected post-transformation:

1. The statement is logged with the script name, function name, matched text, and suggested replacement
2. A `// TODO [H5 Migration]:` comment is inserted above the statement in the output code
3. The original statement is preserved unchanged

**Example flagged output:**
```javascript
// TODO [H5 Migration]: Replace .getDataItem() with .getData()[] - manual review required
var data = someUnknownGrid.getDataItem(idx);
```

#### Why Incompatible Statements Remain

Incompatible grid statements typically remain after transformation when:
- The grid variable could not be captured (e.g., grid is passed as a function parameter without prior assignment)
- The pattern uses an unexpected calling convention not covered by the replacement rules
- The statement is inside a complex expression that the regex cannot fully parse
- The code references a grid from a different scope (e.g., a callback from another module)

In all these cases, the developer must manually identify the correct grid reference and apply the appropriate V2 pattern.

## 6. Message Dialog Migration

This section covers the transformation of Classic UI `ShowDialog` calls into V2 UI `ConfirmDialog.ShowMessageDialog` calls. The Classic UI uses a controller-prefixed method call with positional arguments, while V2 UI uses a structured options object.

### 6.1 Detection Pattern

The migration engine detects `ShowDialog` calls using the following regex:

```regex
[\w.]*Controller\.ShowDialog[\s]*\((?<arguments>.*)\)
```

This matches any prefix before `Controller.ShowDialog`, including:
- `this.controller.Controller.ShowDialog(...)`
- `_this.controller.Controller.ShowDialog(...)`
- `self.controller.Controller.ShowDialog(...)`

The `arguments` named group captures everything inside the parentheses.

### 6.2 Argument Interpretation

The captured arguments string is split by comma. The Classic UI `ShowDialog` signature uses positional parameters:

| Position | Name | Purpose | Used in V2 |
|----------|------|---------|-------------|
| 1st | `arg1` | Header message OR message content (context-dependent) | Yes |
| 2nd | `arg2` | Detail message OR `null` (determines mapping logic) | Yes |
| 3rd | `arg3` | Unknown/unused (typically a boolean) | **Discarded** |
| 4th | `arg4` | Unknown/unused (typically a boolean) | **Discarded** |

The 3rd and 4th arguments from the original call are always discarded in the V2 replacement.

### 6.3 Rule 1: Detail Message Provided (arg2 ≠ null)

When the `ShowDialog` call has 2 or more arguments and the second argument is **not** `null`, the first argument is the header and the second argument is the detail message.

**Mapping:**
- `header` ← `arg1` (first argument, as-is)
- `message` ← `arg2` (second argument, trimmed)

**Before (Classic UI):**

```javascript
this.controller.Controller.ShowDialog("Failed adding voucher header", result["Message"], null, true);
```

**After (V2 UI):**

```javascript
ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: "Failed adding voucher header", message: result["Message"]});
```

**Additional example — variable references:**

**Before:**

```javascript
_this.controller.Controller.ShowDialog(headerText, detailText, false, true);
```

**After:**

```javascript
ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: headerText, message: detailText});
```

### 6.4 Rule 2: Null Detail (arg2 = null)

When the `ShowDialog` call has 2 or more arguments and the second argument **is** `null`, the first argument becomes the message content and the header is set to an empty string.

**Mapping:**
- `header` ← `""` (empty string)
- `message` ← `arg1` (first argument — the message content)

**Before (Classic UI):**

```javascript
this.controller.Controller.ShowDialog(message, null, true);
```

**After (V2 UI):**

```javascript
ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: "", message: message});
```

**Additional example — string literal message:**

**Before:**

```javascript
this.controller.Controller.ShowDialog("Operation completed successfully", null, true);
```

**After:**

```javascript
ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: "", message: "Operation completed successfully"});
```

### 6.5 Rule 3: Fewer Than 2 Arguments — Incompatible Statement

When the `ShowDialog` call has fewer than 2 arguments, the migration engine **cannot** determine the header/message mapping. The statement is flagged as an incompatible statement requiring manual review.

**Behavior:**
- The original statement is left **unchanged**
- A `// TODO [H5 Migration]:` comment is inserted above the statement
- No replacement is made

**Before (Classic UI):**

```javascript
this.controller.Controller.ShowDialog(message);
```

**After (flagged — no transformation applied):**

```javascript
// TODO [H5 Migration]: Replace ShowDialog with ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: <header>, message: <message> })
this.controller.Controller.ShowDialog(message);
```

### 6.6 Replacement Format

The V2 UI replacement always follows this exact format:

```javascript
ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: <headerMessage>, message: <detailMessage>})
```

Key formatting notes:
- The options object is on a single line
- `dialogType` is always `"Alert"`
- Property values use the original argument expressions (variable names, string literals, property accessors) without modification
- No trailing semicolons are added by the replacement itself (the original statement's semicolons are preserved from the surrounding code)

### 6.7 Iterative Matching

The migration engine processes `ShowDialog` calls iteratively within each function body:

1. Start matching from position 0 in the function content
2. When a match is found, apply the replacement (if 2+ arguments)
3. Continue searching from the position after the last match
4. Repeat until no more matches are found

This ensures multiple `ShowDialog` calls within the same function are all transformed.

### 6.8 Edge Cases

| Scenario | Behavior |
|----------|----------|
| Multiple `ShowDialog` calls in one function | Each is matched and transformed independently |
| Argument contains commas inside strings (protected by preprocessing) | String literals are placeholders during matching — comma split operates only on actual argument separators |
| `ShowDialog` inside a comment or string literal | Protected by preprocessing placeholders — not matched |
| Controller prefix varies (`this.controller.Controller`, `_this.controller.Controller`, etc.) | The regex `[\w.]*Controller\.ShowDialog` handles any word/dot prefix |
| Whitespace between `ShowDialog` and `(` | The regex `[\s]*` allows optional whitespace before the opening parenthesis |

### 6.9 V2 UI ConfirmDialog Reference

For the full `ConfirmDialog.ShowMessageDialog` options interface (including `withCancelButton`, `canHide`, `closed` callback, and custom buttons), see **Section 3.7**. The message dialog migration always uses `dialogType: "Alert"` — if a different dialog type is needed (e.g., `"Question"` for confirmation flows), the developer must adjust manually after migration.

## 7. Custom Dialog Migration

This section covers all transformations related to jQuery-based custom dialogs (`.inforMessageDialog`) being migrated to V2 UI dialogs (`H5ControlUtil.H5Dialog.CreateDialogElement`). Custom dialogs are more complex than message dialogs because they involve dialog content elements, option objects with handler callbacks, embedded grids, and scoping issues with `this` references inside click handlers.

### 7.1 Overview and Processing Order

Custom dialog migration applies **only** to functions that contain a `.inforMessageDialog(` call. The migration engine first checks for the presence of this pattern, and if found, applies the following transformations in order:

1. **`this`-capturing variable** — Ensure a variable exists to capture `this` for use inside click handlers
2. **`open:` handler relocation** — Extract the body of `open: function(){...}`, move it after the `CreateDialogElement` call, remove the `open` property
3. **CSV-based replacements** (applied sequentially in CSV row order):
   - `.inforMessageDialog(opts)` → `H5ControlUtil.H5Dialog.CreateDialogElement(content[0], opts)`
   - `click: function()` → `click: function(event, model)`
   - `H5ControlUtil.H5Dialog.DestroyDialog(...)` → `model.close(true)`
   - `Configuration.Current.ListConfig(id, cols, data)` → remove statement, store cols/data
   - `.inforDataGrid(options)` → `.datagrid({columns: cols, dataset: data})`
4. **`this.functionName()` replacement in click handlers** — Replace with `self.functionName()` (or the existing this-capturing variable)

### 7.2 Rule 1: `this`-Capturing Variable Insertion

**Purpose:** Custom dialog click handlers execute in a different scope than the enclosing function. References to `this.methodName()` inside click handlers must be replaced with a captured variable (`self` or an existing alias). This step ensures the variable exists before other transformations run.

**Logic:**
1. Check if the function body already contains a variable assigned to `this` (e.g., `var _this = this`, `const self = this`, `let that = this`)
2. If found: reuse that variable name (e.g., `_this`) — no insertion needed
3. If not found: insert a new variable at the start of the function body:
   - TypeScript: `const self = this;`
   - JavaScript: `var self = this;`

**Detection regex:** `(?<thisVarName>[\w]+)[\s]*=[\s]*this`

**Before (no existing variable — TypeScript):**

```typescript
MyScript.prototype.openDialog = function() {
    var dialogContent = $('<div>...</div>');
    var dialogOptions = {
        // ...
    };
    dialogContent.inforMessageDialog(dialogOptions);
};
```

**After (insertion at function start):**

```typescript
MyScript.prototype.openDialog = function() {
    const self = this;
    var dialogContent = $('<div>...</div>');
    var dialogOptions = {
        // ...
    };
    dialogContent.inforMessageDialog(dialogOptions);
};
```

**Before (no existing variable — JavaScript):**

```javascript
MyScript.prototype.openDialog = function() {
    var dialogContent = $('<div>...</div>');
    // ...
};
```

**After:**

```javascript
MyScript.prototype.openDialog = function() {
    var self = this;
    var dialogContent = $('<div>...</div>');
    // ...
};
```

**Before (existing variable reuse):**

```javascript
MyScript.prototype.openDialog = function() {
    var _this = this;
    var dialogContent = $('<div>...</div>');
    // ...
};
```

**After (no insertion — `_this` is reused in subsequent `this.` replacements):**

```javascript
MyScript.prototype.openDialog = function() {
    var _this = this;
    var dialogContent = $('<div>...</div>');
    // ...
};
```

The captured variable name (`self`, `_this`, `that`, etc.) is stored and used in Rule 7 for replacing `this.functionName()` calls inside click handlers.

---

### 7.3 Rule 2: `open:` Handler Relocation

**Purpose:** The `open: function(){...}` handler in Classic UI dialog options fires when the dialog opens. In V2 UI, the `CreateDialogElement` API does not support an `open` callback. The body of the `open` function must be moved to execute immediately **after** the `CreateDialogElement` call.

**Detection regex:** `open[\s]*:[\s]*function[\s]*.*{`

**Transformation steps:**
1. Find the `open: function() {` handler in the function content
2. Extract the body of the `open` function (everything between the opening `{` and its matching closing `}`, using curly-brace counting)
3. Check if the character after the closing `}` is a comma — if so, remove the comma as well
4. Insert the extracted body immediately after the `.inforMessageDialog(...)` statement (at this point the `.inforMessageDialog` call has not yet been transformed to `CreateDialogElement`)
5. Remove the entire `open: function(){...}` property from the dialog options object

**Before:**

```javascript
var dialogOptions = {
    title: 'My Dialog',
    open: function() {
        $('#myField').val('default');
        initializeGrid();
    },
    click: function() {
        // handle click
    }
};
dialogContent.inforMessageDialog(dialogOptions);
```

**After:**

```javascript
var dialogOptions = {
    title: 'My Dialog',
    click: function() {
        // handle click
    }
};
dialogContent.inforMessageDialog(dialogOptions);
    $('#myField').val('default');
    initializeGrid();
```

> **Note:** The body is placed after the `.inforMessageDialog(...)` statement at this stage. Later, Rule 3 transforms `.inforMessageDialog(...)` into `CreateDialogElement(...)`, so in the final output, the open body appears after the `CreateDialogElement` call. The indentation of the moved content matches the indentation level of the `.inforMessageDialog` line.

**Edge cases:**
- Multiple `open:` handlers in the same function: each is processed independently (iterative matching from last match position forward)
- Empty `open: function() {}`: the empty body is still removed along with the property and trailing comma
- `open` property without trailing comma (last property in object): no comma removal needed

---

### 7.4 Rule 3: `.inforMessageDialog(opts)` → `CreateDialogElement(content[0], opts)`

**Purpose:** Replace the jQuery dialog creation method with the V2 UI dialog API.

**Search regex:** `(?<dialogContentVar>[\w]+)\.inforMessageDialog[\s]*\([\s]*(?<dialogOptionsVar>[\w]+)[\s]*\)`

**Replacement:** `H5ControlUtil.H5Dialog.CreateDialogElement({Replace_dialogContentVar}[0], {Replace_dialogOptionsVar})`

**Variable captures:**
- `dialogContentVar` — the jQuery element variable holding the dialog content
- `dialogOptionsVar` — the options object variable

The replacement appends `[0]` to the content variable to extract the raw DOM element from the jQuery wrapper (required by `CreateDialogElement`).

**Before:**

```javascript
dialogContent.inforMessageDialog(dialogOptions);
```

**After:**

```javascript
H5ControlUtil.H5Dialog.CreateDialogElement(dialogContent[0], dialogOptions);
```

**Additional example:**

**Before:**

```javascript
$dlgContent.inforMessageDialog( opts );
```

**After:**

```javascript
H5ControlUtil.H5Dialog.CreateDialogElement($dlgContent[0], opts);
```

---

### 7.5 Rule 4: `click: function()` → `click: function(event, model)`

**Purpose:** V2 UI dialog click handlers receive two parameters: `event` (the click event) and `model` (the dialog model, used for closing the dialog). Classic UI click handlers have no parameters.

**Search regex:** `click[\s]*:[\s]*function[\s]*\([\s]*\)`

**Replacement:** `click: function(event, model)`

**Before:**

```javascript
var dialogOptions = {
    title: 'Confirm',
    click: function() {
        // process and close
        H5ControlUtil.H5Dialog.DestroyDialog($(SheetDialog));
    }
};
```

**After:**

```javascript
var dialogOptions = {
    title: 'Confirm',
    click: function(event, model) {
        // process and close
        H5ControlUtil.H5Dialog.DestroyDialog($(SheetDialog));
    }
};
```

**Note:** This rule uses a simple text replacement and applies to all `click: function()` patterns in the function, regardless of nesting level. It is applied after the `open:` handler relocation so it does not accidentally match handlers in the `open` block.

---

### 7.6 Rule 5: `H5ControlUtil.H5Dialog.DestroyDialog(...)` → `model.close(true)`

**Purpose:** In V2 UI, dialogs are closed using the `model` parameter received by the click handler, not by calling `DestroyDialog` with a jQuery selector.

**Search regex:** `H5ControlUtil\.H5Dialog\.DestroyDialog[\s]*\(.*\)`

**Replacement:** `model.close(true)`

The regex matches `DestroyDialog` with any argument (the jQuery selector expression is discarded).

**Before:**

```javascript
click: function(event, model) {
    self.processData();
    H5ControlUtil.H5Dialog.DestroyDialog($(SheetDialog));
}
```

**After:**

```javascript
click: function(event, model) {
    self.processData();
    model.close(true);
}
```

**Additional example:**

**Before:**

```javascript
H5ControlUtil.H5Dialog.DestroyDialog($('#myDialog'));
```

**After:**

```javascript
model.close(true);
```

**Note:** The `model` variable is available because Rule 4 added `(event, model)` parameters to the click handler. If `DestroyDialog` appears outside a click handler, it will still be replaced with `model.close(true)` — this may require manual review if the `model` variable is not in scope.

---

### 7.7 Rule 6: `Configuration.Current.ListConfig(id, cols, data)` → Remove and Store Parameters

**Purpose:** The `ListConfig` call in Classic UI configures a grid's columns and data source. In V2 UI, this is not needed — the columns and data are passed directly to `.datagrid()`. The statement is deleted, but the column and data parameters are stored for use in the subsequent `.inforDataGrid` replacement.

**Search regex:** `([\w]+[\s]*=[\s]*){0,1}Configuration\.Current\.ListConfig\((?<Store_InforDataGridIdPropParam>[\w\s'"]+),(?<Store_InforDataGridColumnsParam>[\w\s'"]+),(?<Store_InforDataGridDataparam>[\w\s'"]+)\)[\s]*;{0,1}`

**Replacement:** *(empty — statement is deleted)*

**Stored variables:**
- `InforDataGridColumnsParam` — the second argument (column definitions variable name)
- `InforDataGridDataparam` — the third argument (data source variable name)

The first argument (ID/identifier) is discarded.

**Before:**

```javascript
options = Configuration.Current.ListConfig('id', columns, data);
```

**After:**

```javascript
// (statement removed)
```

The stored values `columns` and `data` are then used by Rule 7.

**Additional example:**

**Before:**

```javascript
var gridConfig = Configuration.Current.ListConfig('showErrorsGrid', cols, gridData);
```

**After:**

```javascript
// (statement removed — cols and gridData stored for .datagrid() replacement)
```

---

### 7.8 Rule 7: `.inforDataGrid(options)` → `.datagrid({columns: cols, dataset: data})`

**Purpose:** Replace the Classic UI jQuery data grid creation with the V2 IDS-based datagrid, using the column and data parameters stored from the `ListConfig` call.

**Search regex:** `\.inforDataGrid\(.*?\)`

**Replacement:** `.datagrid({columns: {Replace_InforDataGridColumnsParam}, dataset: {Replace_InforDataGridDataparam}})`

**Prerequisite:** `InforDataGridColumnsParam` and `InforDataGridDataparam` must have been stored from a preceding `ListConfig` call (Rule 6) within the same function context.

**Before:**

```javascript
options = Configuration.Current.ListConfig('id', columns, data);
grid = $('#showErrorsGrid').inforDataGrid(options);
```

**After (both rules applied):**

```javascript
grid = $('#showErrorsGrid').datagrid({columns: columns, dataset: data});
```

**Additional example with different variable names:**

**Before:**

```javascript
var cfg = Configuration.Current.ListConfig('errorGrid', errorColumns, errorData);
var errorGrid = $("#errorGridContainer").inforDataGrid(cfg);
```

**After:**

```javascript
var errorGrid = $("#errorGridContainer").datagrid({columns: errorColumns, dataset: errorData});
```

**Note:** If `.inforDataGrid()` is found but no `ListConfig` data was stored (the `{Replace_...}` placeholders cannot be resolved), the replacement is **skipped** — the original statement is left unchanged and flagged for manual review.

---

### 7.9 Rule 8: `this.functionName()` → `self.functionName()` in Click Handlers

**Purpose:** Inside dialog click handlers, `this` refers to the dialog scope, not the script class instance. Method calls to other class functions must use the captured `this` variable (typically `self` or an existing alias like `_this`).

**Scope:** This replacement only applies **inside `click: function(event, model){...}` handler bodies**.

**Logic:**
1. Locate each `click: function(...){...}` block (using curly-brace counting to find the matching closing brace)
2. For each function name in the class's function list (`functionsOfClass`):
   - Search for calls matching `this.functionName(` or standalone `functionName(` within the click handler body
   - Replace with `<thisVarName>.functionName(` where `<thisVarName>` is the captured variable from Rule 1

**Detection regex (per function name):** `\s(?<thisStatement>this\.){0,1}(?<functionName>functionNameHere)[\s]*\(`

**Important constraints:**
- Only function names that exist as methods on the **same class** are replaced (the migration engine checks against the full list of class functions)
- The function's own name is skipped (no self-reference replacement)
- The replacement uses the variable name determined in Rule 1 (`self`, `_this`, `that`, etc.)

**Before:**

```javascript
MyScript.prototype.openDialog = function() {
    var self = this;
    var dialogOptions = {
        click: function(event, model) {
            this.processResult();
            this.closeDialog();
            model.close(true);
        }
    };
    H5ControlUtil.H5Dialog.CreateDialogElement(dialogContent[0], dialogOptions);
};

MyScript.prototype.processResult = function() { /* ... */ };
MyScript.prototype.closeDialog = function() { /* ... */ };
```

**After:**

```javascript
MyScript.prototype.openDialog = function() {
    var self = this;
    var dialogOptions = {
        click: function(event, model) {
            self.processResult();
            self.closeDialog();
            model.close(true);
        }
    };
    H5ControlUtil.H5Dialog.CreateDialogElement(dialogContent[0], dialogOptions);
};

MyScript.prototype.processResult = function() { /* ... */ };
MyScript.prototype.closeDialog = function() { /* ... */ };
```

**Example with existing `_this` variable:**

**Before:**

```javascript
MyScript.prototype.showConfirm = function() {
    var _this = this;
    var opts = {
        click: function(event, model) {
            this.saveData();
            model.close(true);
        }
    };
    H5ControlUtil.H5Dialog.CreateDialogElement(content[0], opts);
};

MyScript.prototype.saveData = function() { /* ... */ };
```

**After:**

```javascript
MyScript.prototype.showConfirm = function() {
    var _this = this;
    var opts = {
        click: function(event, model) {
            _this.saveData();
            model.close(true);
        }
    };
    H5ControlUtil.H5Dialog.CreateDialogElement(content[0], opts);
};

MyScript.prototype.saveData = function() { /* ... */ };
```

**Edge cases:**
- Functions not in the class function list are NOT replaced (they may be jQuery methods, utility functions, etc.)
- Multiple click handlers in the same function: each is processed independently
- Nested function calls (e.g., `this.getResult().process()`) — only the first `this.` before a class method name is replaced

---

### 7.10 Rule 9: `<controllerVar>.Controller.ShowDialog(args)` → `ConfirmDialog.ShowMessageDialog(...)`

**Purpose:** Some custom dialog functions also contain `ShowDialog` calls for simple messages. These follow the same mapping as the Message Dialog section (Section 6), but are documented here for completeness within the custom dialog context.

**Mapping:**
- If second argument is not `null`: `ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: arg1, message: arg2 })`
- If second argument is `null`: `ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: "", message: arg1 })`

**Before:**

```javascript
this.controller.Controller.ShowDialog("Error", "Record not found", true);
```

**After:**

```javascript
ConfirmDialog.ShowMessageDialog({ dialogType: "Alert", header: "Error", message: "Record not found"});
```

See **Section 6** for the full ShowDialog migration rules including edge cases and fewer-than-2-argument handling.

---

### 7.11 Complete Transformation Example

Below is a full example showing how all custom dialog rules work together on a realistic function:

**Before (Classic UI):**

```javascript
MyScript.prototype.showErrorDialog = function() {
    var columns = [{ name: 'Error', field: 'error' }, { name: 'Line', field: 'line' }];
    var data = [{ error: 'Missing field', line: 10 }];
    var options = Configuration.Current.ListConfig('errorGrid', columns, data);

    var dialogContent = $('<div><div id="errorGrid"></div></div>');
    var dialogOptions = {
        title: 'Errors Found',
        modal: true,
        open: function() {
            var grid = $('#errorGrid').inforDataGrid(options);
            grid.setColumns(columns);
        },
        buttons: [{
            text: 'OK',
            click: function() {
                this.handleClose();
                H5ControlUtil.H5Dialog.DestroyDialog($(dialogContent));
            }
        }]
    };
    dialogContent.inforMessageDialog(dialogOptions);
};

MyScript.prototype.handleClose = function() {
    // cleanup logic
};
```

**After (V2 UI — all rules applied):**

```javascript
MyScript.prototype.showErrorDialog = function() {
    var self = this;
    var columns = [{ name: 'Error', field: 'error' }, { name: 'Line', field: 'line' }];
    var data = [{ error: 'Missing field', line: 10 }];

    var dialogContent = $('<div><div id="errorGrid"></div></div>');
    var dialogOptions = {
        title: 'Errors Found',
        modal: true,
        buttons: [{
            text: 'OK',
            click: function(event, model) {
                self.handleClose();
                model.close(true);
            }
        }]
    };
    H5ControlUtil.H5Dialog.CreateDialogElement(dialogContent[0], dialogOptions);
    var grid = $('#errorGrid').datagrid({columns: columns, dataset: data});
    grid.setColumns(columns);
};

MyScript.prototype.handleClose = function() {
    // cleanup logic
};
```

**Transformation breakdown:**
1. **Rule 1 (this-variable):** `var self = this;` inserted at function start (no existing this-capture found)
2. **Rule 6 (ListConfig):** `Configuration.Current.ListConfig('errorGrid', columns, data)` removed; `columns` and `data` stored
3. **Rule 2 (open handler):** Body of `open: function(){...}` extracted and moved after `dialogContent.inforMessageDialog(dialogOptions);`; `open` property and trailing comma removed from options
4. **Rule 3 (inforMessageDialog):** `dialogContent.inforMessageDialog(dialogOptions)` → `H5ControlUtil.H5Dialog.CreateDialogElement(dialogContent[0], dialogOptions)`
5. **Rule 4 (click params):** `click: function()` → `click: function(event, model)`
6. **Rule 5 (DestroyDialog):** `H5ControlUtil.H5Dialog.DestroyDialog($(dialogContent))` → `model.close(true)`
7. **Rule 7 (inforDataGrid):** `$('#errorGrid').inforDataGrid(options)` → `$('#errorGrid').datagrid({columns: columns, dataset: data})`
8. **Rule 8 (this→self):** `this.handleClose()` → `self.handleClose()` (inside click handler, `handleClose` is a class method)

### 7.12 Iterative Matching

All CSV-based replacement rules (Rules 3–7) are applied iteratively within the function body:

1. Start matching from position 0 in the function content
2. When a match is found, apply the replacement
3. Continue searching from the position after the last match
4. Repeat until no more matches are found for the current rule
5. Move to the next rule in CSV order

This ensures multiple occurrences of the same pattern within a single function are all transformed.

### 7.13 Variable Storage Across Rules

The custom dialog migration uses a variable storage mechanism for cross-rule dependencies:

| Stored Variable | Source Rule | Consumed By |
|---|---|---|
| `thisVarName` | Rule 1 (this-capture) | Rule 8 (this→self replacement) |
| `InforDataGridColumnsParam` | Rule 6 (ListConfig removal) | Rule 7 (datagrid replacement) |
| `InforDataGridDataparam` | Rule 6 (ListConfig removal) | Rule 7 (datagrid replacement) |

If a consumed variable was never stored (e.g., `.inforDataGrid()` appears without a preceding `ListConfig`), the replacement template retains the `{Replace_...}` placeholder. When this happens:
- The replacement is **skipped** (original code preserved)
- A warning is logged: `"Could not migrate statement: <match>, ToString still contains statements that need to be replaced: <template>"`

### 7.14 Incompatible Statement Detection (Post-Transformation)

After all custom dialog transformations are applied, the following patterns are scanned as potential incompatible statements:

| Remaining Pattern | Severity | Suggested V2 Equivalent |
|---|---|---|
| `.inforMessageDialog(` | WARN | `H5ControlUtil.H5Dialog.CreateDialogElement(` |
| `H5ControlUtil.H5Dialog.DestroyDialog(...)` | WARN | `model.close(true)` |
| `.ListConfig(` | WARN | *(manual review — remove and pass cols/data to .datagrid())* |
| `.inforDataGrid(` | WARN | `.datagrid({columns: columns, dataset: data})` |

For each detected incompatible statement, the migration engine:
1. Logs the occurrence with script name, function name, matched statement, and suggested replacement
2. Preserves the original code unchanged
3. Adds a `// TODO [H5 Migration]:` comment above the statement (applied in the Incompatible Statement Detection phase — Section 10)

### 7.15 Edge Cases and Special Handling

| Scenario | Behavior |
|----------|----------|
| Function has no `.inforMessageDialog(` call | Skip all custom dialog processing for that function |
| Multiple dialogs in one function | Each `.inforMessageDialog` triggers its own `open:` relocation pass |
| `click` handler with existing parameters (non-empty) | The regex `click[\s]*:[\s]*function[\s]*\([\s]*\)` only matches zero-parameter handlers — handlers with existing params are left unchanged |
| `open:` handler with parameters (e.g., `open: function(event)`) | Matched by the regex `open[\s]*:[\s]*function[\s]*.*{` — still relocated |
| `this.method()` where method is NOT a class function | Not replaced (only class method names from `functionsOfClass` list are targeted) |
| Nested dialogs (dialog opens another dialog) | Each is processed in the iterative pass; inner dialogs are transformed when the outer function is processed |
| `DestroyDialog` outside a click handler | Still replaced with `model.close(true)` — may need manual review if `model` is not in scope |
| TypeScript vs JavaScript | Only affects the `this`-capture insertion: `const self = this;` for TS, `var self = this;` for JS |

### 7.16 Source Cross-Reference

| Rule | CSV Source | Helper Script |
|------|-----------|---------------|
| Rule 1 (this-variable) | — | `CustomDialogHelper.ps1` (lines 34–50) |
| Rule 2 (open handler) | — | `CustomDialogHelper.ps1` (lines 55–115) |
| Rule 3 (inforMessageDialog) | `ReplaceList_CustomDialogs.csv` row 1 | — |
| Rule 4 (click params) | `ReplaceList_CustomDialogs.csv` row 2 | — |
| Rule 5 (DestroyDialog) | `ReplaceList_CustomDialogs.csv` row 3 | — |
| Rule 6 (ListConfig) | `ReplaceList_CustomDialogs.csv` row 4 | — |
| Rule 7 (inforDataGrid) | `ReplaceList_CustomDialogs.csv` row 5 | — |
| Rule 8 (this→self) | — | `CustomDialogHelper.ps1` (lines 155–210) |
| Rule 9 (ShowDialog) | — | `MessageDialogHelper.ps1` |
| Incompatible detection | `IncompatibleStatementsList_CustomDialogStatements.csv` | `CustomDialogHelper.ps1` (function `SearchForIncompatibleCustomDialogStatements`) |

## 8. Function Content Migration

This section documents the general-purpose function content transformations that handle utility patterns such as user context access, debug logging, field state changes, busy indicators, and date picker element upgrades. These rules are applied **after** API, Grid, Message Dialog, and Custom Dialog transformations (Phase 5 of 6).

### Rule Application Order

Rules are applied **sequentially in CSV order** (top-to-bottom from `ReplaceList_FunctionContent.csv`). Order matters because earlier rules may transform code that later rules depend on matching. Each rule is applied iteratively (all occurrences matched before moving to the next rule).

| # | Pattern | Replacement |
|---|---------|-------------|
| 1 | `userContext[` (bracket notation) | `ScriptUtil.GetUserContext()[` |
| 2 | `userContext.` (dot notation) | `ScriptUtil.GetUserContext().` |
| 3 | `ScriptDebugConsole.WriteLine(` | `console.log(` |
| 4 | `.GetContentElement().GetElement(f).readOnly` | `.GetElement(f).readonly` |
| 5 | `.readOnly()` (standalone) | `.readonly()` |
| 6 | `$(...).inforBusyIndicator({...})` (object arg) | `ScriptUtil.getController().showBusyIndicator()` |
| 7 | `$(...).inforBusyIndicator("close")` (word arg) | `ScriptUtil.getController().hideBusyIndicator()` |
| 8 | `.GetElement(f).prop("disabled", true)` | `.GetElement(f).readonly()` |
| 9 | `.GetElement(f).prop("disabled", false)` | `.GetElement(f).enable()` |
| 10 | `.GetContentElement().GetElement(f)` (standalone) | `.GetElement(f)` |
| 11 | `<var>.inforDateField(...)` | *(remove statement, store jQuery element variable name)* |
| 12 | `<jQueryVar> = <contentElement>.AddElement(<element>` | *(no change, store element variable name)* |
| 13 | `<element> = new TextBoxElement` | `<element> = new DatePickerElement` |

---

### 8.1 Rule 1: `userContext[` → `ScriptUtil.GetUserContext()[`

**Purpose:** Replace direct `userContext` bracket-notation access with the V2 UI utility method.

**Regex:** `userContext[\s]*\[`
**Replacement:** `ScriptUtil.GetUserContext()[`

The regex allows optional whitespace between `userContext` and `[` to handle formatting variations.

**Before:**

```javascript
var division = userContext['CurrentDivision'];
var company = userContext ['CONO'];
```

**After:**

```javascript
var division = ScriptUtil.GetUserContext()['CurrentDivision'];
var company = ScriptUtil.GetUserContext()['CONO'];
```

---

### 8.2 Rule 2: `userContext.` → `ScriptUtil.GetUserContext().`

**Purpose:** Replace direct `userContext` dot-notation access with the V2 UI utility method.

**Regex:** `userContext[\s]*\.`
**Replacement:** `ScriptUtil.GetUserContext().`

The regex allows optional whitespace between `userContext` and `.` to handle formatting variations.

**Before:**

```javascript
var user = userContext.USID;
var language = userContext .LANC;
```

**After:**

```javascript
var user = ScriptUtil.GetUserContext().USID;
var language = ScriptUtil.GetUserContext().LANC;
```

**Order dependency:** Rules 1 and 2 must run before any other rules. They target a distinct identifier (`userContext`) and do not overlap with subsequent patterns.

---

### 8.3 Rule 3: `ScriptDebugConsole.WriteLine(` → `console.log(`

**Purpose:** Replace the Classic UI debug console with standard `console.log`.

**Regex:** `ScriptDebugConsole.WriteLine[\s]*\(`
**Replacement:** `console.log(`

The regex allows optional whitespace before the opening parenthesis.

**Before:**

```javascript
ScriptDebugConsole.WriteLine("Processing record: " + id);
ScriptDebugConsole.WriteLine (result.Message);
```

**After:**

```javascript
console.log("Processing record: " + id);
console.log(result.Message);
```

---

### 8.4 Rule 4: `.GetContentElement().GetElement(f).readOnly` → `.GetElement(f).readonly`

**Purpose:** Combined transformation that both removes the intermediate `.GetContentElement()` call and corrects the casing from `.readOnly` to `.readonly` (lowercase 'o'). This rule MUST run before Rule 5 so that `.readOnly` instances preceded by `.GetContentElement()` are handled as a combined transformation rather than partially.

**Regex:** `\.GetContentElement\(\).GetElement\((?<fieldName>.*?)\)\.readOnly`
**Replacement:** `.GetElement({Replace_fieldName}).readonly`

The named capture group `fieldName` captures the field argument and inserts it into the replacement.

**Before:**

```javascript
_this.controller.GetContentElement().GetElement("WRTEPY").readOnly();
this.controller.GetContentElement().GetElement("MMSTAT").readOnly();
```

**After:**

```javascript
_this.controller.GetElement("WRTEPY").readonly();
this.controller.GetElement("MMSTAT").readonly();
```

**Order dependency:** Must precede Rule 5 (standalone `.readOnly()`) to prevent partial matching. If Rule 5 ran first, it would change `.readOnly()` to `.readonly()` but leave `.GetContentElement()` in place, and Rule 4's regex would no longer match (it looks for `.readOnly`, not `.readonly`).

---

### 8.5 Rule 5: `.readOnly()` → `.readonly()` (standalone)

**Purpose:** Catch any remaining `.readOnly()` calls (not already handled by Rule 4) and correct the casing to `.readonly()`.

**Regex:** `\.readOnly\(\)`
**Replacement:** `.readonly()`

**Before:**

```javascript
_this.controller.GetElement("WRTEPY").readOnly();
element.readOnly();
```

**After:**

```javascript
_this.controller.GetElement("WRTEPY").readonly();
element.readonly();
```

**Order dependency:** Runs after Rule 4. Any `.readOnly()` that was part of a `.GetContentElement().GetElement(f).readOnly()` chain has already been handled. This rule catches remaining instances that appear without `.GetContentElement()`.

---

### 8.6 Rule 6: `$(...).inforBusyIndicator({...})` → `ScriptUtil.getController().showBusyIndicator()`

**Purpose:** Replace jQuery-based busy indicator **show** calls (identified by an object literal `{...}` argument containing configuration options) with the V2 UI controller method.

**Regex:** `\$.*\.inforBusyIndicator[\s]*\([\s]*{.*}[\s]*\)`
**Replacement:** `ScriptUtil.getController().showBusyIndicator()`

The regex matches any jQuery selector (`$...`) followed by `.inforBusyIndicator(` with an object literal `{...}` argument (the "show" invocation pattern).

**Before:**

```javascript
$("body").inforBusyIndicator({ delay: 100, modal: true });
$('#content').inforBusyIndicator({modal: true});
```

**After:**

```javascript
ScriptUtil.getController().showBusyIndicator();
ScriptUtil.getController().showBusyIndicator();
```

**Note:** The jQuery selector expression is discarded — V2 UI busy indicators are managed at the controller level, not on arbitrary DOM elements.

---

### 8.7 Rule 7: `$(...).inforBusyIndicator("close")` → `ScriptUtil.getController().hideBusyIndicator()`

**Purpose:** Replace jQuery-based busy indicator **hide/close** calls (identified by a word/string argument like `"close"`) with the V2 UI controller method.

**Regex:** `\$.*\.inforBusyIndicator[\s]*\([\w\s]+\)`
**Replacement:** `ScriptUtil.getController().hideBusyIndicator()`

The regex matches any jQuery selector followed by `.inforBusyIndicator(` with a word/string argument (no curly braces — distinguishes it from the "show" pattern in Rule 6).

**Before:**

```javascript
$("body").inforBusyIndicator("close");
$('#content').inforBusyIndicator( close );
```

**After:**

```javascript
ScriptUtil.getController().hideBusyIndicator();
ScriptUtil.getController().hideBusyIndicator();
```

**Order dependency:** Rules 6 and 7 together cover all `.inforBusyIndicator()` patterns. Rule 6 matches object-argument calls (show), Rule 7 matches word-argument calls (hide). Their order relative to each other is not critical since their regexes are mutually exclusive.

---

### 8.8 Rule 8: `.GetElement(f).prop("disabled", true)` → `.GetElement(f).readonly()`

**Purpose:** Replace jQuery `.prop("disabled", true)` with the V2 UI `.readonly()` method for disabling form fields.

**Regex:** `\.GetElement\((?<fieldName>.*?)\)\.prop\((?<RestorePlaceholderQuotedString>ScriptPlaceHolderQuotedString[0-9]{10}),[\s]*true[\s]*\)`
**Replacement:** `.GetElement({Replace_fieldName}).readonly()`

#### RestoreScriptPlaceholdersQuotedString Mechanism

This rule uses a special **placeholder restoration and validation** mechanism:

1. **During preprocessing**, all quoted strings in the script are replaced with placeholder tokens (e.g., `ScriptPlaceHolderQuotedString0000000005`). So by the time this rule runs, `.prop("disabled", true)` in the code appears as `.prop(ScriptPlaceHolderQuotedString0000000005, true)`.

2. **The regex matches** the placeholder token via the named capture group `RestorePlaceholderQuotedString`.

3. **After matching**, the engine restores the original quoted string content from the placeholder. The matched statement becomes `.GetElement("MMFUDS").prop("disabled", true)`.

4. **Post-restoration validation** applies a secondary regex: `\.GetElement\(.*?\)\.prop\((['"])disabled\1`
   - This confirms the first argument to `.prop()` is literally the string `"disabled"` (or `'disabled'`).
   - If validation fails (e.g., the prop was `"readonly"` or something else), the replacement is **skipped** and the original code is preserved.

5. **Only if validation passes** is the replacement applied.

**Before:**

```javascript
this.controller.GetElement("MMFUDS").prop("disabled", true);
this.controller.GetElement("MMSTAT").prop("disabled", true);
```

**After:**

```javascript
this.controller.GetElement("MMFUDS").readonly();
this.controller.GetElement("MMSTAT").readonly();
```

**Non-matching example (validation fails):**

```javascript
// This would NOT be replaced because the prop is "checked", not "disabled":
this.controller.GetElement("MMFUDS").prop("checked", true);
```

---

### 8.9 Rule 9: `.GetElement(f).prop("disabled", false)` → `.GetElement(f).enable()`

**Purpose:** Replace jQuery `.prop("disabled", false)` with the V2 UI `.enable()` method for enabling form fields.

**Regex:** `\.GetElement\((?<fieldName>.*?)\)\.prop\((?<RestorePlaceholderQuotedString>ScriptPlaceHolderQuotedString[0-9]{10}),[\s]*false[\s]*\)`
**Replacement:** `.GetElement({Replace_fieldName}).enable()`

#### RestoreScriptPlaceholdersQuotedString Mechanism

Identical mechanism to Rule 8:

1. Match the placeholder token in the regex
2. Restore the original quoted string after matching
3. Validate with: `\.GetElement\(.*?\)\.prop\((['"])disabled\1`
4. Only replace if the property is literally `"disabled"`

**Before:**

```javascript
this.controller.GetElement("MMSTAT").prop("disabled", false);
this.controller.GetElement("WRTEPY").prop("disabled", false);
```

**After:**

```javascript
this.controller.GetElement("MMSTAT").enable();
this.controller.GetElement("WRTEPY").enable();
```

**Non-matching example (validation fails):**

```javascript
// NOT replaced — prop is "hidden", not "disabled":
this.controller.GetElement("MMSTAT").prop("hidden", false);
```

**Order dependency:** Rules 8 and 9 must run before Rule 10 (standalone `.GetContentElement()` removal). Rules 8/9 target `.GetElement(f).prop(...)` which could theoretically also be preceded by `.GetContentElement()`, but Rule 4 has already stripped `.GetContentElement()` from `.readOnly` chains earlier. At this point, `.GetElement(f).prop(...)` stands alone.

---

### 8.10 Rule 10: `.GetContentElement().GetElement(f)` → `.GetElement(f)` (standalone)

**Purpose:** Remove the intermediate `.GetContentElement()` call for cases NOT followed by `.readOnly()` (those were already handled by Rule 4).

**Regex:** `\.GetContentElement\(\).GetElement\((?<fieldName>.*?)\)`
**Replacement:** `.GetElement({Replace_fieldName})`

**Before:**

```javascript
this.controller.GetContentElement().GetElement("WRTEPY").SetValue("test");
_this.controller.GetContentElement().GetElement("MMSTAT");
```

**After:**

```javascript
this.controller.GetElement("WRTEPY").SetValue("test");
_this.controller.GetElement("MMSTAT");
```

**Order dependency:** Must run AFTER Rules 4 and 5. By this point, all `.GetContentElement().GetElement(f).readOnly()` chains have been transformed to `.GetElement(f).readonly()`. What remains are `.GetContentElement().GetElement(f)` calls followed by other methods (`.SetValue()`, `.enable()`, etc.) or standing alone. This rule safely removes the remaining `.GetContentElement()` intermediaries.

---

### 8.11 Rules 11–13: DatePicker Cross-Statement Transformation

Rules 11, 12, and 13 work together using **cross-statement variable tracking** to identify `TextBoxElement` instances that should become `DatePickerElement` instances. The association is determined by detecting `.inforDateField()` usage on the jQuery element that wraps the text box element.

#### Overview of the Three-Step Process

1. **Rule 11** detects `.inforDateField(...)` and stores the jQuery element variable name
2. **Rule 12** finds the `.AddElement()` call that produces the same jQuery element, stores the element variable name
3. **Rule 13** finds the `new TextBoxElement` assignment for that element and replaces it with `new DatePickerElement`

#### Rule 11: `.inforDateField(...)` → *(remove statement)*

**Purpose:** Detect the jQuery `.inforDateField()` call, store the jQuery element variable name for cross-referencing, and remove the statement (V2 UI `DatePickerElement` handles date formatting natively).

**Regex:** `"(?<Store_datePickerJQueryElementVarName>[\$\w]+)\.inforDateField[\s]*\([\w\W]*\);{0,1}"`
**Replacement:** *(empty — statement is removed)*

The named capture group `Store_datePickerJQueryElementVarName` stores the jQuery element variable name (e.g., `datepicker`) into the variable tracking dictionary for use by subsequent rules.

**Before:**

```javascript
datepicker.inforDateField({ hasInitialValue: false, openOnEnter: false, dateFormat: "ddMMyy" });
```

**After:**

*(statement removed — the entire line is deleted)*

---

#### Rule 12: Store element variable name from `.AddElement()` call

**Purpose:** Find the `.AddElement()` call that assigns to the same jQuery element variable (identified in Rule 11), and store the element variable name passed to `.AddElement()`. This rule does NOT modify the code — it only captures variable associations.

**Regex:** `{Replace_datePickerJQueryElementVarName}[\s]*=[\s]*(?<contentElementStatement>[\w\.\(\)\s]+)\.AddElement[\s]*\((?<Store_datePickerElement>[\w]+)`
**Replacement:** `{Replace_datePickerJQueryElementVarName} = {Replace_contentElementStatement}.AddElement({Replace_datePickerElement}`

The `{Replace_datePickerJQueryElementVarName}` token is substituted with the variable name stored by Rule 11 (e.g., `datepicker`). The named capture group `Store_datePickerElement` captures and stores the element variable passed to `.AddElement()` (e.g., `textElement`).

**Note:** The replacement is identical to the match (preserves the code unchanged). The sole purpose is to store the `datePickerElement` variable name.

**Example match:**

```javascript
datepicker = this.controller.GetContentElement().AddElement(textElement);
```

After this rule runs, `textElement` is stored as `datePickerElement` for use by Rule 13.

---

#### Rule 13: `new TextBoxElement` → `new DatePickerElement`

**Purpose:** Replace the `TextBoxElement` constructor with `DatePickerElement` for the variable identified through Rules 11–12.

**Regex:** `{Replace_datePickerElement}[\s]*=[\s]*new[\s]+TextBoxElement`
**Replacement:** `{Replace_datePickerElement} = new DatePickerElement`

The `{Replace_datePickerElement}` token is substituted with the element variable name stored by Rule 12 (e.g., `textElement`).

**Before:**

```javascript
textElement = new TextBoxElement();
```

**After:**

```javascript
textElement = new DatePickerElement();
```

---

#### Complete DatePicker Example (Rules 11–13 Together)

**Before (Classic UI):**

```javascript
MyScript.prototype.initDateField = function() {
    var textElement = new TextBoxElement();
    textElement.Name = "ORDT";
    textElement.Position = { x: 1, y: 5 };

    var datepicker = this.controller.GetContentElement().AddElement(textElement);
    datepicker.inforDateField({ hasInitialValue: false, openOnEnter: false, dateFormat: "ddMMyy" });
};
```

**After (V2 UI — all three rules applied):**

```javascript
MyScript.prototype.initDateField = function() {
    var textElement = new DatePickerElement();
    textElement.Name = "ORDT";
    textElement.Position = { x: 1, y: 5 };

    var datepicker = this.controller.GetContentElement().AddElement(textElement);
};
```

**Transformation breakdown:**
1. **Rule 11:** `datepicker.inforDateField({...})` matched → `datepicker` stored as `datePickerJQueryElementVarName` → statement removed
2. **Rule 12:** `datepicker = this.controller.GetContentElement().AddElement(textElement)` matched (starts with `datepicker`) → `textElement` stored as `datePickerElement` → code unchanged
3. **Rule 13:** `textElement = new TextBoxElement` matched → replaced with `textElement = new DatePickerElement`

**Note:** Rule 10 (`.GetContentElement()` removal) may also apply to the `.AddElement()` line, transforming it to `datepicker = this.controller.AddElement(textElement)` — but this is independent of the DatePicker rules.

#### Failure Behavior

If cross-statement variable tracking fails (e.g., the `.inforDateField()` call exists but the corresponding `.AddElement()` or `new TextBoxElement` cannot be found):
- The `{Replace_...}` placeholders remain unresolved in the search pattern
- The regex simply does not match
- The original `TextBoxElement` code is **left unchanged** without any warning or flag
- This is by design (Requirement 6.9): fail silently rather than flag for manual review

---

### 8.12 Incompatible Statement Detection (Post-Transformation)

After all 13 replacement rules have been applied, the function content is scanned for remaining incompatible patterns that could not be automatically transformed.

| Remaining Pattern | Severity | Suggested V2 Equivalent |
|---|---|---|
| `.readOnly(` (capital O) | WARN | `.readonly()` |
| `.inforBusyIndicator(` | WARN | `ScriptUtil.getController().showBusyIndicator()` |

**Detection logic:** The incompatible statement search uses regex patterns from `IncompatibleStatementsList_FunctionContentStatements.csv`:

- **`.readOnly(`** — Regex: `.*(?<incompatibleStatement>\.readOnly[\s]*\().*` — Any remaining `.readOnly(` with optional whitespace before the parenthesis. This catches instances that escaped Rules 4 and 5 (e.g., unusual chaining patterns).
- **`.inforBusyIndicator(`** — Regex: `.*(?<incompatibleStatement>\.inforBusyIndicator[\s]*\().*` — Any remaining `.inforBusyIndicator(` call. This catches instances that escaped Rules 6 and 7 (e.g., non-jQuery invocations, atypical argument patterns).

For each detected incompatible statement:
1. The occurrence is logged with script name, function name, matched statement, and suggested replacement
2. The original code is preserved unchanged
3. A `// TODO [H5 Migration]:` comment is added above the statement in the Incompatible Statement Detection phase (Section 10)

---

### 8.13 Iterative Matching

Each rule is applied iteratively within the function body:

1. Start matching from position 0 in the function content
2. When a match is found, apply the replacement
3. Continue searching from the position after the last match's start index + match length
4. Repeat until no more matches are found for the current rule
5. Move to the next rule in CSV order

This ensures multiple occurrences of the same pattern within a single function are all transformed.

---

### 8.14 Variable Storage Across Rules

The function content migration uses a variable storage mechanism for cross-rule dependencies (used by Rules 11–13):

| Stored Variable | Source Rule | Consumed By |
|---|---|---|
| `datePickerJQueryElementVarName` | Rule 11 (`.inforDateField` detection) | Rule 12 (`.AddElement` matching) |
| `contentElementStatement` | Rule 12 (`.AddElement` matching) | Rule 12 replacement template |
| `datePickerElement` | Rule 12 (`.AddElement` matching) | Rule 13 (`TextBoxElement` replacement) |

If a consumed variable was never stored (e.g., Rule 13's pattern uses `{Replace_datePickerElement}` but Rule 12 never matched), the unresolved `{Replace_...}` token remains in the search regex, which simply won't match anything — the rule is effectively skipped without error.

Additionally, the `{Replace_fieldName}` capture from Rules 4, 8, 9, and 10 is resolved within the same rule execution (not cross-rule) — the captured group value is inserted directly into the replacement string on each match.

---

### 8.15 Edge Cases and Special Handling

| Scenario | Behavior |
|----------|----------|
| `userContext` appears inside a string literal | Protected by preprocessing placeholder — not matched |
| `.readOnly` in a comment | Protected by preprocessing placeholder — not matched |
| `.prop("readonly", true)` (not "disabled") | Rules 8/9 post-restoration validation fails — code preserved unchanged |
| `.prop("disabled", true)` without `.GetElement()` prefix | Regex requires `.GetElement(...)` prefix — not matched |
| Multiple `.inforBusyIndicator` calls in one function | Each is matched iteratively and replaced independently |
| `.inforDateField()` with no matching `.AddElement()` | Variable tracking fails silently; `TextBoxElement` left unchanged |
| `.GetContentElement().GetElement(f).readOnly()` | Handled entirely by Rule 4 (combined); Rule 5 and Rule 10 do NOT re-process it |
| Whitespace variations (`userContext ['prop']`, `ScriptDebugConsole.WriteLine (x)`) | Regexes include `[\s]*` to handle optional whitespace |
| `$` in variable names (e.g., `$datepicker`) | Properly captured by `[\$\w]+` in Rule 11 regex; escaped as `\$` in stored variables for use in subsequent regex patterns |

---

### 8.16 Source Cross-Reference

| Rule | CSV Source | Helper Script |
|------|-----------|---------------|
| Rule 1 (userContext bracket) | `ReplaceList_FunctionContent.csv` row 1 | `FunctionContentHelper.ps1` |
| Rule 2 (userContext dot) | `ReplaceList_FunctionContent.csv` row 2 | `FunctionContentHelper.ps1` |
| Rule 3 (ScriptDebugConsole) | `ReplaceList_FunctionContent.csv` row 3 | `FunctionContentHelper.ps1` |
| Rule 4 (GetContentElement+readOnly) | `ReplaceList_FunctionContent.csv` row 4 | `FunctionContentHelper.ps1` |
| Rule 5 (standalone readOnly) | `ReplaceList_FunctionContent.csv` row 5 | `FunctionContentHelper.ps1` |
| Rule 6 (inforBusyIndicator show) | `ReplaceList_FunctionContent.csv` row 6 | `FunctionContentHelper.ps1` |
| Rule 7 (inforBusyIndicator hide) | `ReplaceList_FunctionContent.csv` row 7 | `FunctionContentHelper.ps1` |
| Rule 8 (prop disabled true) | `ReplaceList_FunctionContent.csv` row 8 | `FunctionContentHelper.ps1` |
| Rule 9 (prop disabled false) | `ReplaceList_FunctionContent.csv` row 9 | `FunctionContentHelper.ps1` |
| Rule 10 (GetContentElement standalone) | `ReplaceList_FunctionContent.csv` row 10 | `FunctionContentHelper.ps1` |
| Rule 11 (inforDateField removal) | `ReplaceList_FunctionContent.csv` row 11 | `FunctionContentHelper.ps1` |
| Rule 12 (AddElement store) | `ReplaceList_FunctionContent.csv` row 12 | `FunctionContentHelper.ps1` |
| Rule 13 (TextBoxElement→DatePickerElement) | `ReplaceList_FunctionContent.csv` row 13 | `FunctionContentHelper.ps1` |
| Incompatible detection | `IncompatibleStatementsList_FunctionContentStatements.csv` | `FunctionContentHelper.ps1` (function `SearchForIncompatibleFunctionContentStatements`) |

## 9. Error Handling Patterns

This section summarizes the V2 UI error handling model for MI service calls. For the full implementation details (detection regex, transformation algorithm, variable extraction, and multiple concrete examples), see **section 4.3**.

### 9.1 Why Restructuring Is Needed

The Classic UI and V2 UI handle functional MI errors differently:

| Aspect | Classic UI | V2 UI |
|--------|-----------|-------|
| Functional MI errors (e.g., "Record does not exist") | Delivered to `.catch()` handler | Delivered to `.then()` handler via `response.errorMessage` |
| Technical/network errors (e.g., timeout, server unreachable) | Delivered to `.catch()` handler | Delivered to `.catch()` handler |
| `.catch()` purpose | Handles ALL errors (functional + technical) | Handles ONLY technical/network errors |

Because V2 UI routes functional MI errors to `.then()` instead of `.catch()`, any Classic UI script that relies on `.catch()` for business-level error handling will silently miss those errors after migration unless the `.then()` body is restructured.

### 9.2 The V2 Error Handling Model

**Key principles:**

1. **`response.errorMessage` is the sole mechanism** for detecting functional MI failures in the `.then()` handler
2. **`.catch()` is reserved exclusively** for technical/network errors — it should never contain business logic that handles "Record does not exist" or similar MI-level failures
3. **The `.catch()` handler is preserved unchanged** after restructuring — it remains as a safety net for genuine connection failures

### 9.3 Canonical If/Else Restructuring Pattern

When a Classic UI script has `.then(...)` and `.catch(...)` handlers on an MI call, the `.then()` body is restructured as:

```javascript
.then(function(response) {
    if(response.errorMessage) {
        // Error logic (moved from .catch body, with variable substitution)
    } else {
        // Success logic (original .then body)
    }
})
.catch(function(error) {
    // UNCHANGED — handles technical/network errors only
})
```

### 9.4 Variable Substitution Rules

When moving catch logic into the if-branch inside `.then()`:

- Replace all `<errorVarName>.` references with `<responseVarName>.` (dot-property access only)
- Example: if catch uses `error.errorMessage`, and then uses `response`, the moved logic becomes `response.errorMessage`
- Standalone variable references without dot access are NOT substituted
- If the catch handler already uses the same variable name as the then handler (e.g., both use `response`), the substitution is a no-op

### 9.5 Quick-Reference Before/After Example

**Before (Classic UI):**

```javascript
MIService.Current.executeRequest(myRequest).then(function(response) {
    processResults(response.items);
}).catch(function(error) {
    _this.log.Error(error.errorMessage);
});
```

**After (V2 UI):**

```javascript
MIService.executeRequest(myRequest).then(function(response) {
    if(response.errorMessage) {
        _this.log.Error(response.errorMessage);
    } else {
        processResults(response.items);
    }
}).catch(function(error) {
    _this.log.Error(error.errorMessage);
});
```

**What changed:**
1. `.Current` removed from `MIService.Current` (see section 4.4)
2. `.then()` body wrapped in `if(response.errorMessage) { ... } else { ... }`
3. Catch body copied into the if-branch with `error.` → `response.` substitution
4. Original then body moved into the else-branch
5. `.catch()` handler preserved unchanged

### 9.6 Applies To

This restructuring applies to both:
- `MIService.Current.executeRequest(request).then(...).catch(...)`
- `MIService.Current.execute(program, transaction, record, outputFields).then(...).catch(...)`

Both `function` keyword and arrow `=>` syntax are supported.

> **Full details:** See section 4.3 for detection regex patterns, the complete transformation algorithm, variable extraction logic, edge cases (same variable name in both handlers, conditional logic in catch), and additional before/after examples.

## 10. Incompatible Statement Detection

After all automated transformations have been applied (API, Grid, Message Dialog, Custom Dialog, Function Content), the migration engine performs a final detection pass to identify Classic UI statements that could not be automatically replaced. These are patterns that remain in the code post-transformation — they require manual developer intervention to complete the migration.

### 10.1 Overview

- **Total patterns:** 11 incompatible statement patterns across 4 categories
- **Severity:** All patterns use WARN level
- **Action:** Each detected pattern gets a dedicated `// TODO [H5 Migration]: <description>` comment inserted directly above the statement
- **Graceful degradation:** If any single action (flagging, commenting, or logging) fails for a given statement, processing continues with remaining incompatible statements without aborting the migration

### 10.2 TODO Comment Format

Every incompatible statement detected receives a comment in this exact format:

```
// TODO [H5 Migration]: <description of required manual action>
```

The comment is inserted on the line immediately above the incompatible statement. Each incompatible statement gets its own dedicated TODO comment, even when multiple incompatible statements appear consecutively or in the same function.

### 10.3 Detection Patterns by Category

All detection regexes use a named capture group `(?<incompatibleStatement>...)` to extract the specific incompatible portion from the full line match.

---

#### Category A: API Statements

**Source:** `IncompatibleStatementsList_APIStatements.csv`

##### Pattern 1: `ScriptUtil.ApiRequest(`

Detects remaining `ScriptUtil.ApiRequest` calls that could not be automatically parsed and transformed (e.g., dynamically-constructed URLs).

| Property | Value |
|----------|-------|
| **Regex** | `.*(?<incompatibleStatement>ScriptUtil\.ApiRequest[\s]*\().*` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace with MIService.executeV2(program, transaction, inputParameters)` |

**Before (detected):**
```javascript
ScriptUtil.ApiRequest(dynamicUrl, this.onSuccess.bind(this), this.onError.bind(this));
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace with MIService.executeV2(program, transaction, inputParameters)
ScriptUtil.ApiRequest(dynamicUrl, this.onSuccess.bind(this), this.onError.bind(this));
```

---

#### Category B: Grid Statements

**Source:** `IncompatibleStatementsList_GridStatements.csv`

##### Pattern 2: `.getLength(`

Detects remaining `.getLength()` calls on grid data that were not automatically replaced.

| Property | Value |
|----------|-------|
| **Regex** | `\.getLength[\s]*\(` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .getLength() with .length` |

**Before (detected):**
```javascript
var count = grid.getData().getLength();
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .getLength() with .length
var count = grid.getData().getLength();
```

##### Pattern 3: `.getDataItem(`

Detects remaining `.getDataItem()` calls that were not automatically replaced.

| Property | Value |
|----------|-------|
| **Regex** | `.*(?<incompatibleStatement>\.getDataItem[\s]*\().*` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .getDataItem(index) with .getData()[index]` |

**Before (detected):**
```javascript
var row = someGrid.getDataItem(selectedIndex);
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .getDataItem(index) with .getData()[index]
var row = someGrid.getDataItem(selectedIndex);
```

##### Pattern 4: `.getSelectedRows()`

Detects remaining `.getSelectedRows()` calls that were not automatically replaced.

| Property | Value |
|----------|-------|
| **Regex** | `\.getSelectedRows[\s]*\([\s]*\)` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .getSelectedRows() with .getSelectedGridRows()` |

**Before (detected):**
```javascript
var rows = grid.getSelectedRows();
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .getSelectedRows() with .getSelectedGridRows()
var rows = grid.getSelectedRows();
```

##### Pattern 5: `.getColumnIndex(`

Detects remaining `.getColumnIndex()` calls that were not automatically replaced.

| Property | Value |
|----------|-------|
| **Regex** | `\.getColumnIndex[\s]*\(` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .getColumnIndex() with .getColumns() lookup` |

**Before (detected):**
```javascript
var idx = grid.getColumnIndex(grid.getColumns().find(c => c.colFld == 'MMITDS').id);
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .getColumnIndex() with .getColumns() lookup
var idx = grid.getColumnIndex(grid.getColumns().find(c => c.colFld == 'MMITDS').id);
```

##### Pattern 6: `.deleteItem(`

Detects remaining `.deleteItem()` calls that were not automatically replaced.

| Property | Value |
|----------|-------|
| **Regex** | `\.deleteItem[\s]*\(` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .deleteItem(rowId) with .removeRow(rowId)` |

**Before (detected):**
```javascript
grid.getData().deleteItem(rowId);
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .deleteItem(rowId) with .removeRow(rowId)
grid.getData().deleteItem(rowId);
```

---

#### Category C: Custom Dialog Statements

**Source:** `IncompatibleStatementsList_CustomDialogStatements.csv`

##### Pattern 7: `.inforMessageDialog(`

Detects remaining `.inforMessageDialog()` calls that were not automatically replaced with `H5ControlUtil.H5Dialog.CreateDialogElement`.

| Property | Value |
|----------|-------|
| **Regex** | `.*(?<incompatibleStatement>\.inforMessageDialog[\s]*\().*` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .inforMessageDialog() with H5ControlUtil.H5Dialog.CreateDialogElement(content[0], options)` |

**Before (detected):**
```javascript
dialogContent.inforMessageDialog(dialogOptions);
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .inforMessageDialog() with H5ControlUtil.H5Dialog.CreateDialogElement(content[0], options)
dialogContent.inforMessageDialog(dialogOptions);
```

##### Pattern 8: `H5ControlUtil.H5Dialog.DestroyDialog(...)`

Detects remaining `DestroyDialog` calls that were not automatically replaced with `model.close(true)`.

| Property | Value |
|----------|-------|
| **Regex** | `.*(?<incompatibleStatement>H5ControlUtil\.H5Dialog\.DestroyDialog[\s]*\(.*\)).*` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace H5ControlUtil.H5Dialog.DestroyDialog(...) with model.close(true)` |

**Before (detected):**
```javascript
H5ControlUtil.H5Dialog.DestroyDialog($(SheetDialog));
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace H5ControlUtil.H5Dialog.DestroyDialog(...) with model.close(true)
H5ControlUtil.H5Dialog.DestroyDialog($(SheetDialog));
```

##### Pattern 9: `.ListConfig(`

Detects remaining `.ListConfig()` calls that were not automatically removed (e.g., when the associated `.inforDataGrid()` could not be resolved).

| Property | Value |
|----------|-------|
| **Regex** | `.*(?<incompatibleStatement>\.ListConfig[\s]*\().*` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Remove .ListConfig() call - store columns and data for use with .datagrid()` |

**Before (detected):**
```javascript
options = Configuration.Current.ListConfig('id', columns, data);
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Remove .ListConfig() call - store columns and data for use with .datagrid()
options = Configuration.Current.ListConfig('id', columns, data);
```

##### Pattern 10: `.inforDataGrid(`

Detects remaining `.inforDataGrid()` calls that were not automatically replaced with `.datagrid()`.

| Property | Value |
|----------|-------|
| **Regex** | `.*(?<incompatibleStatement>\.inforDataGrid[\s]*\().*` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .inforDataGrid(options) with .datagrid({columns: columns, dataset: data})` |

**Before (detected):**
```javascript
grid = $("#showErrorsGrid").inforDataGrid(options);
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .inforDataGrid(options) with .datagrid({columns: columns, dataset: data})
grid = $("#showErrorsGrid").inforDataGrid(options);
```

---

#### Category D: Function Content Statements

**Source:** `IncompatibleStatementsList_FunctionContentStatements.csv`

##### Pattern 11: `.readOnly(` (capital O)

Detects remaining `.readOnly()` calls (with capital O) that were not automatically lowercased to `.readonly()`. This typically occurs when the pattern appears in an unexpected context that the automated replacement did not match.

| Property | Value |
|----------|-------|
| **Regex** | `.*(?<incompatibleStatement>\.readOnly[\s]*\().*` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .readOnly() with .readonly() (lowercase 'o')` |

**Before (detected):**
```javascript
_this.controller.GetContentElement().GetElement("WRTEPY").readOnly();
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .readOnly() with .readonly() (lowercase 'o')
_this.controller.GetContentElement().GetElement("WRTEPY").readOnly();
```

##### Pattern 12: `.inforBusyIndicator(` (bonus — 12th pattern from CSV)

Detects remaining `.inforBusyIndicator()` calls that were not automatically replaced with `ScriptUtil.getController().showBusyIndicator()` or `hideBusyIndicator()`.

| Property | Value |
|----------|-------|
| **Regex** | `.*(?<incompatibleStatement>\.inforBusyIndicator[\s]*\().*` |
| **Severity** | WARN |
| **TODO comment** | `// TODO [H5 Migration]: Replace .inforBusyIndicator() with ScriptUtil.getController().showBusyIndicator() or .hideBusyIndicator()` |

**Before (detected):**
```javascript
$("body").inforBusyIndicator({ delay: 100, modal: true });
```

**After (with TODO comment inserted):**
```javascript
// TODO [H5 Migration]: Replace .inforBusyIndicator() with ScriptUtil.getController().showBusyIndicator() or .hideBusyIndicator()
$("body").inforBusyIndicator({ delay: 100, modal: true });
```

---

### 10.4 Detection Processing Rules

#### Execution Timing

Incompatible statement detection runs as the **final phase** (phase 6) of the transformation pipeline, after all automated replacements have been attempted. This ensures only truly unresolved patterns are flagged.

#### Per-Statement Independence

Each of the 11 patterns (plus the bonus `.getColumnIndex` pattern for 12 total from the Grid CSV) is detected independently. Detection of one pattern does not affect detection of others.

#### Multiple Occurrences

If the same pattern appears multiple times in a script, each occurrence receives its own dedicated TODO comment on the line immediately above it.

**Example with multiple occurrences:**
```javascript
// TODO [H5 Migration]: Replace .getSelectedRows() with .getSelectedGridRows()
var rows1 = gridA.getSelectedRows();
// other code...
// TODO [H5 Migration]: Replace .getSelectedRows() with .getSelectedGridRows()
var rows2 = gridB.getSelectedRows();
```

### 10.5 Graceful Degradation

For each incompatible statement detected, the migration engine performs three actions:

1. **Flag** — Mark the statement internally at WARN severity level
2. **Comment** — Insert the `// TODO [H5 Migration]: ...` comment above the statement
3. **Log** — Log the occurrence (script name, function name, line content)

**If any single action fails** (e.g., comment insertion fails due to line indexing issues, or logging encounters an I/O error):

- The migration engine **continues processing** remaining incompatible statements
- The failure does not abort the overall migration
- Other actions for the same statement that did not fail are still applied
- Subsequent incompatible statements in the same file are still processed

This ensures maximum detection coverage even in the presence of individual failures.

### 10.6 Summary Table

| # | Pattern | Category | Regex | TODO Comment |
|---|---------|----------|-------|--------------|
| 1 | `ScriptUtil.ApiRequest(` | API | `.*(?<incompatibleStatement>ScriptUtil\.ApiRequest[\s]*\().*` | Replace with MIService.executeV2(program, transaction, inputParameters) |
| 2 | `.getLength(` | Grid | `\.getLength[\s]*\(` | Replace .getLength() with .length |
| 3 | `.getDataItem(` | Grid | `.*(?<incompatibleStatement>\.getDataItem[\s]*\().*` | Replace .getDataItem(index) with .getData()[index] |
| 4 | `.getSelectedRows()` | Grid | `\.getSelectedRows[\s]*\([\s]*\)` | Replace .getSelectedRows() with .getSelectedGridRows() |
| 5 | `.getColumnIndex(` | Grid | `\.getColumnIndex[\s]*\(` | Replace .getColumnIndex() with .getColumns() lookup |
| 6 | `.deleteItem(` | Grid | `\.deleteItem[\s]*\(` | Replace .deleteItem(rowId) with .removeRow(rowId) |
| 7 | `.inforMessageDialog(` | Custom Dialog | `.*(?<incompatibleStatement>\.inforMessageDialog[\s]*\().*` | Replace .inforMessageDialog() with H5ControlUtil.H5Dialog.CreateDialogElement(content[0], options) |
| 8 | `H5ControlUtil.H5Dialog.DestroyDialog(...)` | Custom Dialog | `.*(?<incompatibleStatement>H5ControlUtil\.H5Dialog\.DestroyDialog[\s]*\(.*\)).*` | Replace H5ControlUtil.H5Dialog.DestroyDialog(...) with model.close(true) |
| 9 | `.ListConfig(` | Custom Dialog | `.*(?<incompatibleStatement>\.ListConfig[\s]*\().*` | Remove .ListConfig() call - store columns and data for use with .datagrid() |
| 10 | `.inforDataGrid(` | Custom Dialog | `.*(?<incompatibleStatement>\.inforDataGrid[\s]*\().*` | Replace .inforDataGrid(options) with .datagrid({columns: columns, dataset: data}) |
| 11 | `.readOnly(` (capital O) | Function Content | `.*(?<incompatibleStatement>\.readOnly[\s]*\().*` | Replace .readOnly() with .readonly() (lowercase 'o') |
| 12 | `.inforBusyIndicator(` | Function Content | `.*(?<incompatibleStatement>\.inforBusyIndicator[\s]*\().*` | Replace .inforBusyIndicator() with ScriptUtil.getController().showBusyIndicator() or .hideBusyIndicator() |


## 11. Transformation Order and Function-Level Scoping

This section documents the fixed dependency order of migration phases, the function-level scoping model, and how class structure parsing enables cross-function awareness.

### 11.1 Fixed Phase Order

All migration phases MUST be applied in the following exact dependency order within each function body. Reordering these phases will produce incorrect results because later phases depend on transformations completed by earlier phases.

| Phase | Name | Description |
|-------|------|-------------|
| 1 | API Transaction Migration | URL parsing → execute statement → response reading → MI request execution → general replacements |
| 2 | Grid Statement Migration | All 17 rules in order with variable tracking |
| 3 | Message Dialog Migration | ShowDialog → ConfirmDialog |
| 4 | Custom Dialog Migration | inforMessageDialog → CreateDialogElement, open handler relocation, click params, DestroyDialog, ListConfig/inforDataGrid, this→self |
| 5 | Function Content Migration | All 13 rules in order |
| 6 | Incompatible Statement Detection | Post-transformation scan for all 12 patterns |

#### Phase Dependency Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Per-Function Transformation Pipeline              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 1: API Transaction         ──┐                               │
│  (ScriptUtil.ApiRequest → MIService)│                               │
│                                     │                               │
│  Phase 2: Grid Statements         ──┤  Phases 1-5 are              │
│  (.getSelectedRows → etc.)          │  replacement phases           │
│                                     │  (modify code)                │
│  Phase 3: Message Dialogs         ──┤                               │
│  (ShowDialog → ConfirmDialog)       │                               │
│                                     │                               │
│  Phase 4: Custom Dialogs          ──┤                               │
│  (.inforMessageDialog → etc.)       │                               │
│                                     │                               │
│  Phase 5: Function Content        ──┘                               │
│  (readOnly, busyIndicator, etc.)                                    │
│                                                                     │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                                                                     │
│  Phase 6: Incompatible Detection    (detection-only phase;          │
│  (scan for unresolved patterns)      flags remaining patterns)      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Why This Order Matters

| Dependency | Explanation |
|---|---|
| Phase 1 before Phase 5 | API replacements change `.MIRecord` to `.items` and `result.Message` to `result.errorMessage`. These are API-specific patterns that Phase 5 (Function Content) does not target. If Phase 5 ran first, it would not recognize or transform these patterns, leaving them for Phase 1 anyway — but ordering Phase 1 first ensures the code is in its final form before later phases scan it. |
| Phase 2 before Phase 6 | Grid rules convert `.getSelectedRows()` → `.getSelectedGridRows()`, `.getDataItem(i)` → `.getData()[i]`, etc. Phase 6 (Incompatible Detection) scans for these same Classic patterns. If detection ran before grid replacement, every grid statement would be flagged as incompatible even though it has an automated replacement. Running Phase 2 first ensures only truly unresolved patterns are flagged. |
| Phase 3 before Phase 4 | Message dialog migration (`ShowDialog`) runs before custom dialog migration (`inforMessageDialog`). The custom dialog phase also handles `<controllerVar>.Controller.ShowDialog(args)` patterns, and running message dialogs first ensures no ambiguity about which ShowDialog pattern applies. |
| Phase 4 before Phase 5 | Custom dialog migration inserts `const self = this;` and changes `.inforDataGrid` to `.datagrid`. Phase 5 would not recognize these custom dialog patterns — they are structurally different from function content patterns. Running Phase 4 first ensures the dialog code is fully migrated before function-level content rules apply. |
| Phase 5 before Phase 6 | Function content rules convert `.readOnly()` → `.readonly()` and `.inforBusyIndicator()` → `showBusyIndicator()`/`hideBusyIndicator()`. Phase 6 detects remaining `.readOnly()` and `.inforBusyIndicator()` calls. If detection ran before function content replacement, every occurrence would be flagged even when an automated rule exists. |
| Phase 6 is always last | Incompatible statement detection is a read-only scan that inserts TODO comments above unresolved patterns. It MUST run after all replacement phases so it only flags patterns that genuinely have no automated resolution. |

### 11.2 Function-Level Transformation Scoping

Transformations are NOT applied globally to the entire script content. Instead, the script is decomposed into individual functions, and each function is transformed independently with its own isolated state.

#### Why Function-Level Scoping

Variable tracking is a key component of several migration rules (especially Grid and Custom Dialog phases). For example:
- Grid rules track the variable name assigned from `controller.GetGrid()` to resolve subsequent grid operations
- Custom dialog rules track whether a `this`-capturing variable (`var self = this`) already exists
- Function content rules track `TextBoxElement` variable associations for `DatePickerElement` conversion

If these tracking variables persisted across function boundaries, a grid variable captured in `functionA` could incorrectly influence replacements in `functionB`, producing invalid code.

#### Scoping Mechanism

For each function extracted from the class:

1. **Initialize fresh tracking state** — all variable trackers (`gridVarName`, `selectedRowsVar`, `rowVarName`, `thisCapturingVar`, etc.) are reset to empty/null
2. **Apply Phase 1** (API Transaction) to the function body
3. **Apply Phase 2** (Grid Statements) to the function body — grid variable tracking operates within this function only
4. **Apply Phase 3** (Message Dialogs) to the function body
5. **Apply Phase 4** (Custom Dialogs) to the function body — `this`-capturing variable detection is scoped to this function
6. **Apply Phase 5** (Function Content) to the function body — `TextBoxElement`/`DatePickerElement` tracking is scoped to this function
7. **Apply Phase 6** (Incompatible Detection) to the function body
8. **Replace** the original function body in the class with the transformed result

After processing all functions, the updated class body replaces the original class body in the full script content.

#### What Resets Per Function

| Tracking Variable | Used By | Description |
|---|---|---|
| `gridVarName` | Phase 2 (Grid) | The variable holding the grid reference (e.g., from `controller.GetGrid()`) |
| `selectedRowsVar` | Phase 2 (Grid) | The variable holding `.getSelectedGridRows()` result |
| `rowVarName` | Phase 2 (Grid) | The variable used for row iteration index |
| `thisCapturingVar` | Phase 4 (Custom Dialog) | Whether `var self = this` / `var _this = this` already exists |
| `textBoxElementVar` | Phase 5 (Function Content) | Variable associated with `new TextBoxElement()` for DatePicker conversion |
| `listConfigData` | Phase 4 (Custom Dialog) | Stored columns/data from `Configuration.Current.ListConfig()` for `.datagrid()` replacement |

### 11.3 Class Structure Parsing

Before function-level transformation can begin, the migration engine must parse the script to identify the class body and extract all functions.

#### Step 1: Identify the Class Body (IIFE Pattern)

H5 scripts use an Immediately Invoked Function Expression (IIFE) to define the script class:

```javascript
var ClassName = (function() {
    // constructor, Init, prototype methods
    return ClassName;
}());
```

**Detection regex:**
```regex
var[\s]+(?<className>[\w]+)[\s]*=[\s]*(?:ScriptPlaceHolderCommentBlock[\d]+){0,1}[\s]*\([\s]*function[\s]*\(\)[\s]*{
```

This matches the IIFE opening pattern, optionally preceded by a block comment placeholder (since comments are already replaced with placeholders at this point).

**Class boundary determination:**
- Start at the match position
- Count opening `{` and closing `}` curly braces character by character
- When the open count equals the close count (and both > 0), the class body ends
- Extract the full content between start and end positions (inclusive of outer braces)

If the script does not match the IIFE pattern (e.g., it uses a different module format), the entire script content is treated as the class body.

#### Step 2: Extract Functions from the Class Body

For JavaScript (`.js`) files, two regex patterns identify function definitions:

**Pattern 1 — Prototype/instance method assignments:**
```regex
\.(?:prototype\.){0,1}(?<functionName>[\w]+)[\s]*=[\s]*function[\s]*\([\w,\s$]*\)[\s=]*{
```

This matches:
- `ClassName.prototype.run = function() {`
- `ClassName.prototype.onSuccess = function(result) {`
- `.Init = function(args) {`

**Pattern 2 — Standalone function declarations:**
```regex
function[\s]+(?<functionName>[\w]+)[\s]*\([\w,\s$]*\)[\s]*{
```

This matches:
- `function MyScript(scriptArgs) {` (constructor)
- `function checkResult(result) {`
- `function onError(e, msg) {`

For TypeScript (`.ts`) files, a single combined pattern handles all method formats:
```regex
(public|private){0,1}[\s]*(static[\s]*){0,1}(?<functionName>[\w_]+)[\s]*\([\w,\s$:]*\)(:(?<functionReturnType>[\s\w\[\]\(\)]+)){0,1}[\s]*{
```

#### Step 3: Determine Function Body Boundaries

For each matched function initializer:

1. Start at the match position in the class content
2. Walk character by character, counting `{` and `}` braces
3. When open count equals close count (both > 0), the function body ends
4. Extract the full function content (from initializer through closing brace)

**Filtering:** Functions named `function` or `catch` (false positives from anonymous functions or try/catch blocks) are skipped.

#### Step 4: Build the Function List

The extraction produces a list of `ScriptFunction` objects, each containing:

| Property | Description |
|---|---|
| `functionInitializer` | The full matched initializer line (e.g., `.prototype.run = function() {`) |
| `functionName` | The extracted function name (e.g., `run`, `Init`, `onSuccess`) |
| `functionContent` | The complete function body including initializer and closing brace |

This list represents ALL functions in the class, including:
- The constructor (e.g., `function MyScript(scriptArgs) { ... }`)
- The static `Init` method (e.g., `.Init = function(args) { ... }`)
- All prototype methods (e.g., `.prototype.run = function() { ... }`)

### 11.4 Cross-Function Knowledge

While most transformation state is scoped to individual functions, one specific rule requires class-level awareness: the **custom dialog `this.` → `self.` replacement** in click handlers.

#### The Problem

Inside a dialog's `click: function(event, model) { ... }` handler, `this` refers to the dialog context, not the script class. Any calls to class methods (e.g., `this.saveRecord()`) will fail because `this` no longer points to the class instance.

The migration engine replaces `this.functionName()` with `self.functionName()` (where `self` is a variable capturing the class `this` before the dialog call). However, it must only replace calls that are actual class methods — not arbitrary property accesses or unrelated function calls.

#### How the Function Name List Enables This

1. **During class parsing** (Step 4 above), the full list of function names is built from ALL functions in the class
2. **This list is passed to the Custom Dialog migration phase** (Phase 4) for each function being processed
3. **Inside the click handler**, the engine iterates through ALL function names from the class (excluding the current function's own initializer)
4. **For each class function name**, it searches the click handler body for calls matching `this.<functionName>(` or just `<functionName>(`
5. **Matches are replaced** with `<selfVarName>.<functionName>(` where `selfVarName` is the `this`-capturing variable (e.g., `self`, `_this`)

#### Matching Regex (per function name)

For each `functionNameOtherFunction` in the class function list:

```regex
\s(?<thisStatement>this\.){0,1}(?<functionName>functionNameOtherFunction)[\s]*\(
```

This matches:
- `this.saveRecord(` — explicit `this.` prefix (most common)
- ` saveRecord(` — bare function call without prefix (also captured and prefixed with `self.`)

#### What Gets Skipped

- The current function's own initializer is excluded from the search (a function does not prefix calls to itself with `self.`)
- Function names that do NOT exist in the class-level function list are NOT candidates for replacement (prevents incorrectly prefixing calls to external functions, DOM methods, or library functions)

#### Example

Given a class with functions: `run`, `loadData`, `saveRecord`, `onError`

Inside `run`'s click handler:
```javascript
// Before
click: function(event, model) {
    this.loadData();        // ← class method, WILL be replaced
    this.saveRecord();      // ← class method, WILL be replaced
    this.onError(e);        // ← class method, WILL be replaced
    this.toString();        // ← NOT a class method, will NOT be replaced
    console.log("done");   // ← not a function call pattern, ignored
}
```

```javascript
// After
click: function(event, model) {
    self.loadData();
    self.saveRecord();
    self.onError(e);
    this.toString();        // ← unchanged (not in class function list)
    console.log("done");
}
```

### 11.5 Complete Transformation Flow Summary

The following pseudocode summarizes the entire transformation pipeline:

```
1. Read script file content
2. PREPROCESS: Replace quoted strings → block comments → line comments with placeholders
3. PARSE CLASS: Identify IIFE class body using regex + brace counting
4. EXTRACT FUNCTIONS: Build function list from class body (all prototype methods, Init, constructor)
5. FOR EACH function in the function list:
     a. Reset all variable tracking state
     b. Apply Phase 1: API Transaction Migration (to function body)
     c. Apply Phase 2: Grid Statement Migration (to function body, with fresh grid var tracking)
     d. Apply Phase 3: Message Dialog Migration (to function body)
     e. Apply Phase 4: Custom Dialog Migration (to function body, passing FULL function list for this→self)
     f. Apply Phase 5: Function Content Migration (to function body)
     g. Apply Phase 6: Incompatible Statement Detection (to function body)
     h. Replace original function content with transformed content in class body
6. Replace original class body with transformed class body in script content
7. POST-PROCESS: Restore comment line placeholders → comment block placeholders → quoted string placeholders
8. Write transformed script to output file
```


## 12. Output Quality Guidelines

This section documents the quality guarantees that the migration engine provides for transformed output. These principles ensure the migrated script is immediately usable, searchable, and maintainable.

### Summary

| # | Principle | Mechanism | Reference |
|---|-----------|-----------|-----------|
| 1 | Comment preservation | Placeholder protection during transformation | Section 2 (Preprocessing) |
| 2 | Indentation preservation | Leading whitespace capture from original statements | Regex `prefixSpaces` capture |
| 3 | String literal protection | Placeholder substitution before any rules execute | Section 2 (Preprocessing) |
| 4 | V2 UI APIs only | All replacement templates reference `h5.script.d.ts` definitions | Section 3 (Script Structure Reference) |
| 5 | Consistent TODO format | `// TODO [H5 Migration]: <description>` | Section 10 (Incompatible Detection) |
| 6 | Incompatible code preserved | Original statement retained unchanged with TODO flag above | Section 10 (Incompatible Detection) |

---

### 12.1 Comment Preservation

**Requirement:** Single-line comments (`// ...`) and block comments (`/* ... */`) are retained in the output unchanged.

**How it works:**

1. During preprocessing (Section 2), all comments are replaced with unique placeholder tokens before any transformation rules execute
2. Transformation rules operate on a version of the code where comments are opaque placeholders — they cannot match or modify comment content
3. After all transformations complete, placeholders are restored to the original comment text verbatim
4. Comments that happen to contain Classic UI patterns (e.g., `// Old approach: ScriptUtil.ApiRequest(...)`) are NOT modified — the placeholder mechanism protects them entirely

**Result:** Every comment in the original script appears in the output exactly as written, in its original position relative to surrounding code.

---

### 12.2 Indentation Preservation

**Requirement:** Non-transformed lines retain their original indentation exactly. Transformed lines maintain the indentation level of the original statement they replaced.

**How it works:**

1. **Non-transformed lines** — Lines that do not match any replacement rule pass through the pipeline unchanged, including all leading whitespace
2. **Transformed lines** — When a regex matches a statement for replacement, the `prefixSpaces` variable (captured from the regex match) preserves the leading whitespace context of the original line
3. **Multi-line replacements** — When a single Classic UI statement expands into multiple V2 UI lines (e.g., `MIRequest` object construction, `if/else` error handling restructuring), the generated lines use:
   - The indentation prefix from the original statement's first line as the base level
   - Additional tab levels for nested content within the generated block (e.g., properties inside `miRequest = { ... }`, body inside `if(...) { ... }`)
4. **Alignment consistency** — All generated replacement code aligns with the surrounding code context, maintaining the visual structure of the script

**Example:**

```javascript
// Original (4-space indent):
    ScriptUtil.ApiRequest(url, this.onSuccess, this.onError);

// After transformation (4-space base indent preserved, nested content indented further):
    var miRequest = new MIRequest();
    miRequest.program = "MMS200MI";
    miRequest.transaction = "Get";
    miRequest.record = {
        ITNO: itemNumber
    }
    MIService.executeRequestV2(miRequest)
        .then((result) => { this.onSuccess(result); })
        .catch((error) => { this.onError('Error', error.error?.terminationReason ?? error.message ?? error); })
```

---

### 12.3 String Literal Protection

**Requirement:** All string literals are protected from transformation rules during pattern matching. Original string content is restored verbatim after all transformations complete.

**How it works:**

1. During preprocessing (Section 2), all single-quoted (`'...'`) and double-quoted (`"..."`) string literals are replaced with unique placeholder tokens (e.g., `ScriptPlaceHolderQuotedString0000000001`)
2. Transformation rules never see or operate on actual string content — they only see placeholder tokens
3. After all transformations complete, placeholders are restored to the original string values
4. This prevents false matches on:
   - URL patterns inside strings (e.g., `"/execute/MMS200MI/Get"` would falsely trigger API URL parsing if not protected)
   - Method names inside strings (e.g., `"getDataItem"` in a log message would falsely trigger grid replacement)
   - Keywords inside strings (e.g., `"readOnly"` in a comment string would falsely trigger function content replacement)

**Protection guarantee:** No string literal content is ever modified, corrupted, or removed by transformation rules, regardless of what text the string contains.

---

### 12.4 V2 UI API References Only

**Requirement:** All generated replacement code references V2 UI APIs as defined in `h5.script.d.ts`. No Classic UI APIs appear in replacement templates.

**What this means:**

- Every `MIRequest` object, `MIService.executeRequestV2()` call, and `ConfirmDialog.ShowMessageDialog()` invocation in replacement templates conforms to the type definitions in `h5.script.d.ts`
- No replacement template generates `ScriptUtil.ApiRequest`, `MIService.Current`, `.inforMessageDialog`, `.inforBusyIndicator`, or any other Classic UI API call
- The API reference in Section 3 documents the authoritative V2 UI API surface that all replacements target
- Developers can verify any generated code against the type definitions to confirm correctness

**Scope:** This guarantee applies only to **generated replacement code**. Original statements that are flagged as incompatible and preserved unchanged (see 12.6) will still contain Classic UI APIs — this is intentional, as they require manual migration.

---

### 12.5 Consistent TODO Comment Format

**Requirement:** All TODO comments inserted by the migration engine follow a single, consistent format that is searchable and filterable in any IDE.

**Exact format:**

```
// TODO [H5 Migration]: <description of required manual action>
```

**Properties:**

- The `[H5 Migration]` tag makes these comments distinguishable from other TODO comments in the codebase
- Searchable in any IDE using: `TODO [H5 Migration]`
- Each incompatible statement gets its own dedicated TODO comment on the line immediately above it
- TODO descriptions are actionable — they state what the developer should replace and what the V2 equivalent is

**Examples:**

```javascript
// TODO [H5 Migration]: Replace with MIService.executeV2(program, transaction, inputParameters)
ScriptUtil.ApiRequest(dynamicUrl, this.onSuccess.bind(this), this.onError.bind(this));

// TODO [H5 Migration]: Replace .getDataItem(index) with .getData()[index]
var row = someGrid.getDataItem(selectedIndex);

// TODO [H5 Migration]: Replace .inforMessageDialog() with H5ControlUtil.H5Dialog.CreateDialogElement(content[0], options)
dialogContent.inforMessageDialog(dialogOptions);
```

---

### 12.6 Incompatible Statement Preservation

**Requirement:** When a statement is flagged as incompatible, the original code is preserved unchanged. Only a TODO comment is added above it.

**Guarantees:**

1. The original line of code is **never modified** — not a single character is changed
2. Only a `// TODO [H5 Migration]: ...` comment is inserted on the line above
3. The code remains compilable/runnable (even if functionally incorrect in V2 UI context)
4. The developer can review each TODO and make informed decisions about the replacement

**Why this matters:**

- Preserving original code allows developers to see exactly what needs changing
- The script can still be loaded and partially tested even with incompatible statements present
- Developers maintain full control over how and when to address each flagged statement
- No information is lost — the original Classic UI pattern is visible alongside the suggested V2 replacement in the TODO comment

**Example showing preservation:**

```javascript
// Original Classic UI code (4 lines):
var apiUrl = buildUrl(program);
ScriptUtil.ApiRequest(apiUrl, onSuccess, onError);
var rows = grid.getSelectedRows();
dialogContent.inforMessageDialog(options);

// After migration (original lines preserved, TODOs inserted above each):
var apiUrl = buildUrl(program);
// TODO [H5 Migration]: Replace with MIService.executeV2(program, transaction, inputParameters)
ScriptUtil.ApiRequest(apiUrl, onSuccess, onError);
// TODO [H5 Migration]: Replace .getSelectedRows() with .getSelectedGridRows()
var rows = grid.getSelectedRows();
// TODO [H5 Migration]: Replace .inforMessageDialog() with H5ControlUtil.H5Dialog.CreateDialogElement(content[0], options)
dialogContent.inforMessageDialog(options);
```
