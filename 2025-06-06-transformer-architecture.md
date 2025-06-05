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
