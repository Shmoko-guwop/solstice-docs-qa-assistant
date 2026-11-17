# Solution Design
## Solstice Learning — Documentation Question & Answer Assistant

**Document type:** Solution Design Proposal  
**Audience:** Solstice Learning — Product, Engineering, IT and Operations  
**Status:** Proposed MVP Architecture  
**Version:** 2.0

---

## 1. Executive Summary

Solstice Learning's support and operations teams regularly need to locate answers across product and operational documentation. Finding the correct information manually can be time-consuming, particularly when questions concern specific procedures, plan limits, billing rules or authentication configurations.

This proposal describes a **Documentation Question & Answer Assistant** that enables users to ask questions in natural language and receive concise, evidence-grounded answers drawn from Solstice Learning's approved documentation.

The solution uses a **Retrieval-Augmented Generation (RAG)** architecture. Relevant passages are retrieved from a vector database and supplied to a language model as supporting context. Responses include source citations, and the system abstains when the available documentation does not adequately support an answer.

The application will be containerized with **Docker** and deployed to **Microsoft Azure** through a **GitHub-based CI/CD workflow**. This provides a repeatable path from source-code changes to tested, versioned container images and deployment.

The initial release will focus on reliable document ingestion, retrieval, answer generation, citations and abstention, with a lightweight web interface.

## 2. Business Problem

Support and operations personnel may need to answer recurring questions such as:

- How can a user reset their password?
- How long does a password-reset link remain valid?
- What is the maximum row limit for CSV exports?
- What are the API rate limits for a particular subscription plan?
- Which single sign-on (SSO) options are supported?

When answers are distributed across multiple documents, staff must identify the right source, search for the relevant passage and interpret it before responding. This creates avoidable effort and can lead to inconsistent answers.

A conventional keyword search can help locate documents, but it does not reliably interpret natural-language questions or synthesize relevant passages. A generative model without document retrieval, on the other hand, may produce plausible but unsupported answers.

The proposed solution addresses both limitations by combining semantic retrieval with controlled, source-grounded generation.

## 3. Objectives and Success Criteria

### 3.1 Objectives

1. Allow users to ask documentation-related questions in natural language.
2. Retrieve relevant passages from the supplied knowledge base and uploaded PDF documents.
3. Generate answers using retrieved documentation as the source of truth.
4. Display citations that identify the source document and page where available.
5. Abstain when the documentation does not provide sufficient evidence.
6. Allow users to query a selected document or the complete document collection.
7. Provide a reproducible, containerized application that can be deployed through GitHub to Azure.

### 3.2 Initial acceptance criteria

The MVP will be evaluated against the five questions supplied with the case study:

| Evaluation question | Expected behaviour |
|---|---|
| How do I reset my password? | Answer using document D-01 and cite the source. |
| How long does a password reset link remain valid? | State 15 minutes using D-01 and cite the source. |
| What is the maximum CSV export row limit? | State 10,000 rows using D-03 and cite the source. |
| What are the Growth plan API rate limits? | State 60 requests per minute using D-06 and cite the source. |
| Can I connect Solstice to Salesforce? | Abstain because the supplied documentation does not establish Salesforce support. |

The system should also be tested with paraphrased questions, document filters, unsupported questions and invalid PDF uploads.

## 4. Proposed Solution

The proposed application consists of a React-based web interface and a Python FastAPI backend.

The backend implements the RAG workflow:

1. Receive the user's question and selected document scope.
2. Generate an embedding for the question using the Gemini embedding service.
3. Search ChromaDB for semantically relevant document chunks, applying the selected document filter.
4. Evaluate whether the retrieved passages provide sufficient and relevant evidence.
5. If evidence is sufficient, send the question and retrieved passages to Gemini for a grounded response.
6. Return the answer together with source citations.
7. If evidence is insufficient, return a clear abstention rather than generating an unsupported answer.

This approach separates retrieval from answer generation and makes it possible to inspect which evidence was used to produce each response.

## 5. Solution Architecture

### 5.1 Logical architecture

```text
                         USER
                           |
                           v
                React + Vite Web Application
                           |
                       HTTPS / API
                           |
                           v
                  FastAPI Backend Service
                           |
             +-------------+-------------+
             |                           |
             v                           v
      Document Ingestion           Question Processing
             |                           |
        PyPDF extraction            Gemini Embeddings
             |                           |
       Text chunking                     v
             |                     ChromaDB Search
             v                           |
      Gemini Embeddings                   v
             |                  Evidence / Relevance Check
             v                           |
          ChromaDB <---------------------+
             |                           |
             |                           v
             |                    Gemini Answer Generation
             |                           |
             +---------------------------+
                                         |
                                  Answer + Citations
                                         |
                                         v
                                React Web Application
```

### 5.2 Application components

| Component | Responsibility | Proposed technology |
|---|---|---|
| Web frontend | Chat interface, PDF upload, document selection and citation display | React, Vite, JavaScript, CSS |
| API backend | REST endpoints, ingestion orchestration and RAG workflow | Python, FastAPI |
| PDF extraction | Extract text and page information from PDF files | PyPDF |
| Chunking | Split extracted text into retrieval-sized passages while retaining metadata | Custom Python implementation |
| Embedding service | Convert document chunks and user questions into vector representations | Gemini Embedding API |
| Vector database | Store embeddings, text chunks and document metadata; perform semantic retrieval | ChromaDB |
| Answer generation | Produce answers grounded in retrieved evidence | Gemini API |
| Optional chat persistence | Store limited conversation history if required by the MVP | SQLite |
| Containerization | Package frontend and backend consistently for local and cloud execution | Docker, Docker Compose |
| Source control and delivery | Version source code, run CI checks, build images and initiate deployment | GitHub, GitHub Actions |
| Cloud hosting | Run the containerized application | Azure Container Apps |
| Container image registry | Store versioned frontend and backend images | Azure Container Registry |
| Persistent storage | Retain uploaded PDFs and ChromaDB data beyond container restarts | Azure Files mounted into the backend container |
| Monitoring | Collect application and container logs | Azure Monitor / Log Analytics |

## 6. Document Ingestion and Knowledge Base

### 6.1 Supported sources

The initial knowledge base will include the six synthetic documentation articles supplied with the case study. The application will also support user-uploaded PDF documents.

### 6.2 Ingestion workflow

1. Accept a PDF through the frontend.
2. Validate the file type and basic upload constraints.
3. Extract text and page information using PyPDF.
4. Divide the extracted text into overlapping chunks using a custom chunking implementation.
5. Generate an embedding for each chunk through the Gemini embedding API.
6. Store the chunk, embedding and associated metadata in ChromaDB.

Metadata should include, where available:

- Document identifier and document name
- Source file name
- Page number
- Chunk identifier
- Upload or ingestion timestamp

Retaining source metadata is essential for document filtering and citation generation.

### 6.3 Document selection

Users will be able to select:

- **All Documents** — search across the complete knowledge base.
- **A specific document** — restrict retrieval to the selected document.

The backend will enforce this selection as a metadata filter during vector retrieval, rather than relying on the frontend alone.

### 6.4 Initial limitations

The MVP will process text-based PDFs. Optical Character Recognition (OCR) for scanned documents is outside the initial scope. Documents from which usable text cannot be extracted should produce a clear validation or ingestion error.

## 7. Retrieval-Augmented Generation Workflow

The RAG pipeline is the central capability of the solution.

### 7.1 Query processing

The backend receives the question and document scope, then generates a query embedding using the Gemini embedding service.

### 7.2 Semantic retrieval

ChromaDB performs similarity search against stored document embeddings. Retrieval is restricted by the selected document identifier when a specific document is selected.

The backend retrieves the most relevant chunks together with their source metadata.

### 7.3 Evidence validation

Similarity alone does not prove that a passage supports a factual answer. The backend will therefore apply relevance and evidence-sufficiency checks before answer generation.

If the retrieved passages are irrelevant, contradictory or insufficient to support the requested answer, the system should abstain. Retrieval thresholds and evidence rules will be tuned against the supplied evaluation questions and additional test cases.

### 7.4 Grounded answer generation

When sufficient evidence is available, the backend sends the question and retrieved passages to Gemini with instructions to:

- Answer only from the supplied context.
- Avoid introducing facts that are not supported by the retrieved material.
- State when the documentation does not contain the requested information.
- Associate claims with the relevant source references.

### 7.5 Citations and abstention

Each answer should include citations identifying the source document and page number where available.

For unsupported questions, the application should return a clear response such as:

> I don't have an answer for that in the available documentation.

The Salesforce evaluation question is an explicit abstention test. The system must not infer product capabilities that are absent from the knowledge base.

## 8. API and Frontend Design

### 8.1 Frontend capabilities

The React application will provide:

- A chat interface for natural-language questions.
- A PDF upload control.
- A document selector with **All Documents** and individual document options.
- Answer rendering with source citations.
- Loading, validation and error states.
- A clear presentation for abstained responses.

The interface will remain intentionally lightweight so that development effort is concentrated on retrieval quality and answer reliability.

### 8.2 Backend responsibilities

FastAPI will expose endpoints for the core application workflows, including:

- Document upload and ingestion.
- Listing available documents for selection.
- Submitting a question with its document scope.
- Returning an answer, citations and abstention status.

The frontend will communicate with the backend through HTTP APIs. API request and response structures should be documented and kept consistent as the application evolves.

## 9. Containerization with Docker

Docker will be used to provide consistent development, testing and deployment environments.

### 9.1 Container design

The application will use separate frontend and backend containers:

- **Frontend container:** builds the React/Vite application and serves the production build through a lightweight web server such as Nginx.
- **Backend container:** runs the FastAPI application using Uvicorn and contains the ingestion and RAG components.

A Docker Compose configuration will orchestrate both services during local development.

### 9.2 Local development workflow

Developers will be able to start the application using Docker Compose, with configuration supplied through environment variables and a local environment file.

Persistent local volumes will be used for ChromaDB data and uploaded documents so that data survives ordinary container restarts.

### 9.3 Container design principles

- Keep frontend and backend responsibilities separate.
- Use reproducible dependency installation and pinned versions where appropriate.
- Do not place API keys or other secrets inside container images.
- Configure services through environment variables.
- Use persistent volumes for data that must survive container replacement.
- Provide health checks and clear startup behaviour.
- Use production server configuration for deployment rather than development servers.

## 10. Azure Hosting and GitHub-Based CI/CD

### 10.1 Deployment approach

The solution will be deployed to Microsoft Azure using Azure Container Apps. Azure Container Registry will store the versioned Docker images.

GitHub will act as the source-code repository and the entry point for the deployment workflow. GitHub Actions will automate validation, image building, publishing and deployment.

### 10.2 Target deployment architecture

```text
                 Developer
                     |
                Git Push / PR
                     |
                     v
              GitHub Repository
                     |
                     v
              GitHub Actions
              - Run checks
              - Build images
              - Tag images
                     |
                     v
       Azure Container Registry (ACR)
              /              \
             v                v
      Frontend Image     Backend Image
             |                |
             v                v
      Azure Container Apps Environment
             |                |
      Frontend App       Backend App
                               |
                    +----------+----------+
                    |                     |
                    v                     v
              Azure Files             Gemini APIs
             - ChromaDB data       - Embeddings
             - Uploaded PDFs       - Generation
```

### 10.3 CI/CD workflow

The proposed GitHub Actions workflow is:

1. A developer pushes a change or opens a pull request.
2. GitHub Actions checks the code and runs the available automated tests.
3. On an approved deployment branch, the workflow builds the frontend and backend Docker images.
4. The workflow tags the images with a traceable version, such as the Git commit SHA.
5. The workflow authenticates to Azure using a securely configured deployment identity.
6. The images are pushed to Azure Container Registry.
7. The workflow updates the corresponding Azure Container Apps with the new image versions.
8. The deployment is verified through health checks and a basic application smoke test.

Pull requests should run validation without automatically deploying production changes. Deployment should be restricted to the agreed deployment branch and protected through repository and Azure permissions.

### 10.4 Azure services

| Azure service | Purpose |
|---|---|
| Azure Container Apps | Host the frontend and backend containers |
| Azure Container Registry | Store and version Docker images |
| Azure Files | Persist uploaded PDFs and ChromaDB data |
| Azure Monitor / Log Analytics | Support application and container monitoring |
| Microsoft Entra ID / Azure deployment identity | Secure GitHub Actions authentication to Azure |

### 10.5 Persistent storage considerations

Container filesystems are not suitable for application data that must survive container replacement. The backend will therefore use an Azure Files mount for its persistent data directory, including uploaded PDFs and the ChromaDB persistence directory.

For the initial deployment, the backend should run with a single replica while using a local persistent ChromaDB instance. This avoids introducing concurrent-write and multi-replica coordination concerns into the MVP. A future scale-out design may require a separately hosted vector database or another storage architecture designed for concurrent access.

### 10.6 Configuration and secrets

Sensitive values, including Gemini API credentials and Azure deployment credentials, must be stored as GitHub Actions secrets or Azure-managed secrets as appropriate.

Secrets must not be committed to GitHub, included in Docker images, or exposed to the browser. The frontend should call the backend; it should not call Gemini directly with a privileged API key.

## 11. Security, Reliability and Operational Considerations

The MVP will follow these baseline practices:

- **Secret management:** keep API keys and deployment credentials outside source control and container images.
- **Input validation:** validate uploaded file types and enforce reasonable upload-size limits.
- **Error handling:** return controlled API errors for invalid files, failed extraction, embedding failures and model-service outages.
- **Grounded responses:** use retrieved evidence and explicit abstention behaviour to reduce unsupported answers.
- **Data persistence:** store uploaded files and vector data on persistent storage.
- **Logging:** record operational errors and request outcomes without logging secrets.
- **Deployment traceability:** associate deployed image versions with Git commits.
- **Access control:** restrict access to the repository, deployment workflow and Azure resources according to team responsibilities.

Authentication and role-based access control for end users are not defined by the supplied case study and should be treated as a separate requirement if the application is intended for broader or production use.

## 12. Technology Selection and Rationale

| Technology | Rationale |
|---|---|
| React + Vite | Lightweight component-based frontend with a fast development workflow. |
| FastAPI | Python API framework with asynchronous support and automatic API documentation. |
| PyPDF | Direct PDF text extraction without introducing a larger document-processing framework. |
| Custom chunking | Provides control over chunk size, overlap and source metadata. |
| Gemini Embeddings | Generates vector representations for semantic document retrieval and user queries. |
| ChromaDB | Provides a straightforward vector store suitable for the MVP and local persistence. |
| Gemini | Generates natural-language answers from retrieved documentation. |
| Docker | Ensures consistent application packaging and execution across developer machines and Azure. |
| Docker Compose | Simplifies local orchestration of the frontend and backend services. |
| GitHub Actions | Automates code validation, container builds and Azure deployment from the source repository. |
| Azure Container Apps | Hosts containerized services without requiring the team to manage Kubernetes infrastructure. |
| Azure Container Registry | Provides a central registry for versioned application images. |
| Azure Files | Provides persistent storage for files and ChromaDB data mounted into the backend container. |

The design intentionally avoids introducing an additional orchestration framework such as LangChain or LlamaIndex in the initial release. The RAG workflow can be implemented directly with the selected APIs and libraries, keeping the system understandable and easier to debug.

## 13. Scope

### 13.1 Included in the MVP

- React/Vite web interface.
- FastAPI backend.
- Ingestion of the supplied case-study documentation.
- PDF upload and text extraction using PyPDF.
- Custom chunking with source metadata.
- Gemini-based embeddings.
- ChromaDB semantic retrieval.
- Document-specific and all-document retrieval.
- Gemini answer generation grounded in retrieved evidence.
- Source citations and explicit abstention.
- Dockerfiles and Docker Compose for local execution.
- GitHub source control and GitHub Actions CI/CD.
- Azure Container Apps deployment using Azure Container Registry.
- Persistent Azure Files storage for uploaded documents and vector data.
- Evaluation against the five supplied case-study questions.

### 13.2 Excluded from the initial release

- OCR for scanned PDFs.
- Advanced document administration, deletion and lifecycle management.
- Duplicate-document detection.
- A document analytics or management dashboard.
- Complex user roles and enterprise identity integration.
- Multi-replica ChromaDB write coordination.
- Advanced conversational memory beyond what is required for the core Q&A workflow.

These capabilities can be considered in a later phase based on stakeholder requirements.

## 14. Implementation Plan

| Phase | Key activities | Deliverable |
|---|---|---|
| 1. Project foundation | Inspect and preserve the supplied starter kit, establish the GitHub repository, add the solution design and configure environment files. | Version-controlled project foundation |
| 2. Application and Docker foundation | Implement the FastAPI and React application structure; create Dockerfiles and Docker Compose configuration. | Both services running locally in containers |
| 3. Document ingestion | Load the supplied documentation, implement PDF extraction, chunking and metadata handling. | Ingestion pipeline |
| 4. Embeddings and vector storage | Integrate Gemini embeddings and persist vectors in ChromaDB. | Searchable vector knowledge base |
| 5. RAG and answer reliability | Implement semantic retrieval, document filtering, evidence checks, grounded generation, citations and abstention. | Working RAG API |
| 6. Frontend integration | Implement upload, document selection, chat, citations and error states. | End-to-end local application |
| 7. Evaluation | Run the supplied five questions and additional tests for paraphrasing, unsupported queries, filtering and persistence. | Evaluation results and fixes |
| 8. Azure deployment | Configure Azure resources, GitHub Actions, ACR, Container Apps, secrets and Azure Files persistence. | Deployed cloud application |
| 9. Handover | Finalize README, architecture notes, environment configuration guidance and demonstration flow. | Documented, demonstrable solution |

## 15. Risks and Mitigations

| Risk | Potential impact | Mitigation |
|---|---|---|
| Retrieved passages are similar but do not support the answer | Incorrect or misleading responses | Apply evidence checks and abstain when support is insufficient. |
| Poor PDF text extraction | Missing or incomplete knowledge-base content | Validate extraction results and report files with no usable text. |
| Chunk boundaries separate related information | Retrieval may miss relevant context | Tune chunk size and overlap using the evaluation questions. |
| Gemini API limits or service interruption | Ingestion or answering may fail | Handle API errors, use bounded retries where appropriate and return clear user-facing errors. |
| Container replacement loses local data | Uploaded documents or vectors disappear | Persist backend data through Azure Files and local Docker volumes. |
| Multiple backend replicas access local ChromaDB | Data consistency or write conflicts | Keep one backend replica for the MVP; revisit storage architecture before scaling. |
| Deployment credentials are exposed | Unauthorized access to cloud resources | Use GitHub secrets and a least-privilege Azure deployment identity. |
| CI/CD deploys an unverified change | Application instability | Run automated checks before image publication and deployment; use traceable image tags. |

## 16. Future Enhancements

After the MVP has been evaluated, potential enhancements include:

- OCR support for scanned documents.
- Authentication and role-based access control.
- Document lifecycle and administration features.
- Improved evaluation using a larger set of representative questions.
- Feedback collection for answer quality.
- A vector database architecture designed for higher availability and multi-replica access.
- More advanced observability, alerting and deployment rollback procedures.
- Additional document formats, subject to business requirements.

## 17. Conclusion

The proposed Solstice Learning Documentation Q&A Assistant combines semantic retrieval with controlled generative answering to make approved documentation easier to use while reducing the risk of unsupported responses.

The architecture is deliberately modular: the React frontend, FastAPI backend, document-ingestion pipeline, embedding service, vector database and answer-generation service have clearly defined responsibilities. Docker provides consistent execution across local development and cloud deployment. GitHub Actions provides a repeatable delivery path to Azure Container Apps, with Azure Container Registry for image management and Azure Files for persistent application data.

The MVP prioritizes the core business outcomes: accurate documentation-based answers, traceable citations and reliable abstention when the knowledge base does not contain the answer. This provides a focused foundation that can be extended as Solstice Learning's operational requirements grow.

---

**Proposed deployment platform:** Microsoft Azure  
**Source control and CI/CD:** GitHub and GitHub Actions  
**Container platform:** Docker and Azure Container Apps  
**Core application pattern:** Retrieval-Augmented Generation (RAG)
