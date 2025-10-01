# Workflow Template Implementation Guide

This document explains every moving part of the **Workflow** template so you can recreate or adapt the same experience in your own tldraw application. It is organized in build order: start by mounting the editor, then add the workflow-specific shapes, UI, interactions, and finally the execution simulator.

## 1. Mounting a Workflow-Aware Editor

1. **Register custom shapes and bindings.** Supply the workflow node and connection shape utilities alongside the connection binding util when mounting `<Tldraw />` so the editor understands how to render and persist these shapes.【F:templates/workflow/src/App.tsx†L22-L25】【F:templates/workflow/src/App.tsx†L74-L99】
2. **Override core UI.** Replace the default toolbar with `WorkflowToolbar`, hide the menu panel, and conditionally hide the style panel unless non-workflow shapes are selected. Inject the on-canvas picker and workflow overlays via the `InFrontOfTheCanvas` slot.【F:templates/workflow/src/App.tsx†L27-L62】
3. **Configure editor behaviour on mount.** Ensure at least one node exists, enable snapping, attach the custom `PointingPort` state node to the select tool, enforce that connection shapes stay below nodes, and prevent transparency changes on workflow shapes.【F:templates/workflow/src/App.tsx†L80-L99】

> **Tip:** Expose `editor` on `window` (as in the template) to make debugging custom interactions easier during development.【F:templates/workflow/src/App.tsx†L80-L83】

## 2. Workflow Toolbar and Menu Overrides

1. **Populate workflow tools.** Register a bespoke tool per node definition inside the `tools` override. Each entry creates a node at the viewport center when selected or supports drag-to-create using tldraw’s helper, then closes the math menu.【F:templates/workflow/src/components/WorkflowToolbar.tsx†L65-L98】
2. **Vertical toolbar composition.** The rendered toolbar groups selection, workflow nodes (including a math menu and specialized slider/conditional/earthquake tools), and standard drawing primitives to keep node creation prominent.【F:templates/workflow/src/components/WorkflowToolbar.tsx†L101-L140】
3. **Node creation helper.** `createNodeShape` wraps creation in a history mark, instantiates the shape with the chosen node props, recenters it on the drop position, and selects it so the user can immediately interact.【F:templates/workflow/src/components/WorkflowToolbar.tsx†L37-L63】

## 3. Node Shape System

1. **Shape metadata.** `NodeShapeUtil` declares the `'node'` type with props `{ node, isOutOfDate }`, disables native resizing/rotation controls, and returns geometry consisting of the body rectangle plus circle geometries for each port so handles can avoid the bounds.【F:templates/workflow/src/nodes/NodeShapeUtil.tsx†L24-L108】
2. **Rendering.** The component fetches live output values and execution status, toggling a CSS class while the execution engine is running. It draws the node header, optional output readout, and renders the per-type body via `NodeBody` (covered below).【F:templates/workflow/src/nodes/NodeShapeUtil.tsx†L151-L189】
3. **Indicator.** The indicator masks ports out of the selection highlight and draws visible port markers, improving feedback while nodes are selected.【F:templates/workflow/src/nodes/NodeShapeUtil.tsx†L111-L149】
4. **Node definitions.** Each node type extends the abstract `NodeDefinition` base, which prescribes titles, icons, port layouts, output computation, async execution, and the React body component. Definitions are cached per editor instance and retrieved by helper functions such as `getNodeDefinition`, `getNodeTypePorts`, and `executeNode`.【F:templates/workflow/src/nodes/types/shared.tsx†L23-L120】【F:templates/workflow/src/nodes/nodeTypes.tsx†L1-L91】【F:templates/workflow/src/nodes/nodeTypes.tsx†L92-L140】
5. **Shared body helpers.** Components like `NodeRow` and `NodeInputRow` handle editable inputs that become read-only when wired to upstream data, while `NodeValue` formats live output values or displays placeholders for halted branches.【F:templates/workflow/src/nodes/types/shared.tsx†L73-L197】

## 4. Ports and Wiring State

1. **Port description & caching.** Port geometry and connections are cached per node via `createComputedCache`, producing fast lookups for drawing, hit testing, and data propagation. Inputs pull values from connected outputs, and outputs synthesize values (including `STOP_EXECUTION` markers) based on the node definition logic.【F:templates/workflow/src/nodes/nodePorts.tsx†L1-L126】
2. **Graph traversal.** `getAllConnectedNodes` walks the workflow graph for cycle detection and region grouping, operating on cached connection data.【F:templates/workflow/src/nodes/nodePorts.tsx†L128-L156】
3. **Port component.** Renders interactive port dots, highlights them when eligible during drag operations, and routes pointer-down events into the custom `PointingPort` state node to start wiring interactions.【F:templates/workflow/src/ports/Port.tsx†L1-L89】
4. **Port UI state.** `portState` is stored per-editor using `EditorAtom`. `updatePortState` centralizes updates so drag interactions can highlight eligible or hinted ports across the canvas.【F:templates/workflow/src/ports/portState.ts†L1-L28】【F:templates/workflow/src/utils.ts†L12-L41】
5. **Hit testing.** `getPortAtPoint` locates the nearest port at a pointer position (respecting terminal direction and optional margin), returning both geometry and existing connections so the drag logic can enforce single-input rules.【F:templates/workflow/src/ports/getPortAtPoint.tsx†L1-L36】【F:templates/workflow/src/ports/getPortAtPoint.tsx†L37-L49】
6. **Pointing tool.** `PointingPort` extends tldraw’s state machine: dragging from an output creates a new connection (or reuses an existing one when inputs are single-use), while clicking spawns a new node to the right and prewires it through the on-canvas picker. Its `onPointerMove` and `onClick` routines manage connection creation, picker launch, and cleanup if the user cancels.【F:templates/workflow/src/ports/PointingPort.tsx†L1-L205】

## 5. Connection Shape Behaviour

1. **Shape utility.** `ConnectionShapeUtil` defines bezier geometry, exposes handles at each terminal, disables snapping so edges don’t interfere with layout, and consults bindings to resolve terminal positions in real time.【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L39-L130】【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L397-L421】
2. **Handle interactions.** Dragging a handle queries nearby ports, highlights valid targets, prevents cycles by inspecting connected nodes, and updates bindings when the pointer enters or leaves a port. Dropping on empty space opens the on-canvas picker for the end terminal or deletes the wire if it is no longer bound.【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L132-L205】【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L205-L275】
3. **Center handle.** A zoom-dependent affordance appears at the midpoint of fully bound connections; clicking it opens the picker in “middle” mode so a new node can be inserted into the wire via `insertNodeWithinConnection`.【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L282-L377】
4. **Visual feedback.** Connections whose upstream node outputs `STOP_EXECUTION` render with an inactive style, making conditional branches obvious.【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L297-L329】
5. **Binding helpers.** `ConnectionBindingUtil` keeps connection endpoints synchronized with ports, invokes node-level `onPortConnect/onPortDisconnect` callbacks, and exposes utilities to query or mutate bindings. This underpins both drag interactions and programmatic graph edits.【F:templates/workflow/src/connection/ConnectionBindingUtil.tsx†L20-L193】
6. **Z-order maintenance.** `keepConnectionsAtBottom` listens to shape mutations and continually reindexes connections so they stay below nodes, avoiding wires overlapping node UIs.【F:templates/workflow/src/connection/keepConnectionsAtBottom.tsx†L1-L99】
7. **Insertion workflow.** `insertNodeWithinConnection` drives the “add node mid-wire” story: it shows the picker, creates and positions the new node, reconnects bindings, nudges downstream nodes to make room, and animates the layout update.【F:templates/workflow/src/connection/insertNodeWithinConnection.tsx†L1-L168】

## 6. On-Canvas Component Picker

1. **State storage.** Picker visibility and callbacks live in an `EditorAtom`, scoped per editor instance.【F:templates/workflow/src/components/OnCanvasComponentPicker.tsx†L23-L33】
2. **Dialog positioning.** A headless Radix dialog is anchored to the connection terminal (start, end, or midpoint) by transforming coordinates from connection space → page space → viewport space inside a `useQuickReactor` hook so it tracks camera movements in real time.【F:templates/workflow/src/components/OnCanvasComponentPicker.tsx†L61-L135】
3. **Menu contents.** The picker lists math and logic node definitions pulled from the same registry used by the toolbar, ensuring parity between creation entry points.【F:templates/workflow/src/components/OnCanvasComponentPicker.tsx†L35-L58】
4. **Selection handling.** Choosing a node calls the stored `onPick`, which creates the node at the terminal position and rebinds the originating connection (or connection pair) to the new ports before closing the dialog.【F:templates/workflow/src/components/OnCanvasComponentPicker.tsx†L138-L185】

## 7. Workflow Execution Simulator

1. **Execution graph.** When the user presses Play, `ExecutionGraph` snapshots the subgraph reachable from the selected starting nodes, tracks per-node state (`waiting/executing/executed`), and recursively executes nodes once their dependencies have produced values. STOP_EXECUTION signals block downstream execution paths.【F:templates/workflow/src/execution/ExecutionGraph.tsx†L1-L167】
2. **State management.** `executionState` keeps the currently running graph inside an `EditorAtom`, stopping any in-progress run before starting a new one and clearing the reference when finished.【F:templates/workflow/src/execution/executionState.ts†L1-L49】
3. **Canvas overlays.** `WorkflowRegions` clusters connected nodes into regions, draws an overlay bounding box, and exposes play/stop controls tied to the execution state. It hides overlays when zoomed out and live-updates positioning as the camera moves.【F:templates/workflow/src/components/WorkflowRegions.tsx†L1-L174】
4. **Node feedback.** During execution, `NodeShape` sets `isOutOfDate` while work is in flight so the UI can dim outputs, and reverts once the node finishes. Connections query this flag to apply inactive styling where appropriate.【F:templates/workflow/src/nodes/NodeShapeUtil.tsx†L151-L189】【F:templates/workflow/src/connection/ConnectionShapeUtil.tsx†L297-L329】

## 8. Supporting Utilities and Styling Contracts

1. **Geometry constants.** Shared dimensions (node width, header height, port radius, spacing) live in `constants.tsx` so shapes, overlays, and insertion logic remain aligned.【F:templates/workflow/src/constants.tsx†L1-L12】
2. **Opacity guard.** `disableTransparency` registers side effects that clamp `opacity` to 1 for workflow shapes, ensuring nodes and connections stay visually legible even if style panel changes slip through.【F:templates/workflow/src/disableTransparency.tsx†L1-L17】
3. **EditorAtom helper.** A thin wrapper over tldraw’s `atom` API creates per-editor reactive stores, used for the picker, port state, and execution status without leaking across editor instances.【F:templates/workflow/src/utils.ts†L1-L41】
4. **Index utilities.** Additional helpers in `utils.ts` manage fractional index lists, mirroring how tldraw orders shapes; these are useful if you need to create ordered collections tied to editor state.【F:templates/workflow/src/utils.ts†L44-L85】

---

By following the steps above—mounting the editor with workflow overrides, defining node and connection behaviour, wiring ports with custom state, surfacing the on-canvas picker, and layering on the execution simulator—you can reproduce the full workflow experience provided by the template or tailor it for your own node-graph applications.
