# Design Journal — issue-27-agent-control-plane

## 2026-08-06 — Design + initial implementation

### Key design decisions

**Raggable MCP pattern:** 6 category tools with model-as-discovery. The model
tree provides action discovery on demand instead of encoding all capabilities
in tool definitions. Keeps context cost constant as Trellis grows. The IntelliJ
MCP comparison was the key insight — IntelliJ needs 40+ tools because it has no
model; Trellis has the model.

**No separate model definition:** Java records and TS interfaces ARE the schema.
`ModelProvider` SPI assembles the tree at query time from service beans. No drift,
no code generation.

**Session logs on disk:** Append-only files, teed from FifoRelay via sink
composition. RandomAccessFile tail-read is O(bytes returned). Input logging uses
bracketed markers for MCP-initiated input.

**Frontend state push:** Sidecar never queries the frontend. Frontend pushes
opaque JSON via `POST /api/model/ui-state`. The sidecar stores and serves as-is.

### Design review outcomes

Standard review: 62 issues across coherence (19), structure (13), robustness (17),
cross-cutting (13). 46 verified and applied. Key improvements: ModelProvider SPI
(was single tool bean), generation counter for freshness, action source field
(backend vs frontend), navigation correlation protocol, SessionLogger as separate
service bean (was WebSocket concern).

### Implementation progress

Tasks 1-3 of 12 complete:
- quarkus-mcp-server-http dependency + bare TrellisTools
- SessionLogger with tail-read and FifoRelay tee
- ModelProvider SPI, TerminalModelProvider, GenerationCounter
