# Frontend UI/UX Improvements Design Spec

* **Date:** 2026-06-25
* **Author:** Antigravity
* **Status:** Draft

---

## 1. Goal & Context
The goal of this spec is to design three frontend UI/UX improvements to the Midden codebase:
1. **Settings Tab Layout**: Redesign the cluttered vertical layout of the admin settings page into a clean, categorized tabbed interface using Alpine.js.
2. **HTMX & AppError Integration**: Provide clear, styled inline error notices for forms powered by HTMX instead of plain, unstyled HTML page redirects.
3. **Redundant Button Removal**: Remove the redundant "Create a paste" button from the home page.

---

## 2. Settings Tab Layout
Currently, the admin settings page (`templates/admin.html`) displays all 15 configuration blocks vertically inside `<details>` components, causing significant clutter and a very long page.

### Proposed Changes:
1. **Alpine.js Tab State**: Wrap the form container with an Alpine.js active tab state:
   ```html
   x-data="{ activeTab: 'general', dirty: false, ... }"
   ```
2. **Tab Categories**: Categorize settings sections into 5 tabs:
   * `general`: Features, Policy, Branding
   * `limits`: Limits, API tokens, Rate limits
   * `storage`: Uploads, Processing, Delivery
   * `security`: Security, URL upload, Moderation notifications
   * `system`: Discovery, Background jobs, Metrics, Scanning
3. **Tab Navigation Bar**: Add a modern navigation bar below the main header in `templates/admin.html` with active state styles.
4. **Conditional Visibility**: Add `x-show="activeTab === '<category>'"` to each settings section (`details.settings-section`).
5. **Sticky Actions**: Keep the settings save bar sticky at the bottom of the viewport so users can easily save changes regardless of the active tab.

---

## 3. HTMX & AppError Integration
When forms/actions using HTMX fail, the backend returns an `AppError` which translates into a plain text/HTML page. This forces a redirect/replacement that is visually unstyled and causes users to lose their form inputs.

### Proposed Changes:
1. **RequestContext (Rust)**:
   * Create a `RequestContext` struct containing `templates`, `settings`, `current_user`, and `is_htmx`.
   * Declare a Tokio `task_local!` variable: `pub static REQUEST_CONTEXT: RequestContext;`.
2. **Context Middleware (Rust)**:
   * Write an Axum middleware in `src/web.rs` that reads the request, resolves the settings/user/htmx state, constructs the context, and runs the handler task inside the task-local scope.
3. **AppError Implementation (Rust)**:
   * Modify `IntoResponse for AppError` in `src/app.rs`:
     * If the task-local request context is active and `is_htmx` is `true`, render a partial error banner:
       ```html
       <div class="notice error" x-data="{ show: true }" x-show="show">
         <span>Error: {message}</span>
         <button type="button" class="link-button" x-on:click="show = false" style="float: right; margin-top: -2px; font-weight: bold; text-decoration: none;">&times;</button>
       </div>
       ```
     * If `is_htmx` is `false`, render the full, styled `error.html` template using `templates.render`.
     * Fallback to the current raw unstyled response if the request context is not set.
4. **Base Template updates**:
   * Add `<div id="global-error" aria-live="assertive"></div>` right above `<main>` in `templates/base.html`.
5. **HTMX Request/Response Listeners (JavaScript)**:
   * In `static/midden.js`, clear `#global-error` on `htmx:beforeRequest`.
   * In `static/midden.js`, intercept error status responses (`status >= 400`) during `htmx:beforeOnLoad`, configure HTMX to swap the response anyway (`shouldSwap = true`), and set the target element to `#global-error`.

---

## 4. Redundant Button Removal
Remove the redundant "Create a paste" link from `templates/index.html`. Keep the "Upload via URL" link if that feature is enabled.

---

## 5. Verification Plan

### Automated Verification
* Run formatting, clippy lint checks, and existing cargo tests to verify no compilation or logical regression is introduced:
  ```bash
  cargo fmt --all -- --check
  cargo clippy --workspace --all-targets --all-features -- -D warnings
  cargo test --workspace --all-features
  ```

### Manual Verification
1. Load the home page and verify the "Create a paste" button is removed.
2. Load the Admin settings page:
   * Verify settings are grouped into tabs and show/hide correctly.
   * Verify the save button and unsaved changes notice work as expected.
3. Trigger an HTMX validation error (e.g. revoke a token or save invalid settings under an HTMX request):
   * Verify the error appears as a styled banner at the top of the page.
   * Verify clicking the `×` button closes the banner.
   * Verify the error banner disappears when a new request is made.
