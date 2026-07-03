# Chapter 6: Model Context Protocol — MCP

So far we have connected LLMs to prompts, memory, and documents. But real assistants need to *do things*: look up an order in a database, check live inventory, read a file, call an internal API. Every framework used to invent its own way of wiring those connections, and the result was chaos — a different custom integration for every tool and every app.

The **Model Context Protocol (MCP)** fixes this. It is an open standard, introduced by Anthropic in late 2024, for connecting AI applications to external tools and data in one consistent way. In this chapter we explain what MCP is, why it exists, how its pieces fit together, and we build a working MCP server in Python that exposes an e-commerce product and inventory database.

Install the library we use:

```bash
pip install fastmcp
# (or the official SDK: pip install "mcp[cli]")
```

---

## 6.1 What Is MCP and Why It Exists (The USB-C Analogy)

**Simple definition:** MCP is an open protocol — a shared "language and set of rules" — that lets any AI application connect to any external tool or data source in a standard way.

**Intuition — the USB-C analogy:** Think back to the bad old days when every phone, camera, and laptop had its own charger. If you had five devices you needed five different cables. Then USB-C arrived: **one** connector that works with everything. You don't need a special cable per device anymore.

MCP is **USB-C for AI applications**. Before MCP, if you had 3 AI apps and 4 data sources, you might have to build 3 × 4 = 12 custom integrations. With MCP, each data source exposes one standard "MCP port," and each AI app speaks one standard "MCP plug." Now it's 3 + 4 = 7 pieces, and any app works with any source. This is often called turning an "M×N problem" into an "M+N problem."

**Why it matters:**

- **Reusability** — build an MCP server once (say, for your orders database) and *every* MCP-compatible app can use it: Claude Desktop, your IDE, custom agents.
- **Separation of concerns** — the team that owns the database builds and secures the MCP server; app builders just connect to it.
- **Ecosystem** — a growing library of ready-made MCP servers exists for GitHub, Google Drive, Slack, Postgres, and more, so you often don't build from scratch.

```
      WITHOUT MCP (M x N custom integrations)        WITH MCP (M + N)

   App1 ---- DB          App1 \        / DB
   App1 ---- API              \      /
   App2 ---- DB          App2 --- MCP --- API
   App2 ---- API              /      \
   App3 ---- DB          App3 /        \ Files
   ... every line hand-built             one standard hub
```

---

## 6.2 MCP Servers vs Clients; Tools, Resources, and Prompts

### 6.2.1 Servers and clients

**Simple definition:** An **MCP server** *exposes* capabilities (tools and data). An **MCP client** *uses* them, on behalf of an AI application (the "host").

**Intuition:** The server is a restaurant kitchen that can cook certain dishes. The client is the waiter who takes the AI's order to the kitchen and brings the result back. The host is the diner (the AI app, e.g., Claude Desktop) that decides what to order.

- **Host** — the AI application the user interacts with (Claude Desktop, an IDE, your agent).
- **Client** — lives inside the host; maintains a 1-to-1 connection to one server.
- **Server** — a separate program that exposes tools, resources, and prompts.

### 6.2.2 The three things a server can expose

MCP servers offer three kinds of capabilities. Getting these distinctions right is a common interview topic.

| Capability | What it is | Who controls it | E-commerce example |
|---|---|---|---|
| **Tools** | Functions the model can *call* to take an action or fetch live data | Model-controlled (the AI decides when to call) | `get_order_status(order_id)`, `check_inventory(sku)` |
| **Resources** | Read-only data the app can load as *context* | App-controlled (the host decides what to load) | The product catalog file, a customer's order history |
| **Prompts** | Reusable prompt templates the user can trigger | User-controlled (user picks them) | A "/refund-email" template that drafts a refund message |

**Simple definitions:**

- **Tools** are like giving the AI verbs it can perform: "look up," "create," "update." The model chooses to invoke them, similar to function calling.
- **Resources** are like giving the AI nouns to read: files, records, documents. They provide context without side effects.
- **Prompts** are pre-written, parameterized instructions the user can invoke, like slash commands.

**Intuition:** Tools are the AI's *hands* (do things), resources are its *eyes* (read things), and prompts are *recipe cards* the user hands it.

---

## 6.3 Build an MCP Server in Python with FastMCP

Let's build a real MCP server. `fastmcp` makes this delightfully simple: you write normal Python functions and decorate them. We'll build a small e-commerce server that can look up orders and check inventory.

```python
# file: shop_server.py
from fastmcp import FastMCP

# Create the server with a human-readable name.
mcp = FastMCP("ShopEasy Server")

# --- Pretend databases (in real life these are SQL queries or API calls) ---
ORDERS = {
    "1001": {"item": "Wireless Headphones", "status": "shipped", "eta": "2 days"},
    "1002": {"item": "Yoga Mat", "status": "processing", "eta": "5 days"},
}
INVENTORY = {"SKU-HEAD": 42, "SKU-YOGA": 0, "SKU-LAMP": 7}


# --- A TOOL: the model can CALL this to take an action / fetch live data ---
@mcp.tool()
def get_order_status(order_id: str) -> dict:
    """Look up the status and ETA of a customer's order by its order ID."""
    order = ORDERS.get(order_id)
    if order is None:
        return {"error": f"No order found with id {order_id}"}
    return order


@mcp.tool()
def check_inventory(sku: str) -> dict:
    """Check how many units of a product (by SKU) are currently in stock."""
    qty = INVENTORY.get(sku)
    if qty is None:
        return {"error": f"Unknown SKU {sku}"}
    return {"sku": sku, "in_stock": qty, "available": qty > 0}


# --- A RESOURCE: read-only context the host can load ---
@mcp.resource("shop://return-policy")
def return_policy() -> str:
    """The store's official return policy text."""
    return "Items may be returned within 30 days of delivery for a full refund."


# --- A PROMPT: a reusable template the user can trigger ---
@mcp.prompt()
def apology_email(customer_name: str, issue: str) -> str:
    """Generate an instruction to draft an apology email to a customer."""
    return (f"Write a warm, professional apology email to {customer_name} "
            f"about the following issue: {issue}. Offer a 10% discount code.")


if __name__ == "__main__":
    # Run the server. By default it communicates over stdio,
    # which is exactly what Claude Desktop and IDEs expect.
    mcp.run()
```

**What just happened, in plain English:**

- `@mcp.tool()` turns a normal function into a tool the AI can call. The **docstring and type hints matter enormously** — MCP sends them to the model so it knows what the tool does and what arguments it expects. Write clear docstrings.
- `@mcp.resource("shop://return-policy")` exposes read-only data at a URI the host can fetch for context.
- `@mcp.prompt()` exposes a reusable prompt template the user can trigger by name.

**Testing your server locally:**

```bash
# fastmcp ships a dev inspector to click through your tools:
fastmcp dev shop_server.py
```

---

## 6.4 Connecting MCP to Claude / IDEs

An MCP server is useless until a host connects to it. The most common host is **Claude Desktop**, which reads a JSON config file listing the servers to launch.

On the config file (`claude_desktop_config.json`), add your server under `mcpServers`:

```json
{
  "mcpServers": {
    "shopeasy": {
      "command": "python",
      "args": ["/absolute/path/to/shop_server.py"]
    }
  }
}
```

When Claude Desktop starts, it launches this command, connects as an MCP client, and discovers your tools, resources, and prompts automatically. Now a user can type:

> "What's the status of order 1001, and is SKU-YOGA back in stock?"

Claude sees your `get_order_status` and `check_inventory` tools, calls both, and answers in natural language — no extra code from you.

**Connecting from Python (a custom client):** You can also connect programmatically, which is how you'd wire an MCP server into your own agent:

```python
import asyncio
from fastmcp import Client

async def main():
    # Point the client at your server script.
    async with Client("shop_server.py") as client:
        # Discover what the server offers.
        tools = await client.list_tools()
        print("Available tools:", [t.name for t in tools])

        # Call a tool by name with arguments.
        result = await client.call_tool("get_order_status", {"order_id": "1001"})
        print("Order 1001:", result)

asyncio.run(main())
```

**IDE integration:** Editors like Cursor and VS Code (via extensions) support MCP the same way — add the server to their MCP config, and coding assistants can then call your tools (e.g., query your database) right inside the editor.

---

## 6.5 Use Case: A Company Product/Inventory Database Exposed via MCP

Let's make the e-commerce example realistic by connecting to an actual SQLite database instead of a Python dictionary. This is the pattern real companies use: wrap the internal database in an MCP server so any approved AI app can query it safely, with the server enforcing what's allowed.

```python
# file: inventory_mcp.py
import sqlite3
from fastmcp import FastMCP

mcp = FastMCP("Inventory DB Server")
DB = "shop.db"   # a real SQLite database file

def _connect():
    conn = sqlite3.connect(DB)
    conn.row_factory = sqlite3.Row     # rows behave like dicts
    return conn


@mcp.tool()
def search_products(keyword: str, max_price: float | None = None) -> list[dict]:
    """Search the product catalog by keyword, optionally under a max price.
    Returns product name, price, and stock quantity."""
    conn = _connect()
    # NOTE: parameterized query -> prevents SQL injection.
    sql = "SELECT name, price, stock FROM products WHERE name LIKE ?"
    params = [f"%{keyword}%"]
    if max_price is not None:
        sql += " AND price <= ?"
        params.append(max_price)
    rows = conn.execute(sql, params).fetchall()
    conn.close()
    return [dict(r) for r in rows]


@mcp.tool()
def low_stock_report(threshold: int = 5) -> list[dict]:
    """List all products with stock at or below the given threshold,
    so staff can decide what to reorder."""
    conn = _connect()
    rows = conn.execute(
        "SELECT name, stock FROM products WHERE stock <= ? ORDER BY stock",
        (threshold,),
    ).fetchall()
    conn.close()
    return [dict(r) for r in rows]


@mcp.resource("catalog://summary")
def catalog_summary() -> str:
    """A quick read-only summary of the catalog size for context."""
    conn = _connect()
    n = conn.execute("SELECT COUNT(*) AS c FROM products").fetchone()["c"]
    conn.close()
    return f"The catalog currently contains {n} products."


if __name__ == "__main__":
    mcp.run()
```

Now, connected to Claude or a custom agent, a store manager can ask in plain English:

> "Show me products with 'lamp' in the name under $40, and give me a low-stock report so I know what to reorder."

The model calls `search_products("lamp", max_price=40)` and `low_stock_report()`, then writes a clear summary. Several things make this production-appropriate:

- **The server is the security boundary.** Only the two safe, read-oriented tools are exposed — the model can never run arbitrary SQL. Parameterized queries prevent SQL injection.
- **One server, many hosts.** The same server serves Claude Desktop, an internal ops dashboard agent, and an IDE assistant, with no duplicated integration code.
- **Clear docstrings = correct tool use.** Because each tool's docstring and types are precise, the model reliably chooses the right tool with the right arguments.

**Finance parallel:** The identical pattern wraps a bank's transactions database in an MCP server exposing narrow, safe tools like `get_account_balance(account_id)` and `list_recent_transactions(account_id, days)` — never raw SQL — so an AI assistant can answer customer questions while the server enforces exactly what data is reachable.

---

## 6.6 Interview Q&A: MCP vs Function Calling

**Q1. Isn't MCP just function calling with extra steps? What's the real difference?**

**A:** Function calling is a *model capability*: the LLM can output a request to call a function whose schema you provided in that API call. MCP is a *protocol and architecture*: a standard way to package tools (and resources and prompts) into reusable servers that any MCP-compatible app can discover and connect to. Function calling is *how* the model asks to use a tool; MCP is *how* tools are defined, hosted, discovered, and shared across applications. In practice, MCP tools are executed via the model's function-calling ability — they complement each other rather than compete.

**Q2. Why not just define my tools inline with the API each time instead of running an MCP server?**

**A:** Inline definitions are fine for one app with a few tools. But they get re-implemented in every application and every language, creating the M×N integration explosion. MCP lets you build a tool server *once* and reuse it across Claude Desktop, IDEs, and custom agents. It also cleanly separates ownership: the data team maintains and secures the server; app teams just connect.

**Q3. What are the three capabilities an MCP server can expose, and who controls each?**

**A:** **Tools** (model-controlled — the AI decides when to call them to act or fetch live data), **Resources** (application-controlled — the host loads them as read-only context), and **Prompts** (user-controlled — the user triggers reusable templates). The control distinction is the key: tools have potential side effects and are chosen by the model, whereas resources are passive context.

**Q4. What are the main benefits of MCP over ad-hoc integrations?**

**A:** Standardization (one protocol instead of many custom connectors), reusability (build a server once, use it everywhere), a growing ecosystem of ready-made servers, cleaner separation of concerns and security boundaries, and interoperability across vendors since it's an open standard.

**Q5. How does MCP improve security compared to giving an LLM raw database access?**

**A:** The MCP server is an explicit boundary. Instead of exposing a database or shell to the model, you expose only a small set of narrow, well-defined tools (e.g., `search_products`), with validation, parameterized queries, and permission checks inside the server. The model can only do what the server permits, which is far safer than open-ended access. That said, MCP servers must still be trusted and sandboxed appropriately, since a malicious server is itself a risk.

**Q6. What transport does an MCP server use, and does it have to run locally?**

**A:** MCP commonly uses **stdio** for local servers (the host launches the server as a subprocess) and HTTP-based transports (such as streamable HTTP / SSE) for remote servers. So a server can run locally next to the app or be hosted remotely and shared across an organization.

**Q7. If I already use LangChain tools or OpenAI function calling, can I use MCP too?**

**A:** Yes. MCP is complementary. You can wrap MCP tools as LangChain tools or expose them to any agent framework, so your existing agents gain access to the whole MCP ecosystem. Frameworks increasingly ship built-in MCP adapters for exactly this.

**Q8. Give a concrete scenario where MCP clearly wins over plain function calling.**

**A:** A company wants its orders database usable by (a) Claude Desktop for support staff, (b) an IDE assistant for engineers, and (c) a custom customer-facing agent. With plain function calling you'd re-implement and re-secure the integration three times. With MCP you build one `orders` server with a few safe tools; all three hosts connect to it, share improvements, and benefit from centralized security and maintenance. That's the M+N advantage in action.

---

## Key Takeaways

- **MCP (Model Context Protocol)** is an open standard — "USB-C for AI" — that lets any AI app connect to any tool or data source in one consistent way, turning an M×N integration problem into M+N.
- The architecture is **host → client → server**: the host is the AI app, the client is its connector, and the server exposes capabilities.
- Servers expose three things: **tools** (model-controlled actions/live data), **resources** (app-controlled read-only context), and **prompts** (user-triggered templates).
- With **FastMCP**, you build a server by decorating plain Python functions — clear docstrings and type hints are essential because the model reads them.
- Hosts like **Claude Desktop and IDEs** connect via a simple JSON config; you can also connect programmatically to embed MCP in custom agents.
- Wrapping a **company database** in an MCP server exposes only narrow, safe tools — the server is your security boundary (e-commerce inventory, banking balances).
- **MCP vs function calling:** function calling is *how* the model requests a tool; MCP is the *standard for defining, hosting, discovering, and sharing* tools across apps. They work together.
