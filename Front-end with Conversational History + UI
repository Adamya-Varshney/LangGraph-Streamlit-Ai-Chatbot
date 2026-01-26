import streamlit as st
from langgraph_tool_backend import chatbot
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
import uuid

# **************************************** utility functions *************************

def generate_thread_id():
    thread_id = uuid.uuid4()
    return thread_id

def add_thread(thread_id):
    if thread_id not in st.session_state['chat_threads']:
        st.session_state['chat_threads'].append(thread_id)

def reset_chat():
    thread_id = generate_thread_id()
    st.session_state['thread_id'] = thread_id
    add_thread(st.session_state['thread_id'])
    st.session_state['message_history'] = []
    # Title will be generated when the first user message is sent.

def load_conversation(thread_id):
    state = chatbot.get_state(config={'configurable': {'thread_id': thread_id}})
    # Check if messages key exists in state values, return empty list if not
    return state.values.get('messages', [])

def generate_title_llm(first_user_message: str) -> str:
    """
    Generate a short, subject-based title for a conversation using the LLM.
    Uses a separate temporary thread_id so we don't pollute the user's real chat memory.
    """
    temp_thread_id = f"title-{uuid.uuid4()}"
    prompt = (
        "You create short conversation titles.\n"
        "Return ONE concise title (3–7 words) summarizing the user's message.\n"
        "No quotes. No punctuation at the end. Title case.\n"
        f"User message: {first_user_message}"
    )

    resp = chatbot.invoke(
        {"messages": [
            SystemMessage(content="You are a helpful assistant that only writes short conversation titles."),
            HumanMessage(content=prompt)
        ]},
        config={"configurable": {"thread_id": temp_thread_id}}
    )

    title = resp["messages"][-1].content.strip()
    # Small safety cleanup
    title = title.strip('"').strip("'").strip()
    if not title:
        title = "New Conversation"
    return title

def ensure_thread_title(thread_id, user_input: str):
    """
    Store a title for this thread_id if it doesn't exist yet.
    Title is based on the first user message of the thread.
    """
    if thread_id not in st.session_state["thread_titles"]:
        st.session_state["thread_titles"][thread_id] = generate_title_llm(user_input)

def ensure_thread_title_from_loaded_messages(thread_id):
    """
    If we loaded a thread (e.g., from sidebar) and no title exists,
    generate it from the first HumanMessage in the saved conversation.
    """
    if thread_id in st.session_state["thread_titles"]:
        return

    messages = load_conversation(thread_id)
    first_user = None
    for msg in messages:
        if isinstance(msg, HumanMessage) and getattr(msg, "content", "").strip():
            first_user = msg.content.strip()
            break

    if first_user:
        st.session_state["thread_titles"][thread_id] = generate_title_llm(first_user)
    else:
        st.session_state["thread_titles"][thread_id] = "New Conversation"


# **************************************** Session Setup ******************************

if 'message_history' not in st.session_state:
    st.session_state['message_history'] = []

if 'thread_id' not in st.session_state:
    st.session_state['thread_id'] = generate_thread_id()

if 'chat_threads' not in st.session_state:
    st.session_state['chat_threads'] = []

# NEW: store titles per thread
if "thread_titles" not in st.session_state:
    st.session_state["thread_titles"] = {}

add_thread(st.session_state['thread_id'])


# **************************************** Sidebar UI *********************************

st.sidebar.title('LangGraph Chatbot')

if st.sidebar.button('New Chat'):
    reset_chat()

st.sidebar.header('My Conversations')

for thread_id in st.session_state['chat_threads'][::-1]:
    # Ensure a title exists for older threads when we display them
    if thread_id not in st.session_state["thread_titles"]:
        ensure_thread_title_from_loaded_messages(thread_id)

    title = st.session_state["thread_titles"].get(thread_id, "New Conversation")

    # Use a unique key so Streamlit doesn't collide if titles repeat
    if st.sidebar.button(title, key=f"thread_btn_{str(thread_id)}"):
        st.session_state['thread_id'] = thread_id
        messages = load_conversation(thread_id)

        temp_messages = []
        for msg in messages:
            if isinstance(msg, HumanMessage):
                role = 'user'
            else:
                role = 'assistant'
            temp_messages.append({'role': role, 'content': msg.content})

        st.session_state['message_history'] = temp_messages


# **************************************** Main UI ************************************

# loading the conversation history
for message in st.session_state['message_history']:
    with st.chat_message(message['role']):
        st.text(message['content'])

user_input = st.chat_input('Type here')

if user_input:
    # If this is the first user message in this thread, generate a title now (LLM-based)
    if len(st.session_state["message_history"]) == 0:
        ensure_thread_title(st.session_state["thread_id"], user_input)

    # first add the message to message_history
    st.session_state['message_history'].append({'role': 'user', 'content': user_input})
    with st.chat_message('user'):
        st.text(user_input)

    CONFIG = {'configurable': {'thread_id': st.session_state['thread_id']}}

    with st.chat_message("assistant"):
        def ai_only_stream():
            for message_chunk, metadata in chatbot.stream(
                {"messages": [HumanMessage(content=user_input)]},
                config=CONFIG,
                stream_mode="messages"
            ):
                if isinstance(message_chunk, AIMessage):
                    # yield only assistant tokens
                    yield message_chunk.content

        ai_message = st.write_stream(ai_only_stream())

    st.session_state['message_history'].append({'role': 'assistant', 'content': ai_message})
