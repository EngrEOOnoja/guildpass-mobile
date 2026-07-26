# Offline Mutation Queue

This document outlines the architecture, usage, and conflict-resolution strategy for the Offline Mutation Queue in GuildPass.

## Overview
While read-caching (via TanStack Query) allows users to view stale data while offline, certain mutations (e.g., updating preferences or profiles) can be safely queued and deferred until the device regains connectivity. 

The `MutationQueue` abstraction provides a persistent, encrypted queue for these actions. It ensures:
- Queueable actions are persisted locally when offline.
- Actions are replayed in order (FIFO) upon reconnect.
- Synchronous-only actions (like access checks) are strictly blocked from being queued.

## Architecture

- **`mutationQueue.ts`**: The core abstraction. It stores an array of `QueueItem` objects `{ id, type, payload, createdAt, status, retryCount, lastError }`. Like the read cache, this queue is persisted via `AsyncStorage` and encrypted at rest using `encryptionService` and the device-bound key from `keyManager`.
- **`mutationTypes.ts`**: Explicitly classifies mutations into two categories:
  1. **Queueable**: Safe to defer (e.g., `UPDATE_PREFERENCES`, `UPDATE_PROFILE`, `SYNC_DRAFT`).
  2. **Synchronous-only**: Must execute in real-time (e.g., `ACCESS_CHECK`, `VERIFY_WALLET`, `VALIDATE_TOKEN`). Any attempt to queue these offline will be rejected.
- **`mutationReplayer.ts`**: Listens for network reconnection using `@react-native-community/netinfo` and processes pending mutations in FIFO order. Upon successful replay, it triggers `queryClient.invalidateQueries()` to ensure the read cache stays coherent with the newly pushed server state.

## Conflict-Resolution Strategy

When replaying queued mutations against the server, conflicts may arise if the server state has changed in a way that makes the queued action invalid.

### Policy
We use a **User-Assisted Conflict Resolution** strategy for the mutation queue. 

1. **Network Errors (5xx or Timeout)**:
   - Status updated to `FAILED`.
   - The replay loop halts to preserve FIFO order and waits for the next stable connection to retry.
2. **Client Errors / Conflicts (4xx)**:
   - Status updated to `CONFLICT`.
   - The item is left in the queue.
   - It requires manual user intervention via the "Pending Changes" UI (`app/pending-changes.tsx`), where the user can choose to either **Retry** or **Discard** the mutation.

### Tradeoffs
- **Pros**: It prevents silent data corruption (e.g., a "last-write-wins" policy might overwrite newer server state blindly with a stale offline change). It gives the user full control over resolving complex state conflicts.
- **Cons**: Requires explicit user action to clear stuck mutations. If the user never visits the "Pending Changes" screen, a conflicting mutation might remain stuck in the queue indefinitely.

## Integration with TanStack Query
To maintain a coherent offline story, the `MutationQueue` and `mutationReplayer` work alongside our existing TanStack Query read-cache setup:
- They share the exact same encryption semantics at rest (`encryptionService` + `keyManager`).
- When a queued mutation is successfully dispatched by `mutationReplayer`, we call `queryClient.invalidateQueries()`. This automatically triggers background refetches for relevant read queries once online, ensuring the UI reflects the fully reconciled server state.
