## Overview

Add auto-focus behavior to the "Restart Level" button in file:components/GameOverModal.svelte when the modal mounts. This ensures the button is focused when the modal appears, though the focus state won't be visually indicated.

## Scope

**In Scope:**

- Add Svelte `onMount` lifecycle hook to GameOverModal
- Add `bind:this` reference to "Restart Level" button
- Call `.focus()` on button element when component mounts

**Out of Scope:**

- Visual focus indicators (explicitly not wanted per Core Flows)
- Keyboard event handling (handled by KeyboardHandler)
- Changes to button click handlers (existing handlers remain)

## Technical Approach

**Implementation Pattern:**

```typescript
// In GameOverModal.svelte <script>
import { onMount } from 'svelte';

let restartButton: HTMLButtonElement;

onMount(() => {
  restartButton?.focus();
});
```

**Button Reference:**

```html
<button bind:this={restartButton} ...>Restart Level</button>
```

## Acceptance Criteria

- [ ] "Restart Level" button is automatically focused when GameOverModal mounts
- [ ] Focus happens without visual indication (no outline or highlight)
- [ ] Existing button click handlers continue to work
- [ ] Mouse users can still click buttons normally
- [ ] No errors if button reference is null (defensive programming)

## References

- **Epic Brief**: spec:epics/navigation/epic-brief.md
- **Core Flows**: spec:epics/navigation/core-flows.md (Flow 2 → Initial State)
- **Tech Plan**: spec:epics/navigation/tech-plan.md (Component Architecture → KeyboardHandler section)

## Files to Modify

- file:components/GameOverModal.svelte - Add onMount and button reference

## Testing Notes

- Verify button is focused when modal appears
- Verify no visual focus indicator appears
- Test that existing click handlers still work
- Test with rapid game over → restart cycles

