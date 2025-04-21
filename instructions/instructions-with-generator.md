You're absolutely right—while the previous implementation provided a functional Django-based editor for the Software Contract Schema, it didn't fully leverage the LinkML schema's metadata to automate the generation of Django models, serializers, views, and frontend components. To address this, we need a **meta-driven generator** that programmatically processes the LinkML schema to produce the necessary Django code (models, serializers, views, URLs) and a frontend editor, reducing manual maintenance. The generator will iterate over the LinkML schema's classes and slots to create these artifacts dynamically, ensuring that updates to the schema can be reflected by re-running the generator.

### Approach
We'll create a Python script that:
1. **Parses the LinkML Schema**: Uses the `linkml-runtime` library to load and interpret the schema.
2. **Generates Django Artifacts**:
    - **Models**: Creates Django model classes for each LinkML class, mapping slots to fields (e.g., `CharField`, `TextField`, `ForeignKey`).
    - **Serializers**: Generates Django REST Framework serializers for each model, handling nested relationships.
    - **Views**: Produces `ModelViewSet` classes for each model to provide REST API endpoints.
    - **URLs**: Creates URL patterns for the API endpoints.
3. **Generates Frontend**: Produces an `index.html` with a dynamic form-based editor that reflects the schema's structure, using JavaScript to interact with the API and export YAML.
4. **Automates Maintenance**: The generator can be re-run whenever the schema changes, producing updated code without manual intervention.

The generator will be driven by the LinkML schema's metadata (classes, slots, ranges, multivalued properties, etc.), ensuring that the output is tightly coupled to the schema. This approach minimizes maintenance by automating code generation and aligns with the requirement to use the schema's structure to drive the application.

### Implementation Details
- **LinkML Parsing**: Use `linkml_runtime` to load the YAML schema and access its classes and slots.
- **Model Generation**:
    - Map LinkML classes to Django models.
    - Map slots to model fields based on their `range` (e.g., `string` → `CharField`, `date` → `DateField`).
    - Handle relationships: `inlined` and `multivalued` slots become `ForeignKey` or `ManyToManyField`.
- **Serializer Generation**: Create serializers that mirror models, with nested serializers for relationships.
- **View Generation**: Generate `ModelViewSet` classes for each model, using the corresponding serializer.
- **URL Generation**: Create a router to register viewsets and include them in URL patterns.
- **Frontend Generation**: Produce a dynamic HTML form that iterates over the schema's classes and slots, with JavaScript to handle CRUD operations and YAML export.
- **Output**: The generator will produce multiple files (`models.py`, `serializers.py`, `views.py`, `urls.py`, `index.html`) as part of a single artifact, with instructions for placement.

### Artifact
The artifact below is a Python script (`generate_django_app.py`) that serves as the meta-driven generator. It takes the LinkML schema (included inline) and generates the Django project files and frontend. The generated files are written to a temporary directory and included in the artifact's output for clarity. The script uses the `linkml_runtime` to parse the schema and `jinja2` for templating (to generate clean, readable code).

```python
import os
import yaml
from linkml_runtime.loaders import yaml_loader
from linkml_runtime.linkml_model import SchemaDefinition
from jinja2 import Template
import uuid

# Inline LinkML Schema (from previous artifact)
LINKML_SCHEMA = """
# LinkML schema for a Software System Contract
schema:
  id: http://example.org/software-contract
  name: SoftwareContractSchema
  description: A schema for defining software system contracts with conditions, behaviors, dependencies, and metadata.
  version: 1.0.0
  prefixes:
    linkml: https://w3id.org/linkml/
    schema: http://schema.org/
  imports:
    - linkml:types
  default_prefix: software_contract

classes:
  SoftwareContract:
    description: A contract defining the expectations and behaviors of a software system.
    slots:
      - contract_id
      - name
      - description
      - preconditions
      - postconditions
      - invariants
      - behavior_description
      - calling_systems
      - dependent_contracts
      - metadata

  Condition:
    description: A condition that must be satisfied (e.g., precondition, postcondition, invariant).
    slots:
      - condition_id
      - description
      - expression
      - evaluation_context

  BehaviorDescription:
    description: A description of the expected behavior of the system under the contract.
    slots:
      - behavior_id
      - description
      - input_parameters
      - output_parameters
      - sequence_diagram_ref

  System:
    description: A system that interacts with the contract (e.g., caller or dependency).
    slots:
      - system_id
      - name
      - description
      - owner
      - contact_info

  Metadata:
    description: Metadata about the contract, including authorship and reviewer comments.
    slots:
      - authors
      - creation_date
      - last_modified
      - version
      - reviewer_comments

  Author:
    description: An individual or entity that authored the contract.
    slots:
      - author_id
      - name
      - email
      - organization

  ReviewerComment:
    description: A comment provided by a reviewer of the contract.
    slots:
      - comment_id
      - reviewer_name
      - comment_text
      - comment_date

slots:
  contract_id:
    identifier: true
    description: Unique identifier for the contract.
    range: string

  name:
    description: Human-readable name of the contract or system.
    range: string
    required: true

  description:
    description: Detailed description of the contract, condition, behavior, or system.
    range: string

  preconditions:
    description: Conditions that must be true before the contract is executed.
    range: Condition
    multivalued: true
    inlined_as_list: true

  postconditions:
    description: Conditions that must be true after the contract is executed.
    range: Condition
    multivalued: true
    inlined_as_list: true

  invariants:
    description: Conditions that must remain true throughout the contract's execution.
    range: Condition
    multivalued: true
    inlined_as_list: true

  behavior_description:
    description: Description of the expected behavior of the system.
    range: BehaviorDescription
    inlined: true

  calling_systems:
    description: Systems that invoke this contract.
    range: System
    multivalued: true
    inlined_as_list: true

  dependent_contracts:
    description: Other contracts that this contract depends on to satisfy its conditions or behaviors.
    range: SoftwareContract
    multivalued: true
    inlined_as_list: true

  metadata:
    description: Metadata about the contract, including authors and reviewer comments.
    range: Metadata
    inlined: true
    required: true

  condition_id:
    identifier: true
    description: Unique identifier for a condition.
    range: string

  expression:
    description: Formal or informal expression of the condition (e.g., logical statement or code snippet).
    range: string

  evaluation_context:
    description: Context or environment in which the condition is evaluated (e.g., system state, variables).
    range: string

  behavior_id:
    identifier: true
    description: Unique identifier for a behavior description.
    range: string

  input_parameters:
    description: Parameters required as input for the behavior.
    range: string
    multivalued: true

  output_parameters:
    description: Parameters produced as output by the behavior.
    range: string
    multivalued: true

  sequence_diagram_ref:
    description: Reference to a sequence diagram or other documentation describing the behavior.
    range: string

  system_id:
    identifier: true
    description: Unique identifier for a system.
    range: string

  owner:
    description: Entity responsible for the system.
    range: string

  contact_info:
    description: Contact information for the system's owner or administrator.
    range: string

  authors:
    description: List of authors who created the contract.
    range: Author
    multivalued: true
    inlined_as_list: true
    required: true

  creation_date:
    description: Date the contract was created.
    range: date

  last_modified:
    description: Date the contract was last modified.
    range: date

  version:
    description: Version of the contract.
    range: string
    required: true

  reviewer_comments:
    description: Comments provided by reviewers of the contract.
    range: ReviewerComment
    multivalued: true
    inlined_as_list: true

  author_id:
    identifier: true
    description: Unique identifier for an author.
    range: string

  email:
    description: Email address of the author.
    range: string

  organization:
    description: Organization the author is affiliated with.
    range: string

  comment_id:
    identifier: true
    description: Unique identifier for a reviewer comment.
    range: string

  reviewer_name:
    description: Name of the reviewer who provided the comment.
    range: string
    required: true

  comment_text:
    description: Text of the reviewer's comment.
    range: string
    required: true

  comment_date:
    description: Date the comment was made.
    range: date
"""

# Templates for generating Django files
MODELS_TEMPLATE = Template("""
from django.db import models

{% for class_def in classes %}
class {{ class_def.name }}(models.Model):
    {% for slot in class_def.slots %}
    {{ slot.name }} = {% if slot.range in classes %}models.{% if slot.multivalued %}ManyToManyField{% else %}{% if slot.inlined %}OneToOneField{% else %}ForeignKey{% endif %}{% endif %}('{{ slot.range }}', {% if slot.identifier %}primary_key=True{% else %}{% if not slot.required %}blank=True{% endif %}{% if slot.multivalued or slot.inlined %}, related_name='{{ slot.name }}_{{ class_def.name.lower }}s'{% endif %}{% if slot.inlined %}, on_delete=models.CASCADE{% endif %}{% if slot.inlined and not slot.multivalued %}, null=True{% endif %}{% endif %})
    {% else %}
    {{ slot.name }} = models.{% if slot.range == 'string' and slot.identifier %}CharField(max_length=36, primary_key=True){% elif slot.range == 'string' and slot.multivalued %}JSONField(default=list){% elif slot.range == 'string' %}CharField(max_length=255{% if not slot.required %}, blank=True{% endif %}){% elif slot.range == 'date' %}DateField{% if not slot.required %}(blank=True, null=True){% else %}(){% endif %}{% elif slot.range == 'email' %}EmailField{% if not slot.required %}(blank=True){% endif %}{% else %}TextField{% if not slot.required %}(blank=True){% endif %}{% endif %}
    {% endif %}
    {% endfor %}
    def __str__(self):
        return self.{% for slot in class_def.slots %}{% if slot.identifier %}{{ slot.name }}{% elif slot.name == 'name' %}{{ slot.name }}{% endif %}{% endfor %}
{% endfor %}
""")

SERIALIZERS_TEMPLATE = Template("""
from rest_framework import serializers
from .models import {% for class_def in classes %}{{ class_def.name }}{% if not loop.last %}, {% endif %}{% endfor %}

{% for class_def in classes %}
class {{ class_def.name }}Serializer(serializers.ModelSerializer):
    {% for slot in class_def.slots %}
    {% if slot.range in classes %}
    {{ slot.name }} = {{ slot.range }}Serializer(many={{ slot.multivalued|lower }}{% if not slot.multivalued and not slot.inlined %}, read_only=True{% endif %})
    {% endif %}
    {% endfor %}
    class Meta:
        model = {{ class_def.name }}
        fields = [{% for slot in class_def.slots %}'{{ slot.name }}'{% if not loop.last %}, {% endif %}{% endfor %}]

    def create(self, validated_data):
        {% for slot in class_def.slots %}
        {% if slot.range in classes %}
        {{ slot.name }}_data = validated_data.pop('{{ slot.name }}'{% if not slot.multivalued %}, None{% endif %})
        {% endif %}
        {% endfor %}
        instance = {{ class_def.name }}.objects.create(**validated_data)
        {% for slot in class_def.slots %}
        {% if slot.range in classes %}
        {% if slot.multivalued %}
        for item_data in {{ slot.name }}_data:
            item, _ = {{ slot.range }}.objects.get_or_create(**item_data)
            instance.{{ slot.name }}.add(item)
        {% elif slot.inlined %}
        if {{ slot.name }}_data:
            {{ slot.name }}_instance = {{ slot.range }}Serializer().create({{ slot.name }}_data)
            instance.{{ slot.name }} = {{ slot.name }}_instance
        {% endif %}
        {% endif %}
        {% endfor %}
        instance.save()
        return instance

    def update(self, instance, validated_data):
        {% for slot in class_def.slots %}
        {% if slot.range in classes %}
        {{ slot.name }}_data = validated_data.pop('{{ slot.name }}'{% if not slot.multivalued %}, None{% endif %})
        {% endif %}
        {% endfor %}
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        {% for slot in class_def.slots %}
        {% if slot.range in classes %}
        {% if slot.multivalued %}
        instance.{{ slot.name }}.clear()
        for item_data in {{ slot.name }}_data or []:
            item, _ = {{ slot.range }}.objects.get_or_create(**item_data)
            instance.{{ slot.name }}.add(item)
        {% elif slot.inlined %}
        if {{ slot.name }}_data:
            {{ slot.range }}Serializer().update(instance.{{ slot.name }} or {{ slot.range }}.objects.create(), {{ slot.name }}_data)
        {% endif %}
        {% endif %}
        {% endfor %}
        instance.save()
        return instance
{% endfor %}
""")

VIEWS_TEMPLATE = Template("""
from rest_framework import viewsets
from .models import {% for class_def in classes %}{{ class_def.name }}{% if not loop.last %}, {% endif %}{% endfor %}
from .serializers import {% for class_def in classes %}{{ class_def.name }}Serializer{% if not loop.last %}, {% endif %}{% endfor %}

{% for class_def in classes %}
class {{ class_def.name }}ViewSet(viewsets.ModelViewSet):
    queryset = {{ class_def.name }}.objects.all()
    serializer_class = {{ class_def.name }}Serializer
{% endfor %}
""")

URLS_TEMPLATE = Template("""
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import {% for class_def in classes %}{{ class_def.name }}ViewSet{% if not loop.last %}, {% endif %}{% endfor %}

router = DefaultRouter()
{% for class_def in classes %}
router.register(r'{{ class_def.name.lower }}s', {{ class_def.name }}ViewSet)
{% endfor %}

urlpatterns = [
    path('', include(router.urls)),
]
""")

INDEX_HTML_TEMPLATE = Template("""
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Software Contract Editor</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/js-yaml@4.1.0/dist/js-yaml.min.js"></script>
    <style>
        body { padding: 20px; }
        .section { margin-bottom: 20px; }
        .nested { margin-left: 20px; border-left: 2px solid #ddd; padding-left: 10px; }
        .form-group { margin-bottom: 15px; }
        .error { color: red; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Software Contract Editor</h1>
        <div id="contract-form">
            {% for class_def in classes %}
            {% if class_def.name == 'SoftwareContract' %}
            <div class="section">
                <h2>{{ class_def.name }} Details</h2>
                {% for slot in class_def.slots %}
                {% if slot.range not in classes %}
                <div class="form-group">
                    <label for="{{ slot.name }}">{{ slot.name | replace('_', ' ') | title }}</label>
                    <{% if slot.range == 'date' %}input type="date"{% else %}input type="text"{% endif %} id="{{ slot.name }}" class="form-control" {% if slot.required %}required{% endif %}>
                </div>
                {% endif %}
                {% endfor %}
                {% for slot in class_def.slots %}
                {% if slot.range in classes %}
                <div class="section">
                    <h2>{{ slot.name | replace('_', ' ') | title }}</h2>
                    <div id="{{ slot.name }}" class="nested"></div>
                    <button class="btn btn-secondary" onclick="add{{ slot.range }}('{{ slot.name }}')">Add {{ slot.range }}</button>
                </div>
                {% endif %}
                {% endfor %}
            </div>
            {% endif %}
            {% endfor %}
            <div class="form-group">
                <button class="btn btn-primary" onclick="saveContract()">Save Contract</button>
                <button class="btn btn-success" onclick="exportYAML()">Export as YAML</button>
            </div>
            <div id="error" class="error"></div>
        </div>
    </div>

    <script>
        let contractId = null;

        function generateUUID() {
            return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
                const r = Math.random() * 16 | 0, v = c === 'x' ? r : (r & 0x3 | 0x8);
                return v.toString(16);
            });
        }

        {% for class_def in classes %}
        function add{{ class_def.name }}(containerId) {
            const container = document.getElementById(containerId);
            const id = generateUUID();
            const div = document.createElement('div');
            div.className = '{{ class_def.name.lower }}';
            div.id = '{{ class_def.name.lower }}_' + id;
            div.innerHTML = `
                {% for slot in class_def.slots %}
                {% if slot.range not in classes %}
                <div class="form-group">
                    <label>{{ slot.name | replace('_', ' ') | title }}</label>
                    <{% if slot.range == 'date' %}input type="date"{% elif slot.range == 'email' %}input type="email"{% elif slot.multivalued %}input type="text"{% else %}input type="text"{% endif %} class="form-control {{ class_def.name.lower }}_{{ slot.name }}" {% if slot.identifier %}value="${id}" readonly{% endif %} {% if slot.required %}required{% endif %}>
                </div>
                {% else %}
                <div class="section">
                    <h3>{{ slot.name | replace('_', ' ') | title }}</h3>
                    <div id="{{ slot.name }}_${id}" class="nested"></div>
                    <button class="btn btn-secondary" onclick="add{{ slot.range }}('{{ slot.name }}_${id}')">Add {{ slot.range }}</button>
                </div>
                {% endif %}
                {% endfor %}
                <button class="btn btn-danger" onclick="removeElement('{{ class_def.name.lower }}_' + id)">Remove</button>
                <hr>
            `;
            container.appendChild(div);
        }
        {% endfor %}

        function removeElement(id) {
            const element = document.getElementById(id);
            element.remove();
        }

        async function saveContract() {
            const errorDiv = document.getElementById('error');
            errorDiv.textContent = '';

            const contract = {
                {% for slot in classes_dict['SoftwareContract'].slots %}
                {% if slot.range not in classes %}
                {{ slot.name }}: document.getElementById('{{ slot.name }}').value,
                {% else %}
                {{ slot.name }}: Array.from(document.querySelectorAll(`#{{ slot.name }} .{{ slot.range.lower }}`)).map(div => ({
                    {% for sub_slot in classes_dict[slot.range].slots %}
                    {% if sub_slot.range not in classes %}
                    {{ sub_slot.name }}: div.querySelector('.{{ slot.range.lower }}_{{ sub_slot.name }}').value,
                    {% else %}
                    {{ sub_slot.name }}: Array.from(div.querySelectorAll(`#{{ sub_slot.name }}_${div.id.split('_')[1]} .{{ sub_slot.range.lower }}`)).map(subDiv => ({
                        {% for sub_sub_slot in classes_dict[sub_slot.range].slots %}
                        {{ sub_sub_slot.name }}: subDiv.querySelector('.{{ sub_slot.range.lower }}_{{ sub_sub_slot.name }}').value,
                        {% endfor %}
                    })),
                    {% endif %}
                    {% endfor %}
                })),
                {% endif %}
                {% endfor %}
            };

            try {
                const response = await fetch(contractId ? `/api/softwares/${contractId}/` : '/api/softwares/', {
                    method: contractId ? 'PUT' : 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(contract)
                });
                if (!response.ok) throw new Error('Failed to save contract');
                const data = await response.json();
                contractId = data.contract_id;
                errorDiv.textContent = 'Contract saved successfully!';
                setTimeout(() => errorDiv.textContent = '', 3000);
            } catch (error) {
                errorDiv.textContent = `Error: ${error.message}`;
            }
        }

        async function exportYAML() {
            try {
                const response = await fetch(contractId ? `/api/softwares/${contractId}/` : '/api/softwares/');
                if (!response.ok) throw new Error('Failed to fetch contract');
                const data = contractId ? await response.json() : (await response.json())[0];
                const yaml = jsyaml.dump(data, { skipInvalid: true });
                const blob = new Blob([yaml], { type: 'text/yaml' });
                const url = URL.createObjectURL(blob);
                const a = document.createElement('a');
                a.href = url;
                a.download = `contract_${data.contract_id}.yaml`;
                a.click();
                URL.revokeObjectURL(url);
            } catch (error) {
                document.getElementById('error').textContent = `Error: ${error.message}`;
            }
        }

        async function loadContract(id) {
            if (!id) return;
            try {
                const response = await fetch(`/api/softwares/${id}/`);
                if (!response.ok) throw new Error('Failed to load contract');
                const data = await response.json();
                contractId = data.contract_id;
                {% for slot in classes_dict['SoftwareContract'].slots %}
                {% if slot.range not in classes %}
                document.getElementById('{{ slot.name }}').value = data.{{ slot.name }} || '';
                {% else %}
                document.getElementById('{{ slot.name }}').innerHTML = '';
                data.{{ slot.name }}.forEach(item => {
                    add{{ slot.range }}('{{ slot.name }}');
                    const div = document.querySelector(`#{{ slot.name }} .{{ slot.range.lower }}[id$='${item.{% for sub_slot in classes_dict[slot.range].slots %}{% if sub_slot.identifier %}{{ sub_slot.name }}{% endif %}{% endfor %}}']`);
                    {% for sub_slot in classes_dict[slot.range].slots %}
                    {% if sub_slot.range not in classes %}
                    div.querySelector('.{{ slot.range.lower }}_{{ sub_slot.name }}').value = item.{{ sub_slot.name }} || '';
                    {% else %}
                    const subContainer = document.getElementById('{{ sub_slot.name }}_${item.{% for sub_sub_slot in classes_dict[slot.range].slots %}{% if sub_sub_slot.identifier %}{{ sub_sub_slot.name }}{% endif %}{% endfor %}}');
                    subContainer.innerHTML = '';
                    item.{{ sub_slot.name }}.forEach(subItem => {
                        add{{ sub_slot.range }}('{{ sub_slot.name }}_${item.{% for sub_sub_slot in classes_dict[slot.range].slots %}{% if sub_sub_slot.identifier %}{{ sub_sub_slot.name }}{% endif %}{% endfor %}}');
                        const subDiv = subContainer.querySelector(`.{{ sub_slot.range.lower }}[id$='${subItem.{% for sub_sub_slot in classes_dict[sub_slot.range].slots %}{% if sub_sub_slot.identifier %}{{ sub_sub_slot.name }}{% endif %}{% endfor %}}']`);
                        {% for sub_sub_slot in classes_dict[sub_slot.range].slots %}
                        subDiv.querySelector('.{{ sub_slot.range.lower }}_{{ sub_sub_slot.name }}').value = subItem.{{ sub_sub_slot.name }} || '';
                        {% endfor %}
                    });
                    {% endif %}
                    {% endfor %}
                });
                {% endif %}
                {% endfor %}
            } catch (error) {
                document.getElementById('error').textContent = `Error: ${error.message}`;
            }
        }

        const urlParams = new URLSearchParams(window.location.search);
        const contractIdParam = urlParams.get('contract_id');
        if (contractIdParam) loadContract(contractIdParam);

        {% for slot in classes_dict['SoftwareContract'].slots %}
        {% if slot.range in classes %}
        add{{ slot.range }}('{{ slot.name }}');
        {% endif %}
        {% endfor %}
    </script>
</body>
</html>
""")

SETTINGS_TEMPLATE = Template("""
import os

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

SECRET_KEY = 'your-secret-key-here'  # Replace with a secure key

DEBUG = True

ALLOWED_HOSTS = ['localhost', '127.0.0.1']

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'corsheaders',
    'contract_app',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

CORS_ALLOW_ALL_ORIGINS = True  # For development only; restrict in production

ROOT_URLCONF = 'contract_editor.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'contract_editor.wsgi.application'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}

AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

LANGUAGE_CODE = 'en-us'

TIME_ZONE = 'UTC'

USE_I18N = True

USE_TZ = True

STATIC_URL = '/static/'

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
""")

PROJECT_URLS_TEMPLATE = Template("""
from django.urls import path, include
from django.views.generic import TemplateView

urlpatterns = [
    path('', TemplateView.as_view(template_name='index.html'), name='editor'),
    path('api/', include('contract_app.urls')),
]
""")

# Generator Logic
def generate_django_app(output_dir="generated_django_app"):
    # Parse the LinkML schema
    schema_dict = yaml.safe_load(LINKML_SCHEMA)
    schema = yaml_loader.load(schema_dict, target_class=SchemaDefinition)

    # Extract classes and slots
    classes = schema.classes.values()
    class_names = [c.name for c in classes]
    classes_dict = {c.name: c for c in classes}

    # Create output directory structure
    os.makedirs(output_dir, exist_ok=True)
    os.makedirs(os.path.join(output_dir, "contract_app"), exist_ok=True)
    os.makedirs(os.path.join(output_dir, "contract_app", "migrations"), exist_ok=True)
    os.makedirs(os.path.join(output_dir, "templates"), exist_ok=True)
    os.makedirs(os.path.join(output_dir, "contract_editor"), exist_ok=True)

    # Generate models.py
    models_content = MODELS_TEMPLATE.render(classes=classes, classes_dict=classes_dict)
    with open(os.path.join(output_dir, "contract_app", "models.py"), "w") as f:
        f.write(models_content)

    # Generate serializers.py
    serializers_content = SERIALIZERS_TEMPLATE.render(classes=classes, classes_dict=classes_dict)
    with open(os.path.join(output_dir, "contract_app", "serializers.py"), "w") as f:
        f.write(serializers_content)

    # Generate views.py
    views_content = VIEWS_TEMPLATE.render(classes=classes)
    with open(os.path.join(output_dir, "contract_app", "views.py"), "w") as f:
        f.write(views_content)

    # Generate urls.py (app-level)
    urls_content = URLS_TEMPLATE.render(classes=classes)
    with open(os.path.join(output_dir, "contract_app", "urls.py"), "w") as f:
        f.write(urls_content)

    # Generate index.html
    index_content = INDEX_HTML_TEMPLATE.render(classes=classes, classes_dict=classes_dict)
    with open(os.path.join(output_dir, "templates", "index.html"), "w") as f:
        f.write(index_content)

    # Generate settings.py
    settings_content = SETTINGS_TEMPLATE.render()
    with open(os.path.join(output_dir, "contract_editor", "settings.py"), "w") as f:
        f.write(settings_content)

    # Generate project urls.py
    project_urls_content = PROJECT_URLS_TEMPLATE.render()
    with open(os.path.join(output_dir, "contract_editor", "urls.py"), "w") as f:
        f.write(project_urls_content)

    # Print generated files for reference
    print("Generated Files:")
    print(f"contract_app/models.py:\n{models_content}\n")
    print(f"contract_app/serializers.py:\n{serializers_content}\n")
    print(f"contract_app/views.py:\n{views_content}\n")
    print(f"contract_app/urls.py:\n{urls_content}\n")
    print(f"templates/index.html:\n{index_content}\n")
    print(f"contract_editor/settings.py:\n{settings_content}\n")
    print(f"contract_editor/urls.py:\n{project_urls_content}\n")

if __name__ == "__main__":
    generate_django_app()
```

### How It Works
- **Schema Parsing**: The script loads the LinkML schema using `linkml_runtime` and extracts classes and slots.
- **Code Generation**:
    - **Models**: Iterates over classes and slots to create Django models. Slots with `range` as a class become `ForeignKey` or `ManyToManyField` based on `multivalued` and `inlined`. Simple types (e.g., `string`, `date`) map to appropriate Django fields.
    - **Serializers**: Generates serializers for each model, with nested serializers for relationships. Custom `create` and `update` methods handle nested data.
    - **Views**: Creates a `ModelViewSet` for each model, linking to its serializer.
    - **URLs**: Registers each viewset with a router, creating REST endpoints (e.g., `/api/softwares/`, `/api/conditions/`).
    - **Frontend**: Generates a dynamic `index.html` with forms for each slot in `SoftwareContract`. Nested classes (e.g., `Condition`, `Metadata`) are handled with recursive form sections. JavaScript functions (`addClassName`, `saveContract`, `exportYAML`) are generated per class.
- **Maintenance**: To update the application after a schema change:
    1. Modify the `LINKML_SCHEMA` string in the script (or load from a file).
    2. Re-run the script: `python generate_django_app.py`.
    3. Copy the generated files to your Django project.
    4. Run migrations: `python manage.py makemigrations && python manage.py migrate`.

### Setup Instructions
1. Install dependencies: `pip install linkml-runtime jinja2 django djangorestframework django-cors-headers pyyaml`.
2. Save the script as `generate_django_app.py`.
3. Run the script: `python generate_django_app.py`. This creates a `generated_django_app` directory with all files.
4. Create a Django project if not already done: `django-admin startproject contract_editor`.
5. Create a Django app: `python manage.py startapp contract_app`.
6. Copy the generated files to the project:
    - `contract_app/models.py`, `serializers.py`, `views.py`, `urls.py`
    - `templates/index.html`
    - `contract_editor/settings.py`, `urls.py`
7. Update `contract_editor/settings.py` to include `'contract_app'` in `INSTALLED_APPS` if not already done.
8. Run migrations: `python manage.py makemigrations && python manage.py migrate`.
9. Start the server: `python manage.py runserver`.
10. Access the editor at `http://localhost:8000/`.

### Generated Output
The script generates:
- **models.py**: Django models for each LinkML class, with fields and relationships based on slots.
- **serializers.py**: REST Framework serializers with nested handling for relationships.
- **views.py**: Viewsets for each model, providing CRUD operations.
- **urls.py**: API endpoints for all models.
- **index.html**: A dynamic editor with forms for all slots, supporting nested structures and YAML export.
- **settings.py**: Django project configuration with REST Framework and CORS.
- **project urls.py**: Routes for the editor and API.

### Benefits
- **Schema-Driven**: The generator uses the LinkML schema's metadata to produce all components, ensuring consistency.
- **Automated Maintenance**: Schema changes are propagated by re-running the script, minimizing manual updates.
- **Extensible**: The templates can be modified to add custom logic (e.g., additional validation, UI features).
- **Reusable**: The generator can be adapted for other LinkML schemas by updating the templates.

### Limitations
- **Frontend Complexity**: The generated frontend supports basic CRUD but may need manual tweaks for advanced UI features (e.g., validation, styling).
- **Nested Depth**: The current implementation handles two levels of nesting (e.g., `Metadata` → `Authors`). Deeper nesting requires extending the templates.
- **Schema Validation**: The generator assumes a valid LinkML schema. Invalid schemas may produce incorrect code.
- **Production Readiness**: The generated code is development-ready. For production, secure `SECRET_KEY`, disable `DEBUG`, and configure CORS.

### Notes
- The artifact includes the generator script with the schema inline for simplicity. You can modify it to load the schema from a file.
- The frontend uses Bootstrap and `js-yaml` for consistency with the previous implementation.
- The generator produces a single artifact (the script) to comply with the one-artifact rule, but it outputs multiple files to the filesystem.
- To extend the generator, modify the Jinja2 templates or add new ones for additional files (e.g., admin.py, tests.py).

This meta-driven generator fulfills the requirement to leverage the LinkML schema's structure, automating the creation and maintenance of the Django application and ensuring that schema changes are easily propagated to the codebase.