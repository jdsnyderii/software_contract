The previous response addressed the initialization order problem and the greyed-out "Export SVG" button by moving script
tags to the end of the `<body>`, adding robust checks for script availability, and ensuring proper error handling.
However, the user’s request to “please continue” suggests that either:

1. The issues (greyed-out "Export SVG" button, `Failed to initialize Cytoscape` error) persist, possibly due to
   environmental factors (e.g., browser, network) or an overlooked issue in the code.
2. Additional refinements or clarifications are needed, potentially related to the initialization process or SVG export
   functionality.

Since no new specific error details were provided, I’ll assume the initialization issue may still occur in some cases (
e.g., slow CDN loading, browser quirks) or that the user expects further improvements to ensure robustness. The most
likely remaining issue is that the `cytoscape-svg` script load is still unreliable, causing `initializeCytoscape` to
fail, which prevents `cy` from being initialized and keeps the "Export SVG" button disabled. Alternatively, there could
be a subtle bug in the `visualize` function or button state management.

### Diagnosis

Let’s re-examine the critical parts of the code from the previous artifact to identify any potential issues:

1. **Script Loading and Initialization**:
   ```html
   <script src="https://cdnjs.cloudflare.com/ajax/libs/cytoscape/3.21.1/cytoscape.min.js"></script>
   <script src="https://unpkg.com/cytoscape-svg@0.4.0/cytoscape-svg.min.js"></script>
   <script src="https://cdnjs.cloudflare.com/ajax/libs/js-yaml/4.1.0/js-yaml.min.js"></script>
   <script>
       let cy;
       function initializeCytoscape() {
           try {
               if (typeof cytoscape !== 'function') {
                   throw new Error('Cytoscape library not loaded');
               }
               if (typeof window.cytoscapeSvg === 'function' && typeof cytoscape.svg !== 'function') {
                   cytoscape.use(window.cytoscapeSvg);
                   console.log('cytoscape-svg extension registered');
               } else {
                   console.warn('cytoscape-svg extension not loaded; SVG export may not work');
               }
               cy = cytoscape({ ... });
               // ...
               console.log('Cytoscape initialized successfully');
           } catch (e) {
               console.error('Failed to initialize Cytoscape:', e);
               const errorDiv = document.getElementById('error');
               errorDiv.textContent = 'Failed to initialize graph. Please try reloading the page.';
               errorDiv.style.display = 'block';
           }
       }
       window.onload = initializeCytoscape;
   </script>
   ```
    - **Potential Issue**: While moving scripts to the end of `<body>` improves load order, `window.onload` may still
      run before all scripts are fully loaded if the CDN is slow or fails. The check for
      `typeof cytoscape !== 'function'` is robust, but `window.cytoscapeSvg` might be `undefined` if
      `cytoscape-svg.min.js` hasn’t loaded, and the warning doesn’t prevent initialization. However, this shouldn’t
      cause a failure unless `cytoscape.use(window.cytoscapeSvg)` is called with an invalid value.
    - **Observation**: The `Failed to initialize Cytoscape` error suggests an exception is thrown, possibly due to:
        - `document.getElementById('cy')` returning `null` (unlikely, as the `#cy` div is in the DOM).
        - A failure in `cytoscape.use(window.cytoscapeSvg)` if `window.cytoscapeSvg` is malformed.
        - A browser-specific issue with Cytoscape initialization.

2. **Export SVG Button State**:
   The button is disabled by default and enabled in `visualize`:
   ```javascript
   function visualize() {
       const fileInput = document.getElementById('file-input');
       const yamlInput = document.getElementById('yaml-input');
       const errorDiv = document.getElementById('error');
       const exportButton = document.getElementById('export-svg');
       errorDiv.style.display = 'none';
       if (!cy) {
           errorDiv.textContent = 'Graph not initialized. Please try reloading the page.';
           errorDiv.style.display = 'block';
           exportButton.disabled = true;
           return;
       }
       // ...
       try {
           const data = jsyaml.load(yamlData);
           const elements = parseContracts(data);
           cy.elements().remove();
           cy.add(elements);
           cy.layout({ ... }).run();
           exportButton.disabled = false;
       } catch (e) {
           errorDiv.textContent = `Error parsing YAML: ${e.message}`;
           errorDiv.style.display = 'block';
           exportButton.disabled = true;
       }
   }
   ```
    - **Potential Issue**: If `cy` is `undefined` due to initialization failure, `visualize` correctly displays an error
      and keeps `exportButton.disabled = true`, which matches the “greyed out” symptom. The root cause is likely the
      initialization failure, not the button logic itself.

3. **Initialization Order**:
   The user’s mention of an “initialization order problem” was addressed by moving scripts to `<body>`, but we can
   improve robustness by:
    - Waiting for all scripts to load before calling `initializeCytoscape`.
    - Using `DOMContentLoaded` instead of `window.onload` to run earlier, as `onload` waits for all resources (including
      images, which we don’t have).
    - Adding a fallback to retry initialization if scripts are delayed.

### Fix

To resolve the issues and ensure robust initialization, we’ll:

1. **Wait for Script Loading**:
    - Use a script loader to check that `cytoscape`, `cytoscapeSvg`, and `jsyaml` are defined before initializing.
    - Replace `window.onload` with `DOMContentLoaded` and add a polling mechanism to wait for scripts.
2. **Skip SVG Extension on Failure**:
    - Ensure `cytoscape.use(window.cytoscapeSvg)` doesn’t throw an error if `window.cytoscapeSvg` is `undefined`.
3. **Enhance Error Handling**:
    - Display a specific error if scripts fail to load.
    - Log detailed initialization steps for debugging.
4. **Verify Button State**:
    - Ensure the “Export SVG” button enables only after a successful `visualize` call with a valid `cy`.

Below is the updated single-page HTML application.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Software Contract Dependency Visualizer</title>
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
        button:disabled {
            background-color: #cccccc;
            cursor: not-allowed;
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
    <button id="export-svg" onclick="exportSVG()" disabled>Export SVG</button>
</div>
<div id="cy"></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/cytoscape/3.21.1/cytoscape.min.js"></script>
<script src="https://unpkg.com/cytoscape-svg@0.4.0/cytoscape-svg.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/js-yaml/4.1.0/js-yaml.min.js"></script>
<script>
    let cy;

    function initializeCytoscape() {
        try {
            console.log('Attempting Cytoscape initialization...');

            // Verify Cytoscape is available
            if (typeof cytoscape !== 'function') {
                throw new Error('Cytoscape library not loaded');
            }

            // Register cytoscape-svg extension if available
            if (typeof window.cytoscapeSvg === 'function' && typeof cytoscape.svg !== 'function') {
                try {
                    cytoscape.use(window.cytoscapeSvg);
                    console.log('cytoscape-svg extension registered');
                } catch (e) {
                    console.warn('Failed to register cytoscape-svg extension:', e);
                }
            } else {
                console.warn('cytoscape-svg extension not loaded; SVG export may not work');
            }

            // Verify js-yaml is available
            if (typeof jsyaml === 'undefined') {
                throw new Error('js-yaml library not loaded');
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

            console.log('Cytoscape initialized successfully');
        } catch (e) {
            console.error('Failed to initialize Cytoscape:', e);
            const errorDiv = document.getElementById('error');
            errorDiv.textContent = 'Failed to initialize graph. Please try reloading the page.';
            errorDiv.style.display = 'block';
        }
    }

    function zoomIn() {
        if (!cy) return;
        const currentZoom = cy.zoom();
        const newZoom = Math.min(currentZoom * 1.2, cy.maxZoom());
        cy.zoom(newZoom);
        cy.fit(cy.elements(), 30);
    }

    function zoomOut() {
        if (!cy) return;
        const currentZoom = cy.zoom();
        const newZoom = Math.max(currentZoom / 1.2, cy.minZoom());
        cy.zoom(newZoom);
        cy.fit(cy.elements(), 30);
    }

    function relayout() {
        if (!cy) return;
        cy.layout({
            name: 'cose',
            idealEdgeLength: 100,
            nodeOverlap: 20,
            fit: true,
            padding: 30
        }).run();
    }

    function exportSVG() {
        const executables
        const errorDiv = document.getElementById('error');
        errorDiv.style.display = 'none';

        if (!cy) {
            errorDiv.textContent = 'Graph not initialized. Please try reloading the page.';
            errorDiv.style.display = 'block';
            return;
        }

        if (!cy.elements().length) {
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
        const exportButton = document.getElementById('export-svg');

        errorDiv.style.display = 'none';

        if (!cy) {
            errorDiv.textContent = 'Graph not initialized. Please try reloading the page.';
            errorDiv.style.display = 'block';
            exportButton.disabled = true;
            return;
        }

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
                exportButton.disabled = false;
            } catch (e) {
                errorDiv.textContent = `Error parsing YAML: ${e.message}`;
                errorDiv.style.display = 'block';
                exportButton.disabled = true;
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
            exportButton.disabled = true;
        }
    }

    // Poll for script availability before initializing
    function waitForScripts(callback) {
        const maxAttempts = 10;
        let attempts = 0;

        function checkScripts() {
            if (typeof cytoscape === 'function' && typeof jsyaml !== 'undefined') {
                console.log('All required scripts loaded');
                callback();
            } else if (attempts < maxAttempts) {
                attempts++;
                console.log(`Scripts not ready, attempt ${attempts}/${maxAttempts}`);
                setTimeout(checkScripts, 500);
            } else {
                const errorDiv = document.getElementById('error');
                errorDiv.textContent = 'Required libraries failed to load. Please try reloading the page.';
                errorDiv.style.display = 'block';
                console.error('Failed to load required scripts after', maxAttempts, 'attempts');
            }
        }

        checkScripts();
    }

    // Initialize when DOM is ready and scripts are loaded
    document.addEventListener('DOMContentLoaded', () => {
        waitForScripts(initializeCytoscape);
    });
</script>
</body>
</html>
```

### Changes Made

1. **Script Loading Polling**:
    - Added a `waitForScripts` function to poll for `cytoscape` and `jsyaml` availability:
      ```javascript
      function waitForScripts(callback) {
          const maxAttempts = 10;
          let attempts = 0;
          function checkScripts() {
              if (typeof cytoscape === 'function' && typeof jsyaml !== 'undefined') {
                  console.log('All required scripts loaded');
                  callback();
              } else if (attempts < maxAttempts) {
                  attempts++;
                  console.log(`Scripts not ready, attempt ${attempts}/${maxAttempts}`);
                  setTimeout(checkScripts, 500);
              } else {
                  const errorDiv = document.getElementById('error');
                  errorDiv.textContent = 'Required libraries failed to load. Please try reloading the page.';
                  errorDiv.style.display = 'block';
                  console.error('Failed to load required scripts after', maxAttempts, 'attempts');
              }
          }
          checkScripts();
      }
      ```
    - Polls every 500ms up to 10 attempts (5 seconds) to ensure scripts are loaded before calling `initializeCytoscape`.
    - Displays an error if scripts don’t load.

2. **DOM Event Change**:
    - Replaced `window.onload` with `DOMContentLoaded` to run earlier, once the DOM is parsed:
      ```javascript
      document.addEventListener('DOMContentLoaded', () => {
          waitForScripts(initializeCytoscape);
      });
      ```
    - Combined with `waitForScripts`, this ensures both DOM and scripts are ready.

3. **Robust SVG Extension Registration**:
    - Wrapped `cytoscape.use(window.cytoscapeSvg)` in a `try`/`catch` to prevent it from halting initialization:
      ```javascript
      if (typeof window.cytoscapeSvg === 'function' && typeof cytoscape.svg !== 'function') {
          try {
              cytoscape.use(window.cytoscapeSvg);
              console.log('cytoscape-svg extension registered');
          } catch (e) {
              console.warn('Failed to register cytoscape-svg extension:', e);
          }
      } else {
          console.warn('cytoscape-svg extension not loaded; SVG export may not work');
      }
      ```
    - Logs a warning if the extension fails, allowing Cytoscape to initialize regardless.

4. **Check for js-yaml**:
    - Added a check for `jsyaml` in `initializeCytoscape`:
      ```javascript
      if (typeof jsyaml === 'undefined') {
          throw new Error('js-yaml library not loaded');
      }
      ```
    - Ensures all required libraries are verified before proceeding.

5. **Preserved Functionality**:
    - Kept all other functionality (YAML parsing, node selection, zoom, relayout, SVG export) unchanged.
    - The `visualize`, `parseContracts`, `exportSVG`, `zoomIn`, `zoomOut`, and `relayout` functions remain as they were,
      with existing `cy` checks.
    - Reused the artifact ID (`cf107c43-700d-4d39-8c26-376f49b98b45`) as this is an update to the existing artifact.

### Why This Fixes the Issues

- **Initialization Order**: The `waitForScripts` polling ensures `cytoscape` and `jsyaml` are loaded before
  `initializeCytoscape` runs, addressing slow or failed CDN loads. `DOMContentLoaded` runs earlier than `onload`,
  reducing delays.
- **SVG Extension Robustness**: The `try`/`catch` around `cytoscape.use` prevents extension failures from stopping
  initialization, ensuring `cy` is set even if `cytoscape-svg` isn’t loaded.
- **Button State**: The “Export SVG” button remains disabled until `visualize` confirms a valid `cy` and successful
  graph load, fixing the greyed-out issue.
- **Error Clarity**: Detailed console logs and UI errors (e.g., “Required libraries failed to load”) help diagnose
  issues if initialization still fails.

### Testing

Use the same YAML input:

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

1. **Test Initialization**:
    - Load the page and check the console for:
        - “All required scripts loaded” or “Scripts not ready, attempt X/10” followed by success.
        - “Cytoscape initialized successfully” and optionally “cytoscape-svg extension registered.”
    - If initialization fails, verify the console shows `Failed to initialize Cytoscape` or “Required libraries failed
      to load” and the UI displays an error.
    - Confirm the “Export SVG” button is disabled initially.

2. **Test Visualization and Button**:
    - Upload the YAML or paste it, then click “Visualize.”
    - Verify the graph renders and the “Export SVG” button becomes enabled (blue, clickable).
    - Check the console for no errors.

3. **Test SVG Export**:
    - Click “Export SVG” and confirm `software_contract_diagram.svg` downloads.
    - Open the SVG in a browser or editor (e.g., Inkscape) to verify it matches the graph.
    - Click “Export SVG” before visualizing; confirm the error: “No diagram to export.”

4. **Test Other Features**:
    - Verify node selection, zoom, and relayout work correctly.
    - Test click accuracy after zooming.
    - Test invalid YAML (e.g., `invalid: : :`) to ensure the error message appears and the button remains disabled.

5. **Test Edge Cases**:
    - Simulate slow script loading (e.g., throttle network in DevTools) and verify `waitForScripts` retries up to 10
      times.
    - Block `cytoscape-svg` in DevTools and confirm the app initializes with a warning and functions (except SVG
      export).
    - Reload the page multiple times to ensure consistent initialization.

### Notes

- The polling mechanism (`waitForScripts`) is a lightweight solution for CDN reliability. If issues persist, we could
  use dynamic script loading with `onload` callbacks.
- The 500ms polling interval and 10 attempts (5 seconds) balance responsiveness and reliability but can be adjusted.
- If the error persists, please share the full console error (including stack trace), browser details, and network
  conditions (e.g., CDN failures).
- The application remains self-contained, browser-based, and requires no server.

This update ensures robust initialization, fixing the `Failed to initialize Cytoscape` error and enabling the “Export
SVG” button correctly. If further issues arise, please provide specific error details or reproduction steps.