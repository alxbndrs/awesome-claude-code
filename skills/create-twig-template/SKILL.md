---
name: create-twig-template
description: Generates Symfony Twig templates at the proper hierarchical location under ./templates/. Enforces context-based organization (templates/{purpose}/{bounded-context}/{type}.html.twig or templates/{purpose}/{type}/{bounded-context}.html.twig — user chooses one style per purpose) with optional component/subcomponent nesting. Refuses flat-folder anti-patterns where templates from different bounded contexts share a single folder or where every template sits at the templates/ root.
---

# Twig Template Generator

Generates Twig templates under `./templates/` following a hierarchical organization that keeps templates from different bounded contexts isolated and discoverable. As the template count grows from tens into hundreds, the hierarchy is what prevents the directory from becoming unmanageable.

## When to Use

- Adding a transactional email template (signup confirmation, password reset, payment receipt)
- Creating an export view (PDF report, CSV download, HTML printable)
- Adding a frontend page, layout, or partial
- Adding admin-panel views
- Adding any new Twig template that belongs to a specific feature or bounded context

## Folder Structure

### Mandatory top-level shape

```
templates/
├── {purpose}/                     # required — what the template is for
│   └── {bounded-context}/         # required — which feature it belongs to
│       └── ...                    # optional — components, subcomponents
└── base.html.twig                 # optional — global base layout
```

A template that touches a specific bounded context MUST live under that context's subfolder. Templates that are genuinely shared across contexts (a global base layout, generic error pages) live at the purpose root or the templates root.

### Two valid placement styles inside a `{purpose}` folder

Pick ONE per purpose block and stick with it.

#### Style A — context-first (default; recommended for feature-team isolation)

```
templates/{purpose}/{bounded-context}/{type}.html.twig
templates/{purpose}/{bounded-context}/{component}/{type}.html.twig
templates/{purpose}/{bounded-context}/{component}/{subcomponent}/{type}.html.twig
```

Best when each bounded context owns multiple templates and a feature team wants to find "all the Order templates" in one place.

#### Style B — type-first

```
templates/{purpose}/{type}/{bounded-context}.html.twig
```

Best when the same `{type}` repeats across several contexts and shares a layout (e.g. a single `report.html.twig` shape rendered for `user`, `order`, `payment` data).

Mixing both inside the same `{purpose}` folder is an anti-pattern (see below).

## Examples

### Emails — Style A, with a deeper hierarchy under `user/`

```
templates/emails/
├── base.html.twig                            # shared email layout
├── user/
│   ├── signup/
│   │   ├── successful.html.twig
│   │   └── confirm_email.html.twig
│   ├── password/
│   │   ├── reset_requested.html.twig
│   │   └── reset_completed.html.twig
│   └── threshold_reached.html.twig
├── order/
│   ├── confirmation.html.twig
│   └── shipped.html.twig
└── payment/
    └── failed.html.twig
```

### Export — Style A, with a `component` level (`transactions`)

```
templates/export/
└── user/
    └── transactions/
        ├── report.html.twig
        ├── report.csv.twig
        └── summary.pdf.twig
```

### Frontend — Style A, with `component/subcomponent` hierarchy (`dashboard/widgets/`)

```
templates/frontend/
├── base.html.twig
└── user/
    └── dashboard/
        ├── index.html.twig
        └── widgets/
            ├── _recent-activity.html.twig
            └── _balance-card.html.twig
```

### Generic report — Style B (type-first, when the type repeats across contexts)

```
templates/export/
└── report/
    ├── user.html.twig
    ├── order.html.twig
    └── payment.html.twig
```

(All three reuse the same report layout/shape; the only thing that differs is the data and labels per context.)

## Generation Process

### Step 1 — Determine the purpose

Common purposes (pick the matching folder name, plural noun):

| Purpose folder | Use for |
|---|---|
| `emails/` | Transactional and system emails |
| `export/` | Generated reports (PDF, CSV, HTML printable, XML) |
| `frontend/` | Public-facing pages and partials |
| `admin/` | Admin panel pages |
| `pdf/` | PDF documents (when separate from `export/`) |
| `sms/` | SMS message bodies |
| `notifications/` | In-app notifications |
| `bundles/` | Bundle override templates (Symfony bundle template overrides) |

### Step 2 — Determine the bounded context

The DDD bounded context this template belongs to: `user/`, `order/`, `payment/`, `inventory/`, `subscription/`, etc. Use lowercase singular.

### Step 3 — Determine the type / filename

The specific template — what it actually represents: `confirmation`, `report`, `index`, `successful`, `confirm_email`, `dashboard`, etc.

### Step 4 — Determine the hierarchy depth

- Flat: `{purpose}/{BC}/{type}.html.twig`
- Component-grouped: `{purpose}/{BC}/{component}/{type}.html.twig`
- Deep: `{purpose}/{BC}/{component}/{subcomponent}/{type}.html.twig`

Add a `{component}` level when the bounded context naturally has multiple related templates that share a sub-area (e.g. `user/signup/` for all signup-flow templates).

### Step 5 — Pick the placement style

- **Style A (default)** — `{purpose}/{BC}/{type}` keeps a bounded context's templates together
- **Style B** — `{purpose}/{type}/{BC}` keeps the same type across contexts together

### Step 6 — Pick the format

| Extension | Renders to |
|---|---|
| `.html.twig` | HTML (default) |
| `.txt.twig` | Plain text (alongside `.html.twig` for multipart emails) |
| `.csv.twig` | CSV export |
| `.xml.twig` | XML feed |
| `.json.twig` | JSON (rare — usually serialize from PHP instead) |
| `.pdf.twig` | HTML rendered to PDF by a renderer (Dompdf, wkhtmltopdf, etc.) |

### Step 7 — Generate the file with starter content

Starter content matches the purpose:
- Emails: extend the email base, define `subject` and `body` blocks
- Frontend: extend the site base, define `title` and `content` blocks
- Export: extend an export base if one exists, define `content` block
- Partials (prefixed `_`): no `extends`, just the fragment

## Anti-patterns to avoid

The generator REFUSES to produce any of these layouts:

### 1. Flat folder mixing bounded contexts

```
❌ templates/emails/
   ├── signup_successful.html.twig
   ├── confirm_email.html.twig
   ├── order_confirmation.html.twig
   ├── payment_failed.html.twig
   └── threshold_reached.html.twig
```

**Why:** templates from User, Order, and Payment contexts share a single folder; the filename prefix carries context info that should be in folder hierarchy. As contexts grow, the folder explodes and ownership becomes unclear.

**Fix:** group by bounded context first.
```
✓ templates/emails/
  ├── user/
  │   ├── signup/successful.html.twig
  │   ├── signup/confirm_email.html.twig
  │   └── threshold_reached.html.twig
  ├── order/
  │   └── confirmation.html.twig
  └── payment/
      └── failed.html.twig
```

### 2. Every template at the templates root

```
❌ templates/
   ├── user_dashboard.html.twig
   ├── user_profile.html.twig
   ├── order_list.html.twig
   ├── order_details.html.twig
   └── payment_form.html.twig
```

**Why:** no purpose layer, no context layer — everything competes for one namespace. Discoverability is zero.

**Fix:** introduce purpose AND context layers.
```
✓ templates/
  ├── frontend/
  │   ├── user/
  │   │   ├── dashboard.html.twig
  │   │   └── profile.html.twig
  │   ├── order/
  │   │   ├── list.html.twig
  │   │   └── details.html.twig
  │   └── payment/
  │       └── form.html.twig
  └── base.html.twig
```

### 3. Mixing Style A and Style B inside the same purpose

```
❌ templates/emails/
   ├── user/                                  # Style A — context-first
   │   └── signup.html.twig
   └── confirmation/                          # Style B — type-first
       └── order.html.twig
```

**Why:** the reader can't predict where a new template lands. Two organizational rules apply at the same level.

**Fix:** pick one style per purpose and apply it consistently.

### 4. Per-template purpose folders

```
❌ templates/
   ├── user_signup_email/
   │   └── successful.html.twig
   └── order_confirmation_email/
       └── default.html.twig
```

**Why:** the folder name reinvents the hierarchy as a single string. The purpose / context / type split is the right way to encode the same information.

**Fix:** use the proper hierarchy.
```
✓ templates/emails/user/signup/successful.html.twig
✓ templates/emails/order/confirmation.html.twig
```

### 5. Cross-context partials shared from one context's folder

```
❌ templates/frontend/
   └── user/
       ├── _shared-table.html.twig             # used by order/ too
       └── dashboard/...
```

**Why:** the partial sits inside `user/` but is also imported from `order/` and `payment/`. The folder ownership is misleading.

**Fix:** move shared partials to a context-neutral location.
```
✓ templates/frontend/
  ├── _shared/
  │   └── _table.html.twig
  └── user/dashboard/...
```

## Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Purpose folder | lowercase, plural noun (category of output) | `emails/`, `export/`, `frontend/` |
| Bounded context folder | lowercase, singular noun | `user/`, `order/`, `payment/` |
| Component / subcomponent | lowercase, kebab-case or single word | `signup/`, `dashboard/`, `recent-activity/` |
| Type / filename | lowercase, snake_case or kebab-case (be consistent within a project) | `confirmation.html.twig`, `recent_activity.html.twig` |
| Base / shared layouts | `base.html.twig` at the purpose root | `templates/emails/base.html.twig` |
| Partials | prefix with `_` | `_table.html.twig`, `_widget.html.twig` |
| Format extension | matches render format | `.html.twig`, `.txt.twig`, `.csv.twig`, `.xml.twig`, `.pdf.twig` |

Project-wide consistency matters more than the specific choice (snake_case vs kebab-case for filenames — pick one and apply it everywhere).

## File Placement

```
templates/
├── base.html.twig                                       # optional global base
├── emails/
│   ├── base.html.twig                                   # email-wide base
│   └── {bounded-context}/{component?}/{type}.html.twig
├── export/
│   └── {bounded-context}/{component?}/{type}.{format}.twig
├── frontend/
│   ├── base.html.twig
│   └── {bounded-context}/{component?}/{type}.html.twig
├── admin/
│   └── {bounded-context}/{component?}/{type}.html.twig
└── ...
```

> The `templates/` folder is the Symfony default. If your project configures `twig.default_path` elsewhere, substitute accordingly — the hierarchy rules don't change.

## Quick Template Reference

### Email — extends the purpose-level base

```twig
{# templates/emails/user/signup/confirm_email.html.twig #}
{% extends 'emails/base.html.twig' %}

{% block subject %}Confirm your email{% endblock %}

{% block body %}
    <h1>Hello {{ user.firstName }}</h1>
    <p>Please confirm your email by clicking the link below:</p>
    <a href="{{ confirmUrl }}">Confirm email</a>
{% endblock %}
```

### Email base layout

```twig
{# templates/emails/base.html.twig #}
<!DOCTYPE html>
<html lang="{{ app.request.locale | default('en') }}">
<head>
    <meta charset="utf-8">
    <title>{{ block('subject') }}</title>
</head>
<body>
    {% block body %}{% endblock %}
    <hr>
    <p>{{ 'email.signature' | trans }}</p>
</body>
</html>
```

### Frontend page

```twig
{# templates/frontend/user/dashboard/index.html.twig #}
{% extends 'base.html.twig' %}

{% block title %}Dashboard{% endblock %}

{% block content %}
    <h1>Welcome, {{ user.firstName }}</h1>
    {{ include('frontend/user/dashboard/widgets/_recent-activity.html.twig', {
        activities: activities,
    }) }}
{% endblock %}
```

### Partial (reusable fragment, prefixed `_`)

```twig
{# templates/frontend/user/dashboard/widgets/_recent-activity.html.twig #}
<ul class="recent-activity">
    {% for item in activities %}
        <li>{{ item.title }} — {{ item.date | date('Y-m-d H:i') }}</li>
    {% endfor %}
</ul>
```

### Export — CSV

```twig
{# templates/export/user/transactions/report.csv.twig #}
"Date";"Description";"Amount";"Balance"
{% for tx in transactions %}
"{{ tx.date | date('Y-m-d') }}";"{{ tx.description | replace({'"': '""'}) }}";"{{ tx.amount }}";"{{ tx.balance }}"
{% endfor %}
```

### Export — PDF-ready HTML

```twig
{# templates/export/user/transactions/report.pdf.twig #}
{% extends 'export/base.pdf.twig' %}

{% block content %}
    <h1>Transactions — {{ user.fullName }}</h1>
    <table>
        <thead>
            <tr><th>Date</th><th>Description</th><th>Amount</th></tr>
        </thead>
        <tbody>
            {% for tx in transactions %}
                <tr>
                    <td>{{ tx.date | date('Y-m-d') }}</td>
                    <td>{{ tx.description }}</td>
                    <td>{{ tx.amount | number_format(2) }}</td>
                </tr>
            {% endfor %}
        </tbody>
    </table>
{% endblock %}
```

## Usage

Provide:
- **Purpose** — `emails`, `frontend`, `export`, `admin`, etc.
- **Bounded context** — `user`, `order`, `payment`, etc.
- **Type / filename** — `confirmation`, `dashboard`, `report`, etc.
- **Optional component / subcomponent** — `signup`, `dashboard/widgets`, `transactions`, etc.
- **Placement style** — A (context-first, default) or B (type-first)
- **Format extension** — `.html.twig` (default), `.txt.twig`, `.csv.twig`, `.pdf.twig`, ...
- **Whether to wire to a base layout** — if a base exists at the purpose root, extend it

The generator will:
1. Resolve the full path under `./templates/` from the inputs
2. Refuse to generate if the resolved path would violate any anti-pattern above
3. Create missing parent directories
4. Write the file with starter content matching the purpose and extension
5. If a matching base layout exists at the purpose root, wire the `{% extends %}` and starter blocks

## References

For broader Symfony patterns (controllers, DI, security, validators), see `acc:symfony-knowledge`. For DDD bounded-context naming, see `acc:ddd-knowledge`.
