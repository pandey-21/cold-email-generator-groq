
# Technical Report: Document Research & Theme Identification Chatbot

**Author:** [Your Name]
**Project:** Wasserstoff AI Software Intern Qualification Task
**Date:** [Date of Submission]

**Live Application:** [https://your-final-app-url.streamlit.app/](https://your-final-app-url.streamlit.app/)

---

## 1. Project Overview & Objective

The primary objective of this project was to build an advanced AI-powered chatbot that goes beyond a basic Retrieval-Augmented Generation (RAG) system. The goal was to create a tool capable of ingesting a large, diverse set of documents and providing users with two distinct levels of insight:
1.  **Granular, Cited Answers:** The ability to extract specific information from each individual document and provide precise citations (page and paragraph).
2.  **High-Level Thematic Synthesis:** The ability to analyze all extracted information and identify overarching themes and patterns across the entire document corpus.

The final product is a functional and robust web application that successfully fulfills these requirements, demonstrating a deep, practical understanding of modern Generative AI pipelines.

## 2. System Architecture

To meet the project's unique demands, I designed a multi-stage data processing and analysis pipeline. This architecture ensures accuracy, traceability, and a clear separation of concerns.

### Stage 1: Document Ingestion and Pre-processing

This initial stage is responsible for converting raw, unstructured files into clean, indexed data.

*   **File Handling:** The system accepts PDFs, text files, and images. A dispatcher function intelligently routes each file to a specialized processor based on its extension.
*   **Robust OCR Pipeline:** Recognizing that real-world documents are often imperfect, I implemented a sophisticated OCR pipeline for scanned PDFs and images. When direct text extraction from a PDF page yields insufficient content, the system assumes it's an image-based page. It then:
    1.  Uses **PyMuPDF** to render the entire page as a high-resolution image (`get_pixmap`). This method is highly effective for any type of scanned page.
    2.  Passes the generated image to the **Tesseract** OCR engine to accurately extract the text.
*   **Semantic Chunking:** Instead of using naive, fixed-size chunks, the extracted text is segmented by paragraphs (`\n\n`). This preserves the semantic integrity of the text, providing the language model with more coherent context. Each chunk is stored as a Pydantic `DocumentSnippet` object containing its content and metadata (doc_id, page, paragraph).

### Stage 2: Indexing and Retrieval

The processed snippets are indexed for efficient semantic search.

*   **Vectorization:** Each `DocumentSnippet`'s content is converted into a numerical vector (embedding) by **ChromaDB's** built-in default embedding model. This vector captures the semantic meaning of the text.
*   **Vector Store:** **ChromaDB** is used as the vector database to store the embeddings and their associated metadata.
*   **Filtered Retrieval:** For a user query, the system does not perform a global search. Instead, it iterates through each document and performs a metadata-filtered search (`where doc_id = ...`), ensuring that the retrieved snippets for a given analysis step come from only one document.

### Stage 3: Two-Stage AI Analysis

This is the core of the "not a basic RAG" architecture.

*   **Per-Document Answer Extraction:** The top snippets from a single document are passed to a fast LLM (`llama3-8b` via **Groq**). The model is given a strict prompt to synthesize a single, consolidated answer and return it as a structured **JSON** object, including the source page and paragraph. This ensures traceability.
*   **Cross-Document Theme Synthesis:** The answers from all documents are then aggregated and passed to a more powerful LLM (`llama3-70b`). This model is prompted to act as a research analyst, clustering the answers into common themes and generating a title, summary, and list of supporting documents for each, again in a structured **JSON** format.

## 3. Challenges Faced and Solutions Implemented

Building a robust AI system involves overcoming numerous practical challenges. This project was no exception.

*   **Challenge: Duplicate Document Ingestion**
    *   **Problem:** Initially, re-uploading the same set of files would cause the application to crash with a `DuplicateIDError` from the database.
    *   **Solution:** I implemented a robust state management system in the Streamlit frontend. The application maintains a persistent `set` of processed document IDs. Before processing any uploaded file, it checks if the file's ID is already in this set. If so, it gracefully skips the file and notifies the user, making the ingestion process idempotent and error-free.

*   **Challenge: Unreliable LLM JSON Output**
    *   **Problem:** The language models, even when instructed to produce JSON, would occasionally fail. Sometimes they would generate multiple JSON objects instead of one, and other times they would omit required fields (like a `summary` for a theme), causing the application to crash during parsing.
    *   **Solution:** I implemented a two-pronged solution:
        1.  **Advanced Prompt Engineering:** I refined the prompts to be extremely explicit, including rules like "Your final output MUST be a SINGLE JSON object" and providing one-shot examples of the desired schema.
        2.  **Defensive Python Parsing:** I wrote the Python code that handles the LLM's response to be resilient. It now defensively checks if the received JSON contains all the required keys for a given data model *before* attempting to create the Pydantic object. Incomplete objects from the LLM are skipped with a warning instead of crashing the app.

*   **Challenge: Poor OCR on Scanned Documents**
    *   **Problem:** A basic OCR approach can be unreliable. Text needs to be extracted consistently from various PDF formats.
    *   **Solution:** I implemented a fallback mechanism. The system first attempts direct text extraction. If this yields minimal content (a strong indicator of a scanned page), it then uses PyMuPDF's `get_pixmap` function to render the page as an image. This image is then processed by Tesseract, providing a much more reliable text extraction result for image-based PDFs.

## 4. Conclusion

This project successfully demonstrates the design and implementation of an advanced, multi-stage RAG pipeline. By addressing practical challenges like duplicate data, unreliable model outputs, and poor-quality source documents, the resulting application is not just a proof-of-concept, but a robust and functional tool. The architecture is modular and could be effectively scaled for production use by decoupling the frontend from a dedicated FastAPI backend.
