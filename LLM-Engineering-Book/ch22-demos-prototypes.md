# Chapter 22: Demos & Prototypes

Before you build a full production system, you usually need to *show* something. A working demo convinces your boss, your client, or your investors far better than a slide deck. The good news: with modern Python tools you can turn an LLM into a clickable web app in minutes, and share it with the world for free.

This chapter covers three tools that make demos fast: **Gradio** (chat UIs in a few lines), **Streamlit** (data-app style chat with memory), and **Hugging Face Spaces** (free hosting). We finish with a real story: shipping a stakeholder demo in a single afternoon.

---

## 22.1 Gradio: ChatInterface in 10 Lines

**Simple definition:** Gradio is a Python library that turns a function into a web interface automatically. For chat apps, its `ChatInterface` gives you a full chat window with almost no code.

**Intuitive explanation / analogy:** Gradio is like a "magic wrapper." You write a normal Python function that takes a message and returns a reply. Gradio wraps it in a polished web page — with a text box, a send button, and chat history — so anyone can use it in a browser. You focus on the logic; Gradio builds the UI.

**Runnable code — a full chat app in about 10 lines:**

```python
# Install: pip install gradio openai
import gradio as gr
from openai import OpenAI

client = OpenAI()  # uses your OPENAI_API_KEY (or point base_url to Ollama/local)

# This function receives the new message + the past conversation
def chat(message, history):
    # Rebuild the conversation in OpenAI format from Gradio's history
    messages = [{"role": "system", "content": "You are a helpful assistant."}]
    for turn in history:
        messages.append(turn)          # each turn is {"role":..., "content":...}
    messages.append({"role": "user", "content": message})

    reply = client.chat.completions.create(model="gpt-4o-mini", messages=messages)
    return reply.choices[0].message.content

# ChatInterface builds the entire chat UI around our function
gr.ChatInterface(chat, type="messages", title="My First LLM Demo").launch()
```

Running this opens a chat window in your browser. Add `share=True` to `launch()` and Gradio gives you a temporary public link you can send to anyone.

**Use case — Finance:** A fintech engineer wraps a "explain-this-financial-term" function in a Gradio ChatInterface and shares the public link with the marketing team to gather feedback before building the real app.

**Use case — E-commerce:** A developer builds a Gradio demo of a "find-me-a-gift" assistant in 15 minutes and screen-shares it in a meeting; stakeholders type questions live and immediately see value.

---

## 22.2 Streamlit: Chat App with Session State

**Simple definition:** Streamlit is a Python library for building data and AI web apps by writing simple scripts. It has built-in chat components and a "session state" to remember the conversation.

**Intuitive explanation / analogy:** Streamlit reruns your whole script every time the user does something (like clicking send). Because the script restarts, you need a place to remember things between runs — that place is `st.session_state`, a little backpack that survives each rerun. You put the chat history in the backpack so it is not forgotten.

**Runnable code — a chat app that remembers the conversation:**

```python
# Install: pip install streamlit openai
# Run with: streamlit run app.py
import streamlit as st
from openai import OpenAI

client = OpenAI()
st.title("Product Q&A Assistant")

# 1. Create the memory backpack on first run
if "messages" not in st.session_state:
    st.session_state.messages = [
        {"role": "system", "content": "You answer questions about our store's products."}
    ]

# 2. Show the conversation so far (skip the hidden system message)
for msg in st.session_state.messages[1:]:
    st.chat_message(msg["role"]).write(msg["content"])

# 3. Get new input from the user
if prompt := st.chat_input("Ask about a product..."):
    st.session_state.messages.append({"role": "user", "content": prompt})
    st.chat_message("user").write(prompt)

    # 4. Get the model's reply using the full remembered history
    reply = client.chat.completions.create(
        model="gpt-4o-mini", messages=st.session_state.messages
    ).choices[0].message.content

    st.session_state.messages.append({"role": "assistant", "content": reply})
    st.chat_message("assistant").write(reply)
```

Because history lives in `st.session_state`, the model sees the whole conversation and can answer follow-up questions like "and does it come in blue?"

**Use case — Finance:** An analyst builds a Streamlit app where users chat with a model about a quarterly report; session state lets them ask follow-ups ("what about the second quarter?") without repeating context.

**Use case — E-commerce:** A team ships a Streamlit "product Q&A" app on the company intranet so support agents can ask multi-turn questions about product specs during customer calls.

---

## 22.3 Hugging Face Spaces: Free Deployment Walkthrough

**Simple definition:** Hugging Face Spaces is a free hosting service where you can put a Gradio or Streamlit app online with a public URL, so anyone can use it without installing anything.

**Intuitive explanation / analogy:** A Space is like a free website for your AI app. You upload your code, and Hugging Face runs it on their servers and gives you a shareable link. No servers to manage yourself.

**Walkthrough — deploy a Gradio app to Spaces:**

1. Create a free account on huggingface.co.
2. Click **New Space**. Give it a name, choose **Gradio** (or Streamlit) as the SDK, and pick the free CPU hardware.
3. A Space is just a Git repository. Add two files:

```text
app.py             # your Gradio or Streamlit code (must be named app.py)
requirements.txt   # the Python libraries your app needs
```

```text
# requirements.txt example
gradio
openai
```

4. Push your files (via the web upload button or git):

```bash
# Clone your empty Space, add files, and push
git clone https://huggingface.co/spaces/YOUR_NAME/my-demo
cd my-demo
# ... add app.py and requirements.txt ...
git add . && git commit -m "first demo" && git push
```

5. Spaces automatically installs the requirements and launches your app at
   `https://huggingface.co/spaces/YOUR_NAME/my-demo`. Share that link.

**Tip on secrets:** Never put API keys in your code. In the Space's **Settings → Secrets**, add `OPENAI_API_KEY`; your app reads it as an environment variable safely.

**Use case — Finance:** A student building a portfolio deploys a "budgeting tips" chatbot to a free Space and adds the link to their resume so recruiters can try it instantly.

**Use case — E-commerce:** A small business deploys a "size-recommendation" demo to Spaces and shares the link in a customer survey to test interest before investing in a full build.

---

## 22.4 Use Case: Shipping a Stakeholder Demo in One Afternoon

**The scenario.** It is 1 PM. Tomorrow morning, the leadership of an online electronics store wants to see whether an AI "product expert" could help shoppers. You have one afternoon.

**The plan (afternoon timeline):**

- **1:00 PM — Gather data.** Export a spreadsheet of 200 products (name, price, specs, description) as a CSV.
- **1:30 PM — Load and format.** Write a few lines to turn each product row into a text snippet the model can read.
- **2:00 PM — Build the chat logic.** For each question, include the relevant product text in the prompt so the model answers from *your* catalog, not made-up facts.
- **3:00 PM — Wrap in Streamlit.** Add the chat UI and session state for follow-up questions.
- **4:00 PM — Deploy to Spaces.** Push `app.py` and `requirements.txt`, add the API key as a secret.
- **4:30 PM — Test and polish.** Try real shopper questions, fix the system prompt, done.

**Runnable code — the core of the afternoon demo (e-commerce product Q&A):**

```python
# app.py — a simple product Q&A demo for stakeholders
import streamlit as st, pandas as pd
from openai import OpenAI

client = OpenAI()
st.title("Ask Our Product Expert")

# 1. Load the catalog once and keep it in memory
@st.cache_data
def load_catalog():
    df = pd.read_csv("products.csv")     # name, price, specs, description
    # Turn each row into a readable line for the model
    return "\n".join(
        f"- {r['name']} (${r['price']}): {r['specs']}. {r['description']}"
        for _, r in df.iterrows()
    )

catalog = load_catalog()

# 2. Set up conversation memory with the catalog in the system prompt
if "messages" not in st.session_state:
    st.session_state.messages = [{
        "role": "system",
        "content": (
            "You are a product expert for our electronics store. "
            "Answer ONLY using the catalog below. If it is not in the "
            f"catalog, say you do not carry it.\n\nCATALOG:\n{catalog}"
        ),
    }]

# 3. Show history and take new questions
for m in st.session_state.messages[1:]:
    st.chat_message(m["role"]).write(m["content"])

if q := st.chat_input("What are you shopping for?"):
    st.session_state.messages.append({"role": "user", "content": q})
    st.chat_message("user").write(q)
    ans = client.chat.completions.create(
        model="gpt-4o-mini", messages=st.session_state.messages
    ).choices[0].message.content
    st.session_state.messages.append({"role": "assistant", "content": ans})
    st.chat_message("assistant").write(ans)
```

**The result:** By 5 PM you have a live URL. In the morning, stakeholders type "I need a laptop under $800 for video editing" and watch the assistant recommend real products from the catalog. The demo lands the project — and it took one afternoon and zero infrastructure.

**Finance version of the same pattern:** Swap the product CSV for a table of the firm's financial products (accounts, cards, loan rates). The same code becomes a "which account is right for me?" demo for a bank's product team — built just as fast.

**Why this works:** Demos do not need to be perfect or production-grade. They need to be *real enough to click*. Gradio/Streamlit plus Spaces removes the infrastructure barrier so you spend your time on the idea, not the plumbing.

---

## Key Takeaways

- **Demos win support.** A clickable app convinces stakeholders far more than slides, and modern tools make them fast to build.
- **Gradio** turns a Python function into a chat web app in ~10 lines with `ChatInterface`; add `share=True` for an instant public link.
- **Streamlit** builds richer apps by rerunning your script; use **`st.session_state`** as memory so the chat remembers past turns and handles follow-up questions.
- **Hugging Face Spaces** hosts Gradio/Streamlit apps for free — push `app.py` + `requirements.txt`, store keys in **Secrets**, and share the URL.
- **Ground the model in your data.** Putting your real catalog or documents in the prompt makes answers accurate and specific to your business.
- A complete stakeholder demo (like an **e-commerce product Q&A**) can go from data to live URL in **one afternoon** with no infrastructure to manage.
