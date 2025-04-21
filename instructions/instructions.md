To create a web-based editor for the Software Contract Schema using Django, we need to build a Django application that allows users to create, edit, and save contract instances as YAML files conforming to the provided LinkML schema. The application will use Django's ORM to model the schema's classes and slots, Django REST Framework for API endpoints, and a frontend (using HTML, JavaScript, and a library like Bootstrap) to provide an interactive editor. The views and viewsets will be generated based on the schema to minimize maintenance overhead, and the application will support exporting the data as YAML.

Given the complexity of the schema (with nested classes like `SoftwareContract`, `Condition`, `BehaviorDescription`, `System`, `Metadata`, `Author`, and `ReviewerComment`), we'll use Django models to represent these classes, Django REST Framework to create API endpoints, and a simple frontend to interact with the data. The editor will allow users to input data through forms, save it to the backend, and export it as a YAML file that conforms to the LinkML schema. To make maintenance easier, we'll structure the code to map directly to the schema, so updates to the schema can be reflected by regenerating models and serializers.

Since Django is a server-side framework, we'll create a single-page HTML application with JavaScript to handle the frontend, communicating with the Django backend via REST API. We'll avoid React for simplicity, as the user didn't specify it, and instead use vanilla JavaScript with Bootstrap for styling. The application will be a single HTML page with embedded JavaScript, served by Django, to meet the requirement of a web technology solution.

Below is the implementation, including Django models, serializers, views, URLs, and a frontend HTML page with JavaScript for the editor. The code is organized into multiple files, but since only one artifact is allowed per response, I'll provide the main `index.html` (including the frontend) and describe the structure of the Django project. The other files (models, serializers, views, URLs) will be included as content within the artifact for clarity, with instructions on where they should be placed in the project directory.

### Project Structure
```
/contract_editor/
  /contract_app/
    /migrations/
    __init__.py
    admin.py
    apps.py
    models.py        # Django models generated from the schema
    serializers.py   # DRF serializers for API
    views.py         # DRF viewsets for API
    urls.py          # API URL routing
  /templates/
    index.html       # Frontend editor
  /contract_editor/
    __init__.py
    settings.py      # Django settings
    urls.py          # Main URL configuration
  manage.py
```

### Key Components
1. **Models (`models.py`)**: Django models for each schema class (`SoftwareContract`, `Condition`, etc.), with fields mapping to the schema's slots. ForeignKey and ManyToManyField handle relationships.
2. **Serializers (`serializers.py`)**: Django REST Framework serializers to convert models to JSON and validate input, mirroring the schema structure.
3. **Views (`views.py`)**: Viewsets for each model, providing CRUD operations via REST API.
4. **URLs (`urls.py`)**: API endpoints for each model and a route for the editor page.
5. **Frontend (`index.html`)**: A single-page HTML application with forms to edit contract data, JavaScript to interact with the API, and a button to export as YAML.
6. **Settings (`settings.py`)**: Configures Django with REST Framework and CORS for API access.

### Implementation Details
- **Model Generation**: Each LinkML class becomes a Django model. Slots become model fields (e.g., `CharField`, `TextField`, `DateField`). Relationships (`multivalued`, `inlined`) are handled with `ForeignKey` or `ManyToManyField`.
- **Serializer Generation**: Serializers mirror models, with nested serializers for relationships (e.g., `preconditions`, `authors`).
- **View Generation**: ModelViewSets provide REST endpoints for each model, with automatic CRUD operations.
- **Frontend**: The editor loads a contract (or creates a new one), displays forms for each component (contract, conditions, etc.), and allows saving/exporting. YAML export is handled client-side using `js-yaml`.
- **YAML Export**: The frontend fetches the contract data via API, converts it to a YAML-compatible structure, and downloads it as a file.
- **Maintenance**: The models, serializers, and views are structured to map directly to the schema. If the schema changes, you can regenerate these files by updating the field definitions to match the new slots/classes.

### Artifact
The artifact below contains the main `index.html` for the frontend editor and includes the content of other necessary files (`models.py`, `serializers.py`, `views.py`, `urls.py`, `settings.py`) as comments, with instructions on where to place them. The `index.html` includes Bootstrap for styling, `js-yaml` for YAML conversion, and JavaScript to handle API calls and form interactions.

```html
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
            <div class="section">
                <h2>Contract Details</h2>
                <div class="form-group">
                    <label for="contract_id">Contract ID</label>
                    <input type="text" id="contract_id" class="form-control" required>
                </div>
                <div class="form-group">
                    <label for="name">Name</label>
                    <input type="text" id="name" class="form-control" required>
                </div>
                <div class="form-group">
                    <label for="description">Description</label>
                    <textarea id="description" class="form-control"></textarea>
                </div>
            </div>
            <div class="section">
                <h2>Preconditions</h2>
                <div id="preconditions" class="nested"></div>
                <button class="btn btn-secondary" onclick="addPrecondition()">Add Precondition</button>
            </div>
            <div class="section">
                <h2>Postconditions</h2>
                <div id="postconditions" class="nested"></div>
                <button class="btn btn-secondary" onclick="addPostcondition()">Add Postcondition</button>
            </div>
            <div class="section">
                <h2>Invariants</h2>
                <div id="invariants" class="nested"></div>
                <button class="btn btn-secondary" onclick="addInvariant()">Add Invariant</button>
            </div>
            <div class="section">
                <h2>Behavior Description</h2>
                <div id="behavior_description" class="nested">
                    <div class="form-group">
                        <label for="behavior_id">Behavior ID</label>
                        <input type="text" id="behavior_id" class="form-control" required>
                    </div>
                    <div class="form-group">
                        <label for="behavior_description">Description</label>
                        <textarea id="behavior_description_text" class="form-control"></textarea>
                    </div>
                    <div class="form-group">
                        <label for="input_parameters">Input Parameters (comma-separated)</label>
                        <input type="text" id="input_parameters" class="form-control">
                    </div>
                    <div class="form-group">
                        <label for="output_parameters">Output Parameters (comma-separated)</label>
                        <input type="text" id="output_parameters" class="form-control">
                    </div>
                    <div class="form-group">
                        <label for="sequence_diagram_ref">Sequence Diagram Reference</label>
                        <input type="text" id="sequence_diagram_ref" class="form-control">
                    </div>
                </div>
            </div>
            <div class="section">
                <h2>Calling Systems</h2>
                <div id="calling_systems" class="nested"></div>
                <button class="btn btn-secondary" onclick="addCallingSystem()">Add Calling System</button>
            </div>
            <div class="section">
                <h2>Dependent Contracts</h2>
                <div id="dependent_contracts" class="nested"></div>
                <button class="btn btn-secondary" onclick="addDependentContract()">Add Dependent Contract</button>
            </div>
            <div class="section">
                <h2>Metadata</h2>
                <div id="metadata" class="nested">
                    <div class="form-group">
                        <label for="version">Version</label>
                        <input type="text" id="version" class="form-control" required>
                    </div>
                    <div class="form-group">
                        <label for="creation_date">Creation Date</label>
                        <input type="date" id="creation_date" class="form-control">
                    </div>
                    <div class="form-group">
                        <label for="last_modified">Last Modified</label>
                        <input type="date" id="last_modified" class="form-control">
                    </div>
                    <div class="section">
                        <h3>Authors</h3>
                        <div id="authors" class="nested"></div>
                        <button class="btn btn-secondary" onclick="addAuthor()">Add Author</button>
                    </div>
                    <div class="section">
                        <h3>Reviewer Comments</h3>
                        <div id="reviewer_comments" class="nested"></div>
                        <button class="btn btn-secondary" onclick="addReviewerComment()">Add Reviewer Comment</button>
                    </div>
                </div>
            </div>
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

        function addPrecondition() {
            const container = document.getElementById('preconditions');
            const id = generateUUID();
            const div = document.createElement('div');
            div.className = 'condition';
            div.id = `precondition_${id}`;
            div.innerHTML = `
                <div class="form-group">
                    <label>Condition ID</label>
                    <input type="text" class="form-control condition_id" value="${id}" readonly>
                </div>
                <div class="form-group">
                    <label>Description</label>
                    <textarea class="form-control condition_description"></textarea>
                </div>
                <div class="form-group">
                    <label>Expression</label>
                    <input type="text" class="form-control condition_expression">
                </div>
                <div class="form-group">
                    <label>Evaluation Context</label>
                    <input type="text" class="form-control condition_evaluation_context">
                </div>
                <button class="btn btn-danger" onclick="removeElement('precondition_${id}')">Remove</button>
                <hr>
            `;
            container.appendChild(div);
        }

        function addPostcondition() {
            const container = document.getElementById('postconditions');
            const id = generateUUID();
            const div = document.createElement('div');
            div.className = 'condition';
            div.id = `postcondition_${id}`;
            div.innerHTML = `
                <div class="form-group">
                    <label>Condition ID</label>
                    <input type="text" class="form-control condition_id" value="${id}" readonly>
                </div>
                <div class="form-group">
                    <label>Description</label>
                    <textarea class="form-control condition_description"></textarea>
                </div>
                <div class="form-group">
                    <label>Expression</label>
                    <input type="text" class="form-control condition_expression">
                </div>
                <div class="form-group">
                    <label>Evaluation Context</label>
                    <input type="text" class="form-control condition_evaluation_context">
                </div>
                <button class="btn btn-danger" onclick="removeElement('postcondition_${id}')">Remove</button>
                <hr>
            `;
            container.appendChild(div);
        }

        function addInvariant() {
            const container = document.getElementById('invariants');
            const id = generateUUID();
            const div = document.createElement('div');
            div.className = 'condition';
            div.id = `invariant_${id}`;
            div.innerHTML = `
                <div class="form-group">
                    <label>Condition ID</label>
                    <input type="text" class="form-control condition_id" value="${id}" readonly>
                </div>
                <div class="form-group">
                    <label>Description</label>
                    <textarea class="form-control condition_description"></textarea>
                </div>
                <div class="form-group">
                    <label>Expression</label>
                    <input type="text" class="form-control condition_expression">
                </div>
                <div class="form-group">
                    <label>Evaluation Context</label>
                    <input type="text" class="form-control condition_evaluation_context">
                </div>
                <button class="btn btn-danger" onclick="removeElement('invariant_${id}')">Remove</button>
                <hr>
            `;
            container.appendChild(div);
        }

        function addCallingSystem() {
            const container = document.getElementById('calling_systems');
            const id = generateUUID();
            const div = document.createElement('div');
            div.className = 'system';
            div.id = `system_${id}`;
            div.innerHTML = `
                <div class="form-group">
                    <label>System ID</label>
                    <input type="text" class="form-control system_id" value="${id}" readonly>
                </div>
                <div class="form-group">
                    <label>Name</label>
                    <input type="text" class="form-control system_name" required>
                </div>
                <div class="form-group">
                    <label>Description</label>
                    <textarea class="form-control system_description"></textarea>
                </div>
                <div class="form-group">
                    <label>Owner</label>
                    <input type="text" class="form-control system_owner">
                </div>
                <div class="form-group">
                    <label>Contact Info</label>
                    <input type="text" class="form-control system_contact_info">
                </div>
                <button class="btn btn-danger" onclick="removeElement('system_${id}')">Remove</button>
                <hr>
            `;
            container.appendChild(div);
        }

        function addDependentContract() {
            const container = document.getElementById('dependent_contracts');
            const id = generateUUID();
            const div = document.createElement('div');
            div.className = 'dependent_contract';
            div.id = `dependent_contract_${id}`;
            div.innerHTML = `
                <div class="form-group">
                    <label>Contract ID</label>
                    <input type="text" class="form-control dependent_contract_id" value="${id}" readonly>
                </div>
                <div class="form-group">
                    <label>Name</label>
                    <input type="text" class="form-control dependent_contract_name" required>
                </div>
                <div class="form-group">
                    <label>Description</label>
                    <textarea class="form-control dependent_contract_description"></textarea>
                </div>
                <button class="btn btn-danger" onclick="removeElement('dependent_contract_${id}')">Remove</button>
                <hr>
            `;
            container.appendChild(div);
        }

        function addAuthor() {
            const container = document.getElementById('authors');
            const id = generateUUID();
            const div = document.createElement('div');
            div.className = 'author';
            div.id = `author_${id}`;
            div.innerHTML = `
                <div class="form-group">
                    <label>Author ID</label>
                    <input type="text" class="form-control author_id" value="${id}" readonly>
                </div>
                <div class="form-group">
                    <label>Name</label>
                    <input type="text" class="form-control author_name" required>
                </div>
                <div class="form-group">
                    <label>Email</label>
                    <input type="email" class="form-control author_email">
                </div>
                <div class="form-group">
                    <label>Organization</label>
                    <input type="text" class="form-control author_organization">
                </div>
                <button class="btn btn-danger" onclick="removeElement('author_${id}')">Remove</button>
                <hr>
            `;
            container.appendChild(div);
        }

        function addReviewerComment() {
            const container = document.getElementById('reviewer_comments');
            const id = generateUUID();
            const div = document.createElement('div');
            div.className = 'reviewer_comment';
            div.id = `reviewer_comment_${id}`;
            div.innerHTML = `
                <div class="form-group">
                    <label>Comment ID</label>
                    <input type="text" class="form-control reviewer_comment_id" value="${id}" readonly>
                </div>
                <div class="form-group">
                    <label>Reviewer Name</label>
                    <input type="text" class="form-control reviewer_name" required>
                </div>
                <div class="form-group">
                    <label>Comment Text</label>
                    <textarea class="form-control reviewer_comment_text" required></textarea>
                </div>
                <div class="form-group">
                    <label>Comment Date</label>
                    <input type="date" class="form-control reviewer_comment_date">
                </div>
                <button class="btn btn-danger" onclick="removeElement('reviewer_comment_${id}')">Remove</button>
                <hr>
            `;
            container.appendChild(div);
        }

        function removeElement(id) {
            const element = document.getElementById(id);
            element.remove();
        }

        async function saveContract() {
            const errorDiv = document.getElementById('error');
            errorDiv.textContent = '';

            const contract = {
                contract_id: document.getElementById('contract_id').value || generateUUID(),
                name: document.getElementById('name').value,
                description: document.getElementById('description').value,
                preconditions: Array.from(document.querySelectorAll('#preconditions .condition')).map(div => ({
                    condition_id: div.querySelector('.condition_id').value,
                    description: div.querySelector('.condition_description').value,
                    expression: div.querySelector('.condition_expression').value,
                    evaluation_context: div.querySelector('.condition_evaluation_context').value
                })),
                postconditions: Array.from(document.querySelectorAll('#postconditions .condition')).map(div => ({
                    condition_id: div.querySelector('.condition_id').value,
                    description: div.querySelector('.condition_description').value,
                    expression: div.querySelector('.condition_expression').value,
                    evaluation_context: div.querySelector('.condition_evaluation_context').value
                })),
                invariants: Array.from(document.querySelectorAll('#invariants .condition')).map(div => ({
                    condition_id: div.querySelector('.condition_id').value,
                    description: div.querySelector('.condition_description').value,
                    expression: div.querySelector('.condition_expression').value,
                    evaluation_context: div.querySelector('.condition_evaluation_context').value
                })),
                behavior_description: {
                    behavior_id: document.getElementById('behavior_id').value || generateUUID(),
                    description: document.getElementById('behavior_description_text').value,
                    input_parameters: document.getElementById('input_parameters').value.split(',').map(s => s.trim()).filter(s => s),
                    output_parameters: document.getElementById('output_parameters').value.split(',').map(s => s.trim()).filter(s => s),
                    sequence_diagram_ref: document.getElementById('sequence_diagram_ref').value
                },
                calling_systems: Array.from(document.querySelectorAll('#calling_systems .system')).map(div => ({
                    system_id: div.querySelector('.system_id').value,
                    name: div.querySelector('.system_name').value,
                    description: div.querySelector('.system_description').value,
                    owner: div.querySelector('.system_owner').value,
                    contact_info: div.querySelector('.system_contact_info').value
                })),
                dependent_contracts: Array.from(document.querySelectorAll('#dependent_contracts .dependent_contract')).map(div => ({
                    contract_id: div.querySelector('.dependent_contract_id').value,
                    name: div.querySelector('.dependent_contract_name').value,
                    description: div.querySelector('.dependent_contract_description').value
                })),
                metadata: {
                    version: document.getElementById('version').value,
                    creation_date: document.getElementById('creation_date').value,
                    last_modified: document.getElementById('last_modified').value,
                    authors: Array.from(document.querySelectorAll('#authors .author')).map(div => ({
                        author_id: div.querySelector('.author_id').value,
                        name: div.querySelector('.author_name').value,
                        email: div.querySelector('.author_email').value,
                        organization: div.querySelector('.author_organization').value
                    })),
                    reviewer_comments: Array.from(document.querySelectorAll('#reviewer_comments .reviewer_comment')).map(div => ({
                        comment_id: div.querySelector('.reviewer_comment_id').value,
                        reviewer_name: div.querySelector('.reviewer_name').value,
                        comment_text: div.querySelector('.reviewer_comment_text').value,
                        comment_date: div.querySelector('.reviewer_comment_date').value
                    }))
                }
            };

            try {
                const response = await fetch(contractId ? `/api/contracts/${contractId}/` : '/api/contracts/', {
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
                const response = await fetch(contractId ? `/api/contracts/${contractId}/` : '/api/contracts/');
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
                const response = await fetch(`/api/contracts/${id}/`);
                if (!response.ok) throw new Error('Failed to load contract');
                const data = await response.json();
                contractId = data.contract_id;
                document.getElementById('contract_id').value = data.contract_id;
                document.getElementById('name').value = data.name;
                document.getElementById('description').value = data.description;

                document.getElementById('preconditions').innerHTML = '';
                data.preconditions.forEach(c => {
                    addPrecondition();
                    const div = document.querySelector(`#precondition_${c.condition_id}`);
                    div.querySelector('.condition_id').value = c.condition_id;
                    div.querySelector('.condition_description').value = c.description || '';
                    div.querySelector('.condition_expression').value = c.expression || '';
                    div.querySelector('.condition_evaluation_context').value = c.evaluation_context || '';
                });

                document.getElementById('postconditions').innerHTML = '';
                data.postconditions.forEach(c => {
                    addPostcondition();
                    const div = document.querySelector(`#postcondition_${c.condition_id}`);
                    div.querySelector('.condition_id').value = c.condition_id;
                    div.querySelector('.condition_description').value = c.description || '';
                    div.querySelector('.condition_expression').value = c.expression || '';
                    div.querySelector('.condition_evaluation_context').value = c.evaluation_context || '';
                });

                document.getElementById('invariants').innerHTML = '';
                data.invariants.forEach(c => {
                    addInvariant();
                    const div = document.querySelector(`#invariant_${c.condition_id}`);
                    div.querySelector('.condition_id').value = c.condition_id;
                    div.querySelector('.condition_description').value = c.description || '';
                    div.querySelector('.condition_expression').value = c.expression || '';
                    div.querySelector('.condition_evaluation_context').value = c.evaluation_context || '';
                });

                document.getElementById('behavior_id').value = data.behavior_description.behavior_id;
                document.getElementById('behavior_description_text').value = data.behavior_description.description || '';
                document.getElementById('input_parameters').value = data.behavior_description.input_parameters.join(', ');
                document.getElementById('output_parameters').value = data.behavior_description.output_parameters.join(', ');
                document.getElementById('sequence_diagram_ref').value = data.behavior_description.sequence_diagram_ref || '';

                document.getElementById('calling_systems').innerHTML = '';
                data.calling_systems.forEach(s => {
                    addCallingSystem();
                    const div = document.querySelector(`#system_${s.system_id}`);
                    div.querySelector('.system_id').value = s.system_id;
                    div.querySelector('.system_name').value = s.name;
                    div.querySelector('.system_description').value = s.description || '';
                    div.querySelector('.system_owner').value = s.owner || '';
                    div.querySelector('.system_contact_info').value = s.contact_info || '';
                });

                document.getElementById('dependent_contracts').innerHTML = '';
                data.dependent_contracts.forEach(c => {
                    addDependentContract();
                    const div = document.querySelector(`#dependent_contract_${c.contract_id}`);
                    div.querySelector('.dependent_contract_id').value = c.contract_id;
                    div.querySelector('.dependent_contract_name').value = c.name;
                    div.querySelector('.dependent_contract_description').value = c.description || '';
                });

                document.getElementById('version').value = data.metadata.version;
                document.getElementById('creation_date').value = data.metadata.creation_date || '';
                document.getElementById('last_modified').value = data.metadata.last_modified || '';

                document.getElementById('authors').innerHTML = '';
                data.metadata.authors.forEach(a => {
                    addAuthor();
                    const div = document.querySelector(`#author_${a.author_id}`);
                    div.querySelector('.author_id').value = a.author_id;
                    div.querySelector('.author_name').value = a.name;
                    div.querySelector('.author_email').value = a.email || '';
                    div.querySelector('.author_organization').value = a.organization || '';
                });

                document.getElementById('reviewer_comments').innerHTML = '';
                data.metadata.reviewer_comments.forEach(c => {
                    addReviewerComment();
                    const div = document.querySelector(`#reviewer_comment_${c.comment_id}`);
                    div.querySelector('.reviewer_comment_id').value = c.comment_id;
                    div.querySelector('.reviewer_name').value = c.reviewer_name;
                    div.querySelector('.reviewer_comment_text').value = c.comment_text;
                    div.querySelector('.reviewer_comment_date').value = c.comment_date || '';
                });
            } catch (error) {
                document.getElementById('error').textContent = `Error: ${error.message}`;
            }
        }

        // Load a contract if ID is provided in URL
        const urlParams = new URLSearchParams(window.location.search);
        const contractIdParam = urlParams.get('contract_id');
        if (contractIdParam) loadContract(contractIdParam);

        // Add one instance of each repeatable section to start
        addPrecondition();
        addPostcondition();
        addInvariant();
        addCallingSystem();
        addDependentContract();
        addAuthor();
        addReviewerComment();
    </script>

    <!--
    Place the following files in the appropriate directories as described below.
    -->

    <!-- File: contract_app/models.py -->
    <!--
    from django.db import models

    class Condition(models.Model):
        condition_id = models.CharField(max_length=36, primary_key=True)
        description = models.TextField(blank=True)
        expression = models.TextField(blank=True)
        evaluation_context = models.TextField(blank=True)

        def __str__(self):
            return self.condition_id

    class BehaviorDescription(models.Model):
        behavior_id = models.CharField(max_length=36, primary_key=True)
        description = models.TextField(blank=True)
        input_parameters = models.JSONField(default=list)
        output_parameters = models.JSONField(default=list)
        sequence_diagram_ref = models.TextField(blank=True)

        def __str__(self):
            return self.behavior_id

    class System(models.Model):
        system_id = models.CharField(max_length=36, primary_key=True)
        name = models.CharField(max_length=255)
        description = models.TextField(blank=True)
        owner = models.CharField(max_length=255, blank=True)
        contact_info = models.CharField(max_length=255, blank=True)

        def __str__(self):
            return self.name

    class Author(models.Model):
        author_id = models.CharField(max_length=36, primary_key=True)
        name = models.CharField(max_length=255)
        email = models.EmailField(blank=True)
        organization = models.CharField(max_length=255, blank=True)

        def __str__(self):
            return self.name

    class ReviewerComment(models.Model):
        comment_id = models.CharField(max_length=36, primary_key=True)
        reviewer_name = models.CharField(max_length=255)
        comment_text = models.TextField()
        comment_date = models.DateField(blank=True, null=True)

        def __str__(self):
            return f"{self.reviewer_name}: {self.comment_id}"

    class Metadata(models.Model):
        version = models.CharField(max_length=50)
        creation_date = models.DateField(blank=True, null=True)
        last_modified = models.DateField(blank=True, null=True)
        authors = models.ManyToManyField(Author)
        reviewer_comments = models.ManyToManyField(ReviewerComment)

        def __str__(self):
            return self.version

    class SoftwareContract(models.Model):
        contract_id = models.CharField(max_length=36, primary_key=True)
        name = models.CharField(max_length=255)
        description = models.TextField(blank=True)
        preconditions = models.ManyToManyField(Condition, related_name='precondition_contracts')
        postconditions = models.ManyToManyField(Condition, related_name='postcondition_contracts')
        invariants = models.ManyToManyField(Condition, related_name='invariant_contracts')
        behavior_description = models.OneToOneField(BehaviorDescription, on_delete=models.CASCADE)
        calling_systems = models.ManyToManyField(System, related_name='calling_contracts')
        dependent_contracts = models.ManyToManyField('self', symmetrical=False, blank=True)
        metadata = models.OneToOneField(Metadata, on_delete=models.CASCADE)

        def __str__(self):
            return self.name
    -->

    <!-- File: contract_app/serializers.py -->
    <!--
    from rest_framework import serializers
    from .models import Condition, BehaviorDescription, System, Author, ReviewerComment, Metadata, SoftwareContract

    class ConditionSerializer(serializers.ModelSerializer):
        class Meta:
            model = Condition
            fields = ['condition_id', 'description', 'expression', 'evaluation_context']

    class BehaviorDescriptionSerializer(serializers.ModelSerializer):
        class Meta:
            model = BehaviorDescription
            fields = ['behavior_id', 'description', 'input_parameters', 'output_parameters', 'sequence_diagram_ref']

    class SystemSerializer(serializers.ModelSerializer):
        class Meta:
            model = System
            fields = ['system_id', 'name', 'description', 'owner', 'contact_info']

    class AuthorSerializer(serializers.ModelSerializer):
        class Meta:
            model = Author
            fields = ['author_id', 'name', 'email', 'organization']

    class ReviewerCommentSerializer(serializers.ModelSerializer):
        class Meta:
            model = ReviewerComment
            fields = ['comment_id', 'reviewer_name', 'comment_text', 'comment_date']

    class MetadataSerializer(serializers.ModelSerializer):
        authors = AuthorSerializer(many=True)
        reviewer_comments = ReviewerCommentSerializer(many=True)

        class Meta:
            model = Metadata
            fields = ['version', 'creation_date', 'last_modified', 'authors', 'reviewer_comments']

        def create(self, validated_data):
            authors_data = validated_data.pop('authors')
            comments_data = validated_data.pop('reviewer_comments')
            metadata = Metadata.objects.create(**validated_data)
            for author_data in authors_data:
                author, _ = Author.objects.get_or_create(**author_data)
                metadata.authors.add(author)
            for comment_data in comments_data:
                comment, _ = ReviewerComment.objects.get_or_create(**comment_data)
                metadata.reviewer_comments.add(comment)
            return metadata

        def update(self, instance, validated_data):
            authors_data = validated_data.pop('authors', [])
            comments_data = validated_data.pop('reviewer_comments', [])
            for attr, value in validated_data.items():
                setattr(instance, attr, value)
            instance.authors.clear()
            for author_data in authors_data:
                author, _ = Author.objects.get_or_create(**author_data)
                instance.authors.add(author)
            instance.reviewer_comments.clear()
            for comment_data in comments_data:
                comment, _ = ReviewerComment.objects.get_or_create(**comment_data)
                instance.reviewer_comments.add(comment)
            instance.save()
            return instance

    class SoftwareContractSerializer(serializers.ModelSerializer):
        preconditions = ConditionSerializer(many=True)
        postconditions = ConditionSerializer(many=True)
        invariants = ConditionSerializer(many=True)
        behavior_description = BehaviorDescriptionSerializer()
        calling_systems = SystemSerializer(many=True)
        dependent_contracts = serializers.PrimaryKeyRelatedField(many=True, queryset=SoftwareContract.objects.all(), required=False)
        metadata = MetadataSerializer()

        class Meta:
            model = SoftwareContract
            fields = [
                'contract_id', 'name', 'description', 'preconditions', 'postconditions',
                'invariants', 'behavior_description', 'calling_systems', 'dependent_contracts', 'metadata'
            ]

        def create(self, validated_data):
            preconditions_data = validated_data.pop('preconditions')
            postconditions_data = validated_data.pop('postconditions')
            invariants_data = validated_data.pop('invariants')
            behavior_data = validated_data.pop('behavior_description')
            systems_data = validated_data.pop('calling_systems')
            dependent_contracts_data = validated_data.pop('dependent_contracts', [])
            metadata_data = validated_data.pop('metadata')

            behavior = BehaviorDescription.objects.create(**behavior_data)
            metadata = MetadataSerializer().create(metadata_data)
            contract = SoftwareContract.objects.create(
                behavior_description=behavior, metadata=metadata, **validated_data
            )

            for condition_data in preconditions_data:
                condition, _ = Condition.objects.get_or_create(**condition_data)
                contract.preconditions.add(condition)
            for condition_data in postconditions_data:
                condition, _ = Condition.objects.get_or_create(**condition_data)
                contract.postconditions.add(condition)
            for condition_data in invariants_data:
                condition, _ = Condition.objects.get_or_create(**condition_data)
                contract.invariants.add(condition)
            for system_data in systems_data:
                system, _ = System.objects.get_or_create(**system_data)
                contract.calling_systems.add(system)
            for dep_contract in dependent_contracts_data:
                contract.dependent_contracts.add(dep_contract)

            return contract

        def update(self, instance, validated_data):
            preconditions_data = validated_data.pop('preconditions', [])
            postconditions_data = validated_data.pop('postconditions', [])
            invariants_data = validated_data.pop('invariants', [])
            behavior_data = validated_data.pop('behavior_description', None)
            systems_data = validated_data.pop('calling_systems', [])
            dependent_contracts_data = validated_data.pop('dependent_contracts', [])
            metadata_data = validated_data.pop('metadata', None)

            for attr, value in validated_data.items():
                setattr(instance, attr, value)

            if behavior_data:
                for attr, value in behavior_data.items():
                    setattr(instance.behavior_description, attr, value)
                instance.behavior_description.save()

            if metadata_data:
                MetadataSerializer().update(instance.metadata, metadata_data)

            instance.preconditions.clear()
            for condition_data in preconditions_data:
                condition, _ = Condition.objects.get_or_create(**condition_data)
                instance.preconditions.add(condition)
            instance.postconditions.clear()
            for condition_data in postconditions_data:
                condition, _ = Condition.objects.get_or_create(**condition_data)
                instance.postconditions.add(condition)
            instance.invariants.clear()
            for condition_data in invariants_data:
                condition, _ = Condition.objects.get_or_create(**condition_data)
                instance.invariants.add(condition)
            instance.calling_systems.clear()
            for system_data in systems_data:
                system, _ = System.objects.get_or_create(**system_data)
                instance.calling_systems.add(system)
            instance.dependent_contracts.clear()
            for dep_contract in dependent_contracts_data:
                instance.dependent_contracts.add(dep_contract)

            instance.save()
            return instance
    -->

    <!-- File: contract_app/views.py -->
    <!--
    from rest_framework import viewsets
    from .models import SoftwareContract, Condition, BehaviorDescription, System, Author, ReviewerComment, Metadata
    from .serializers import (
        SoftwareContractSerializer, ConditionSerializer, BehaviorDescriptionSerializer,
        SystemSerializer, AuthorSerializer, ReviewerCommentSerializer, MetadataSerializer
    )

    class SoftwareContractViewSet(viewsets.ModelViewSet):
        queryset = SoftwareContract.objects.all()
        serializer_class = SoftwareContractSerializer

    class ConditionViewSet(viewsets.ModelViewSet):
        queryset = Condition.objects.all()
        serializer_class = ConditionSerializer

    class BehaviorDescriptionViewSet(viewsets.ModelViewSet):
        queryset = BehaviorDescription.objects.all()
        serializer_class = BehaviorDescriptionSerializer

    class SystemViewSet(viewsets.ModelViewSet):
        queryset = System.objects.all()
        serializer_class = SystemSerializer

    class AuthorViewSet(viewsets.ModelViewSet):
        queryset = Author.objects.all()
        serializer_class = AuthorSerializer

    class ReviewerCommentViewSet(viewsets.ModelViewSet):
        queryset = ReviewerComment.objects.all()
        serializer_class = ReviewerCommentSerializer

    class MetadataViewSet(viewsets.ModelViewSet):
        queryset = Metadata.objects.all()
        serializer_class = MetadataSerializer
    -->

    <!-- File: contract_app/urls.py -->
    <!--
    from django.urls import path, include
    from rest_framework.routers import DefaultRouter
    from .views import (
        SoftwareContractViewSet, ConditionViewSet, BehaviorDescriptionViewSet,
        SystemViewSet, AuthorViewSet, ReviewerCommentViewSet, MetadataViewSet
    )

    router = DefaultRouter()
    router.register(r'contracts', SoftwareContractViewSet)
    router.register(r'conditions', ConditionViewSet)
    router.register(r'behaviors', BehaviorDescriptionViewSet)
    router.register(r'systems', SystemViewSet)
    router.register(r'authors', AuthorViewSet)
    router.register(r'reviewer_comments', ReviewerCommentViewSet)
    router.register(r'metadata', MetadataViewSet)

    urlpatterns = [
        path('', include(router.urls)),
    ]
    -->

    <!-- File: contract_editor/urls.py -->
    <!--
    from django.urls import path, include
    from django.views.generic import TemplateView

    urlpatterns = [
        path('', TemplateView.as_view(template_name='index.html'), name='editor'),
        path('api/', include('contract_app.urls')),
    ]
    -->

    <!-- File: contract_editor/settings.py -->
    <!--
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
    -->

    <!--
    Setup Instructions:
    1. Create a Django project: `django-admin startproject contract_editor`
    2. Create a Django app: `python manage.py startapp contract_app`
    3. Place the above files in the specified locations.
    4. Install dependencies: `pip install django djangorestframework django-cors-headers pyyaml`
    5. Run migrations: `python manage.py makemigrations && python manage.py migrate`
    6. Run the server: `python manage.py runserver`
    7. Access the editor at http://localhost:8000/
    -->
</body>
</html>
```

### How It Works
- **Frontend (`index.html`)**:
    - Provides a form-based editor with sections for each part of the contract (e.g., preconditions, metadata).
    - Users can add/remove items (e.g., conditions, authors) dynamically.
    - The `saveContract` function sends the form data to the Django API.
    - The `exportYAML` function fetches the contract data and converts it to YAML using `js-yaml`.
    - Bootstrap is used for styling, and vanilla JavaScript handles API calls and DOM manipulation.

- **Backend**:
    - **Models**: Map directly to the LinkML schema classes. Relationships are handled with `ForeignKey`, `OneToOneField`, and `ManyToManyField`.
    - **Serializers**: Handle nested data structures and validation. The `SoftwareContractSerializer` manages the entire contract, including nested components.
    - **Views**: Use Django REST Framework's `ModelViewSet` for automatic CRUD operations.
    - **URLs**: Expose API endpoints for all models and serve the editor page.

- **Maintenance**:
    - If the schema changes (e.g., new slots or classes), update `models.py` to reflect new fields or models, then update `serializers.py` and `views.py` accordingly.
    - The frontend may need minor updates to add form fields for new slots.
    - Since the models and serializers are generated based on the schema, you can script this process (e.g., using a LinkML-to-Django generator) to automate updates.

### Setup Instructions
1. Install Python and Django: `pip install django djangorestframework django-cors-headers pyyaml`.
2. Create the Django project and app as described in the artifact comments.
3. Place the files in the specified locations.
4. Run migrations: `python manage.py makemigrations && python manage.py migrate`.
5. Start the server: `python manage.py runserver`.
6. Open `http://localhost:8000/` in a browser to access the editor.

### Notes
- The artifact includes all necessary code within a single `index.html` file to comply with the one-artifact rule, with other files embedded as comments.
- The backend uses SQLite for simplicity; switch to PostgreSQL or another database for production.
- CORS is enabled for development; restrict origins in production.
- The YAML export matches the schema structure but may need minor tweaks (e.g., prefix handling) for strict LinkML compliance.
- For production, secure the `SECRET_KEY`, disable `DEBUG`, and configure proper CORS settings.

This implementation provides a maintainable, schema-driven editor that meets the requirements while keeping the codebase aligned with the LinkML schema for easy updates.