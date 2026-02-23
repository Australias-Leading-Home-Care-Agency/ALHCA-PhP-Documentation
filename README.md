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


## 2. GravityFormsCustomRules.php — Advanced Technical Documentation

### 2.1 Overview

GravityFormsCustomRules.php implements a modular, object-oriented framework for enforcing custom validation and UI rules on Gravity Forms fields within WordPress. The class is designed for extensibility, maintainability, and robust integration with Gravity Forms’ hooks and client-side behaviors.

### 2.2 Architectural Principles

- **OOP Encapsulation:** All rules are encapsulated within a class, allowing for easy extension and override.
- **Hook Registration:** Rules are registered via WordPress and Gravity Forms hooks in the constructor, ensuring self-registration and minimal boilerplate.
- **Dual Enforcement:** Both server-side (PHP) and client-side (JavaScript) enforcement are used for critical rules, providing defense-in-depth against bypasses.
- **Single Responsibility:** Each method is responsible for a distinct aspect of rule enforcement, facilitating unit testing and future refactoring.

### 2.3 Integration Points

- **WordPress & Gravity Forms Hooks:**
    - `gform_datepicker_options_pre_init`: Filters datepicker options before initialization, allowing for dynamic restriction of selectable dates.
    - `gform_enqueue_scripts`: Injects custom JavaScript into the form rendering pipeline, enabling client-side enforcement and UI modifications.
- **Form & Field Targeting:**
    - Rules are scoped to specific form and field IDs (e.g., Form #21, Field #9), preventing unintended side effects on other forms.
    - The class can be extended to support multiple forms/fields via configuration arrays or dynamic registration.

### 2.4 Rule Implementation: Date Locking

- **Server-Side Enforcement:**
    - The `lock_field_to_today` method intercepts datepicker initialization for Form #21, Field #9.
    - Sets both `minDate` and `maxDate` to 0, restricting the datepicker to only allow the current date.
    - Ensures that even if JavaScript is disabled, the field cannot be set to a date outside today via Gravity Forms’ backend validation.
- **Client-Side Enforcement:**
    - The `add_datepicker_lock_js` method injects a script that:
        - Sets the field value to today’s date in DD/MM/YYYY format.
        - Disables manual typing and pasting into the field, preventing user circumvention.
        - Restricts the jQuery UI datepicker widget to only allow today.
        - Listens for changes and forcibly resets the value if the user attempts to select a different date.
- **Edge Cases & Fallbacks:**
    - If the field is not rendered as a datepicker (e.g., due to theme conflicts), the script still disables manual input.
    - The script checks for the presence of the field and its classes before applying restrictions, avoiding JavaScript errors.

### 2.5 Extending the Class

- **Adding New Rules:**
    - Register additional hooks in the constructor.
    - Implement new methods for each rule, following the single-responsibility principle.
    - Use configuration arrays to support multiple forms/fields.
- **Example: Custom Validation**

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

### 2.6 Security Considerations

- **Input Sanitization:** All user input should be validated and sanitized, both server-side and client-side.
- **Defense-in-Depth:** By enforcing rules at multiple layers, the class mitigates risks from JavaScript-disabled browsers, malicious users, and plugin conflicts.
- **Auditability:** The OOP structure allows for easy auditing and code review, with clear separation of concerns.

### 2.7 Performance & Reliability

- **Minimal Overhead:** Hooks are registered only for targeted forms/fields, avoiding unnecessary processing.
- **Graceful Degradation:** If JavaScript fails to load, server-side restrictions remain in place.
- **Compatibility:** Designed to work with Gravity Forms’ standard field rendering and jQuery UI datepicker.

### 2.8 Troubleshooting & Edge Cases

- **Field Not Found:** If the field selector changes (e.g., due to theme or plugin updates), update the JavaScript selector accordingly.
- **Multiple Date Fields:** Extend the class to handle arrays of field IDs if multiple date fields require locking.
- **AJAX Forms:** The script is injected on every form render, including AJAX-loaded forms, ensuring consistent enforcement.

### 2.9 Future Enhancements

- **Configurable Rules:** Refactor to accept a configuration array for dynamic rule registration.
- **Unit Tests:** Implement PHPUnit tests for server-side methods.
- **Admin UI:** Provide a WordPress admin interface for managing custom rules.

### 2.10 Example Usage & Initialization

The class is initialized automatically:

```php
new GravityFormsCustomRules();
```

To extend for additional rules, subclass or modify the constructor to register more hooks.

### 2.11 References

- [Gravity Forms Developer Documentation](https://docs.gravityforms.com/)
- [WordPress Plugin API](https://developer.wordpress.org/plugins/hooks/)
- [jQuery UI Datepicker](https://jqueryui.com/datepicker/)

### 2.12 Contact & Support

For advanced customization or troubleshooting, contact the ALHCA web development team or consult the references above.

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
