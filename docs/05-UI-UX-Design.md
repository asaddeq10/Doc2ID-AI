```markdown
# UI/UX Design Specification Document

**Project:** Doc2ID AI  
**Document Version:** 1.0 (Enterprise Specification)  
**Status:** Pending Approval  
**Author:** Lead Product Designer & Senior UX Architect  

---

## 1. DOCUMENT INFORMATION
*   **Title:** Doc2ID AI Enterprise UI/UX Design Specification
*   **Purpose:** Definitive visual and interaction design guide for the Doc2ID AI SaaS platform.
*   **Audience:** Frontend Engineers, UI/UX Designers, QA Testers.

## 2. DESIGN PRINCIPLES
*   **Simplicity:** Abstract complex backend operations (e.g., massive asynchronous queuing) behind clean, linear user flows. Reduce cognitive load using progressive disclosure.
*   **Accessibility (WCAG 2.1 AA):** Ensure the platform is usable by everyone, strictly adhering to contrast ratios, keyboard navigation, and semantic HTML for screen readers.
*   **Consistency:** Utilize a strict, token-based Design System (implemented via Tailwind CSS) so the application feels cohesive across every module.
*   **Responsive Design:** Desktop-first for complex tasks (Template Editor, Data Grids), but gracefully degrading to view-only mode on mobile devices.
*   **Performance:** UI must feel instantaneous. Use optimistic UI updates and skeleton loaders instead of blocking spinners.
*   **Professional SaaS Appearance:** Establish trust through a clean, high-contrast, modern B2B aesthetic.

## 3. BRAND IDENTITY & TOKENS
The platform utilizes a modern, trustworthy palette aimed at B2B enterprise users.

*   **Color Palette (Tailwind Tokens):**
    *   **Primary (Brand/Action):** `blue-600` (#2563eb) base, `blue-700` (#1d4ed8) hover, `blue-50` (#eff6ff) for active backgrounds.
    *   **Success:** `emerald-500` (#10b981) - Used for completed batches and valid data.
    *   **Warning:** `amber-500` (#f59e0b) - Used for missing photos or incomplete data rows.
    *   **Danger/Error:** `red-500` (#ef4444) - Used for failed jobs and destructive actions.
    *   **Neutral (Surface/Text):** `gray-900` (#111827) primary text, `gray-500` (#6b7280) secondary text, `gray-50` (#f9fafb) app background, `white` (#ffffff) card surface.
*   **Typography:** `Inter` (sans-serif) for high legibility in data-dense tables.
*   **Icon Style:** `Lucide React` (clean, 2px stroke, outline style).
*   **Borders & Radius:** `border-gray-200` for dividers. `rounded-lg` (8px) for cards/modals, `rounded-md` (6px) for inputs/buttons.
*   **Shadows:** `shadow-sm` for standard cards, `shadow-xl` for dropdowns/modals.

## 4. USER ROLES (UX CONTEXT)
The UI dynamically adapts based on the active user's role:
*   **Super Admin:** Can see system-wide analytics, manage global templates, and assume identity into specific tenant accounts for support.
*   **Organization Admin (Owner/Admin):** Has access to the `Settings` and `Billing` tabs. Can invite users and configure retention policies.
*   **Staff/User (Editor/Viewer):** Cannot see `Settings` or `Billing`. Editors focus purely on the `Projects` workspace. Viewers only see dashboards and `Downloads`.

## 5. COMPLETE USER JOURNEY (THE "HAPPY PATH")
1.  **Registration:** User signs up.
2.  **Email Verification:** Magic link verifies email.
3.  **Onboarding:** User names their Organization.
4.  **Dashboard:** User lands on the main dashboard. Clicks "New Project".
5.  **Create Project:** Names project "2026 Staff IDs".
6.  **Upload Spreadsheet:** Drops a `.xlsx` file. UI shows a data grid preview.
7.  **Upload Photos:** Drops a `.zip` of photos. UI auto-matches based on filenames.
8.  **Create/Select Template:** Chooses a pre-built ID template.
9.  **Map Fields:** Drags data fields (e.g., `First Name`) onto the template canvas.
10. **Generate IDs:** Clicks "Generate". Progress bar appears.
11. **Download:** Batch completes. User downloads a single multi-page PDF.

## 6. SCREEN SPECIFICATIONS

### 6.1 Authentication (Login / Register / Forgot Password)
*   **Layout:** Split screen. Left 50% is a branded graphic/illustration. Right 50% is a clean white container with the auth form.
*   **UX:** Password visibility toggle. "Remember Me" checkbox. Clear error validation below fields.

### 6.2 Dashboard
*   **Header:** Greeting + Quick Stats (Total Cards Generated, Storage Used).
*   **Body:** "Recent Projects" table and "Recent Downloads" list.
*   **Action:** Primary "New Project" floating action button.

### 6.3 Projects List
*   **View:** Paginated data table. Search bar. Filter by Status dropdown.
*   **Row Actions:** Context menu (three dots) for Edit, Duplicate, Delete.

### 6.4 Project Workspace (Stepper Flow)
A horizontal stepper component sits at the top: `1. Data` ➔ `2. Photos` ➔ `3. Design` ➔ `4. Generate`.

### 6.5 Spreadsheet Import (Step 1)
*   **Upload:** Massive dashed dropzone.
*   **Data Grid:** Interactive table. Sticky headers. Soft-red highlighting on rows missing required fields.

### 6.6 Photo Upload & Match (Step 2)
*   **Upload:** ZIP dropzone.
*   **Split Pane:** Left side shows unmatched thumbnails. Right side shows data rows. Auto-match button animates thumbnails into rows.

### 6.7 Template Editor (Step 3) - *See Section 8*

### 6.8 Generation Progress (Step 4)
*   **View:** Real-time circular progress indicator. Displays estimated time remaining. Button to "Cancel/Pause" job.

### 6.9 Downloads
*   **View:** List of finalized files. Expiration badge (e.g., "Expires in 3 days"). Download buttons.

### 6.10 Settings & Billing
*   **Tabs:** Profile, Organization, Team Members, API Keys, Billing.
*   **Team:** Input to invite via email, dropdown for Role assignment.

### 6.11 Notifications Center
*   **Purpose:** To provide a centralized, asynchronous hub where users can track critical system events, background task completions, and administrative alerts without interrupting their active workflow.
*   **Notification Categories:**
    *   Completed generation jobs (with direct download links).
    *   Failed imports (with error logs).
    *   Photo matching issues (e.g., "300 photos could not be matched").
    *   Team invitations (Accept/Decline).
    *   Billing reminders (e.g., "Quota at 90%").
    *   Security alerts (e.g., "New login from unverified IP").
    *   System maintenance announcements.
*   **Priority Levels (Visual Indicators):**
    *   *Info (Blue):* General updates, team invites.
    *   *Success (Green):* Completed generations, successful data imports.
    *   *Warning (Amber):* Quota limits, partial matching success.
    *   *Error (Red):* Failed jobs, billing failures.
*   **State Management & Actions:**
    *   Distinct visual styling for Unread (bold text, slight blue background tint, active indicator dot) vs. Read (normal text, white background) states.
    *   Hovering over a notification reveals a "Mark as Read" action.
    *   A global "Mark All as Read" button located in the header.
*   **Filtering and Search:**
    *   Tabs to filter by: "All", "Unread", "System", "Jobs".
    *   A search bar to quickly find specific project notifications.
*   **Real-time Delivery:**
    *   Notifications arrive instantly via WebSockets (or Server-Sent Events). A badge on the bell icon in the top navigation increments immediately.
*   **History and Retention:**
    *   Notifications are retained for 30 days before being automatically purged to manage database bloat.
*   **Mobile Responsiveness:**
    *   On mobile, the notification dropdown converts into a full-screen sliding modal to maximize readability.
*   **Accessibility (WCAG 2.1 AA):**
    *   New notifications trigger a polite `aria-live="polite"` announcement for screen readers.
    *   The notification bell icon has an `aria-label="Notifications, 3 unread"`.
*   **Wireframe Descriptions:**
    *   **Dropdown Widget (Top Nav):** Clicking the bell icon opens a 350px wide dropdown panel. Header contains "Notifications" and a "Mark all read" link. The body contains a scrollable list of the 5 most recent alerts. The footer contains a "View All Notifications" link.
    *   **Full Page Center:** A dedicated `/notifications` route. A master list view taking up the center canvas. Left side of the page contains the filtering tabs (All, Unread, Errors). Each row contains an icon (indicating priority), the timestamp (e.g., "2 mins ago"), the message, and a contextual action button (e.g., "Download PDF").
*   **UX Recommendations for Notification Fatigue:**
    *   Batch similar notifications (e.g., instead of 5 notifications for 5 failed rows, send 1 notification: "Import complete with 5 errors").
    *   Allow users to toggle off non-critical notifications (like login alerts) in their Profile Settings.

## 7. DASHBOARD LAYOUT (GLOBAL SHELL)
*   **Sidebar (Left):** 240px wide (collapsible to 64px). Contains primary nav (Home, Projects, Templates, Downloads, Settings). Organization switcher at the top.
*   **Top Navigation:** 64px high. Breadcrumbs on the left. Global Search, Notification Bell, and Avatar dropdown on the right.
*   **Main Content Area:** `bg-gray-50` with centered `bg-white` surface cards. Max-width constraint (e.g., `max-w-7xl`) to prevent ultrawide stretching.

## 8. TEMPLATE EDITOR (DRAG-AND-DROP)
The most critical UX component of the application.
*   **Canvas Layout (Center):** Accurately scaled representation of the physical ID (e.g., CR80). Shows bleed margins.
*   **Toolbar (Top):** Undo/Redo, Zoom (In, Out, Fit), Snap-to-Grid toggle, "Preview Live Data" toggle.
*   **Toolbox (Left Panel):** Draggable icons: Text, Image, Shape, QR Code, Barcode.
*   **Properties Panel (Right Panel):** Context-aware. If text is selected, shows font family, size, color, alignment, and a dropdown for **Data Binding** (mapping to spreadsheet columns).
*   **Layers Panel:** Ability to reorder z-index (bring to front/send to back).

## 9. COMPONENTS LIBRARY
*   **Buttons:** Standard sizing (sm, md, lg). States: Default, Hover, Active, Disabled (opacity-50, cursor-not-allowed), Loading (spinner icon replaces text).
*   **Inputs:** 1px border. Focus ring (`focus:ring-2 focus:ring-blue-500 focus:outline-none`).
*   **Tables:** Striped rows. Sticky headers. Checkboxes for bulk actions.
*   **Modals:** Dark backdrop blur. Focus trapped inside. Esc key to close.
*   **Toast Notifications:** Slide-in from bottom-right. Auto-dismiss after 4s.
*   **Empty States:** Illustration + Helper Text + Primary Call to Action.
*   **Skeleton Loaders:** Used instead of spinners for initial page data loads to reduce perceived latency.

## 10. RESPONSIVE DESIGN
*   **Desktop (1024px+):** Full experience. Sidebar open. Complex editors fully visible.
*   **Tablet (768px - 1023px):** Sidebar collapses. Tables require horizontal scrolling.
*   **Mobile (<768px):** View-only mode. The Template Editor is hidden behind a warning: "Please use a desktop device to design templates." Dashboard and monitoring features stack vertically.

## 11. ACCESSIBILITY (WCAG 2.1 AA)
*   **Keyboard Navigation:** Entire template editor and data grid must be navigable via `Tab`, `Enter`, and `Arrow` keys.
*   **Screen Readers:** `aria-label` tags on all icon-only buttons. `role="alert"` for toast notifications.
*   **Color Contrast:** All text passes 4.5:1 ratio against backgrounds.
*   **Focus Indicators:** Mandatory visible focus rings for keyboard users.

## 12. ANIMATIONS
*   **Transitions:** Fast, subtle CSS transitions (`transition-all duration-200 ease-in-out`) on hovers and active states.
*   **Photo Matching:** When "Auto-Match" is clicked, unmatched thumbnails animate smoothly over to their respective data rows.
*   **Progress:** Smooth linear transition on progress bars during batch generation.

## 13. DESIGN SYSTEM (TAILWIND INTEGRATION)
*   **Naming Convention:** Standard utility classes. Custom component classes constructed via `clsx` and `tailwind-merge`.
*   **Spacing Scale:** Standard 4px scale (`p-4` = 16px, `m-8` = 32px).
*   **Typography Scale:** `text-sm` (14px) for data tables, `text-base` (16px) for body, `text-2xl` (24px) for page headers.

## 14. WIREFRAME DESCRIPTIONS

### Project Workspace: Data Grid
**Top:** Breadcrumbs: "Projects > Staff IDs 2026 > Data". Stepper shows step 1 is active.
**Center:** Large table. Headers: ID, First Name, Last Name, Department. 
**Right:** A sticky panel showing "Data Health: 98% Valid. 2 Rows Missing First Name."

### Template Editor
**Top:** Toolbar with Zoom %, Undo, Redo, "Live Preview" toggle.
**Left:** Vertical bar with Icons: [T] Text, [🖼️] Image, [QR] Code.
**Center:** Grey background. White rectangle (the ID card). Contains placeholder text `{{First_Name}}` and a generic avatar placeholder.
**Right:** Panel titled "Text Properties". Input for Hex Color, Dropdowns for Font and Weight. Dropdown labeled "Bind to Data Column" showing `First_Name`.

## 15. UX FOR LARGE BATCH IMPORTS (100k+ RECORDS)
To prevent browser crashes:
*   **Virtualization:** The Data Grid uses DOM virtualization (e.g., `@tanstack/react-virtual`). Only the 30 rows visible on screen are rendered in HTML, even if the dataset has 100,000 rows.
*   **Chunked Uploads:** Frontend chunks massive 500MB ZIP files into 10MB parts before pushing to S3 to prevent timeout timeouts and allow resumable uploads.

## 16. FUTURE ENHANCEMENTS
*   **Dark Mode:** Addition of a full dark-mode palette token system.
*   **AI Design Assistant:** A prompt box in the template editor: "Make this look like a modern tech company badge," automatically generating a canvas layout.
*   **Mobile Upload App:** A companion mobile view where users can snap photos on their phone that instantly sync to the active project workspace on desktop.

---
**End of UI/UX Specification**
```
