Brief explanation: a compact guideline describing an ImpressCMS theme layout, required files, RTL support and how module template overrides work (they mirror the files in the module's `templates` folder). Example theme tree provided.

1. Theme root and purpose
    1. Place the theme in `themes/yourtheme/`.
    2. The theme provides:
        - global wrappers (layout and wrapper templates),
        - CSS and asset files,
        - block templates and admin variants,
        - module-specific template overrides.

2. Key files and folders (what to include and why)
    1. `theme.html` — main site wrapper (common HTML structure, placeholders for content/blocks).
    2. `theme_admin.html` — admin area wrapper (optional, for backend).
    3. `centerblocks.html.tpl` and `centerblocks_admin.html.tpl` — block placements/wrappers used by the system.
    4. CSS files in the root folder: `style.css`, `style_admin.css`— theme-defined styles (a new theme may redefine all).
    5. `img/` and `icons/` — graphics, logos, favicons.
    6. `js/` — JavaScript files (optional).
    7. `modules/` — module-specific overrides (see next section).
    8. `rtl/` — RTL variants of CSS files and templates for right-to-left locales.
    9. `index.html` placeholders in folders to prevent directory listing.
    10. `README.md` — theme metadata/instructions in markdown format.

3. Module template overrides (rules and workflow)
    1. Module templates live in the module source at `modules/{module}/templates/`.
    2. To override a module template, create a file in the theme under `themes/{yourtheme}/modules/{module}/` that matches the template filename (and relative path) found in the module's `templates` folder.
        - Example: to override `modules/forum/templates/view.thread.html`, place the override at `themes/{yourtheme}/modules/forum/view.thread.html`.
        - - Example: to override `modules/forum/templates/topic.thread.html.tpl`, place the override at `themes/{yourtheme}/modules/forum/topic.thread.html.tpl`.
    3. The CMS resolution order:
        1. Theme override at `themes/{yourtheme}/modules/{module}/{template-file}`
        2. Template override in the ImpressCMS template set (resolved by ImpressCMS internally)
        3. Module's own `modules/{module}/templates/{template-file}`
        4. Module default behavior (fallback)
    4. Keep template names identical and preserve subdirectory structure if module templates are grouped in subfolders.
    5. Admin module template overrides follow the same pattern inside `themes/{yourtheme}/modules/{module}/`.

4. RTL and admin variants
    1. Provide RTL equivalents in `themes/{yourtheme}/rtl/` (e.g., `rtl/style.css`) and load them when site locale is RTL.
    2. Provide `style_admin.css` and `theme_admin.html` for backend styling; optionally include `rtl/` admin variants.

5. Best practices
    1. Keep structural templates (`theme.html`, `centerblocks.html`) minimal and include hooks/placeholders for blocks and content.
    2. Do not change module templates in the module folder; prefer theme overrides so updates to the module don’t overwrite your theme.
    3. Include `index.html` in folders to improve security and prevent listing.
    4. Document in `README.txt` what templates you override and any special assets.

Example (recommended minimal structure)
```text
themes/yourtheme/
    `theme.html`
    `theme_admin.html`
    `style.css`
    `style_admin.css`
    `README.md`
    `index.html`
    blocks/
        `centerblocks.html`
        `centerblocks_admin.html`        
    img/
        `logo.png`
        `header_bg.jpg`
        `index.html`
    icons/
        `favicon.ico`
        `icon.png`
        `index.html`
    js/
        `index.html`
    modules/
        forum/
            `view.thread.html`          <-- override for modules/forum/templates/view.thread.html
            `index.html`
        system/
            `system_siteclosed.html`   <-- override for modules/system/templates/system_siteclosed.html
            `index.html`
    rtl/
        `style.css`
        `index.html`
```

Core rule summary: theme template overrides must match the filenames (and relative paths) found in the module's `templates` folder; place those files under `themes/{yourtheme}/modules/{module}/` to override.
