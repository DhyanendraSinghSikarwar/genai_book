# Appendix C: Glossary of 150 LLM Terms

A quick, plain-English dictionary of the terms used throughout this book. Skim it now, then come back whenever a word trips you up. Terms are in alphabetical order.

## A

1. **Activation** — The output value of a neuron after it processes its input.
2. **Adapter** — A small set of extra weights added to a model during fine-tuning (as in LoRA).
3. **Agent** — An LLM that can plan, call tools, and take multiple steps to finish a task.
4. **Alignment** — Training a model to behave the way humans want (helpful, honest, harmless).
5. **API (Application Programming Interface)** — A way for your code to call a model over the internet.
6. **Attention** — The mechanism that lets a model focus on the most relevant words in the input.
7. **AutoRegressive** — Generating text one token at a time, each based on the ones before it.
8. **AWQ** — A quantization method that shrinks models while keeping quality.

## B

9. **Backpropagation** — The algorithm that adjusts a model's weights during training.
10. **Base model** — A model that has been pretrained but not yet instruction-tuned.
11. **Batch** — A group of inputs processed together for speed.
12. **BERT** — An early Transformer model good at understanding text (not generating).
13. **Bias (fairness)** — Unfair or skewed behavior a model learned from data.
14. **Bias (math)** — A learnable constant added inside a neuron.
15. **BF16 (bfloat16)** — A 16-bit number format used to train models efficiently.
16. **bitsandbytes** — A library that enables 4-bit and 8-bit model loading.

## C

17. **Chain-of-Thought (CoT)** — Prompting a model to reason step by step before answering.
18. **Chunk** — A small piece of a document, used in RAG.
19. **Chunking** — Splitting documents into chunks before embedding them.
20. **Context window** — The maximum number of tokens a model can read at once.
21. **CUDA** — NVIDIA's software that lets GPUs run deep-learning math.
22. **Cosine similarity** — A common way to measure how close two vectors (meanings) are.

## D

23. **Dataset** — The collection of examples used to train or evaluate a model.
24. **Decoder** — The part of a Transformer that generates text.
25. **Deployment** — Making a model available for real use.
26. **Distillation** — Training a small model to copy a bigger one.
27. **DPO (Direct Preference Optimization)** — A simpler way than RLHF to align a model using preferred vs rejected answers.
28. **Dropout** — A training trick that randomly ignores neurons to prevent overfitting.

## E

29. **Embedding** — A list of numbers that represents the meaning of text.
30. **Embedding model** — A model that turns text into embeddings.
31. **Encoder** — The part of a Transformer that reads and understands input.
32. **Epoch** — One full pass through the training data.
33. **Evaluation (eval)** — Measuring how good a model's outputs are.

## F

34. **Few-shot** — Giving the model a few examples in the prompt to guide it.
35. **Fine-tuning** — Training an existing model further on your own data.
36. **FlashAttention** — A faster, memory-saving way to compute attention.
37. **FP16 (float16)** — A 16-bit number format for model weights.
38. **FP32 (float32)** — Full 32-bit precision; accurate but memory-heavy.
39. **Foundation model** — A large model trained on broad data, used as a starting point.
40. **Frozen weights** — Weights kept unchanged during fine-tuning.

## G

41. **GGUF** — A file format for quantized models that run locally (llama.cpp/Ollama).
42. **GPT (Generative Pretrained Transformer)** — A family of text-generating models.
43. **GPTQ** — A popular post-training quantization method.
44. **GPU (Graphics Processing Unit)** — Hardware that runs model math fast.
45. **Gradient** — A signal showing how to change weights to reduce error.
46. **Gradient descent** — The method of nudging weights to lower the loss.
47. **Greedy decoding** — Always picking the single most likely next token.
48. **Grounding** — Basing a model's answer on real supplied documents.
49. **Guardrails** — Rules that block unsafe or unwanted model behavior.

## H

50. **Hallucination** — When a model states something false as if it were true.
51. **Head (attention head)** — One of several parallel attention mechanisms in a layer.
52. **Hugging Face** — A platform hosting models, datasets, and libraries.
53. **Hyperparameter** — A setting you choose before training (like learning rate).

## I

54. **In-context learning** — The model learning a task just from examples in the prompt.
55. **Inference** — Using a trained model to produce outputs.
56. **Instruction tuning** — Fine-tuning a model to follow instructions.
57. **INT4 / INT8** — 4-bit and 8-bit integer formats used in quantization.

## J

58. **JSON mode** — Forcing a model to output valid JSON.
59. **Jailbreak** — A trick prompt that makes a model ignore its safety rules.

## K

60. **KV cache (key-value cache)** — Stored attention data that speeds up generation but uses VRAM.
61. **k-NN (k nearest neighbors)** — Finding the closest vectors; the core of vector search.

## L

62. **LangChain** — A framework for building RAG apps and agents.
63. **LangGraph** — A framework for building controllable, multi-step agents.
64. **Langfuse** — A tool for tracing and evaluating LLM apps.
65. **Latency** — How long a model takes to respond.
66. **Layer** — One processing stage inside a neural network.
67. **Learning rate** — How big a step training takes when updating weights.
68. **LlamaIndex** — A framework focused on RAG and data connections.
69. **llama.cpp** — A fast C++ engine for running models locally.
70. **LLM (Large Language Model)** — A big model trained to understand and generate text.
71. **LoRA (Low-Rank Adaptation)** — Efficient fine-tuning that trains small adapters.
72. **Loss** — A number measuring how wrong the model's output is.

## M

73. **MCP (Model Context Protocol)** — A standard for connecting models to tools and data.
74. **Mixture of Experts (MoE)** — A model that routes each token to a few specialized sub-networks.
75. **MMLU** — A benchmark testing knowledge across many subjects.
76. **Multimodal** — A model that handles more than text (images, audio).
77. **Multi-head attention** — Running several attention heads in parallel.

## N

78. **NER (Named Entity Recognition)** — Finding names, dates, and places in text.
79. **Neuron** — A basic computing unit in a neural network.
80. **NLP (Natural Language Processing)** — The field of computers working with language.
81. **Normalization** — Scaling numbers to keep training stable (e.g., LayerNorm).

## O

82. **Ollama** — A tool to run LLMs locally with one command.
83. **One-shot** — Giving the model exactly one example in the prompt.
84. **OpenAI-compatible API** — An endpoint that mimics OpenAI's format so tools work with it.
85. **Overfitting** — When a model memorizes training data and fails on new data.

## P

86. **PagedAttention** — A memory trick (used by vLLM) that serves many users efficiently.
87. **Parameter** — A single learnable number (weight) in a model.
88. **PEFT (Parameter-Efficient Fine-Tuning)** — Methods like LoRA that train few parameters.
89. **Perplexity** — A score for how well a model predicts text (lower is better).
90. **pgvector** — A PostgreSQL extension for vector search.
91. **Pinecone** — A managed vector database.
92. **Pipeline** — A sequence of steps chained together.
93. **Positional encoding** — Information that tells the model the order of tokens.
94. **Pretraining** — The first, large-scale training on broad text.
95. **Prompt** — The input text you give a model.
96. **Prompt engineering** — Crafting prompts to get better outputs.
97. **Prompt injection** — An attack where hidden instructions hijack the model.

## Q

98. **QLoRA** — LoRA fine-tuning on a 4-bit base model; very memory-efficient.
99. **Quantization** — Shrinking a model by using lower-precision numbers.
100. **Query (retrieval)** — The user's question used to search the vector DB.
101. **Qdrant** — An open-source vector database.

## R

102. **RAG (Retrieval-Augmented Generation)** — Fetching documents before answering.
103. **RAGAS** — A library for evaluating RAG systems.
104. **Re-ranking** — Reordering retrieved chunks to put the best ones first.
105. **ReAct** — An agent pattern of Reasoning + Acting in a loop.
106. **Recall** — How many of the truly relevant items your search found.
107. **Reinforcement Learning** — Training by rewards and penalties.
108. **RLHF (RL from Human Feedback)** — Aligning models using human preference ratings.
109. **RoPE (Rotary Position Embedding)** — A popular way to encode token positions.

## S

110. **Sampling** — Randomly picking the next token based on probabilities.
111. **Self-attention** — Attention where tokens attend to other tokens in the same sequence.
112. **Semantic search** — Searching by meaning (via embeddings) rather than keywords.
113. **SFT (Supervised Fine-Tuning)** — Fine-tuning on prompt-answer pairs.
114. **smolagents** — A lightweight Hugging Face agent framework.
115. **Softmax** — A function that turns scores into probabilities.
116. **System prompt** — Instructions that set the model's role and rules.

## T

117. **Temperature** — A setting controlling randomness (higher = more creative).
118. **Tensor** — A multi-dimensional array of numbers.
119. **TGI (Text Generation Inference)** — Hugging Face's production serving engine.
120. **Throughput** — How many requests a system handles per second.
121. **Token** — A chunk of text (word or word-part) the model processes.
122. **Tokenizer** — The tool that splits text into tokens.
123. **Tool calling** — When a model asks to run a function/API.
124. **Top-k** — Sampling only from the k most likely tokens.
125. **Top-p (nucleus)** — Sampling from the smallest set of tokens whose probability adds up to p.
126. **Transformer** — The neural network architecture behind modern LLMs.
127. **TRL** — A library for SFT, DPO, and RLHF training.

## U

128. **Underfitting** — When a model is too simple and learns too little.
129. **Unsloth** — A library for fast, memory-light fine-tuning.

## V

130. **Vector** — A list of numbers; here, a text embedding.
131. **Vector database** — A store that finds vectors by similarity.
132. **vLLM** — A high-throughput engine for serving models in production.
133. **VRAM** — The memory on a GPU; the main limit for model size.

## W

134. **Weights** — The learned numbers that make up a model.
135. **Weights & Biases (W&B)** — A tool to track and compare training runs.
136. **Weaviate** — An open-source vector database.
137. **Window (sliding)** — Processing long text in overlapping pieces.

## X, Y, Z and Extras

138. **Zero-shot** — Asking a model to do a task with no examples given.
139. **Chroma** — A simple local vector database.
140. **Cosine distance** — 1 minus cosine similarity; smaller means more alike.
141. **Context stuffing** — Putting retrieved text directly into the prompt.
142. **Endpoint** — The URL your app calls to reach a model.
143. **Fine-tune dataset** — The prompt-answer pairs used for fine-tuning.
144. **Guard model** — A separate model that checks inputs/outputs for safety.
145. **Hybrid search** — Combining keyword search and semantic search.
146. **Idempotent** — An operation that gives the same result if repeated.
147. **Judge model (LLM-as-a-judge)** — A model used to grade other models' answers.
148. **Rate limit** — A cap on how many requests you can send in a time window.
149. **Retriever** — The component that fetches relevant chunks in RAG.
150. **Streaming** — Sending the answer token by token as it is generated.
