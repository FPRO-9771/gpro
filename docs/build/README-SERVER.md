# G-Code Generator Web Application - Backend Build Instructions

This document provides instructions for Claude Code sessions to build the Flask backend for the G-code generator web application.

## Overview

Transform the existing Python CLI G-code generator into a web application backend with:
- Settings management (materials, machine parameters, G-code standards)
- Project CRUD operations with drill/cut operations and patterns
- G-code generation on-demand
- PostgreSQL database for persistent storage

**Tech Stack**: Flask + SQLAlchemy + PostgreSQL + Gunicorn

---

## Project Structure

Create this directory structure:

```
generate-g-code/
├── app.py                      # Flask app entry point
├── config.py                   # Configuration management
├── Procfile                    # Heroku: web: gunicorn app:app
├── runtime.txt                 # Heroku: python-3.13.0
├── requirements.txt            # Updated dependencies
├── seed_data.py                # Populate default settings
│
├── migrations/                 # Flask-Migrate database migrations
│
├── src/                        # Core logic (existing + new)
│   ├── __init__.py
│   ├── file_parser.py          # Keep existing
│   ├── gcode_generator.py      # Refactor for loops + settings integration
│   ├── pattern_expander.py     # NEW: Expand patterns to coordinates
│   ├── tube_void_checker.py    # NEW: Tube void detection
│   ├── visualizer.py           # Modify for web (return SVG/PNG data)
│   └── models.py               # NEW: Shared dataclasses
│
├── web/                        # Web layer
│   ├── __init__.py
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── main.py             # Home, dashboard
│   │   ├── projects.py         # Project CRUD
│   │   ├── settings.py         # Settings pages
│   │   └── api.py              # AJAX endpoints
│   ├── models.py               # SQLAlchemy database models
│   └── services/
│       ├── __init__.py
│       ├── settings_service.py # Settings management
│       ├── project_service.py  # Project management
│       └── gcode_service.py    # G-code generation orchestration
│
├── templates/                  # Jinja2 templates
└── static/                     # CSS, JS, images
```

---

## Dependencies (requirements.txt)

```
Flask>=3.0.0
gunicorn>=21.0.0
WTForms>=3.1.0
Flask-WTF>=1.2.0
Flask-SQLAlchemy>=3.1.0
Flask-Migrate>=4.0.0
psycopg2-binary>=2.9.0
matplotlib>=3.8.0
numpy>=1.26.0
Pillow>=10.0.0
python-dotenv>=1.0.0
```

---

## Flask App Setup (app.py)

Create Flask application with blueprints and database:

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from config import Config

db = SQLAlchemy()
migrate = Migrate()

def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)

    # Initialize extensions
    db.init_app(app)
    migrate.init_app(app, db)

    # Register blueprints
    from web.routes.main import main_bp
    from web.routes.projects import projects_bp
    from web.routes.settings import settings_bp
    from web.routes.api import api_bp

    app.register_blueprint(main_bp)
    app.register_blueprint(projects_bp, url_prefix='/projects')
    app.register_blueprint(settings_bp, url_prefix='/settings')
    app.register_blueprint(api_bp, url_prefix='/api')

    return app

app = create_app()

if __name__ == '__main__':
    app.run(debug=True)
```

---

## Configuration (config.py)

```python
import os

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-key-change-in-production')

    # Database URL - Heroku sets DATABASE_URL automatically
    # For local dev, use SQLite
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL', 'sqlite:///gcode.db')

    # Heroku uses postgres:// but SQLAlchemy needs postgresql://
    if SQLALCHEMY_DATABASE_URI.startswith('postgres://'):
        SQLALCHEMY_DATABASE_URI = SQLALCHEMY_DATABASE_URI.replace('postgres://', 'postgresql://', 1)

    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

---

## Database Models (web/models.py)

```python
from app import db
from datetime import datetime
import uuid

class Material(db.Model):
    """Material type with G-code standards per tool size."""
    id = db.Column(db.String(50), primary_key=True)
    display_name = db.Column(db.String(100), nullable=False)
    base_material = db.Column(db.String(50), nullable=False)  # 'aluminum', 'polycarbonate'
    form = db.Column(db.String(20), nullable=False)  # 'sheet' or 'tube'

    # For sheets
    thickness = db.Column(db.Float, nullable=True)

    # For tubes
    outer_width = db.Column(db.Float, nullable=True)
    outer_height = db.Column(db.Float, nullable=True)
    wall_thickness = db.Column(db.Float, nullable=True)

    # G-code standards stored as JSON
    gcode_standards = db.Column(db.JSON, nullable=False, default=dict)
    # Format: {"drill": {"0.125": {"spindle_speed": 1000, ...}}, "cut": {...}}


class MachineSettings(db.Model):
    """Machine configuration (singleton - one row)."""
    id = db.Column(db.Integer, primary_key=True, default=1)
    name = db.Column(db.String(100), default='OMIO CNC')
    max_x = db.Column(db.Float, default=15.0)
    max_y = db.Column(db.Float, default=15.0)
    units = db.Column(db.String(10), default='inches')
    controller_type = db.Column(db.String(20), default='mach3')
    supports_loops = db.Column(db.Boolean, default=True)
    supports_canned_cycles = db.Column(db.Boolean, default=True)


class GeneralSettings(db.Model):
    """General G-code settings (singleton - one row)."""
    id = db.Column(db.Integer, primary_key=True, default=1)
    safety_height = db.Column(db.Float, default=0.5)
    travel_height = db.Column(db.Float, default=0.2)
    spindle_warmup_seconds = db.Column(db.Integer, default=2)


class Tool(db.Model):
    """Available tools (drill bits and end mills)."""
    id = db.Column(db.Integer, primary_key=True)
    tool_type = db.Column(db.String(20), nullable=False)  # 'drill' or 'end_mill'
    size = db.Column(db.Float, nullable=False)
    description = db.Column(db.String(100))


class Project(db.Model):
    """User project with operations."""
    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    name = db.Column(db.String(200), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    modified_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    project_type = db.Column(db.String(20), nullable=False)  # 'drill' or 'cut'
    material_id = db.Column(db.String(50), db.ForeignKey('material.id'))
    material = db.relationship('Material')

    drill_bit_size = db.Column(db.Float)
    end_mill_size = db.Column(db.Float)

    # Operations stored as JSON (flexible structure)
    operations = db.Column(db.JSON, nullable=False, default=dict)
    # Format: {"drill_holes": [...], "circular_cuts": [...], "line_cuts": [...]}

    # Tube void settings
    tube_void_skip = db.Column(db.Boolean, default=False)
```

---

## Operations JSON Structure

The `operations` column in the Project model stores a JSON object with this structure:

```json
{
  "drill_holes": [
    {"id": "d1", "type": "single", "x": 1.25, "y": 0.5},
    {
      "id": "d2",
      "type": "pattern_linear",
      "start_x": 1.25, "start_y": 1.0,
      "axis": "y", "spacing": 0.5, "count": 4
    },
    {
      "id": "d3",
      "type": "pattern_grid",
      "start_x": 3.0, "start_y": 0.5,
      "x_spacing": 1.0, "y_spacing": 0.5,
      "x_count": 3, "y_count": 4
    }
  ],
  "circular_cuts": [
    {"id": "c1", "type": "single", "center_x": 1.25, "center_y": 4.02, "diameter": 0.8},
    {
      "id": "c2",
      "type": "pattern_linear",
      "start_center_x": 0.5, "start_center_y": 1.0,
      "diameter": 0.5, "axis": "x", "spacing": 2.0, "count": 4
    }
  ],
  "line_cuts": [
    {
      "id": "l1",
      "points": [
        {"x": 0, "y": 0, "line_type": "start"},
        {"x": 5.5, "y": 0, "line_type": "straight"},
        {"x": 5.5, "y": 2.5, "line_type": "straight"},
        {"x": 2.5, "y": 2.5, "line_type": "arc", "arc_center_x": 4.0, "arc_center_y": 2.5}
      ],
      "closed": true
    }
  ]
}
```

**Pattern Types:**
- `single`: One point at (x, y)
- `pattern_linear`: Repeat along an axis (start_x, start_y, axis, spacing, count)
- `pattern_grid`: Rectangular grid (start_x, start_y, x_spacing, y_spacing, x_count, y_count)

**Line Types:**
- `straight`: Linear interpolation to this point
- `arc`: Circular arc to this point using `arc_center_x`, `arc_center_y` as the arc center

---

## Pattern Expander (src/pattern_expander.py)

Create a new module to expand pattern definitions into individual coordinates:

```python
import math
from dataclasses import dataclass
from typing import List

@dataclass
class ExpandedPoint:
    x: float
    y: float
    source_id: str


class PatternExpander:
    @staticmethod
    def expand_linear(start_x, start_y, axis, spacing, count, source_id) -> List[ExpandedPoint]:
        """Expand 'every N inches for M items on axis' pattern."""
        points = []
        for i in range(count):
            x = start_x + (i * spacing if axis == 'x' else 0)
            y = start_y + (i * spacing if axis == 'y' else 0)
            points.append(ExpandedPoint(x, y, source_id))
        return points

    @staticmethod
    def expand_grid(start_x, start_y, x_spacing, y_spacing, x_count, y_count, source_id) -> List[ExpandedPoint]:
        """Expand rectangular grid pattern."""
        points = []
        for row in range(y_count):
            for col in range(x_count):
                points.append(ExpandedPoint(
                    start_x + col * x_spacing,
                    start_y + row * y_spacing,
                    source_id
                ))
        return points

    @staticmethod
    def expand_circular(center_x, center_y, radius, count, start_angle, source_id) -> List[ExpandedPoint]:
        """Expand bolt circle pattern."""
        points = []
        for i in range(count):
            angle = math.radians(start_angle) + (i * 2 * math.pi / count)
            points.append(ExpandedPoint(
                center_x + radius * math.cos(angle),
                center_y + radius * math.sin(angle),
                source_id
            ))
        return points
```

---

## Tube Void Checker (src/tube_void_checker.py)

For tube stock, detect and skip operations that fall within the hollow center:

```python
from dataclasses import dataclass
from typing import List, Tuple

@dataclass
class TubeProfile:
    """Defines the cross-section of a tube."""
    outer_width: float   # inches
    outer_height: float  # inches
    wall_thickness: float  # inches

    @property
    def void_bounds(self) -> Tuple[float, float, float, float]:
        """Return (x_min, y_min, x_max, y_max) of the hollow void."""
        return (
            self.wall_thickness,
            self.wall_thickness,
            self.outer_width - self.wall_thickness,
            self.outer_height - self.wall_thickness
        )


class TubeVoidChecker:
    """Check if operations fall within the void area of tube stock."""

    def __init__(self, tube: TubeProfile):
        self.tube = tube
        self.void = tube.void_bounds

    def point_in_void(self, x: float, y: float, tool_radius: float = 0) -> bool:
        """Check if a point (with tool radius) is entirely in the void."""
        x_min, y_min, x_max, y_max = self.void
        return (
            x - tool_radius > x_min and
            x + tool_radius < x_max and
            y - tool_radius > y_min and
            y + tool_radius < y_max
        )

    def filter_drill_points(self, points: List, tool_radius: float = 0) -> List:
        """Remove drill points that fall entirely in the void."""
        return [p for p in points if not self.point_in_void(p.x, p.y, tool_radius)]

    def segment_crosses_void(self, x1: float, y1: float, x2: float, y2: float) -> bool:
        """Check if a line segment crosses the void (for cut operations)."""
        x_min, y_min, x_max, y_max = self.void
        # Check if both endpoints are in void
        if self.point_in_void(x1, y1) and self.point_in_void(x2, y2):
            return True
        return False

    def get_cut_segments(self, points: List) -> List[List]:
        """
        Split a cutting path into segments that skip the void.
        Returns list of point lists, where each list is a continuous cut.
        """
        segments = []
        current_segment = []

        for i, point in enumerate(points):
            in_void = self.point_in_void(point.x, point.y)

            if not in_void:
                current_segment.append(point)
            else:
                if current_segment:
                    segments.append(current_segment)
                    current_segment = []

        if current_segment:
            segments.append(current_segment)

        return segments
```

---

## G-Code Generator Updates (src/gcode_generator.py)

Modify the existing generator to:

1. **Accept settings-based parameters** instead of interactive prompts
2. **Use G-code loops** for multi-pass operations (when controller supports it)
3. **Use G83 peck drilling** for drill operations

Key changes to implement:

```python
import math

class GCodeGenerator:
    def __init__(self, params, settings, use_loops=True):
        self.params = params
        self.settings = settings  # General settings (safety_height, travel_height)
        self.use_loops = use_loops
        self.var_counter = 100

    def _generate_peck_drill(self, points, params):
        """Use G83 canned cycle for drilling (reduces file size)."""
        lines = []
        peck_mm = params.pecking_depth * 25.4
        depth_mm = params.material_depth * 25.4

        first = points[0]
        lines.append(f"G83 X{first.x*25.4:.3f} Y{first.y*25.4:.3f} "
                     f"Z-{depth_mm:.3f} R{self.settings['travel_height']*25.4:.3f} "
                     f"Q{peck_mm:.3f} F{params.plunge_rate}")

        for p in points[1:]:
            lines.append(f"X{p.x*25.4:.3f} Y{p.y*25.4:.3f}")

        lines.append("G80 ; Cancel canned cycle")
        return lines

    def _generate_looped_circular_cut(self, cut, params):
        """Use WHILE loop for multi-pass circular cuts."""
        total_depth_mm = params.material_depth * 25.4
        pass_depth_mm = params.path_depth * 25.4
        num_passes = math.ceil(total_depth_mm / pass_depth_mm)

        v = self.var_counter
        self.var_counter += 5

        lines = [
            f"#{v} = 0 (pass counter)",
            f"#{v+1} = {num_passes} (total passes)",
            f"#{v+2} = {pass_depth_mm:.3f} (depth per pass)",
            f"WHILE [#{v} LT #{v+1}] DO1",
            f"  #{v+3} = [[#{v} + 1] * #{v+2}]",
            f"  G1 Z-#{v+3} F{params.plunge_rate}",
            f"  G2 X... Y... I... J0 F{params.feed_rate}",
            f"  #{v} = [#{v} + 1]",
            f"END1"
        ]
        return lines
```

---

## API Routes (web/routes/api.py)

```python
from flask import Blueprint, request, jsonify, send_file
from web.services.project_service import ProjectService
from web.services.gcode_service import GCodeService
import io

api_bp = Blueprint('api', __name__)


@api_bp.route('/projects/<project_id>/save', methods=['POST'])
def save_project(project_id):
    data = request.get_json()
    ProjectService.save(project_id, data)
    return jsonify({'status': 'saved'})


@api_bp.route('/projects/<project_id>/preview', methods=['POST'])
def preview(project_id):
    project = ProjectService.get(project_id)
    svg_data = GCodeService.generate_preview_svg(project)
    return jsonify({'svg': svg_data})


@api_bp.route('/projects/<project_id>/download/<gcode_type>')
def download_gcode(project_id, gcode_type):
    project = ProjectService.get(project_id)
    gcode = GCodeService.generate(project, gcode_type)  # 'drill' or 'cut'

    buffer = io.BytesIO(gcode.encode('utf-8'))
    return send_file(
        buffer,
        mimetype='text/plain',
        as_attachment=True,
        download_name=f"{project['name']}_{gcode_type}.gcode"
    )


@api_bp.route('/projects/<project_id>/validate', methods=['POST'])
def validate(project_id):
    project = ProjectService.get(project_id)
    errors = GCodeService.validate(project)
    return jsonify({'valid': len(errors) == 0, 'errors': errors})
```

---

## Seed Data Script (seed_data.py)

Create a script to populate initial settings:

```python
from app import create_app, db
from web.models import Material, MachineSettings, GeneralSettings, Tool

app = create_app()

with app.app_context():
    # Add default materials
    if not Material.query.first():
        materials = [
            Material(
                id='aluminum_sheet_0125',
                display_name='Aluminum Sheet 1/8"',
                base_material='aluminum',
                form='sheet',
                thickness=0.125,
                gcode_standards={
                    'drill': {'0.125': {'spindle_speed': 1000, 'feed_rate': 2.0, 'plunge_rate': 1.0, 'pecking_depth': 0.05}},
                    'cut': {'0.125': {'spindle_speed': 10000, 'feed_rate': 10.0, 'plunge_rate': 1.5, 'pass_depth': 0.02}}
                }
            ),
            Material(
                id='aluminum_sheet_025',
                display_name='Aluminum Sheet 1/4"',
                base_material='aluminum',
                form='sheet',
                thickness=0.25,
                gcode_standards={
                    'drill': {'0.125': {'spindle_speed': 800, 'feed_rate': 1.5, 'plunge_rate': 0.8, 'pecking_depth': 0.04}},
                    'cut': {'0.125': {'spindle_speed': 8000, 'feed_rate': 8.0, 'plunge_rate': 1.0, 'pass_depth': 0.015}}
                }
            ),
            Material(
                id='polycarbonate_sheet_025',
                display_name='Polycarbonate Sheet 1/4"',
                base_material='polycarbonate',
                form='sheet',
                thickness=0.25,
                gcode_standards={
                    'drill': {'0.125': {'spindle_speed': 2000, 'feed_rate': 4.0, 'plunge_rate': 2.0, 'pecking_depth': 0.1}},
                    'cut': {'0.125': {'spindle_speed': 15000, 'feed_rate': 20.0, 'plunge_rate': 3.0, 'pass_depth': 0.05}}
                }
            ),
            Material(
                id='aluminum_tube_2x1_0125',
                display_name='Aluminum Tube 2x1 (0.125 wall)',
                base_material='aluminum',
                form='tube',
                outer_width=2.0,
                outer_height=1.0,
                wall_thickness=0.125,
                gcode_standards={
                    'drill': {'0.125': {'spindle_speed': 1000, 'feed_rate': 2.0, 'plunge_rate': 1.0, 'pecking_depth': 0.05}},
                    'cut': {'0.125': {'spindle_speed': 10000, 'feed_rate': 10.0, 'plunge_rate': 1.5, 'pass_depth': 0.02}}
                }
            ),
        ]
        db.session.add_all(materials)

    # Add default machine settings
    if not MachineSettings.query.first():
        db.session.add(MachineSettings(
            name='OMIO CNC',
            max_x=15.0,
            max_y=15.0,
            units='inches',
            controller_type='mach3',
            supports_loops=True,
            supports_canned_cycles=True
        ))

    # Add default general settings
    if not GeneralSettings.query.first():
        db.session.add(GeneralSettings(
            safety_height=0.5,
            travel_height=0.2,
            spindle_warmup_seconds=2
        ))

    # Add default tools
    if not Tool.query.first():
        tools = [
            Tool(tool_type='drill', size=0.125, description='1/8" drill bit'),
            Tool(tool_type='drill', size=0.1875, description='3/16" drill bit'),
            Tool(tool_type='drill', size=0.25, description='1/4" drill bit'),
            Tool(tool_type='end_mill', size=0.125, description='1/8" end mill'),
            Tool(tool_type='end_mill', size=0.1875, description='3/16" end mill'),
            Tool(tool_type='end_mill', size=0.25, description='1/4" end mill'),
        ]
        db.session.add_all(tools)

    db.session.commit()
    print("Seed data added successfully!")
```

---

## Local Development

```bash
# Create virtual environment
python -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Initialize database migrations (first time only)
flask db init

# Create and run migrations
flask db migrate -m "Initial models"
flask db upgrade

# Seed default data
python seed_data.py

# Run locally
flask run --debug

# Or with gunicorn (production-like)
gunicorn app:app
```

---

## Implementation Order

1. **Phase 1**: Flask app setup
   - Create `app.py` with blueprints and SQLAlchemy
   - Create `config.py` for configuration
   - Create `web/models.py` with all database models
   - Set up Flask-Migrate

2. **Phase 2**: Services layer
   - Create `web/services/settings_service.py`
   - Create `web/services/project_service.py`
   - Create `web/services/gcode_service.py`

3. **Phase 3**: Core modules
   - Create `src/pattern_expander.py`
   - Create `src/tube_void_checker.py`
   - Refactor `src/gcode_generator.py` for loops (G83, WHILE)

4. **Phase 4**: Routes
   - Create `web/routes/main.py` for home/dashboard
   - Create `web/routes/projects.py` for project CRUD
   - Create `web/routes/settings.py` for settings management
   - Create `web/routes/api.py` for AJAX endpoints

5. **Phase 5**: Seed data
   - Create `seed_data.py`
   - Test with local SQLite database

---

## Key Files to Create/Modify

| File | Action | Purpose |
|------|--------|---------|
| `app.py` | Create | Flask entry point with db init |
| `config.py` | Create | Configuration with DATABASE_URL |
| `requirements.txt` | Update | Add Flask, SQLAlchemy, psycopg2 |
| `seed_data.py` | Create | Populate default settings |
| `web/__init__.py` | Create | Web package init |
| `web/models.py` | Create | SQLAlchemy models |
| `web/routes/__init__.py` | Create | Routes package init |
| `web/routes/main.py` | Create | Home/dashboard routes |
| `web/routes/projects.py` | Create | Project CRUD routes |
| `web/routes/settings.py` | Create | Settings routes |
| `web/routes/api.py` | Create | API endpoints |
| `web/services/__init__.py` | Create | Services package init |
| `web/services/settings_service.py` | Create | Settings business logic |
| `web/services/project_service.py` | Create | Project business logic |
| `web/services/gcode_service.py` | Create | G-code generation |
| `src/gcode_generator.py` | Modify | Add loops, settings integration |
| `src/pattern_expander.py` | Create | Pattern expansion |
| `src/tube_void_checker.py` | Create | Tube void detection |
| `src/visualizer.py` | Modify | Return SVG/PNG data |

---

## Services Layer

### Settings Service (web/services/settings_service.py)

```python
from typing import Optional, Dict, Any, List
from app import db
from web.models import Material, MachineSettings, GeneralSettings, Tool


class SettingsService:
    """Service for managing application settings."""

    # --- Material Methods ---

    @staticmethod
    def get_all_materials() -> List[Material]:
        """Get all materials ordered by display name."""
        return Material.query.order_by(Material.display_name).all()

    @staticmethod
    def get_material(material_id: str) -> Optional[Material]:
        """Get a single material by ID."""
        return Material.query.get(material_id)

    @staticmethod
    def get_materials_dict() -> Dict[str, Dict[str, Any]]:
        """Get all materials as a dictionary for JSON serialization."""
        materials = Material.query.all()
        return {
            m.id: {
                'id': m.id,
                'display_name': m.display_name,
                'base_material': m.base_material,
                'form': m.form,
                'thickness': m.thickness,
                'outer_width': m.outer_width,
                'outer_height': m.outer_height,
                'wall_thickness': m.wall_thickness,
                'gcode_standards': m.gcode_standards
            }
            for m in materials
        }

    @staticmethod
    def create_material(data: Dict[str, Any]) -> Material:
        """Create a new material."""
        material = Material(
            id=data['id'],
            display_name=data['display_name'],
            base_material=data['base_material'],
            form=data['form'],
            thickness=data.get('thickness'),
            outer_width=data.get('outer_width'),
            outer_height=data.get('outer_height'),
            wall_thickness=data.get('wall_thickness'),
            gcode_standards=data.get('gcode_standards', {})
        )
        db.session.add(material)
        db.session.commit()
        return material

    @staticmethod
    def update_material(material_id: str, data: Dict[str, Any]) -> Optional[Material]:
        """Update an existing material."""
        material = Material.query.get(material_id)
        if not material:
            return None

        material.display_name = data.get('display_name', material.display_name)
        material.base_material = data.get('base_material', material.base_material)
        material.form = data.get('form', material.form)
        material.thickness = data.get('thickness')
        material.outer_width = data.get('outer_width')
        material.outer_height = data.get('outer_height')
        material.wall_thickness = data.get('wall_thickness')
        material.gcode_standards = data.get('gcode_standards', material.gcode_standards)

        db.session.commit()
        return material

    @staticmethod
    def delete_material(material_id: str) -> bool:
        """Delete a material. Returns False if material has associated projects."""
        material = Material.query.get(material_id)
        if not material:
            return False

        # Check for associated projects
        from web.models import Project
        if Project.query.filter_by(material_id=material_id).first():
            return False

        db.session.delete(material)
        db.session.commit()
        return True

    # --- Machine Settings Methods ---

    @staticmethod
    def get_machine_settings() -> MachineSettings:
        """Get machine settings (singleton pattern)."""
        settings = MachineSettings.query.get(1)
        if not settings:
            settings = MachineSettings(id=1)
            db.session.add(settings)
            db.session.commit()
        return settings

    @staticmethod
    def update_machine_settings(data: Dict[str, Any]) -> MachineSettings:
        """Update machine settings."""
        settings = SettingsService.get_machine_settings()
        settings.name = data.get('name', settings.name)
        settings.max_x = float(data.get('max_x', settings.max_x))
        settings.max_y = float(data.get('max_y', settings.max_y))
        settings.units = data.get('units', settings.units)
        settings.controller_type = data.get('controller_type', settings.controller_type)
        settings.supports_loops = data.get('supports_loops', settings.supports_loops)
        settings.supports_canned_cycles = data.get('supports_canned_cycles', settings.supports_canned_cycles)
        db.session.commit()
        return settings

    # --- General Settings Methods ---

    @staticmethod
    def get_general_settings() -> GeneralSettings:
        """Get general settings (singleton pattern)."""
        settings = GeneralSettings.query.get(1)
        if not settings:
            settings = GeneralSettings(id=1)
            db.session.add(settings)
            db.session.commit()
        return settings

    @staticmethod
    def update_general_settings(data: Dict[str, Any]) -> GeneralSettings:
        """Update general settings."""
        settings = SettingsService.get_general_settings()
        settings.safety_height = float(data.get('safety_height', settings.safety_height))
        settings.travel_height = float(data.get('travel_height', settings.travel_height))
        settings.spindle_warmup_seconds = int(data.get('spindle_warmup_seconds', settings.spindle_warmup_seconds))
        db.session.commit()
        return settings

    # --- Tool Methods ---

    @staticmethod
    def get_all_tools() -> List[Tool]:
        """Get all tools ordered by type and size."""
        return Tool.query.order_by(Tool.tool_type, Tool.size).all()

    @staticmethod
    def get_tools_by_type(tool_type: str) -> List[Tool]:
        """Get tools filtered by type ('drill' or 'end_mill')."""
        return Tool.query.filter_by(tool_type=tool_type).order_by(Tool.size).all()

    @staticmethod
    def create_tool(data: Dict[str, Any]) -> Tool:
        """Create a new tool."""
        tool = Tool(
            tool_type=data['tool_type'],
            size=float(data['size']),
            description=data.get('description', '')
        )
        db.session.add(tool)
        db.session.commit()
        return tool

    @staticmethod
    def delete_tool(tool_id: int) -> bool:
        """Delete a tool."""
        tool = Tool.query.get(tool_id)
        if not tool:
            return False
        db.session.delete(tool)
        db.session.commit()
        return True
```

---

### Project Service (web/services/project_service.py)

```python
from typing import Optional, Dict, Any, List
from datetime import datetime
from app import db
from web.models import Project, Material
import uuid
import json


class ProjectService:
    """Service for managing projects."""

    @staticmethod
    def get_all() -> List[Project]:
        """Get all projects ordered by modification date (newest first)."""
        return Project.query.order_by(Project.modified_at.desc()).all()

    @staticmethod
    def get(project_id: str) -> Optional[Project]:
        """Get a single project by ID."""
        return Project.query.get(project_id)

    @staticmethod
    def get_as_dict(project_id: str) -> Optional[Dict[str, Any]]:
        """Get a project as a dictionary for JSON serialization."""
        project = Project.query.get(project_id)
        if not project:
            return None

        return {
            'id': project.id,
            'name': project.name,
            'project_type': project.project_type,
            'material_id': project.material_id,
            'drill_bit_size': project.drill_bit_size,
            'end_mill_size': project.end_mill_size,
            'operations': project.operations or {'drill_holes': [], 'circular_cuts': [], 'line_cuts': []},
            'tube_void_skip': project.tube_void_skip,
            'created_at': project.created_at.isoformat() if project.created_at else None,
            'modified_at': project.modified_at.isoformat() if project.modified_at else None
        }

    @staticmethod
    def create(data: Dict[str, Any]) -> Project:
        """Create a new project."""
        project = Project(
            id=str(uuid.uuid4()),
            name=data['name'],
            project_type=data['project_type'],
            material_id=data.get('material_id'),
            drill_bit_size=data.get('drill_bit_size'),
            end_mill_size=data.get('end_mill_size'),
            operations={'drill_holes': [], 'circular_cuts': [], 'line_cuts': []},
            tube_void_skip=data.get('tube_void_skip', False)
        )
        db.session.add(project)
        db.session.commit()
        return project

    @staticmethod
    def save(project_id: str, data: Dict[str, Any]) -> Optional[Project]:
        """Save/update a project from editor data."""
        project = Project.query.get(project_id)
        if not project:
            return None

        # Update fields
        project.name = data.get('name', project.name)
        project.project_type = data.get('project_type', project.project_type)
        project.material_id = data.get('material_id', project.material_id)
        project.drill_bit_size = data.get('drill_bit_size')
        project.end_mill_size = data.get('end_mill_size')
        project.tube_void_skip = data.get('tube_void_skip', False)

        # Update operations (ensure proper structure)
        operations = data.get('operations', {})
        project.operations = {
            'drill_holes': operations.get('drill_holes', []),
            'circular_cuts': operations.get('circular_cuts', []),
            'line_cuts': operations.get('line_cuts', [])
        }

        project.modified_at = datetime.utcnow()
        db.session.commit()
        return project

    @staticmethod
    def delete(project_id: str) -> bool:
        """Delete a project."""
        project = Project.query.get(project_id)
        if not project:
            return False
        db.session.delete(project)
        db.session.commit()
        return True

    @staticmethod
    def duplicate(project_id: str, new_name: str = None) -> Optional[Project]:
        """Duplicate a project with a new ID."""
        original = Project.query.get(project_id)
        if not original:
            return None

        new_project = Project(
            id=str(uuid.uuid4()),
            name=new_name or f"{original.name} (Copy)",
            project_type=original.project_type,
            material_id=original.material_id,
            drill_bit_size=original.drill_bit_size,
            end_mill_size=original.end_mill_size,
            operations=json.loads(json.dumps(original.operations)),  # Deep copy
            tube_void_skip=original.tube_void_skip
        )
        db.session.add(new_project)
        db.session.commit()
        return new_project
```

---

### G-Code Service (web/services/gcode_service.py)

```python
from typing import Dict, Any, List, Optional, Tuple
from dataclasses import dataclass
import math
import io

from web.models import Project, Material
from web.services.settings_service import SettingsService
from src.pattern_expander import PatternExpander, ExpandedPoint
from src.tube_void_checker import TubeVoidChecker, TubeProfile


@dataclass
class GCodeParams:
    """Parameters for G-code generation."""
    spindle_speed: int
    feed_rate: float
    plunge_rate: float
    pecking_depth: float = 0.0
    pass_depth: float = 0.0
    material_depth: float = 0.0


class GCodeService:
    """Service for G-code generation and preview."""

    INCHES_TO_MM = 25.4

    @staticmethod
    def get_gcode_params(material: Material, tool_size: float, operation_type: str) -> GCodeParams:
        """Get G-code parameters for a material/tool combination."""
        standards = material.gcode_standards.get(operation_type, {})
        tool_key = str(tool_size)

        # Try exact match, then find closest
        if tool_key in standards:
            params = standards[tool_key]
        else:
            # Default fallback parameters
            if operation_type == 'drill':
                params = {'spindle_speed': 1000, 'feed_rate': 2.0, 'plunge_rate': 1.0, 'pecking_depth': 0.05}
            else:
                params = {'spindle_speed': 10000, 'feed_rate': 10.0, 'plunge_rate': 1.5, 'pass_depth': 0.02}

        return GCodeParams(
            spindle_speed=params.get('spindle_speed', 1000),
            feed_rate=params.get('feed_rate', 10.0),
            plunge_rate=params.get('plunge_rate', 1.0),
            pecking_depth=params.get('pecking_depth', 0.05),
            pass_depth=params.get('pass_depth', 0.02),
            material_depth=material.thickness or material.wall_thickness or 0.125
        )

    @staticmethod
    def expand_operations(operations: Dict[str, Any]) -> Tuple[List[ExpandedPoint], List[Dict], List[Dict]]:
        """
        Expand all operations (patterns to individual points).
        Returns: (drill_points, circular_cuts, line_cuts)
        """
        drill_points = []
        circular_cuts = []
        line_cuts = []

        # Expand drill holes
        for op in operations.get('drill_holes', []):
            if op['type'] == 'single':
                drill_points.append(ExpandedPoint(op['x'], op['y'], op['id']))
            elif op['type'] == 'pattern_linear':
                drill_points.extend(PatternExpander.expand_linear(
                    op['start_x'], op['start_y'], op['axis'],
                    op['spacing'], op['count'], op['id']
                ))
            elif op['type'] == 'pattern_grid':
                drill_points.extend(PatternExpander.expand_grid(
                    op['start_x'], op['start_y'],
                    op['x_spacing'], op['y_spacing'],
                    op['x_count'], op['y_count'], op['id']
                ))

        # Expand circular cuts
        for op in operations.get('circular_cuts', []):
            if op['type'] == 'single':
                circular_cuts.append({
                    'center_x': op['center_x'],
                    'center_y': op['center_y'],
                    'diameter': op['diameter'],
                    'source_id': op['id']
                })
            elif op['type'] == 'pattern_linear':
                points = PatternExpander.expand_linear(
                    op['start_center_x'], op['start_center_y'], op['axis'],
                    op['spacing'], op['count'], op['id']
                )
                for p in points:
                    circular_cuts.append({
                        'center_x': p.x,
                        'center_y': p.y,
                        'diameter': op['diameter'],
                        'source_id': op['id']
                    })

        # Line cuts don't need expansion (already defined as point sequences)
        line_cuts = operations.get('line_cuts', [])

        return drill_points, circular_cuts, line_cuts

    @staticmethod
    def generate(project: Project, gcode_type: str) -> str:
        """Generate G-code for a project."""
        material = project.material
        general = SettingsService.get_general_settings()
        machine = SettingsService.get_machine_settings()

        # Get tool size
        if gcode_type == 'drill':
            tool_size = project.drill_bit_size or 0.125
        else:
            tool_size = project.end_mill_size or 0.125

        params = GCodeService.get_gcode_params(material, tool_size, gcode_type)

        # Expand operations
        drill_points, circular_cuts, line_cuts = GCodeService.expand_operations(project.operations)

        # Apply tube void filtering if enabled
        if project.tube_void_skip and material.form == 'tube':
            tube = TubeProfile(
                material.outer_width,
                material.outer_height,
                material.wall_thickness
            )
            checker = TubeVoidChecker(tube)
            tool_radius = tool_size / 2
            drill_points = checker.filter_drill_points(drill_points, tool_radius)

        # Generate G-code
        lines = GCodeService._generate_header(project.name, params, general)

        if gcode_type == 'drill' and drill_points:
            if machine.supports_canned_cycles:
                lines.extend(GCodeService._generate_peck_drill(drill_points, params, general))
            else:
                lines.extend(GCodeService._generate_manual_drill(drill_points, params, general))
        elif gcode_type == 'cut':
            if circular_cuts:
                lines.extend(GCodeService._generate_circular_cuts(
                    circular_cuts, params, general, machine.supports_loops, tool_size
                ))
            if line_cuts:
                lines.extend(GCodeService._generate_line_cuts(
                    line_cuts, params, general, machine.supports_loops
                ))

        lines.extend(GCodeService._generate_footer())

        return '\n'.join(lines)

    @staticmethod
    def _generate_header(name: str, params: GCodeParams, general) -> List[str]:
        """Generate G-code header."""
        safety_mm = general.safety_height * GCodeService.INCHES_TO_MM
        return [
            f"; G-Code for: {name}",
            f"; Generated by FIRST Robotics G-Code Generator",
            "",
            "G21 ; Units: mm",
            "G90 ; Absolute positioning",
            "G17 ; XY plane selection",
            f"G0 Z{safety_mm:.3f} ; Move to safety height",
            f"M3 S{params.spindle_speed} ; Start spindle",
            f"G4 P2 ; Dwell for spindle warmup",
            ""
        ]

    @staticmethod
    def _generate_footer() -> List[str]:
        """Generate G-code footer."""
        return [
            "",
            "M5 ; Stop spindle",
            "G0 Z12.7 ; Retract to safe height",
            "G0 X0 Y0 ; Return to origin",
            "M30 ; Program end"
        ]

    @staticmethod
    def _generate_peck_drill(points: List[ExpandedPoint], params: GCodeParams, general) -> List[str]:
        """Generate G83 peck drilling cycle."""
        lines = ["; --- Drill Operations (G83 Peck Cycle) ---"]

        peck_mm = params.pecking_depth * GCodeService.INCHES_TO_MM
        depth_mm = params.material_depth * GCodeService.INCHES_TO_MM
        travel_mm = general.travel_height * GCodeService.INCHES_TO_MM

        if not points:
            return lines

        first = points[0]
        lines.append(
            f"G83 X{first.x * GCodeService.INCHES_TO_MM:.3f} "
            f"Y{first.y * GCodeService.INCHES_TO_MM:.3f} "
            f"Z-{depth_mm:.3f} R{travel_mm:.3f} "
            f"Q{peck_mm:.3f} F{params.plunge_rate * GCodeService.INCHES_TO_MM:.1f}"
        )

        for p in points[1:]:
            lines.append(
                f"X{p.x * GCodeService.INCHES_TO_MM:.3f} "
                f"Y{p.y * GCodeService.INCHES_TO_MM:.3f}"
            )

        lines.append("G80 ; Cancel canned cycle")
        return lines

    @staticmethod
    def _generate_manual_drill(points: List[ExpandedPoint], params: GCodeParams, general) -> List[str]:
        """Generate manual drilling (for controllers without G83)."""
        lines = ["; --- Drill Operations (Manual Peck) ---"]

        peck_mm = params.pecking_depth * GCodeService.INCHES_TO_MM
        depth_mm = params.material_depth * GCodeService.INCHES_TO_MM
        safety_mm = general.safety_height * GCodeService.INCHES_TO_MM
        travel_mm = general.travel_height * GCodeService.INCHES_TO_MM

        for p in points:
            x_mm = p.x * GCodeService.INCHES_TO_MM
            y_mm = p.y * GCodeService.INCHES_TO_MM

            lines.append(f"G0 X{x_mm:.3f} Y{y_mm:.3f} ; Move to hole position")
            lines.append(f"G0 Z{travel_mm:.3f} ; Rapid to travel height")

            # Peck drilling
            current_depth = 0
            while current_depth < depth_mm:
                current_depth = min(current_depth + peck_mm, depth_mm)
                lines.append(f"G1 Z-{current_depth:.3f} F{params.plunge_rate * GCodeService.INCHES_TO_MM:.1f}")
                lines.append(f"G0 Z{travel_mm:.3f} ; Retract to clear chips")

            lines.append(f"G0 Z{safety_mm:.3f} ; Retract to safety height")
            lines.append("")

        return lines

    @staticmethod
    def _generate_circular_cuts(cuts: List[Dict], params: GCodeParams, general, use_loops: bool, tool_size: float) -> List[str]:
        """Generate circular cut operations."""
        lines = ["; --- Circular Cut Operations ---"]

        safety_mm = general.safety_height * GCodeService.INCHES_TO_MM
        depth_mm = params.material_depth * GCodeService.INCHES_TO_MM
        pass_depth_mm = params.pass_depth * GCodeService.INCHES_TO_MM
        num_passes = math.ceil(depth_mm / pass_depth_mm)

        tool_radius_mm = (tool_size / 2) * GCodeService.INCHES_TO_MM

        for cut in cuts:
            cx = cut['center_x'] * GCodeService.INCHES_TO_MM
            cy = cut['center_y'] * GCodeService.INCHES_TO_MM
            radius_mm = (cut['diameter'] / 2) * GCodeService.INCHES_TO_MM

            # Adjust for tool radius (cut on inside of circle)
            cut_radius = radius_mm - tool_radius_mm

            # Start point (3 o'clock position)
            start_x = cx + cut_radius
            start_y = cy

            lines.append(f"; Circle at ({cut['center_x']}, {cut['center_y']}), dia {cut['diameter']}\"")
            lines.append(f"G0 X{start_x:.3f} Y{start_y:.3f}")
            lines.append(f"G0 Z{general.travel_height * GCodeService.INCHES_TO_MM:.3f}")

            if use_loops:
                # Use WHILE loop for multi-pass
                lines.append(f"#100 = 0 ; pass counter")
                lines.append(f"#101 = {num_passes} ; total passes")
                lines.append(f"#102 = {pass_depth_mm:.3f} ; depth per pass")
                lines.append(f"WHILE [#100 LT #101] DO1")
                lines.append(f"  #103 = [[#100 + 1] * #102]")
                lines.append(f"  G1 Z-#103 F{params.plunge_rate * GCodeService.INCHES_TO_MM:.1f}")
                lines.append(f"  G2 I-{cut_radius:.3f} J0 F{params.feed_rate * GCodeService.INCHES_TO_MM:.1f}")
                lines.append(f"  #100 = [#100 + 1]")
                lines.append(f"END1")
            else:
                # Explicit passes
                for i in range(num_passes):
                    z_depth = min((i + 1) * pass_depth_mm, depth_mm)
                    lines.append(f"G1 Z-{z_depth:.3f} F{params.plunge_rate * GCodeService.INCHES_TO_MM:.1f}")
                    lines.append(f"G2 I-{cut_radius:.3f} J0 F{params.feed_rate * GCodeService.INCHES_TO_MM:.1f}")

            lines.append(f"G0 Z{safety_mm:.3f}")
            lines.append("")

        return lines

    @staticmethod
    def _generate_line_cuts(line_cuts: List[Dict], params: GCodeParams, general, use_loops: bool) -> List[str]:
        """Generate line cut operations."""
        lines = ["; --- Line Cut Operations ---"]

        safety_mm = general.safety_height * GCodeService.INCHES_TO_MM
        depth_mm = params.material_depth * GCodeService.INCHES_TO_MM
        pass_depth_mm = params.pass_depth * GCodeService.INCHES_TO_MM
        num_passes = math.ceil(depth_mm / pass_depth_mm)

        for cut in line_cuts:
            points = cut.get('points', [])
            if not points:
                continue

            lines.append(f"; Line cut with {len(points)} points")

            # Move to start
            start = points[0]
            lines.append(f"G0 X{start['x'] * GCodeService.INCHES_TO_MM:.3f} Y{start['y'] * GCodeService.INCHES_TO_MM:.3f}")
            lines.append(f"G0 Z{general.travel_height * GCodeService.INCHES_TO_MM:.3f}")

            # Multi-pass cutting
            for pass_num in range(num_passes):
                z_depth = min((pass_num + 1) * pass_depth_mm, depth_mm)
                lines.append(f"; Pass {pass_num + 1}")
                lines.append(f"G1 Z-{z_depth:.3f} F{params.plunge_rate * GCodeService.INCHES_TO_MM:.1f}")

                # Cut to each subsequent point
                for point in points[1:]:
                    x_mm = point['x'] * GCodeService.INCHES_TO_MM
                    y_mm = point['y'] * GCodeService.INCHES_TO_MM

                    if point.get('line_type') == 'arc':
                        # Arc move
                        arc_cx = point.get('arc_center_x', point['x']) * GCodeService.INCHES_TO_MM
                        arc_cy = point.get('arc_center_y', point['y']) * GCodeService.INCHES_TO_MM
                        # Calculate I, J (relative to start point)
                        # This is simplified - actual implementation would need previous point
                        lines.append(f"G2 X{x_mm:.3f} Y{y_mm:.3f} I{arc_cx:.3f} J{arc_cy:.3f} F{params.feed_rate * GCodeService.INCHES_TO_MM:.1f}")
                    else:
                        # Straight move
                        lines.append(f"G1 X{x_mm:.3f} Y{y_mm:.3f} F{params.feed_rate * GCodeService.INCHES_TO_MM:.1f}")

                # Return to start if closed
                if cut.get('closed') and len(points) > 1:
                    lines.append(f"G1 X{start['x'] * GCodeService.INCHES_TO_MM:.3f} Y{start['y'] * GCodeService.INCHES_TO_MM:.3f}")

                # Retract between passes
                if pass_num < num_passes - 1:
                    lines.append(f"G0 Z{general.travel_height * GCodeService.INCHES_TO_MM:.3f}")
                    lines.append(f"G0 X{start['x'] * GCodeService.INCHES_TO_MM:.3f} Y{start['y'] * GCodeService.INCHES_TO_MM:.3f}")

            lines.append(f"G0 Z{safety_mm:.3f}")
            lines.append("")

        return lines

    @staticmethod
    def generate_preview_svg(project: Project) -> str:
        """Generate an SVG preview of the project operations."""
        from src.visualizer import WebVisualizer

        drill_points, circular_cuts, line_cuts = GCodeService.expand_operations(project.operations)

        # Get material dimensions for bounds
        material = project.material
        if material:
            if material.form == 'tube':
                width = material.outer_width
                height = material.outer_height
            else:
                width = 12  # Default sheet width
                height = 12  # Default sheet height
        else:
            width = 12
            height = 12

        # Check for tube void
        void_bounds = None
        if project.tube_void_skip and material and material.form == 'tube':
            void_bounds = (
                material.wall_thickness,
                material.wall_thickness,
                material.outer_width - material.wall_thickness,
                material.outer_height - material.wall_thickness
            )

        return WebVisualizer.generate_svg(
            drill_points=drill_points,
            circular_cuts=circular_cuts,
            line_cuts=line_cuts,
            width=width,
            height=height,
            void_bounds=void_bounds
        )

    @staticmethod
    def validate(project: Project) -> List[str]:
        """Validate a project and return list of errors."""
        errors = []
        machine = SettingsService.get_machine_settings()

        if not project.material_id:
            errors.append("No material selected")

        if project.project_type == 'drill' and not project.drill_bit_size:
            errors.append("No drill bit size selected")

        if project.project_type == 'cut' and not project.end_mill_size:
            errors.append("No end mill size selected")

        # Check bounds
        drill_points, circular_cuts, line_cuts = GCodeService.expand_operations(project.operations)

        for p in drill_points:
            if p.x < 0 or p.x > machine.max_x:
                errors.append(f"Drill point ({p.x}, {p.y}) exceeds X bounds (0-{machine.max_x})")
            if p.y < 0 or p.y > machine.max_y:
                errors.append(f"Drill point ({p.x}, {p.y}) exceeds Y bounds (0-{machine.max_y})")

        for cut in circular_cuts:
            radius = cut['diameter'] / 2
            if cut['center_x'] - radius < 0 or cut['center_x'] + radius > machine.max_x:
                errors.append(f"Circular cut at ({cut['center_x']}, {cut['center_y']}) exceeds X bounds")
            if cut['center_y'] - radius < 0 or cut['center_y'] + radius > machine.max_y:
                errors.append(f"Circular cut at ({cut['center_x']}, {cut['center_y']}) exceeds Y bounds")

        return errors
```

---

## Route Handlers

### Main Routes (web/routes/main.py)

```python
from flask import Blueprint, render_template
from web.services.project_service import ProjectService

main_bp = Blueprint('main', __name__)


@main_bp.route('/')
def index():
    """Home page - show all projects."""
    projects = ProjectService.get_all()
    return render_template('index.html', projects=projects)
```

---

### Project Routes (web/routes/projects.py)

```python
from flask import Blueprint, render_template, redirect, url_for, request, flash
import json
from web.services.project_service import ProjectService
from web.services.settings_service import SettingsService

projects_bp = Blueprint('projects', __name__)


@projects_bp.route('/new')
def new():
    """Show new project form."""
    materials = SettingsService.get_all_materials()
    return render_template('project/new.html', materials=materials)


@projects_bp.route('/create', methods=['POST'])
def create():
    """Create a new project."""
    data = {
        'name': request.form.get('name'),
        'project_type': request.form.get('project_type'),
        'material_id': request.form.get('material_id')
    }

    if not data['name']:
        flash('Project name is required', 'danger')
        return redirect(url_for('projects.new'))

    project = ProjectService.create(data)
    flash('Project created successfully', 'success')
    return redirect(url_for('projects.edit', project_id=project.id))


@projects_bp.route('/<project_id>')
def edit(project_id):
    """Edit project page - main editor interface."""
    project = ProjectService.get(project_id)
    if not project:
        flash('Project not found', 'danger')
        return redirect(url_for('main.index'))

    # Get data for template
    materials = SettingsService.get_all_materials()
    materials_dict = SettingsService.get_materials_dict()
    tools = SettingsService.get_all_tools()
    project_dict = ProjectService.get_as_dict(project_id)

    return render_template(
        'project/edit.html',
        project=project,
        materials=materials,
        tools=tools,
        # JSON data for JavaScript
        project_json=json.dumps(project_dict),
        materials_json=json.dumps(materials_dict)
    )


@projects_bp.route('/<project_id>/delete', methods=['POST'])
def delete(project_id):
    """Delete a project."""
    if ProjectService.delete(project_id):
        flash('Project deleted', 'success')
    else:
        flash('Failed to delete project', 'danger')
    return redirect(url_for('main.index'))


@projects_bp.route('/<project_id>/duplicate', methods=['POST'])
def duplicate(project_id):
    """Duplicate a project."""
    new_project = ProjectService.duplicate(project_id)
    if new_project:
        flash('Project duplicated', 'success')
        return redirect(url_for('projects.edit', project_id=new_project.id))
    else:
        flash('Failed to duplicate project', 'danger')
        return redirect(url_for('main.index'))
```

---

### Settings Routes (web/routes/settings.py)

```python
from flask import Blueprint, render_template, redirect, url_for, request, flash
from web.services.settings_service import SettingsService

settings_bp = Blueprint('settings', __name__)


@settings_bp.route('/')
def index():
    """Settings dashboard."""
    return render_template('settings/index.html')


# --- Materials ---

@settings_bp.route('/materials')
def materials():
    """Materials list."""
    materials = SettingsService.get_all_materials()
    return render_template('settings/materials.html', materials=materials)


@settings_bp.route('/materials/create', methods=['POST'])
def create_material():
    """Create a new material."""
    data = {
        'id': request.form.get('id'),
        'display_name': request.form.get('display_name'),
        'base_material': request.form.get('base_material'),
        'form': request.form.get('form'),
        'thickness': float(request.form.get('thickness')) if request.form.get('thickness') else None,
        'outer_width': float(request.form.get('outer_width')) if request.form.get('outer_width') else None,
        'outer_height': float(request.form.get('outer_height')) if request.form.get('outer_height') else None,
        'wall_thickness': float(request.form.get('wall_thickness')) if request.form.get('wall_thickness') else None,
        'gcode_standards': {}  # Will be set in edit page
    }

    try:
        SettingsService.create_material(data)
        flash('Material created successfully', 'success')
    except Exception as e:
        flash(f'Error creating material: {str(e)}', 'danger')

    return redirect(url_for('settings.materials'))


@settings_bp.route('/materials/<material_id>/edit')
def edit_material(material_id):
    """Edit material page."""
    material = SettingsService.get_material(material_id)
    if not material:
        flash('Material not found', 'danger')
        return redirect(url_for('settings.materials'))

    tools = SettingsService.get_all_tools()
    return render_template('settings/material_edit.html', material=material, tools=tools)


@settings_bp.route('/materials/<material_id>/update', methods=['POST'])
def update_material(material_id):
    """Update a material."""
    # Build gcode_standards from form
    gcode_standards = {'drill': {}, 'cut': {}}

    # Parse form data for each tool size
    for key in request.form:
        if key.startswith('drill_') or key.startswith('cut_'):
            parts = key.split('_')
            op_type = parts[0]  # 'drill' or 'cut'
            tool_size = parts[1]  # e.g., '0.125'
            param = '_'.join(parts[2:])  # e.g., 'spindle_speed'

            if tool_size not in gcode_standards[op_type]:
                gcode_standards[op_type][tool_size] = {}

            value = request.form.get(key)
            if value:
                gcode_standards[op_type][tool_size][param] = float(value)

    data = {
        'display_name': request.form.get('display_name'),
        'base_material': request.form.get('base_material'),
        'form': request.form.get('form'),
        'thickness': float(request.form.get('thickness')) if request.form.get('thickness') else None,
        'outer_width': float(request.form.get('outer_width')) if request.form.get('outer_width') else None,
        'outer_height': float(request.form.get('outer_height')) if request.form.get('outer_height') else None,
        'wall_thickness': float(request.form.get('wall_thickness')) if request.form.get('wall_thickness') else None,
        'gcode_standards': gcode_standards
    }

    SettingsService.update_material(material_id, data)
    flash('Material updated successfully', 'success')
    return redirect(url_for('settings.materials'))


@settings_bp.route('/materials/<material_id>/delete', methods=['POST'])
def delete_material(material_id):
    """Delete a material."""
    if SettingsService.delete_material(material_id):
        flash('Material deleted', 'success')
    else:
        flash('Cannot delete material - it may be in use by projects', 'danger')
    return redirect(url_for('settings.materials'))


# --- Machine Settings ---

@settings_bp.route('/machine')
def machine():
    """Machine settings page."""
    machine = SettingsService.get_machine_settings()
    return render_template('settings/machine.html', machine=machine)


@settings_bp.route('/machine/save', methods=['POST'])
def save_machine():
    """Save machine settings."""
    data = {
        'name': request.form.get('name'),
        'max_x': request.form.get('max_x'),
        'max_y': request.form.get('max_y'),
        'controller_type': request.form.get('controller_type'),
        'supports_loops': 'supports_loops' in request.form,
        'supports_canned_cycles': 'supports_canned_cycles' in request.form
    }

    SettingsService.update_machine_settings(data)
    flash('Machine settings saved', 'success')
    return redirect(url_for('settings.machine'))


# --- General Settings ---

@settings_bp.route('/general')
def general():
    """General settings page."""
    general = SettingsService.get_general_settings()
    return render_template('settings/general.html', general=general)


@settings_bp.route('/general/save', methods=['POST'])
def save_general():
    """Save general settings."""
    data = {
        'safety_height': request.form.get('safety_height'),
        'travel_height': request.form.get('travel_height'),
        'spindle_warmup_seconds': request.form.get('spindle_warmup_seconds')
    }

    SettingsService.update_general_settings(data)
    flash('General settings saved', 'success')
    return redirect(url_for('settings.general'))


# --- Tools ---

@settings_bp.route('/tools')
def tools():
    """Tools list page."""
    tools = SettingsService.get_all_tools()
    return render_template('settings/tools.html', tools=tools)


@settings_bp.route('/tools/create', methods=['POST'])
def create_tool():
    """Create a new tool."""
    data = {
        'tool_type': request.form.get('tool_type'),
        'size': request.form.get('size'),
        'description': request.form.get('description')
    }

    SettingsService.create_tool(data)
    flash('Tool added successfully', 'success')
    return redirect(url_for('settings.tools'))


@settings_bp.route('/tools/<int:tool_id>/delete', methods=['POST'])
def delete_tool(tool_id):
    """Delete a tool."""
    if SettingsService.delete_tool(tool_id):
        flash('Tool deleted', 'success')
    else:
        flash('Failed to delete tool', 'danger')
    return redirect(url_for('settings.tools'))
```

---

### API Routes - Updated (web/routes/api.py)

```python
from flask import Blueprint, request, jsonify, send_file
from web.services.project_service import ProjectService
from web.services.gcode_service import GCodeService
from web.models import Project
import io

api_bp = Blueprint('api', __name__)


@api_bp.route('/projects/<project_id>/save', methods=['POST'])
def save_project(project_id):
    """Save project data from the editor."""
    data = request.get_json()
    project = ProjectService.save(project_id, data)

    if project:
        return jsonify({
            'status': 'saved',
            'modified_at': project.modified_at.isoformat()
        })
    else:
        return jsonify({'status': 'error', 'message': 'Project not found'}), 404


@api_bp.route('/projects/<project_id>/preview', methods=['POST'])
def preview(project_id):
    """Generate SVG preview for project."""
    # Use posted data if available (for preview of unsaved changes)
    data = request.get_json()

    project = ProjectService.get(project_id)
    if not project:
        return jsonify({'error': 'Project not found'}), 404

    # Temporarily apply posted operations for preview
    if data and 'operations' in data:
        project.operations = data['operations']

    try:
        svg_data = GCodeService.generate_preview_svg(project)
        return jsonify({'svg': svg_data})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@api_bp.route('/projects/<project_id>/download/<gcode_type>')
def download_gcode(project_id, gcode_type):
    """Download generated G-code file."""
    project = ProjectService.get(project_id)
    if not project:
        return jsonify({'error': 'Project not found'}), 404

    if gcode_type not in ('drill', 'cut'):
        return jsonify({'error': 'Invalid G-code type'}), 400

    try:
        gcode = GCodeService.generate(project, gcode_type)

        buffer = io.BytesIO(gcode.encode('utf-8'))
        buffer.seek(0)

        # Clean filename
        safe_name = "".join(c for c in project.name if c.isalnum() or c in (' ', '-', '_')).strip()
        filename = f"{safe_name}_{gcode_type}.nc"

        return send_file(
            buffer,
            mimetype='text/plain',
            as_attachment=True,
            download_name=filename
        )
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@api_bp.route('/projects/<project_id>/validate', methods=['POST'])
def validate(project_id):
    """Validate project configuration."""
    project = ProjectService.get(project_id)
    if not project:
        return jsonify({'error': 'Project not found'}), 404

    errors = GCodeService.validate(project)
    return jsonify({
        'valid': len(errors) == 0,
        'errors': errors
    })


@api_bp.route('/materials/<material_id>/gcode-params')
def get_material_params(material_id):
    """Get G-code parameters for a material."""
    from web.services.settings_service import SettingsService

    material = SettingsService.get_material(material_id)
    if not material:
        return jsonify({'error': 'Material not found'}), 404

    return jsonify({
        'id': material.id,
        'gcode_standards': material.gcode_standards
    })
```

---

## Visualizer for Web (src/visualizer.py modifications)

Add a `WebVisualizer` class to generate SVG output:

```python
# Add to existing src/visualizer.py or create new class

class WebVisualizer:
    """Generate SVG visualizations for web display."""

    @staticmethod
    def generate_svg(drill_points, circular_cuts, line_cuts, width, height, void_bounds=None) -> str:
        """
        Generate an SVG string for the toolpath preview.

        Args:
            drill_points: List of ExpandedPoint objects
            circular_cuts: List of dicts with center_x, center_y, diameter
            line_cuts: List of dicts with points list
            width: Material width in inches
            height: Material height in inches
            void_bounds: Optional tuple (x_min, y_min, x_max, y_max) for tube void
        """
        # SVG dimensions (pixels) - maintain aspect ratio
        scale = 50  # pixels per inch
        svg_width = width * scale
        svg_height = height * scale
        padding = 20

        total_width = svg_width + 2 * padding
        total_height = svg_height + 2 * padding

        # Start SVG
        svg_parts = [
            f'<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {total_width} {total_height}" '
            f'width="100%" height="100%" style="max-height: 500px;">',
            '<defs>',
            '  <pattern id="grid" width="50" height="50" patternUnits="userSpaceOnUse">',
            '    <path d="M 50 0 L 0 0 0 50" fill="none" stroke="#e0e0e0" stroke-width="0.5"/>',
            '  </pattern>',
            '</defs>',
            # Background
            f'<rect x="0" y="0" width="{total_width}" height="{total_height}" fill="#f8f9fa"/>',
            # Material outline
            f'<rect x="{padding}" y="{padding}" width="{svg_width}" height="{svg_height}" '
            f'fill="url(#grid)" stroke="#333" stroke-width="2"/>',
        ]

        # Draw tube void if present
        if void_bounds:
            x_min, y_min, x_max, y_max = void_bounds
            void_x = padding + x_min * scale
            void_y = padding + (height - y_max) * scale  # Flip Y
            void_w = (x_max - x_min) * scale
            void_h = (y_max - y_min) * scale
            svg_parts.append(
                f'<rect x="{void_x}" y="{void_y}" width="{void_w}" height="{void_h}" '
                f'fill="#ffcccc" stroke="#cc0000" stroke-width="1" stroke-dasharray="5,5"/>'
            )

        # Helper to convert coordinates (flip Y axis)
        def to_svg(x, y):
            return padding + x * scale, padding + (height - y) * scale

        # Draw drill points
        for point in drill_points:
            sx, sy = to_svg(point.x, point.y)
            svg_parts.append(
                f'<circle cx="{sx}" cy="{sy}" r="4" fill="#0066cc" stroke="#003366" stroke-width="1"/>'
            )
            svg_parts.append(
                f'<circle cx="{sx}" cy="{sy}" r="8" fill="none" stroke="#0066cc" stroke-width="1" opacity="0.5"/>'
            )

        # Draw circular cuts
        for cut in circular_cuts:
            sx, sy = to_svg(cut['center_x'], cut['center_y'])
            radius = (cut['diameter'] / 2) * scale
            svg_parts.append(
                f'<circle cx="{sx}" cy="{sy}" r="{radius}" fill="none" stroke="#cc6600" stroke-width="2"/>'
            )
            # Center point marker
            svg_parts.append(
                f'<circle cx="{sx}" cy="{sy}" r="2" fill="#cc6600"/>'
            )

        # Draw line cuts
        for cut in line_cuts:
            points = cut.get('points', [])
            if len(points) < 2:
                continue

            # Build path
            path_parts = []
            for i, point in enumerate(points):
                sx, sy = to_svg(point['x'], point['y'])
                if i == 0:
                    path_parts.append(f'M {sx} {sy}')
                elif point.get('line_type') == 'arc':
                    # Simplified arc - just use line for preview
                    # Full arc rendering would require more complex SVG arc calculation
                    path_parts.append(f'L {sx} {sy}')
                else:
                    path_parts.append(f'L {sx} {sy}')

            # Close path if specified
            if cut.get('closed'):
                path_parts.append('Z')

            svg_parts.append(
                f'<path d="{" ".join(path_parts)}" fill="none" stroke="#009933" stroke-width="2"/>'
            )

            # Draw points on the path
            for point in points:
                sx, sy = to_svg(point['x'], point['y'])
                svg_parts.append(
                    f'<circle cx="{sx}" cy="{sy}" r="3" fill="#009933"/>'
                )

        # Add axis labels
        svg_parts.append(
            f'<text x="{padding + svg_width/2}" y="{total_height - 5}" '
            f'text-anchor="middle" font-size="12" fill="#666">X (inches)</text>'
        )
        svg_parts.append(
            f'<text x="10" y="{padding + svg_height/2}" '
            f'text-anchor="middle" font-size="12" fill="#666" '
            f'transform="rotate(-90, 10, {padding + svg_height/2})">Y (inches)</text>'
        )

        # Add scale markers
        for i in range(int(width) + 1):
            x = padding + i * scale
            svg_parts.append(
                f'<line x1="{x}" y1="{total_height - padding}" x2="{x}" y2="{total_height - padding + 5}" '
                f'stroke="#666" stroke-width="1"/>'
            )
            if i % 2 == 0:  # Label every 2 inches
                svg_parts.append(
                    f'<text x="{x}" y="{total_height - padding + 15}" '
                    f'text-anchor="middle" font-size="10" fill="#666">{i}</text>'
                )

        for i in range(int(height) + 1):
            y = padding + (height - i) * scale
            svg_parts.append(
                f'<line x1="{padding - 5}" y1="{y}" x2="{padding}" y2="{y}" '
                f'stroke="#666" stroke-width="1"/>'
            )
            if i % 2 == 0:
                svg_parts.append(
                    f'<text x="{padding - 10}" y="{y + 4}" '
                    f'text-anchor="end" font-size="10" fill="#666">{i}</text>'
                )

        # Legend
        legend_y = 15
        svg_parts.append(f'<circle cx="{total_width - 100}" cy="{legend_y}" r="4" fill="#0066cc"/>')
        svg_parts.append(f'<text x="{total_width - 90}" y="{legend_y + 4}" font-size="10" fill="#333">Drill</text>')

        svg_parts.append(f'<circle cx="{total_width - 100}" cy="{legend_y + 15}" r="6" fill="none" stroke="#cc6600" stroke-width="2"/>')
        svg_parts.append(f'<text x="{total_width - 90}" y="{legend_y + 19}" font-size="10" fill="#333">Circle</text>')

        svg_parts.append(f'<line x1="{total_width - 106}" y1="{legend_y + 30}" x2="{total_width - 94}" y2="{legend_y + 30}" stroke="#009933" stroke-width="2"/>')
        svg_parts.append(f'<text x="{total_width - 90}" y="{legend_y + 34}" font-size="10" fill="#333">Line</text>')

        svg_parts.append('</svg>')

        return '\n'.join(svg_parts)
```

---

## Complete G-Code Generator (src/gcode_generator.py)

Here's the complete refactored G-code generator that works with both the CLI and web interfaces:

```python
"""
G-Code Generator for CNC machining operations.
Supports both CLI (interactive) and web (settings-based) modes.
"""

import math
from dataclasses import dataclass
from typing import List, Dict, Any, Optional


@dataclass
class MachiningParams:
    """Parameters for a machining operation."""
    spindle_speed: int
    feed_rate: float      # inches/min
    plunge_rate: float    # inches/min
    material_depth: float  # inches
    pecking_depth: float = 0.05  # For drilling
    pass_depth: float = 0.02     # For cutting
    safety_height: float = 0.5
    travel_height: float = 0.2


class GCodeGenerator:
    """Generate G-code for CNC machining operations."""

    INCHES_TO_MM = 25.4

    def __init__(self, params: MachiningParams, use_loops: bool = True, use_canned_cycles: bool = True):
        """
        Initialize the generator.

        Args:
            params: Machining parameters
            use_loops: Whether to use WHILE loops for multi-pass (Mach3/4 feature)
            use_canned_cycles: Whether to use G83 peck drilling
        """
        self.params = params
        self.use_loops = use_loops
        self.use_canned_cycles = use_canned_cycles
        self.var_counter = 100  # For loop variables
        self.lines: List[str] = []

    def generate_header(self, project_name: str) -> List[str]:
        """Generate G-code header with setup commands."""
        safety_mm = self.params.safety_height * self.INCHES_TO_MM

        return [
            f"; G-Code for: {project_name}",
            f"; Generated by FIRST Robotics G-Code Generator",
            f"; Spindle Speed: {self.params.spindle_speed} RPM",
            f"; Feed Rate: {self.params.feed_rate} in/min",
            "",
            "G21 ; Units: millimeters",
            "G90 ; Absolute positioning",
            "G17 ; XY plane selection",
            f"G0 Z{safety_mm:.3f} ; Move to safety height",
            f"M3 S{self.params.spindle_speed} ; Start spindle",
            "G4 P2 ; Dwell 2 seconds for spindle warmup",
            ""
        ]

    def generate_footer(self) -> List[str]:
        """Generate G-code footer with cleanup commands."""
        return [
            "",
            "; --- End of operations ---",
            "M5 ; Stop spindle",
            "G0 Z12.7 ; Retract to safe height (0.5 inch)",
            "G0 X0 Y0 ; Return to origin",
            "M30 ; Program end and rewind"
        ]

    def generate_drill_operations(self, points: List[Any]) -> List[str]:
        """
        Generate drilling operations.

        Args:
            points: List of objects with x, y attributes (inches)
        """
        if not points:
            return ["; No drill points defined"]

        if self.use_canned_cycles:
            return self._generate_peck_drill_cycle(points)
        else:
            return self._generate_manual_peck_drill(points)

    def _generate_peck_drill_cycle(self, points: List[Any]) -> List[str]:
        """Generate G83 peck drilling canned cycle."""
        lines = ["; --- Drill Operations (G83 Peck Drilling) ---"]

        depth_mm = self.params.material_depth * self.INCHES_TO_MM
        peck_mm = self.params.pecking_depth * self.INCHES_TO_MM
        retract_mm = self.params.travel_height * self.INCHES_TO_MM
        feed_mm = self.params.plunge_rate * self.INCHES_TO_MM

        # First point establishes the cycle
        first = points[0]
        x_mm = first.x * self.INCHES_TO_MM
        y_mm = first.y * self.INCHES_TO_MM

        lines.append(
            f"G83 X{x_mm:.3f} Y{y_mm:.3f} Z-{depth_mm:.3f} "
            f"R{retract_mm:.3f} Q{peck_mm:.3f} F{feed_mm:.1f}"
        )

        # Subsequent points only need X, Y
        for point in points[1:]:
            x_mm = point.x * self.INCHES_TO_MM
            y_mm = point.y * self.INCHES_TO_MM
            lines.append(f"X{x_mm:.3f} Y{y_mm:.3f}")

        lines.append("G80 ; Cancel canned cycle")
        lines.append(f"G0 Z{self.params.safety_height * self.INCHES_TO_MM:.3f} ; Safety height")

        return lines

    def _generate_manual_peck_drill(self, points: List[Any]) -> List[str]:
        """Generate manual peck drilling for controllers without G83."""
        lines = ["; --- Drill Operations (Manual Peck) ---"]

        depth_mm = self.params.material_depth * self.INCHES_TO_MM
        peck_mm = self.params.pecking_depth * self.INCHES_TO_MM
        safety_mm = self.params.safety_height * self.INCHES_TO_MM
        travel_mm = self.params.travel_height * self.INCHES_TO_MM
        feed_mm = self.params.plunge_rate * self.INCHES_TO_MM

        for i, point in enumerate(points):
            x_mm = point.x * self.INCHES_TO_MM
            y_mm = point.y * self.INCHES_TO_MM

            lines.append(f"; Hole {i + 1} at ({point.x:.3f}, {point.y:.3f}) inches")
            lines.append(f"G0 X{x_mm:.3f} Y{y_mm:.3f}")
            lines.append(f"G0 Z{travel_mm:.3f}")

            # Peck drilling loop
            current_depth = 0.0
            peck_num = 1
            while current_depth < depth_mm:
                current_depth = min(current_depth + peck_mm, depth_mm)
                lines.append(f"G1 Z-{current_depth:.3f} F{feed_mm:.1f} ; Peck {peck_num}")
                lines.append(f"G0 Z{travel_mm:.3f} ; Retract")
                peck_num += 1

            lines.append(f"G0 Z{safety_mm:.3f}")
            lines.append("")

        return lines

    def generate_circular_cut(self, center_x: float, center_y: float,
                              diameter: float, tool_diameter: float) -> List[str]:
        """
        Generate circular pocket/cutout operation.

        Args:
            center_x, center_y: Center coordinates (inches)
            diameter: Circle diameter (inches)
            tool_diameter: End mill diameter (inches)
        """
        lines = [f"; --- Circular Cut: {diameter}\" dia at ({center_x}, {center_y}) ---"]

        # Calculate cut radius (compensate for tool)
        cut_radius = (diameter / 2) - (tool_diameter / 2)
        if cut_radius <= 0:
            lines.append(f"; WARNING: Tool too large for this circle")
            return lines

        # Convert to mm
        cx_mm = center_x * self.INCHES_TO_MM
        cy_mm = center_y * self.INCHES_TO_MM
        radius_mm = cut_radius * self.INCHES_TO_MM
        depth_mm = self.params.material_depth * self.INCHES_TO_MM
        pass_mm = self.params.pass_depth * self.INCHES_TO_MM
        safety_mm = self.params.safety_height * self.INCHES_TO_MM
        travel_mm = self.params.travel_height * self.INCHES_TO_MM
        feed_mm = self.params.feed_rate * self.INCHES_TO_MM
        plunge_mm = self.params.plunge_rate * self.INCHES_TO_MM

        num_passes = math.ceil(depth_mm / pass_mm)

        # Start position (3 o'clock)
        start_x = cx_mm + radius_mm
        start_y = cy_mm

        lines.append(f"G0 X{start_x:.3f} Y{start_y:.3f}")
        lines.append(f"G0 Z{travel_mm:.3f}")

        if self.use_loops and num_passes > 1:
            # Use WHILE loop for efficiency
            v = self.var_counter
            self.var_counter += 10

            lines.append(f"#{v} = 0 ; Pass counter")
            lines.append(f"#{v+1} = {num_passes} ; Total passes")
            lines.append(f"#{v+2} = {pass_mm:.3f} ; Depth per pass")
            lines.append(f"#{v+3} = {depth_mm:.3f} ; Total depth")
            lines.append(f"WHILE [#{v} LT #{v+1}] DO1")
            lines.append(f"  #{v+4} = [[#{v} + 1] * #{v+2}]")
            lines.append(f"  IF [#{v+4} GT #{v+3}] THEN #{v+4} = #{v+3}")
            lines.append(f"  G1 Z-#{v+4} F{plunge_mm:.1f}")
            lines.append(f"  G2 I-{radius_mm:.3f} J0 F{feed_mm:.1f} ; Full circle")
            lines.append(f"  #{v} = [#{v} + 1]")
            lines.append(f"END1")
        else:
            # Explicit passes
            for pass_num in range(num_passes):
                z_depth = min((pass_num + 1) * pass_mm, depth_mm)
                lines.append(f"; Pass {pass_num + 1} of {num_passes}")
                lines.append(f"G1 Z-{z_depth:.3f} F{plunge_mm:.1f}")
                lines.append(f"G2 I-{radius_mm:.3f} J0 F{feed_mm:.1f} ; Full circle CW")

        lines.append(f"G0 Z{safety_mm:.3f}")
        lines.append("")

        return lines

    def generate_line_cut(self, points: List[Dict[str, Any]], closed: bool = False) -> List[str]:
        """
        Generate line cutting operation.

        Args:
            points: List of point dicts with x, y, line_type keys
            closed: Whether to close the path back to start
        """
        if not points or len(points) < 2:
            return ["; No line points defined"]

        lines = [f"; --- Line Cut: {len(points)} points ---"]

        depth_mm = self.params.material_depth * self.INCHES_TO_MM
        pass_mm = self.params.pass_depth * self.INCHES_TO_MM
        safety_mm = self.params.safety_height * self.INCHES_TO_MM
        travel_mm = self.params.travel_height * self.INCHES_TO_MM
        feed_mm = self.params.feed_rate * self.INCHES_TO_MM
        plunge_mm = self.params.plunge_rate * self.INCHES_TO_MM

        num_passes = math.ceil(depth_mm / pass_mm)

        # Move to start
        start = points[0]
        start_x_mm = start['x'] * self.INCHES_TO_MM
        start_y_mm = start['y'] * self.INCHES_TO_MM

        lines.append(f"G0 X{start_x_mm:.3f} Y{start_y_mm:.3f}")
        lines.append(f"G0 Z{travel_mm:.3f}")

        # Cut each pass
        for pass_num in range(num_passes):
            z_depth = min((pass_num + 1) * pass_mm, depth_mm)
            lines.append(f"; Pass {pass_num + 1} of {num_passes}")
            lines.append(f"G1 Z-{z_depth:.3f} F{plunge_mm:.1f}")

            # Cut to each point
            prev_point = start
            for point in points[1:]:
                x_mm = point['x'] * self.INCHES_TO_MM
                y_mm = point['y'] * self.INCHES_TO_MM

                if point.get('line_type') == 'arc' and 'arc_center_x' in point:
                    # Arc interpolation
                    # Calculate I, J relative to current position
                    arc_cx = point['arc_center_x'] * self.INCHES_TO_MM
                    arc_cy = point['arc_center_y'] * self.INCHES_TO_MM
                    prev_x = prev_point['x'] * self.INCHES_TO_MM
                    prev_y = prev_point['y'] * self.INCHES_TO_MM
                    i = arc_cx - prev_x
                    j = arc_cy - prev_y
                    lines.append(f"G2 X{x_mm:.3f} Y{y_mm:.3f} I{i:.3f} J{j:.3f} F{feed_mm:.1f}")
                else:
                    # Linear interpolation
                    lines.append(f"G1 X{x_mm:.3f} Y{y_mm:.3f} F{feed_mm:.1f}")

                prev_point = point

            # Close path if requested
            if closed:
                lines.append(f"G1 X{start_x_mm:.3f} Y{start_y_mm:.3f} F{feed_mm:.1f}")

            # Retract and return for next pass
            if pass_num < num_passes - 1:
                lines.append(f"G0 Z{travel_mm:.3f}")
                lines.append(f"G0 X{start_x_mm:.3f} Y{start_y_mm:.3f}")

        lines.append(f"G0 Z{safety_mm:.3f}")
        lines.append("")

        return lines
```
