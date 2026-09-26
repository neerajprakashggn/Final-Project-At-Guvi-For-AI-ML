# Project 1 — AI-Powered MongoDB Query & Migration System

## 1. Project Overview

This project is a Streamlit-based AI application that provides two capabilities:

- **Natural Language MongoDB Querying:** Users ask questions in plain English. Groq generates a PyMongo query using MongoDB schema context, and the application executes the read-only query.
- **MongoDB → TiDB/MySQL Migration:** Users describe the data they want to migrate. The AI identifies the collection and fields, extracts the data, flattens nested fields, creates a TiDB/MySQL table, and inserts the records.

**Technologies:** Python, Streamlit, MongoDB Atlas, Groq, PyMongo, Pandas, PyMySQL, TiDB/MySQL, python-dotenv.

## 2. Project Structure

```text
Final_Project2/
│
├── Final_Project2.py      # Main Streamlit application
├── README.md              # Project documentation
├── .env                   # Credentials and configuration
└── requirements.txt       # Python dependencies
```

## 3. Application Workflow

### Process 1 — AI MongoDB Query

```text
User Question
      ↓
Streamlit UI
      ↓
MongoDB Schema Context
      ↓
Groq AI
      ↓
PyMongo Query
      ↓
Read-only Query Execution
      ↓
Results in DataFrame
```

The application uses the MongoDB `sample_mflix` dataset. The schema and sample documents are provided to the AI as context so it can select the appropriate collection and fields.

### Process 2 — MongoDB to TiDB/MySQL

```text
Migration Request
      ↓
Groq identifies collection & fields
      ↓
Fetch MongoDB data
      ↓
Flatten nested fields
      ↓
Generate CREATE TABLE SQL
      ↓
Create TiDB/MySQL table
      ↓
Insert data
      ↓
Display migrated records
```

## 4. Key Features

- Natural-language database querying
- RAG-based MongoDB schema context
- AI-generated PyMongo queries
- Conversation history for follow-up questions
- Read-only query operations
- MongoDB-to-TiDB/MySQL migration
- Automatic table-schema generation
- Nested document flattening
- Streamlit interactive UI
- Result display using Pandas DataFrames

## 5. Setup

### Step 1 — Create the environment

```bash
python -m venv venv
```

Windows:

```bash
venv\Scripts\activate
```

### Step 2 — Install dependencies

```bash
pip install -r requirements.txt
```

If `requirements.txt` is not available, install the main packages:

```bash
pip install streamlit pymongo pandas pymysql groq bson python-dotenv
```

### Step 3 — Configure MongoDB Atlas

1. Create a MongoDB Atlas account.
2. Create a cluster.
3. Load the MongoDB **Sample Dataset**.
4. Ensure the `sample_mflix` database is available.
5. Create a database user.
6. Allow your IP address under Network Access.
7. Copy the MongoDB connection string.

### Step 4 — Create `.env`

Create `.env` in the project directory:

```env
MONGO_URI=your_mongodb_connection_string
GROQ_API_KEY=your_groq_api_key
GROQ_MODEL=qwen/qwen3.8-27b

TIDB_HOST=your_tidb_host
TIDB_PORT=4000
TIDB_USER=your_tidb_user
TIDB_PASSWORD=your_tidb_password
TIDB_DATABASE=your_tidb_database
```

`MONGO_URI` and `GROQ_API_KEY` are required for Process 1. TiDB settings are required for Process 2.

## 6. Execution

From the project directory, run:

```bash
streamlit run Final_Project2.py
```

Open:

```text
http://localhost:8501
```

## 7. Example Queries

Try these in **Process 1**:

```text
Show top 10 movies with highest IMDb rating
```

```text
List all movies in the Horror genre
```

```text
Who directed the movie Inception?
```

```text
Show movies with more than 1000 votes
```

For follow-up questions, for example:

```text
Show top-rated movies
```

then:

```text
Show only the ones released after 2000
```

The application retains the recent conversation history for context.

## 8. Example Migration Request

In **Process 2**, enter:

```text
Extract movie title, year, genres, runtime and IMDb rating from MongoDB and store it in MySQL
```

The application will identify the required MongoDB fields, fetch the data, create the target table, insert the records, and display sample migrated data.

## 9. Important Technical Notes

- MongoDB source database: `sample_mflix`
- AI provider: Groq
- Default AI model configured in the application: `qwen/qwen3.8-27b`
- Streamlit provides the web interface.
- MongoDB queries are intended to be read-only.
- Query results are capped to keep the application responsive.
- TiDB is MySQL-compatible and is used as the migration target.
