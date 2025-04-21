To add a button that generates an SVG of the diagram in the Software Contract Dependency Visualizer, we need to:

1. Add a new "Export SVG" button to the UI, styled consistently with existing buttons.
2. Implement a function to export the Cytoscape.js graph as an SVG using the `cytoscape-svg` extension.
3. Trigger a download of the SVG file when the button is clicked.
4. Ensure all existing functionality (YAML parsing, node selection, zoom, relayout) remains unchanged.

The `cytoscape-svg` extension allows us to convert the Cytoscape.js canvas to SVG format, preserving nodes, edges, and
styles. We’ll include the extension via CDN, generate the SVG content, and use a downloadable link to save it as a file.

Below is the updated single-page HTML application with the new "Export SVG" button and functionality.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Software Contract Dependency Visualizer</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/cytoscape/3.21.1/cytoscape.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/js-yaml/4.1.0/js-yaml.min.js"></script>
    <script src="https://unpkg.com/cytoscape-svg@0.4.0/cytoscape-svg.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 20px;
            background-color: #f4f4f9;
        }
        #cy {
            width: 100%;
            height: 600px;
            border: 1px solid #ccc;
            background-color: #fff;
        }
        #input-section {
            margin-bottom: 20px;
        }
        #yaml-input {
            width: 100%;
            height: 100px;
            margin-top: 10px;
        }
        #controls {
            margin-bottom: 10px;
        }
        button {
            padding: 10px 20px;
            margin: 5px;
            background-color: #007bff;
            color: white;
            border: none;
            cursor: pointer;
            border-radius: 4px;
        }
        button:hover {
            background-color: #0056b3;
        }
        #error {
            color: red;
            margin-top: 10px;
            display: none;
        }
    </style>
</head>
<body>
<div id="input-section">
    <h2>Software Contract Dependency Visualizer</h2>
    <p>Upload a YAML file or paste YAML data representing Software Contracts to visualize the call-chain.</p>
    <input type="file" id="file-input" accept=".yaml,.yml">
    <textarea id="yaml-input" placeholder="Or paste YAML data here..."></textarea>
    <button onclick="visualize()">Visualize</button>
    <div id="error"></div>
</div>
<div id="controls">
    <button onclick="zoomIn()">Zoom In</button>
    <button onclick="zoomOut()">Zoom Out</button>
    <button onclick="relayout()">Relayout</button>
    <button onclick="exportSVG()">Export SVG</button>
</div>
<div id="cy"></div>

<script>
    let cy;

    function initializeCytoscape() {
        // Register cytoscape-svg extension
        if (typeof cytoscape.svg !== 'function') {
            cytoscape.use(window.cytoscapeSvg);
        }

        cy = cytoscape({
            container: document.getElementById('cy'),
            style: [
                {
                    selector: 'node[type="system"]',
                    style: {
                        'background-color': '#007bff',
                        'label': 'data(label)',
                        'color': '#fff',
                        'text-valign': 'center',
                        'text-halign': 'center',
                        'shape': 'rectangle',
                        'width': 'label',
                        'height': 'label',
                        'padding': '10px'
                    }
                },
                {
                    selector: 'node[type="contract"]',
                    style: {
                        'background-color': '#28a745',
                        'label': 'data(label)',
                        'color': '#fff',
                        'text-valign': 'center',
                        'text-halign': 'center',
                        'shape': 'ellipse',
                        'width': 'label',
                        'height': 'label',
                        'padding': '10px'
                    }
                },
                {
                    selector: 'node.selected',
                    style: {
                        'border-width': '3px',
                        'border-color': '#ff0',
                        'background-color': '#ff9900'
                    }
                },
                {
                    selector: 'edge',
                    style: {
                        'width': 2,
                        'line-color': '#666',
                        'target-arrow-color': '#666',
                        'target-arrow-shape': 'triangle',
                        'curve-style': 'bezier'
                    }
                }
            ],
            layout: {
                name: 'cose',
                idealEdgeLength: 100,
                nodeOverlap: 20,
                fit: true,
                padding: 30
            },
            minZoom: 0.1,
            maxZoom: 10
        });

        cy.on('tap click', 'node', function(evt) {
            const node = evt.target;
            const label = node.data('label') || 'Unknown';
            const type = node.data('type') || 'Unknown';
            const pos = evt.position || evt.cyPosition;
            console.log(`Clicked node: ${label} (Type: ${type}) at model position: x=${pos.x}, y=${pos.y}`);

            if (node.hasClass('selected')) {
                node.removeClass('selected');
            } else {
                node.addClass('selected');
            }
        });
    }

    function zoomIn() {
        const currentZoom = cy.zoom();
        const newZoom = Math.min(currentZoom * 1.2, cy.maxZoom());
        cy.zoom(newZoom);
        cy.fit(cy.elements(), 30);
    }

    function zoomOut() {
        const currentZoom = cy.zoom();
        const newZoom = Math.max(currentZoom / 1.2, cy.minZoom());
        cy.zoom(newZoom);
        cy.fit(cy.elements(), 30);
    }

    function relayout() {
        cy.layout({
            name: 'cose',
            idealEdgeLength: 100,
            nodeOverlap: 20,
            fit: true,
            padding: 30
        }).run();
    }

    function exportSVG() {
        if (!cy.elements().length) {
            const errorDiv = document.getElementById('error');
            errorDiv.textContent = 'No diagram to export. Please visualize a graph first.';
            errorDiv.style.display = 'block';
            return;
        }

        try {
            // Generate SVG content
            const svgContent = cy.svg({ full: true });
            
            // Create a downloadable link
            const blob = new Blob([svgContent], { type: 'image/svg+xml' });
            const url = URL.createObjectURL(blob);
            const link = document.createElement('a');
            link.href = url;
            link.download = 'software_contract_diagram.svg';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
            URL.revokeObjectURL(url);
        } catch (e) {
            const errorDiv = document.getElementById('error');
            errorDiv.textContent = `Error exporting SVG: ${e.message}`;
            errorDiv.style.display = 'block';
        }
    }

    function parseContracts(data) {
        const elements = [];
        const contracts = Array.isArray(data) ? data : [data];

        contracts.forEach(contract => {
            elements.push({
                data: {
                    id: contract.contract_id,
                    label: contract.name || 'Unnamed Contract',
                    type: 'contract'
                }
            });

            if (contract.calling_systems) {
                contract.calling_systems.forEach(system => {
                    elements.push({
                        data: {
                            id: system.system_id,
                            label: system.name || 'Unnamed System',
                            type: 'system'
                        }
                    });
                    elements.push({
                        data: {
                            id: `edge-${system.system_id}-${contract.contract_id}`,
                            source: system.system_id,
                            target: contract.contract_id
                        }
                    });
                });
            }

            if (contract.dependent_contracts) {
                contract.dependent_contracts.forEach(depContract => {
                    elements.push({
                        data: {
                            id: depContract.contract_id,
                            label: depContract.name || 'Unnamed Contract',
                            type: 'contract'
                        }
                    });
                    elements.push({
                        data: {
                            id: `edge-${contract.contract_id}-${depContract.contract_id}`,
                            source: contract.contract_id,
                            target: depContract.contract_id
                        }
                    });
                });
            }
        });

        return elements;
    }

    function visualize() {
        const fileInput = document.getElementById('file-input');
        const yamlInput = document.getElementById('yaml-input');
        const errorDiv = document.getElementById('error');

        errorDiv.style.display = 'none';

        const processYaml = (yamlData) => {
            try {
                const data = jsyaml.load(yamlData);
                const elements = parseContracts(data);
                cy.elements().remove();
                cy.add(elements);
                cy.layout({
                    name: 'cose',
                    idealEdgeLength: 100,
                    nodeOverlap: 20,
                    fit: true,
                    padding: 30
                }).run();
            } catch (e) {
                errorDiv.textContent = `Error parsing YAML: ${e.message}`;
                errorDiv.style.display = 'block';
            }
        };

        if (fileInput.files.length > 0) {
            const file = fileInput.files[0];
            const reader = new FileReader();
            reader.onload = (e) => processYaml(e.target.result);
            reader.readAsText(file);
        } else if (yamlInput.value.trim()) {
            processYaml(yamlInput.value);
        } else {
            errorDiv.textContent = 'Please upload a YAML file or paste YAML data.';
            errorDiv.style.display = 'block';
        }
    }

    window.onload = initializeCytoscape;
</script>
</body>
</html>
```

### Changes Made

1. **Added Export SVG Button**:
    - Added a new button to the `#controls` div:
      ```html
      <button onclick="exportSVG()">Export SVG</button>
      ```
    - Styled consistently with existing buttons (same CSS, placed alongside Zoom In, Zoom Out, Relayout).

2. **Included Cytoscape-SVG Extension**:
    - Added the `cytoscape-svg` library via CDN:
      ```html
      <script src="https://unpkg.com/cytoscape-svg@0.4.0/cytoscape-svg.min.js"></script>
      ```
    - Registered the extension in `initializeCytoscape`:
      ```javascript
      if (typeof cytoscape.svg !== 'function') {
          cytoscape.use(window.cytoscapeSvg);
      }
      ```
    - This ensures the `cy.svg()` method is available for SVG export.

3. **Implemented `exportSVG` Function**:
    - Added a new function to generate and download the SVG:
      ```javascript
      function exportSVG() {
          if (!cy.elements().length) {
              const errorDiv = document.getElementById('error');
              errorDiv.textContent = 'No diagram to export. Please visualize a graph first.';
              errorDiv.style.display = 'block';
              return;
          }
 
          try {
              const svgContent = cy.svg({ full: true });
              const blob = new Blob([svgContent], { type: 'image/svg+xml' });
              const url = URL.createObjectURL(blob);
              const link = document.createElement('a');
              link.href = url;
              link.download = 'software_contract_diagram.svg';
              document.body.appendChild(link);
              link.click();
              document.body.removeChild(link);
              URL.revokeObjectURL(url);
          } catch (e) {
              const errorDiv = document.getElementById('error');
              errorDiv.textContent = `Error exporting SVG: ${e.message}`;
              errorDiv.style.display = 'block';
          }
      }
      ```
    - Checks if the graph has elements to avoid exporting an empty canvas.
    - Uses `cy.svg({ full: true })` to export the entire graph, including all nodes, edges, and styles.
    - Creates a downloadable SVG file named `software_contract_diagram.svg` using a `Blob` and temporary `<a>` element.
    - Handles errors by displaying a message in the error div.

4. **Preserved Existing Functionality**:
    - Kept all other functionality (YAML parsing, node selection, zoom, relayout) unchanged.
    - The `visualize`, `parseContracts`, `zoomIn`, `zoomOut`, `relayout`, and `initializeCytoscape` functions remain as
      they were, with the fixed `visualize` logic (corrected `else if` from the previous response).
    - Reused the artifact ID (`cf107c43-700d-4d39-8c26-376f49b98b45`) as this is an update to the existing artifact.

### Testing

To test the new SVG export feature, use the same YAML input:

```yaml
- contract_id: C1
  name: User Authentication
  calling_systems:
    - system_id: S1
      name: Web Frontend
  dependent_contracts:
    - contract_id: C2
      name: Session Management
- contract_id: C2
  name: Session Management
  calling_systems:
    - system_id: S2
      name: API Gateway
  dependent_contracts: [ ]
```

1. **Test SVG Export**:
    - Upload the YAML or paste it, then click "Visualize."
    - Click "Export SVG" and verify that a file named `software_contract_diagram.svg` downloads.
    - Open the SVG in a browser or vector graphics editor (e.g., Inkscape) to confirm it matches the displayed graph (
      nodes, edges, styles).

2. **Test Empty Graph**:
    - Click "Export SVG" before visualizing any graph.
    - Verify the error message: “No diagram to export. Please visualize a graph first.”
    - Check the console for no JavaScript errors.

3. **Test Existing Features**:
    - Confirm node selection (click nodes to highlight), zoom (in/out), and relayout work as before.
    - Test with the YAML input to ensure the graph renders correctly and all interactions are functional.
    - Check the console to ensure no errors occur during visualization or export.

### Notes

- The `cytoscape-svg` extension preserves the graph’s visual styles (colors, shapes, arrows), ensuring the SVG matches
  the canvas.
- The SVG export includes the entire graph (`full: true`), but could be modified to export only the visible viewport if
  needed.
- Error handling ensures the app remains robust if the `cy.svg()` call fails or no graph is loaded.
- The application remains self-contained, runs in any modern browser, and requires no server.
- If specific SVG export requirements (e.g., custom filename, specific styling) are needed, please provide details.

This update adds the requested SVG export functionality while maintaining the visualizer’s existing features. If any
issues arise (e.g., SVG rendering errors, UI concerns), please share details for further refinement.