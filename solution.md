# Solution to IT Helpdesk Triage Agent Assessment

## Q1: Design Choices — Architecture
**Evaluation:** The decision to share `conversation_history` across tickets in a batch run is **flawed and inappropriate** for this use case.

**Reasoning:**
1.  **Context Bleeding:** Support tickets are typically independent events. Sharing history allows context from Ticket A (e.g., a network issue) to potentially bias or confuse the model when processing Ticket B (e.g., a software issue).
2.  **Context Window Exhaustion:** Accumulating history across all tickets in a batch will rapidly consume the model's context window, leading to higher costs and eventual truncation of relevant information for later tickets.
3.  **Privacy/Security:** It risks leaking sensitive information from one user's ticket to the processing context of another user's ticket.

**Correct Approach:** The `conversation_history` should be re-initialized (cleared) for each ticket in the loop.

## Q2: Trace the State
**Scenario:** The LLM emits a response containing **two** `tool_use` blocks (e.g., `ToolA` and `ToolB`).

**Trace:**
1.  The code `tool_use = next(b for b in response.content if b.type == "tool_use")` finds only the **first** tool use (`ToolA`).
2.  It executes `search_runbooks` for `ToolA` only.
3.  The following messages are appended to `conversation_history`:
    *   **Assistant Message:** Contains the *full* response with **both** `ToolA` and `ToolB` requests.
    *   **User Message:** Contains a `tool_result` block for **only `ToolA`**.

**Information Lost:**
*   The execution and result of **`ToolB`** are completely lost.
*   The model receives a history where it requested two tools but only received a result for one. This puts the conversation in an invalid or incomplete state, often causing API errors in subsequent turns (e.g., "missing tool result") or causing the model to get stuck/confused.

## Q3: Predict the Failure Mode
**Failure Prediction:**
The `_prune_history` function naively slices the list (`del history_list[:-10]`). This will eventually cause a **hard API rejection**.

**Explanation:**
LLM APIs (like Anthropic's) strictly enforce that every `tool_result` block in a User message must be preceded by a corresponding `tool_use` block in the previous Assistant message.
*   **The Bug:** Naive slicing based on `len()` can cut the history **between** an Assistant message (containing `tool_use`) and the subsequent User message (containing `tool_result`).
*   **The Result:** The pruned history will start with (or contain) a User message holding a "orphaned" `tool_result` whose parent `tool_use` has been deleted. The API will reject this malformed conversation structure.

## Q4: What happens with THIS input?
**Input:** "My laptop screen keeps flickering when I connect it to the docking station. Can someone replace the dock?"

**Result:**
*   **Runbook Chunks Included:** **None** (Empty list).

**Explanation:**
1.  **Embedding Failure:** The `embed_query` function relies on strict keyword matching. The input string does not contain any of the category keywords (`"network"`, `"software"`, `"hardware"`, `"access"`) nor does it match any `QUERY_EMBEDDINGS` patterns. Even though "laptop" and "dock" imply hardware, the specific string `"hardware"` is missing.
2.  **Fallback:** The function returns a **uniform fallback embedding** (`[0.25, 0.25, ...]`).
3.  **Similarity Threshold:** This uniform embedding has a low cosine similarity with the specialized/peaked embeddings of the runbook chunks. The calculated scores fall well below the `SIMILARITY_THRESHOLD` of `0.85`, so `retrieve_chunks` returns an empty list.

## Q5: Fix evaluation
**Evaluation:** The proposed fix (increasing `top_k` from 3 to 10) is **ineffective**.

**Actual Root Cause:**
The issue is not that the correct result was "hidden" deeper in the list (positions 4-10). The issue is that **no results were returned at all**.
*   Ticket TKT-001 ("error code 619") does not contain the word "network" or match the exact phrase "vpn connection error".
*   Consequently, `embed_query` generated a uniform garbage embedding that failed to meet the `0.85` similarity threshold for *any* chunk.

**Better Approach:**
*   **Short-term:** Update `embed_query` to map relevant keywords like "vpn", "wifi", "connect", "error" to the `"network"` category (or add "vpn" to the `categories` list).
*   **Long-term:** Replace the manual keyword-weighted embedding logic with a **real semantic embedding model** (e.g., `text-embedding-3-small`) to capture the semantic meaning of queries like "error 619" without relying on brittle keyword matching.
