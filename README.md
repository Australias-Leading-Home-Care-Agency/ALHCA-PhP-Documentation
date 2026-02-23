# ALHCA Website Submission & Customization Codebase

This directory contains custom PHP code for advanced Gravity Forms processing, CSV export, and theme-level enhancements for the ALHCA website. It is designed for maintainability, extensibility, and robust integration with WordPress and Gravity Forms.

---

## Contents

- `SubmissionToCSV.php` - OOP class for exporting Gravity Forms entries to CSV, with flexible mapping and child data splitting.
- `GravityFormsCustomRules.php` - OOP class for enforcing custom Gravity Forms field rules (e.g., date locking).
- `GravityFormsProcessingSpinner.php` - OOP class to prevent duplicate form submissions with a processing spinner overlay.
- `Functions.php` - Main theme functions file, containing hooks, filters, and legacy logic.

---

## 1. SubmissionToCSV.php

A highly configurable class for converting Gravity Forms entries to CSV files and attaching them to notifications. Supports advanced field mapping, splitting repeating child sections, custom filenames, and automatic cleanup.

### Key Features
- **Flexible Field Mapping:** Map admin labels or field IDs to custom CSV columns.
- **Child Data Splitting:** Optionally split repeating child sections into separate CSVs.
- **Custom Filenames:** Supports custom filename prefixes.
- **Extra Fields & Column Control:** Add extra fields, reorder, or remove columns.
- **Automatic Cleanup:** Deletes generated CSVs after email notifications.

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

### Hooks
- **Notification Filter:** Attaches CSVs to Gravity Forms notifications.
- **Email Action:** Cleans up CSV files after emails are sent.

### Usage Example
```php
new SubmissionToCSV(21, array(
    'field_map' => array(
        'Educator Name' => array(
            'first_name' => 'Educator First Name',
            'last_name'  => 'Educator Last Name',
        ),
        // ...
    ),
    'field_id_map' => array(
        13 => 'time_of_incident',
    ),
    'filename_prefix' => 'ihc-',
    'extra_fields' => array(
        'start' => array('hidden_form_id' => ''),
        'end' => array('hidden_contact_entry_id' => '', 'contact_entry_id' => ''),
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

## 5. General Development Notes

- **OOP Structure:** New features should be implemented as classes, self-registering via hooks in their constructors.
- **Separation of Concerns:** Keep unrelated logic in separate files/classes for maintainability.
- **Asset Management:** Place CSS/JS in separate files and enqueue via WordPress functions where possible.
- **Security:** Avoid exposing sensitive data in code or uploads. Validate and sanitize all user input.
- **Technical Debt:** Some legacy code remains for backward compatibility. Refactoring is encouraged as time allows.

---

## 6. Adding New Features

1. Create a new class file in this directory for your feature.
2. Register hooks and actions in the class constructor.
3. Require the file in `Functions.php` (or set up autoloading).
4. Document configuration and usage in this README.

---

## 7. Contact & Support

For questions or contributions, contact the ALHCA web development team.

---

© Australia’s Leading Home Care Agency. All rights reserved.
