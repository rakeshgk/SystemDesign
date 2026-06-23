# LLM Inference

LLM Inference knowledge is increasingly becoming the AI equivalent of knowing how REST APIs, databases, and caching work.

## Inference Stack aka System Architecture

When a user sends a prompt (Example - `Explain Kubernetes`) the inference stack roughly looks like: 

```
User
  |
API Gateway
  |
Authentication
  |
Rate Limiter
  |
Inference Service
  |
Tokenizer
  |
GPU Model Server
  |
LLM
  |
Detokenizer
  |
Response  
``` 

### Prompt

Input text sent to the model. 

```
{
  "model": "gpt-5",
  "messages": [
    {
      "role": "user",
      "content": "Explain Kubernetes"
    }
  ]
}
```

### Tokens

LLMs don't see words, they see tokens. For example - `Kubernetes is awesome` might become `[38192, 552, 8847]`.

#### Interview Questions

1. What is a token?
    * A token is the basic unit of text that an LLM processes. A token is not necessarily a word. 
    * The exact tokenization of a prompt depends on the model's tokenizer.
2. Why do tokens matter?
    * Tokens matter because LLMs operate entirely on tokens, not raw text.
    * Every model has a maximum number of tokens it can handle. For example - If a model supports 128K tokens then Input Tokens + Output Tokens <= 128K.
    * Tokens determine latency - More tokens == More computation.
    * Tokens determine throughput - Inference servers are typically measured in tokens/second or generated tokens/second. 
    * Interviewers often ask - `How do you measure LLM serving performance?`. The typical answer for this question should be 
        - Latency
        - Throughput
        - Tokens/second
        - Time To First Token (TTFT)
3. Why does cost depend on tokens? 
    * More tokens require more GPU computation.
    * Cloud providers charge based on how much compute you consume.
    * Generating output (like text) is also expensive. The model generates one output token at a time.
    * Many providers charge more for output tokens because generation is sequential. All input tokens are processed together in a parallel fashion. Output token generation is sequential because each token depends on the previous token. (Also called autoregressive generation)

### Context Window

Context Window is the maximum number of tokens the model can attend to during inference. Larger context windows require more GPU memory because the model must maintain larger KV caches and perform attention over longer sequences. Context Window is a model capability, while GPU memory is a hardware resource used to support that capability. 

Sample model context windows:

1. 8K
2. 32K
3. 128K
4. 1M

#### Interview Questions

1. What happens if prompt exceeds context?
    * Option 1: Request Rejected - Many inference APIs behave this way.
    * Option 2: Truncation - System drops part of the prompt. Either it removes the oldest content or the newest content to fit within the limit.
    * Option 3: Sliding Window - Keep only the most recent tokens and conversations. This is common in chat applications. 
    * Option 4: Summarize Earlier Conversation - Use summary for older messages and send the new message as is to accommodate the prompt within the context.
    * Option 5: Retrieve relevant history - Similar to RAG, instead of sending the entire conversation, the system retrieves only relevant prior turns.
    * If the prompt exceeds the context window, the model cannot attend to all tokens simultaneously. Depending on the implementation, the request may fail, be truncated, or use a sliding window strategy. Any information outside the context window becomes inaccessible to the model.
2. How would you handle large documents?
    * Option 1: Chunking - Split the document into smaller pieces. Each chunk is small enough to fit into the context window.
    * Option 2: Retrieval Augmented Generation (RAG) - This is the most common production solution. Split the document into chunks, create vector embeddings for each chunk, store the embeddings in a vector database. During Query time, the system will only retrieve the relevant chunks and send them along with the user's question to the LLM. 
    * Option 3: Hierarchical Summarization - Useful when you need understanding of the entire document. Summarize every chapter -> Summarize chapter summaries -> Generate final summary. This is also called Map/Reduce summarization
    * Option 4: Sliding Window Processing - Window 1: Tokens 1-10K, Window 2: Tokens 8K-18K, Window 3: Tokens 16K-26K. Overlap between windows preserves context. Used for Long Transcripts, Log analysis and streaming data.
    * If a document exceeds the model's context window, I would avoid sending the entire document to the model. In production, the most common approach is Retrieval-Augmented Generation (RAG), where the document is chunked, embedded, and stored in a vector database. At query time, only the most relevant chunks are retrieved and sent to the model. For tasks requiring understanding of the entire document, I would use hierarchical summarization or map-reduce style processing. This keeps token usage, latency, and inference cost manageable while preserving answer quality.
3. Why not just increase the context window?
    Larger contexts increase latency, KV-cache memory, GPU usage, and cost. Retrieval is usually more efficient than sending everything.

