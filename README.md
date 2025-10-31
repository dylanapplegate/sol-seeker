# Sol-Seeker" Autonomous Job Search Crew

Version: 1.0

Date: October 27, 2025

Owner: User

## 1. Objective (The "Why")

To design and build a local-first, multi-agent AI system that automates the most repetitive and time-consuming tasks of a job search. This system will find, filter, and prepare application materials for relevant job postings, allowing the user to focus on high-value activities (interviewing, networking, and final application submission).

**Core Problem:** The modern job search is a low-signal, high-volume task. A user must sift through hundreds of postings, filter out irrelevant roles, and manually tailor a resume and cover letter for each application.

**Proposed Solution:** A "crew" of AI agents will run on a local machine, using a local LLM and a personal vector database to perform this work autonomously, with a "human-in-the-loop" for final approval.

## 2. System Architecture (The "How")

The system is comprised of five key components that run entirely on the user's Mac, ensuring 100% privacy and control.

1. **Orchestrator (n8n):** The "trigger" for the entire system.
2. **Agent Framework (AutoGen):** The "scaffolding" that enables the agent crew to collaborate.
3. **Inference Engine (Ollama):** The "brain" that powers each agent's reasoning.
4. **Memory (ChromaDB):** The "long-term memory" holding the user's personal profile.
5. **Toolset (Python):** The "hands" that allow agents to interact with the web and file system.

### Component Interaction Flow:

1. `n8n` (Workflow) triggers on a schedule (e.g., 9 AM daily).
2. `n8n` (Execute Command Node) runs a Python script: `python3 run_job_search.py`.
3. This script initializes the `AutoGen` `GroupChatManager` and the "crew" of agents.
4. The `UserProxyAgent` (representing the user) gives the initial task: "Find new software developer jobs in Boulder, CO."
5. The `Scout_Agent` receives the task. It uses its Python `search_web` tool to find 10-15 URLs for new job postings.
6. `Scout_Agent` uses its `scrape_website` tool to extract clean text from each URL.
7. `Scout_Agent` passes the list of job descriptions to the `Analyzer_Agent`.
8. `Analyzer_Agent` reads the _User's Preferences_ (see 4.4) and performs initial filtering (e.g., discards any job not mentioning `pnpm`).
9. For the remaining jobs, `Analyzer_Agent` uses its `analyze_fit` tool, which queries the `ChromaDB` (containing the user's resume/projects) to get a "fit score" and key skills.
10. `Analyzer_Agent` passes a "Top 3" list of jobs (with score and analysis) to the `Writer_Agent`.
11. `Writer_Agent` takes the job data. For each job, it drafts a tailored cover letter and a modified resume. It saves these files to a new directory (e.g., `output/Company-Title/`).
12. The `Writer_Agent` reports "Task Complete. 3 jobs processed."
13. The `UserProxyAgent` (configured with `human_input_mode="TERMINATE"`) receives the final message. The script pauses, waiting for user input in the terminal.
14. `n8n` (Notification Node) sends a desktop notification: "Sol-Seeker: Job run complete. Awaiting review."
15. The **User** reviews the files in the `output/` directory, then types "approve" in the terminal to end the session.

## 3. Core Components (The "What")

### 3.1. Inference Engine: Ollama

- **Technology:** [Ollama](https://ollama.com/ "null")
- **Requirement:** Must be installed and running as a background service on the Mac.
- **API:** Provides a local OpenAI-compatible endpoint at `http://localhost:11434/v1`.
- **Model Requirement:** `phi3:14b-medium-128k-q4_K_M`.
    - **Reasoning:** Your 18GB of unified memory can comfortably run this 14B parameter model. Its strong reasoning and large context window are essential for parsing long, messy job descriptions and writing coherent cover letters. `llama3:8b` is a fallback.

### 3.2. Agent Framework: AutoGen

- **Technology:** [Microsoft AutoGen](https://microsoft.github.io/autogen/ "null") (`pyautogen`)
- **Configuration (`OAI_CONFIG_LIST`):**

    '''
    [
      {
        "model": "phi3:14b-medium-128k-q4_K_M",
        "api_key": "ollama",
        "base_url": "http://localhost:11434/v1"
      }
    ]
    '''

- **Agent Definitions (The Crew):**
    
    1. **`user_proxy` (UserProxyAgent):**
        
        - **Role:** Represents you, the user. Kicks off the chat and approves the final result.
        - **`human_input_mode`:** `"TERMINATE"`. This ensures the agent crew pauses for your final approval before exiting.
        - **`code_execution_config`:** `{"work_dir": "job_search_workspace"}`. Defines a safe directory for agents to write files.
            
    2. **`scout` (AssistantAgent):**
        
        - **Role:** The "Job Finder"
        - **System Prompt:** "You are an expert web researcher. Your job is to find and extract the full text from job posting URLs. You must use the tools provided."
        - **Tools:** `search_job_sites`, `scrape_website`
            
    3. **`analyzer` (AssistantAgent):**
        
        - **Role:** The "Career Analyst"
        - **System Prompt:** "You are a meticulous career analyst. You must read a job description and compare it against the user's profile, which you will access via the `analyze_fit` tool. You must strictly filter jobs based on the user's `preferences.txt` file."
        - **Tools:** `analyze_fit`, `read_file`
            
    4. **`writer` (AssistantAgent):**
        
        - **Role:** The "Application Wordsmith"
        - **System Prompt:** "You are an expert technical writer. Your job is to draft tailored cover letters and resumes. You must save these files using the `save_materials` tool."
        - **Tools:** `save_materials`

### 3.3. Workflow Orchestrator: n8n

- **Technology:** [n8n.io](https://n8n.io/ "null") (Self-hosted or Desktop app).
- **Workflow ("Daily Job Hunt"):**
    
    1. **Trigger Node:** `Schedule` (Mode: "Every Day", Hour: 9).
    2. **Action Node:** `Execute Command` (Command: `cd /path/to/project && /usr/bin/python3 run_job_search.py`).
    3. **Notification Node:** `Desktop` (Title: "Sol-Seeker Run Complete", Message: "Job crew has finished its run. Review results in the terminal.").

### 3.4. Long-Term Memory: ChromaDB

- **Technology:** [ChromaDB](https://www.trychroma.com/ "null") (Python client).
- **Requirement:** A one-time setup script (`ingest_profile.py`) will populate the database.
- **Storage:** Persisted locally to a directory (e.g., `local_chroma_db/`).
- **Collections (Data Stores):**
    
    1. **`resume`:** Stores chunks of the user's `resume.md` file for similarity search.
    2. **`projects`:** Stores chunks of individual project `README.md` files.
    3. **`preferences`:** Stores the `preferences.txt` file. This is key for personalization.

### 3.5. Agent Toolset (Custom Python Functions)

These are the Python functions you will define and register with your AutoGen agents.

- `search_job_sites(keywords: str, location: str) -> list[str]:`
    - Uses the `duckduckgo_search` library to find 10-15 job posting URLs. (More reliable than a full Google Search API).
- `scrape_website(url: str) -> str:`
    - Uses `requests` to get HTML and `beautifulsoup4` to parse and extract all human-readable text. Returns a clean string.
- `analyze_fit(job_description: str) -> dict:`
    - This is the core RAG function. It queries the `resume` and `projects` collections in ChromaDB to find the most relevant skills and experiences.
    - Returns a JSON object: `{"score": 8.5, "key_matches": ["Python", "Docker"], "gaps": ["Kubernetes"]}`.
- `save_materials(company: str, job_title: str, cover_letter: str, resume: str) -> str:`
    - Sanitizes filenames.
    - Creates a new directory: `output/[company]_[job_title]/`.
    - Saves `cover_letter.md` and `resume.md` inside.
    - Returns the path to the new directory.

## 4. Data Requirements

### 4.1. Input Data (User-Provided)

A new directory named `profile/` must be created by the user:

- `profile/resume.md`: The user's master resume in Markdown.
- `profile/projects/project_A.md`: A detailed description of Project A.
- `profile/projects/project_B.md`: A detailed description of Project B.
- `profile/preferences.txt`: A plain text file with explicit rules.
    - **Example `preferences.txt`:**

        '''
        # HARD REQUIREMENTS (MUST-HAVES)
        - Must mention "python"
        - Must mention "pnpm" or "docker"
        - Must be in "Boulder, CO" or "Remote"
        
        # HARD REJECTIONS (DEAL-BREAKERS)
        - Reject if "npm" is required and "pnpm" is not mentioned
        - Reject if title includes "Senior" or "Principal"
        - Reject if "blockchain" or "crypto" is mentioned
        '''

### 4.2. Output Data (Agent-Generated)

A directory named `output/` will be created by the agents.

- `output/ACME-Inc_Software-Engineer/cover_letter.md`
- `output/ACME-Inc_Software-Engineer/resume.md`
- `output/TechCorp_Backend-Developer/cover_letter.md`
- `output/TechCorp_Backend-Developer/resume.md`

## 5. Key Features / Epics

- **Epic 1: Autonomous Job Scouting**
    - As a user, I want the system to automatically search multiple job sites for new postings based on my keywords.
- **Epic 2: Intelligent Job Analysis & Matching**
    - As a user, I want the system to read a job description and compare it against my resume to score its relevance.
    - As a user, I want the system to **strictly** filter out jobs that don't match my explicit preferences (like requiring `pnpm`).
- **Epic 3: Automated Application Material Generation**
    - As a user, I want the system to write a first-draft cover letter and a tailored resume for each high-scoring job.
- **Epic 4: Privacy-First & Human-in-the-Loop**
    - As a user, I want all my personal data (resume, preferences) to remain on my local machine.
    - As a user, I want to give final, manual approval in my terminal before the agent's run is considered "complete."

## 6. Future Enhancements (Roadmap)

- **v1.1 (Web UI):** Create a simple `Flask` or `FastAPI` server to wrap the AutoGen script. Build a single-page HTML/JS frontend to trigger the run and display the results, replacing the terminal-based approval.
- **v1.2 (Interview Prep):** Add a new `interviewer` agent. Once you approve a job, you can trigger this agent to read the job description and your resume, then conduct a mock technical/behavioral interview with you in the console.
- **v2.0 (Automated Submission):** (Use with extreme caution) Integrate a `Selenium` tool to allow an agent to fill out "Easy Apply" forms on sites like LinkedIn.
