# Transformers: Why “Attention Is All You Need” Changed Everything

Original Paper : https://proceedings.neurips.cc/paper_files/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf

Jay Alammar blog (to best visualize what transformers do) : https://jalammar.github.io/illustrated-transformer/


## 1. The Old Guard: RNNs and LSTMs Weren’t Enough

- Sequential bottleneck: Computation happens timestep by timestep — slow and unscalable.
- Context vanishing: LSTMs try to remember long-term dependencies, but fail when inputs are too long (e.g., 100K tokens).
- Limited parallelism: Since everything happens in a sequence, we can’t parallelize easily.

“We wanted to understand all tokens at once — not one at a time.”

## 2. Enter Self-Attention: The Matrix That Changed It All

- Idea: Can I compute relationships between all words in one go?
- Each word (token) is compared with every other word: this gives us an n × n self-attention matrix.
- This matrix tells us how much one word “attends” to another.

“It’s like every word voting on what other words it finds important.”

Benefits:
- Massive parallelism (matrix ops like multiplication are parallelizable).
- Captures global relationships across entire input.

## 3. The Parallelism Problem: But What About Word Order?

- Self-attention ignores sequence — meaning is lost.
  - “The cat sat on the mat” ≠ “The mat sat on the cat”
- RNNs inherently preserved order. Transformers need to reintroduce it.

## 4. Fixing the Order Problem: Positional Encoding

- Embeddings represent each word as a k-dimensional vector.
- To reintroduce order, we add position information to each embedding.
- But:
  - Discrete positions (0, 1, 2...) don’t work well with gradient-based learning.
  - Solution: Use continuous positional encodings (like sinusoids).
  - This way, the model can learn order-sensitive patterns using smooth functions.

## 5. Injecting Order: How Positional Encoding Brings Back Sequence Awareness

Attention models process all tokens in parallel — which is powerful, but they lose order in the process. The word “apple” could appear at position 2 or 200, and the model wouldn’t know unless we explicitly give it position information.

So, how can we encode position?
- We want each position (0, 1, 2, ..., 100, ...) to have a unique vector — one that the model can distinguish from others.
- Simply using integers (like 1 for first word, 2 for second) isn’t ideal. Neural nets learn better with continuous, smooth functions.
- The solution? Use sinusoidal positional encodings — a mix of sine and cosine functions at different frequencies.

This way:
- Each position gets a unique vector.
- Similar positions have similar encodings (which helps learning).
- These vectors are added to the original word embeddings.

## 6. Understanding Relationships: The Heart of Self-Attention

Let’s go deeper into how the model actually learns relationships between words.

Take this sentence:

“I love eating fresh apple in the morning.”

Humans intuitively know that “fresh” modifies “apple”. Similarly, in:

“She locked the door with a key.”

We know that “locked” goes with “door” and “key” explains how.

So, how can a model figure this out?

➤ The Trick: Representing Words with Query, Key, and Value Vectors

Each word in the sentence is projected into three vectors:

- Query (Q): How much impact does the other token have on me?
- Key (K): What is my impact on you?
- Value (V): What content each word carries.

This triple comes from a brilliant analogy:

- When you search for something on YouTube:
  - Your search phrase is the query.
  - The video title is the key.
  - The actual video is the value.

Just like that, in self-attention:
- We compute how well each query matches each key (using dot product).
- The stronger the match, the more attention is paid to the corresponding value.

➤ Self-Attention Mechanism, Step-by-Step

1. Compute similarity:
- Take the dot product of the query and key vectors for every pair of tokens.
- This gives us a score matrix showing how much each word attends to the others.

2. Scale it:
- The raw scores can get very large as the embedding dimension increases.
- So, we scale it by dividing with √(dimension of key vector), i.e., √dk.

3. Normalize with Softmax:
- Convert these scores into probabilities using Softmax.
- Words that are closely related will have high probability (~1), unrelated ones near 0.

4. Weighted sum with Value:
- Multiply these probabilities with the value vectors.
- This gives the final representation — a new vector that’s a blend of all words, weighted by relevance.

🔁 Recap: Self-Attention in Intuition

“Every word decides how much attention it should pay to every other word, and then gathers a summary of the sentence based on that.”

And the best part?
All of this can be done in parallel for every word — no recurrence, no sequential loops. That's the foundation of the transformer's speed and power.

## From One to Many: Multi-Head Self-Attention

So far, we’ve discussed how self-attention models relationships between every pair of tokens.

But real language is complex — consider this example:

“Naveen gave Satish a beautiful bike on his birthday.”

Here, multiple relationships must be learned:
 - What is being given? → bike
 - Who gave the bike? → Naveen
 - Who received it? → Satish
 - When was it given? → birthday

This means: one attention matrix isn’t enough.

➤ Multi-Head Attention to the Rescue

Instead of learning just one self-attention pattern, the transformer learns multiple self-attention matrices, each capturing a different kind of relationship.

- Think of it like CNNs: we don't hand-code filters to detect edges.
- We start with N random filters, and the model learns the patterns that matter.
- Similarly, in transformers, each "head" in multi-head attention starts with random weights and learns a unique relational focus.

Finally:
- Outputs from each attention head are concatenated and combined linearly to get the full picture.

“It’s like having multiple sets of eyes — each head looks at the sentence differently and notices a unique kind of relationship.”

## The Full Picture: Encoder-Decoder Architecture
The original transformer model in the paper is split into two main parts:

🧱 Encoder

- Converts input tokens into vector representations.
- Focuses on understanding the entire input sequence and learning its relationships.

🧱 Decoder

- Uses these vectors to generate output tokens, one at a time.
- Predicts the next word by attending to both:
  - Previously generated outputs.
  - Encoder outputs.

This is the classic encoder-decoder pattern also used in machine translation (e.g., English → French).

9. Putting It All Together: The Stack
In the paper, each encoder and decoder consists of N = 6 layers (referred to as Nx in the diagram).

Each encoder block contains:

- Multi-head self-attention
- Feed-forward network
- Skip connections (residuals) — to preserve original input signals.
- Layer normalization

“Skip connections ensure that even after multiple complex transformations, the model doesn’t forget the original input.”

And one key insight:
- GPT only uses the decoder stack (autoregressive).
- BERT only uses the encoder stack (bidirectional).

✅ Summary: Why This Changed Everything

- Parallelism: Attention replaces recurrence — faster and more scalable.
- Context: Global relationships are modeled directly.
- Modularity: Stacking layers and attention heads makes it flexible.
- Generalization: The same building blocks power BERT, GPT, Gemini, Claude, etc.
