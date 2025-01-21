## Form Builder

### Design Document

This document outlines the design for a system that allows users to create `Forms` with various input types and conditional logic. The system is designed to handle scenarios where the same set of `Columns` needs to be repeated across different `Rows` (e.g., components in a device), allowing for individual data capture for each `Row`.

### Core Concepts

The system uses tables of **Column**, **Row**, **Cell**, **Layout**, **Condition**, and **Form**. **Columns** represent individual input fields. **Rows** represent components in a device or anything requiring repeated data entry for the Columns. **Cells** store the user inputs for each Column in a Row. **Layouts** define the structure and organization of widgets within a form. These Layouts can be nested, and their visibility can be controlled based on **Conditions**. **Forms** contain Rows and Cells.

### Credits

```

Copyright 2025 DWJ
Distributed under the Boost Software License, Version 1.0.
https://www.boost.org/LICENSE_1_0.txt

Modifications:
Works in React Native Expo - native (Android/iOS) and web.
Works with a relational database. Columns, Rows, Cells, Layouts, Conditions, and Forms can use foreign key constraints and keep referential integrity.
Form layout can be inverted/sorted by Row or by Column.
New Cell type which can store Column inputs for multiple Rows in a Form.
New Condition type which can affect Layouts and have sub-Conditions.
New/modified Columns for holding data (checkbox, star, dropdown, number, text, bigtext, signature, gps, file, document, photo, video, audio, list, api).
New/modified Layouts for structuring the form (input, description, picture, separator, group, accordion, tab).
New TypeScript types and React helper hooks and components.
https://github.com/dwidge/form-builder

```

```

Copyright (c) 2019 EclipseSource Munich
The JSON Forms project is licensed under the MIT License.
https://github.com/eclipsesource/jsonforms

```

### Column Types

The system supports the following `Column` types:

- **`checkbox`**: A 2 or 3 state checkbox.
- **`star`**: Select from a predefined list of icons and a predefined list of reasons in a dropdown.
- **`dropdown`**: Select from a predefined list of options.
- **`number`**: Input a numerical value.
- **`text`**: A single-line text input field.
- **`bigtext`**: A multi-line text input field.
- **`signature`**: Captures the user's digital signature.
- **`gps`**: Captures the user's geographical coordinates.
- **`api`**: Allows fetching from and submitting to an external API endpoint using a predefined JSON schema.

### Layout Types

The `Layout` table defines the structure and organization of the form. Here are the different layout types:

- **`input`**: Represents a linked `Column`. This layout type is used for input fields that the user submits. It is currently the only layout type with a non-null `ColumnId`.
- **`group`**: A container layout used to visually group other layouts. It indents its children, surrounds them with a box, and displays its optional name as a bold header.
- **`accordion`**: Similar to `group`. It collapses its child layouts under each one's name, expanding only one child at a time.
- **`tab`**: Similar to `accordion`. Displays child layout names in a row above, with the current name highlighted, and the current child's content below.
- **`description`**: Displays static text content. The text content and style are defined in its schema.
- **`picture`**: Displays a static image. The style is defined in its schema.
- **`separator`**: A visual break or line to separate sections within the form. The style is defined in its schema.

### TypeScript

```typescript
type ColumnType =
  | "checkbox"
  | "star"
  | "dropdown"
  | "number"
  | "text"
  | "bigtext"
  | "signature"
  | "gps"
  | "api";

type LayoutType =
  | "input"
  | "description"
  | "picture"
  | "separator"
  | "group"
  | "accordion"
  | "tab";

interface BaseColumn {
  id: string; // Unique identifier for the Column
  name: string; // Unique name for references
  type: ColumnType | null;
  schema: Json | null; // Predefined settings specific to the Column type
}

interface CheckboxColumn extends BaseColumn {
  type: "checkbox";
  schema: Json<{
    tristate?: boolean;
    reasons?: Record<boolean | null, string[]>;
  }> | null; // Indicate if it's a 3-state checkbox
}

interface StarColumn extends BaseColumn {
  type: "star";
  schema: Json<{
    icons?: Record<number, string>; // Maps a numerical value to an icon name string
    reasons?: Record<number, string[]>; // Maps a numerical value to an array of reason strings
  }> | null;
}

interface DropdownColumn extends BaseColumn {
  type: "dropdown";
  schema: Json<{
    options: string[];
    reasons?: Record<string, string[]>; // Selectable reasons for each option
  }> | null;
}

interface NumberColumn extends BaseColumn {
  type: "number";
  schema: Json<{ min?: number; max?: number }> | null;
}

interface TextColumn extends BaseColumn {
  type: "text";
  schema: Json<{
    minLength?: number;
    maxLength?: number;
    format?: "email" | "phone" | "url" | "password";
  }> | null;
}

interface BigTextColumn extends BaseColumn {
  type: "bigtext";
  schema: Json<{ minLength?: number; maxLength?: number }> | null;
}

interface SignatureColumn extends BaseColumn {
  type: "signature";
  schema: null;
}

type GpsCoord = { latitude: number; longitude: number };

interface GpsColumn extends BaseColumn {
  type: "gps";
  schema: Json<{ region: GpsCoord[]; inside: boolean } | {}> | null; // Allow coords only inside/outside a region
}

interface ApiColumn extends BaseColumn {
  type: "api";
  schema: Json<{
    request: {
      url: string;
      method: "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
      schema?: Ajv.JsonSchemaType;
      // Consider adding options for handling authentication, headers, params, data with references like #FormId, #UserId, #CompanyId, #columnName1#keyNameA
    };
    response?: {
      schema: Ajv.JsonSchemaType;
      show: boolean;
    }; // response is not stored if the schema is not defined
  }> | null;
}

type Column =
  | CheckboxColumn
  | StarColumn
  | DropdownColumn
  | NumberColumn
  | TextColumn
  | BigTextColumn
  | SignatureColumn
  | GpsColumn
  | ApiColumn;

interface BaseLayout {
  id: string; // Unique identifier for the Layout
  name: string | null; // Label displayed to user
  type: LayoutType | null;
  schema: Json | null; // Predefined settings specific to the Layout type
  order: number | null; // The position index relative to sibling Layouts
  required: boolean | null; // User input is required if true, allowed if false, disallowed/readonly/static if null
  LayoutId: string | null; // Id of the parent Layout, null if top level
  ColumnId: string | null; // Id of the linked Column
}

interface InputLayout extends BaseLayout {
  type: "input";
  schema: Json<{ content: string }> | null;
  ColumnId: string;
}

interface DescriptionLayout extends BaseLayout {
  type: "description";
  schema: Json<{ content: string }> | null;
  ColumnId: null;
}

interface PictureLayout extends BaseLayout {
  type: "picture";
  schema: Json<{ width?: number; height?: number }> | null;
  ColumnId: null;
  // Create Attachment with LayoutId and FileId to let designer upload the photo
}

interface SeparatorLayout extends BaseLayout {
  type: "separator";
  schema: Json<{ line?: boolean }> | null;
  ColumnId: null;
}

interface GroupLayout extends BaseLayout {
  type: "group";
  schema: null;
  ColumnId: null;
}

interface AccordionLayout extends BaseLayout {
  type: "accordion";
  schema: null;
  ColumnId: null;
}

interface TabLayout extends BaseLayout {
  type: "tab";
  schema: null;
  ColumnId: null;
}

type Layout =
  | InputLayout
  | DescriptionLayout
  | PictureLayout
  | SeparatorLayout
  | GroupLayout
  | AccordionLayout
  | TabLayout;

interface Condition {
  id: string; // Unique identifier for the condition
  type:
    | "and"
    | "or"
    | "equals"
    | "notEquals"
    | "greaterThan"
    | "lessThan"
    | "contains"
    | "startsWith"
    | "endsWith"; // Type of comparison
  value: string | null; // Value to compare against, can use references
  valueColumnId: string | null; // Id of the Column to evaluate its value
  effect: "show" | "hide" | "enable" | "disable" | "require" | null;
  effectLayoutId: string | null; // Id of the Layout to affect
  parentConditionId: string | null; // Id of the parent condition
}

interface Row {
  id: string; // Unique identifier for the row
  name: string | null; // Name of the row (e.g., 'Motherboard', 'CPU')
  FormId: string | null; // Foreign key referencing the Form table
}

interface Cell {
  id: string; // Unique identifier for the cell data
  ColumnId: string; // Foreign key referencing the Column table
  RowId: string | null; // Foreign key referencing the Row table, null if global to the Form
  FormId: string; // Foreign key referencing the Form table
  data: string | null; // The data captured for this Column in this specific Row
}

interface Form {
  id: string; // Unique identifier for the form instance
  name: string | null; // Name of the form
  // Potentially other metadata about the form
}
```

### Understanding Layouts

The `Layout` table introduces a structured way to organize the elements of a form. Think of `Layouts` as the blueprints that define how your form looks and feels. They provide different **views** of the same underlying data. The actual data entered by the user is stored in the `Cells` table.

Here's a breakdown of each layout type:

- **`input` Layout:** Links to a `Column`. It represents an input field where users submit or view data. When a `Layout` has the type 'input', its `ColumnId` will point to a specific record in the `Column` table. This is how the actual input control (like a text box or checkbox) is included in the form structure.

- **`group` Layout:** Visually groups related layouts together. Imagine drawing a box around several input fields with a bold title. The `group` layout helps in organizing the form into logical sections.

- **`accordion` Layout:** Similar to a `group`, but it presents its child layouts in a collapsible manner. Only one section is expanded at a time, saving screen space and improving navigation for forms with many sections.

- **`tab` Layout:** Organizes its children into separate tabs. Users can switch between tabs to view different sets of related layouts. This is useful for dividing a form into distinct logical steps or categories.

- **`description` Layout:** Displays static text content, such as instructions or explanations within the form. The `schema` JSON will contain the text and styles.

- **`picture` Layout:** Displays static images within the form. The `schema` JSON will contain the dimensions and styles.

- **`separator` Layout:** Inserts visual breaks, like horizontal lines, to divide the form into sections and improve readability.

#### How Layouts and Columns, Rows, Cells Work Together

Think of the `Layout` as the **structure** or the **view**, the `Column` as the **schema**, the `Row` as a repeated item and the `Cells` as the **data**.

1. **Layout Defines the View:** The `Layout` table dictates how the form is presented to the user. It arranges columns into groups, tabs, accordions, and places static content like descriptions and pictures. It defines the order of the widgets, hierarchy, and requiredness.

2. **'input' Layout Points to Data Definition:** When a `Layout` has the type 'input', its `ColumnId` points to a specific `Column` record. This `Column` record defines the type of input (text, checkbox, etc.), its reference name, and any settings specific to its type.

3. **Cells Store the Data:** The `Cells` table holds the actual data entered by the user for each `Column` within a specific `Form` and `Row`. When a `Layout` of type 'input' with a certain `ColumnId` is displayed, the system looks in the `Cells` table for a record with the same `ColumnId` (and the relevant `FormId` and `RowId`) to retrieve and display the user's input.

**Analogy: Different Views of the Same Data:** Think of the data you collect as a central database. The `Layout` table provides different lenses or views through which you can see this data. A 'group' layout is like organizing data into folders, a 'tab' layout is like having different sections in a document, and the 'input' layout is the actual field where the data (`Cells`) is displayed or entered. The 'input' layout acts as the connection point between the form's structure and the actual data.

### Top-Level Layouts

When a `Layout` record has a `LayoutId` that is `NULL`, it signifies a **top-level parent layout**. These top-level layouts represent entire forms. Your application can query the `Layout` table for records where `LayoutId` is `NULL` to get a list of all available forms. This allows users to choose which form they want to fill out or how they want to view the data.

#### Example

Imagine you have the following `Layout` records:

| id              | type  | ColumnId  | LayoutId    | name                 | order |
| --------------- | ----- | --------- | ----------- | -------------------- | ----- |
| form-123        | group | NULL      | NULL        | Application Form     | 1     |
| section-456     | group | NULL      | form-123    | Personal Information | 1     |
| name-input      | input | input-789 | section-456 |                      | 1     |
| address-input   | input | input-901 | section-456 |                      | 2     |
| section-789     | group | NULL      | form-123    | Employment History   | 2     |
| job-title-input | input | input-321 | section-789 |                      | 1     |
| form-456        | tab   | NULL      | NULL        | Feedback Survey      | 2     |
| tab-901         | group | NULL      | form-456    | About You            | 1     |
| rating-input    | input | input-654 | tab-901     |                      | 1     |

In this example, `form-123` and `form-456` are top-level layouts (their `LayoutId` is `NULL`). This indicates that "Application Form" and "Feedback Survey" are available forms that users can select. The other `Layout` records are part of these forms, defining their internal structure.

### Grouping Layouts

The `group`, `accordion`, and `tab` Layouts serve as containers to organize other Layouts. Layouts are associated with a container by setting their `LayoutId` to the `id` of the container Layout. This creates a hierarchical structure within the form. If the parent is not a container type (`group`, `accordion`, or `tab`), any children will just be indented under the parent without special styling.

#### Example

```typescript
const environmentTypeColumn: DropdownColumn = {
  id: "environment-input-425",
  name: "environment-input",
  type: "dropdown",
  schema: JSON.stringify({
    options: ["Lab", "Factory Floor", "Outdoor", "Server Room"],
  }),
};

const deviceGroup: GroupLayout = {
  id: "device-group-876",
  name: "Device Information",
  type: "group",
  order: 1,
  schema: null,
};
const deviceDescription: DescriptionLayout = {
  id: "device-desc-954",
  name: "Details",
  type: "description",
  order: 1,
  LayoutId: "device-group-876", // inside deviceGroup
  schema: JSON.stringify({
    content: "Details about CPU...",
  }),
  required: null,
};
const environmentDropdown: InputLayout = {
  id: "environment-input-231",
  name: "Operating Environment",
  type: "input",
  order: 2,
  required: false,
  schema: null,
  ColumnId: "environment-input-425", // links environmentTypeColumn
  LayoutId: "device-group-876", // inside deviceGroup
};

const columns: Column[] = [environmentTypeColumn];
const layouts: Layout[] = [deviceGroup, deviceDescription, environmentDropdown];
// In this example, 'deviceDescription' and 'environmentDropdown' are children of the 'deviceGroup'.
```

### Conditional Logic

The `Condition` dynamically controls the visibility of the layout specified by `effectLayoutId`. Each condition is evaluated based on rules that compare the corresponding data of the `Column` specified by `valueColumnId` in the `Cells` table. A `Condition` can have child `Condition`s, with its type set to 'and' or 'or'. If the condition and its children evaluate to `true`, the layout specified by `effectLayoutId` is displayed or enabled, based on the `effect` ("show", "hide", "enable", or "disable").

#### Example

```typescript
const voltageCorrectCheckboxColumn: CheckboxColumn = {
  id: "voltage-correct-567",
  name: "voltage-correct",
  type: "checkbox",
  label: "Voltage within Specifications",
  schema: null,
};
const voltageReasonColumn: TextColumn = {
  id: "voltage-reason-890",
  name: "voltage-reason",
  type: "text",
  schema: null,
};

const powerSectionGroupLayout: GroupLayout = {
  id: "power-group-112",
  name: "power-group",
  type: "group",
  label: "Power Supply",
  required: null,
  LayoutId: null,
  order: 1,
  schema: null,
};
const powerDescriptionLayout: DescriptionLayout = {
  id: "power-desc-334",
  name: "power-desc",
  type: "description",
  LayoutId: "power-group-112",
  label: "Details",
  required: null,
  order: 1,
  schema: JSON.stringify({ content: "Details about the power supply..." }),
};
const voltageCorrectCheckboxLayout: InputLayout = {
  id: "voltage-correct-567",
  type: "input",
  name: "Voltage within Specifications",
  required: null,
  LayoutId: "power-group-112",
  ColumnId: "voltage-correct-567",
  schema: null,
  order: 1,
};
const voltageReasonLayout: InputLayout = {
  id: "voltage-reason-890",
  type: "input",
  name: "Please specify the reason",
  LayoutId: "power-group-112",
  ColumnId: "voltage-reason-890",
  required: true,
  order: 1,
  schema: null,
};

const showVoltageReason: Condition = {
  id: "show-voltage-reason-456",
  valueColumnId: "voltage-correct-567",
  type: "equals",
  value: "false",
  effect: "show",
  effectLayoutId: "voltage-reason-890",
};

const columns: Column[] = [voltageCorrectCheckboxColumn, voltageReasonColumn];

const layouts: Layout[] = [
  powerSectionGroupLayout,
  powerDescriptionLayout,
  voltageCorrectCheckboxLayout,
  voltageReasonLayout,
];

const conditions: Condition[] = [showVoltageReason];

// 'voltage-reason-890' will only be visible if the value of the 'voltage-correct-567' Column is 'false'.
```

### Repetition of Columns Across Rows

To handle scenarios where the same set of columns needs to be applied to multiple distinct entities (e.g., components in a device), we introduce the **Rows** table and the linking **`Cells`** table.

**Rows:** Represent the different components or contexts where the form is being filled.

**`Cells` Table:** This table acts as a junction table, connecting **Rows** to **Columns** within a specific **Form**, and storing the specific data collected for each `Column` within each `Row`. This avoids duplicating the `Column` definitions for every `Row`. If some `Column` inputs apply to the whole form and not separate rows, set `RowId` to null.

```typescript
interface Form {
  id: string; // Unique identifier for the form instance
  name: string | null; // Name of the form
  // Potentially other metadata about the form
}

interface Row {
  id: string; // Unique identifier for the row
  name: string | null; // Name of the row (e.g., 'Motherboard', 'CPU')
}

interface Cell {
  id: string; // Unique identifier for the cell data
  FormId: string; // Foreign key referencing the Form table
  RowId: string | null; // Foreign key referencing the Row table, null if global to Form
  ColumnId: string; // Foreign key referencing the Column table
  data: string | null; // The data captured for this Column in this specific Row
}
```

#### Example

```typescript
const powerVoltageCorrect: CheckboxColumn = {
  id: "voltage-correct-987",
  name: "voltage-correct",
  type: "checkbox",
  schema: null,
};
const voltageReason: TextColumn = {
  id: "voltage-reason-654",
  name: "voltage-reason",
  type: "text",
  schema: null,
};

const deviceInformationGroupLayout: GroupLayout = {
  id: "device-info-321",
  name: "device-info",
  type: "group",
  order: 1,
  label: "Device Information",
  schema: null,
};
const deviceInfoDescriptionLayout: DescriptionLayout = {
  id: "device-desc-654",
  name: "device-desc",
  type: "description",
  label: "Info",
  schema: JSON.stringify({
    content: "General device information and instructions...",
  }),
  required: null,
  LayoutId: "device-info-321",
  order: 1,
};
const powerVoltageCorrectLayout: InputLayout = {
  id: "voltage-correct-987",
  name: "voltage-correct",
  type: "input",
  label: "Voltage Correct",
  required: true,
  LayoutId: "device-info-321",
  ColumnId: "voltage-correct-987",
  order: 3,
  schema: null,
};
const voltageReasonLayout: InputLayout = {
  id: "voltage-reason-654",
  name: "voltage-reason",
  type: "input",
  label: "Reason for Incorrect Voltage",
  LayoutId: "device-info-321",
  ColumnId: "voltage-reason-654",
  required: null,
  order: 1,
  schema: null,
};

const showVoltageReason: Condition = {
  id: "show-voltage-reason-321",
  valueColumnId: "voltage-correct-987",
  type: "equals",
  value: "false",
  effect: "show",
  effectLayoutId: "voltage-reason-654",
};

const columns: Column[] = [powerVoltageCorrect, voltageReason];

const layouts: Layout[] = [
  deviceInformationGroupLayout,
  deviceInfoDescriptionLayout,
  powerVoltageCorrectLayout,
  voltageReasonLayout,
];

const conditions: Condition[] = [showVoltageReason];

const rows: Row[] = [
  { id: "row-motherboard-542", FormId: null, name: "Motherboard" },
  // 1 motherboard exists in all computers, global to all forms, so FormId is null
  { id: "row-cpu-424", FormId: null, name: "CPU" },
  { id: "row-gpu-789", FormId: null, name: "GPU" },
  // some computers have a gpu, global to all forms but still optional
  { id: "row-ram-stick-123", FormId: "form-001", name: "RAM Stick 1" }, // specific Form
  { id: "row-ram-stick-427", FormId: "form-001", name: "RAM Stick 2" },
];

const forms: Form[] = [
  { id: "form-001", name: "Electronics Assembly Check - Rev 3" },
];

const cells: Cell[] = [
  // Non row specific data for 'form-001'
  {
    id: "cell-111",
    FormId: "form-001",
    RowId: null,
    ColumnId: "environment-input-741",
    data: "Lab",
  },

  // Data for 'Motherboard' row in 'form-001'
  {
    id: "cell-222",
    FormId: "form-001",
    RowId: "row-motherboard-542",
    ColumnId: "voltage-correct-987",
    data: "true",
  },

  // Data for 'CPU' row in 'form-001'
  {
    id: "cell-333",
    FormId: "form-001",
    RowId: "row-cpu-424",
    ColumnId: "voltage-correct-987",
    data: "false",
  },
  {
    id: "cell-444",
    FormId: "form-001",
    RowId: "row-cpu-424",
    ColumnId: "voltage-reason-654",
    data: "Inconsistent power delivery",
  },

  // ... and so on for other rows and forms
];
```

#### Explanation

- The `columns` array defines the structure of the form columns.
- The `rows` array lists the different components where the form is being filled.
- The `cells` array stores the actual data collected for each `Column` in each row. Notice how the `ColumnId` refers back to the columns defined in `columns`, and the `RowId` refers to the rows defined in the `rows` array. `RowId` can be null, meaning it's for the form in general.

This approach avoids duplicating the `Column` definitions for each row, making the data structure more efficient and maintainable. When displaying the form for a specific row, the system would fetch the relevant `Cell` records based on the `RowId` and each `ColumnId`.

### Custom Api Endpoints

The form can connect to api endpoints with `ApiColumn`.

In online/offline mode, the user can fill out the form defined by `Column.schema.request.schema` and store the JSON in `Cell.data.request`.

In online mode, a fetch/submit button is enabled. It will fetch/submit the optional request data entered by the user and store the response data in `Cell.data.response` when clicked. The response must adhere to the JSON schema defined in `Column.schema.response.schema` and can be displayed to the user if `Column.schema.response.show` is true.

In offline mode, the fetch/submit button is disabled, but stored data in `Cell.data.request` and `Cell.data.response`, if it exists, can be displayed.

If this `Column` is required, and the user has not fetched/submitted successfully yet, the `Column` will show a required warning message.

#### Example

```typescript
const selectFactoryLocationColumn: GpsColumn = {
  id: "factory-location-765",
  name: "factory-location",
  type: "gps",
  schema: null,
};
const partLookupColumn: ApiColumn = {
  id: "part-lookup-432",
  name: "part-lookup",
  type: "api",
  schema: JSON.stringify({
    request: {
      url: "https://api.example.com/parts?lat=${#factory-location#latitude}&lon=${#factory-location#longitude}",
      method: "GET",
      schema: {}, // Add a basic schema for the request if needed
    },
    response: {
      schema: {
        properties: {
          modelNumber: { type: "string", title: "Model Number" },
          manufacturer: { type: "string", title: "Manufacturer" },
          // ... other properties from the API response
        },
      },
      show: true,
    },
  }),
};

const selectFactoryLocationLayout: InputLayout = {
  id: "factory-location-765",
  name: "Factory Location",
  type: "input",
  ColumnId: "factory-location-765",
  required: false,
  order: 1,
};
const partLookupLayout: InputLayout = {
  id: "part-lookup-432",
  name: "Find Part Suppliers Nearby",
  type: "input",
  ColumnId: "part-lookup-432",
  required: false,
  order: 2,
};

const columns: Column[] = [selectFactoryLocationColumn, partLookupColumn];
const layouts: Layout[] = [selectFactoryLocationLayout, partLookupLayout];
```

### Benefits of this Design

- **Flexibility:** Supports a wide range of input types.
- **Organization:** Grouping allows for logical structuring of forms.
- **Dynamic Behavior:** Conditional logic enables dynamic display of columns.
- **Data Efficiency:** The `Cells` table prevents definition duplication when repeating columns across multiple rows within the same form.
- **Maintainability:** Clear separation of `Column` definitions and `Row`-specific data.
- **API Integration:** `ApiColumn` allows integration with external data sources and submission endpoints. The form can keep track of the results.
