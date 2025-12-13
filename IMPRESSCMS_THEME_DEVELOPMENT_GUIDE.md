# ImpressCMS Theme Development Guide

A comprehensive guide for AI code assistants and developers to understand and work with ImpressCMS themes effectively.

## Table of Contents

1. [Overview](#overview)
2. [Directory Structure](#directory-structure)
3. [Core Theme Files](#core-theme-files)
4. [Smarty Template Basics](#smarty-template-basics)
5. [Module Template Overrides](#module-template-overrides)
6. [Block Templates](#block-templates)
7. [CSS and Assets](#css-and-assets)
8. [RTL Support](#rtl-support)
9. [Admin Theme](#admin-theme)
10. [Common Pitfalls](#common-pitfalls)
11. [Practical Workflows](#practical-workflows)
12. [ImpressCMS Theme APIs](#impresscms-theme-apis)

---

## Overview

ImpressCMS themes control the visual presentation of your website. A theme is a collection of:

- **Templates** - Smarty template files that define HTML structure
- **Stylesheets** - CSS files for styling
- **Assets** - Images, icons, JavaScript files
- **Module Overrides** - Custom templates for specific modules
- **Configuration** - Theme metadata and settings

### Key Principles

- **Themes are non-destructive** - They override module templates without modifying module source code
- **Separation of concerns** - Keep presentation logic in templates, business logic in modules
- **Reusability** - Themes can be switched without affecting content or functionality
- **Localization-aware** - Support multiple languages and RTL (right-to-left) text directions

---

## Directory Structure

### Recommended Theme Layout

```
themes/yourtheme/
├── theme.html                    # Main site wrapper template
├── theme_admin.html              # Admin area wrapper (optional)
├── style.css                     # Main stylesheet
├── style_admin.css               # Admin stylesheet
├── README.md                     # Theme documentation
├── index.html                    # Security placeholder
│
├── blocks/
│   ├── centerblocks.html         # Block container template
│   ├── centerblocks_admin.html   # Admin block container
│   └── index.html
│
├── img/
│   ├── logo.png
│   ├── header_bg.jpg
│   └── index.html
│
├── icons/
│   ├── favicon.ico
│   ├── icon.png
│   └── index.html
│
├── js/
│   ├── theme.js                  # Optional theme JavaScript
│   └── index.html
│
├── modules/
│   ├── forum/
│   │   ├── view.thread.html      # Override for forum templates
│   │   ├── index.html
│   │   └── ...
│   ├── system/
│   │   ├── system_siteclosed.html
│   │   └── index.html
│   └── [other-modules]/
│
└── rtl/
    ├── style.css                 # RTL-specific styles
    ├── style_admin.css           # RTL admin styles
    └── index.html
```

### Directory Purposes

| Directory | Purpose | Required |
|-----------|---------|----------|
| `blocks/` | Block container templates | Yes |
| `img/` | Theme images, logos, backgrounds | No |
| `icons/` | Favicon, app icons | No |
| `js/` | Theme-specific JavaScript | No |
| `modules/` | Module template overrides | No |
| `rtl/` | Right-to-left language support | No |

---

## Core Theme Files

### 1. theme.html (Main Site Wrapper)

The primary template that wraps all front-end content. This is the outermost HTML structure.

**Key Responsibilities:**
- Define HTML5 doctype and structure
- Include `<head>` with meta tags, stylesheets, scripts
- Provide placeholders for content and blocks
- Handle navigation and header/footer

**Example Structure:**

```html
<!DOCTYPE html>
<html lang="<{$icms_language}>">

<{assign var=theme_name value=$xoTheme->folderName}>

<head>
<!-- Theme name -->
<{assign var=theme_name value=$xoTheme->folderName}>

<!-- Title and general metadata tags -->
<title><{$smarty.const._IMPRESSCMS_ADMIN}> <{$icms_sitename}></title>
    
<meta http-equiv="content-type" content="text/html; charset=<{$icms_charset}>" />
<meta name="robots" content="<{$icms_meta_robots}>" />
<meta name="keywords" content="<{$icms_meta_keywords}>" />
<meta name="description" content="<{$icms_meta_description}>" />
<meta name="rating" content="<{$icms_meta_rating}>" />
<meta name="author" content="<{$icms_meta_author}>" />
<meta name="copyright" content="<{$icms_meta_copyright}>" />
<meta name="generator" content="ImpressCMS" />

<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title><{$icms_pagetitle}></title>

<{* ImpressCMS automatically includes stylesheets here *}>
<{$icms_css}>

<{* Theme stylesheet *}>
<link rel="stylesheet" href="<{$icms_themepath}>/style.css">
    
<{* Module Header tags (if any) *}>
<{$icms_module_header}>
    
<{* Additional head content *}>
<{$icms_meta}>
</head>
<body>
    <header>
        <h1><{$icms_sitename}></h1>
        <{* Navigation, logo, etc. *}>
    </header>

    <main>
        <{* Left blocks *}>
        <aside class="left-blocks">
            <{$icms_lblocks}>
        </aside>

        <{* Main content *}>
        <article class="main-content">
            <{$icms_contents}>
        </article>

        <{* Right blocks *}>
        <aside class="right-blocks">
            <{$icms_rblocks}>
        </aside>
    </main>

    <footer>
        <{* Footer content *}>
    </footer>

    <{* ImpressCMS footer scripts *}>
    <{$icms_footer}>
</body>
</html>
```

**Available Smarty Variables:**

| Variable                  | Type | Description                                                   |
|---------------------------|------|---------------------------------------------------------------|
| `<{$icms_sitename}>`      | string | Site name from configuration                                  |
| `<{$icms_pagetitle}>`     | string | Current page title                                            |
| `<{$icms_language}>`      | string | Current language code (e.g., 'en')                            |
| `<{$icms_themepath}>`     | string | URL path to theme directory                                   |
| `<{$icms_css}>`           | string | Auto-generated CSS includes                                   |
| `<{$icms_meta}>`          | string | Meta tags and additional head content                         |
| `<{$icms_contents}>`      | string | Main page content                                             |
| `<{$icms_lblocks}>`       | string | Left sidebar blocks                                           |
| `<{$icms_rblocks}>`       | string | Right sidebar blocks                                          |
| `<{$icms_footer}>`        | string | Footer scripts and content                                    |
| `<{$icms_module_header}>` | string | Meta tags for the module that is used on this page (optional) |


### 2. theme_admin.html (Admin Wrapper)

Optional template for the admin backend. If not provided, ImpressCMS uses a default admin theme.

**Example:**

```html
<!DOCTYPE html>
<html lang="<{$icms_language}>">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><{$icms_pagetitle}></title>
    <{$icms_css}>
    <link rel="stylesheet" href="<{$icms_themepath}>/style_admin.css">
    <{$icms_meta}>
</head>
<body class="admin-panel">
    <header class="admin-header">
        <h1><{$icms_sitename}> - Admin</h1>
    </header>
    <div class="admin-container">
        <nav class="admin-sidebar">
            <{$icms_adminmenu}>
        </nav>
        <main class="admin-content">
            <{$icms_contents}>
        </main>
    </div>
    <{$icms_footer}>
</body>
</html>
```

### 3. blocks/centerblocks.html (Block Container)

Defines how blocks are rendered in the center/main content area.

**Example:**

```html
<div class="block" id="block-<{$block.bid}>">
    <div class="block-header">
        <h3><{$block.title}></h3>
    </div>
    <div class="block-content">
        <{$block.content}>
    </div>
</div>
```

**Available Variables:**
- `<{$block.bid}>` - Block ID
- `<{$block.title}>` - Block title
- `<{$block.content}>` - Block HTML content
- `<{$block.weight}>` - Block display order

### 4. blocks/centerblocks_admin.html (Admin Block Container)

Similar to centerblocks.html but for the admin panel.

---

## Smarty Template Basics

ImpressCMS uses **Smarty v2** for templating with custom delimiters `<{` and `}>` instead of the standard `{` and `}`. Here are essential syntax rules for theme developers.

### ⚠️ Critical: Smarty Delimiter Syntax

**ImpressCMS uses `<{` and `}>` as Smarty delimiters, NOT the standard `{` and `}`**

This is critical for all template development in ImpressCMS themes. Always use `<{` to open and `}>` to close Smarty tags.

### Variable Output

```smarty
<{* Simple variable output (auto-escaped) *}>
<{$variable_name}>

<{* With filters *}>
<{$variable_name|upper}>
<{$variable_name|truncate:50}>
<{$variable_name|date_format:"%Y-%m-%d"}>

<{* Conditional output *}>
<{if $user_id > 0}>
    Welcome back, <{$user_name}>!
<{else}>
    Please log in
<{/if}>
```

### Loops

```smarty
<{* Loop through arrays *}>
<{foreach item=item from=$items}>
    <div class="item">
        <h4><{$item.title}></h4>
        <p><{$item.description}></p>
    </div>
<{/foreach}>

<{* Loop with key-value pairs *}>
<{foreach key=key item=value from=$settings}>
    <label><{$key}>: <{$value}></label>
<{/foreach}>
```

### ImpressCMS-Specific Functions

```smarty
<{* Use language constants - ImpressCMS will provide them in the correct language *}>
<{$smarty.const.THEME_HOME}>

<{* Format timestamps *}>
<{$timestamp|date_format:"%Y-%m-%d %H:%M"}>

<{* Create URLs *}>
<a href="<{$icms_url}>/modules/modulename/index.php">Module Link</a>

<{* Include other templates *}>
<{include file="db:modulename_header.html"}>

<{* Assign variables in template *}>
<{assign var="page_title" value="My Page"}>

<{* store the theme folder for later use in file references within the theme. Add this at the beginning of the theme.html and theme_admin.html *}>
<{assign var=theme_name value=$xoTheme->folderName}>


```

### Best Practices for Smarty in Themes

- **Use `<{` and `}>` delimiters** - Never use standard `{` and `}` in ImpressCMS
- **Indent with 4 spaces** (per CLAUDE.md guidelines)
- **Use meaningful variable names** - `$user_name` not `$u`
- **Escape output by default** - Smarty auto-escapes unless using `|smarty:nodefaults`
- **Keep logic minimal** - Complex logic belongs in PHP, not templates
- **Use comments** - Document non-obvious template sections with `<{* comment *}>`

```smarty
<{* Display user profile section *}>
<{if $user_id > 0}>
    <div class="user-profile">
        <h3><{$user_name|escape}></h3>
        <p>Member since <{$join_date|date_format:"%B %Y"}></p>
    </div>
<{/if}>
```

---

## Module Template Overrides

Module template overrides allow themes to customize how modules display content without modifying module source code.

### How Template Resolution Works

When ImpressCMS needs to render a template, it searches in this order:

1. Theme override - themes/{yourtheme}/modules/{module}/{template-file}
2. ImpressCMS template set - Internal template overrides
3. Module default - modules/{module}/templates/{template-file}
4. Fallback - Module's built-in rendering logic

### Creating an Override

Step 1: Locate the original template
Step 2: Create the override directory
Step 3: Create the override file
Step 4: Modify as needed

### Important Rules

- **Preserve the filename exactly** - Including extension (`.html`, `.tpl`, etc.)
- **Preserve subdirectory structure** - If original is in `templates/admin/`, create `modules/forum/admin/`
- **Keep variable names consistent** - Don't rename variables the module passes to the template
- **Test thoroughly** - Ensure your override works with the module's data

### Example: Forum Thread Override

**Original location:** `modules/forum/templates/view.thread.html`

**Override location:** `themes/yourtheme/modules/forum/view.thread.html`

**Original template:**
```html
<div class="thread">
    <h2><{$thread.title}></h2>
    <p class="meta">Posted by <{$thread.author}> on <{$thread.date}></p>
    <div class="content">
        <{$thread.body}>
    </div>
</div>
```

**Customized override:**
```html
<article class="forum-thread">
    <header class="thread-header">
        <h2 class="thread-title"><{$thread.title}></h2>
        <div class="thread-meta">
            <span class="author"><{$thread.author}></span>
            <time datetime="<{$thread.date|date_format:'%Y-%m-%dT%H:%M:%S'}>">
                <{$thread.date|date_format:"%B %d, %Y"}>
            </time>
        </div>
    </header>
    <div class="thread-body">
        <{$thread.body}>
    </div>
</article>
```

### Finding Module Templates

To find what templates a module uses:

1. **Check module documentation** - Usually in `modules/{module}/docs/`
2. **Browse module source** - Look in `modules/{module}/templates/`
3. **Check icms_version.php** - Module configuration lists templates
4. **Use browser inspector** - Look for template names in HTML comments

---

## Block Templates

Blocks are reusable content containers that appear in sidebars or designated areas.

### Block Template Structure

Block templates receive data from block functions and render it for display.

**Example block template:** `themes/yourtheme/blocks/news_recent.html`

```html
<div class="news-block">
    <h3 class="block-title">Recent News</h3>
    <ul class="news-list">
        <{foreach item=item from=$news_items}>
            <li>
                <a href="<{$item.url}>">
                    <{$item.title}>
                </a>
                <span class="date"><{$item.date|date_format:"%b %d"}></span>
            </li>
        <{/foreach}>
    </ul>
</div>
```

### Block Variables

Blocks pass data as arrays. Common patterns:

```smarty
<{* Simple list *}>
<{foreach item=item from=$items}>
    <div><{$item.title}></div>
<{/foreach}>

<{* With conditional rendering *}>
<{if count($items) > 0}>
    <ul>
        <{foreach item=item from=$items}>
            <li><{$item.title}></li>
        <{/foreach}>
    </ul>
<{else}>
    <p>No items available</p>
<{/if}>
```

### Overriding Block Templates

Block templates are overridden the same way as module templates:

1. Find original: `modules/{module}/templates/{blockname}.html`
2. Create override: `themes/{yourtheme}/modules/{module}/{blockname}.html`
3. Customize as needed

---

## CSS and Assets

### Stylesheet Organization

Main stylesheet at themes/yourtheme/style.css with reset, base styles, layout, blocks, and responsive design.

### Admin Stylesheet

File at themes/yourtheme/style_admin.css focused on usability and information density.

### Asset Paths in Templates

Use Smarty variables for asset paths:

```html
<{* Correct - uses theme path variable *}>
<img src="<{$icms_themepath}>/img/logo.png" alt="Logo">
<link rel="stylesheet" href="<{$icms_themepath}>/style.css">
<script src="<{$icms_themepath}>/js/theme.js"></script>

<{* Avoid hardcoding paths *}>
<img src="/themes/mytheme/img/logo.png">  <{* Wrong - breaks if theme moves *}>
```

### Including External Libraries

```html
<{* Bootstrap CSS *}>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">

<{* jQuery *}>
<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>

<{* Font Awesome *}>
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
```

---

## RTL Support

Right-to-left (RTL) language support is essential for Arabic, Hebrew, Persian, and other RTL languages.

### RTL Stylesheet

Create themes/yourtheme/rtl/style.css with RTL-specific overrides.

### Detecting RTL in Templates

```smarty
<{if $icms_language == 'ar' || $icms_language == 'he' || $icms_language == 'fa'}>
    <{* RTL-specific content *}>
    <div class="rtl-content">
        <{$content}>
    </div>
<{else}>
    <{* LTR content *}>
    <div class="ltr-content">
        <{$content}>
    </div>
<{/if}>
```

### ImpressCMS RTL Loading

ImpressCMS automatically loads RTL stylesheets when needed.

---

## Common Pitfalls

### 1. Using Standard Smarty Delimiters Instead of ImpressCMS Delimiters

**❌ Wrong:**
```html
<img src="{$icms_themepath}/img/logo.png">
<h1>{$icms_sitename}</h1>
```

**✅ Correct:**
```html
<img src="<{$icms_themepath}>/img/logo.png">
<h1><{$icms_sitename}></h1>
```

### 2. Hardcoded Paths

**❌ Wrong:**
```html
<img src="/themes/mytheme/img/logo.png">
<link rel="stylesheet" href="/themes/mytheme/style.css">
```

**✅ Correct:**
```html
<img src="<{$icms_themepath}>/img/logo.png">
<link rel="stylesheet" href="<{$icms_themepath}>/style.css">
```

### 3. Modifying Module Source Instead of Overriding

**❌ Wrong:**
```
modules/forum/templates/view.thread.html  (modified directly)
```

**✅ Correct:**
```
themes/yourtheme/modules/forum/view.thread.html  (override)
```

### 4. Missing index.html Files

**❌ Wrong:**
```
themes/yourtheme/img/
themes/yourtheme/js/
```

**✅ Correct:**
```
themes/yourtheme/img/index.html
themes/yourtheme/js/index.html
```

### 5. Incorrect Template Variable Names

**❌ Wrong:**
```smarty
<{$thread.title}>  <{* Module passes $thread_title *}>
```

**✅ Correct:**
```smarty
<{$thread_title}>  <{* Match module's variable names *}>
```

### 6. Not Testing RTL

Always test your theme with RTL languages to ensure proper display.

### 7. Forgetting to Escape Output

**❌ Wrong:**
```smarty
<{$user_input|smarty:nodefaults}>  <{* Unsafe - allows XSS *}>
```

**✅ Correct:**
```smarty
<{$user_input}>  <{* Auto-escaped by default *}>
```

### 8. Inconsistent Block Structure

**❌ Wrong:**
```html
<{* Inconsistent block markup *}>
<div class="block">
    <{$block.content}>
</div>
```

**✅ Correct:**
```html
<{* Consistent structure for all blocks *}>
<div class="block" id="block-<{$block.bid}>">
    <div class="block-header">
        <h3><{$block.title}></h3>
    </div>
    <div class="block-content">
        <{$block.content}>
    </div>
</div>
```

---

## Practical Workflows

### Workflow 1: Creating a New Theme from Scratch

**Step 1: Create directory structure**

```bash
mkdir -p themes/mytheme/{blocks,img,icons,js,modules,rtl}
touch themes/mytheme/index.html
touch themes/mytheme/blocks/index.html
touch themes/mytheme/img/index.html
touch themes/mytheme/icons/index.html
touch themes/mytheme/js/index.html
touch themes/mytheme/rtl/index.html
```

**Step 2: Create core templates**

Create `themes/mytheme/theme.html` with proper `<{` and `}>` delimiters.

Create `themes/mytheme/blocks/centerblocks.html` with block structure.

**Step 3: Create stylesheets**

Create `themes/mytheme/style.css` with your design.

**Step 4: Test**

Switch to your theme in ImpressCMS admin panel and verify it displays correctly.

### Workflow 2: Overriding a Module Template

**Step 1: Identify the template to override**

Example: Override forum thread display

**Step 2: Create override directory**

```bash
mkdir -p themes/mytheme/modules/forum
touch themes/mytheme/modules/forum/index.html
```

**Step 3: Copy and modify template**

Copy `modules/forum/templates/view.thread.html` to `themes/mytheme/modules/forum/view.thread.html`

Edit to customize appearance while keeping variable names intact and using `<{` and `}>` delimiters.

**Step 4: Test**

View a forum thread and verify your custom template is used.

### Workflow 3: Adding RTL Support

**Step 1: Create RTL stylesheet**

```bash
touch themes/mytheme/rtl/style.css
touch themes/mytheme/rtl/style_admin.css
```

**Step 2: Add RTL-specific rules**

In `themes/mytheme/rtl/style.css`:
```css
body {
    direction: rtl;
    text-align: right;
}

/* Reverse layout-specific rules */
main {
    grid-template-columns: 250px 1fr 250px;
    direction: rtl;
}
```

**Step 3: Test with RTL language**

Switch site language to Arabic or Hebrew and verify layout.

---

## ImpressCMS Theme APIs

### Smarty Variables Available in All Templates

| Variable | Type | Description |
|----------|------|-------------|
| `<{$icms_url}>` | string | Base URL of ImpressCMS installation |
| `<{$icms_themepath}>` | string | URL path to current theme |
| `<{$icms_sitename}>` | string | Site name |
| `<{$icms_pagetitle}>` | string | Current page title |
| `<{$icms_language}>` | string | Current language code |
| `<{$icms_contents}>` | string | Main page content |
| `<{$icms_lblocks}>` | string | Left sidebar blocks |
| `<{$icms_rblocks}>` | string | Right sidebar blocks |
| `<{$icms_css}>` | string | Auto-generated CSS includes |
| `<{$icms_meta}>` | string | Meta tags and head content |
| `<{$icms_footer}>` | string | Footer scripts |
| `<{$icms_isadmin}>` | bool | Is current user admin? |
| `<{$icms_userid}>` | int | Current user ID (0 if anonymous) |
| `<{$icms_username}>` | string | Current username |

### Smarty Filters Useful in Themes

```smarty
<{* Text formatting *}>
<{$text|upper}>                    <{* Uppercase *}>
<{$text|lower}>                    <{* Lowercase *}>
<{$text|capitalize}>               <{* Capitalize first letter *}>
<{$text|truncate:50}>              <{* Truncate to 50 chars *}>

<{* Date formatting *}>
<{$timestamp|date_format:"%Y-%m-%d"}>
<{$timestamp|date_format:"%B %d, %Y"}>

<{* Number formatting *}>
<{$price|number_format:2}>         <{* 2 decimal places *}>

<{{* HTML escaping *}>
<{$user_input|escape}>             <{* Escape HTML *}>
<{$user_input|escape:'html'}>      <{* Explicit HTML escape *}>
```

### ImpressCMS Helper Functions in Templates

```smarty
<{* Translate language constants *}>
<{_e('_MD_MODULENAME_TITLE')}>

<{* Format timestamps *}>
<{$timestamp|date_format:"%Y-%m-%d %H:%M"}>

<{{* Create module URLs *}>
<a href="<{$icms_url}>/modules/modulename/index.php?id=<{$item_id}>">
    <{$item_title}>
</a>

<{{* Include other templates *}>
<{include file="db:modulename_header.html"}>

<{{* Conditional blocks *}>
<{if $icms_isadmin}>
    <a href="<{$icms_url}>/admin/">Admin Panel</a>
<{/if}>
```

### Theme Configuration (README.md)

Document your theme in `themes/yourtheme/README.md`:

```markdown
# My Theme

## Description
A modern, responsive theme for ImpressCMS.

## Features
- Responsive design
- RTL language support
- Custom module overrides
- Dark mode support

## Requirements
- ImpressCMS 2.0+
- PHP 8.2+

## Installation
1. Extract to `themes/mytheme/`
2. Go to Admin > Themes
3. Select "My Theme" and click "Activate"

## Module Overrides
- Forum: Custom thread display
- News: Custom article layout

## Browser Support
- Chrome 90+
- Firefox 88+
- Safari 14+
- Edge 90+

## License
GPL 2.0
```

## Modernizing ImpressCMS Themes

Older ImpressCMS themes still use the old xoops variables in Smarty. Update your themes to the icms variants to remain compatible with newer releases.

| Old Variable | Replace with          |
|--------------|-----------------------|
| `<{$xoops_url}>` | `<{$icms_url}>`       |
| `<{$xoops_themepath}>` | `<{$icms_themepath}>` |
| `<{$xoops_sitename}>` | `<{$icms_sitename}>`  |
| `<{$xoops_pagetitle}>` | `<{$icms_pagetitle}>` |
| `<{$xoops_language}>` | `<{$icms_language}>`  |
| `<{$xoops_contents}>` | `<{$icms_language}>`  |
| `<{$xoops_lblocks}>` | `<{$icms_lblocks}>`   | 
| `<{$xoops_rblocks}>` | `<{$icms_rblocks}>`   | 
| `<{$xoops_css}>` | `<{$icms_css}>`       | 
| `<{$xoops_meta}>` | `<{$icms_meta}>`      |
| `<{$xoops_footer}>` | `<{$icms_footer}>`    | 
| `<{$xoops_isadmin}>` | `<{$icms_isadmin}>`   | 
| `<{$xoops_userid}>` | `<{$icms_userid}>`    | 
| `<{$xoops_username}>` | `<{$icms_username}>`  |
---

## Summary

**Key Takeaways for AI Assistants:**

1. **Theme location:** `themes/{themename}/`
2. **Core files:** `theme.html`, `style.css`, `blocks/centerblocks.html`
3. **Template language:** Smarty v2 with `<{` and `}>` delimiters (NOT standard `{` and `}`)
4. **Module overrides:** Mirror module template structure in `modules/` subdirectory
5. **Asset paths:** Always use `<{$icms_themepath}>` variable
6. **RTL support:** Provide `rtl/` directory with RTL-specific files
7. **Security:** Never modify module source; use theme overrides
8. **Testing:** Test with multiple languages, especially RTL languages

**Critical Reminders:**

- **ALWAYS use `<{` and `}>` delimiters** - This is ImpressCMS-specific Smarty syntax
- **Never use standard `{` and `}` delimiters** - They will not work in ImpressCMS
- **Smarty v2 syntax** - Use `foreach` with `item=` and `from=` parameters
- **Always preserve original variable names** from modules
- **Use Smarty's auto-escaping** for security
- **Keep templates focused on presentation** - Complex logic belongs in PHP
- **Document custom overrides** in README.md
- **Test responsive design and RTL support** thoroughly

**When working with themes:**
- Check existing themes (like iTheme) for syntax examples
- Use `<{$icms_themepath}>` for all asset references
- Create `index.html` files in all directories for security
- Test with multiple languages, especially RTL languages
- Preserve module template variable names exactly
- Use consistent block structure across all blocks
