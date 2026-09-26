# Project 1 — AI-Powered MongoDB Query System

A Streamlit web application that allows users to query MongoDB using natural language.

The application uses MongoDB schema information as context, sends the user’s question to Groq, generates a PyMongo query, executes the query, and displays the results in a table.

---

## Features

- Ask MongoDB questions in plain English
- Automatically inspect MongoDB collection schemas
- Use schema context for AI-powered query generation
- Generate PyMongo queries with Groq
- Execute read-only MongoDB queries
- Display query results in a Streamlit table
- Maintain conversation history for follow-up questions
- Support the MongoDB Atlas `sample_mflix` dataset
- Include MongoDB-to-TiDB/MySQL migration support

---

## How It Works

1. The application reads collection names, fields, and sample documents from MongoDB.
2. The schema information is provided to the Groq language model as RAG context.
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

Before running the application, install:

- Python 3.9 or newer
- A MongoDB Atlas account
- The MongoDB Atlas `sample_mflix` sample dataset
- A Groq API key
- TiDB or MySQL credentials for the migration feature

---

## Project Structure

```text
project-folder/
├── app.py
├── requirements.txt
├── .env
├── .gitignore
└── README.md
