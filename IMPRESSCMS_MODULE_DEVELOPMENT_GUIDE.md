# ImpressCMS Module Development Guide

**Version:** 3.0
**Target PHP Version:** 8.2+
**Target ImpressCMS Version:** 2.0+
**Last Updated:** 2025-10-14
**Architecture:** Modern MVC with Fluent Interfaces

---

## Table of Contents

1. [Introduction](#introduction)
2. [Module Structure](#module-structure)
3. [Controller Architecture](#controller-architecture)
4. [Fluent Interfaces](#fluent-interfaces)
5. [Core Files](#core-files)
6. [Directory Structure](#directory-structure)
7. [Best Practices](#best-practices)
8. [ImpressCMS Core Libraries](#impresscms-core-libraries)
9. [Security](#security)
10. [Code Examples](#code-examples)
11. [Testing](#testing)
12. [Documentation](#documentation)

---

## Introduction

This guide provides comprehensive standards and best practices for developing modern ImpressCMS modules compatible with PHP 8.2+ and ImpressCMS 2.0+. It follows PSR-1, PSR-2, and PSR-12 coding standards while leveraging ImpressCMS's powerful IPF (ImpressCMS Persistable Framework) and core libraries.

**Version 3.0 introduces a modern, controller-based architecture with fluent interfaces** for cleaner, more maintainable code.

### Core Principles

1. **Controller-Based MVC** - Use dedicated controllers for request handling and business logic
2. **Fluent Interfaces** - Leverage method chaining for cleaner, more readable code
3. **Follow ImpressCMS conventions** - Use documented ImpressCMS patterns as foundation
4. **Leverage IPF framework** - Maximize use of `icms_ipf_*` classes with modern patterns
5. **Security by default** - Always validate input, sanitize output, use prepared statements
6. **PHP 8.2+ features** - Use modern PHP syntax (typed properties, constructor promotion, etc.)
7. **Separation of concerns** - Controllers handle requests, Handlers manage data, Views render output

---

## Module Structure

### Standard Directory Layout

```
module_name/
├── admin/                      # Admin interface files
│   ├── index.php              # Main admin page (routes to controller)
│   ├── admin_menu.php         # Admin menu definition
│   └── [entity].php           # Entity management (routes to controller)
├── blocks/                     # Block definitions
│   └── [blockname].php        # Block implementation
├── class/                      # Module classes
│   ├── Controllers/           # Controller classes (NEW)
│   │   ├── EntityController.php      # User-side controller
│   │   └── Admin/
│   │       └── EntityController.php  # Admin controller
│   ├── [Entity].php           # IPF Object classes
│   └── [Entity]Handler.php    # IPF Handler classes (with fluent interface)
├── docs/                       # Documentation
│   ├── changelog.txt
│   ├── install.txt
│   └── readme.txt
├── images/                     # Module images
│   ├── module_icon.png        # Module icon (64x64)
│   └── admin_icon.png         # Admin icon
├── include/                    # Include files
│   ├── common.php             # Common functions
│   ├── functions.php          # Helper functions
│   └── update.php             # Update scripts
├── language/                   # Language files
│   ├── english/
│   │   ├── admin.php          # Admin language
│   │   ├── blocks.php         # Block language
│   │   ├── main.php           # Main language
│   │   └── modinfo.php        # Module info language
│   └── [other_languages]/
├── sql/                        # SQL files
│   └── mysql.sql              # Database schema
├── templates/                  # Smarty templates
│   ├── admin/                 # Admin templates
│   └── [template_name].html   # User-side templates
├── icms_version.php           # Module configuration (REQUIRED)
├── index.php                  # Main module page
├── header.php                 # Module header
├── footer.php                 # Module footer
├── module.css                 # Module stylesheet
├── module_rtl.css             # RTL stylesheet
├── rss.php                    # RSS feed (optional)
├── comment_*.php              # Comment handlers (if using comments)
└── notification_update.php    # Notification handler (if using notifications)
```

---

## Controller Architecture

### Overview

Modern ImpressCMS modules use a **Controller-Based MVC architecture** that separates concerns:

- **Controllers** - Handle HTTP requests, coordinate business logic, prepare data for views
- **Handlers (Models)** - Manage data persistence and business rules via IPF framework
- **Views (Templates)** - Render output using Smarty templates

### Benefits

1. **Cleaner Code** - Request logic separated from data access
2. **Reusability** - Controllers can be called from multiple entry points
3. **Testability** - Controllers can be unit tested independently
4. **Maintainability** - Clear separation of concerns
5. **Consistency** - Standard patterns across all modules

### Controller Structure

#### Base Controller

All controllers extend a base controller class:

```php
<?php
/**
 * Base controller for module
 *
 * @package Modulename
 */
abstract class ModulenameBaseController
{
    protected ModulenameEntityHandler $entityHandler;
    protected icms_view_Tpl $template;
    protected icms_core_Security $security;

    public function __construct()
    {
        $this->entityHandler = icms_getModuleHandler('entity', 'modulename');
        $this->template = new icms_view_Tpl();
        $this->security = icms::$security;
    }

    /**
     * Check if user has permission
     */
    protected function checkPermission(string $permission): bool
    {
        if (!icms::$user) {
            return false;
        }

        $gperm_handler = icms::handler('icms_member_groupperm');
        $groups = icms::$user->getGroups();

        return $gperm_handler->checkRight(
            $permission,
            0,
            $groups,
            icms::$module->getVar('mid')
        );
    }

    /**
     * Redirect with message
     */
    protected function redirect(string $url, string $message, int $delay = 3): void
    {
        redirect_header($url, $delay, $message);
    }

    /**
     * Render template
     */
    protected function render(string $template, array $data = []): void
    {
        foreach ($data as $key => $value) {
            $this->template->assign($key, $value);
        }

        $this->template->display("db:modulename_{$template}.html");
    }
}
```

#### User-Side Controller Example

```php
<?php
/**
 * Entity controller for user-side operations
 *
 * @package Modulename\Controllers
 */
class ModulenameEntityController extends ModulenameBaseController
{
    /**
     * Display list of entities (index action)
     */
    public function index(): void
    {
        $categoryId = isset($_GET['category_id']) ? (int)$_GET['category_id'] : null;
        $page = isset($_GET['page']) ? (int)$_GET['page'] : 1;
        $limit = icms::$module->config['items_per_page'];
        $start = ($page - 1) * $limit;

        // Fluent query
        $entities = $this->entityHandler
            ->where('status', 1)
            ->when($categoryId, fn($q) => $q->where('category_id', $categoryId))
            ->orderBy('created', 'DESC')
            ->limit($limit)
            ->offset($start)
            ->get();

        $totalCount = $this->entityHandler
            ->where('status', 1)
            ->when($categoryId, fn($q) => $q->where('category_id', $categoryId))
            ->count();

        // Pagination
        $pagenav = new icms_view_PageNav($totalCount, $limit, $page, 'page');

        $this->render('index', [
            'entities' => $this->formatEntities($entities),
            'pagenav' => $pagenav->renderNav(),
            'category_id' => $categoryId
        ]);
    }

    /**
     * Display single entity (show action)
     */
    public function show(): void
    {
        $entityId = isset($_GET['entity_id']) ? (int)$_GET['entity_id'] : 0;

        $entity = $this->entityHandler
            ->where('entity_id', $entityId)
            ->where('status', 1)
            ->first();

        if (!$entity) {
            $this->redirect('index.php', _MD_MODULENAME_ENTITY_NOT_FOUND);
            return;
        }

        // Increment view counter
        $entity->incrementCounter('views');

        $this->render('single', [
            'entity' => $this->formatEntity($entity)
        ]);
    }

    /**
     * Store new entity (store action)
     */
    public function store(): void
    {
        if (!$this->security->check() || !$this->checkPermission('modulename_submit')) {
            $this->redirect('index.php', _NOPERM);
            return;
        }

        // Fluent entity creation and saving
        $entity = $this->entityHandler->create()
            ->setTitle($_POST['title'])
            ->setDescription($_POST['description'])
            ->setBody($_POST['body'])
            ->setStatus($_POST['status'])
            ->setCategoryId($_POST['category_id'])
            ->save();

        if ($entity) {
            $this->redirect(
                'entity.php?entity_id=' . $entity->id(),
                _MD_MODULENAME_ENTITY_CREATED
            );
        } else {
            $this->redirect('index.php', _MD_MODULENAME_ENTITY_CREATE_ERROR);
        }
    }
}
```

### Routing to Controllers

#### User-Side Entry Point (index.php)

```php
<?php
/**
 * Module index - routes to controller
 */

include_once '../../mainfile.php';
include_once ICMS_ROOT_PATH . '/header.php';
include_once __DIR__ . '/include/common.php';

$controller = new ModulenameEntityController();
$controller->index();

include_once ICMS_ROOT_PATH . '/footer.php';
```

#### Entity Page (entity.php)

```php
<?php
/**
 * Entity page - routes to controller based on action
 */

include_once '../../mainfile.php';
include_once ICMS_ROOT_PATH . '/header.php';
include_once __DIR__ . '/include/common.php';

$controller = new ModulenameEntityController();
$action = $_GET['action'] ?? 'show';

match($action) {
    'show' => $controller->show(),
    'create' => $controller->create(),
    'store' => $controller->store(),
    'edit' => $controller->edit(),
    'update' => $controller->update(),
    'destroy' => $controller->destroy(),
    default => $controller->show()
};

include_once ICMS_ROOT_PATH . '/footer.php';
```

---

## Fluent Interfaces

### Overview

Fluent interfaces use method chaining to create more readable and expressive code. Instead of multiple separate method calls, operations flow naturally from left to right.

### Before and After Comparison

#### Traditional Approach (Verbose)

```php
<?php
// Query building
$criteria = new icms_db_criteria_Compo();
$criteria->add(new icms_db_criteria_Item('status', 1));
$criteria->add(new icms_db_criteria_Item('category_id', $categoryId));
$criteria->setSort('created');
$criteria->setOrder('DESC');
$criteria->setLimit(10);
$criteria->setStart(0);
$entities = $handler->getObjects($criteria);

// Object manipulation
$entity = $handler->create();
$entity->setVar('title', 'My Title');
$entity->setVar('description', 'Description');
$entity->setVar('status', 1);
$entity->setVar('category_id', 5);
$handler->insert($entity);
```

#### Modern Fluent Approach

```php
<?php
// Query building - clean and readable
$entities = $handler
    ->where('status', 1)
    ->where('category_id', $categoryId)
    ->orderBy('created', 'DESC')
    ->limit(10)
    ->offset(0)
    ->get();

// Object manipulation - chainable setters
$entity = $handler->create()
    ->setTitle('My Title')
    ->setDescription('Description')
    ->setStatus(1)
    ->setCategoryId(5)
    ->save();
```

### Implementing Fluent Handlers

Extend the IPF Handler to support fluent queries:

```php
<?php
/**
 * Fluent Entity Handler
 *
 * @package Modulename
 */
class ModulenameEntityHandler extends icms_ipf_Handler
{
    private array $whereClauses = [];
    private ?string $orderByField = null;
    private string $orderDirection = 'ASC';
    private int $limitValue = 0;
    private int $offsetValue = 0;

    public function __construct(icms_db_legacy_Database &$db)
    {
        parent::__construct($db, 'entity', 'entity_id', 'title', 'description', 'modulename');
    }

    /**
     * Add WHERE clause (fluent)
     */
    public function where(string $field, mixed $value, string $operator = '='): self
    {
        $this->whereClauses[] = compact('field', 'value', 'operator');
        return $this;
    }

    /**
     * Conditional WHERE clause (fluent)
     */
    public function when(mixed $condition, callable $callback): self
    {
        if ($condition) {
            $callback($this);
        }
        return $this;
    }

    /**
     * Set ORDER BY (fluent)
     */
    public function orderBy(string $field, string $direction = 'ASC'): self
    {
        $this->orderByField = $field;
        $this->orderDirection = strtoupper($direction);
        return $this;
    }

    /**
     * Set LIMIT (fluent)
     */
    public function limit(int $limit): self
    {
        $this->limitValue = $limit;
        return $this;
    }

    /**
     * Set OFFSET (fluent)
     */
    public function offset(int $offset): self
    {
        $this->offsetValue = $offset;
        return $this;
    }

    /**
     * Execute query and get results
     */
    public function get(): array
    {
        $criteria = $this->buildCriteria();
        $results = $this->getObjects($criteria);
        $this->reset();
        return $results;
    }

    /**
     * Get first result
     */
    public function first(): ?ModulenameEntity
    {
        $this->limit(1);
        $results = $this->get();
        return $results[0] ?? null;
    }

    /**
     * Count results
     */
    public function count(): int
    {
        $criteria = $this->buildCriteria();
        $count = $this->getCount($criteria);
        $this->reset();
        return $count;
    }

    /**
     * Build criteria from fluent calls
     */
    private function buildCriteria(): icms_db_criteria_Compo
    {
        $criteria = new icms_db_criteria_Compo();

        foreach ($this->whereClauses as $clause) {
            $criteria->add(new icms_db_criteria_Item(
                $clause['field'],
                $clause['value'],
                $clause['operator']
            ));
        }

        if ($this->orderByField) {
            $criteria->setSort($this->orderByField);
            $criteria->setOrder($this->orderDirection);
        }

        if ($this->limitValue > 0) {
            $criteria->setLimit($this->limitValue);
        }

        if ($this->offsetValue > 0) {
            $criteria->setStart($this->offsetValue);
        }

        return $criteria;
    }

    /**
     * Reset query builder state
     */
    private function reset(): void
    {
        $this->whereClauses = [];
        $this->orderByField = null;
        $this->orderDirection = 'ASC';
        $this->limitValue = 0;
        $this->offsetValue = 0;
    }
}
```

---


## Core Files

### 1. icms_version.php (REQUIRED)

This is the most critical file - it defines all module metadata, configuration, and integration points.

**File Status:** MANDATORY
**Location:** Module root directory

```php
<?php
/**
 * Module version information
 *
 * @copyright  The ImpressCMS Project
 * @license    GPL 2.0
 */

defined("ICMS_ROOT_PATH") or die("ImpressCMS root path not defined");

// Load language file
icms_loadLanguageFile('modulename', 'modinfo');

$modversion = [
    // Basic Information
    'name'          => _MI_MODULENAME_NAME,
    'version'       => '1.0.0',
    'description'   => _MI_MODULENAME_DESC,
    'author'        => 'Your Name',
    'credits'       => 'Credits',
    'help'          => 'page=help',
    'license'       => 'GPL 2.0',
    'official'      => 0,
    'dirname'       => basename(__DIR__),
    'modname'       => 'modulename',

    // Images
    'iconsmall'     => 'images/icon_small.png',
    'iconbig'       => 'images/icon_big.png',
    'image'         => 'images/module_icon.png',

    // Database
    'sqlfile'       => ['mysql' => 'sql/mysql.sql'],
    'tables'        => [
        'modulename_entity',
        'modulename_category'
    ],

    // Admin Interface
    'hasAdmin'      => 1,
    'adminindex'    => 'admin/index.php',
    'adminmenu'     => 'admin/admin_menu.php',

    // Main Menu
    'hasMain'       => 1,

    // Search
    'hasSearch'     => 1,
    'search'        => [
        'file' => 'include/search.inc.php',
        'func' => 'modulename_search'
    ],

    // Comments
    'hasComments'   => 1,
    'comments'      => [
        'itemName'      => 'entity',
        'pageName'      => 'entity.php',
        'extraParams'   => ['entity_id'],
        'dirname'       => basename(__DIR__)
    ],

    // Notifications
    'hasNotification' => 1,
];

// Module Configuration Options
$modversion['config'][] = [
    'name'        => 'items_per_page',
    'title'       => '_MI_MODULENAME_ITEMS_PER_PAGE',
    'description' => '_MI_MODULENAME_ITEMS_PER_PAGE_DSC',
    'formtype'    => 'textbox',
    'valuetype'   => 'int',
    'default'     => 10
];

$modversion['config'][] = [
    'name'        => 'enable_feature',
    'title'       => '_MI_MODULENAME_ENABLE_FEATURE',
    'description' => '_MI_MODULENAME_ENABLE_FEATURE_DSC',
    'formtype'    => 'yesno',
    'valuetype'   => 'int',
    'default'     => 1
];

// Blocks
$modversion['blocks'][] = [
    'file'        => 'recent_items.php',
    'name'        => _MI_MODULENAME_BLOCK_RECENT,
    'description' => _MI_MODULENAME_BLOCK_RECENT_DSC,
    'show_func'   => 'modulename_recent_items_show',
    'edit_func'   => 'modulename_recent_items_edit',
    'options'     => '5|0',
    'template'    => 'modulename_block_recent.html'
];

// Templates
$modversion['templates'][] = [
    'file'        => 'modulename_index.html',
    'description' => 'Main index template'
];

$modversion['templates'][] = [
    'file'        => 'modulename_single.html',
    'description' => 'Single item view'
];

// Notification Categories
$modversion['notification']['category'][] = [
    'name'           => 'global',
    'title'          => _MI_MODULENAME_NOTIFY_GLOBAL,
    'description'    => _MI_MODULENAME_NOTIFY_GLOBAL_DSC,
    'subscribe_from' => ['index.php', 'entity.php']
];

// Notification Events
$modversion['notification']['event'][] = [
    'name'          => 'new_entity',
    'category'      => 'global',
    'title'         => _MI_MODULENAME_NOTIFY_NEW_ENTITY,
    'caption'       => _MI_MODULENAME_NOTIFY_NEW_ENTITY_CAP,
    'description'   => _MI_MODULENAME_NOTIFY_NEW_ENTITY_DSC',
    'mail_template' => 'new_entity_notify',
    'mail_subject'  => _MI_MODULENAME_NOTIFY_NEW_ENTITY_MAIL_SBJ
];
```

### 2. admin/index.php

Main admin interface page.

```php
<?php
/**
 * Admin index page
 */

defined("ICMS_ROOT_PATH") or die("ImpressCMS root path not defined");

icms_loadLanguageFile('modulename', 'admin');

// Include common admin functions
include_once ICMS_ROOT_PATH . '/modules/modulename/include/common.php';

// Check admin permissions
if (!icms::$user || !icms::$user->isAdmin(icms::$module->getVar('mid'))) {
    redirect_header(ICMS_URL, 3, _NOPERM);
}

// Create admin page
$icmsAdminTpl = new icms_view_Tpl();
$icmsAdminTpl->assign('modulename_admin_url', ICMS_URL . '/modules/modulename/admin/');

// Display dashboard or statistics
$icmsAdminTpl->display('db:modulename_admin_index.html');
```


### 3. admin/admin_menu.php

Defines the admin menu structure.

```php
<?php
/**
 * Admin menu definition
 */

defined("ICMS_ROOT_PATH") or die("ImpressCMS root path not defined");

$adminmenu = [];

$adminmenu[] = [
    'title' => _AM_MODULENAME_DASHBOARD,
    'link'  => 'admin/index.php'
];

$adminmenu[] = [
    'title' => _AM_MODULENAME_ENTITIES,
    'link'  => 'admin/entity.php'
];

$adminmenu[] = [
    'title' => _AM_MODULENAME_CATEGORIES,
    'link'  => 'admin/category.php'
];

$adminmenu[] = [
    'title' => _AM_MODULENAME_ABOUT,
    'link'  => 'admin/about.php'
];
```

### 4. header.php & footer.php

Standard header and footer for module pages.

**header.php:**
```php
<?php
/**
 * Module header
 */

defined("ICMS_ROOT_PATH") or die("ImpressCMS root path not defined");

include_once ICMS_ROOT_PATH . '/header.php';
include_once ICMS_ROOT_PATH . '/modules/modulename/include/common.php';

// Module-specific initialization
$modulename_handler = icms_getModuleHandler('entity', 'modulename');
```

**footer.php:**
```php
<?php
/**
 * Module footer
 */

defined("ICMS_ROOT_PATH") or die("ImpressCMS root path not defined");

include_once ICMS_ROOT_PATH . '/footer.php';
```

---

## Directory Structure

### File Status Reference

| File/Directory | Status | Purpose |
|----------------|--------|---------|
| `icms_version.php` | **MANDATORY** | Module configuration and metadata |
| `admin/` | **MANDATORY** (if hasAdmin=1) | Admin interface files |
| `admin/index.php` | **MANDATORY** | Main admin page |
| `admin/admin_menu.php` | **MANDATORY** | Admin menu definition |
| `class/` | **RECOMMENDED** | Module classes (IPF Objects/Handlers) |
| `templates/` | **MANDATORY** | Smarty templates |
| `language/` | **MANDATORY** | Language files |
| `language/english/` | **MANDATORY** | English language (fallback) |
| `language/english/modinfo.php` | **MANDATORY** | Module info translations |
| `sql/` | **MANDATORY** (if using DB) | Database schema files |
| `sql/mysql.sql` | **MANDATORY** | MySQL schema |
| `images/` | **RECOMMENDED** | Module images and icons |
| `blocks/` | Optional | Block implementations |
| `include/` | **RECOMMENDED** | Helper functions and includes |
| `docs/` | **RECOMMENDED** | Documentation files |
| `index.php` | **MANDATORY** (if hasMain=1) | Main module page |
| `header.php` | **RECOMMENDED** | Module header |
| `footer.php` | **RECOMMENDED** | Module footer |
| `module.css` | Optional | Module stylesheet |
| `rss.php` | Optional | RSS feed |
| `comment_*.php` | **MANDATORY** (if hasComments=1) | Comment handlers |
| `notification_update.php` | **MANDATORY** (if hasNotification=1) | Notification handler |

---

## Best Practices

### PHP 8.2+ Standards

#### 1. Use Typed Properties

```php
<?php
class ModulenameEntity extends icms_ipf_Object
{
    public string $title;
    public ?string $description = null;
    public int $status = 1;
    public \DateTime $createdAt;

    public function __construct(icms_ipf_Handler &$handler)
    {
        parent::__construct($handler);
        $this->initVar('title', XOBJ_DTYPE_TXTBOX, '', true, 255);
        $this->initVar('description', XOBJ_DTYPE_TXTAREA, '');
        $this->initVar('status', XOBJ_DTYPE_INT, 1);
        $this->initVar('created', XOBJ_DTYPE_LTIME, time());
    }
}
```

#### 2. Constructor Property Promotion

```php
<?php
class ModulenameService
{
    public function __construct(
        private readonly icms_ipf_Handler $handler,
        private readonly icms_core_Security $security,
        private ?array $config = null
    ) {
        $this->config ??= icms::$module->config;
    }
}
```

#### 3. Return Type Declarations

```php
<?php
class ModulenameEntityHandler extends icms_ipf_Handler
{
    public function getPublishedEntities(int $limit = 10): array
    {
        $criteria = new icms_db_criteria_Item('status', 1);
        $criteria->setLimit($limit);
        $criteria->setSort('created');
        $criteria->setOrder('DESC');

        return $this->getObjects($criteria);
    }

    public function getEntityById(int $id): ?ModulenameEntity
    {
        return $this->get($id);
    }

    public function deleteEntity(int $id): bool
    {
        return $this->delete($id);
    }
}
```


#### 4. Nullable Types

Use short nullable syntax: `?Type` instead of `Type|null`

```php
<?php
public function findBySlug(string $slug): ?ModulenameEntity
{
    $criteria = new icms_db_criteria_Item('slug', $slug);
    $entities = $this->getObjects($criteria);

    return $entities[0] ?? null;
}
```

#### 5. Void Return Types

Always specify `void` for methods that return nothing:

```php
<?php
public function sendNotification(ModulenameEntity $entity): void
{
    $notification_handler = icms::handler('icms_data_notification');
    $notification_handler->triggerEvent(
        'global',
        0,
        'new_entity',
        ['ENTITY_TITLE' => $entity->getVar('title')]
    );
}
```

### Naming Conventions

- **Classes:** PascalCase (`ModulenameEntity`, `ModulenameEntityHandler`)
- **Methods/Variables:** camelCase (`getUserName()`, `$firstName`)
- **Constants:** UPPER_SNAKE_CASE (`MODULENAME_STATUS_ACTIVE`)
- **Database Tables:** lowercase_snake_case (`modulename_entity`)
- **Language Constants:** `_PREFIX_SECTION_NAME` (e.g., `_MI_MODULENAME_NAME`)

### Control Flow Best Practices

#### Happy Path Last

Handle error conditions first, success case last:

```php
<?php
public function saveEntity(array $data): bool
{
    if (!icms::$user) {
        return false;
    }

    if (!$this->validateData($data)) {
        return false;
    }

    $entity = $this->create();
    $entity->setVar('title', $data['title']);

    return $this->insert($entity);
}
```

#### Avoid Else Statements

Use early returns instead:

```php
<?php
public function canEdit(ModulenameEntity $entity): bool
{
    if (!icms::$user) {
        return false;
    }

    if (icms::$user->isAdmin()) {
        return true;
    }

    return $entity->getVar('uid') === icms::$user->getVar('uid');
}
```

#### Always Use Curly Brackets

Even for single statements:

```php
<?php
if ($condition) {
    doSomething();
}
```

---

## ImpressCMS Core Libraries

### Database Layer (`icms_db_*`)

#### Using Criteria for Queries

```php
<?php
// Simple criteria
$criteria = new icms_db_criteria_Item('status', 1);

// Composite criteria
$criteria = new icms_db_criteria_Compo();
$criteria->add(new icms_db_criteria_Item('status', 1));
$criteria->add(new icms_db_criteria_Item('published', 1));
$criteria->setSort('created');
$criteria->setOrder('DESC');
$criteria->setLimit(10);
$criteria->setStart(0);

// Get objects
$entities = $handler->getObjects($criteria);

// Count objects
$count = $handler->getCount($criteria);
```

#### Direct Database Access (Use Sparingly)

When IPF doesn't fit your needs, use the database layer directly with prepared statements:

```php
<?php
$db = icms::$xoopsDB;

// SELECT with prepared statement
$sql = "SELECT * FROM " . $db->prefix('modulename_entity') . " WHERE status = ? AND category_id = ?";
$result = $db->query($sql, [1, $categoryId]);

while ($row = $db->fetchArray($result)) {
    // Process row
}

// INSERT with prepared statement
$sql = "INSERT INTO " . $db->prefix('modulename_entity') . " (title, description, status) VALUES (?, ?, ?)";
$result = $db->query($sql, [$title, $description, $status]);

// Get last insert ID
$newId = $db->getInsertId();

// UPDATE with prepared statement
$sql = "UPDATE " . $db->prefix('modulename_entity') . " SET status = ? WHERE entity_id = ?";
$result = $db->query($sql, [0, $entityId]);

// DELETE with prepared statement
$sql = "DELETE FROM " . $db->prefix('modulename_entity') . " WHERE entity_id = ?";
$result = $db->query($sql, [$entityId]);
```

### IPF Framework (`icms_ipf_*`)

The IPF (ImpressCMS Persistable Framework) is the cornerstone of rapid module development.

#### Creating an IPF Object

```php
<?php
/**
 * Entity object class
 *
 * @package Modulename
 */
class ModulenameEntity extends icms_ipf_Object
{
    public function __construct(icms_ipf_Handler &$handler)
    {
        parent::__construct($handler);

        $this->quickInitVar('entity_id', XOBJ_DTYPE_INT, true);
        $this->quickInitVar('title', XOBJ_DTYPE_TXTBOX, true);
        $this->quickInitVar('slug', XOBJ_DTYPE_TXTBOX, true);
        $this->quickInitVar('description', XOBJ_DTYPE_TXTAREA);
        $this->quickInitVar('body', XOBJ_DTYPE_TXTAREA);
        $this->quickInitVar('status', XOBJ_DTYPE_INT);
        $this->quickInitVar('category_id', XOBJ_DTYPE_INT);
        $this->quickInitVar('uid', XOBJ_DTYPE_INT);
        $this->quickInitVar('created', XOBJ_DTYPE_LTIME);
        $this->quickInitVar('updated', XOBJ_DTYPE_LTIME);

        // Set identifier and summary fields
        $this->setControl('title', 'text');
        $this->setControl('slug', 'text');
        $this->setControl('description', 'textarea');
        $this->setControl('body', ['name' => 'dhtmltextarea']);
        $this->setControl('status', ['name' => 'select', 'itemHandler' => 'entity', 'method' => 'getStatusArray']);
        $this->setControl('category_id', ['name' => 'select', 'itemHandler' => 'category', 'method' => 'getCategoriesArray']);

        $this->IcmsPersistableObject($handler);
    }

    /**
     * Get status options
     */
    public function getStatusArray(): array
    {
        return [
            0 => _CO_MODULENAME_ENTITY_STATUS_DRAFT,
            1 => _CO_MODULENAME_ENTITY_STATUS_PUBLISHED,
            2 => _CO_MODULENAME_ENTITY_STATUS_ARCHIVED
        ];
    }

    /**
     * Get item link
     */
    public function getItemLink(): string
    {
        return '<a href="' . ICMS_URL . '/modules/modulename/entity.php?entity_id=' . $this->getVar('entity_id') . '">' . $this->getVar('title') . '</a>';
    }
}
```


#### Creating an IPF Handler

```php
<?php
/**
 * Entity handler class
 *
 * @package Modulename
 */
class ModulenameEntityHandler extends icms_ipf_Handler
{
    public function __construct(icms_db_legacy_Database &$db)
    {
        parent::__construct(
            $db,
            'entity',              // Item name
            'entity_id',           // Primary key
            'title',               // Identifier field
            'description',         // Summary field
            'modulename'           // Module dirname
        );

        // Enable uploads if needed
        $this->enableUpload(['image/png', 'image/jpeg', 'image/gif'], 2000000, 800, 600);
    }

    /**
     * Get published entities
     */
    public function getPublishedEntities(int $limit = 0, int $start = 0, ?int $categoryId = null): array
    {
        $criteria = new icms_db_criteria_Compo();
        $criteria->add(new icms_db_criteria_Item('status', 1));

        if ($categoryId !== null) {
            $criteria->add(new icms_db_criteria_Item('category_id', $categoryId));
        }

        $criteria->setSort('created');
        $criteria->setOrder('DESC');

        if ($limit > 0) {
            $criteria->setLimit($limit);
            $criteria->setStart($start);
        }

        return $this->getObjects($criteria);
    }

    /**
     * Before insert event
     */
    protected function beforeInsert(&$obj): bool
    {
        // Auto-generate slug if empty
        if (empty($obj->getVar('slug'))) {
            $obj->setVar('slug', $this->generateSlug($obj->getVar('title')));
        }

        // Set created timestamp
        $obj->setVar('created', time());
        $obj->setVar('updated', time());

        // Set current user
        if (icms::$user) {
            $obj->setVar('uid', icms::$user->getVar('uid'));
        }

        return true;
    }

    /**
     * Before update event
     */
    protected function beforeUpdate(&$obj): bool
    {
        // Update timestamp
        $obj->setVar('updated', time());

        return true;
    }

    /**
     * After insert event
     */
    protected function afterInsert(&$obj): void
    {
        // Send notification
        $notification_handler = icms::handler('icms_data_notification');
        $notification_handler->triggerEvent(
            'global',
            0,
            'new_entity',
            [
                'ENTITY_TITLE' => $obj->getVar('title'),
                'ENTITY_URL' => $obj->getItemLink()
            ]
        );
    }

    /**
     * Generate URL-friendly slug
     */
    private function generateSlug(string $title): string
    {
        $slug = strtolower(trim($title));
        $slug = preg_replace('/[^a-z0-9-]/', '-', $slug);
        $slug = preg_replace('/-+/', '-', $slug);

        return trim($slug, '-');
    }
}
```

#### Using IPF Controller

The IPF Controller handles form submissions and CRUD operations:

```php
<?php
// In admin/entity.php

include_once '../../../include/cp_header.php';
include_once ICMS_ROOT_PATH . '/modules/modulename/include/common.php';

$modulename_entity_handler = icms_getModuleHandler('entity', 'modulename');

// Create controller
$icmspersistableController = new icms_ipf_Controller($modulename_entity_handler);

// Process operations
$icmspersistableController->handleObjectActions();
```

### Form Layer (`icms_form_*`)

#### Creating Forms

```php
<?php
// Create a form
$form = new icms_form_Theme(_AM_MODULENAME_ENTITY_CREATE, 'entity_form', 'entity.php', 'post', true);

// Add elements
$form->addElement(new icms_form_elements_Text(_AM_MODULENAME_ENTITY_TITLE, 'title', 50, 255, $entity->getVar('title', 'e')), true);

$form->addElement(new icms_form_elements_Textarea(_AM_MODULENAME_ENTITY_DESCRIPTION, 'description', $entity->getVar('description', 'e'), 5, 50));

$form->addElement(new icms_form_elements_Editor(_AM_MODULENAME_ENTITY_BODY, 'body', $entity->getVar('body', 'e')));

$form->addElement(new icms_form_elements_Select(_AM_MODULENAME_ENTITY_STATUS, 'status', $entity->getVar('status'), $entity->getStatusArray()));

$form->addElement(new icms_form_elements_Select(_AM_MODULENAME_ENTITY_CATEGORY, 'category_id', $entity->getVar('category_id'), $category_handler->getCategoriesArray()));

// Add hidden fields
$form->addElement(new icms_form_elements_Hidden('entity_id', $entity->getVar('entity_id')));
$form->addElement(new icms_form_elements_Hidden('op', 'save'));

// Add buttons
$form->addElement(new icms_form_elements_Button('', 'submit', _SUBMIT, 'submit'));

// Display form
$form->display();
```

#### IPF Auto-Generated Forms

IPF can automatically generate forms from object definitions:

```php
<?php
$entity = $modulename_entity_handler->get($entityId);
$form = $entity->getForm(_AM_MODULENAME_ENTITY_EDIT, 'entity_form');
$form->display();
```

### View Layer (`icms_view_*` and `icms_ipf_view_*`)

#### Using IPF Table View

```php
<?php
// Create table
$objectTable = new icms_ipf_view_Table($modulename_entity_handler, false, ['title', 'status', 'created']);

// Add filters
$objectTable->addFilter('status', 'getStatusArray');
$objectTable->addFilter('category_id', 'category', 'getCategoriesArray');

// Add actions
$objectTable->addActionButton('edit', false, _EDIT);
$objectTable->addActionButton('delete', false, _DELETE);

// Add intro button
$objectTable->addIntroButton('create', 'entity.php?op=mod', _AM_MODULENAME_ENTITY_CREATE);

// Fetch and display
$objectTable->fetchObjects();
$table = $objectTable->render();

$icmsAdminTpl->assign('modulename_entity_table', $table);
```

### Security (`icms_core_Security`)

#### CSRF Protection

Always use security tokens in forms:

```php
<?php
// In form
$form->addElement(new icms_form_elements_Hiddentoken());

// In processing
if (!icms::$security->check()) {
    redirect_header('index.php', 3, _NOPERM);
}
```

#### Input Validation

```php
<?php
// Sanitize input
$title = icms_core_DataFilter::checkVar($_POST['title'], 'str');
$status = icms_core_DataFilter::checkVar($_POST['status'], 'int');

// HTML filtering
$body = icms_core_HTMLFilter::filterHTML($_POST['body']);
```

#### Permission Checks

```php
<?php
// Check if user is admin
if (!icms::$user || !icms::$user->isAdmin(icms::$module->getVar('mid'))) {
    redirect_header(ICMS_URL, 3, _NOPERM);
}

// Check specific permission
$gperm_handler = icms::handler('icms_member_groupperm');
$groups = icms::$user ? icms::$user->getGroups() : [ICMS_GROUP_ANONYMOUS];

if (!$gperm_handler->checkRight('modulename_submit', 0, $groups, icms::$module->getVar('mid'))) {
    redirect_header('index.php', 3, _NOPERM);
}
```

---

## Security

### SQL Injection Prevention

**ALWAYS use prepared statements or IPF methods:**

```php
<?php
// ✅ GOOD: Using IPF
$entity = $handler->get($entityId);

// ✅ GOOD: Using prepared statements
$sql = "SELECT * FROM " . $db->prefix('modulename_entity') . " WHERE entity_id = ?";
$result = $db->query($sql, [$entityId]);

// ❌ BAD: Direct SQL injection vulnerability
$sql = "SELECT * FROM " . $db->prefix('modulename_entity') . " WHERE entity_id = " . $_GET['id'];
$result = $db->query($sql);
```


### XSS Prevention

Always escape output:

```php
<?php
// In PHP
echo htmlspecialchars($entity->getVar('title'), ENT_QUOTES, 'UTF-8');

// Or use ImpressCMS methods
echo $entity->getVar('title', 'e');  // 'e' = escaped
echo $entity->getVar('title', 's');  // 's' = sanitized
```

In Smarty templates:

```smarty
{* Escaped by default *}
{$entity.title}

{* Raw output (use with caution) *}
{$entity.body|smarty:nodefaults}
```

### File Upload Security

```php
<?php
// Validate file type
$allowed_mimetypes = ['image/png', 'image/jpeg', 'image/gif'];
$uploader = new icms_file_MediaUploadHandler($upload_dir, $allowed_mimetypes, $max_size, $max_width, $max_height);

if ($uploader->fetchMedia($_POST['xoops_upload_file'][0])) {
    if (!$uploader->upload()) {
        $entity->setErrors($uploader->getErrors());
    } else {
        $entity->setVar('image', $uploader->getSavedFileName());
    }
}
```

---

## Code Examples

### Complete CRUD Module Example

#### 1. Database Schema (sql/mysql.sql)

```sql
CREATE TABLE IF NOT EXISTS `modulename_entity` (
    `entity_id` int(11) unsigned NOT NULL AUTO_INCREMENT,
    `title` varchar(255) NOT NULL DEFAULT '',
    `slug` varchar(255) NOT NULL DEFAULT '',
    `description` text,
    `body` text,
    `status` tinyint(1) NOT NULL DEFAULT '0',
    `category_id` int(11) unsigned NOT NULL DEFAULT '0',
    `uid` int(11) unsigned NOT NULL DEFAULT '0',
    `created` int(11) unsigned NOT NULL DEFAULT '0',
    `updated` int(11) unsigned NOT NULL DEFAULT '0',
    PRIMARY KEY (`entity_id`),
    KEY `status` (`status`),
    KEY `category_id` (`category_id`),
    KEY `uid` (`uid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS `modulename_category` (
    `category_id` int(11) unsigned NOT NULL AUTO_INCREMENT,
    `name` varchar(255) NOT NULL DEFAULT '',
    `description` text,
    `weight` int(11) NOT NULL DEFAULT '0',
    PRIMARY KEY (`category_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

#### 2. Language File (language/english/modinfo.php)

```php
<?php
/**
 * Module info language file
 */

defined("ICMS_ROOT_PATH") or die("ImpressCMS root path not defined");

// Module Info
define('_MI_MODULENAME_NAME', 'Module Name');
define('_MI_MODULENAME_DESC', 'Module Description');

// Menu
define('_MI_MODULENAME_MENU_HOME', 'Home');
define('_MI_MODULENAME_MENU_SUBMIT', 'Submit');

// Config
define('_MI_MODULENAME_ITEMS_PER_PAGE', 'Items per page');
define('_MI_MODULENAME_ITEMS_PER_PAGE_DSC', 'Number of items to display per page');

// Blocks
define('_MI_MODULENAME_BLOCK_RECENT', 'Recent Items');
define('_MI_MODULENAME_BLOCK_RECENT_DSC', 'Display recent items');

// Notifications
define('_MI_MODULENAME_NOTIFY_GLOBAL', 'Global');
define('_MI_MODULENAME_NOTIFY_GLOBAL_DSC', 'Global notifications');
define('_MI_MODULENAME_NOTIFY_NEW_ENTITY', 'New Entity');
define('_MI_MODULENAME_NOTIFY_NEW_ENTITY_CAP', 'Notify me when a new entity is published');
define('_MI_MODULENAME_NOTIFY_NEW_ENTITY_DSC', 'Receive notification when a new entity is published');
define('_MI_MODULENAME_NOTIFY_NEW_ENTITY_MAIL_SBJ', '[{X_SITENAME}] New entity: {ENTITY_TITLE}');
```

#### 3. Block Implementation (blocks/recent_items.php)

```php
<?php
/**
 * Recent items block
 */

defined("ICMS_ROOT_PATH") or die("ImpressCMS root path not defined");

function modulename_recent_items_show(array $options): array
{
    $block = [];

    $modulename_entity_handler = icms_getModuleHandler('entity', 'modulename');

    $limit = (int)$options[0];
    $categoryId = (int)$options[1] ?: null;

    $entities = $modulename_entity_handler->getPublishedEntities($limit, 0, $categoryId);

    foreach ($entities as $entity) {
        $block['entities'][] = [
            'id' => $entity->getVar('entity_id'),
            'title' => $entity->getVar('title'),
            'description' => $entity->getVar('description'),
            'url' => ICMS_URL . '/modules/modulename/entity.php?entity_id=' . $entity->getVar('entity_id'),
            'date' => formatTimestamp($entity->getVar('created'), 's')
        ];
    }

    return $block;
}

function modulename_recent_items_edit(array $options): string
{
    $form = '<table>';

    // Limit option
    $form .= '<tr><td>' . _MB_MODULENAME_LIMIT . '</td><td>';
    $form .= '<input type="text" name="options[0]" value="' . $options[0] . '" /></td></tr>';

    // Category filter
    $form .= '<tr><td>' . _MB_MODULENAME_CATEGORY . '</td><td>';
    $form .= '<select name="options[1]">';
    $form .= '<option value="0">' . _MB_MODULENAME_ALL_CATEGORIES . '</option>';

    $category_handler = icms_getModuleHandler('category', 'modulename');
    $categories = $category_handler->getObjects();

    foreach ($categories as $category) {
        $selected = ($options[1] == $category->getVar('category_id')) ? ' selected="selected"' : '';
        $form .= '<option value="' . $category->getVar('category_id') . '"' . $selected . '>';
        $form .= $category->getVar('name') . '</option>';
    }

    $form .= '</select></td></tr>';
    $form .= '</table>';

    return $form;
}
```

#### 4. Template Example (templates/modulename_index.html)

```smarty
<{* Module index template *}>

<div class="modulename-index">
    <h1><{$modulename_title}></h1>

    <{if $modulename_categories}>
    <div class="modulename-categories">
        <h2><{$smarty.const._MD_MODULENAME_CATEGORIES}></h2>
        <ul>
        <{foreach from=$modulename_categories item=category}>
            <li>
                <a href="<{$icms_url}>/modules/modulename/index.php?category_id=<{$category.id}>">
                    <{$category.name}>
                </a>
                (<{$category.count}>)
            </li>
        <{/foreach}>
        </ul>
    </div>
    <{/if}>

    <{if $modulename_entities}>
    <div class="modulename-entities">
        <{foreach from=$modulename_entities item=entity}>
        <article class="modulename-entity">
            <h2>
                <a href="<{$icms_url}>/modules/modulename/entity.php?entity_id=<{$entity.id}>">
                    <{$entity.title}>
                </a>
            </h2>

            <div class="entity-meta">
                <span class="entity-date"><{$entity.date}></span>
                <span class="entity-author"><{$entity.author}></span>
                <{if $entity.category}>
                <span class="entity-category">
                    <a href="<{$icms_url}>/modules/modulename/index.php?category_id=<{$entity.category.id}>">
                        <{$entity.category.name}>
                    </a>
                </span>
                <{/if}>
            </div>

            <div class="entity-description">
                <{$entity.description}>
            </div>

            <a href="<{$icms_url}>/modules/modulename/entity.php?entity_id=<{$entity.id}>" class="read-more">
                <{$smarty.const._MD_MODULENAME_READ_MORE}>
            </a>
        </article>
        <{/foreach}>
    </div>

    <{if $modulename_pagenav}>
    <div class="modulename-pagenav">
        <{$modulename_pagenav}>
    </div>
    <{/if}>
    <{else}>
    <p><{$smarty.const._MD_MODULENAME_NO_ENTITIES}></p>
    <{/if}>
</div>
```

---

## Testing

### Unit Testing with Pest

ImpressCMS modules should use Pest 4 for testing:

```php
<?php
// tests/Unit/EntityHandlerTest.php

use function Pest\Laravel\{get, post};

beforeEach(function () {
    $this->handler = icms_getModuleHandler('entity', 'modulename');
});

test('can create entity', function () {
    $entity = $this->handler->create();
    $entity->setVar('title', 'Test Entity');
    $entity->setVar('status', 1);

    $result = $this->handler->insert($entity);

    expect($result)->toBeTrue();
    expect($entity->getVar('entity_id'))->toBeGreaterThan(0);
});

test('can retrieve entity by id', function () {
    $entity = $this->handler->create();
    $entity->setVar('title', 'Test Entity');
    $this->handler->insert($entity);

    $retrieved = $this->handler->get($entity->getVar('entity_id'));

    expect($retrieved)->toBeInstanceOf(ModulenameEntity::class);
    expect($retrieved->getVar('title'))->toBe('Test Entity');
});

test('can get published entities', function () {
    // Create test entities
    for ($i = 0; $i < 5; $i++) {
        $entity = $this->handler->create();
        $entity->setVar('title', "Entity $i");
        $entity->setVar('status', 1);
        $this->handler->insert($entity);
    }

    $entities = $this->handler->getPublishedEntities(3);

    expect($entities)->toHaveCount(3);
});
```

---

## Documentation

### PHPDoc Standards

```php
<?php
/**
 * Entity handler class
 *
 * Manages CRUD operations for entities
 *
 * @package    Modulename
 * @subpackage Handlers
 * @author     Your Name <your@email.com>
 * @copyright  2025 Your Organization
 * @license    GPL 2.0
 * @version    1.0.0
 * @link       https://example.com/modulename
 */
class ModulenameEntityHandler extends icms_ipf_Handler
{
    /**
     * Get published entities
     *
     * Retrieves entities with status = 1, optionally filtered by category
     *
     * @param int      $limit      Maximum number of entities to return (0 = all)
     * @param int      $start      Offset for pagination
     * @param int|null $categoryId Optional category filter
     *
     * @return ModulenameEntity[] Array of entity objects
     */
    public function getPublishedEntities(int $limit = 0, int $start = 0, ?int $categoryId = null): array
    {
        // Implementation
    }
}
```

### README.md Template

```markdown
# Module Name

Brief description of what the module does.

## Features

- Feature 1
- Feature 2
- Feature 3

## Requirements

- ImpressCMS 2.0+
- PHP 8.2+
- MySQL 5.7+ or MariaDB 10.2+

## Installation

1. Download the module
2. Extract to `/modules/modulename/`
3. Go to System Admin > Modules
4. Install the module

## Configuration

Configure the module in System Admin > Modules > Module Name > Preferences

## Usage

Describe how to use the module

## Support

- Documentation: https://example.com/docs
- Issues: https://github.com/user/modulename/issues

## License

GPL 2.0

## Credits

- Author: Your Name
- Contributors: List contributors
```

---

## Summary

This guide provides a comprehensive foundation for developing modern ImpressCMS modules. Key takeaways:

1. **Use IPF Framework** - Leverage `icms_ipf_*` classes for rapid development
2. **Follow PHP 8.2+ Standards** - Use typed properties, return types, and modern syntax
3. **Security First** - Always use prepared statements, validate input, escape output
4. **Leverage Core Libraries** - Maximize use of `icms_db_*`, `icms_form_*`, `icms_view_*` classes
5. **Follow Conventions** - Adhere to ImpressCMS naming and structure conventions
6. **Document Everything** - Use PHPDoc, README files, and inline comments
7. **Test Thoroughly** - Write unit tests with Pest 4

For more information, consult the ImpressCMS documentation and examine the example modules referenced in this guide.


