To enhance the Software Contract Dependency Visualizer by adding control buttons for zooming in, zooming out, and
relaying out the diagram, we need to:

1. Add three buttons to the UI: "Zoom In," "Zoom Out," and "Relayout."
2. Implement JavaScript functions to handle zoom (adjusting the Cytoscape.js zoom level) and relayout (re-running the
   COSE layout).
3. Style the buttons to match the existing UI.
4. Keep all other functionality (YAML parsing, graph construction, node selection, etc.) unchanged.

The node selection bug fix from the previous response (visual highlighting, robust event handling) will be preserved.
The new buttons will improve user interaction by allowing manual control over the graph’s zoom and layout, which is
useful for exploring complex dependency graphs.

Below is the updated single-page HTML application with the new control buttons.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Software Contract Dependency Visualizer</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/cytoscape/3.21.1/cytoscape.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/js-yaml/4.1.0/js-yaml.min.js"></script>
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
</div>
<div id="cy"></div>

<script>
    let cy;

    function initializeCytoscape() {
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
            }
        });

        // Handle node selection with click or tap
        cy.on('tap click', 'node', function(evt) {
            const node = evt.target;
            const label = node.data('label') || 'Unknown';
            const type = node.data('type') || 'Unknown';

            // Toggle selection state
            if (node.hasClass('selected')) {
                node.removeClass('selected');
            } else {
                node.addClass('selected');
            }

            // Log selection for debugging
            console.log(`Selected node: ${label} (Type: ${type})`);
        });
    }

    function zoomIn() {
        const currentZoom = cy.zoom();
        cy.zoom(currentZoom * 1.2); // Increase zoom by 20%
        cy.center(); // Re-center the graph
    }

    function zoomOut() {
        const currentZoom = cy.zoom();
        cy.zoom(currentZoom / 1.2); // Decrease zoom by 20%
        cy.center(); // Re-center the graph
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

    function parseContracts(data) {
        const elements = [];
        const contracts = Array.isArray(data) ? data : [data];

        contracts.forEach(contract => {
            // Add contract node
            elements.push({
                data: {
                    id: contract.contract_id,
                    label: contract.name || 'Unnamed Contract',
                    type: 'contract'
                }
            });

            // Add calling systems and edges
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

            // Add dependent contracts and edges
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

    // Initialize Cytoscape on page load
    window.onload = initializeCytoscape;
</script>
</body>
</html>
```

### Changes Made

1. **Added Control Buttons**:
    - Added a new `<div id="controls">` containing three buttons:
      ```html
      <div id="controls">
          <button onclick="zoomIn()">Zoom In</button>
          <button onclick="zoomOut()">Zoom Out</button>
          <button onclick="relayout()">Relayout</button>
      </div>
      ```
    - Positioned the controls div between the input section and the Cytoscape canvas.

2. **Implemented Zoom Functions**:
    - Added `zoomIn` and `zoomOut` functions:
      ```javascript
      function zoomIn() {
          const currentZoom = cy.zoom();
          cy.zoom(currentZoom * 1.2); // Increase zoom by 20%
          cy.center(); // Re-center the graph
      }
 
      function zoomOut() {
          const currentZoom = cy.zoom();
          cy.zoom(currentZoom / 1.2); // Decrease zoom by 20%
          cy.center(); // Re-center the graph
      }
      ```
    - Each function adjusts the zoom level by 20% (multiplicative for smooth scaling) and re-centers the graph using
      `cy.center()` to keep the content in view.

3. **Implemented Relayout Function**:
    - Added a `relayout` function:
      ```javascript
      function relayout() {
          cy.layout({
              name: 'cose',
              idealEdgeLength: 100,
              nodeOverlap: 20,
              fit: true,
              padding: 30
          }).run();
      }
      ```
    - Re-runs the COSE layout with the same parameters as the initial layout, rearranging nodes to optimize the graph’s
      appearance.

4. **Styled Buttons**:
    - Updated the CSS for `button` to include `border-radius: 4px` for a polished look and `margin: 5px` for spacing in
      the controls div:
      ```css
      button {
          padding: 10px 20px;
          margin: 5px;
          background-color: #007bff;
          color: white;
          border: none;
          cursor: pointer;
          border-radius: 4px;
      }
      ```
    - Added a `#controls` style to ensure proper spacing:
      ```css
      #controls {
          margin-bottom: 10px;
      }
      ```

5. **Preserved Functionality**:
    - Kept all existing functionality (YAML parsing, node selection, graph visualization) unchanged.
    - The node selection logic (fixed in the previous response) remains intact, with visual highlighting and robust data
      handling.
    - Reused the artifact ID (`cf107c43-700d-4d39-8c26-376f49b98b45`) as this is an update to the existing artifact.

### Testing

To test the new controls, use the same YAML input as before:

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

1. Upload the YAML or paste it, then click "Visualize."
2. Use the new buttons:
    - **Zoom In**: Click to zoom in by 20%; the graph should enlarge and stay centered.
    - **Zoom Out**: Click to zoom out by 20%; the graph should shrink and stay centered.
    - **Relayout**: Click to rearrange the nodes using the COSE layout; the graph should reposition while maintaining
      the same nodes and edges.
3. Verify that node selection (clicking nodes to highlight) still works as expected.

### Notes

- The zoom functions use a 20% scaling factor for smooth adjustments, but this can be tweaked (e.g., `* 1.1` for finer
  control) if needed.
- The relayout uses the same COSE layout parameters as the initial visualization to ensure consistency, but different
  layouts (e.g., `dagre` for hierarchical graphs) could be added as options in future updates.
- The buttons are styled to match the existing "Visualize" button, ensuring a cohesive UI.
- The application remains self-contained, runs in any modern browser, and requires no server.

This update adds the requested control buttons while preserving the tool’s functionality for visualizing software
contract dependencies. If further refinements (e.g., specific zoom levels, additional layouts) are needed, please
provide details.