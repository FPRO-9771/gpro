# G-Code Generator Web Application - Frontend Build Instructions

This document provides instructions for Claude Code sessions to build the frontend UI for the G-code generator web application.

## Overview

Build a web UI with:
- Project dashboard showing all projects
- Project editor with operation management
- Preview visualization (SVG)
- Settings management pages
- Unsaved changes warning
- G-code download functionality

**Tech Stack**: Jinja2 templates + Bootstrap 5 + Vanilla JavaScript

---

## Template Structure

```
templates/
├── base.html                    # Base template with nav
├── index.html                   # Projects dashboard
├── project/
│   ├── new.html                 # New project form
│   └── edit.html                # Project editor (main workspace)
└── settings/
    ├── index.html               # Settings dashboard
    ├── materials.html           # Materials management
    ├── machine.html             # Machine settings
    └── general.html             # General G-code settings

static/
├── css/
│   └── styles.css               # Custom styles
└── js/
    ├── project-editor.js        # Project editor logic
    └── unsaved-changes.js       # Unsaved changes tracker
```

---

## Base Template (templates/base.html)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}G-Code Generator{% endblock %}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="{{ url_for('static', filename='css/styles.css') }}" rel="stylesheet">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{{ url_for('main.index') }}">FIRST Robotics G-Code</a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <div class="navbar-nav ms-auto">
                    <a class="nav-link" href="{{ url_for('main.index') }}">Projects</a>
                    <a class="nav-link" href="{{ url_for('settings.index') }}">Settings</a>
                </div>
            </div>
        </div>
    </nav>

    <main class="container mt-4">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% for category, message in messages %}
                <div class="alert alert-{{ category }} alert-dismissible fade show">
                    {{ message }}
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            {% endfor %}
        {% endwith %}

        {% block content %}{% endblock %}
    </main>

    <footer class="container mt-5 mb-3 text-center text-muted">
        <small>G-Code Generator for FIRST Robotics</small>
    </footer>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    {% block scripts %}{% endblock %}
</body>
</html>
```

---

## Projects Dashboard (templates/index.html)

```html
{% extends "base.html" %}
{% block title %}Projects - G-Code Generator{% endblock %}

{% block content %}
<div class="d-flex justify-content-between align-items-center mb-4">
    <h1>Projects</h1>
    <a href="{{ url_for('projects.new') }}" class="btn btn-success">
        <i class="bi bi-plus-lg"></i> New Project
    </a>
</div>

{% if projects %}
<div class="row">
    {% for project in projects %}
    <div class="col-md-4 mb-3">
        <div class="card h-100">
            <div class="card-body">
                <h5 class="card-title">{{ project.name }}</h5>
                <p class="card-text">
                    <span class="badge bg-{{ 'info' if project.project_type == 'drill' else 'warning' }}">
                        {{ project.project_type | capitalize }}
                    </span>
                    {% if project.material %}
                    <span class="text-muted">{{ project.material.display_name }}</span>
                    {% endif %}
                </p>
                <p class="card-text">
                    <small class="text-muted">
                        Modified: {{ project.modified_at.strftime('%Y-%m-%d %H:%M') }}
                    </small>
                </p>
            </div>
            <div class="card-footer bg-transparent">
                <a href="{{ url_for('projects.edit', project_id=project.id) }}"
                   class="btn btn-primary btn-sm">Edit</a>
                <a href="{{ url_for('api.download_gcode', project_id=project.id, gcode_type=project.project_type) }}"
                   class="btn btn-outline-success btn-sm">Download</a>
                <form action="{{ url_for('projects.delete', project_id=project.id) }}"
                      method="POST" class="d-inline">
                    <button type="submit" class="btn btn-outline-danger btn-sm"
                            onclick="return confirm('Delete this project?')">Delete</button>
                </form>
            </div>
        </div>
    </div>
    {% endfor %}
</div>
{% else %}
<div class="text-center py-5">
    <p class="text-muted mb-3">No projects yet. Create your first project to get started!</p>
    <a href="{{ url_for('projects.new') }}" class="btn btn-primary btn-lg">Create First Project</a>
</div>
{% endif %}
{% endblock %}
```

---

## New Project Form (templates/project/new.html)

```html
{% extends "base.html" %}
{% block title %}New Project - G-Code Generator{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6">
        <h1 class="mb-4">New Project</h1>

        <form method="POST" action="{{ url_for('projects.create') }}">
            <div class="mb-3">
                <label for="name" class="form-label">Project Name</label>
                <input type="text" class="form-control" id="name" name="name"
                       required placeholder="e.g., Drivetrain Gusset">
            </div>

            <div class="mb-3">
                <label for="project_type" class="form-label">Project Type</label>
                <select class="form-select" id="project_type" name="project_type" required>
                    <option value="">Select type...</option>
                    <option value="drill">Drill - Create holes only</option>
                    <option value="cut">Cut - Create cutouts and outlines</option>
                </select>
                <div class="form-text">
                    Drill projects use drill bits. Cut projects use end mills.
                </div>
            </div>

            <div class="mb-3">
                <label for="material_id" class="form-label">Material</label>
                <select class="form-select" id="material_id" name="material_id" required>
                    <option value="">Select material...</option>
                    {% for material in materials %}
                    <option value="{{ material.id }}">{{ material.display_name }}</option>
                    {% endfor %}
                </select>
            </div>

            <div class="d-grid gap-2">
                <button type="submit" class="btn btn-primary">Create Project</button>
                <a href="{{ url_for('main.index') }}" class="btn btn-outline-secondary">Cancel</a>
            </div>
        </form>
    </div>
</div>
{% endblock %}
```

---

## Project Editor (templates/project/edit.html)

This is the main workspace with three main sections:

```html
{% extends "base.html" %}
{% block title %}{{ project.name }} - G-Code Generator{% endblock %}

{% block content %}
<div class="row">
    <!-- Left Column: Project Info + Operations -->
    <div class="col-lg-5">
        <!-- Project Info Card -->
        <div class="card mb-3">
            <div class="card-header">
                <strong>Project Info</strong>
            </div>
            <div class="card-body">
                <div class="mb-3">
                    <label for="project-name" class="form-label">Name</label>
                    <input type="text" class="form-control" id="project-name"
                           value="{{ project.name }}" data-field="name">
                </div>

                <div class="row">
                    <div class="col-md-6 mb-3">
                        <label for="project-type" class="form-label">Type</label>
                        <select class="form-select" id="project-type" data-field="project_type">
                            <option value="drill" {{ 'selected' if project.project_type == 'drill' }}>Drill</option>
                            <option value="cut" {{ 'selected' if project.project_type == 'cut' }}>Cut</option>
                        </select>
                    </div>
                    <div class="col-md-6 mb-3">
                        <label for="material" class="form-label">Material</label>
                        <select class="form-select" id="material" data-field="material_id">
                            {% for mat in materials %}
                            <option value="{{ mat.id }}" {{ 'selected' if project.material_id == mat.id }}>
                                {{ mat.display_name }}
                            </option>
                            {% endfor %}
                        </select>
                    </div>
                </div>

                <div class="mb-3" id="tool-selection">
                    <label for="tool-size" class="form-label">Tool Size</label>
                    <select class="form-select" id="tool-size" data-field="tool_size">
                        {% for tool in tools %}
                        <option value="{{ tool.size }}"
                                data-type="{{ tool.tool_type }}"
                                {{ 'selected' if (project.drill_bit_size == tool.size or project.end_mill_size == tool.size) }}>
                            {{ tool.description }}
                        </option>
                        {% endfor %}
                    </select>
                </div>

                <!-- Tube Void Skip (shown for tube materials) -->
                <div class="form-check mb-3" id="tube-void-section" style="display: none;">
                    <input class="form-check-input" type="checkbox" id="tube-void-skip"
                           data-field="tube_void_skip" {{ 'checked' if project.tube_void_skip }}>
                    <label class="form-check-label" for="tube-void-skip">
                        Skip tube void (hollow center)
                    </label>
                </div>
            </div>
        </div>

        <!-- Operations Card -->
        <div class="card mb-3">
            <div class="card-header d-flex justify-content-between align-items-center">
                <strong>Operations</strong>
                <div class="btn-group btn-group-sm">
                    <button class="btn btn-outline-primary" data-bs-toggle="modal"
                            data-bs-target="#addDrillModal" id="btn-add-drill">
                        + Drill
                    </button>
                    <button class="btn btn-outline-primary" data-bs-toggle="modal"
                            data-bs-target="#addCircleModal" id="btn-add-circle">
                        + Circle
                    </button>
                    <button class="btn btn-outline-primary" data-bs-toggle="modal"
                            data-bs-target="#addLineModal" id="btn-add-line">
                        + Line
                    </button>
                </div>
            </div>
            <div class="card-body" id="operations-list" style="max-height: 400px; overflow-y: auto;">
                <!-- Dynamically populated by JavaScript -->
                <p class="text-muted" id="no-operations-msg">
                    No operations yet. Add drill holes, circles, or line cuts above.
                </p>
            </div>
        </div>
    </div>

    <!-- Right Column: Preview + Download -->
    <div class="col-lg-7">
        <!-- Preview Card -->
        <div class="card mb-3">
            <div class="card-header d-flex justify-content-between align-items-center">
                <strong>Preview</strong>
                <button class="btn btn-sm btn-outline-secondary" id="refresh-preview">
                    Refresh
                </button>
            </div>
            <div class="card-body">
                <div id="preview-container"
                     style="min-height: 400px; border: 1px solid #dee2e6; background: #f8f9fa;">
                    <!-- SVG preview inserted here -->
                    <div class="d-flex align-items-center justify-content-center h-100 text-muted">
                        Click "Refresh" to generate preview
                    </div>
                </div>
            </div>
        </div>

        <!-- Download Card -->
        <div class="card">
            <div class="card-header">
                <strong>Download G-Code</strong>
            </div>
            <div class="card-body">
                <div id="validation-errors" class="alert alert-danger" style="display: none;"></div>
                <div class="d-grid gap-2 d-md-flex">
                    <button class="btn btn-success flex-grow-1" id="download-drill"
                            {{ 'disabled' if project.project_type != 'drill' }}>
                        Download Drill G-Code
                    </button>
                    <button class="btn btn-success flex-grow-1" id="download-cut"
                            {{ 'disabled' if project.project_type != 'cut' }}>
                        Download Cut G-Code
                    </button>
                </div>
            </div>
        </div>
    </div>
</div>

<!-- Unsaved Changes Indicator -->
<div id="unsaved-indicator" class="position-fixed bottom-0 start-50 translate-middle-x mb-3"
     style="display: none; z-index: 1050;">
    <div class="alert alert-warning shadow d-flex align-items-center gap-3 mb-0">
        <span>You have unsaved changes</span>
        <button class="btn btn-sm btn-primary" id="save-btn">Save</button>
        <button class="btn btn-sm btn-outline-secondary" id="discard-btn">Discard</button>
    </div>
</div>

<!-- Add Drill Modal -->
<div class="modal fade" id="addDrillModal" tabindex="-1">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title">Add Drill Hole</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <div class="mb-3">
                    <label class="form-label">Pattern Type</label>
                    <select class="form-select" id="drill-type">
                        <option value="single">Single Point</option>
                        <option value="pattern_linear">Linear Pattern</option>
                        <option value="pattern_grid">Grid Pattern</option>
                    </select>
                </div>

                <!-- Single Point Fields -->
                <div id="drill-single-fields">
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">X (inches)</label>
                            <input type="number" step="0.001" class="form-control" id="drill-x">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Y (inches)</label>
                            <input type="number" step="0.001" class="form-control" id="drill-y">
                        </div>
                    </div>
                </div>

                <!-- Linear Pattern Fields -->
                <div id="drill-linear-fields" style="display: none;">
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">Start X</label>
                            <input type="number" step="0.001" class="form-control" id="drill-linear-start-x">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Start Y</label>
                            <input type="number" step="0.001" class="form-control" id="drill-linear-start-y">
                        </div>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Direction</label>
                        <select class="form-select" id="drill-linear-axis">
                            <option value="x">Along X axis (horizontal)</option>
                            <option value="y">Along Y axis (vertical)</option>
                        </select>
                    </div>
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">Spacing (inches)</label>
                            <input type="number" step="0.001" class="form-control" id="drill-linear-spacing">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Count</label>
                            <input type="number" min="1" class="form-control" id="drill-linear-count">
                        </div>
                    </div>
                </div>

                <!-- Grid Pattern Fields -->
                <div id="drill-grid-fields" style="display: none;">
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">Start X</label>
                            <input type="number" step="0.001" class="form-control" id="drill-grid-start-x">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Start Y</label>
                            <input type="number" step="0.001" class="form-control" id="drill-grid-start-y">
                        </div>
                    </div>
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">X Spacing</label>
                            <input type="number" step="0.001" class="form-control" id="drill-grid-x-spacing">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Y Spacing</label>
                            <input type="number" step="0.001" class="form-control" id="drill-grid-y-spacing">
                        </div>
                    </div>
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">Columns (X)</label>
                            <input type="number" min="1" class="form-control" id="drill-grid-x-count">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Rows (Y)</label>
                            <input type="number" min="1" class="form-control" id="drill-grid-y-count">
                        </div>
                    </div>
                </div>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                <button type="button" class="btn btn-primary" id="add-drill-confirm">Add Drill</button>
            </div>
        </div>
    </div>
</div>

<!-- Add Circle Modal -->
<div class="modal fade" id="addCircleModal" tabindex="-1">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title">Add Circular Cut</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <div class="mb-3">
                    <label class="form-label">Pattern Type</label>
                    <select class="form-select" id="circle-type">
                        <option value="single">Single Circle</option>
                        <option value="pattern_linear">Linear Pattern</option>
                    </select>
                </div>

                <!-- Single Circle Fields -->
                <div id="circle-single-fields">
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">Center X</label>
                            <input type="number" step="0.001" class="form-control" id="circle-center-x">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Center Y</label>
                            <input type="number" step="0.001" class="form-control" id="circle-center-y">
                        </div>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Diameter (inches)</label>
                        <input type="number" step="0.001" class="form-control" id="circle-diameter">
                    </div>
                </div>

                <!-- Linear Pattern Fields -->
                <div id="circle-linear-fields" style="display: none;">
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">Start Center X</label>
                            <input type="number" step="0.001" class="form-control" id="circle-linear-start-x">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Start Center Y</label>
                            <input type="number" step="0.001" class="form-control" id="circle-linear-start-y">
                        </div>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Diameter (inches)</label>
                        <input type="number" step="0.001" class="form-control" id="circle-linear-diameter">
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Direction</label>
                        <select class="form-select" id="circle-linear-axis">
                            <option value="x">Along X axis</option>
                            <option value="y">Along Y axis</option>
                        </select>
                    </div>
                    <div class="row">
                        <div class="col-6 mb-3">
                            <label class="form-label">Spacing</label>
                            <input type="number" step="0.001" class="form-control" id="circle-linear-spacing">
                        </div>
                        <div class="col-6 mb-3">
                            <label class="form-label">Count</label>
                            <input type="number" min="1" class="form-control" id="circle-linear-count">
                        </div>
                    </div>
                </div>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                <button type="button" class="btn btn-primary" id="add-circle-confirm">Add Circle</button>
            </div>
        </div>
    </div>
</div>

<!-- Add Line Modal -->
<div class="modal fade" id="addLineModal" tabindex="-1">
    <div class="modal-dialog modal-lg">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title">Add Line Cut</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body">
                <p class="text-muted">Define points for a cutting path. First point is the start.</p>

                <div id="line-points-container">
                    <!-- Points added dynamically -->
                </div>

                <button type="button" class="btn btn-outline-primary btn-sm" id="add-line-point">
                    + Add Point
                </button>

                <div class="form-check mt-3">
                    <input class="form-check-input" type="checkbox" id="line-closed" checked>
                    <label class="form-check-label" for="line-closed">
                        Closed path (return to start)
                    </label>
                </div>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                <button type="button" class="btn btn-primary" id="add-line-confirm">Add Line Cut</button>
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% block scripts %}
<script>
    // Pass project data to JavaScript
    const PROJECT_ID = "{{ project.id }}";
    const PROJECT_DATA = {{ project_json | safe }};
    const MATERIALS = {{ materials_json | safe }};
</script>
<script src="{{ url_for('static', filename='js/unsaved-changes.js') }}"></script>
<script src="{{ url_for('static', filename='js/project-editor.js') }}"></script>
{% endblock %}
```

---

## Settings Dashboard (templates/settings/index.html)

```html
{% extends "base.html" %}
{% block title %}Settings - G-Code Generator{% endblock %}

{% block content %}
<h1 class="mb-4">Settings</h1>

<div class="row">
    <div class="col-md-4 mb-3">
        <div class="card h-100">
            <div class="card-body">
                <h5 class="card-title">Materials</h5>
                <p class="card-text">
                    Configure materials and their G-code parameters for different tool sizes.
                </p>
                <a href="{{ url_for('settings.materials') }}" class="btn btn-primary">
                    Manage Materials
                </a>
            </div>
        </div>
    </div>

    <div class="col-md-4 mb-3">
        <div class="card h-100">
            <div class="card-body">
                <h5 class="card-title">Machine</h5>
                <p class="card-text">
                    Set machine limits, controller type, and feature support.
                </p>
                <a href="{{ url_for('settings.machine') }}" class="btn btn-primary">
                    Machine Settings
                </a>
            </div>
        </div>
    </div>

    <div class="col-md-4 mb-3">
        <div class="card h-100">
            <div class="card-body">
                <h5 class="card-title">General</h5>
                <p class="card-text">
                    Configure default heights, speeds, and other G-code parameters.
                </p>
                <a href="{{ url_for('settings.general') }}" class="btn btn-primary">
                    General Settings
                </a>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

## Materials Settings (templates/settings/materials.html)

```html
{% extends "base.html" %}
{% block title %}Materials - Settings{% endblock %}

{% block content %}
<div class="d-flex justify-content-between align-items-center mb-4">
    <h1>Materials</h1>
    <button class="btn btn-success" data-bs-toggle="modal" data-bs-target="#addMaterialModal">
        + Add Material
    </button>
</div>

<div class="table-responsive">
    <table class="table table-striped">
        <thead>
            <tr>
                <th>Name</th>
                <th>Base Material</th>
                <th>Form</th>
                <th>Dimensions</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {% for material in materials %}
            <tr>
                <td>{{ material.display_name }}</td>
                <td>{{ material.base_material | capitalize }}</td>
                <td>{{ material.form | capitalize }}</td>
                <td>
                    {% if material.form == 'sheet' %}
                        {{ material.thickness }}" thick
                    {% else %}
                        {{ material.outer_width }}" x {{ material.outer_height }}"
                        ({{ material.wall_thickness }}" wall)
                    {% endif %}
                </td>
                <td>
                    <a href="{{ url_for('settings.edit_material', material_id=material.id) }}"
                       class="btn btn-sm btn-outline-primary">Edit</a>
                    <form action="{{ url_for('settings.delete_material', material_id=material.id) }}"
                          method="POST" class="d-inline">
                        <button type="submit" class="btn btn-sm btn-outline-danger"
                                onclick="return confirm('Delete this material?')">Delete</button>
                    </form>
                </td>
            </tr>
            {% endfor %}
        </tbody>
    </table>
</div>

<!-- Add Material Modal -->
<div class="modal fade" id="addMaterialModal" tabindex="-1">
    <div class="modal-dialog modal-lg">
        <div class="modal-content">
            <form method="POST" action="{{ url_for('settings.create_material') }}">
                <div class="modal-header">
                    <h5 class="modal-title">Add Material</h5>
                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                </div>
                <div class="modal-body">
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Display Name</label>
                            <input type="text" class="form-control" name="display_name"
                                   required placeholder="e.g., Aluminum Sheet 1/8&quot;">
                        </div>
                        <div class="col-md-6 mb-3">
                            <label class="form-label">ID (no spaces)</label>
                            <input type="text" class="form-control" name="id"
                                   required pattern="[a-z0-9_]+" placeholder="e.g., aluminum_sheet_0125">
                        </div>
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Base Material</label>
                            <select class="form-select" name="base_material" required>
                                <option value="aluminum">Aluminum</option>
                                <option value="polycarbonate">Polycarbonate</option>
                                <option value="steel">Steel</option>
                                <option value="hdpe">HDPE</option>
                            </select>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Form</label>
                            <select class="form-select" name="form" id="material-form" required>
                                <option value="sheet">Sheet</option>
                                <option value="tube">Tube</option>
                            </select>
                        </div>
                    </div>

                    <!-- Sheet dimensions -->
                    <div id="sheet-dimensions">
                        <div class="mb-3">
                            <label class="form-label">Thickness (inches)</label>
                            <input type="number" step="0.001" class="form-control" name="thickness">
                        </div>
                    </div>

                    <!-- Tube dimensions -->
                    <div id="tube-dimensions" style="display: none;">
                        <div class="row">
                            <div class="col-md-4 mb-3">
                                <label class="form-label">Outer Width</label>
                                <input type="number" step="0.001" class="form-control" name="outer_width">
                            </div>
                            <div class="col-md-4 mb-3">
                                <label class="form-label">Outer Height</label>
                                <input type="number" step="0.001" class="form-control" name="outer_height">
                            </div>
                            <div class="col-md-4 mb-3">
                                <label class="form-label">Wall Thickness</label>
                                <input type="number" step="0.001" class="form-control" name="wall_thickness">
                            </div>
                        </div>
                    </div>

                    <hr>
                    <h6>G-Code Standards (per tool size)</h6>
                    <p class="text-muted small">Configure default speeds and feeds for each tool size.</p>
                    <!-- G-code standards form would go here - expandable sections for each tool size -->
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                    <button type="submit" class="btn btn-primary">Add Material</button>
                </div>
            </form>
        </div>
    </div>
</div>

<script>
document.getElementById('material-form').addEventListener('change', function() {
    document.getElementById('sheet-dimensions').style.display =
        this.value === 'sheet' ? 'block' : 'none';
    document.getElementById('tube-dimensions').style.display =
        this.value === 'tube' ? 'block' : 'none';
});
</script>
{% endblock %}
```

---

## Machine Settings (templates/settings/machine.html)

```html
{% extends "base.html" %}
{% block title %}Machine Settings{% endblock %}

{% block content %}
<h1 class="mb-4">Machine Settings</h1>

<div class="row justify-content-center">
    <div class="col-md-8">
        <form method="POST" action="{{ url_for('settings.save_machine') }}">
            <div class="card">
                <div class="card-body">
                    <div class="mb-3">
                        <label class="form-label">Machine Name</label>
                        <input type="text" class="form-control" name="name"
                               value="{{ machine.name }}">
                    </div>

                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Max X Travel (inches)</label>
                            <input type="number" step="0.1" class="form-control" name="max_x"
                                   value="{{ machine.max_x }}">
                        </div>
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Max Y Travel (inches)</label>
                            <input type="number" step="0.1" class="form-control" name="max_y"
                                   value="{{ machine.max_y }}">
                        </div>
                    </div>

                    <div class="mb-3">
                        <label class="form-label">Controller Type</label>
                        <select class="form-select" name="controller_type">
                            <option value="mach3" {{ 'selected' if machine.controller_type == 'mach3' }}>
                                Mach3
                            </option>
                            <option value="mach4" {{ 'selected' if machine.controller_type == 'mach4' }}>
                                Mach4
                            </option>
                            <option value="grbl" {{ 'selected' if machine.controller_type == 'grbl' }}>
                                GRBL
                            </option>
                            <option value="linuxcnc" {{ 'selected' if machine.controller_type == 'linuxcnc' }}>
                                LinuxCNC
                            </option>
                        </select>
                    </div>

                    <div class="mb-3">
                        <div class="form-check">
                            <input class="form-check-input" type="checkbox" name="supports_loops"
                                   id="supports-loops" {{ 'checked' if machine.supports_loops }}>
                            <label class="form-check-label" for="supports-loops">
                                Controller supports loops (WHILE/DO)
                            </label>
                        </div>
                        <div class="form-text">
                            Enable to use loops for multi-pass operations, reducing file size.
                        </div>
                    </div>

                    <div class="mb-3">
                        <div class="form-check">
                            <input class="form-check-input" type="checkbox" name="supports_canned_cycles"
                                   id="supports-canned" {{ 'checked' if machine.supports_canned_cycles }}>
                            <label class="form-check-label" for="supports-canned">
                                Controller supports canned cycles (G83)
                            </label>
                        </div>
                        <div class="form-text">
                            Enable to use G83 peck drilling cycle for drill operations.
                        </div>
                    </div>
                </div>
                <div class="card-footer">
                    <button type="submit" class="btn btn-primary">Save Settings</button>
                    <a href="{{ url_for('settings.index') }}" class="btn btn-outline-secondary">Cancel</a>
                </div>
            </div>
        </form>
    </div>
</div>
{% endblock %}
```

---

## General Settings (templates/settings/general.html)

```html
{% extends "base.html" %}
{% block title %}General Settings{% endblock %}

{% block content %}
<h1 class="mb-4">General G-Code Settings</h1>

<div class="row justify-content-center">
    <div class="col-md-8">
        <form method="POST" action="{{ url_for('settings.save_general') }}">
            <div class="card">
                <div class="card-body">
                    <div class="row">
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Safety Height (inches)</label>
                            <input type="number" step="0.01" class="form-control" name="safety_height"
                                   value="{{ general.safety_height }}">
                            <div class="form-text">Height for rapid moves between operations</div>
                        </div>
                        <div class="col-md-6 mb-3">
                            <label class="form-label">Travel Height (inches)</label>
                            <input type="number" step="0.01" class="form-control" name="travel_height"
                                   value="{{ general.travel_height }}">
                            <div class="form-text">Height above material for positioning</div>
                        </div>
                    </div>

                    <div class="mb-3">
                        <label class="form-label">Spindle Warmup Time (seconds)</label>
                        <input type="number" min="0" class="form-control" name="spindle_warmup_seconds"
                               value="{{ general.spindle_warmup_seconds }}">
                        <div class="form-text">Dwell time after spindle start (G4 command)</div>
                    </div>
                </div>
                <div class="card-footer">
                    <button type="submit" class="btn btn-primary">Save Settings</button>
                    <a href="{{ url_for('settings.index') }}" class="btn btn-outline-secondary">Cancel</a>
                </div>
            </div>
        </form>
    </div>
</div>
{% endblock %}
```

---

## JavaScript: Project Editor (static/js/project-editor.js)

```javascript
/**
 * Project Editor - Main controller for the project editing interface
 */
class ProjectEditor {
    constructor(projectId, initialData, materials) {
        this.projectId = projectId;
        this.data = JSON.parse(JSON.stringify(initialData)); // Deep copy
        this.materials = materials;
        this.unsavedTracker = new UnsavedChangesTracker(initialData);
        this.operationIdCounter = 0;

        this.init();
    }

    init() {
        this.bindEvents();
        this.renderOperations();
        this.updateUIForProjectType();
        this.updateUIForMaterial();
    }

    bindEvents() {
        // Save button
        document.getElementById('save-btn').addEventListener('click', () => this.save());

        // Discard button
        document.getElementById('discard-btn').addEventListener('click', () => this.discard());

        // Download buttons
        document.getElementById('download-drill').addEventListener('click', () => this.download('drill'));
        document.getElementById('download-cut').addEventListener('click', () => this.download('cut'));

        // Refresh preview
        document.getElementById('refresh-preview').addEventListener('click', () => this.refreshPreview());

        // Form field changes
        document.querySelectorAll('[data-field]').forEach(el => {
            el.addEventListener('change', (e) => this.onFieldChange(e));
        });

        // Project type change
        document.getElementById('project-type').addEventListener('change', () => this.updateUIForProjectType());

        // Material change
        document.getElementById('material').addEventListener('change', () => this.updateUIForMaterial());

        // Modal pattern type changes
        document.getElementById('drill-type').addEventListener('change', (e) => this.showDrillFields(e.target.value));
        document.getElementById('circle-type').addEventListener('change', (e) => this.showCircleFields(e.target.value));

        // Add operation confirms
        document.getElementById('add-drill-confirm').addEventListener('click', () => this.addDrill());
        document.getElementById('add-circle-confirm').addEventListener('click', () => this.addCircle());
        document.getElementById('add-line-point').addEventListener('click', () => this.addLinePoint());
        document.getElementById('add-line-confirm').addEventListener('click', () => this.addLine());

        // Initialize line points
        this.initLineModal();
    }

    onFieldChange(e) {
        const field = e.target.dataset.field;
        const value = e.target.type === 'checkbox' ? e.target.checked : e.target.value;

        if (field === 'tool_size') {
            const projectType = document.getElementById('project-type').value;
            if (projectType === 'drill') {
                this.data.drill_bit_size = parseFloat(value);
            } else {
                this.data.end_mill_size = parseFloat(value);
            }
        } else {
            this.data[field] = value;
        }

        this.unsavedTracker.checkChanges(this.data);
    }

    updateUIForProjectType() {
        const type = document.getElementById('project-type').value;

        // Update download buttons
        document.getElementById('download-drill').disabled = (type !== 'drill');
        document.getElementById('download-cut').disabled = (type !== 'cut');

        // Update add operation buttons
        document.getElementById('btn-add-drill').style.display = (type === 'drill') ? 'inline-block' : 'none';
        document.getElementById('btn-add-circle').style.display = (type === 'cut') ? 'inline-block' : 'none';
        document.getElementById('btn-add-line').style.display = (type === 'cut') ? 'inline-block' : 'none';

        // Filter tools by type
        const toolSelect = document.getElementById('tool-size');
        Array.from(toolSelect.options).forEach(opt => {
            const toolType = opt.dataset.type;
            const showTool = (type === 'drill' && toolType === 'drill') ||
                           (type === 'cut' && toolType === 'end_mill');
            opt.style.display = showTool ? '' : 'none';
        });
    }

    updateUIForMaterial() {
        const materialId = document.getElementById('material').value;
        const material = this.materials[materialId];

        // Show/hide tube void section
        const tubeSection = document.getElementById('tube-void-section');
        if (material && material.form === 'tube') {
            tubeSection.style.display = 'block';
        } else {
            tubeSection.style.display = 'none';
        }
    }

    showDrillFields(type) {
        document.getElementById('drill-single-fields').style.display = (type === 'single') ? 'block' : 'none';
        document.getElementById('drill-linear-fields').style.display = (type === 'pattern_linear') ? 'block' : 'none';
        document.getElementById('drill-grid-fields').style.display = (type === 'pattern_grid') ? 'block' : 'none';
    }

    showCircleFields(type) {
        document.getElementById('circle-single-fields').style.display = (type === 'single') ? 'block' : 'none';
        document.getElementById('circle-linear-fields').style.display = (type === 'pattern_linear') ? 'block' : 'none';
    }

    generateId() {
        return 'op_' + (++this.operationIdCounter);
    }

    addDrill() {
        const type = document.getElementById('drill-type').value;
        let operation = { id: this.generateId(), type };

        if (type === 'single') {
            operation.x = parseFloat(document.getElementById('drill-x').value);
            operation.y = parseFloat(document.getElementById('drill-y').value);
        } else if (type === 'pattern_linear') {
            operation.start_x = parseFloat(document.getElementById('drill-linear-start-x').value);
            operation.start_y = parseFloat(document.getElementById('drill-linear-start-y').value);
            operation.axis = document.getElementById('drill-linear-axis').value;
            operation.spacing = parseFloat(document.getElementById('drill-linear-spacing').value);
            operation.count = parseInt(document.getElementById('drill-linear-count').value);
        } else if (type === 'pattern_grid') {
            operation.start_x = parseFloat(document.getElementById('drill-grid-start-x').value);
            operation.start_y = parseFloat(document.getElementById('drill-grid-start-y').value);
            operation.x_spacing = parseFloat(document.getElementById('drill-grid-x-spacing').value);
            operation.y_spacing = parseFloat(document.getElementById('drill-grid-y-spacing').value);
            operation.x_count = parseInt(document.getElementById('drill-grid-x-count').value);
            operation.y_count = parseInt(document.getElementById('drill-grid-y-count').value);
        }

        if (!this.data.operations.drill_holes) {
            this.data.operations.drill_holes = [];
        }
        this.data.operations.drill_holes.push(operation);

        this.renderOperations();
        this.unsavedTracker.checkChanges(this.data);
        bootstrap.Modal.getInstance(document.getElementById('addDrillModal')).hide();
    }

    addCircle() {
        const type = document.getElementById('circle-type').value;
        let operation = { id: this.generateId(), type };

        if (type === 'single') {
            operation.center_x = parseFloat(document.getElementById('circle-center-x').value);
            operation.center_y = parseFloat(document.getElementById('circle-center-y').value);
            operation.diameter = parseFloat(document.getElementById('circle-diameter').value);
        } else if (type === 'pattern_linear') {
            operation.start_center_x = parseFloat(document.getElementById('circle-linear-start-x').value);
            operation.start_center_y = parseFloat(document.getElementById('circle-linear-start-y').value);
            operation.diameter = parseFloat(document.getElementById('circle-linear-diameter').value);
            operation.axis = document.getElementById('circle-linear-axis').value;
            operation.spacing = parseFloat(document.getElementById('circle-linear-spacing').value);
            operation.count = parseInt(document.getElementById('circle-linear-count').value);
        }

        if (!this.data.operations.circular_cuts) {
            this.data.operations.circular_cuts = [];
        }
        this.data.operations.circular_cuts.push(operation);

        this.renderOperations();
        this.unsavedTracker.checkChanges(this.data);
        bootstrap.Modal.getInstance(document.getElementById('addCircleModal')).hide();
    }

    initLineModal() {
        this.linePoints = [];
        this.addLinePoint(); // Add first point
        this.addLinePoint(); // Add second point
    }

    addLinePoint() {
        const container = document.getElementById('line-points-container');
        const idx = this.linePoints.length;
        const isFirst = idx === 0;

        const pointHtml = `
            <div class="row mb-2 line-point" data-index="${idx}">
                <div class="col-3">
                    <input type="number" step="0.001" class="form-control form-control-sm"
                           placeholder="X" data-point-field="x">
                </div>
                <div class="col-3">
                    <input type="number" step="0.001" class="form-control form-control-sm"
                           placeholder="Y" data-point-field="y">
                </div>
                <div class="col-4">
                    <select class="form-select form-select-sm" data-point-field="line_type"
                            ${isFirst ? 'disabled' : ''}>
                        ${isFirst ? '<option value="start">Start</option>' :
                          '<option value="straight">Straight</option><option value="arc">Arc</option>'}
                    </select>
                </div>
                <div class="col-2">
                    ${!isFirst ? '<button type="button" class="btn btn-sm btn-outline-danger remove-point">&times;</button>' : ''}
                </div>
            </div>
        `;

        container.insertAdjacentHTML('beforeend', pointHtml);
        this.linePoints.push({ x: 0, y: 0, line_type: isFirst ? 'start' : 'straight' });

        // Bind remove button
        const removeBtn = container.querySelector('.line-point:last-child .remove-point');
        if (removeBtn) {
            removeBtn.addEventListener('click', (e) => {
                const row = e.target.closest('.line-point');
                const index = parseInt(row.dataset.index);
                row.remove();
                this.linePoints.splice(index, 1);
            });
        }
    }

    addLine() {
        // Collect points from form
        const points = [];
        document.querySelectorAll('.line-point').forEach(row => {
            const x = parseFloat(row.querySelector('[data-point-field="x"]').value);
            const y = parseFloat(row.querySelector('[data-point-field="y"]').value);
            const lineType = row.querySelector('[data-point-field="line_type"]').value;
            points.push({ x, y, line_type: lineType });
        });

        const operation = {
            id: this.generateId(),
            points: points,
            closed: document.getElementById('line-closed').checked
        };

        if (!this.data.operations.line_cuts) {
            this.data.operations.line_cuts = [];
        }
        this.data.operations.line_cuts.push(operation);

        this.renderOperations();
        this.unsavedTracker.checkChanges(this.data);
        bootstrap.Modal.getInstance(document.getElementById('addLineModal')).hide();

        // Reset line modal
        document.getElementById('line-points-container').innerHTML = '';
        this.linePoints = [];
        this.initLineModal();
    }

    removeOperation(type, index) {
        if (type === 'drill') {
            this.data.operations.drill_holes.splice(index, 1);
        } else if (type === 'circle') {
            this.data.operations.circular_cuts.splice(index, 1);
        } else if (type === 'line') {
            this.data.operations.line_cuts.splice(index, 1);
        }
        this.renderOperations();
        this.unsavedTracker.checkChanges(this.data);
    }

    renderOperations() {
        const container = document.getElementById('operations-list');
        const noOpsMsg = document.getElementById('no-operations-msg');

        const ops = this.data.operations;
        const hasOps = (ops.drill_holes?.length || ops.circular_cuts?.length || ops.line_cuts?.length);

        if (!hasOps) {
            noOpsMsg.style.display = 'block';
            container.innerHTML = '';
            container.appendChild(noOpsMsg);
            return;
        }

        noOpsMsg.style.display = 'none';
        container.innerHTML = '';

        // Render drill holes
        (ops.drill_holes || []).forEach((op, idx) => {
            container.innerHTML += this.renderOperationCard('drill', op, idx);
        });

        // Render circles
        (ops.circular_cuts || []).forEach((op, idx) => {
            container.innerHTML += this.renderOperationCard('circle', op, idx);
        });

        // Render lines
        (ops.line_cuts || []).forEach((op, idx) => {
            container.innerHTML += this.renderOperationCard('line', op, idx);
        });

        // Bind remove buttons
        container.querySelectorAll('.remove-op').forEach(btn => {
            btn.addEventListener('click', (e) => {
                const type = e.target.dataset.type;
                const idx = parseInt(e.target.dataset.index);
                this.removeOperation(type, idx);
            });
        });
    }

    renderOperationCard(type, op, idx) {
        let desc = '';
        let badge = '';

        if (type === 'drill') {
            badge = 'bg-info';
            if (op.type === 'single') {
                desc = `Point at (${op.x}, ${op.y})`;
            } else if (op.type === 'pattern_linear') {
                desc = `${op.count} holes along ${op.axis.toUpperCase()}, ${op.spacing}" spacing`;
            } else if (op.type === 'pattern_grid') {
                desc = `${op.x_count}x${op.y_count} grid`;
            }
        } else if (type === 'circle') {
            badge = 'bg-warning text-dark';
            if (op.type === 'single') {
                desc = `${op.diameter}" dia at (${op.center_x}, ${op.center_y})`;
            } else if (op.type === 'pattern_linear') {
                desc = `${op.count}x ${op.diameter}" circles along ${op.axis.toUpperCase()}`;
            }
        } else if (type === 'line') {
            badge = 'bg-success';
            desc = `${op.points.length} points${op.closed ? ' (closed)' : ''}`;
        }

        return `
            <div class="d-flex justify-content-between align-items-center mb-2 p-2 bg-light rounded">
                <div>
                    <span class="badge ${badge} me-2">${type}</span>
                    <span>${desc}</span>
                </div>
                <button class="btn btn-sm btn-outline-danger remove-op"
                        data-type="${type}" data-index="${idx}">&times;</button>
            </div>
        `;
    }

    async save() {
        // Update data from form fields
        this.data.name = document.getElementById('project-name').value;
        this.data.project_type = document.getElementById('project-type').value;
        this.data.material_id = document.getElementById('material').value;
        this.data.tube_void_skip = document.getElementById('tube-void-skip').checked;

        const response = await fetch(`/api/projects/${this.projectId}/save`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(this.data)
        });

        if (response.ok) {
            this.unsavedTracker.markSaved(this.data);
            // Show success message briefly
            const indicator = document.getElementById('unsaved-indicator');
            indicator.querySelector('.alert').className = 'alert alert-success shadow d-flex align-items-center gap-3 mb-0';
            indicator.querySelector('span').textContent = 'Saved!';
            setTimeout(() => {
                indicator.style.display = 'none';
                indicator.querySelector('.alert').className = 'alert alert-warning shadow d-flex align-items-center gap-3 mb-0';
                indicator.querySelector('span').textContent = 'You have unsaved changes';
            }, 1500);
        }
    }

    discard() {
        if (confirm('Discard all changes?')) {
            window.location.reload();
        }
    }

    download(type) {
        window.location.href = `/api/projects/${this.projectId}/download/${type}`;
    }

    async refreshPreview() {
        const container = document.getElementById('preview-container');
        container.innerHTML = '<div class="d-flex align-items-center justify-content-center h-100"><div class="spinner-border text-primary"></div></div>';

        try {
            const response = await fetch(`/api/projects/${this.projectId}/preview`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(this.data)
            });
            const result = await response.json();
            container.innerHTML = result.svg;
        } catch (error) {
            container.innerHTML = '<div class="alert alert-danger m-3">Failed to generate preview</div>';
        }
    }
}

// Initialize editor when DOM is ready
document.addEventListener('DOMContentLoaded', () => {
    window.editor = new ProjectEditor(PROJECT_ID, PROJECT_DATA, MATERIALS);
});
```

---

## JavaScript: Unsaved Changes (static/js/unsaved-changes.js)

```javascript
/**
 * Tracks unsaved changes and warns user before leaving page
 */
class UnsavedChangesTracker {
    constructor(initialData) {
        this.originalData = JSON.stringify(initialData);
        this.isDirty = false;

        // Warn on page unload
        window.addEventListener('beforeunload', (e) => {
            if (this.isDirty) {
                e.preventDefault();
                e.returnValue = 'You have unsaved changes. Are you sure you want to leave?';
                return e.returnValue;
            }
        });
    }

    checkChanges(currentData) {
        const currentJson = JSON.stringify(currentData);
        this.isDirty = currentJson !== this.originalData;

        const indicator = document.getElementById('unsaved-indicator');
        if (indicator) {
            indicator.style.display = this.isDirty ? 'block' : 'none';
        }

        return this.isDirty;
    }

    markSaved(currentData) {
        this.originalData = JSON.stringify(currentData);
        this.isDirty = false;

        const indicator = document.getElementById('unsaved-indicator');
        if (indicator) {
            indicator.style.display = 'none';
        }
    }
}
```

---

## Custom Styles (static/css/styles.css)

```css
/* Custom styles for G-Code Generator */

/* Cards */
.card {
    box-shadow: 0 0.125rem 0.25rem rgba(0, 0, 0, 0.075);
}

.card-header {
    background-color: #f8f9fa;
    font-weight: 500;
}

/* Preview container */
#preview-container {
    background: #f8f9fa;
    border-radius: 0.25rem;
}

#preview-container svg {
    max-width: 100%;
    height: auto;
}

/* Operations list */
#operations-list .bg-light {
    transition: background-color 0.15s;
}

#operations-list .bg-light:hover {
    background-color: #e9ecef !important;
}

/* Unsaved indicator */
#unsaved-indicator {
    z-index: 1050;
}

/* Modal improvements */
.modal-body .row {
    margin-bottom: 0.5rem;
}

/* Form adjustments */
.form-control:focus,
.form-select:focus {
    border-color: #0d6efd;
    box-shadow: 0 0 0 0.2rem rgba(13, 110, 253, 0.15);
}

/* Button group adjustments */
.btn-group-sm > .btn {
    padding: 0.25rem 0.5rem;
    font-size: 0.75rem;
}

/* Table improvements */
.table th {
    font-weight: 600;
    background-color: #f8f9fa;
}

/* Navbar brand */
.navbar-brand {
    font-weight: 600;
}

/* Footer */
footer {
    border-top: 1px solid #dee2e6;
    padding-top: 1rem;
}

/* Responsive adjustments */
@media (max-width: 991.98px) {
    #preview-container {
        min-height: 300px !important;
    }

    .col-lg-5,
    .col-lg-7 {
        margin-bottom: 1rem;
    }
}

/* Line point rows */
.line-point {
    align-items: center;
}

.line-point .form-control-sm,
.line-point .form-select-sm {
    height: calc(1.5em + 0.5rem + 2px);
}

/* Badge colors for operation types */
.badge.bg-info {
    background-color: #0dcaf0 !important;
}

.badge.bg-warning {
    background-color: #ffc107 !important;
}

.badge.bg-success {
    background-color: #198754 !important;
}
```

---

## Implementation Order

1. **Phase 1**: Base templates
   - Create `templates/base.html`
   - Create `static/css/styles.css`

2. **Phase 2**: Dashboard and project list
   - Create `templates/index.html`
   - Create `templates/project/new.html`

3. **Phase 3**: Project editor
   - Create `templates/project/edit.html`
   - Create `static/js/unsaved-changes.js`
   - Create `static/js/project-editor.js`

4. **Phase 4**: Settings pages
   - Create `templates/settings/index.html`
   - Create `templates/settings/materials.html`
   - Create `templates/settings/machine.html`
   - Create `templates/settings/general.html`

5. **Phase 5**: Polish
   - Test all modals and forms
   - Test unsaved changes warning
   - Test preview refresh
   - Test download functionality

---

## Key Files to Create

| File | Purpose |
|------|---------|
| `templates/base.html` | Base template with nav and Bootstrap |
| `templates/index.html` | Projects dashboard |
| `templates/project/new.html` | New project form |
| `templates/project/edit.html` | Main project editor workspace |
| `templates/settings/index.html` | Settings dashboard |
| `templates/settings/materials.html` | Materials management |
| `templates/settings/machine.html` | Machine settings |
| `templates/settings/general.html` | General G-code settings |
| `static/css/styles.css` | Custom styles |
| `static/js/project-editor.js` | Project editor controller |
| `static/js/unsaved-changes.js` | Unsaved changes tracker |
| `templates/settings/material_edit.html` | Material G-code standards editor |
| `templates/settings/tools.html` | Tools management page |

---

## Template Data Passing

This section explains how data flows from routes to templates and then to JavaScript.

### How Routes Pass Data to Templates

Routes use `render_template()` to pass data. Here's the pattern for each page:

**Index Page (projects list):**
```python
# In web/routes/main.py
@main_bp.route('/')
def index():
    projects = ProjectService.get_all()  # Returns list of Project objects
    return render_template('index.html', projects=projects)
```

Template receives `projects` as a list of SQLAlchemy model objects:
```html
{% for project in projects %}
    <h5>{{ project.name }}</h5>
    <p>{{ project.material.display_name }}</p>  {# Access relationship #}
{% endfor %}
```

**Project Editor Page:**
```python
# In web/routes/projects.py
@projects_bp.route('/<project_id>')
def edit(project_id):
    project = ProjectService.get(project_id)
    materials = SettingsService.get_all_materials()
    materials_dict = SettingsService.get_materials_dict()
    tools = SettingsService.get_all_tools()
    project_dict = ProjectService.get_as_dict(project_id)

    return render_template(
        'project/edit.html',
        project=project,           # SQLAlchemy object for Jinja
        materials=materials,       # List for select dropdowns
        tools=tools,               # List for select dropdowns
        project_json=json.dumps(project_dict),    # JSON string for JS
        materials_json=json.dumps(materials_dict) # JSON string for JS
    )
```

### Passing Data to JavaScript

Use `json.dumps()` in Python and the `| safe` filter in Jinja to embed JSON in the page:

```html
{% block scripts %}
<script>
    // These variables are available to JavaScript
    const PROJECT_ID = "{{ project.id }}";
    const PROJECT_DATA = {{ project_json | safe }};
    const MATERIALS = {{ materials_json | safe }};
</script>
<script src="{{ url_for('static', filename='js/project-editor.js') }}"></script>
{% endblock %}
```

The `| safe` filter prevents Jinja from escaping the JSON, which would break JavaScript parsing.

### Data Dictionary Structures

**project_json structure:**
```javascript
{
    "id": "uuid-string",
    "name": "My Project",
    "project_type": "drill",          // or "cut"
    "material_id": "aluminum_sheet_0125",
    "drill_bit_size": 0.125,
    "end_mill_size": null,
    "tube_void_skip": false,
    "operations": {
        "drill_holes": [...],
        "circular_cuts": [...],
        "line_cuts": [...]
    },
    "created_at": "2024-01-15T10:30:00",
    "modified_at": "2024-01-15T14:22:00"
}
```

**materials_json structure:**
```javascript
{
    "aluminum_sheet_0125": {
        "id": "aluminum_sheet_0125",
        "display_name": "Aluminum Sheet 1/8\"",
        "base_material": "aluminum",
        "form": "sheet",
        "thickness": 0.125,
        "outer_width": null,
        "outer_height": null,
        "wall_thickness": null,
        "gcode_standards": {
            "drill": {
                "0.125": {"spindle_speed": 1000, "feed_rate": 2.0, ...}
            },
            "cut": {
                "0.125": {"spindle_speed": 10000, "feed_rate": 10.0, ...}
            }
        }
    },
    "aluminum_tube_2x1_0125": {
        "form": "tube",
        "outer_width": 2.0,
        "outer_height": 1.0,
        "wall_thickness": 0.125,
        ...
    }
}
```

---

## Form Validation

### Server-Side Validation

Always validate on the server. Here's a pattern using Flask's `request.form`:

```python
# In web/routes/projects.py
@projects_bp.route('/create', methods=['POST'])
def create():
    # Extract form data
    name = request.form.get('name', '').strip()
    project_type = request.form.get('project_type')
    material_id = request.form.get('material_id')

    # Validate required fields
    errors = []
    if not name:
        errors.append('Project name is required')
    if not project_type or project_type not in ('drill', 'cut'):
        errors.append('Valid project type is required')
    if not material_id:
        errors.append('Material selection is required')

    # Check name uniqueness (optional)
    existing = Project.query.filter_by(name=name).first()
    if existing:
        errors.append('A project with this name already exists')

    # If errors, flash and redirect back
    if errors:
        for error in errors:
            flash(error, 'danger')
        return redirect(url_for('projects.new'))

    # Create project
    project = ProjectService.create({
        'name': name,
        'project_type': project_type,
        'material_id': material_id
    })
    flash('Project created successfully', 'success')
    return redirect(url_for('projects.edit', project_id=project.id))
```

### Client-Side Validation

Add HTML5 validation attributes and JavaScript validation:

```html
<form method="POST" action="{{ url_for('projects.create') }}" id="new-project-form">
    <div class="mb-3">
        <label for="name" class="form-label">Project Name</label>
        <input type="text" class="form-control" id="name" name="name"
               required minlength="1" maxlength="200"
               pattern="[A-Za-z0-9\s\-_]+"
               placeholder="e.g., Drivetrain Gusset">
        <div class="invalid-feedback">
            Please enter a valid project name (letters, numbers, spaces, hyphens, underscores)
        </div>
    </div>

    <div class="mb-3">
        <label for="project_type" class="form-label">Project Type</label>
        <select class="form-select" id="project_type" name="project_type" required>
            <option value="">Select type...</option>
            <option value="drill">Drill</option>
            <option value="cut">Cut</option>
        </select>
        <div class="invalid-feedback">
            Please select a project type
        </div>
    </div>

    <button type="submit" class="btn btn-primary">Create Project</button>
</form>

<script>
// Bootstrap validation
(function() {
    'use strict';
    const form = document.getElementById('new-project-form');
    form.addEventListener('submit', function(event) {
        if (!form.checkValidity()) {
            event.preventDefault();
            event.stopPropagation();
        }
        form.classList.add('was-validated');
    }, false);
})();
</script>
```

### API Validation (JSON endpoints)

For AJAX endpoints, return JSON error responses:

```python
# In web/routes/api.py
@api_bp.route('/projects/<project_id>/save', methods=['POST'])
def save_project(project_id):
    data = request.get_json()

    # Validate
    errors = []
    if not data.get('name', '').strip():
        errors.append('Project name is required')

    # Validate operations structure
    operations = data.get('operations', {})
    for drill in operations.get('drill_holes', []):
        if drill.get('type') == 'single':
            if 'x' not in drill or 'y' not in drill:
                errors.append(f"Drill operation {drill.get('id')} missing coordinates")

    if errors:
        return jsonify({'status': 'error', 'errors': errors}), 400

    # Save
    project = ProjectService.save(project_id, data)
    return jsonify({'status': 'saved', 'modified_at': project.modified_at.isoformat()})
```

JavaScript handling:
```javascript
async save() {
    const response = await fetch(`/api/projects/${this.projectId}/save`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(this.data)
    });

    const result = await response.json();

    if (!response.ok || result.status === 'error') {
        // Show errors
        const errors = result.errors || ['Unknown error occurred'];
        alert('Validation errors:\n' + errors.join('\n'));
        return false;
    }

    this.unsavedTracker.markSaved(this.data);
    return true;
}
```

---

## Additional Templates

### Material Edit Page (templates/settings/material_edit.html)

This template allows editing G-code standards for each tool size:

```html
{% extends "base.html" %}
{% block title %}Edit Material - {{ material.display_name }}{% endblock %}

{% block content %}
<h1 class="mb-4">Edit Material: {{ material.display_name }}</h1>

<form method="POST" action="{{ url_for('settings.update_material', material_id=material.id) }}">
    <div class="row">
        <!-- Basic Info Column -->
        <div class="col-md-4">
            <div class="card mb-3">
                <div class="card-header">Basic Information</div>
                <div class="card-body">
                    <div class="mb-3">
                        <label class="form-label">Display Name</label>
                        <input type="text" class="form-control" name="display_name"
                               value="{{ material.display_name }}" required>
                    </div>

                    <div class="mb-3">
                        <label class="form-label">Base Material</label>
                        <select class="form-select" name="base_material">
                            <option value="aluminum" {{ 'selected' if material.base_material == 'aluminum' }}>Aluminum</option>
                            <option value="polycarbonate" {{ 'selected' if material.base_material == 'polycarbonate' }}>Polycarbonate</option>
                            <option value="steel" {{ 'selected' if material.base_material == 'steel' }}>Steel</option>
                            <option value="hdpe" {{ 'selected' if material.base_material == 'hdpe' }}>HDPE</option>
                        </select>
                    </div>

                    <div class="mb-3">
                        <label class="form-label">Form</label>
                        <select class="form-select" name="form" id="material-form">
                            <option value="sheet" {{ 'selected' if material.form == 'sheet' }}>Sheet</option>
                            <option value="tube" {{ 'selected' if material.form == 'tube' }}>Tube</option>
                        </select>
                    </div>

                    <!-- Sheet dimensions -->
                    <div id="sheet-dims" {% if material.form != 'sheet' %}style="display:none"{% endif %}>
                        <div class="mb-3">
                            <label class="form-label">Thickness (inches)</label>
                            <input type="number" step="0.001" class="form-control" name="thickness"
                                   value="{{ material.thickness or '' }}">
                        </div>
                    </div>

                    <!-- Tube dimensions -->
                    <div id="tube-dims" {% if material.form != 'tube' %}style="display:none"{% endif %}>
                        <div class="mb-3">
                            <label class="form-label">Outer Width (inches)</label>
                            <input type="number" step="0.001" class="form-control" name="outer_width"
                                   value="{{ material.outer_width or '' }}">
                        </div>
                        <div class="mb-3">
                            <label class="form-label">Outer Height (inches)</label>
                            <input type="number" step="0.001" class="form-control" name="outer_height"
                                   value="{{ material.outer_height or '' }}">
                        </div>
                        <div class="mb-3">
                            <label class="form-label">Wall Thickness (inches)</label>
                            <input type="number" step="0.001" class="form-control" name="wall_thickness"
                                   value="{{ material.wall_thickness or '' }}">
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- G-Code Standards Column -->
        <div class="col-md-8">
            <div class="card mb-3">
                <div class="card-header">G-Code Standards by Tool Size</div>
                <div class="card-body">
                    <p class="text-muted">Configure speeds and feeds for each tool size.</p>

                    <!-- Tabs for Drill vs Cut -->
                    <ul class="nav nav-tabs" role="tablist">
                        <li class="nav-item">
                            <button class="nav-link active" data-bs-toggle="tab" data-bs-target="#drill-tab">
                                Drill Operations
                            </button>
                        </li>
                        <li class="nav-item">
                            <button class="nav-link" data-bs-toggle="tab" data-bs-target="#cut-tab">
                                Cut Operations
                            </button>
                        </li>
                    </ul>

                    <div class="tab-content mt-3">
                        <!-- Drill Tab -->
                        <div class="tab-pane fade show active" id="drill-tab">
                            {% for tool in tools if tool.tool_type == 'drill' %}
                            {% set tool_key = tool.size | string %}
                            {% set params = material.gcode_standards.get('drill', {}).get(tool_key, {}) %}
                            <div class="card mb-2">
                                <div class="card-header py-2">
                                    <strong>{{ tool.description }}</strong> ({{ tool.size }}")
                                </div>
                                <div class="card-body py-2">
                                    <div class="row g-2">
                                        <div class="col-md-3">
                                            <label class="form-label small">Spindle Speed (RPM)</label>
                                            <input type="number" class="form-control form-control-sm"
                                                   name="drill_{{ tool.size }}_spindle_speed"
                                                   value="{{ params.get('spindle_speed', '') }}">
                                        </div>
                                        <div class="col-md-3">
                                            <label class="form-label small">Feed Rate (in/min)</label>
                                            <input type="number" step="0.1" class="form-control form-control-sm"
                                                   name="drill_{{ tool.size }}_feed_rate"
                                                   value="{{ params.get('feed_rate', '') }}">
                                        </div>
                                        <div class="col-md-3">
                                            <label class="form-label small">Plunge Rate (in/min)</label>
                                            <input type="number" step="0.1" class="form-control form-control-sm"
                                                   name="drill_{{ tool.size }}_plunge_rate"
                                                   value="{{ params.get('plunge_rate', '') }}">
                                        </div>
                                        <div class="col-md-3">
                                            <label class="form-label small">Pecking Depth (in)</label>
                                            <input type="number" step="0.001" class="form-control form-control-sm"
                                                   name="drill_{{ tool.size }}_pecking_depth"
                                                   value="{{ params.get('pecking_depth', '') }}">
                                        </div>
                                    </div>
                                </div>
                            </div>
                            {% endfor %}
                        </div>

                        <!-- Cut Tab -->
                        <div class="tab-pane fade" id="cut-tab">
                            {% for tool in tools if tool.tool_type == 'end_mill' %}
                            {% set tool_key = tool.size | string %}
                            {% set params = material.gcode_standards.get('cut', {}).get(tool_key, {}) %}
                            <div class="card mb-2">
                                <div class="card-header py-2">
                                    <strong>{{ tool.description }}</strong> ({{ tool.size }}")
                                </div>
                                <div class="card-body py-2">
                                    <div class="row g-2">
                                        <div class="col-md-3">
                                            <label class="form-label small">Spindle Speed (RPM)</label>
                                            <input type="number" class="form-control form-control-sm"
                                                   name="cut_{{ tool.size }}_spindle_speed"
                                                   value="{{ params.get('spindle_speed', '') }}">
                                        </div>
                                        <div class="col-md-3">
                                            <label class="form-label small">Feed Rate (in/min)</label>
                                            <input type="number" step="0.1" class="form-control form-control-sm"
                                                   name="cut_{{ tool.size }}_feed_rate"
                                                   value="{{ params.get('feed_rate', '') }}">
                                        </div>
                                        <div class="col-md-3">
                                            <label class="form-label small">Plunge Rate (in/min)</label>
                                            <input type="number" step="0.1" class="form-control form-control-sm"
                                                   name="cut_{{ tool.size }}_plunge_rate"
                                                   value="{{ params.get('plunge_rate', '') }}">
                                        </div>
                                        <div class="col-md-3">
                                            <label class="form-label small">Pass Depth (in)</label>
                                            <input type="number" step="0.001" class="form-control form-control-sm"
                                                   name="cut_{{ tool.size }}_pass_depth"
                                                   value="{{ params.get('pass_depth', '') }}">
                                        </div>
                                    </div>
                                </div>
                            </div>
                            {% endfor %}
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <div class="d-flex gap-2">
        <button type="submit" class="btn btn-primary">Save Changes</button>
        <a href="{{ url_for('settings.materials') }}" class="btn btn-outline-secondary">Cancel</a>
    </div>
</form>

<script>
document.getElementById('material-form').addEventListener('change', function() {
    document.getElementById('sheet-dims').style.display = this.value === 'sheet' ? 'block' : 'none';
    document.getElementById('tube-dims').style.display = this.value === 'tube' ? 'block' : 'none';
});
</script>
{% endblock %}
```

---

### Tools Management Page (templates/settings/tools.html)

```html
{% extends "base.html" %}
{% block title %}Tools - Settings{% endblock %}

{% block content %}
<div class="d-flex justify-content-between align-items-center mb-4">
    <h1>Tools</h1>
    <button class="btn btn-success" data-bs-toggle="modal" data-bs-target="#addToolModal">
        + Add Tool
    </button>
</div>

<div class="row">
    <!-- Drill Bits -->
    <div class="col-md-6">
        <div class="card mb-3">
            <div class="card-header">Drill Bits</div>
            <ul class="list-group list-group-flush">
                {% for tool in tools if tool.tool_type == 'drill' %}
                <li class="list-group-item d-flex justify-content-between align-items-center">
                    <div>
                        <strong>{{ tool.size }}"</strong>
                        {% if tool.description %}<span class="text-muted">- {{ tool.description }}</span>{% endif %}
                    </div>
                    <form action="{{ url_for('settings.delete_tool', tool_id=tool.id) }}" method="POST" class="d-inline">
                        <button type="submit" class="btn btn-sm btn-outline-danger"
                                onclick="return confirm('Delete this tool?')">Delete</button>
                    </form>
                </li>
                {% else %}
                <li class="list-group-item text-muted">No drill bits defined</li>
                {% endfor %}
            </ul>
        </div>
    </div>

    <!-- End Mills -->
    <div class="col-md-6">
        <div class="card mb-3">
            <div class="card-header">End Mills</div>
            <ul class="list-group list-group-flush">
                {% for tool in tools if tool.tool_type == 'end_mill' %}
                <li class="list-group-item d-flex justify-content-between align-items-center">
                    <div>
                        <strong>{{ tool.size }}"</strong>
                        {% if tool.description %}<span class="text-muted">- {{ tool.description }}</span>{% endif %}
                    </div>
                    <form action="{{ url_for('settings.delete_tool', tool_id=tool.id) }}" method="POST" class="d-inline">
                        <button type="submit" class="btn btn-sm btn-outline-danger"
                                onclick="return confirm('Delete this tool?')">Delete</button>
                    </form>
                </li>
                {% else %}
                <li class="list-group-item text-muted">No end mills defined</li>
                {% endfor %}
            </ul>
        </div>
    </div>
</div>

<a href="{{ url_for('settings.index') }}" class="btn btn-outline-secondary">Back to Settings</a>

<!-- Add Tool Modal -->
<div class="modal fade" id="addToolModal" tabindex="-1">
    <div class="modal-dialog">
        <div class="modal-content">
            <form method="POST" action="{{ url_for('settings.create_tool') }}">
                <div class="modal-header">
                    <h5 class="modal-title">Add Tool</h5>
                    <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
                </div>
                <div class="modal-body">
                    <div class="mb-3">
                        <label class="form-label">Tool Type</label>
                        <select class="form-select" name="tool_type" required>
                            <option value="drill">Drill Bit</option>
                            <option value="end_mill">End Mill</option>
                        </select>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Size (inches)</label>
                        <input type="number" step="0.001" class="form-control" name="size" required
                               placeholder="e.g., 0.125">
                        <div class="form-text">Common sizes: 0.125 (1/8"), 0.1875 (3/16"), 0.25 (1/4")</div>
                    </div>
                    <div class="mb-3">
                        <label class="form-label">Description (optional)</label>
                        <input type="text" class="form-control" name="description"
                               placeholder="e.g., 1/8&quot; cobalt drill bit">
                    </div>
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Cancel</button>
                    <button type="submit" class="btn btn-primary">Add Tool</button>
                </div>
            </form>
        </div>
    </div>
</div>
{% endblock %}
```

---

## JavaScript Utility Functions

Add these utility functions to help with common tasks:

### static/js/utils.js

```javascript
/**
 * Utility functions for the G-Code Generator
 */

// Format a number as a fraction string (for display)
function formatAsFraction(decimal) {
    const fractions = {
        0.0625: '1/16',
        0.125: '1/8',
        0.1875: '3/16',
        0.25: '1/4',
        0.3125: '5/16',
        0.375: '3/8',
        0.4375: '7/16',
        0.5: '1/2'
    };
    return fractions[decimal] || decimal.toString();
}

// Debounce function for input handlers
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

// Generate a unique ID for operations
function generateId(prefix = 'op') {
    return `${prefix}_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

// Deep clone an object
function deepClone(obj) {
    return JSON.parse(JSON.stringify(obj));
}

// Show a toast notification (requires Bootstrap)
function showToast(message, type = 'info') {
    const toastContainer = document.getElementById('toast-container') ||
        createToastContainer();

    const toastEl = document.createElement('div');
    toastEl.className = `toast align-items-center text-white bg-${type} border-0`;
    toastEl.setAttribute('role', 'alert');
    toastEl.innerHTML = `
        <div class="d-flex">
            <div class="toast-body">${message}</div>
            <button type="button" class="btn-close btn-close-white me-2 m-auto"
                    data-bs-dismiss="toast"></button>
        </div>
    `;

    toastContainer.appendChild(toastEl);
    const toast = new bootstrap.Toast(toastEl, { delay: 3000 });
    toast.show();

    toastEl.addEventListener('hidden.bs.toast', () => toastEl.remove());
}

function createToastContainer() {
    const container = document.createElement('div');
    container.id = 'toast-container';
    container.className = 'toast-container position-fixed bottom-0 end-0 p-3';
    document.body.appendChild(container);
    return container;
}

// Validate a coordinate value
function isValidCoordinate(value, min = 0, max = 15) {
    const num = parseFloat(value);
    return !isNaN(num) && num >= min && num <= max;
}

// Format coordinate for display
function formatCoordinate(value) {
    return parseFloat(value).toFixed(3);
}
```

---

## Error Handling Patterns

### Displaying Validation Errors in Forms

```javascript
// In project-editor.js

function showFieldError(fieldId, message) {
    const field = document.getElementById(fieldId);
    if (!field) return;

    field.classList.add('is-invalid');

    // Add or update feedback element
    let feedback = field.nextElementSibling;
    if (!feedback || !feedback.classList.contains('invalid-feedback')) {
        feedback = document.createElement('div');
        feedback.className = 'invalid-feedback';
        field.parentNode.insertBefore(feedback, field.nextSibling);
    }
    feedback.textContent = message;
}

function clearFieldError(fieldId) {
    const field = document.getElementById(fieldId);
    if (!field) return;

    field.classList.remove('is-invalid');
    const feedback = field.nextElementSibling;
    if (feedback && feedback.classList.contains('invalid-feedback')) {
        feedback.remove();
    }
}

function clearAllErrors() {
    document.querySelectorAll('.is-invalid').forEach(el => {
        el.classList.remove('is-invalid');
    });
    document.querySelectorAll('.invalid-feedback').forEach(el => {
        el.remove();
    });
}
```

### API Error Handling

```javascript
// Wrapper for fetch with error handling
async function apiCall(url, options = {}) {
    try {
        const response = await fetch(url, {
            ...options,
            headers: {
                'Content-Type': 'application/json',
                ...options.headers
            }
        });

        const data = await response.json();

        if (!response.ok) {
            throw new ApiError(data.error || data.message || 'Request failed', response.status, data);
        }

        return data;
    } catch (error) {
        if (error instanceof ApiError) {
            throw error;
        }
        throw new ApiError('Network error - please check your connection', 0);
    }
}

class ApiError extends Error {
    constructor(message, status, data = null) {
        super(message);
        this.status = status;
        this.data = data;
    }
}

// Usage:
async function saveProject() {
    try {
        const result = await apiCall(`/api/projects/${this.projectId}/save`, {
            method: 'POST',
            body: JSON.stringify(this.data)
        });
        showToast('Project saved!', 'success');
        return result;
    } catch (error) {
        if (error.data && error.data.errors) {
            error.data.errors.forEach(err => showToast(err, 'danger'));
        } else {
            showToast(error.message, 'danger');
        }
        return null;
    }
}
```
