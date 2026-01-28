# OpenCode Storage Structure

This directory structure appears to be the internal storage for **OpenCode** (or a similar AI coding assistant), used to persist your chat sessions, message history, and project contexts.

Here is the breakdown of the data hierarchy:

*   **`project/`**: Contains metadata about the projects you've worked on.
    *   `global.json`: likely defines global settings or the default context (rooted at `/`).
*   **`session/`** & **`session_diff/`**: specific records for your interaction sessions.
    *   `session_diff` files likely track state changes or diffs within a specific conversation `ses_...`.
*   **`message/`**: Stores metadata for individual messages, organized by session ID.
    *   For example, `message/ses_.../msg_...` contains details like the role (user/assistant), timestamp, and the AI model used.
*   **`part/`**: Contains the actual content of the messages, organized by message ID.
    *   `part/msg_.../prt_...` holds the text or data pieces (like your prompts or the AI's code blocks).

**In summary:**
`Project` → `Session` → `Message` (Metadata) → `Part` (Content)

This structure allows the application to reconstruct your past conversations and context.