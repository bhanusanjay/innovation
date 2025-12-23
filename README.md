# Optimal Context Management Strategies for GenAI Assistants

## Current Approach: Full Summarization
**Your Current Method:** Summarizing all previous queries and responses

**Problems with this approach:**
- Loss of specific details and nuance
- Compression artifacts (important info might be lost)
- High token cost for summarization on every turn
- Temporal information becomes unclear
- Can't reference specific past exchanges

---

## Recommended Approaches (Ranked)

### 🥇 **Approach 1: Hybrid Sliding Window + Semantic Memory**
**Best for: Most production applications**

**How it works:**
1. Keep last N messages in full (sliding window)
2. Summarize older messages into semantic memory chunks
3. Use vector similarity to retrieve relevant past context
4. Combine recent + relevant context for each query

**Implementation Strategy:**

```
Recent Context (Full Detail):
- Last 6-10 messages (3-5 turns)
- No summarization, full fidelity

Semantic Memory (Summarized):
- Older conversations grouped by topic
- Each chunk: 1-2 paragraphs
- Stored with embeddings in vector DB

Active Context Assembly:
1. Always include: Recent sliding window
2. Conditionally add: Retrieved relevant memories
3. Add metadata: timestamps, topic tags
```

**Advantages:**
- Balances detail with efficiency
- Captures long-term patterns
- Retrieves only relevant history
- Scales to long conversations

**Token Budget Example:**
- Recent context: 2,000-3,000 tokens
- Retrieved memories: 1,000-1,500 tokens
- System prompt: 500-1,000 tokens
- User query: 200-500 tokens
- **Total: ~4,000-6,000 tokens** (well within limits)

---

### 🥈 **Approach 2: Hierarchical Summarization**
**Best for: Long conversations with clear phases**

**How it works:**
1. Keep detailed last N turns
2. Mid-term: Progressive summarization (every 5 turns → summary)
3. Long-term: Meta-summaries (summaries of summaries)
4. Maintain key facts separately

**Structure:**
```
Level 0 (Current): Full last 3-5 turns
Level 1 (Recent): Summaries of turns 6-20
Level 2 (Archive): Meta-summary of turns 21+
Key Facts Store: Extracted entities, preferences, decisions
```

**Advantages:**
- Preserves detail at multiple timescales
- Efficient compression
- Easy to implement progressive summarization

---

### 🥉 **Approach 3: Smart Sliding Window with Key Extraction**
**Best for: Simpler implementations, moderate conversation length**

**How it works:**
1. Maintain sliding window of last N full messages
2. Extract and persist key information separately
3. Inject key facts as structured context

**Implementation:**

```
Sliding Window: Last 8 messages (4 turns)

Key Facts Store (Always Included):
- User preferences: "Prefers Python over JavaScript"
- Current context: "Working on LangGraph project"
- Important decisions: "Chose approach #2 for API design"
- Entities: "User: Alex, Company: TechCorp"

Summary Header (Optional):
- One paragraph overview if conversation > 10 turns
```

**Advantages:**
- Simple to implement
- Predictable token usage
- Preserves critical information
- Fast retrieval

---

## Implementation Code Examples

### Option 1: Hybrid Implementation (LangGraph)

```python
from langgraph.graph import StateGraph, MessagesState
from langchain_core.messages import HumanMessage, AIMessage
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

class ConversationState(MessagesState):
    key_facts: dict
    semantic_memory_ids: list
    
class ContextManager:
    def __init__(self):
        self.vector_store = Chroma(
            embedding_function=OpenAIEmbeddings()
        )
        self.sliding_window_size = 10  # last 10 messages
        
    def get_optimal_context(self, state: ConversationState):
        messages = state["messages"]
        
        # 1. Recent context (full detail)
        recent_messages = messages[-self.sliding_window_size:]
        
        # 2. Retrieve relevant semantic memories
        if len(messages) > self.sliding_window_size:
            query = messages[-1].content
            relevant_memories = self.vector_store.similarity_search(
                query, k=3
            )
            memory_context = "\n".join([doc.page_content for doc in relevant_memories])
        else:
            memory_context = ""
        
        # 3. Key facts (always included)
        key_facts_str = self._format_key_facts(state["key_facts"])
        
        # 4. Assemble context
        context_parts = []
        
        if key_facts_str:
            context_parts.append(f"Key Information:\n{key_facts_str}")
            
        if memory_context:
            context_parts.append(f"Relevant Past Context:\n{memory_context}")
            
        context_header = "\n\n".join(context_parts)
        
        return context_header, recent_messages
    
    def should_create_memory(self, messages):
        """Create memory every 10 messages"""
        return len(messages) % 10 == 0 and len(messages) > self.sliding_window_size
    
    async def create_semantic_memory(self, messages_chunk, llm):
        """Summarize a chunk and store in vector DB"""
        conversation_text = self._format_messages(messages_chunk)
        
        summary_prompt = f"""Summarize this conversation segment, preserving:
- Key decisions and conclusions
- Important facts and entities
- User preferences or requirements
- Context for future reference

Conversation:
{conversation_text}

Summary:"""
        
        summary = await llm.ainvoke(summary_prompt)
        
        # Store with metadata
        self.vector_store.add_texts(
            texts=[summary.content],
            metadatas=[{
                "type": "conversation_memory",
                "message_count": len(messages_chunk),
                "timestamp": messages_chunk[-1].timestamp
            }]
        )
```

### Option 2: Smart Sliding Window Implementation

```python
class SimpleContextManager:
    def __init__(self, window_size=8):
        self.window_size = window_size
        self.key_facts = {}
        
    def get_context(self, messages):
        # Recent messages (full)
        recent = messages[-self.window_size:]
        
        # Add summary header if conversation is long
        context_header = ""
        if len(messages) > self.window_size:
            context_header = self._create_summary_header(messages[:-self.window_size])
        
        # Format key facts
        if self.key_facts:
            facts_str = "Key Information:\n" + "\n".join(
                f"- {k}: {v}" for k, v in self.key_facts.items()
            )
            context_header = f"{facts_str}\n\n{context_header}" if context_header else facts_str
        
        return context_header, recent
    
    def extract_key_facts(self, message, llm):
        """Extract important facts from messages"""
        extraction_prompt = f"""Extract any important facts, preferences, or context from this message that should be remembered:

Message: {message.content}

Return as JSON with keys like: user_preference, entity, decision, requirement, etc.
If nothing important, return empty JSON."""
        
        facts = llm.invoke(extraction_prompt)
        # Update key_facts dictionary
        self.key_facts.update(facts)
    
    def _create_summary_header(self, old_messages):
        """Create brief summary of conversation before window"""
        # Simple: count turns and note it
        turns = len(old_messages) // 2
        return f"[Previous context: {turns} earlier conversation turns]"
```

---

## Decision Matrix

| Approach | Complexity | Token Efficiency | Detail Preservation | Scalability | Best For |
|----------|------------|------------------|---------------------|-------------|----------|
| Hybrid (Sliding + Semantic) | High | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Production apps, long conversations |
| Hierarchical Summary | Medium | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Multi-session conversations |
| Smart Sliding Window | Low | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Simple apps, shorter conversations |
| Full Summarization (Current) | Low | ⭐⭐ | ⭐⭐ | ⭐⭐ | Quick prototypes only |

---

## Key Recommendations

### 1. **Start with Smart Sliding Window**
If you're currently doing full summarization, migrate to smart sliding window first. It's simpler and already better than your current approach.

### 2. **Add Semantic Retrieval When Scale Demands**
Once conversations regularly exceed 20+ turns, add vector-based semantic memory retrieval.

### 3. **Always Maintain Key Facts**
Regardless of approach, extract and persist:
- User preferences and requirements
- Important decisions made
- Entity information (names, projects, etc.)
- Current task/goal

### 4. **Monitor Token Usage**
Track your context token consumption and adjust window size based on:
- Model's context limit (Llama 3.3: 128k tokens)
- Cost considerations
- Response quality

### 5. **Use Structured Context Injection**

```
System Prompt
---
Key Facts: [Always included]
- User: Building LangGraph assistant
- Preference: Structured responses
---
Relevant Memory: [If retrieved]
Earlier in conversation, user discussed...
---
Recent Conversation: [Sliding window]
User: ...
Assistant: ...
---
Current Query: [New message]
```

---

## Migration Path from Your Current Approach

**Phase 1:** Implement smart sliding window (1-2 days)
- Keep last 8-10 messages full
- Add key facts extraction

**Phase 2:** Add progressive summarization (3-5 days)
- Summarize every 10 messages beyond window
- Store summaries with metadata

**Phase 3:** Add semantic retrieval (1 week)
- Set up vector database
- Implement similarity search
- Retrieve relevant context dynamically

**Phase 4:** Optimize (ongoing)
- Monitor performance metrics
- Adjust window size based on usage
- Refine summarization prompts
