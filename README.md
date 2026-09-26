# Project 1 — AI-Powered MongoDB Query System

A Streamlit web application that allows users to query MongoDB using natural language.

The application reads MongoDB schema information, sends the user’s question and schema context to Groq, generates a PyMongo query, executes the query, and displays the results in a table.

It also includes a MongoDB-to-TiDB/MySQL migration workflow.

---

## Features

- Ask MongoDB questions in plain English.
- Automatically inspect MongoDB collection schemas.
- Use schema information as RAG context.
- Generate PyMongo queries using Groq.
- Execute read-only MongoDB queries.
- Display query results in a Streamlit table.
- Maintain conversation history for follow-up questions.
- Support the MongoDB Atlas `sample_mflix` dataset.
- Migrate MongoDB data to TiDB/MySQL.
- Automatically generate a target SQL table.
- Flatten nested MongoDB documents before migration.

---

## How It Works

1. The application reads collection names, fields, and sample documents from MongoDB.
2. The schema information is sent to Groq as RAG context.
3. Groq generates a PyMongo query based on the user’s question.
4. The application validates and executes the generated query.
5. The results are displayed in the Streamlit interface.

---

## Technology Stack

- Python
- Streamlit
- MongoDB Atlas
- PyMongo
- Groq API
- Pandas
- TiDB/MySQL
- PyMySQL
- Python Dotenv

---

## Requirements

Before running the application, install or configure the following:

- Python 3.9 or newer
- MongoDB Atlas account
- MongoDB Atlas `sample_mflix` sample dataset
- Groq API key
- TiDB or MySQL database credentials for the migration feature

---

## Project Structure

```text
project-folder/
├── app.py
├── requirements.txt
├── .env
├── .gitignore
└── README.md
