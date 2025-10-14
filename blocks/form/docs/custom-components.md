# Creating Custom Form Components

This guide explains how to create custom components for the Form block, which follows an MVC (Model-View-Controller) architecture. It covers the structure, authorable properties, extension mechanism, and best practices for custom components.

---

## Architecture Overview
- **Model:** Defined by the JSON schema for each field/component. Authorable properties are specified in the corresponding JSON file (see `blocks/form/models/form-components`).
- **View:** The HTML structure for each field type is described in [form-field-types.md](./form-field-types.md). This is the base structure your component will extend or modify.
- **Controller/Component Logic:** Implemented in JavaScript, either as OOTB (out-of-the-box) or custom components.

---

## OOTB Components
- OOTB (out-of-the-box) components are located in `blocks/form/models/form-components`.
- Each OOTB component has a JSON file defining its authorable properties (e.g., `_text-input.json`, `_drop-down.json`).
- These properties are available to authors in the form builder and are passed to the component as part of the field definition (`fd`).
- The base HTML structure for each OOTB component is documented in [form-field-types.md](./form-field-types.md).

---

## Custom Components: Structure & Placement
- Custom components reside in the `blocks/form/components` folder.
- Each custom component must be placed in its own folder, named after the component (e.g., `countdown-timer`).
- Inside the folder, you must have:
  - `countdown-timer.js` (main logic)
  - `countdown-timer.css` (optional, for styles)
- The name of the folder and the JS/CSS files must match.

---

## Defining New Properties for Custom Components
- If you need to capture new properties for your custom component, **create a new JSON file** in `blocks/form/models/form-components` that extends an existing component definition.
- This is typically done by referencing the base component and adding or overriding properties as needed.
- The new JSON file should include the new properties under the `properties` or `fields` section.
- **Example: Countdown Timer**
  - See `_countdown-timer.json` for a real-world example. This file defines a custom component that extends the base button component and adds new properties like `initialText`, `finalText`, `time`, and `retries`.
  - These properties are defined in the `fields` array of the JSON file, making them available to authors in the form builder.
- The custom component can also be identified using the `:type` property, which can be set as `fd:viewType` in the JSON file (e.g., `fd:viewType: countdown-timer`). This allows the system to recognize and load the correct custom component.
- Any new properties added in the JSON definition are available in the field definition as `properties.<propertyName>` in your component's JS logic.

---

## Extending OOTB Components
- Custom components **must extend** from a predefined set of OOTB components.
- The system identifies which OOTB component to extend based on the `variant` property in the field's JSON (`fd.properties.variant`) or the `:type` property (which can be set as `fd:viewType`).
- The system maintains a registry of allowed custom component variants. Only variants listed in this registry can be used.
- When rendering a form, the system checks the `variant` property or `:type`/`fd:viewType`, and if it matches a registered custom component, loads the corresponding JS and CSS files from the `blocks/form/components` folder.
- The custom component is then applied to the base HTML structure of the OOTB component, allowing you to enhance or override its behavior and appearance.

---

## Creating composite component

- Custom components can be created by combining exisiting components.
- One has to define this composition structure as a template inside the respective component json.
- Lets take the example of Terms and condition component. It consists of a parent panel that has a plain-text field for the terms and a checkbox for capturing the agreement from the user, so its composition structure would look as shown below:

```json
{
  "definitions": [
    {
      "title": "Terms and conditions",
      "id": "tnc",
      "plugins": {
        "xwalk": {
          "page": {
            "resourceType": "core/fd/components/form/termsandconditions/v1/termsandconditions",
            "template": {
              "jcr:title": "Terms and conditions",
              "fieldType": "panel",
              "fd:viewType": "tnc",
              "text": {
                "value": "Text related to the terms and conditions come here.",
                "sling:resourceType": "core/fd/components/form/text/v1/text",
                "fieldType": "plain-text",
                "textIsRich": true
              },
              "approvalcheckbox": {
                "name": "approvalcheckbox",
                "jcr:title": "I agree to the terms & conditions.",
                "sling:resourceType": "core/fd/components/form/checkbox/v1/checkbox",
                "fieldType": "checkbox",
                "required": true,
                "type": "string",
                "enum": [
                  "true"
                ]
              }
            }
          }
        }
      }
    }
  ],
  ...
}
```
- This composite structure is converted to html and is passed on to the decorate callback of the custom component discussed in the following sections.

---


## Using the Scaffolder tool 
You can run the scaffolder cli tool to setup the boilerplate code required for your custom component.
- Open the terminal. Navigate to the root of your form project and run the following commands:
```sh
npm install
npm run create:custom-component
```

- This will prompt you to give the name of your new component and the base component which it is extending. (which would be `countdown-timer` and `button` for this example).
- Once finished it would create the required boilerplate files and do the wiring required to whitelist your component in authoring.

---


## Component JS API
- Your custom component's JS file **must export a default function** (commonly called `decorate`).
- **Signature:**
  ```js
  export default function decorate(element, fd, container, formId) {
    // element: The HTML structure of the OOTB component you are extending
    // fd: The JSON field definition (all authorable properties)
    // container: The parent element (fieldset or form)
    // formId: The id of the form
    // ... your logic here ...
  }
  ```
- You can modify the `element`, add event listeners, inject additional markup, etc.
- Access any new properties you defined in your JSON as `fd.properties.<propertyName>`.

---

## Reusing Existing Utility Functions

When creating custom components, it's important to leverage existing utility functions from `util.js` and `form.js` if required to avoid code duplication.

Example: For creating or updating error messages for components we can re-use the `updateOrCreateInvalidMsg`

---

## Listening to Field Changes: How `subscribe` Works
- The `subscribe` function allows your component to react to changes in the field's value or other custom events.
- **Important**: Use `subscribe` only when you need to update the view when a field's value is changed programmatically. Do NOT use it for handling user input events.
- When you call `subscribe(element, formId, callback)`, the system registers your callback to be notified when the field changes.
- **Callback Signature:**
  - The callback receives two arguments:
    1. `element`: The HTML element for the field.
    2. `fieldModel`: An object representing the field's state and events (created by `afb-runtime.js`).
- To listen to value changes or custom events, use `fieldModel.subscribe((event) => { ... }, 'eventName')` inside your callback. The `event` object contains details about what changed.
- **Example: Countdown Timer**
  ```js
  import { subscribe } from '../../rules/index.js';

  export default function decorate(fieldDiv, fieldJson, container, formId) {
    // Access custom properties defined in the JSON
    const { initialText, finalText, time } = fieldJson?.properties;

    // ... setup logic ...

    subscribe(fieldDiv, formId, (_fieldDiv, fieldModel) => {
      fieldModel.subscribe(() => {
        // React to custom event (e.g., resetOtpCounter)
        // ... countdown logic ...
      }, 'resetOtpCounter');
    });
  }
  ```
- This allows your component to update its UI or perform logic whenever the field value or properties change programmatically, or when a custom event is triggered.

### Important: Subscribe Callback Invocation Timing

**Critical**: The callback passed to the `subscribe` method is **only invoked during form initialization**, not on every field change. The `fieldModel` object passed as an argument to this callback is the live model instance for the field.

**Key Points:**
- The callback runs **once** when the form initializes and the field is ready
- The `fieldModel` parameter is the actual field model instance that persists throughout the form's lifecycle
- If you need to update any model property, it should be done directly on this `fieldModel` object
- Use `fieldModel.subscribe()` inside the callback to listen for ongoing changes to the field

**Example:**
```js
import { subscribe } from '../../rules/index.js';

export default function decorate(element, fd, container, formId) {
  subscribe(element, formId, (_fieldDiv, fieldModel) => {
    // This callback runs ONCE during form initialization
    // fieldModel is the live model instance for this field
    
    // Update model properties directly on fieldModel
    fieldModel.visible = true;
    fieldModel.enabled = false;
    
    // Listen for ongoing changes using fieldModel.subscribe()
    fieldModel.subscribe((event) => {
      // This callback runs whenever the field changes
      const changes = event.payload.changes;
      changes.forEach(change => {
        if (change.propertyName === 'value') {
          console.log('Value changed:', change.currentValue);
        }
      });
    }, 'change');
  });
}
```

---

## FieldModel API Reference

The `fieldModel` object passed to your `subscribe` callback is an instance of the `Field` class from `afb-runtime.js`. It provides access to the field's state and methods for programmatic control.

### Core Properties (Getters/Setters)

#### Field State Properties
- **`value`** - Get/set the field's current value
- **`visible`** - Get/set field visibility (boolean)
- **`enabled`** - Get/set field enabled state (boolean)
- **`readOnly`** - Get/set field read-only state (boolean)
- **`required`** - Get/set field required state (boolean)
- **`valid`** - Get/set field validation state (boolean)

#### Field Configuration Properties
- **`enum`** - Get/set available options for dropdown/radio/checkbox groups
- **`enumNames`** - Get/set display names for enum options
- **`maximum`** - Get/set maximum value (for number/date fields)
- **`minimum`** - Get/set minimum value (for number/date fields)
- **`placeholder`** - Get placeholder text
- **`tooltip`** - Get tooltip text
- **`description`** - Get/set field description/help text
- **`label`** - Get/set field label object
- **`errorMessage`** - Get/set custom error message
- **`constraintMessage`** - Set constraint-specific error messages

#### Field Metadata Properties
- **`id`** - Field's unique identifier
- **`name`** - Field's name attribute
- **`fieldType`** - Type of field (e.g., 'text-input', 'drop-down')
- **`type`** - Data type (e.g., 'string', 'number', 'boolean')
- **`properties`** - Custom properties object (access via `fieldModel.properties.customProperty`)

### Core Methods

#### Event Handling
- **`subscribe(callback, eventName)`** - Subscribe to field changes or custom events
- **`dispatch(action)`** - Dispatch custom events or actions
- **`queueEvent(action)`** - Queue an event for processing

#### Field Control
- **`focus()`** - Set focus to the field
- **`reset()`** - Reset field to default state
- **`validate()`** - Trigger field validation
- **`markAsInvalid(message, constraint)`** - Mark field as invalid with custom message

#### Data Management
- **`getDataNode()`** - Get the underlying data node
- **`updateDataNodeAndTypedValue(value)`** - Update field value and data model

### Important Notes

1. **Property Access**: All properties are reactive - changing them will trigger appropriate events and update the form model.

2. **Custom Properties**: Access custom properties defined in your JSON schema via `fieldModel.properties.propertyName`.

3. **Event Subscription**: The `subscribe` method returns an object with an `unsubscribe()` method for cleanup.

4. **Validation**: Use `markAsInvalid()` to set custom error messages, or modify `errorMessage` property.

5. **Data Binding**: The fieldModel automatically handles data binding to the form's data model.

### Example Usage in Custom Components

```js
import { subscribe } from '../../rules/index.js';

export default function decorate(element, fd, container, formId) {
  subscribe(element, formId, (_fieldDiv, fieldModel) => {
    // Listen to value changes
    fieldModel.subscribe((event) => {
      const changes = event.payload.changes;
      changes.forEach(change => {
        if (change.propertyName === 'value') {
          console.log('Value changed:', change.currentValue);
          // Update your custom UI based on new value
        }
      });
    }, 'change');

    // Programmatically update field properties
    fieldModel.visible = true;
    fieldModel.enabled = false;
    fieldModel.value = 'new value';
    
    // Set custom error message
    fieldModel.markAsInvalid('Custom validation failed', 'custom');
    
    // Access custom properties
    const customProp = fieldModel.properties.myCustomProperty;
  });
}
```



### How `fieldModel.subscribe` works?


- A `change` event is triggered whenever any property of the field model changes. 
- User can define custom events too but they need to be explicity triggered from rule editor.
- The event object receieved in the `fieldModel.subscribe` callback has the following structure:

```json
{
  "payload": {
    "changes": [
      {
        "propertyName": "visible",
        "currentValue": true,
        "prevValue": false
      }
    ]
  },
  "type": "change",
  "isCustomEvent": false
}

```
---

## Understanding Model Updates and Event Propagation

### How `index.js` Handles HTML Changes

The `index.js` file in the form system continuously listens to changes in the HTML and automatically updates the form model accordingly. This means:

- **Automatic Model Updates**: When users interact with form fields (typing, selecting, etc.), the system automatically detects these changes and updates the underlying form model.
- **Event Propagation**: These model updates propagate throughout the form system, triggering various listeners and callbacks.

**Important**: This automatic model update behavior means that when creating custom components, you must either override this change propagation or handle it appropriately to prevent conflicts with your custom logic.

### Handling Model Updates in Custom Components

When creating custom components, you need to be aware of this automatic model update behavior:

#### 1. **Override Event Propagation When Necessary**
If your custom component needs to prevent the default model update behavior, you must explicitly override or handle the change propagation:

```js
export default function decorate(element, fd, container, formId) {
  // Example: Prevent default change propagation for a custom input
  const input = element.querySelector('change');
  
  input.addEventListener('change', (event) => {
    // Prevent the default model update
    event.stopPropagation();
    
    // Handle the change manually
    // ... your custom logic here ...
  });
}
```

#### 2. **Use `subscribe` Only for View Updates**
The `subscribe` function should **only** be used when you need to update the view when a field's value is changed programmatically. It should NOT be used for handling user input events:

```js
import { subscribe } from '../../rules/index.js';

export default function decorate(element, fd, container, formId) {
  // Only subscribe if you need to react to programmatic changes
  subscribe(element, formId, (fieldDiv, fieldModel) => {
    fieldModel.subscribe((event) => {
      // This callback fires when the model is updated programmatically
      // Use this to update your component's view accordingly
      updateComponentView(event.payload.changes);
    }, 'change');
  });
}
```

#### 3. **Avoid Infinite Loops**
**Critical**: Every time the model is updated, if you have subscribed to changes, you will receive a callback. This can lead to infinite loops if not handled carefully:

```js
// ❌ WRONG - This can cause infinite loops
subscribe(element, formId, (fieldDiv, fieldModel) => {
  fieldModel.subscribe((event) => {
    // Don't update the model from within a model change callback
    fieldModel.updateValue('someValue'); // This triggers another callback!
  }, 'change');
});

// ✅ CORRECT - Only update the view, not the model
subscribe(element, formId, (fieldDiv, fieldModel) => {
  fieldModel.subscribe((event) => {
    // Only update the visual representation
    updateVisualState(event.payload.changes);
    // Don't call fieldModel.updateValue() or similar methods here
  }, 'change');
});
```

### Key Principles for Custom Components

1. **Separate Concerns**: Use `subscribe` only for view updates, not for handling user input
2. **Prevent Propagation**: Use `event.stopPropagation()` when you need custom input handling
3. **Avoid Model Updates in Callbacks**: Never update the model from within a model change callback
4. **Test for Loops**: Always test your components to ensure they don't create infinite update loops
5. **Handle HTML Changes Appropriately**: Either override the automatic model update behavior or work with it, but don't ignore it

---

## Reusing and Extending Fields in Custom Components

When defining fields in your custom component's JSON (for any field group—basic, validation, help, etc.), follow these best practices for maintainability and consistency:

- **Reuse standard/shared fields** by referencing existing shared containers or field definitions (e.g., `../form-common/_basic-input-placeholder-fields.json#/fields`, `../form-common/_basic-validation-fields.json#/fields`). This ensures you inherit all standard options without duplicating them.
- **Add only new or custom fields** explicitly in your container. This keeps your schema DRY and focused.
- **Remove or avoid duplicating fields** that are already included via references. Only define fields that are unique to your component's logic.
- **Reference help containers and other shared content** (e.g., `../form-common/_help-container.json`) as needed for consistency and maintainability.

**Minimal Example: Container in a Custom Component JSON**
```json
{
  "component": "container",
  "name": "validation",
  "label": "Validation",
  "collapsible": true,
  "fields": [
    { "...": "../form-common/_basic-validation-fields.json#/fields" },
    { "component": "number", "name": "minAge", "label": "Minimum Age", "valueType": "number", "description": "Minimum age allowed for date of birth." },
    { "component": "number", "name": "maxAge", "label": "Maximum Age", "valueType": "number", "description": "Maximum age allowed for date of birth." },
    { "component": "text", "name": "minimumErrorMessage", "label": "Minimum age error message", "valueType": "string", "description": "Error message shown when the age is below the minimum allowed." },
    { "component": "text", "name": "maximumErrorMessage", "label": "Maximum age error message", "valueType": "string", "description": "Error message shown when the age is above the maximum allowed." }
  ]
}
```

**Tip:**
- This pattern makes it easy to update or extend logic in the future, and ensures your custom components remain consistent with the rest of the form system.
- Always check for existing shared containers or field definitions before adding new ones.

---

## Step-by-Step: Creating a Custom Component


1. **Choose an OOTB component to extend** (e.g., button, drop-down, text-input, etc.).
2. **Create a folder** in `blocks/form/components` with your component's name (e.g., `countdown-timer`).
3. **Add a JS file** with the same name:
   - `blocks/form/components/countdown-timer/countdown-timer.js`
4. **(Optional) Add a CSS file** for custom styles:
   - `blocks/form/components/countdown-timer/countdown-timer.css`
5. **Define a new JSON file** (e.g., `_countdown-timer.json`) in the same folder as your component JS file (`blocks/form/components/countdown-timer/_countdown-timer.json`). This JSON should extend an existing component and set `fd:viewType` to your component's name (see minimal example above).
   - **For all field groups (basic, validation, help, etc.):** Reference standard/shared fields as shown above, and add only your custom fields explicitly.
6. **Implement the JS logic:**
   - Export a default function as described above.
   - Use the `element` parameter to modify the base HTML structure.
   - Use the `fd` parameter if needed for standard field data.
   - Use `subscribe` to listen to field changes or custom events if needed.
7. **Register your component as a variant** in the form builder and set the `variant` property or `fd:viewType`/`:type` in the JSON to your component's name (e.g., `countdown-timer`).
8. **Update `mappings.js`:** Add your component's name to the `OOTBComponentDecorators` (for OOTB-style components) or `customComponents` list so it is recognized and loaded by the system.
9. **Update `_form.json`:** Add your component's name to the `filters.components` array so it can be dropped in the authoring UI.
10. **Update `_component-definition.json`:** Add an entry for the new component in the custom form component section of this json similar to the range component.
11. **Run the build:json script:** Execute `npm run build:json` to compile and merge all component JSON definitions into a single file to be served from the server. This ensures your new component's schema is included in the merged output.

**JS implementation (`countdown-timer.js`):**
```js
import { subscribe } from '../../rules/index.js';

export default function decorate(fieldDiv, fieldJson, container, formId) {
  // Access custom properties defined in your JSON
  const { initialText, finalText, time } = fieldJson?.properties;

  // ... setup logic ...

  subscribe(fieldDiv, formId, (_fieldDiv, fieldModel) => {
    fieldModel.subscribe((e) => {
      const customProperty = fieldModel.properties.initialText;
      const changes = e.payload.changes;
      console.log('field changed')
    }, 'change');
  });
}
```

---

## Best Practices

- **Keep your component logic focused**: Only add/override what is necessary for your custom behavior.
- **Leverage the base structure**: Use the OOTB HTML as your starting point.
- **Use authorable properties**: Expose configurable options via the JSON schema.
- **Namespace your CSS**: Avoid style collisions by using unique class names.
- **Reuse existing utility functions**: Always check `util.js` and `form.js` for existing functions before implementing custom logic.

---

## References
- [form-field-types.md](./form-field-types.md): Base HTML structures and properties for all field types.
- `blocks/form/models/form-components`: OOTB and custom component property definitions.
- `blocks/form/components`: Place for your custom components.
- Example: `blocks/form/components/countdown-timer/_countdown-timer.json` shows how to extend a base component and add new properties.

---

By following this guide, you can create robust, maintainable custom components that integrate seamlessly with the Form block's architecture.
