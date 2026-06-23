# Transformer

The Transformer architecture powers most modern LLMs (GPT, Claude, Llama, Gemini, etc.). It defines how tokens are processed and used to predict the next token.

## High-Level Transformer Architecture

1. You have a prompt.
2. The text within the prompt is tokenized. 
3. Each token is converted into a vector embedding. 
4. The transformer then processes these vectors through multiple layers. 

```
Input Tokens
      |
      v
Embeddings
      |
      v
+-------------------+
| Transformer Layer |
+-------------------+
      |
      v
+-------------------+
| Transformer Layer |
+-------------------+
      |
      v
+-------------------+
| Transformer Layer |
+-------------------+
      |
      v
Output Logits
      |
      v
Next Token Prediction
```

Modern LLMs have different numbers of layers:

1. 24 layers (small)
2. 80 layers (GPT-3 class)
3. 100+ layers (very large models)

## What is inside a Transformer Layer?

Each layer contains two major components. 

```
Input
  |
  +--> Self-Attention
  |
  +--> Feed Forward Network (MLP)
  |
Output
```

More precisly, each layer looks like this. 

```
Input
  |
LayerNorm
  |
Self-Attention
  |
Residual Add
  |
LayerNorm
  |
Feed Forward Network
  |
Residual Add
  |
Output
```

## Self-Attention: The Core Innovation

Transformers became dominant due to the self-attention mechanism. For example, in the prompt `"The animal didn't cross the road because it was tired."`, when processing the token `it`, the model needs to determine what `it` refers to.

Self-Attention allows `it` to consider

```
animal
cross
road
tired
```

and assign different importance scores. The model learns that `it` probably refers to `animal`.

Most people understand self-attention conceptually. The high level idea is that a token looks at other tokens and decides which ones are important. 

### Build Self-Attention Intuitively

Self-Attention works through 3 vectors

1. Query - What am I looking for? A better version once you get more familiar with the concepts is this definition - A Query defines which other tokens are useful for constructing a better representation of the current token. The query is not a human-readable question. It is simply a vector. 
2. Key - What do I contain?
3. Value - What information do I give if selected?

Imagine a conference where each attendee wears a Name Tag listing their specialization (Key) and has knowledge (Value). You walk around with a Query, like "Looking for experts in distributed systems." First, you read badges to find relevant people. Once identified, you ask your questions and gain insights. You don't interview everyone to determine who is relevant.

### Self-Attention Math

Imagine you have 3 words: `"The cat sat"`. Each word is represented as a small embedding vector (normally 512+ dimensions, but we'll use 4 for clarity in this example)

```
"The" → x1 = [1, 0, 1, 0]
"cat" → x2 = [0, 1, 0, 1]
"sat" → x3 = [1, 1, 0, 0]
```

Through training, we learn three weight matrices - `Wq, Wk, Wv`. These weight matrices start off random and get nudged towards useful values. In this example, we will use these values for the weight matrices. 

```
Wq = [[1, 0, 1, 0],
      [0, 1, 0, 1]]

Wk = [[0, 1, 0, 1],
      [1, 0, 1, 0]]

Wv = [[1, 1, 0, 0],
      [0, 0, 1, 1]]
```

For each token embedding `x` we do

```
Q = xWq
K = xWk
V = xWv

q1 = x1 . Wq = [2, 0]   k1 = x1 . Wk = [0, 2]   v1 = x1 . Wv = [1, 1]
q2 = x2 . Wq = [0, 2]   k2 = x2 . Wk = [2, 0]   v2 = x2 . Wv = [1, 1]
q3 = x3 . Wq = [1, 1]   k3 = x3 . Wk = [1, 1]   v3 = x3 . Wv = [2, 0]
```

For each token, we compute its attention score. For example - How much should `cat` attend to each word?

```
Score(cat, The) = q2 . k1 = 4
Score(cat, cat) = q2 . k2 = 0
Score(cat, sat) = q2 . k3 = 2
```

Once the attention scores are computed, we scale each score. Divide by $\sqrt{d_k}$ where $d_k$ = dimension of keys. We then perform softmax to convert the scaled scores to probabilities. 

```
scaled scores = [2.83, 0, 1.41]
softmax([2.83, 0, 1.41]) = [0.72, 0.04, 0.24]
```

The word `cat` pays 72% attention to the first word, 4% to itself and 24% to the third word. After this step, you compute the weighted sum of values. This value now becomes the new representation of the word `cat`. This value has been enriched with context from other words. 

```
outputs = 0.72 . v1 + 0.04 . v2 + 0.24 . v3
        = [1.24, 0.76]
```