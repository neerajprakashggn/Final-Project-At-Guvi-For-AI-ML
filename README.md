
"""
Project 1:Agentic AI / RAG-Based Natural Language MongoDB Querying and MongoDB-to-MySQL Data Migration System
This app combines multiple technologies to make database access easier for non-technical users:
- MongoDB Atlas is used as the source database (sample_mflix dataset)
- Groq is used to generate natural-language-to-query logic
- Streamlit provides the interactive web interface
- TiDB/MySQL is used for migration and data transfer workflow

The main idea is simple: the user asks a question in plain English,
AI converts that request into a MongoDB query, and the application executes it.
"""

# Import required libraries
import streamlit as st
import pymongo
import json
import re
import os
import pandas as pd
import pymysql
from groq import Groq
from bson import ObjectId
import datetime
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

# ─── Environment Configuration ───────────────────────────────────────────────────
# These values are loaded from the .env file and are used to connect to the services.
# MongoDB and Groq credentials are required; TiDB credentials are checked when migration starts.
MONGO_URI = os.getenv("MONGO_URI", "")
GROQ_API_KEY = os.getenv("GROQ_API_KEY", "")
GROQ_MODEL = os.getenv("GROQ_MODEL", "qwen/qwen3.8-27b")

#_____TiDb Login Credentials_____
TIDB_HOST = os.getenv("TIDB_HOST", "")
TIDB_PORT = int(os.getenv("TIDB_PORT", "4000"))
TIDB_USER = os.getenv("TIDB_USER", "")
TIDB_PASSWORD = os.getenv("TIDB_PASSWORD", "")
TIDB_DATABASE = os.getenv("TIDB_DATABASE", "")

# ─── Page Config ────────────────────────────────────────────────────────────────
# Streamlit page Title metadata for branding and layout.
st.set_page_config(page_title=" AI Based Query System on MongoDB ", page_icon="🤖", layout="wide")

# ─── Custom CSS ─────────────────────────────────────────────────────────────────
# Fixed styling for a clean, focused Streamlit dashboard without extra theme controls.
st.markdown("""
<style>
    .stApp { background: #0f172a; color: #e5e7eb; }
    .stApp .block-container { padding-top: 1.5rem; padding-bottom: 2rem; }
    .stTabs [data-baseweb="tab-list"] {
        display: flex;
        flex-wrap: wrap;
        gap: 0.35rem;
        background: transparent;
        border: none;
        padding: 0;
        margin-bottom: 0.75rem;
    }
    .stTabs [data-baseweb="tab"] {
        background: #111827;
        border: 1px solid #1f2937;
        border-bottom: 2px solid #1f2937;
        border-radius: 8px 8px 0 0;
        padding: 0.6rem 1rem;
        min-height: 48px;
        flex: 0 1 auto;
        white-space: nowrap;
        justify-content: center;
        align-items: center;
        font-weight: 700;
        color: #cbd5e1;
        box-shadow: none;
    }
    .stTabs [data-baseweb="tab"][aria-selected="true"] {
        background: #111827;
        border-color: #1f2937;
        border-bottom: 3px solid #60a5fa;
        color: #e5e7eb;
        box-shadow: inset 0 -2px 0 #60a5fa;
    }
    .stTab { background: transparent; }
    .migration-title {
        font-size: 1.8rem;
        line-height: 1.2;
        margin: 0 0 0.4rem 0;
        font-weight: 700;
    }
    .status-card {
        background: linear-gradient(135deg, rgba(37,99,235,0.12), rgba(14,165,233,0.06));
        border: 1px solid #1f2937;
        border-left: 4px solid #60a5fa;
        border-radius: 12px;
        padding: 0.2rem 1rem;
        width: 100%;
        max-width: 320px;
        margin: 0.15rem auto 0.25rem;
        color: #e5e7eb;
    }
    .result-card {
        background: #111827;
        border: 1px solid #1f2937;
        border-left: 5px solid #60a5fa;
        border-radius: 10px;
        padding: 0.9rem 1rem;
        margin: 0.75rem 0;
        color: #e5e7eb;
    }
    .error-card {
        background: rgba(239, 68, 68, 0.08);
        border: 1px solid rgba(239,68,68,0.5);
        border-left: 5px solid #ef4444;
        border-radius: 10px;
        padding: 0.9rem 1rem;
        margin: 0.75rem 0;
        color: #e5e7eb;
    }
    .success-card {
        background: rgba(34, 197, 94, 0.08);
        border: 1px solid rgba(34,197,94,0.5);
        border-left: 5px solid #22c55e;
        border-radius: 10px;
        padding: 0.9rem 1rem;
        margin: 0.75rem 0;
        color: #e5e7eb;
    }
    .warning-card {
        background: rgba(245, 158, 11, 0.08);
        border: 1px solid rgba(245,158,11,0.5);
        border-left: 5px solid #f59e0b;
        border-radius: 10px;
        padding: 0.9rem 1rem;
        margin: 0.75rem 0;
        color: #e5e7eb;
    }
    .query-box { background: #111827; color: #e5e7eb; padding: 12px; border-radius: 8px; font-family: monospace; font-size: 14px; }
</style>
""", unsafe_allow_html=True)

# ─── Sidebar: Application Information ─────────────────────────────────────────────
# The sidebar gives a short overview of the project for the user.
with st.sidebar:
    st.header("ℹ️ About")
    st.markdown("**Project 1 — AI based MongoDB Query System**")
    st.markdown("Ask questions in Natural English and The AI generates and runs the MongoDB query.")
    st.markdown("---")
    st.markdown("**Database:** MongoDB Atlas `sample_mflix`")
    st.markdown(f"**AI Model:** {GROQ_MODEL} via Groq")

# Read variables once for easier reuse in the app.
mongo_uri = MONGO_URI
groq_key = GROQ_API_KEY

# If required keys are missing, stop the app before any database work starts.
if not mongo_uri or not groq_key:
    st.error("Set MONGO_URI and GROQ_API_KEY in .env before starting the app.")
    st.stop()

# ─── Helper Functions ─────────────────────────────────────────────────────────────
# These functions support data conversion, database access, and AI response processing.


def json_safe(obj):
    """Convert MongoDB objects into JSON-friendly data types for display in Streamlit."""
    if isinstance(obj, list):
        return [json_safe(i) for i in obj]
    if isinstance(obj, dict):
        return {k: json_safe(v) for k, v in obj.items()}
    if isinstance(obj, ObjectId):
        return str(obj)
    if isinstance(obj, (datetime.datetime, datetime.date)):
        return obj.isoformat()
    return obj


@st.cache_resource(show_spinner="Connecting to MongoDB…")
def get_mongo_client(uri: str):
    """Create and cache a MongoDB client connection for the app."""
    client = pymongo.MongoClient(uri, serverSelectionTimeoutMS=5000)
    client.admin.command("ping")      # This raises an error if the MongoDB server is unreachable.
    return client


@st.cache_resource(show_spinner="Connecting to TiDB…")
def get_tidb_connection():
    """Create and cache a TiDB/MySQL connection using environment variables."""
    missing_settings = [
        name for name, value in {
            "TIDB_HOST": TIDB_HOST,
            "TIDB_USER": TIDB_USER,
            "TIDB_PASSWORD": TIDB_PASSWORD,
            "TIDB_DATABASE": TIDB_DATABASE,
        }.items() if not value
    ]
    if missing_settings:
        raise RuntimeError(
            "Missing TiDB settings: " + ", ".join(missing_settings) +
            ". Add them to .env, or use a running local MySQL/TiDB server."
        )
    return pymysql.connect(
        host=TIDB_HOST,
        port=TIDB_PORT,
        user=TIDB_USER,
        password=TIDB_PASSWORD,
        database=TIDB_DATABASE,
        ssl={"ca": None},
        ssl_verify_cert=False,
        ssl_verify_identity=False,
    )


def build_schema_context(client, db_name="sample_mflix"):
    """Collect a small schema summary from MongoDB to help the LLM generate valid queries."""
    db = client[db_name]
    context_parts = []
    for coll_name in db.list_collection_names():
        sample = list(db[coll_name].find({}, {"_id": 0}).limit(2))
        if sample:
            fields = list(sample[0].keys())
            context_parts.append(
                f"Collection: `{coll_name}`\n"
                f"Fields: {fields}\n"
                f"Sample doc: {json.dumps(json_safe(sample[0]), default=str)[:400]}"
            )
    return "\n\n".join(context_parts)


def ask_groq(model: Groq, prompt: str) -> str:
    """Send a prompt to Groq and return the generated text response."""
    response = model.chat.completions.create(
        model=GROQ_MODEL,
        messages=[{"role": "user", "content": prompt}],
        max_tokens=1000,
    )
    return response.choices[0].message.content.strip()


def flatten_doc(doc: dict, prefix="") -> dict:
    """Flatten nested MongoDB documents so they fit into a pandas DataFrame for migration."""
    flat = {}
    for key, value in doc.items():
        flat_key = f"{prefix}_{key}" if prefix else key
        if isinstance(value, dict):
            flat.update(flatten_doc(value, flat_key))
        elif isinstance(value, list):
            flat[flat_key] = ", ".join(str(item) for item in value[:5])
        else:
            flat[flat_key] = value
    return flat


def extract_sql_block(text: str) -> str:
    """Extract SQL from a fenced markdown block if the AI response contains one."""
    fenced = re.search(r"```(?:sql)?\s*([\s\S]+?)```", text)
    if fenced:
        return fenced.group(1).strip()
    return text.strip()


def extract_python_query(text: str) -> str:
    """Extract the PyMongo query from the AI response and ignore explanation text."""
    # First, try to read a fenced Python code block.
    fenced = re.search(r"```(?:python)?\s*([\s\S]+?)```", text)
    if fenced:
        return fenced.group(1).strip()

    # Fallback: if the model returns plain text, look for a valid MongoDB command.
    for line in text.splitlines():
        line = line.strip()
        if line.startswith("db.") or line.startswith("collection"):
            return line
    return text.strip()


def execute_query(client, query_code: str, db_name="sample_mflix"):
    """
    Execute a generated PyMongo query intended for read-only operations.
    The query text is generated by the AI and should be validated before production use.
    """
    db = client[db_name]
    allowed_ops = ("find", "aggregate", "count_documents", "distinct")
    if not any(op in query_code for op in allowed_ops):
        return None, "Only read operations (find / aggregate / count_documents) are permitted."

    # Restrict the generated expression's database variable to the selected database.
    local_ns = {"db": db}
    try:
        exec(f"_result = {query_code}", local_ns)   # noqa: S102
        result = local_ns.get("_result")
        if hasattr(result, "__iter__") and not isinstance(result, (dict, str, int, float)):
            return list(result)[:50], None   # Limit the output to keep the app responsive.
        return result, None
    except Exception as exc:
        return None, str(exc)


def render_status_box(title: str, message: str, kind: str = "info"):
    """Render a styled message box for better user feedback."""
    card_class = {
        "info": "result-card",
        "success": "success-card",
        "warning": "warning-card",
        "error": "error-card",
    }.get(kind, "result-card")
    st.markdown(
        f"""
        <div class="{card_class}">
            <div style="font-weight:700; margin-bottom:4px;">{title}</div>
            <div>{message}</div>
        </div>
        """,
        unsafe_allow_html=True,
    )


# ─── Session Memory Setup ────────────────────────────────────────────────────────
# This keeps chat history within the current Streamlit session for conversational context.
if "history" not in st.session_state:
    st.session_state.history = []   # Example structure: [{"role": "user", "content": "..."}]

# ─── Main App Tabs ────────────────────────────────────────────────────────────────
# The application is split into two tabs:
# 1. Query generation from natural-language input
# 2. MongoDB-to-TiDB migration workflow
#tab1, = st.tabs(["Process 1"])
tab1, tab2 = st.tabs(["Process 1", "Process 2"])


with tab1:
    # ── Tab 1: Natural Language MongoDB Query Interface ─────────────────────────
    # This section lets the user ask questions in plain English and see the AI-generated
    # MongoDB query executed against the sample_mflix dataset.
    st.markdown("<div class='migration-title'>🤖 AI-Powered MongoDB Query System</div>", unsafe_allow_html=True)
    st.caption("Ask questions in natural English — the AI generates and runs the MongoDB query for you.")

    # ── Initialise /Making Connection to MongoDB and Groq clients ──────────────────────────
    try:
        client = get_mongo_client(mongo_uri)
    except Exception as e:
        st.error(f"❌ MongoDB connection failed: {e}")
        st.stop()

    try:
        model = Groq(api_key=groq_key)
    except Exception as e:
        st.error(f"❌ Groq initialisation failed: {e}")
        st.stop()

    col1, col2, col3 = st.columns(3)
    with col1:
        st.markdown("<div class='status-card'><b>MongoDB</b>&nbsp;✅ Connected</div>", unsafe_allow_html=True)
    with col2:
        st.markdown("<div class='status-card'><b>Groq</b>&nbsp;✅ Ready</div>", unsafe_allow_html=True)
    with col3:
        st.markdown("<div class='status-card'><b>Dataset:</b>&nbsp;sample_mflix</div>", unsafe_allow_html=True)

    # ── Build the retrieval context used by the AI model ────────────────────────
    # The schema summary helps the model understand the available collections and fields.
    with st.spinner("📚 Loading schema context (RAG)…"):
        schema_ctx = build_schema_context(client)

    with st.expander("📋 View RAG Schema Context"):
        st.text(schema_ctx)

    st.markdown("---")

    st.subheader("💬 Ask a question")
    st.markdown(
        "Examples: \n"
        "- Find top 10 movies by IMDb rating\n"
        "- Show action movies released after 2015\n"
        "- Find movies directed by Christopher Nolan"
    )

    # ── Load previous chat messages from the session state ───────────────────────
    for msg in st.session_state.history:
        with st.chat_message(msg["role"]):
            st.markdown(msg["content"])

    # ── User input box for natural-language database questions ──────────────────
    user_prompt = st.chat_input("Ask a question about movies…  " \
    "e.g. List top 10 movies by IMDb rating "
    "or Find movies directed by Christopher Nolan"
    "Horror movies released after 2010 with a rating above 7.5")

    if user_prompt:
        # Save the latest user message before sending it to the AI.
        st.session_state.history.append({"role": "user", "content": user_prompt})
        with st.chat_message("user"):
            st.markdown(user_prompt)

        # Include only a few recent messages so the model has context without exceeding limits.
        history_text = "\n".join(
            f"{m['role'].upper()}: {m['content']}"
            for m in st.session_state.history[-6:]   # last 6 turns
        )

        # ── Prompt engineering for MongoDB query generation ───────────────────────
        # The AI is instructed to produce a single PyMongo expression using the db object.
        ai_prompt = f"""
You are an expert MongoDB query generator.

DATABASE: sample_mflix (MongoDB Atlas sample dataset)

SCHEMA CONTEXT (RAG):
{schema_ctx}

CONVERSATION HISTORY:
{history_text}

USER QUESTION: {user_prompt}

Instructions:
1. Identify the correct collection.
2. Write a single PyMongo expression (Python syntax using `db`).
3. Always add `.limit(20)` unless the user asks for a specific number ≤ 50.
4. Return ONLY the PyMongo expression inside a Python code block.
5. Do not explain — just output the code.

Example:
```python
db.movies.find({{}}, {{"title": 1, "imdb.rating": 1, "_id": 0}}).sort("imdb.rating", -1).limit(10)
```
"""
        with st.spinner("🤔 Generating query…"):
            ai_reply = ask_groq(model, ai_prompt)

        # Extract the actual PyMongo command from the AI text.
        query_code = extract_python_query(ai_reply)

        # ── Execute the query on MongoDB and capture the result ───────────────────
        with st.spinner("⚡ Running query on MongoDB…"):
            results, error = execute_query(client, query_code)

        # ── Render the generated query and the result in the chat window ───────────
        with st.chat_message("assistant"):
            st.markdown("**Generated MongoDB Query:**")
            st.code(query_code, language="python")

            if error:
                render_status_box("Query error", f"The generated query could not be executed. Details: {error}", kind="error")
                reply_text = f"❌ Query error: {error}"
            elif results is None:
                render_status_box("No records found", "The query executed successfully but returned no documents.", kind="warning")
                reply_text = "⚠️ No results returned."
            elif isinstance(results, list) and len(results) == 0:
                render_status_box("Empty result set", "The query ran successfully but returned 0 documents.", kind="info")
                reply_text = "Query ran but returned 0 documents."
            elif isinstance(results, list):
                safe_results = json_safe(results)
                render_status_box("Query successful", f"Returned {len(safe_results)} document(s).", kind="success")
                st.markdown(f"**Results** ({len(safe_results)} document(s)):")
                results_df = pd.DataFrame(safe_results)
                results_df.insert(0, "Sequence", range(1, len(results_df) + 1))
                st.dataframe(results_df, use_container_width=True, hide_index=True)
                reply_text = f"Returned {len(safe_results)} document(s)."
            else:
                render_status_box("Result", str(results), kind="info")
                reply_text = str(results)

            st.caption(f"Status: {reply_text}")

        # Save the assistant's final answer in chat history for follow-up context.
        st.session_state.history.append({
            "role": "assistant",
            "content": f"Query:\n```python\n{query_code}\n```\n\n{reply_text}"
        })

    # ── Clear conversation button ─────────────────────────────────────────────────
    if st.session_state.history:
        if st.button("Clear conversation"):
            st.session_state.history = []
            st.rerun()

# ─── Process Tab 2: MongoDB to MySQL Migration Workflow ──────────────────────────
# This tab allows the user to describe a migration task and the system will:
# 1) detect the target MongoDB collection and fields,
# 2) pull data from MongoDB,
# 3) flatten nested documents,
# 4) generate a MySQL/TiDB table schema,
# 5) insert data and display results.
with tab2:
    st.markdown("<div class='migration-title'>🔄 Agentic MongoDB → MySQL Data Migration</div>", unsafe_allow_html=True)
    st.caption("Describe what data to migrate — the AI agent handles extraction, schema creation, and insertion.")

    # ── Connect to MongoDB and Groq for migration mode ──────────────────────────
    try:
        client2 = get_mongo_client(mongo_uri)
    except Exception as e:
        st.error(f"❌ MongoDB connection failed: {e}")
        st.stop()

    try:
        model2 = Groq(api_key=GROQ_API_KEY)
    except Exception as e:
        st.error(f"❌ Groq initialisation failed: {e}")
        st.stop()

    # ── Try to connect to TiDB/MySQL and notify the user if migration is unavailable ─
    tidb_conn = None
    tidb_error = None
    try:
        tidb_conn = get_tidb_connection()
    except Exception as e:
        tidb_error = str(e)

    col1, col2, col3 = st.columns(3)
    with col1:
        st.markdown("<div class='status-card'><b>MongoDB</b>&nbsp;✅ Ready</div>", unsafe_allow_html=True)
    with col2:
        st.markdown("<div class='status-card'><b>Groq</b>&nbsp;✅ Ready</div>", unsafe_allow_html=True)
    with col3:
        st.markdown("<div class='status-card'><b>TiDB</b>&nbsp;" + ("✅ Connected" if tidb_conn else "⚠️ Not ready") + "</div>", unsafe_allow_html=True)

    if not tidb_conn:
        st.warning(f"⚠️ Migration is unavailable: {tidb_error}")

    st.markdown("---")

    st.subheader("📦 Migration request")
    st.markdown(
        "Examples:\n"
        "- Extract movie title, year, genres, runtime, and IMDb rating from MongoDB\n"
        "- Migrate the first 100 movie records into TiDB\n"
        "- Copy director, cast, and year information from the movies collection"
    )

    # ── User input for migration request ─────────────────────────────────────────
    migration_prompt = st.text_area(
        "Enter migration prompt:",
        placeholder="e.g. Extract movie title, year, genres, runtime and IMDb rating from MongoDB and store it in MySQL",
        height=90
    )

    if st.button("▶️ Run Migration", type="primary"):
        if not migration_prompt.strip():
            render_status_box("Missing input", "Please describe the data you want to migrate before running the process.", kind="warning")
        elif tidb_conn is None:
            render_status_box("TiDB not configured", "Configure TiDB in the .env file and restart the app before running a migration.", kind="error")
        else:
            # ── Step 1: AI selects collection and fields based on the user prompt ─
            with st.spinner("🤖 Agent analysing prompt…"):
                schema_ctx_p2 = build_schema_context(client2)
                step1_prompt = f"""
You are a data migration agent. Based on the user prompt, identify:
1. The MongoDB collection name
2. The list of fields to extract (use dot notation for nested fields e.g. imdb.rating)

SCHEMA CONTEXT:
{schema_ctx_p2}

USER PROMPT: {migration_prompt}

Respond ONLY in this JSON format with no explanation:
{{"collection": "movies", "fields": ["title", "year", "genres", "runtime", "imdb.rating"]}}
"""
                step1_reply = ask_groq(model2, step1_prompt)
                json_match = re.search(r"\{[\s\S]+\}", step1_reply)
                if not json_match:
                    render_status_box("Migration setup failed", "The AI could not determine the MongoDB collection and fields from your request.", kind="error")
                    st.stop()
                migration_info = json.loads(json_match.group())
                collection_name = migration_info["collection"]
                fields = migration_info["fields"]

            # Create a deterministic table name from the collection and selected fields.
            fields_suffix = "_".join(
                re.sub(r"[^a-zA-Z0-9]", "", f.split(".")[-1])[:6]
                for f in fields[:4]
            ).lower()
            table_name = f"{collection_name}_{fields_suffix}"[:60]

            st.info(f"📦 Collection: `{collection_name}` | Fields: `{', '.join(fields)}` | Table: `{table_name}`")

            # ── Step 2: Fetch data from MongoDB ─────────────────────────────────────
            with st.spinner("📥 Fetching data from MongoDB…"):
                projection = {"_id": 0}
                for f in fields:
                    projection[f] = 1
                db2 = client2["sample_mflix"]
                raw_docs = list(db2[collection_name].find({}, projection).limit(100))

            st.info(f"✅ Fetched {len(raw_docs)} documents from MongoDB")

            # ── Step 3: Flatten nested documents for table insertion ─────────────────
            with st.spinner("🔧 Flattening nested fields…"):
                flat_docs = [flatten_doc(doc) for doc in raw_docs]
                df = pd.DataFrame(flat_docs)
                df.columns = [re.sub(r"[^a-zA-Z0-9_]", "_", col) for col in df.columns]
                df = df.where(pd.notnull(df), None)

            # ── Step 4: Ask the AI to generate a CREATE TABLE statement ─────────────
            with st.spinner("🤖 Agent generating MySQL schema…"):
                sample_row = df.iloc[0].to_dict() if len(df) > 0 else {}
                step4_prompt = f"""
You are a MySQL schema expert. Generate a CREATE TABLE statement for TiDB (MySQL compatible).

Table name: {table_name}
Columns and sample values: {json.dumps({k: str(v)[:50] for k, v in sample_row.items()}, default=str)}

Rules:
- Use VARCHAR(500) for text fields
- Use INT for integer numbers
- Use FLOAT for decimal numbers
- Always add id INT AUTO_INCREMENT PRIMARY KEY as the first column
- Keep it simple, no foreign keys
- Return ONLY the CREATE TABLE SQL, no explanation

```sql
CREATE TABLE ...
```
"""
                step4_reply = ask_groq(model2, step4_prompt)
                create_sql = extract_sql_block(step4_reply)
                # Normalize an optional IF NOT EXISTS clause before recreating the table.
                create_sql = create_sql.replace("CREATE TABLE IF NOT EXISTS", "CREATE TABLE", 1)

            # ── Step 5: Create the target table in TiDB ──────────────────────────────
            with st.spinner("🛠️ Creating table in TiDB…"):
                try:
                    cursor = tidb_conn.cursor()
                    cursor.execute(f"DROP TABLE IF EXISTS {table_name}")
                    tidb_conn.commit()
                    cursor.execute(create_sql)
                    tidb_conn.commit()
                except Exception as e:
                    render_status_box("Table creation failed", f"TiDB could not create the target table. Details: {e}", kind="error")
                    st.stop()

            render_status_box("Table ready", f"The table '{table_name}' is ready in TiDB.", kind="success")

            # ── Step 6: Insert the flattened records into the TiDB table ─────────────
            with st.spinner("📤 Inserting data into TiDB…"):
                try:
                    cursor = tidb_conn.cursor()
                    cursor.execute(f"DESCRIBE {table_name}")
                    db_columns = [row[0] for row in cursor.fetchall() if row[0] != "id"]
                    insert_cols = [c for c in db_columns if c in df.columns]
                    insert_df = df[insert_cols]
                    placeholders = ", ".join(["%s"] * len(insert_cols))
                    col_names = ", ".join(insert_cols)
                    insert_sql = f"INSERT INTO {table_name} ({col_names}) VALUES ({placeholders})"
                    rows_inserted = 0
                    for _, row in insert_df.iterrows():
                        vals = tuple(None if pd.isna(v) else v for v in row.values)
                        try:
                            cursor.execute(insert_sql, vals)
                            rows_inserted += 1
                        except Exception:
                            continue
                    tidb_conn.commit()
                except Exception as e:
                    render_status_box("Data insertion failed", f"The records could not be inserted into TiDB. Details: {e}", kind="error")
                    st.stop()

            # ── Step 7: Retrieve and display the migrated results ───────────────────
            with st.spinner("🔍 Running SELECT query…"):
                select_sql = f"SELECT * FROM {table_name} LIMIT 20"
                cursor.execute(select_sql)
                rows = cursor.fetchall()
                col_names_out = [desc[0] for desc in cursor.description]
                result_df = pd.DataFrame(rows, columns=col_names_out)

            # ── Final output summary ─────────────────────────────────────────────────
            st.markdown("---")
            render_status_box("Migration complete", f"Successfully captured data from MongoDB and inserted it into TiDB. Rows inserted: {rows_inserted}.", kind="success")
            st.markdown(f"**Rows Inserted:** {rows_inserted}")
            st.markdown(f"**SELECT Query:** `{select_sql}`")
            st.markdown("**Sample Output:**")
            if result_df.empty:
                render_status_box("No migrated rows", "No data was returned after migration.", kind="warning")
            else:
                st.dataframe(result_df, use_container_width=True)
