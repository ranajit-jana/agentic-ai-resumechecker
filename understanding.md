# Easy Understanding of `resume.py`

## What this file does

`resume.py` is a small Streamlit web app for **resume screening**.

It lets a user:

1. Enter job requirements
2. Upload a resume file
3. Send both to Google's Gemini model
4. Get an AI-written analysis
5. Extract a suitability score like `85%`
6. Save the analysis into a Chroma vector database

In short:

This file compares a resume with a job description and tells how well they match.

---

## Main flow in very simple words

The app works like this:

1. Load environment variables from `.env`
2. Check if `GOOGLE_API_KEY` exists
3. Show a Streamlit page with:
   - a text box for job requirements
   - a file uploader for the resume
4. When the user clicks **Analyze**:
   - read text from the uploaded resume
   - send resume text + job requirements to Gemini
   - show the AI analysis
   - pull out the `Suitability Score`
   - store the analysis in Chroma
   - let the user download the result

---

## Simple flow chart

```text
Start
  |
  v
Load .env and read GOOGLE_API_KEY
  |
  v
Show Streamlit app
  |
  +-------------------------------+
  |                               |
  v                               v
Enter job requirements      Upload resume file
  |                               |
  +---------------+---------------+
                  |
                  v
          Click Analyze
                  |
                  v
     Extract text from resume file
                  |
                  v
 Build prompt using:
 - job requirements
 - resume text
                  |
                  v
      Send prompt to Gemini
                  |
                  v
       Receive analysis text
                  |
                  +-----------------------------+
                  |                             |
                  v                             v
Extract Suitability Score             Show analysis on screen
                  |
                  v
     Split analysis into chunks
                  |
                  v
   Store analysis chunks in Chroma
                  |
                  v
 Show success message + download button
                  |
                  v
                 End
```

---

## Imports: why they are used

- `os`: file paths, environment variables, folders
- `streamlit as st`: builds the web app UI
- `PromptTemplate`: creates the AI prompt
- `StrOutputParser`: converts model output to plain text
- `ChatGoogleGenerativeAI`: Gemini chat model
- `PyPDFLoader`, `Docx2txtLoader`, `TextLoader`: read resume files
- `Chroma`: vector database for storing analysis
- `RecursiveCharacterTextSplitter`: splits long text into chunks
- `GoogleGenerativeAIEmbeddings`: turns text into embeddings
- `RunnableMap`: passes inputs into the LangChain pipeline
- `google.generativeai as genai`: configures Google API
- `load_dotenv`: loads `.env` values
- `re`: used to find the score using regex
- `traceback`: shows technical error details

---

## Important variables

- `GOOGLE_API_KEY`: API key from `.env`
- `LLM_MODEL = "gemini-2.5-flash-lite"`: model used for analysis
- `EMBEDDING_MODEL = "models/gemini-embedding-001"`: model used for embeddings
- `VECTOR_STORE_DIR = "chroma_store"`: folder where Chroma stores data

If the API key is missing, the app stops immediately.

---

## Function-by-function explanation

### `get_embedding_model()`

This creates the embedding model object.

Why needed:

Chroma stores text as vectors, and embeddings convert text into vector form.

---

### `get_vectorstore()`

This creates or opens the Chroma vector database.

What it does:

- makes sure the `chroma_store` folder exists
- connects Chroma with the embedding model

---

### `extract_text_from_resume(file)`

This function reads the uploaded resume and returns plain text.

How it works:

1. Save the uploaded file temporarily
2. Check the file extension
3. Use the correct loader:
   - `.pdf` -> `PyPDFLoader`
   - `.docx` -> `Docx2txtLoader`
   - `.txt` -> `TextLoader`
4. Load all text
5. Join all pages/sections into one string
6. Delete the temporary file

Why this is helpful:

The AI model needs text, not a raw PDF or DOCX file.

---

### `split_text(text)`

This breaks long text into smaller chunks.

Settings used:

- chunk size: `500`
- overlap: `50`

Why chunking matters:

Vector databases work better when large text is stored in smaller parts.

---

### `store_resume_analysis(resume_text, analysis, doc_id)`

This saves the AI analysis into Chroma.

What it does:

1. Split the analysis into chunks
2. Open the vector store
3. Add the chunks with IDs like:
   - `resume_name_chunk_0`
   - `resume_name_chunk_1`

Important note:

The parameter `resume_text` is passed in, but it is not actually used inside this function.

---

### `extract_suitability_score(text)`

This finds the score from the AI response.

It looks for text in this exact format:

`Suitability Score: XX%`

Example:

If Gemini writes `Suitability Score: 82%`, this function returns `82`.

If that format is missing, it returns `None`.

---

### `run_analysis(chain, job_requirements, resume_text)`

This runs the AI pipeline safely.

What it does:

- sends job requirements and resume text into the chain
- returns the final analysis text

If something fails:

- shows an error in Streamlit
- shows the raw error message
- gives extra warnings for:
  - `429` or quota problems
  - `404` or missing model problems
- shows full traceback for debugging
- stops the app

This is basically the app's error-handling wrapper for AI analysis.

---

### `main()`

This is the heart of the app.

It builds the full Streamlit UI and connects everything together.

What happens inside:

1. Set page title and wide layout
2. Show app title
3. Create two columns
4. Left side:
   - text area for job requirements
5. Right side:
   - resume uploader
6. When **Analyze** is clicked:
   - extract resume text
   - optionally display the raw resume text
   - create the Gemini model
   - build a prompt template
   - create a LangChain pipeline
   - run the analysis
   - show the result
   - extract and display the score
   - store the analysis in Chroma
   - allow download of the analysis

---

## The LangChain chain in simple words

This part:

```python
chain = (
    RunnableMap(...)
    | prompt_template
    | llm
    | StrOutputParser()
)
```

means:

1. Prepare the input values
2. Put them into the prompt
3. Send the prompt to Gemini
4. Convert the response into plain text

This is the AI pipeline of the app.

---

## Prompt meaning

The prompt tells Gemini:

- act like an HR/recruitment expert
- compare resume vs job requirements
- give a structured analysis
- end with `Suitability Score: XX%`

That last instruction is important because the code later uses regex to pull the score out.

---

## What the user sees on screen

The user experience is:

- type the job requirements
- upload a resume
- click **Analyze**
- wait for processing
- see:
  - extracted resume text
  - AI analysis
  - suitability score
  - success message for vector DB storage
  - download button

---

## Good things in this code

- supports PDF, DOCX, and TXT resumes
- checks for missing API key early
- handles Gemini errors nicely
- stores analysis for later semantic search use
- separates logic into small functions

---

## Things to notice or improve later

### 1. `resume_text` is unused in `store_resume_analysis`

The function accepts `resume_text`, but never uses it.

That parameter could be removed unless needed later.

### 2. Vector DB stores only analysis, not the original resume

Right now only the AI analysis is stored in Chroma.

If you want future search on resume content too, the resume text could also be stored.

So the actual order is:

1. Resume file is uploaded
2. Resume text is extracted
3. Resume text is sent to Gemini
4. Gemini creates the analysis
5. Only the analysis is stored in Chroma

### 3. Score extraction depends on exact wording

The regex only works if the model writes:

`Suitability Score: XX%`

If Gemini changes the format, score extraction may fail.

### 4. Temporary file naming could collide

It uses:

`temp_{file.name}`

If two uploads have the same name at the same time, there could be a conflict.

---

## One-line summary of the whole file

`resume.py` is a Streamlit app that reads a resume, compares it to job requirements using Gemini, shows a match score, and stores the analysis in Chroma.
