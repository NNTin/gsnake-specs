# Epic Brief: Rust Core & CLI Integration

## Summary

This Epic focuses on transitioning the core gSnake GameEngine from TypeScript to Rust to establish a robust, cross-platform foundation. By compiling to WebAssembly, the Rust core will serve as a single source of truth for game logic in the web frontend while simultaneously enabling a native terminal-based CLI. The architecture will support a push-based state communication model, where the Rust core emits frames and state updates to the host environment. This transition prioritizes high coding standards and extensibility, ensuring that improvements to the engine automatically benefit both the web and CLI interfaces.

## Context & Problem

Currently, the gSnake engine is implemented in TypeScript, coupling the game logic to the web environment. This prevents the game from being played natively in the terminal and requires browser-based agents for any automated interaction (e.g., LLMs). This coupling also makes it difficult to maintain a consistent game state across different platforms. By migrating the core logic to Rust, we decouple the "brain" of the game from its visual representation, allowing the web frontend to focus solely on rendering while providing a first-class CLI experience.

## Success Criteria

- **Logic Parity:** 100% consistent game behavior between Web and CLI targets.
- **Portability:** The Rust core is cleanly decoupled from host-specific I/O (Web vs. Terminal).
- **Scalability:** The architecture provides a clear path for future command-driven interfaces (e.g., for LLMs), although the MVP focuses on real-time interactive gameplay starting at Level 1.
- **Resilience:** Clear error reporting with tracebacks for environment failures (e.g., missing level data).
