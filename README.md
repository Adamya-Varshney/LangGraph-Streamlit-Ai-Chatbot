# LangGraph Streamlit AI Chatbot

An end‑to‑end conversational AI built using **LangGraph** and **Streamlit**.  This project demonstrates how to combine a graph‑based language model workflow with modern web UI components to build multi‑threaded chat experiences, stream large language model (LLM) responses, persist conversations via checkpoints, invoke external tools, and retrieve information from uploaded documents.  It is designed as a reference implementation that can be adapted for educational, prototyping, or product development purposes.

## Contents

This repository contains several back‑end variants and matching Streamlit front‑ends.  Each variant illustrates a different capability of the LangGraph ecosystem:

| File | Purpose |
| --- | --- |
| **`langgraph_backend.py`** | Minimal example that constructs a chat graph using an in‑memory checkpointer.  Suitable when you only need a single session or do not require persistence across restarts. |
| **`langgraph_database_backend.py`** | Uses an SQLite checkpointer to store conversation state and provides a helper to list all thread IDs.  Enables persistent, multi‑threaded conversations across sessions. |
| **`langgraph_tool_backend.py`** | Adds built‑in tools such as a DuckDuckGo search wrapper, a stock price fetcher, and a calculator.  The graph uses conditional edges so the LLM can decide when to call a tool. |
| **`langgraph_mcp_backend.py`** | Demonstrates how to load tools from remote Multi‑Server MCP services.  It runs the LLM asynchronously and streams responses while tools execute. |
| **`langraph_rag_backend.py`** | Extends the tool backend with a Retrieval‑Augmented Generation (RAG) pipeline.  Uploaded PDFs are ingested into a FAISS vector store, and a `rag_tool` is available to answer document‑specific questions. |
| **`streamlit_frontend_database.py`** | Streamlit UI for the database backend.  Supports creating new chats, selecting past conversations, and streaming model responses. |
| **`streamlit_frontend_tool.py`** | Streamlit UI for the tool backend.  Shows tool usage in real time and updates the chat UI as the LLM invokes tools. |
| **`streamlit_frontend_mcp.py`** | Streamlit UI for the MCP backend.  Handles asynchronous streaming and displays status indicators while remote tools run. |
| **`streamlit_rag_frontend.py`** | Streamlit UI for the RAG backend.  Lets users upload PDFs, displays document metadata, and allows queries that combine RAG with other tools. |

## Key Features

### Multi‑Threaded Conversations

* The project assigns a unique thread ID (UUID) for each conversation.  When running a Streamlit front‑end, users can start new chats or select existing thread IDs from the sidebar.  Threads are persisted when using the database or tool back‑ends, enabling chat history to survive restarts.
* Functions like `retrieve_all_threads()` (defined in several back‑end files) enumerate existing threads by reading from the underlying checkpointer.

### Stateful Graph of LLM Calls

* **LangGraph** is used to construct directed graphs where each node is a callable that consumes and produces a state object.  A `ChatState` typed dictionary holds the list of messages, and the graph automatically manages message accumulation via the `add_messages` annotation.
* Checkpointers (`InMemorySaver`, `SqliteSaver`, or `AsyncSqliteSaver`) persist the node states between steps, making it possible to restore and continue conversations.

### Streaming LLM Responses

* All Streamlit UIs use the `chatbot.stream` or `chatbot.astream` methods.  These APIs yield partial responses as soon as they are available, leading to a responsive chat experience.  Streamlit’s `st.write_stream` helper is used to render streaming content line by line.

### Tool Integration

* The tool back‑end (`langgraph_tool_backend.py`) and MCP back‑end (`langgraph_mcp_backend.py`) register custom tools using the `@tool` decorator.  These tools include:
  * **Web search:** a DuckDuckGo search wrapper (`DuckDuckGoSearchRun`) that fetches current information from the web.
  * **Stock price lookup:** calls the AlphaVantage API to retrieve real‑time stock quotes.
  * **Calculator:** performs basic arithmetic (add, sub, mul, div) and returns the result as a JSON object.
* Tools are bound to the LLM via `llm.bind_tools(tools)`.  Conditional edges in the graph (`tools_condition`) allow the model to decide when to call a tool.  In the UI, tool invocations are annotated with status indicators.

### Retrieval‑Augmented Generation (RAG)

* The RAG backend (`langraph_rag_backend.py`) introduces a `rag_tool` that enables the LLM to answer questions about uploaded PDF documents.  Key components include:
  * **Ingestion:** uploaded PDFs are loaded with `PyPDFLoader`, split into chunks via a `RecursiveCharacterTextSplitter`, and stored in a FAISS vector store using `OpenAIEmbeddings`.
  * **Retriever:** each thread maintains its own FAISS retriever.  When the `rag_tool` is invoked with a query and thread ID, it retrieves the most relevant chunks and returns their content and metadata.
  * **System prompt:** the `chat_node` prepends a system message instructing the LLM to call `rag_tool` for document‑related questions, ensuring the model knows how to use the retrieval capability.

### Streamlit Front‑End Design

* Every UI file establishes a sidebar with conversation management controls (new chat, thread selection).  The main chat area uses `st.chat_message` to display messages with appropriate roles.  Chat input is captured via `st.chat_input`.
* Status updates during tool execution are presented using `st.status`, so users can see when a tool is running and when it completes.
* The RAG front‑end includes a file uploader, displays document metadata, and indicates which document is being used for the current chat thread.

## Contributing

Contributions are welcome! If you find a bug, have a question, or would like to propose a feature, feel free to open an issue or submit a pull request. Suggestions for additional tools or front‑end improvements are particularly appreciated.

When contributing, please:

* Fork the repository and create a new branch for your feature or fix.

* Follow the existing code style and include docstrings where appropriate.

* Write a clear description of what your change accomplishes.

* Ensure that the application still runs correctly in the different modes.

## License

This project is licensed under the Apache License 2.0. See the LICENSE
 file for details.

## Acknowledgements

This chatbot demonstrates the capabilities of the LangGraph framework and draws on the ecosystem of LangChain components and open‑source tools. Credit goes to the maintainers of these libraries and to the original repository creator for assembling this example.
