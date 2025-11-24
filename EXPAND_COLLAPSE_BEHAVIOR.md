# Tree Node Expand/Collapse Behavior Analysis

## Overview

This document explains how expand and collapse works with initial open state for sibling-level tree nodes with children, and how opening/closing one sibling affects other siblings.

## Key Finding: Sibling Independence

**Sibling nodes operate completely independently.** Opening or closing one sibling node does NOT affect the open/closed state of other sibling nodes. Each node's state is managed independently through the `openIds` array.

## Architecture

### 1. State Management (`useOpenIdsHelper` hook)

The tree's open/closed state is managed by the `useOpenIdsHelper` hook in `src/hooks/useOpenIdsHelper.ts`:

```typescript
const [openIds, setOpenIds] = useState<NodeModel["id"][]>(initialOpenIds);
```

- `openIds` is an array of node IDs that are currently open
- Each node's open state is determined by checking if its ID exists in this array
- Sibling nodes share no state dependencies - they're just different IDs in the same array

### 2. Initial Open State (`initialOpen` prop)

The `initialOpen` prop controls which nodes are open when the tree first renders:

**Location**: `src/hooks/useOpenIdsHelper.ts` lines 28-38

```typescript
const initialOpenIds: NodeModel["id"][] = useMemo(() => {
  if (initialOpen === true) {
    // Open ALL parent nodes (nodes with children)
    return tree
      .filter((node) => hasChildNodes(tree, node.id))
      .map((node) => node.id);
  } else if (Array.isArray(initialOpen)) {
    // Open only the specified node IDs
    return initialOpen;
  }
  // No nodes open initially
  return [];
}, [initialOpen]);
```

**Three modes:**
1. **`initialOpen: true`** - Opens all parent nodes (nodes that have children)
2. **`initialOpen: [id1, id2, ...]`** - Opens only the specified node IDs
3. **`initialOpen: false` or `undefined`** - No nodes open initially

**Important**: When `initialOpen` prop changes, the tree state resets (line 42):
```typescript
useEffect(() => setOpenIds(initialOpenIds), [initialOpenIds]);
```

### 3. Toggle Behavior (`handleToggle`)

**Location**: `src/hooks/useOpenIdsHelper.ts` lines 44-54

```typescript
const handleToggle: ToggleHandler = (targetId: NodeModel["id"], callback) => {
  const newOpenIds = openIds.includes(targetId)
    ? openIds.filter((id) => id !== targetId)  // Remove if already open
    : [...openIds, targetId];                   // Add if closed
  
  setOpenIds(newOpenIds);
  
  if (callback) {
    callback(newOpenIds);
  }
};
```

**Behavior:**
- If node is open → removes its ID from `openIds` (closes it)
- If node is closed → adds its ID to `openIds` (opens it)
- **No sibling interaction** - only the target node's ID is added/removed

### 4. Node Rendering (`Node.tsx`)

**Location**: `src/Node.tsx` lines 30-31, 90-97

```typescript
const open = openIds.includes(props.id);
```

Each node checks if its ID is in `openIds` to determine if it should render its children:

```typescript
{enableAnimateExpand && params.hasChild && (
  <AnimateHeight isVisible={open}>
    <Container parentId={props.id} depth={props.depth + 1} />
  </AnimateHeight>
)}
{!enableAnimateExpand && params.hasChild && open && (
  <Container parentId={props.id} depth={props.depth + 1} />
)}
```

### 5. Sibling Rendering (`Container.tsx`)

**Location**: `src/Container.tsx` lines 17, 55-64

Siblings are rendered independently:

```typescript
const nodes = treeContext.tree.filter((l) => l.parent === props.parentId);

// Each sibling is rendered as a separate Node component
{view.map((node, index) => (
  <React.Fragment key={node.id}>
    <Placeholder ... />
    <Node id={node.id} depth={props.depth} />
  </React.Fragment>
))}
```

Each `<Node>` component independently checks `openIds.includes(node.id)` to determine its open state.

## Example Scenarios

### Scenario 1: Initial Open with Siblings

**Tree Structure:**
```
Root (0)
├── Folder A (1) [has children]
│   ├── File A1 (2)
│   └── File A2 (3)
└── Folder B (4) [has children]
    ├── File B1 (5)
    └── File B2 (6)
```

**With `initialOpen: true`:**
- Both Folder A (1) and Folder B (4) open initially
- `openIds = [1, 4]`
- Both siblings are open independently

**With `initialOpen: [1]`:**
- Only Folder A (1) opens initially
- `openIds = [1]`
- Folder B (4) remains closed
- Opening Folder B later: `openIds = [1, 4]` (both remain open)

### Scenario 2: Toggling Siblings

**Starting state**: `openIds = [1]` (only Folder A open)

**Action**: User clicks to open Folder B
- `handleToggle(4)` is called
- `openIds` becomes `[1, 4]`
- **Folder A remains open** - no change to its state
- Both siblings are now open

**Action**: User clicks to close Folder A
- `handleToggle(1)` is called
- `openIds` becomes `[4]` (removes 1)
- **Folder B remains open** - unaffected by Folder A closing

### Scenario 3: Multiple Siblings at Same Level

**Tree Structure:**
```
Root (0)
├── Folder 1 (1) [has children]
├── Folder 2 (2) [has children]
├── Folder 3 (3) [has children]
└── Folder 4 (4) [has children]
```

**Initial state**: `initialOpen: [1, 3]`
- `openIds = [1, 3]`
- Folders 1 and 3 are open
- Folders 2 and 4 are closed

**User opens Folder 2**:
- `openIds = [1, 2, 3]`
- Folders 1, 2, and 3 are open
- Folder 4 remains closed

**User closes Folder 1**:
- `openIds = [2, 3]`
- Folders 2 and 3 remain open
- Folder 1 closes, but others unaffected

## Key Behaviors Summary

1. **Sibling Independence**: Each sibling node's open/closed state is completely independent
2. **No Cascading**: Opening one sibling does NOT close others
3. **No Grouping**: Siblings are not grouped - they're just different IDs in the same array
4. **State Persistence**: Once a node is opened, it stays open until explicitly closed (unless `initialOpen` prop changes)
5. **Initial State Reset**: When `initialOpen` prop changes, all open states reset to the new initial state

## Code Flow

```
User clicks node toggle
    ↓
Node.tsx: handleToggle() calls treeContext.onToggle(item.id)
    ↓
TreeProvider.tsx: onToggle(id) calls handleToggle(id, props.onChangeOpen)
    ↓
useOpenIdsHelper.ts: handleToggle() updates openIds array
    ↓
React re-renders all Node components
    ↓
Each Node checks: openIds.includes(props.id)
    ↓
Node renders children if open === true
```

## Related Files

- `src/hooks/useOpenIdsHelper.ts` - Core expand/collapse logic
- `src/providers/TreeProvider.tsx` - Tree context provider
- `src/Node.tsx` - Individual node component
- `src/Container.tsx` - Container that renders siblings
- `src/types.ts` - Type definitions for `InitialOpen`, `ToggleHandler`, etc.

## Conclusion

The tree component implements a **simple, independent state model** where:
- Each node's open state is tracked by its ID in the `openIds` array
- Sibling nodes have no knowledge of or influence on each other
- Opening/closing one sibling has zero effect on other siblings
- Initial state can be controlled via the `initialOpen` prop
- State changes are isolated to the specific node being toggled

This design provides maximum flexibility and predictable behavior - each node operates independently without side effects on its siblings.
