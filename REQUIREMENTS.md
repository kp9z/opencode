1. Persistent Agent Memory (AI behavior + productivity)

What: A memory layer that persists facts across sessions — project conventions, user preferences, past decisions. Agents write to it, read from it automatically at session start.

Why interesting: The codebase has full session history but no cross-session memory. Tree-sitter and SQLite are already there. Natural fit.

How: New memory tool + SQLite table. Embeddings (via a provider) for semantic recall, or simple keyword tagging for a simpler v1.

Files to touch: src/tool/, src/storage/, src/session/prompt.ts


Related to this: [text](https://github.com/anomalyco/opencode/issues/8043)