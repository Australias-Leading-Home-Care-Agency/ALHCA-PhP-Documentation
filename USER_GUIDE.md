# TETO User Guide
**Complete Documentation for Setup, Configuration & Daily Use**

---

## Table of Contents
1. [Quick Setup](#quick-setup)
2. [Admin Interface](#admin-interface)
3. [Field Mappings](#field-mappings)
4. [File Approval Workflow](#file-approval-workflow)
5. [Archiving System](#archiving-system)
6. [Authorization & Access Control](#authorization--access-control)
7. [Filename Templates](#filename-templates)
8. [Users Files Management](#users-files-management)
9. [TETO Viewer](#teto-viewer)
10. [Security Features](#security-features)
11. [AWS Cost Optimization](#aws-cost-optimization)

---

## Quick Setup

### Prerequisites
- WordPress 5.0+
- PHP 7.3+
- Gravity Forms + Gravity PDF
- AWS S3 bucket configured
- AWS credentials (Access Key + Secret Key)

### Step 1: Install Database Table

Run once after activating TETO:

```php
// In WordPress Admin → Tools → Theme Editor
// Or via WP-CLI

\ALHCA\TETO\TETO::install();
```

This creates the `wp_teto_documents` table for tracking documents.

### Step 2: Configure AWS Credentials

Add to `wp-config.php`:

```php
// AWS S3 Configuration
define('AWS_DEFAULT_REGION', 'ap-southeast-2');  // Regional setting
define('AWS_BUCKET_NAME', 'your-bucket-name');
define('AWS_ACCESS_KEY_ID', 'YOUR_ACCESS_KEY');
define('AWS_SECRET_ACCESS_KEY', 'YOUR_SECRET_KEY');

// Optional: Upload limits (admins are exempt from rate limits)
define('TETO_MAX_UPLOADS_PER_HOUR', 20);
define('TETO_MAX_FILE_SIZE', 5242880); // 5MB in bytes
```

### Step 3: Configure the First Form

See [Field Mappings](#field-mappings) section below.

### Step 4: Test

1. Submit a form with a file upload
2. Check S3 bucket for uploaded file in `USER_{id}/` folder
3. Check logs at `wp-content/uploads/teto-logs/`
4. View file in TETO Viewer admin panel

---

## Admin Interface

### Accessing TETO

After installation, **TETO** appears in the WordPress admin sidebar with a cloud icon.

### Available Sections

#### 1. TETO Access (Main Dashboard)
- **Dashboard Tab**: AWS status, statistics, quick links
- **Expirations & Mappings Tab**: Configure form expirations and field mappings for S3 upload
- **Document Reports Tab**: View expiring/expired documents
- **Users Files Tab**: Unified file management and approval interface
- **System Info Tab**: Configuration, logs, diagnostics

#### 2. TETO Viewer (Separate Menu Item)
- Browse user folders
- Download files
- View file statistics
- Cache management

### Dashboard Features

**AWS Status Indicator:**
- Green: Connection successful
- Red: Connection failed (check credentials)

**Quick Statistics:**
- Total documents tracked
- Pending approvals count
- Expiring documents (next 30 days)
- Expired documents

**Quick Actions:**
- Add field mapping
- View pending approvals
- Refresh cache
- View logs

---

## Field Mappings

Field mappings tell TETO which Gravity Forms fields contain files to upload to S3.

### Via Admin Interface (Recommended)

1. Navigate to: **TETO → TETO Access → Expirations & Mappings**
2. Click **"Add New Mapping"**
3. Fill in:
   - **Select Form**: Choose the Gravity Form
   - **Field ID**: Enter the file upload field ID number
   - **Document Type**: Select type (License, WWCC, First Aid, etc.)
   - **Custom Filename Template** *(optional)*: Use placeholders like `{user_id}_{last_name}_license.pdf`
4. Click **"Add Mapping"**

### Via Code (Alternative)

Add to `functions.php`:

```php
// Form 29, Field 26 = Anaphylaxis plan
alhca_add_field_mapping(29, 26, 'anaphylaxis_plan');

// Multiple fields in one form
alhca_add_field_mapping(15, 10, 'license');
alhca_add_field_mapping(15, 12, 'wwcc');
alhca_add_field_mapping(15, 14, 'first_aid');
```

### Available Document Types

| Type | Has Expiry | Reminder Days | Subfolder |
|------|-----------|---------------|-----------|
| License | Yes | 30 | documents/license/ |
| WWCC | Yes | 30 | documents/wwcc/ |
| First Aid Certificate | Yes | 30 | documents/first_aid/ |
| Anaphylaxis Action Plan | Yes | 90 | documents/anaphylaxis_plan/ |
| Medical Certificate | Yes | 30 | documents/medical_cert/ |
| Consent Form | No | - | documents/consent/ |
| Photo ID | No | - | documents/photo_id/ |
| Qualification Certificate | No | - | documents/qualification/ |
| Insurance Certificate | Yes | 30 | documents/insurance/ |
| Policy Document | No | - | documents/policy/ |
| Supporting Document | No | - | documents/supporting/ |
| Form CSV Export | Yes | 7 | documents/form_csv/ |
| Other | No | - | documents/other/ |

### Add Custom Document Types

```php
add_filter('teto_document_types', function($types) {
    $types['police_check'] = [
        'label' => 'Police Check',
        'has_expiry' => true,
        'reminder_days' => 60,
        'required' => true
    ];
    return $types;
});
```

### Finding Field IDs

**Method 1**: Edit form in Gravity Forms → Click field → See "Field ID" in settings (top right)

**Method 2**: Look at form URL: `?id=29` (form ID)

**Method 3**: Check TETO logs after form submission

### Supported File Types

- **PDFs** (application/pdf)
- **PNG** images (image/png)
- **JPEG** images (image/jpeg, image/jpg)
- **GIF** images (image/gif)
- **WebP** images (image/webp)
- **DOC/DOCX** (Word documents)
- **CSV** files (text/csv, application/csv) - *For form CSV exports with expiration*

All files are validated with MIME type checks and magic byte verification.

---

## File Approval Workflow

TETO includes a comprehensive approval system where administrators review and approve/reject documents before they become active.

### How It Works

```
1. User submits form with file upload
   ↓
2. File uploaded to S3 with timestamp
   ↓
3. Document status = 'pending' (awaiting approval)
   ↓
4. Appears in admin approval queue
   ↓
5. Admin reviews:
   
   [APPROVE]              [REJECT]
      ↓                       ↓
   Active in system      Moved to Archive/
   Available to user     Rejection email sent
```

### Database Fields

| Column | Description |
|--------|-------------|
| `form_id` | Gravity Form ID |
| `approval_status` | pending/approved/rejected |
| `approval_date` | When reviewed |
| `approved_by` | Admin user ID |
| `rejection_reason` | Why rejected |

### Visual Indicators

When viewing files:
- **Yellow background**: Pending approval
- **Green background**: Approved
- **Red background**: Rejected

### Approving Documents

#### Via Users Files Tab (Primary Method)
1. Navigate to: **TETO → TETO Access → Users Files**
2. Find user with pending files (yellow highlight)
3. Click **"View Profile & Files"**
4. Switch to **"Pending"** tab
5. Click **"Approve"** on document
6. Confirm action

#### Bulk Approval (Via Database)
For existing installations migrating to approval workflow:

```sql
UPDATE wp_teto_documents 
SET approval_status = 'approved', 
    approval_date = NOW(), 
    approved_by = 1 
WHERE approval_status = 'pending' 
AND upload_date < '2026-04-16';
```

### Rejecting Documents

1. Find pending document
2. Click **"Reject"** button
3. Enter rejection reason (required):
   - Be specific and helpful
   - Example: "Certificate has expired. Please upload a current certificate."
4. Click **"Confirm Rejection"**
5. System automatically:
   - Moves file to Archive/ folder in S3
   - Marks as rejected in database
   - Sends rejection email to user

### Rejection Email

Users receive an email containing:
- Document name
- Upload date
- Rejection reason
- Portal link to upload corrected version

**Example:**
```
Subject: Document Rejected: license_20260416_103045.pdf

Dear John Smith,

The uploaded document has been rejected.

Document: license_20260416_103045.pdf
Uploaded: April 16, 2026

Reason for rejection:
Certificate has expired. Please upload a current certificate 
dated within the last year.

ACTION REQUIRED:
Please upload a corrected version.

Portal: https://yoursite.com/portal
```

### Customizing Rejection Emails

```php
// Custom subject
add_filter('teto_rejection_email_subject', function($subject, $document, $user) {
    return "Document Review Required";
}, 10, 3);

// Custom message
add_filter('teto_rejection_email_message', function($message, $document, $user, $reason) {
    return "Custom email body with {$reason}...";
}, 10, 4);
```

### Form-Based Expiration

Configure automatic expiration for specific forms with two options:

1. Navigate to: **TETO → TETO Access → Expirations & Mappings**
2. Click **"Configure Expiration"** for a form
3. Choose expiration type:

#### Option 1: Days After Submission
- Enter number of days (e.g., 30, 90, 365)
- Document expires X days after upload
- Simple and straightforward

#### Option 2: Yearly Expiration with Leeway
- Select a specific date each year (e.g., July 1st for EOFY)
- Set a "leeway" threshold in days (e.g., 60 days)
- **How it works:**
  - If form is submitted within leeway period BEFORE the yearly date, it doesn't expire until NEXT year
  - If submitted after the yearly date, expires on next year's date
  - Perfect for annual compliance documents

**Example Yearly Expiration:**
- Yearly Date: July 1st
- Leeway: 60 days
- Form submitted May 15th (47 days before July 1st, within leeway) → Expires July 1st NEXT year
- Form submitted March 1st (122 days before July 1st, outside leeway) → Expires July 1st THIS year
- Form submitted August 1st (after July 1st) → Expires July 1st next year

**Use Cases:**
- EOFY compliance (July 1st expiration)
- Annual training renewals (specific month/day)
- Calendar year documents (January 1st expiration)

**Automatic Calculation:**
```
// Days-based
expiry_date = upload_date + configured_days

// Yearly-based (with leeway logic)
expiry_date = calculate_next_occurrence(yearly_date, upload_date, leeway_days)
```

### CSV Export Upload Feature

Forms with expiration configured automatically upload their CSV exports to S3, ensuring complete record retention even for forms without file uploads or PDF generation.

#### How It Works

1. **Configure Form Expiration** (either days or yearly)
2. **Form generates CSV** via SubmissionToCSV class
3. **TETO automatically uploads** CSV to S3
4. **Tracks with expiration** based on form configuration
5. **Renamed meaningfully**: `FormTitle_UserName_timestamp.csv`

#### Example Flow

**Without CSV Upload:**
```
User submits Form 39 (HR Medical Declaration)
   ↓
Form creates CSV: 3167_1777252363.csv
   ↓
CSV attached to email notification
   ↓
CSV deleted after email sent
   ↓
No permanent record 
```

**With CSV Upload (TETO):**
```
User submits Form 39 (HR Medical Declaration)
   ↓
Form creates CSV: 3167_1777252363.csv
   ↓
TETO detects form has expiration config
   ↓
CSV uploaded to S3: HR_Medical_Declaration_John_Smith_1777252363.csv
   ↓
Tracked in database with expiry date
   ↓
CSV attached to email notification
   ↓
Local CSV deleted (S3 copy preserved)
```

#### Requirements

- Form must have **expiration configured** (days or yearly)
- Form must use **SubmissionToCSV** class for CSV generation
- User must be identified (created_by field or user field in form)

#### Filename Format

```
{FormTitle}_{UserFirstName}_{UserLastName}_{timestamp}.csv
```

**Examples:**
- `HR_Medical_Declaration_John_Smith_1777252363.csv`
- `Employee_Onboarding_Sarah_Johnson_1777260000.csv`
- `Annual_Review_Form_Mike_Chen_1777268000.csv`

#### S3 Storage Location

```
USER_{id}/documents/form_csv/FormTitle_UserName_timestamp.csv
```

#### Document Tracking

- **Document Type**: `form_csv`
- **Has Expiry**: Yes
- **Reminder Days**: 7
- **Metadata Includes**:
  - Form ID and title
  - Entry ID
  - Original filename
  - Expiration type (days/yearly)

#### Benefits

- **Complete Record Retention**: Never lose form submissions
- **Automatic**: No manual intervention required
- **Searchable**: CSVs stored with meaningful names
- **Compliant**: Maintains audit trail
- **Organized**: Stored in user-specific folders
- **Tracked**: Expiration and reminders apply

#### Validation

CSV files undergo same security validation:
- MIME type check (text/csv, application/csv, etc.)
- File existence verification
- User authentication
- Rate limiting (admins exempt)

#### Viewing CSV Exports

1. Navigate to: **TETO → TETO Access → Users Files**
2. Find user and click **"View Profile & Files"**
3. CSVs appear in file list with `form_csv` type
4. Download via presigned URL (5-minute expiry)

---

## Archiving System

TETO includes intelligent version-based archiving to keep the S3 bucket organized while maintaining complete audit trails.

### Automatic File Timestamping

**Every** uploaded file receives a timestamp:
```
original_filename_YYYYMMDD_HHMMSS.ext
```

**Examples:**
- `license_20260416_143022.pdf`
- `Job Description - John Smith_20260416_034602.pdf`

This ensures:
- No file overwrites
- Complete audit trail
- Easy chronological sorting
- Unique identifiers

### Version-Based Archiving (Automatic)

When a new version of a document is uploaded:

1. System extracts base filename (removes our timestamp)
2. Searches for existing files with same base name
3. Automatically moves older versions to `Archive/` folder
4. Keeps only newest version in active folder

**Example Flow:**

**Month 1 (January):**
```
Upload: "license.pdf"
Saved as: USER_123/license_20260116_103000.pdf
```

**Month 2 (February) - Same document updated:**
```
Upload: "license.pdf"
Saved as: USER_123/license_20260216_140000.pdf
Auto-archived: USER_123/Archive/license_20260116_103000.pdf
```

**Month 3 (March) - Another update:**
```
Upload: "license.pdf"
Saved as: USER_123/license_20260316_091500.pdf
Archive contains:
  - license_20260116_103000.pdf
  - license_20260216_140000.pdf
```

### S3 Bucket Structure

```
the-bucket/
├── USER_123/
│   ├── license_20260416_120000.pdf (current)
│   ├── wwcc_20260416_130000.pdf (current)
│   └── Archive/
│       ├── license_20260115_090000.pdf (previous)
│       ├── license_20251201_080000.pdf (older)
│       └── wwcc_20251115_100000.pdf (previous)
├── USER_456/
│   ├── first_aid_20260416_140000.pdf (current)
│   └── Archive/
│       └── first_aid_20251201_110000.pdf (previous)
```

**Benefits:**
- Automatic - no manual intervention
- Smart - only archives when newer version uploaded
- Clean - active folder shows only current files
- Safe - all previous versions preserved
- Legal - complete audit trail maintained

### Expired Document Archiving

Documents marked as expired are automatically moved to Archive/:
- Daily cron checks for expired documents
- Moves to Archive/ folder
- Updates database status
- Maintains access for admins

### Age-Based Archiving (Optional)

**Default**: Disabled (version-based archiving is primary)

To enable:
```php
add_filter('teto_enable_age_based_archiving', '__return_true');
```

Configure threshold:
```php
// Archive documents older than 90 days
update_option('teto_archive_after_days', 90);
```

### Legal Compliance

- **No File Deletion**: Files are moved, never deleted
- **Complete Audit Trail**: All timestamps preserved
- **Traceability**: Document tracker maintains history
- **Version History**: All previous versions accessible
- **Access Control**: Archive files maintain security permissions

---

## Authorization & Access Control

TETO implements comprehensive authorization to control who can view, download, and delete files.

### Authorization Rules

When a user attempts to access files, TETO checks:

1. **Own Files?** Always allowed
2. **Administrator?** Full access to all files (with logging)
3. **Explicit Permission?** Users can be granted access to specific users' files
4. **None of above?** Denied and logged

**Note:** Administrators are exempt from rate limiting (20 uploads/hour, 40 downloads/hour, 40 presigned URLs/hour).

### Protected Operations

#### 1. Viewing/Downloading Files
```php
$teto = TETO::get_instance();
$url = $teto->get_presigned_url($entry_id);
// Returns false if user lacks permission
```

Checks:
- User has permission to view entry owner's files
- Logs unauthorized attempts
- Rate limits: 40 presigned URLs per hour (admins exempt)

#### 2. Listing User Files
```php
$teto = TETO::get_instance();
$files = $teto->get_user_files($user_id);
// Returns false if user lacks permission
```

#### 3. Deleting Files
```php
$teto = TETO::get_instance();
$result = $teto->delete_file($entry_id);
// Only admins and file owners can delete
```

### Permission Management

#### Grant File Access (Admin Only)
```php
$teto = TETO::get_instance();

// Grant user 456 permission to view user 789's files
$teto->grant_file_access_permission(456, 789);
```

#### Revoke File Access (Admin Only)
```php
$teto = TETO::get_instance();

// Revoke user 456's permission
$teto->revoke_file_access_permission(456, 789);
```

#### Check Permissions
```php
$teto = TETO::get_instance();

// Get list of users that user 456 can view files for
$permitted_users = $teto->get_user_file_access_permissions(456);
// Returns: [789, 123, 456]

// Check if specific access is allowed
$can_view = $teto->can_user_view_user_files(456, 789);
```

### Use Cases

**Manager viewing employee files:**
```php
// Admin grants manager permission
$teto->grant_file_access_permission($manager_id, $employee_id);

// Now manager can access
$files = $teto->get_user_files($employee_id); // Works!
```

**HR department access:**
```php
// Grant HR access to multiple employees
$hr_user_id = 123;
$employees = [456, 789, 1011, 1213];

foreach ($employees as $employee_id) {
    $teto->grant_file_access_permission($hr_user_id, $employee_id);
}
```

**Temporary access:**
```php
// Grant for audit
$teto->grant_file_access_permission($auditor_id, $user_id);

// ... audit happens ...

// Revoke after completion
$teto->revoke_file_access_permission($auditor_id, $user_id);
```

### Security Logging

All authorization checks are logged:
- Successful access (with details)
- Denied attempts (with IP, user IDs, action)
- Permission grants/revocations

Logs include:
- Requesting user ID
- Target user ID
- IP address
- Timestamp
- Action attempted
- Result

### Data Storage

Permissions stored in WordPress user meta:
```php
// Meta key: teto_file_access_permissions
// Value: Array of user IDs

get_user_meta(456, 'teto_file_access_permissions', true);
// Returns: [789, 1011, 1213]
```

---

## Filename Templates

Filename templates automatically rename uploaded files using form data.

### Available Placeholders

#### System Placeholders

| Placeholder | Output Example |
|-------------|----------------|
| `{entry_id}` | 2997 |
| `{user_id}` | 2886 |
| `{form_id}` | 29 |
| `{date}` | 2026-04-14 |
| `{date_dmy}` | 14-04-2026 |
| `{date_mdy}` | 04-14-2026 |
| `{timestamp}` | 1713055200 |
| `{index}` | 0, 1, 2 (0-based) |
| `{index1}` | 1, 2, 3 (1-based) |

#### Form Field Placeholders

**Using Field IDs:**
- `{1}` - Value from field ID 1
- `{26}` - Value from field ID 26

**Using Field Labels:**
- `{first_name}` - Field labeled "First Name"
- `{last_name}` - Field labeled "Last Name"
- `{child_name}` - Field labeled "Child Name"

Labels are converted to lowercase with underscores.

### Template Examples

#### Simple Name + Type
```
Template: {first_name}_{last_name}_license.pdf
Output: john_smith_license.pdf
```

#### User ID + Date
```
Template: {user_id}_anaphylaxis_{date}.pdf
Output: 2886_anaphylaxis_2026-04-14.pdf
```

#### Multi-File Upload
```
Template: {child_name}_photo_{index1}.jpg
Output: emma_photo_1.jpg, emma_photo_2.jpg, emma_photo_3.jpg
```

#### Professional Format
```
Template: {user_id}_{last_name}_{first_name}_wwcc_{date}.pdf
Output: 2886_smith_john_wwcc_2026-04-14.pdf
```

### Setting Up Templates

1. Go to: **TETO → Expirations & Mappings**
2. Add or edit mapping
3. Enter template in **"Custom Filename Template"** field
4. Click **"Add Mapping"** or **"Update"**

### Character Sanitization

TETO automatically cleans filenames:
- Spaces → `_` (underscores)
- Invalid characters removed: `/ \ : * ? " < > |`
- Multiple underscores → single `_`
- Length limited to 200 characters

### Best Practices

1. **Include Unique Identifier**
   ```
   {user_id}_{last_name}_license.pdf
   ```

2. **Add Dates for Time-Sensitive Docs**
   ```
   {last_name}_{first_name}_license_{date}.pdf
   ```

3. **Plan for Multi-Uploads**
   ```
   {child_name}_document_{index1}.pdf
   ```

4. **Keep It Simple**
   - Good: `{user_id}_{last_name}_license.pdf`
   - Too complex: `{org}_{dept}_{first}_{middle}_{last}_{suffix}_{type}_{date}_{version}.pdf`

---

## Users Files Management

The **Users Files** tab is a unified interface for managing all user files and approvals.

### Overview

**Replaces previous tabs:**
- File Viewer → Now integrated
- File Approval → Now user-centric

**Scales to:** Thousands of users with efficient search and filtering

### User List View

Shows all users with uploaded files as visual cards:

**Card Information:**
- User avatar
- Display name
- Email
- User ID
- Pending approval count (yellow highlight)
- Total file count
- User roles
- "View Profile & Files" button

**Visual Indicators:**
- Yellow left border = Has pending files
- Orange background = Orphaned user (no WP account)

### Search & Filter

**Search Bar:**
- Search by name, email, or user ID
- Real-time filtering
- Clear button

**Filter Options:**
- All Users
- Users with Pending Files (quick approval queue)
- Users without Pending Files
- Orphaned Users

**Sort Options:**
- Most Pending First (default - prioritize approvals)
- Least Pending First
- Name (A-Z / Z-A)
- Most Files First
- Least Files First

### User Profile Modal

Click **"View Profile & Files"** to open user's complete profile:

**Profile Header:**
- Large avatar
- Name, email, ID
- Role badges

**Statistics:**
- Total Files
- Pending Approval
- Approved
- Rejected

**File Tabs:**
1. **Pending**: Files awaiting approval with Approve/Reject buttons
2. **All Files**: All S3 files with download links
3. **Approved**: Previously approved documents
4. **Rejected**: Rejected documents with reasons

**Quick Actions:**
- Approve documents
- Reject with reason
- Download files
- Copy download URLs

### Approval Workflow

**Approve:**
1. Open user profile
2. Switch to "Pending" tab
3. Click "Approve" button
4. Confirm
5. Document status updates
6. Counts refresh automatically

**Reject:**
1. Click "Reject" button
2. Modal appears
3. Enter rejection reason (required)
4. Confirm
5. File moves to Archive/
6. Rejection email sent to user
7. Counts refresh

### Pagination

- Shows 20 users per page
- Previous/Next navigation
- Automatically resets to page 1 on search/filter

### Form-Based Expiration Settings

At bottom of Users Files tab:

1. Shows all Gravity Forms
2. Enter expiration days for each form
3. Click "Save" button
4. Forms without expiration show "N/A"

**Example:**
- Form #61: 30 days
- Form #45: 90 days
- Form #12: 0 (disabled)

---

## TETO Viewer

Separate admin interface for browsing S3 folders and downloading files.

### Accessing Viewer

Navigate to: **WordPress Admin → TETO Viewer** (separate menu item)

### First Load

- Fetches all user folders from S3
- Takes 5-30 seconds depending on folder count
- Data cached for 1 hour
- Shows progress indicator

### Dashboard

**Statistics:**
- Total users with folders
- Total files stored
- Orphaned folders (users deleted from WP)

**Cache Status:**
- Shows if cache is active
- Time since last update
- Expiry countdown
- Manual refresh button

### User List

**Table Columns:**
- User ID
- Display Name
- Folder (S3 path)
- File Count
- Last Modified
- Actions (View Files button)

**Features:**
- Search bar (client-side, no S3 calls)
- Sortable columns
- Paginated (configurable items per page)

### Viewing Files

Click **"View Files"** on any user:

**Modal Shows:**
- User name and folder path
- List of all files with:
  - File name
  - File size (formatted: KB, MB)
  - Last modified date
  - Download button (5-minute presigned URL)
  - Copy URL button

**Features:**
- Opens in new tab
- URL copied to clipboard
- URLs expire after 5 minutes (security)

### Caching System

**Benefits:**
- Reduces S3 API calls by ~95%
- Potential savings: $50-100/month for high-traffic sites
- Faster page loads

**Cache Duration:**
- User folder listings: 1 hour
- File count queries: 1 hour
- Statistics: 2 hours

**Manual Refresh:**
- Click "Refresh Cache" button
- Takes 5-30 seconds
- Page auto-reloads with new data

**Cache Status Indicators:**
```
✓ Cache Active
Updated 15 minutes ago • Expires in 45 minutes
```

```
⚠ No Cache
Data will be loaded from S3
```

### Search Function

Type in search bar to filter by:
- Display name: "John Smith"
- Username: "jsmith"
- Email: "john@example.com"
- User ID: "2886"

Results filter instantly (client-side, no new S3 calls).

---

## Security Features

TETO implements multi-layer security to protect files and data.

### File Validation

**Before Upload:**
1. File exists
2. MIME type validation
3. PDF magic byte verification (`%PDF-`)
4. Image magic byte verification
5. Office document validation
6. File size within limits (5MB default)
7. Header content scan (first 1KB)

**Allowed File Types:**
- application/pdf
- image/jpeg, image/jpg, image/png, image/gif, image/webp
- application/msword, application/vnd.openxmlformats-officedocument.wordprocessingml.document

### Rate Limiting

**Current Limits:**
| Action | Non-Admins | Admins |
|--------|-----------|--------|
| Uploads | 20/hour | Exempt |
| Downloads (Presigned URLs) | 40/hour | Exempt |
| Presigned URL Generation | 40/hour | Exempt |

Administrators with `manage_options` capability are completely exempt from all rate limits.

**Configuration:**
```php
// In wp-config.php
define('TETO_MAX_UPLOADS_PER_HOUR', 20);
```

**Monitoring:**
- Violations logged with IP address
- User ID and timestamp recorded
- Automatic blocking after threshold

### IP-Based Security

**Features:**
- Proxy-aware IP detection
- IP blocking capability
- All operations log IP address

**Block IP:**
```php
$security = $teto->get_security();
$security->block_ip($ip_address, 'Reason for blocking');
```

### User Folder Isolation

**Enforced Paths:**
- Each user has isolated folder: `USER_{id}/`
- Path traversal attempts blocked
- No cross-user access without permission

**Protection Against:**
- `../` sequences
- `..\\` sequences
- Null bytes
- Absolute paths

### Encryption

**Server-Side Encryption (AES256):**
- All uploads encrypted at rest
- AWS-managed keys (no extra cost)
- Optional KMS support

**Enable KMS:**
```php
define('TETO_AWS_KMS_KEY_ID', 'your-kms-key-id');
```

**HTTPS Enforcement:**
- All API calls use SSL/TLS
- Presigned URLs are HTTPS
- No unencrypted transport

### Security Logging

**Logged Events:**
- All file uploads (with user, IP, filename, size)
- File downloads
- Approval/rejection actions
- Failed validation attempts
- Rate limit violations
- Unauthorized access attempts

**Log Location:** `wp-content/uploads/teto-logs/teto-YYYY-MM-DD.log`

**WP Activity Log Integration:**
- Successful uploads logged (Event ID 5700)
- Includes file name, S3 key, file size, MIME type
- User ID and IP address
- Object URL and version ID

### IAM Best Practices

**Recommended IAM Policy:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::bucket-name",
                "arn:aws:s3:::bucket-name/*"
            ]
        }
    ]
}
```

**Key Points:**
- Never use root credentials
- Create dedicated IAM user
- Minimal permissions (least privilege)
- Resource-scoped to specific bucket

### S3 Bucket Security

**Recommended Bucket Policy:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnencryptedUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::bucket-name/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "AES256"
                }
            }
        },
        {
            "Sid": "DenyInsecureTransport",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::bucket-name",
                "arn:aws:s3:::bucket-name/*"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```

**Block Public Access:**
Enable all 4 settings:
- Block public access granted through new ACLs
- Block public access granted through any ACLs
- Block public access granted through new bucket policies
- Block public access granted through any bucket policies

### Recommended Additional Security

**1. Enable S3 Versioning:**
- Protects against accidental deletion
- Can recover deleted files
- Minimal cost impact

**2. Enable CloudTrail:**
- Logs all S3 API calls
- Security auditing
- Compliance requirements
- Approximately $0.10 per 100,000 events

**3. Enable S3 Access Logging:**
- Free to enable
- Tracks all bucket access
- Helps identify unusual patterns

**4. CloudWatch Alarms:**
- Alert on excessive uploads
- Alert on failed authentication
- Alert on unusual delete activity

### Attack Scenarios & Mitigations

**Compromised IAM Credentials:**
- Scoped permissions limit damage
- Rate limiting prevents mass upload
- Bucket policy blocks dangerous actions
- CloudTrail logs all activity

**Malicious File Upload:**
- MIME type validation
- Magic byte verification
- Content scanning
- Consider: ClamAV integration

**S3 Bucket Enumeration:**
- Private ACL
- No public access
- Presigned URLs with short expiry

**Path Traversal:**
- User folder isolation
- Sanitized S3 keys
- No user input in paths

**DoS via Storage:**
- 5MB file size limit
- 20 uploads/hour rate limit (non-admins)
- Consider: S3 Lifecycle policy

---

## AWS Cost Optimization

TETO implements several cost-saving strategies.

### Implemented Optimizations

#### 1. Hourly Caching System
**Savings:** ~95% reduction in S3 API calls

**What's Cached:**
- User folder listings: 1 hour
- File count queries: 1 hour
- Statistics: 2 hours

**Cost Impact:**
- S3 ListObjectsV2: $0.005 per 1,000 requests
- Potential savings: $50-100/month for high-traffic sites

#### 2. Short Presigned URL Expiry
**Setting:** 5 minutes

**Benefits:**
- Reduces security risk
- Prevents long-lived URL abuse
- On-demand generation only

#### 3. Limited MaxKeys
**Setting:** 1000 items per S3 list operation

**Benefits:**
- Prevents excessive API usage
- Caps data transfer per request

#### 4. Private ACL
**Setting:** All uploads use `ACL='private'`

**Benefits:**
- Prevents unauthorized public access
- Reduces bandwidth from hotlinking
- Forces access through presigned URLs

#### 5. Rate Limiting
**Settings:**
- Admin file viewing: 50 requests per 5 minutes
- General operations: User-specific limits

**Benefits:**
- Prevents abuse
- Protects against DoS
- Limits API call costs

### Recommended Optimizations

#### 6. S3 Lifecycle Policies
**Savings:** 40-70% on storage costs

**Setup:** AWS Console → S3 → Bucket → Management → Lifecycle Rules

**Example Policy:**
```json
{
  "Rules": [
    {
      "Id": "ArchiveOldFiles",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 180,
          "StorageClass": "GLACIER"
        }
      ]
    },
    {
      "Id": "DeleteTempFiles",
      "Filter": {
        "Prefix": "temp/"
      },
      "Expiration": {
        "Days": 7
      }
    }
  ]
}
```

**Storage Class Costs:**
- **S3 Standard:** $0.023/GB
- **S3 Standard-IA:** $0.0125/GB (46% cheaper)
- **S3 Glacier:** $0.004/GB (83% cheaper)

#### 7. Intelligent-Tiering
**Auto-optimizes** storage class based on access patterns

**Setup:**
```json
{
  "Id": "IntelligentTier",
  "Status": "Enabled",
  "Transitions": [
    {
      "Days": 0,
      "StorageClass": "INTELLIGENT_TIERING"
    }
  ]
}
```

**Cost:** Small monitoring fee (~$0.0025 per 1,000 objects)

#### 8. S3 Request Metrics
**Monitor** actual request costs

**Setup:** S3 → Metrics → Request Metrics

**Benefits:**
- Identify high-cost operations
- Track API call patterns
- Optimize based on real data

### Cost Monitoring

**CloudWatch Metrics:**
- BucketSizeBytes
- NumberOfObjects
- AllRequests
- GetRequests
- PutRequests

**Set Budget Alerts:**
```
AWS Budgets → Create Budget
- Budget Type: Cost
- Budget Amount: $10/month
- Alert Threshold: 80%
```

### Estimated Monthly Costs

**Typical Usage (100 users):**
- Storage: 1GB → $0.023
- PUT requests: 2,000 → $0.01
- GET requests (with caching): 5,000 → $0.002
- Data transfer out: 1GB → $0.09
- **Total:** ~$0.13/month

**With Caching Disabled:**
- GET requests: 100,000 → $0.04
- **Total:** ~$0.17/month (+30%)

**With Lifecycle Policies:**
- Storage moved to IA after 90 days: -46%
- **Total:** ~$0.09/month (-31%)

### Best Practices

1. **Enable Caching** (Already active)
2. **Set Lifecycle Policies** for old files
3. **Monitor Request Metrics** monthly
4. **Use Intelligent-Tiering** for unpredictable access
5. **Delete Unused Test Files** regularly
6. **Compress Large Files** before upload (if applicable)
7. **Set Budget Alerts** in AWS

---

## Troubleshooting

### Common Issues

**Files Not Uploading:**
1. Check AWS credentials in wp-config.php
2. Verify IAM permissions
3. Check TETO logs for errors
4. Confirm field mapping exists
5. Verify file size within limits

**"File not found for intake":**
- Field mapping not configured
- Wrong field ID
- Gravity Forms upload failed
- Check PHP upload_max_filesize

**"Access denied" errors:**
- Check S3 bucket policy
- Verify IAM permissions
- Ensure bucket exists
- Check region matches

**Approval workflow not showing:**
1. Run database migration: `TETO::install()`
2. Check if columns exist: `wp_teto_documents`
3. Clear browser cache
4. Check user has admin permissions

**Emails not sending:**
1. Test WordPress wp_mail()
2. Install WP Mail SMTP plugin
3. Check user has valid email
4. Review TETO logs for errors

**Cache not refreshing:**
1. Clear WordPress object cache
2. Manually click "Refresh Cache"
3. Check transients in database
4. Verify cron is running

### Log Files

**Location:** `wp-content/uploads/teto-logs/`

**Format:** `teto-YYYY-MM-DD.log`

**Log Levels:**
- INFO: Normal operations
- SUCCESS: Successful uploads
- WARNING: Rate limits, validation issues
- ERROR: Failed operations
- DEBUG: Detailed debugging info

**Enable Debug Logging:**
```php
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
```

### Support Resources

**Documentation:**
- README_TETO.md - Plugin overview
- TECHNICAL_GUIDE.md - Complete technical reference

**AWS Resources:**
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [S3 Pricing](https://aws.amazon.com/s3/pricing/)

**WordPress:**
- Gravity Forms Documentation
- Gravity PDF Documentation

---

## Appendix

### Quick Command Reference

```php
// Install/update database
\ALHCA\TETO\TETO::install();

// Add field mapping
alhca_add_field_mapping($form_id, $field_id, $document_type);

// Get user files
$teto = TETO::get_instance();
$files = $teto->get_user_files($user_id);

// Generate presigned URL
$url = $teto->get_presigned_url($entry_id);

// Grant file access permission
$teto->grant_file_access_permission($user_id, $target_user_id);

// Revoke file access permission
$teto->revoke_file_access_permission($user_id, $target_user_id);

// Approve document
$tracker = $teto->get_document_tracker();
$tracker->approve_document($document_id, $admin_user_id);

// Reject document
$tracker->reject_document($document_id, $admin_user_id, $reason);
```

### Filters & Hooks

```php
// Customize document types
add_filter('teto_document_types', function($types) {
    // Add custom types
    return $types;
});

// Customize allowed file types
add_filter('teto_allowed_upload_mime_types', function($types) {
    // Add custom MIME types
    return $types;
});

// Customize portal URL
add_filter('teto_portal_url', function($url) {
    return home_url('/portal');
});

// Customize rejection email
add_filter('teto_rejection_email_subject', function($subject, $doc, $user) {
    return 'Custom Subject';
}, 10, 3);

// After file uploaded to S3
add_action('teto_file_uploaded_to_s3', function($data) {
    // Custom logic after upload
}, 10, 1);

// Enable age-based archiving
add_filter('teto_enable_age_based_archiving', '__return_true');
```

### Configuration Constants

```php
// AWS credentials
define('AWS_DEFAULT_REGION', 'ap-southeast-2');
define('AWS_BUCKET_NAME', 'your-bucket');
define('AWS_ACCESS_KEY_ID', 'YOUR_KEY');
define('AWS_SECRET_ACCESS_KEY', 'YOUR_SECRET');

// Optional configurations
define('TETO_MAX_UPLOADS_PER_HOUR', 20);
define('TETO_MAX_FILE_SIZE', 5242880); // 5MB
define('TETO_ARCHIVE_AFTER_DAYS', 90);
define('TETO_AWS_KMS_KEY_ID', 'your-kms-key'); // Optional KMS
define('TETO_ENABLE_ENCRYPTION', true); // Optional
```

---

## Updates & Changelog

### Version 0.2.22
- Fixed visual bugs in Field Mappings tab (empty forms no longer shown)
- Added yearly expiration with leeway threshold feature
  - Configure forms to expire on specific date each year (e.g., July 1st for EOFY)
  - Leeway threshold prevents premature expiration for submissions near target date
  - Full UI with radio buttons, month/day selectors, and leeway input
- CSV automatic upload for forms with expiration
  - Forms with expiration configured automatically upload CSV exports to S3
  - CSVs renamed with form title and user name for easy identification
  - Tracked as `form_csv` document type with expiration
  - Complete record retention even without file uploads or PDF generation
- Added CSV mime types to TETOSecurity validation layer
- Fixed DateTime namespace issue in expiration display

### Version 0.2.21
- Notification Logging Added
- Email Notifications for Expiration, Reminder, Rejection, ect all working.

### Version 0.2.20
- Fixed Bug with Expirations applying to Non-Expiring Uploads
- Fixed Email Reminders for Pending & Actual Expirations

### Version 0.2.19
- Bug Fixes regarding JavaScript & Naming System
- Fixed DB to S3 Auditing Process

### Version 0.2.18
- Add Edit Field Mapping
- Add Edit File Name
- Field Mapping Name Template Sanatization

### Version 0.2.17
- Make Upload Field Mappings more readable
- All Expirations send reminder email at 30 days.
- Fix minor bugs with field mapping

### Version 0.2.16
- Rotation Reminder
- Deep Freeze Option for Old Archived Files

### Version 0.2.15
- Removed Form Restrictions

### Version 0.2.14
- Restrictions to Non-Logged in Users
- Form Exemptions from this Process
- Cleaned up Mappings
- Added GF Shortcode Mapper

### Version 0.2.13
- Better Syncing between Database & S3

### Version 0.2.12
- Added two tier Plugin Loading
- Memory Usage Optimizations
- AWS SDK On-Demand Loading Added.
- ~80% Memory Savings Achieved

### Version 0.2.11
- Fixed AJAX Issues
- Added admin upload to user files
- Fixed Folder Collapsing Issues

### Version 0.2.6-0.2.10
- Various Bug Fixes

### Version 0.2.5
- Added WP Activity Log integration for S3 uploads
- Updated rate limiter: 20 uploads/hour, 40 downloads/hour, 40 presigned/hour
- Administrators now exempt from all rate limiting
- Added `teto_file_uploaded_to_s3` action hook

### Version 0.2.4
- Added Users Files unified management interface
- User-centric approval workflow
- Enhanced search and filtering
- Scalable to thousands of users

### Version 0.2.3
- Added filename templates with placeholders
- Multi-file upload support
- Custom field-based file naming

### Version 0.2.2
- Added file approval workflow
- Form-based expiration configuration
- Rejection email notifications

### Version 0.2.1
- Added authorization system
- File access permissions
- Admin file viewing controls

## Version 0.2.0
- Version-based archiving
- Automatic file timestamping
- Enhanced TETO Viewer with caching
- Update 0.2.0 Represents the first 'Functional Version' of this Plugin

---

**End of User Guide**

For technical implementation details, see [TECHNICAL_GUIDE.md](TECHNICAL_GUIDE.md)
