# AI Database Automation System

A Streamlit web app that combines:
- MongoDB natural-language querying via Groq + RAG
- Process 1: asking movie questions in plain English
- Process 2: migrating MongoDB data into TiDB/MySQL
- result summaries and row-count checks after writes

## Features

### Process 1: Ask Movies
- User asks a natural-language question about the `sample_mflix` dataset
- App builds schema context from MongoDB
- Groq generates a valid PyMongo query
- Query is executed safely against MongoDB
- Results are shown in a DataFrame
- User can trigger a TiDB insert of the latest result set

### Process 2: Migrate to TiDB
- User provides a migration prompt in plain English
- AI identifies the MongoDB collection, fields, filter, and row limit
- App fetches MongoDB data
- Nested documents are flattened for table insertion
- AI generates a TiDB-compatible CREATE TABLE statement
- Data is inserted into a table in TiDB
- App displays the migrated output and a total row count

## Tech Stack
- Python
- Streamlit
- MongoDB Atlas / PyMongo
- Groq API
- PyMySQL
- Pandas

## Project Structure
- `Final_Project2.py` — main Streamlit application
- `README.md` — setup and usage guide
- `requirements.txt` — Python dependencies

## Setup

### 1. Install dependencies
```bash
pip install -r requirements.txt
```

### 2. Create a MongoDB Atlas cluster
1. Sign up at https://www.mongodb.com/cloud/atlas
2. Create a cluster and load the sample dataset
3. Ensure the `sample_mflix` database is available
4. Create a database user with read access
5. Add your IP or allow access from all IPs for local testing
6. Copy the MongoDB connection string

Example connection string:
```text
mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/
```

### 3. Get a Groq API key
1. Go to https://console.groq.com/keys
2. Create an API key
3. Copy the API key value

### 4. Configure TiDB/MySQL settings
Create a `.env` file in the project folder with:

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

## Run the app
```bash
streamlit run Final_Project2.py
```

Then open the local Streamlit URL in your browser, typically:
```text
http://localhost:8501
```

## How to use the app

### Process 1: Ask Movies
Select `Process 1: Ask Movies`.

Try prompts like:
- Show top 10 movies by IMDb rating
- Find action movies released after 2015
- List movies directed by Christopher Nolan
- Find horror movies with rating above 7.5

The app will:
- generate a MongoDB query
- execute it safely
- display the results
- allow migration of the result set into TiDB

### Process 2: Migrate to TiDB
Select `Process 2: Migrate to TiDB`.

Example prompts:
- Extract movie title, year, genres, runtime, and IMDb rating from MongoDB
- Migrate the first 100 movie records into TiDB
- Copy director, cast, and year information from the movies collection

The app then:
- identifies the target collection and fields
- fetches matching records
- flattens nested fields
- drafts the table schema
- creates the TiDB table
- inserts the rows
- displays the migrated output and the total record count

## Safety and constraints
- The MongoDB query executor only allows read-only operations such as `find`, `aggregate`, `count_documents`, and `distinct`.
- Query results are kept limited for responsiveness.
- The app uses a simple schema summary to help the AI generate valid queries.

## Screenshots

The app is ready for screenshots, but the actual image files should be saved into the `screenshots/` folder using the filenames below.

### Process 1: Ask Movies dashboard

<img width="1907" height="915" alt="image" src="https://github.com/user-attachments/assets/899ad68a-d4e6-41b7-968b-ccc04839bd82" />


### Process 2: Migration prompt screen

**<img width="1457" height="737" alt="image" src="https://github.com/user-attachments/assets/61826277-0f15-43ed-a176-3bad6a56fcaa" />

### TiDB migrated results

<img width="1572" height="496" alt="image" src="https://github.com/user-attachments/assets/f81e22eb-2d24-418e-9d3d-4a52362f4f5b" />

## Notes
- The default MongoDB database used by the app is `sample_mflix`.
- TiDB settings are required only when using the migration workflow.
- The app keeps recent chat context for the Process 1 query workflow.
