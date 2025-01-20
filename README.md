## Form Builder

### Design Document

This document outlines the design for a system that allows users to create forms with various input types and conditional logic. The system is designed to handle scenarios where the same set of widgets needs to be repeated across different items (e.g., components in a device), allowing for individual data capture for each item.

### Core Concepts

The system uses tables of **Widgets**, **Items**, **Conditions**, and **Forms**. Widgets represent individual input fields or static content within a form. These widgets can be grouped and their visibility can be controlled based on conditional logic. Items represent components in a device or anything that needs repeated entry of data for the Widgets. **ItemWidgets** store the user inputs for each Widget in an Item within a specific Form.

### Credits

```

Copyright 2025 DWJ
Distributed under the Boost Software License, Version 1.0.
https://www.boost.org/LICENSE_1_0.txt

Modifications:
Works in react native expo - native (android/ios) and web.
Works with a relational database. Forms, Items, Widgets, ItemWidgets, Conditions can use foreign key constraints and keep referential integrity.
Form layout can be inverted/sorted by Item or by Widget.
New ItemWidgets type which can store Widget inputs for multiple Items in a Form.
New Condition type which can affect Widgets and have sub Conditions.
New/modified Widgets (checkbox, star, dropdown, description, picture, separator, number, text, bigtext, signature, gps, file, document, photo, video, audio, list, group, accordian, tab, api).
New Typescript types and React helper hooks and components.

```

```

Copyright (c) 2019 EclipseSource Munich
The JSON Forms project is licensed under the MIT License.
https://github.com/eclipsesource/jsonforms

```

### Widget Types

The system supports the following widget types:

- **`checkbox`**: A 2 or 3 state checkbox.
- **`star`**: Select from a predefined list of icons and a predefined list of reasons in a dropdown.
- **`dropdown`**: Select from a predefined list of options.
- **`description`**: Static text. No input required.
- **`picture`**: Static image. No input required.
- **`separator`**: A visual break or line to separate sections within the form.
- **`number`**: Input a numerical value.
- **`text`**: A single-line text input field.
- **`bigtext`**: A multi-line text input field.
- **`signature`**: Captures the user's digital signature.
- **`gps`**: Captures the user's geographical coordinates.
- **`group`**: A container widget used to visually group other widgets. It indents its children and surrounds them with a box, and displays its optional name as a bold header.
- **`accordian`**: Similar to group. It collapses its children under each one's name. Only expands one child at a time.
- **`tab`**: Similar to accordian. Displays children names in a row above, with current name highlighted, and current child's content below.
- **`api`**: Allows fetching from and submitting to an external API endpoint using predefined JSON schema.

### Widget Structure (TypeScript Type)

```typescript
type WidgetType =
  | "checkbox"
  | "star"
  | "dropdown"
  | "description"
  | "picture"
  | "separator"
  | "number"
  | "text"
  | "bigtext"
  | "signature"
  | "gps"
  | "group"
  | "accordian"
  | "tab"
  | "api";

interface BaseWidget {
  id: string; // Unique identifier for the widget
  name: string; // Unique name for placeholders
  label: string | null; // Label displayed to user
  type: WidgetType | null;
  schema: Json | null; // Predefined settings specific to the widget type
  order: number | null; // The position index relative to sibling widgets
  required: boolean | null; // User input is required if true, allowed if false, disallowed/readonly/static if null
  WidgetId: string | null; // Id of the parent widget, null if top level
}

interface CheckboxWidget extends BaseWidget {
  type: "checkbox";
  schema: Json<{
    tristate?: boolean;
    reasons?: Record<boolean | null, string[]>;
  }> | null; // Indicate if it's a 3-state checkbox
}

interface StarWidget extends BaseWidget {
  type: "star";
  schema: Json<{
    icons?: Record<number, string>; // Maps a numerical value to an icon name string
    reasons?: Record<number, string[]>; // Maps a numerical value to an array of reason strings
  }> | null;
}

interface DropdownWidget extends BaseWidget {
  type: "dropdown";
  schema: Json<{
    options: string[];
    reasons?: Record<string, string[]>; // Selectable reasons for each option
  }> | null;
}

interface DescriptionWidget extends BaseWidget {
  type: "description";
  schema: Json<{ content: string }> | null;
}

interface PictureWidget extends BaseWidget {
  type: "picture";
  schema: Json<{ width?: number; height?: number }> | null;
  // Create Attachment with WidgetId and FileId to let designer upload the photo
}

interface SeparatorWidget extends BaseWidget {
  type: "separator";
  schema: Json<{ line?: boolean }> | null;
}

interface NumberWidget extends BaseWidget {
  type: "number";
  schema: Json<{ min?: number; max?: number }> | null;
}

interface TextWidget extends BaseWidget {
  type: "text";
  schema: Json<{
    minLength?: number;
    maxLength?: number;
    format?: "email" | "phone" | "url" | "password";
  }> | null;
}

interface BigTextWidget extends BaseWidget {
  type: "bigtext";
  schema: Json<{ minLength?: number; maxLength?: number }> | null;
}

interface SignatureWidget extends BaseWidget {
  type: "signature";
  schema: null;
}

type GpsCoord = { latitude: number; longitude: number };

interface GpsWidget extends BaseWidget {
  type: "gps";
  schema: Json<{ region: GpsCoord[]; inside: boolean } | {}> | null; // Allow coords only inside/outside a region
}

interface GroupWidget extends BaseWidget {
  type: "group";
  schema: null;
}

interface AccordianWidget extends BaseWidget {
  type: "accordian";
  schema: null;
}

interface TabWidget extends BaseWidget {
  type: "tab";
  schema: null;
}

interface ApiWidget extends BaseWidget {
  type: "api";
  schema: Json<{
    request: {
      url: string;
      method: "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
      schema?: Ajv.JsonSchemaType;
      // Consider adding options for handling authentication, headers, params, data with references like #FormId, #UserId, #CompanyId, #widgetName1#inputNameA
    };
    response?: {
      schema: Ajv.JsonSchemaType;
      show: boolean;
    }; // response is not stored if the schema is not defined
  }> | null;
}

type Widget =
  | CheckboxWidget
  | StarWidget
  | DropdownWidget
  | DescriptionWidget
  | PictureWidget
  | SeparatorWidget
  | NumberWidget
  | TextWidget
  | BigTextWidget
  | SignatureWidget
  | GpsWidget
  | GroupWidget
  | AccordianWidget
  | TabWidget
  | ApiWidget;
```

```typescript
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
  effect: "show" | "hide" | "enable" | "disable" | null;
  effectWidgetId: string | null; // ID of the widget to affect
  compareWidgetId: string | null; // ID of the widget to evaluate its value
  parentConditionId: string | null; // ID of the parent condition
}
```

```typescript
interface Form {
  id: string; // Unique identifier for the form instance
  name: string | null; // Name of the form
  // Potentially other metadata about the form
}

interface Item {
  id: string; // Unique identifier for the item
  name: string; // Name of the item (e.g., 'Motherboard', 'CPU')
}

interface ItemWidget {
  id: string; // Unique identifier for the item widget data
  FormId: string; // Foreign key referencing the Form table
  ItemId: string | null; // Foreign key referencing the Item table
  WidgetId: string; // Foreign key referencing the Widget table
  data: string | null; // The data captured for this widget in this specific item
}
```

### Grouping Widgets

The `group`, `accordian`, and `tab` widgets serve as containers to organize other widgets. Widgets are associated with a container by setting their parent `WidgetId` to the `id` of the container widget. This creates a hierarchical structure within the form. If the parent is not a container type widget (`group`, `accordian`, and `tab`), any children will just be indented under the parent without special styling.

**Example:**

```typescript
const deviceGroup: GroupWidget = {
  id: "device-group-876",
  name: "device-group",
  type: "group",
  order: 1,
  label: "Device Information",
  schema: null,
};
const deviceDescription: DescriptionWidget = {
  id: "device-desc-954",
  name: "device-desc",
  type: "description",
  order: 1,
  WidgetId: "device-group-876", // inside deviceGroup
  schema: JSON.stringify({
    content: "Details about CPU...",
  }),
  label: null,
  required: null,
};
const environmentDropdown: DropdownWidget = {
  id: "environment-input-231",
  name: "environment-input",
  type: "dropdown",
  order: 2,
  WidgetId: "device-group-876", // inside deviceGroup
  label: "Operating Environment",
  required: false,
  schema: JSON.stringify({
    options: ["Lab", "Factory Floor", "Outdoor", "Server Room"],
  }),
};

const widgets: Widget[] = [deviceGroup, deviceDescription, environmentDropdown];
// In this example, 'deviceDescription' and 'environmentDropdown' are children of the 'deviceGroup'.
```

### Conditional Logic

The `Condition` dynamically controls the visibility of its `effectWidgetId` widget. Each condition is evaluated based on rules that compare the corresponding data of widget `compareWidgetId` in the `ItemWidgets` table. A `Condition` can have `Condition` children, and its type set to 'and' or 'or'. If the condition and its children evaluate to `true`, the `effectWidgetId` widget is displayed or enabled, based on the `effect` ("show" or "hide" or "enable" or "disable").

**Example:**

```typescript
const powerSectionGroup: GroupWidget = {
  id: "power-group-112",
  name: "power-group",
  type: "group",
  label: "Power Supply",
  required: null,
  WidgetId: null,
  order: 1,
  schema: null,
};
const powerDescription: DescriptionWidget = {
  id: "power-desc-334",
  name: "power-desc",
  type: "description",
  WidgetId: "power-group-112",
  label: "Details",
  required: null,
  order: 1,
  schema: JSON.stringify({ content: "Details about the power supply..." }),
};
const voltageCorrectCheckbox: CheckboxWidget = {
  id: "voltage-correct-567",
  name: "voltage-correct",
  type: "checkbox",
  label: "Voltage within Specifications",
  required: null,
  WidgetId: "power-group-112",
  schema: null,
  order: 1,
};
const voltageReason: TextWidget = {
  id: "voltage-reason-890",
  name: "voltage-reason",
  type: "text",
  label: "Please specify the reason",
  WidgetId: "power-group-112",
  required: true,
  order: 1,
  schema: null,
};

const showVoltageReason: Condition = {
  id: "show-voltage-reason-456",
  compareWidgetId: "voltage-correct-567",
  type: "equals",
  value: "false",
  effect: "show",
  effectWidgetId: "voltage-reason-890",
};

const widgets: Widget[] = [
  powerSectionGroup,
  powerDescription,
  voltageCorrectCheckbox,
  voltageReason,
];

const conditions: Condition[] = [showVoltageReason];

// 'voltage-reason' will only be visible if the value of the 'voltage-correct' widget is 'false'.
```

### Managing Repetition of Widgets Across Items: The `ItemWidgets` Table

To handle scenarios where the same set of widgets needs to be applied to multiple distinct items (e.g., components in a device), we introduce the table **Items** and the linking table **`ItemWidgets`**.

**Items:** Represent the different components or contexts where the form is being filled.

**`ItemWidgets` Table:** This table acts as a junction table, connecting **Items** to **Widgets** within a specific **Form** and storing the specific data collected for each widget within each item. This avoids duplicating the widget definitions for every item. If some widget inputs apply to the whole form and not separate items, set ItemId to null.

**TypeScript Types for Forms, Items and ItemWidgets:**

```typescript
interface Form {
  id: string; // Unique identifier for the form instance
  name: string; // Name of the form
  // Potentially other metadata about the form
}

interface Item {
  id: string; // Unique identifier for the item
  name: string; // Name of the item (e.g., 'Motherboard', 'CPU')
}

interface ItemWidget {
  id: string; // Unique identifier for the item widget data
  FormId: string; // Foreign key referencing the Form table
  ItemId: string | null; // Foreign key referencing the Item table
  WidgetId: string; // Foreign key referencing the Widget table
  data: string | null; // The data captured for this widget in this specific item
}
```

**Example:**

```typescript
const deviceInformationGroup: GroupWidget = {
  id: "device-info-321",
  name: "device-info",
  type: "group",
  order: 1,
  label: "Device Information",
  schema: null,
};
const deviceInfoDescription: DescriptionWidget = {
  id: "device-desc-654",
  name: "device-desc",
  type: "description",
  label: "Info",
  schema: JSON.stringify({
    content: "General device information and instructions...",
  }),
  required: null,
  WidgetId: "device-info-321",
  order: 1,
};
const powerVoltageCorrect: CheckboxWidget = {
  id: "voltage-correct-987",
  name: "voltage-correct",
  type: "checkbox",
  label: "Voltage Correct",
  required: null,
  WidgetId: "device-info-321",
  order: 3,
  schema: null,
};
const voltageReason: TextWidget = {
  id: "voltage-reason-654",
  name: "voltage-reason",
  type: "text",
  label: "Reason for Incorrect Voltage",
  WidgetId: "device-info-321",
  required: null,
  order: 1,
  schema: null,
};

const showVoltageReason: Condition = {
  id: "show-voltage-reason-321",
  type: "equals",
  effectWidgetId: "voltage-reason-654",
  compareWidgetId: "voltage-correct-987",
  value: "false",
};

const widgets: Widget[] = [
  deviceInformationGroup,
  deviceInfoDescription,
  environmentInput,
  powerVoltageCorrect,
  voltageReason,
];

const conditions: Condition[] = [showVoltageReason];

const items: Item[] = [
  { id: "item-motherboard-542", name: "Motherboard" },
  { id: "item-cpu-424", name: "CPU" },
  { id: "item-gpu-789", name: "GPU" },
  { id: "item-ram-123", name: "RAM Stick 1" },
];

const forms: Form[] = [
  { id: "form-001", name: "Electronics Assembly Check - Rev 3" },
];

const itemWidgets: ItemWidget[] = [
  // Non item specific data for 'form-001'
  {
    id: "item-widget-111",
    FormId: "form-001",
    ItemId: null,
    WidgetId: "environment-input-741",
    data: "Lab",
  },

  // Data for 'Motherboard' item in 'form-001'
  {
    id: "item-widget-222",
    FormId: "form-001",
    ItemId: "item-motherboard-542",
    WidgetId: "voltage-correct-987",
    data: "true",
  },

  // Data for 'CPU' item in 'form-001'
  {
    id: "item-widget-333",
    FormId: "form-001",
    ItemId: "item-cpu-424",
    WidgetId: "voltage-correct-987",
    data: "false",
  },
  {
    id: "item-widget-444",
    FormId: "form-001",
    ItemId: "item-cpu-424",
    WidgetId: "voltage-reason-654",
    data: "Inconsistent power delivery",
  },

  // ... and so on for other items and forms
];
```

**Explanation:**

- The `widgets` array defines the structure of the form widgets.
- The `items` array lists the different components where the form is being filled.
- The `itemWidgets` array stores the actual data collected for each widget in each item. Notice how the `WidgetId` refers back to the widgets defined in `widgets`, and the `ItemId` refers to the items defined in the `items` array. `ItemId` can be null, meaning its for the form in general.

This approach avoids duplicating the widget definitions for each item, making the data structure more efficient and maintainable. When displaying the form for a specific item, the system would fetch the relevant `ItemWidget` records based on the `ItemId` and each `WidgetId`.

### Custom Api Endpoints with ApiWidget

In online/offline mode, the user can fill out the form defined by `Widget.schema.request.schema` and store the json in ItemWidget.data.request.

In online mode, a fetch/submit button is enabled, it will fetch/submit the optional request data entered by user and store the response data in the ItemWidget.data.response when clicked. The response must adhere to the json schema defined in Widget.schema.response.schema and can be displayed to user if Widget.schema.response.show is true.

In offline mode, the fetch/submit button is disabled, but stored data in ItemWidget.data.request and ItemWidget.data.response if it exists can be displayed.

If this widget is required, and user has not fetched/submitted successfully yet, the widget will show a required warning message.

**Example:**

```typescript
const selectFactoryLocation: GpsWidget = {
  id: "factory-location-765",
  name: "factory-location",
  label: "Factory Location",
  type: "gps",
  order: 1,
  WidgetId: null,
  required: false,
  schema: null,
};

const partLookupApi: ApiWidget = {
  id: "part-lookup-432",
  name: "part-lookup",
  type: "api",
  label: "Find Part Suppliers Nearby,
  required: false,
  WidgetId: null,
  order: 2,
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

const widgets: Widget[] = [selectFactoryLocation, partLookupApi];
```

### Benefits of this Design

- **Flexibility:** Supports a wide range of input types.
- **Organization:** Grouping allows for logical structuring of forms.
- **Dynamic Behavior:** Conditional logic enables dynamic display of widgets.
- **Data Efficiency:** The `ItemWidgets` table prevents data duplication when repeating widgets across multiple items within the same form.
- **Maintainability:** Clear separation of widget definitions and item-specific data.
- **API Integration:** The `api` widget allows integration with external data sources and submission endpoints.
