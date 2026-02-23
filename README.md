# ALHCA Website Submission & Customization Codebase

This directory contains custom PHP code for advanced Gravity Forms processing, CSV export, and theme-level enhancements for the ALHCA website. It is designed for maintainability, extensibility, and robust integration with WordPress and Gravity Forms.

---


## Contents

- `SubmissionToCSV.php` - OOP class for exporting Gravity Forms entries to CSV, with flexible mapping and child data splitting.
- `GravityFormsCustomRules.php` - OOP class for enforcing custom Gravity Forms field rules (e.g., date locking).
- `GravityFormsProcessingSpinner.php` - OOP class to prevent duplicate form submissions with a processing spinner overlay.
- `Functions.php` - Main theme functions file, containing hooks, filters, and legacy logic.
- `GravityFormsRepeatedFieldsEducationalPlanner.php` - OOP class for injecting custom repeater fields into Gravity Forms (Form 25), supporting multi-block child data entry and admin label mapping for export compatibility.
- `GravityPDF/blank-slate-custom.php` - Custom Gravity PDF template for Form 25, designed to output repeater field data in a print-friendly layout. Supports looping through child blocks and displaying all admin-labeled fields.

---


## 1. SubmissionToCSV.php — Advanced Architectural & Data Processing Reference

### Configuration Options
| Option            | Type           | Description                                                                 |
|-------------------|----------------|-----------------------------------------------------------------------------|
| split_children    | `bool`         | Split child data into separate CSVs.                                        |
| child_count       | `int`          | Number of child sections to expect.                                         |
| child_fields      | `array`        | Base names for child fields.                                                |
| field_map         | `array`        | Map admin labels to CSV column names or sublabel arrays.                    |
| field_id_map      | `array`        | Map field IDs directly to CSV column names.                                 |
| filename_prefix   | `string|null`  | Custom prefix for CSV filenames.                                            |
| extra_fields      | `array`        | Extra fields to add at the start/end of CSVs.                               |
| child_columns     | `array`        | Explicit child column names per child number.                               |
| column_reorder    | `array`        | Operations to reorder or remove columns in main CSV.                        |

### 1.1 Architectural Overview

SubmissionToCSV.php is engineered as a highly modular, extensible class for deterministic transformation of Gravity Forms entry data into CSV artifacts. The class is designed to operate in both monolithic and distributed form environments, supporting complex field mapping, child data segmentation, and robust integration with WordPress notification and file management APIs.

#### Core Design Principles

- **Config-Driven Instantiation:** The constructor accepts a configuration array, enabling declarative specification of mapping, splitting, column ordering, and file naming strategies. This allows for runtime flexibility and minimizes hardcoded logic.
- **Hook Registration:** All integration points (notification attachment, post-email cleanup) are registered in the constructor, ensuring atomic initialization and encapsulation of side effects.
- **Data Pipeline:** The class implements a multi-stage data pipeline: field extraction → mapping → child segmentation → column reordering → CSV serialization → file attachment → cleanup.

### 1.2 Field Mapping & Extraction

- **Admin Label Mapping:** The `field_map` configuration enables mapping of Gravity Forms admin labels to CSV column names or sublabel arrays. Supports both simple string mapping and complex sublabel aggregation.
- **Field ID Mapping:** The `field_id_map` allows direct mapping of Gravity Forms field IDs to CSV columns, providing a fallback for fields lacking admin labels or for legacy forms.
- **Sublabel Aggregation:** Supports aggregation of multiple sub-inputs into a single CSV column, with configurable separators and label matching logic. Handles date fields, dropdowns, and custom input structures.
- **Edge Case Handling:** Implements multi-strategy extraction for date fields, including parsing combined date strings, sub-input decomposition, and fallback to main field values. Ensures robust extraction across Gravity Forms field types and plugin versions.

### 1.3 Child Data Segmentation & CSV Generation

- **Split Children Mode:** When `split_children` is enabled, the class segments child data into discrete CSVs based on either explicit `child_columns` or dynamically generated field names. Each child CSV is generated only if meaningful data is present, avoiding empty artifacts.
- **Child Field Name Generation:** The `build_child_field_names` method constructs field names for each child section, supporting both explicit and dynamic naming strategies.
- **Column Reordering:** The `column_reorder` configuration supports both removal and positional reordering of columns, enabling precise control over CSV schema. Operations are applied in sequence, with in-place mutation of the data array.
- **Extra Fields Injection:** The `extra_fields` configuration allows injection of fields at the start and end of each CSV, supporting metadata, hidden fields, and integration tokens.

#### CSV Serialization

- **Header & Data Row Construction:** CSVs are serialized with explicit header rows derived from the mapped data keys, followed by sanitized data rows. Line endings and control characters are normalized to ensure compatibility with downstream processing tools.
- **File Naming Strategy:** Filenames are constructed using configurable prefixes, entry IDs, and timestamps, supporting both deterministic and random naming schemes. Ensures uniqueness and traceability in multi-form environments.
- **Upload Directory Integration:** Files are written to the WordPress upload directory, leveraging `wp_upload_dir()` for path resolution and ensuring compliance with WordPress file management conventions.

### 1.4 Notification Integration & Cleanup

- **Attachment Hook:** CSV files are attached to Gravity Forms notifications via the `gform_notification_{form_id}` filter. Attachment logic is scoped to specific notification names (e.g., 'Admin Notification'), avoiding cross-notification contamination.
- **Post-Email Cleanup:** The `gform_after_email` action triggers cleanup of generated CSV files, ensuring that temporary artifacts do not persist beyond their intended lifecycle. Cleanup is atomic and robust against partial failures.

### 1.5 Edge Cases & Advanced Scenarios

- **Empty Child Handling:** Child CSVs are only generated if at least one key field (e.g., first_name, surname) contains meaningful data, avoiding empty files and reducing noise in notification attachments.
- **Date Field Parsing:** Implements multi-strategy parsing for date fields, including AU-format (DD/MM/YYYY) handling, sub-input decomposition, and fallback to main field values. Ensures compatibility with Gravity Forms’ evolving field conventions.
- **Multi-Form Support:** The class can be instantiated multiple times for different forms, each with its own configuration, supporting heterogeneous form schemas and notification workflows.
- **Legacy Compatibility:** Supports legacy field structures and procedural CSV splitting logic, enabling gradual migration to OOP paradigms.

### 1.6 Extensibility & Customization

- **Declarative Configuration:** All mapping, splitting, and ordering logic is specified via configuration arrays, enabling runtime customization and minimizing code changes.
- **Subclassing & Method Override:** Advanced users can subclass SubmissionToCSV to override extraction, mapping, or serialization logic for bespoke requirements.
- **Integration with External Systems:** CSV artifacts can be further processed, uploaded, or integrated with external systems via post-processing hooks or custom notification actions.

### 1.7 Example: Complex Configuration

```php
new SubmissionToCSV(21, array(
    'field_map' => array(
        'Educator Name' => array(
            'first_name' => array('labels' => ['First Name'], 'separator' => ' '),
            'last_name'  => array('labels' => ['Last Name'], 'separator' => ' '),
        ),
        'Incident Date' => array(
            'incident_day'   => 'Day',
            'incident_month' => 'Month',
            'incident_year'  => 'Year',
        ),
        // ...
    ),
    'field_id_map' => array(
        13 => 'time_of_incident',
        22 => 'incident_location',
    ),
    'filename_prefix' => 'ihc-',
    'extra_fields' => array(
        'start' => array('hidden_form_id' => '', 'submission_token' => ''),
        'end' => array('hidden_contact_entry_id' => '', 'contact_entry_id' => '', 'integration_id' => ''),
    ),
    'split_children' => true,
    'child_count' => 7,
    'child_columns' => array(
        1 => ['child_1_first_name', 'child_1_surname', 'child_1_date_of_birth_day', 'child_1_date_of_birth_month', 'child_1_date_of_birth_year'],
        // ...
    ),
    'column_reorder' => array(
        array('action' => 'remove', 'column' => 'parent_2_address'),
        array('action' => 'move_before', 'column' => 'incident_location', 'before' => 'incident_date'),
    ),
));
```

---



## 2. GravityFormsCustomRules.php — Deep Technical Reference

### 2.1 Architectural Synopsis

GravityFormsCustomRules.php is architected as a high-cohesion, low-coupling module for deterministic enforcement of Gravity Forms field constraints. The class leverages the WordPress plugin API and Gravity Forms’ extensible event model to inject both server-side and client-side controls, ensuring that business logic is not only enforced at the UI layer but also at the data validation boundary.

#### Class Structure & Hook Binding

- **Constructor Pattern:** The class constructor is responsible for registering all relevant hooks, filters, and actions. This ensures that instantiation is atomic and side-effect free, and that all rule logic is encapsulated within the class scope.
- **Event-Driven Enforcement:** Hooks such as `gform_datepicker_options_pre_init` and `gform_enqueue_scripts` are bound to class methods, allowing for granular interception of Gravity Forms lifecycle events.
- **Field Targeting:** All rule logic is parameterized by form and field IDs, allowing for precise targeting and avoiding cross-form contamination. This is critical in multi-form deployments where field semantics may diverge.

### 2.2 Integration & Execution Flow

- **Server-Side Enforcement:**
    - The `lock_field_to_today` method is registered as a filter on the datepicker options initialization event. It mutates the options object, setting `minDate` and `maxDate` to zero, thereby constraining the selectable date range to the current day. This mutation is performed in situ, ensuring compatibility with downstream Gravity Forms logic.
    - The server-side filter is robust against client-side circumvention, as Gravity Forms will reject invalid dates during backend processing.
- **Client-Side Enforcement:**
    - The `add_datepicker_lock_js` method injects a JavaScript payload into the form rendering pipeline. This script programmatically sets the field value to the current date, disables manual input (keydown/paste), and configures the jQuery UI datepicker widget to restrict selection to today. The script also listens for change events, forcibly resetting the value if the user attempts to select an invalid date.
    - The client-side logic is defensive, checking for field presence and class membership before applying restrictions, thereby avoiding runtime errors in edge-case scenarios.

#### Edge Case Handling

- If the field is not rendered as a datepicker (e.g., due to theme or plugin conflicts), the script falls back to disabling manual input, ensuring that the constraint is still enforced.
- The script is injected on every form render, including AJAX-loaded forms, guaranteeing consistent enforcement across dynamic UI states.

### 2.3 Extensibility & Custom Rule Injection

- **Rule Registration:**
    - New rules can be registered by binding additional hooks in the constructor. Each rule should be encapsulated in its own method, adhering to the single-responsibility principle.
    - For complex scenarios, consider parameterizing the class with a configuration array mapping form/field IDs to rule definitions. This enables dynamic rule registration and reduces boilerplate.
- **Example: Custom Field Validation**

```php
class GravityFormsCustomRules {
    // ...existing code...

    public function __construct() {
        // ...existing code...
        add_filter('gform_field_validation_21_10', array($this, 'validate_custom_field'), 10, 4);
    }

    public function validate_custom_field($result, $value, $form, $field) {
        if ($value !== 'expected') {
            $result['is_valid'] = false;
            $result['message'] = 'Custom validation failed.';
        }
        return $result;
    }
}
```

### 2.4 Security & Data Integrity

- **Input Validation:** All user input is validated and sanitized at both the client and server layers. The dual enforcement model ensures that even if JavaScript is disabled or bypassed, server-side validation will reject invalid data.
- **Defense-in-Depth:** By layering constraints at multiple points in the execution flow, the class mitigates risks from malicious actors, plugin conflicts, and browser idiosyncrasies.
- **Auditability:** The OOP structure and event-driven design facilitate code review and auditing, with clear traceability from hook registration to rule execution.

### 2.5 Performance, Reliability & Compatibility

- **Minimal Overhead:** Hooks are registered only for targeted forms/fields, minimizing runtime overhead and avoiding unnecessary event processing.
- **Graceful Degradation:** If client-side scripts fail to load, server-side restrictions remain active, ensuring that constraints are always enforced.
- **Compatibility:** The module is designed to interoperate with Gravity Forms’ standard field rendering and jQuery UI datepicker, and is resilient to theme/plugin overrides.

### 2.6 Troubleshooting & Advanced Scenarios

- **Selector Drift:** If field selectors change due to theme or plugin updates, update the JavaScript selector logic accordingly. Consider abstracting selectors into a configuration array for maintainability.
- **Multi-Field Constraints:** Extend the class to handle arrays of field IDs for scenarios where multiple fields require identical constraints.
- **AJAX & Dynamic Forms:** The script is injected on every form render, including AJAX-loaded forms, ensuring that constraints persist across dynamic UI states.

### 2.7 Future Directions

- **Configurable Rule Engine:** Refactor the class to accept a configuration array for dynamic rule registration, enabling declarative constraint specification.
- **Automated Testing:** Implement PHPUnit tests for server-side methods and JavaScript unit tests for client-side logic.
- **Administrative UI:** Develop a WordPress admin interface for managing custom rules, enabling non-developer configuration.

### 2.8 Usage & Initialization

The class is instantiated as follows:

```php
new GravityFormsCustomRules();
```

For advanced scenarios, subclass or parameterize the constructor to register additional rules or inject configuration.

### 2.9 Reference Materials

- Gravity Forms Developer Documentation: https://docs.gravityforms.com/
- WordPress Plugin API: https://developer.wordpress.org/plugins/hooks/
- jQuery UI Datepicker: https://jqueryui.com/datepicker/

### 2.10 Support & Contact

For advanced customization, integration, or troubleshooting, contact the ALHCA web development team or consult the reference materials above.

---


## 3. GravityFormsProcessingSpinner.php

Prevents users from submitting a Gravity Form multiple times by disabling the submit button and displaying a processing spinner overlay.

### Features
- **User Feedback:** Shows a spinner and disables the button on submit.
- **Duplicate Prevention:** Blocks duplicate submissions until processing completes.
- **Automatic Reset:** Restores the form state after AJAX validation or errors.

---

## 4. Functions.php

The main theme functions file. Contains:
- Theme and asset enqueuing
- Login redirect logic
- Custom roles and access control
- Email sender configuration
- Legacy CSV splitting logic (for Super Forms)
- Miscellaneous hooks and filters

> **Note:** This file contains legacy and procedural code. For new features, prefer the OOP class-based approach as shown above.

---

## 5. GravityFormsRepeatedFieldsEducationalPlanner.php — Custom Repeater Field Injection

### Architectural Overview

This OOP class dynamically injects a custom repeater field into Gravity Forms (Form 25), supporting up to 7 child blocks. Each block contains structured fields (first/last name, outcomes, planning, etc.), all mapped with admin labels for export compatibility. The class uses WordPress hooks to add/remove the repeater field and ensures placement after a specific field (ID 13).

#### Features
- Multi-block child data entry (up to 7 blocks)
- Section and textarea fields for outcomes and planning
- Admin label mapping for CSV/PDF export
- Compatible with Save & Continue and Gravity PDF
- OOP structure for maintainability

#### Usage
Require the file in Functions.php. The class self-registers via hooks:

```php
require_once __DIR__ . '/GravityFormsRepeatedFieldsEducationalPlanner.php';
```

---

## 6. GravityPDF/blank-slate-custom.php — Custom PDF Template for Repeater Fields

### Architectural Overview

This custom Gravity PDF template is designed to output repeater field data from Form 25 in a print-friendly layout. It loops through each child block in the repeater, displaying all admin-labeled fields. The template can be uploaded to `wp-content/uploads/PDF_EXTENDED_TEMPLATES/` and selected in Gravity PDF settings.

#### Features
- Loops through repeater field (ID 2000) and outputs all child blocks
- Uses adminLabel keys for field mapping
- Print-friendly HTML/CSS styling
- Compatible with Gravity PDF feed automation

#### Usage
Upload the file to `wp-content/uploads/PDF_EXTENDED_TEMPLATES/` and select it in the PDF feed for Form 25.

---

## 7. General Development Notes

- **OOP Structure:** New features should be implemented as classes, self-registering via hooks in their constructors.
- **Separation of Concerns:** Keep unrelated logic in separate files/classes for maintainability.
- **Asset Management:** Place CSS/JS in separate files and enqueue via WordPress functions where possible.
- **Security:** Avoid exposing sensitive data in code or uploads. Validate and sanitize all user input.
- **Technical Debt:** Some legacy code remains for backward compatibility. Refactoring is encouraged as time allows.

---

## 8. Adding New Features

1. Create a new class file in this directory for your feature.
2. Register hooks and actions in the class constructor.
3. Require the file in `Functions.php` (or set up autoloading).
4. Document configuration and usage in this README.

---

## 9. Contact & Support

For questions or contributions, contact the ALHCA web development team.

---

© Australia’s Leading Home Care Agency. All rights reserved.
