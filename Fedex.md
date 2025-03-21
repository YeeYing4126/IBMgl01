<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>FedEx-Delivery-Process</title>
    <script src="https://cdn.interactjs.io/v1.10.11/interact.min.js"></script>
    <script src="https://html2canvas.hertzen.com/dist/html2canvas.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <style>
        body {
            margin: 0;
            padding: 20px;
            font-family: Arial, sans-serif;
        }
        
        #toolbar {
            margin-bottom: 20px;
            display: flex;
            gap: 10px;
        }
        
        .canvas-container {
            display: flex;
        }
        
        #nodes-panel {
            width: 150px;
            padding: 10px;
            background: #f0f0f0;
            border-radius: 5px;
            margin-right: 20px;
        }
        
        .node-item {
            padding: 10px;
            margin: 5px 0;
            background: white;
            border: 1px solid #ccc;
            cursor: move;
            border-radius: 4px;
        }
        
        #canvas {
            flex-grow: 1;
            min-height: 600px;
            border: 2px dashed #ccc;
            position: relative;
        }
        
        .flow-node {
            position: absolute;
            width: 120px;
            padding: 15px;
            background: #fff;
            border: 2px solid #4CAF50;
            border-radius: 8px;
            text-align: center;
            cursor: move;
        }
        
        .connector {
            width: 10px;
            height: 10px;
            background: #666;
            position: absolute;
            bottom: -5px;
            left: 50%;
            transform: translateX(-50%);
            border-radius: 50%;
            cursor: crosshair;
        }
        
        .delete-btn {
            position: absolute;
            top: 2px;
            right: 2px;
            cursor: pointer;
            color: red;
        }
        
        .connection {
            position: absolute;
            pointer-events: none;
        }
    </style>
</head>
<body>
    <div id="toolbar">
        <button onclick="exportPNG()">Export PNG</button>
        <button onclick="undo()">Undo</button>
        <button onclick="redo()">Redo</button>
        <input type="color" id="nodeColor" onchange="changeNodeColor(this.value)">
    </div>

    <div class="canvas-container">
        <div id="nodes-panel">
            <div class="node-item" data-type="Supplier">Supplier</div>
            <div class="node-item" data-type="Warehouse">Warehouse</div>
            <div class="node-item" data-type="Distribution">Distribution</div>
            <div class="node-item" data-type="Retail">Retail</div>
            <div class="node-item" data-type="Transport">Transport</div>
        </div>
        <div id="canvas"></div>
    </div>

    <script>
        let nodes = [];
        let connections = [];
        let history = [];
        let currentHistoryIndex = -1;
        let draggingConnection = null;
        let selectedNode = null;

        // Initialize interact.js
        interact('.node-item').draggable({
            onstart: (event) => {
                const type = event.target.getAttribute('data-type');
                createNode(type, event.clientX, event.clientY);
            }
        });

        function createNode(type, x, y) {
            const nodeId = Date.now();
            const node = document.createElement('div');
            node.className = 'flow-node';
            node.id = `node-${nodeId}`;
            node.innerHTML = `
                <span class="delete-btn" onclick="deleteNode(${nodeId})">×</span>
                <div class="connector" data-node="${nodeId}"></div>
                <div contenteditable="true">${type}</div>
            `;
            
            node.style.left = `${x - 60}px`;
            node.style.top = `${y - 20}px`;
            
            document.getElementById('canvas').appendChild(node);
            nodes.push({ id: nodeId, element: node, x, y });
            
            makeDraggable(node);
            setupConnector(node.querySelector('.connector'));
            saveState();
        }

        function makeDraggable(element) {
            interact(element).draggable({
                onmove: (event) => {
                    const target = event.target;
                    const x = (parseFloat(target.style.left) || 0) + event.dx;
                    const y = (parseFloat(target.style.top) || 0) + event.dy;
                    target.style.left = `${x}px`;
                    target.style.top = `${y}px`;
                    updateConnections();
                }
            });
        }

        function setupConnector(connector) {
            interact(connector).draggable({
                onstart: (event) => {
                    draggingConnection = {
                        startNode: event.target.getAttribute('data-node'),
                        element: document.createElement('div'),
                        x1: event.clientX,
                        y1: event.clientY
                    };
                    draggingConnection.element.className = 'connection';
                    document.getElementById('canvas').appendChild(draggingConnection.element);
                },
                onmove: (event) => {
                    if (!draggingConnection) return;
                    const canvas = document.getElementById('canvas').getBoundingClientRect();
                    const x2 = event.clientX - canvas.left;
                    const y2 = event.clientY - canvas.top;
                    drawConnectionLine(draggingConnection.element, 
                        draggingConnection.x1 - canvas.left, 
                        draggingConnection.y1 - canvas.top, 
                        x2, y2);
                },
                onend: (event) => {
                    if (!draggingConnection) return;
                    const endConnector = document.elementFromPoint(event.clientX, event.clientY);
                    if (endConnector && endConnector.classList.contains('connector')) {
                        createConnection(draggingConnection.startNode, 
                            endConnector.getAttribute('data-node'));
                    }
                    draggingConnection.element.remove();
                    draggingConnection = null;
                }
            });
        }

        function createConnection(startId, endId) {
            const connection = {
                start: startId,
                end: endId,
                element: document.createElement('div')
            };
            connection.element.className = 'connection';
            connections.push(connection);
            document.getElementById('canvas').appendChild(connection.element);
            updateConnections();
            saveState();
        }

        function updateConnections() {
            connections.forEach(conn => {
                const startNode = nodes.find(n => n.id == conn.start);
                const endNode = nodes.find(n => n.id == conn.end);
                if (startNode && endNode) {
                    const startRect = startNode.element.getBoundingClientRect();
                    const endRect = endNode.element.getBoundingClientRect();
                    const canvasRect = document.getElementById('canvas').getBoundingClientRect();
                    drawConnectionLine(conn.element,
                        startRect.left + startRect.width/2 - canvasRect.left,
                        startRect.bottom - canvasRect.top,
                        endRect.left + endRect.width/2 - canvasRect.left,
                        endRect.top - canvasRect.top
                    );
                }
            });
        }

        function drawConnectionLine(element, x1, y1, x2, y2) {
            const length = Math.sqrt((x2-x1)*(x2-x1) + (y2-y1)*(y2-y1));
            const angle = Math.atan2(y2 - y1, x2 - x1);
            element.style.width = `${length}px`;
            element.style.left = `${x1}px`;
            element.style.top = `${y1}px`;
            element.style.transform = `rotate(${angle}rad)`;
            element.style.borderBottom = '2px solid #333';
        }

        function deleteNode(nodeId) {
            const nodeIndex = nodes.findIndex(n => n.id === nodeId);
            if (nodeIndex > -1) {
                nodes[nodeIndex].element.remove();
                nodes.splice(nodeIndex, 1);
                connections = connections.filter(conn => 
                    conn.start !== nodeId && conn.end !== nodeId);
                updateConnections();
                saveState();
            }
        }

        function saveState() {
            history = history.slice(0, currentHistoryIndex + 1);
            history.push({
                nodes: nodes.map(n => ({ ...n, element: null })),
                connections: [...connections]
            });
            currentHistoryIndex++;
        }

        function undo() {
            if (currentHistoryIndex > 0) {
                currentHistoryIndex--;
                restoreState();
            }
        }

        function redo() {
            if (currentHistoryIndex < history.length - 1) {
                currentHistoryIndex++;
                restoreState();
            }
        }

        function restoreState() {
            const state = history[currentHistoryIndex];
            // Clear current state
            nodes.forEach(n => n.element.remove());
            connections.forEach(c => c.element.remove());
            
            // Restore nodes
            nodes = state.nodes.map(n => {
                const node = document.createElement('div');
                // ... recreation logic ...
                return { ...n, element: node };
            });
            
            // Restore connections
            connections = state.connections.map(conn => {
                // ... recreation logic ...
            });
            
            updateConnections();
        }

        function exportPNG() {
            html2canvas(document.querySelector("#canvas")).then(canvas => {
                const link = document.createElement('a');
                link.download = 'flowchart.png';
                link.href = canvas.toDataURL();
                link.click();
            });
        }

        function changeNodeColor(color) {
            if (selectedNode) {
                selectedNode.style.backgroundColor = color;
                saveState();
            }
        }

        // Initialize node selection
        document.addEventListener('click', (e) => {
            selectedNode = e.target.closest('.flow-node');
        });
    </script>
    
    <footer>
        <p>Contact me at: <a href="mailto:averihere4126@gmail.com">averihere4126@gmail.com</a></p>
    </footer>
</body>
</html>
