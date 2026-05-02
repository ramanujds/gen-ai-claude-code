# How Claude Works — A Complete Explanation

## The Example Prompt: *"What is the capital of France?"*

---

## Overview

When you type a message to Claude, it doesn't simply "look up" an answer. Instead, your text travels through a sophisticated pipeline involving tokenization, hundreds of neural network layers, probabilistic decoding, and safety filtering — all to produce one token at a time. Here is every step, in full detail.

---

## Step 1 — Input Assembly (Building the Context Window)

Claude never receives your message in isolation. Before any processing begins, several components are assembled into a single **context window** — a long sequence of text that becomes the full input to the model:

- **Your prompt:** `"What is the capital of France?"`
- **System prompt:** A set of instructions written by Anthropic (or an operator) that defines Claude's persona, rules, and constraints — e.g., *"Be helpful, harmless, and honest."*
- **Conversation history:** Every prior message in the current session, both from you and from Claude, in order.
- **Tool definitions:** Descriptions of any tools available (web search, code execution, etc.), written in a structured format so Claude knows what it can call.

All of this is concatenated into one long string and fed into the model together. Claude has no persistent memory between sessions — everything it "knows" about your conversation lives inside this window.

> **Key insight:** Claude doesn't distinguish between "your message" and "the system prompt" at a fundamental level. They are all just tokens in one sequence. The model has learned from training to respect instruction-style text, but mechanically it's one flat stream.

---

## Step 2 — Tokenization

Raw text cannot be fed directly into a neural network. It must first be converted into numbers. This is done by a **tokenizer**.

Claude uses a variant of **Byte-Pair Encoding (BPE)** with a vocabulary of roughly 100,000 tokens. A token is not always a word — it can be a subword, a punctuation mark, a number, or even a single character.

Your prompt breaks down like this:

```
"What is the capital of France?"
→ ["What", " is", " the", " capital", " of", " France", "?"]
```

Note the leading spaces — they are part of the token and carry meaning. Each token is then mapped to an integer ID (e.g., `"capital"` → token ID `18766`), and that integer is used to look up a **dense vector** in an embedding table.

### Embeddings & Positional Encoding

Each token ID maps to an **embedding vector** — a list of thousands of floating-point numbers that encodes the token's meaning in a high-dimensional space. Words with similar meanings have vectors that point in similar directions.

Because the transformer architecture itself has no inherent sense of order, **positional encodings** are added on top of each embedding — mathematical signals that tell the model *where* in the sequence each token sits. Without this, "France capital the of is What?" would look identical to "What is the capital of France?"

---

## Step 3 — Self-Attention (Tokens Talk to Each Other)

The embedded, position-encoded vectors now enter the first transformer layer. The core mechanism is **multi-head self-attention**, and it is where the model builds contextual understanding.

### How attention works

For every token, the model asks: *"Which other tokens in this sequence are relevant to understanding me?"*

It does this by computing three vectors per token — a **Query (Q)**, a **Key (K)**, and a **Value (V)** — and then calculating attention scores between every pair of tokens using the dot product of Q and K. High scores mean "these two tokens are strongly related." The scores are normalized with softmax and used to create a weighted blend of all Value vectors.

### What this looks like for our example

- `"capital"` attends strongly to `"France"` → the model recognizes this is a geography question about France specifically.
- `"What"` + `"?"` together signal that a direct factual answer is expected.
- `"the"` and `"of"` get lower attention scores — they are syntactic glue, not content.

This happens in **multiple heads** simultaneously — each head learns to attend to different kinds of relationships (syntactic, semantic, coreference, etc.). The outputs of all heads are concatenated and projected back into the model's main representation space.

---

## Step 4 — Feed-Forward Network & Layer Stacking

After attention, each token's vector passes through a **position-wise feed-forward network (FFN)** — two linear transformations with a non-linear activation in between. This is where much of the model's factual knowledge is believed to be stored, in the billions of weight parameters.

Each attention + FFN block is one **transformer layer**. Claude is stacked with many such layers (the exact number is not publicly disclosed, but large models typically have 80–120+ layers). Between layers, **residual connections** add the input back to the output, and **layer normalization** keeps the values numerically stable.

### What happens layer by layer (conceptually)

| Layer range | What the model learns |
|---|---|
| Early layers (1–10) | Syntax, grammar, token boundaries |
| Mid layers (10–40) | Semantics, word sense, named entities |
| Deep layers (40–80) | World knowledge, facts, reasoning patterns |
| Final layers | Task-specific output preparation |

By the time our prompt has passed through all layers, the representation of the final token (or a special summary position) encodes the distilled meaning: *"this is a factual question asking for the capital city of France, and the answer is Paris."*

---

## Step 5 — Output Logits & Probability Distribution

The final transformer layer produces a vector for each position in the sequence. The vector at the **last position** is passed through a linear projection (the "language model head") that maps it to a score — called a **logit** — for every token in the ~100,000-token vocabulary.

These logits are converted to probabilities using **softmax**. For our example, the distribution looks roughly like:

| Token | Probability |
|---|---|
| `"Paris"` | ~94.7% |
| `"Lyon"` | ~1.2% |
| `"Marseille"` | ~0.9% |
| `"London"` | ~0.3% |
| `"Berlin"` | ~0.2% |
| *...96,000+ other tokens* | ~2.7% total |

### Temperature & Sampling

Before sampling, two parameters shape the distribution:

- **Temperature:** Values below 1.0 make the distribution sharper (more deterministic). Values above 1.0 flatten it (more creative/random). For factual questions, Claude uses lower effective temperature.
- **Top-p (nucleus sampling):** Only the smallest set of tokens whose cumulative probability exceeds a threshold *p* are considered. This prevents low-probability nonsense tokens from ever being chosen.

The next token is then **sampled** from this distribution. For factual questions like ours, `"The"` is almost certainly chosen first.

---

## Step 6 — The Autoregressive Loop

This is the most important architectural truth about Claude: **it generates one token at a time, feeding each new token back into the input for the next pass.**

This is called **autoregressive decoding**, and it proceeds like this:

**Pass 1:**
- Input: `[...full context... + "What is the capital of France?"]`
- Output token: `"The"`

**Pass 2:**
- Input: `[...full context... + "What is the capital of France?" + "The"]`
- Output token: `" capital"`

**Pass 3:**
- Input: `[...full context... + "... The capital"]`
- Output token: `" of"`

**Pass 4:**
- Input: `[...full context... + "... The capital of"]`
- Output token: `" France"`

**Pass 5:**
- Input: `[...full context... + "... The capital of France"]`
- Output token: `" is"`

**Pass 6:**
- Input: `[...full context... + "... The capital of France is"]`
- Output token: `" Paris"`

**Pass 7:**
- Input: `[...full context... + "... is Paris"]`
- Output token: `"."` then `[EOS]` — End of Sequence

Each pass runs the **entire neural network** from scratch. A long response means hundreds or thousands of full forward passes. This is why inference on large language models is computationally expensive.

---

## Step 7 — Safety & Value Alignment (Baked into the Weights)

Claude's safety properties are not a post-processing filter bolted on at the end. They are **trained into the model's weights** through two key techniques:

### Reinforcement Learning from Human Feedback (RLHF)
Human raters evaluated thousands of responses and ranked them. A separate **reward model** was trained on these rankings. Claude was then fine-tuned using reinforcement learning to maximize this reward — learning to produce responses that humans rate as helpful, accurate, and safe.

### Constitutional AI (CAI)
Claude was trained with a written **constitution** — a set of principles about harmlessness, honesty, and helpfulness. The model was trained to critique and revise its own outputs against these principles, internalizing them as values rather than just rules to follow.

For our question `"What is the capital of France?"`, this step is trivial — the response is factual and benign. For a harmful prompt, the model's internalized values would cause it to generate a refusal instead, because that's what the training made the most probable output.

---

## Step 8 — Detokenization

Once all tokens have been generated (terminated by `[EOS]`), they are joined back into a human-readable string by the **detokenizer**:

```
["The", " capital", " of", " France", " is", " Paris", "."]
→ "The capital of France is Paris."
```

This is the inverse of tokenization — spaces and subword pieces are reassembled into natural text.

---

## Step 9 — Streaming to Your Screen

Claude doesn't wait until the full response is generated before sending it to you. Using **server-sent events**, each token is transmitted to your browser *the moment it is generated*. This is why you see Claude's text appear word by word — each word appearing on screen represents **one completed forward pass** through the entire neural network, typically taking a few milliseconds.

This streaming behavior has a meaningful implication: Claude is genuinely "thinking as it writes." The token it generates at position *n* influences what it generates at position *n+1*. There is no pre-planned response — each token is the statistically most appropriate continuation given everything that came before.

---

## Summary Table

| Step | What happens | Technical name |
|---|---|---|
| 1 | Prompt + system + history assembled | Context window construction |
| 2 | Text split into numbers + meaning vectors | Tokenization & embedding |
| 3 | Tokens relate to each other | Multi-head self-attention |
| 4 | Meaning refined through many layers | Transformer forward pass |
| 5 | Scores assigned to all possible next words | Logit projection + softmax |
| 6 | One token chosen, fed back in, repeated | Autoregressive decoding |
| 7 | Safety & values shape output | RLHF + Constitutional AI |
| 8 | Token IDs rejoined into text | Detokenization |
| 9 | Words appear on screen as generated | Token streaming |

---

## The Big Picture

Claude is, at its core, a very large function: **given all the text so far, predict the most appropriate next token.** Repeated thousands of times, this simple operation produces coherent reasoning, factual recall, creative writing, and nuanced conversation — because the billions of parameters encoding that function were trained on an enormous slice of human knowledge and refined to reflect human values.

The apparent intelligence emerges not from any single mechanism, but from the depth of the network, the scale of training, and the careful alignment work that shapes *which* continuations the model finds most probable.
