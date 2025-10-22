## **Project Specification: Dharma Stream**

### **Part 1: Project Overview & System Architecture**

**Version:** 1.3
**Date:** October 21, 2025

---

### **1. Project Vision & Goals**

#### **1.1. Vision**

To create a modern, robust, and serene Single-Page Application (SPA) dedicated to hosting and sharing Buddhist audio and video teachings. The platform will provide a high-quality user experience for content consumption, while empowering an administrative team with powerful tools for content management, transcription, and community-driven content improvement.

#### **1.2. Core Objectives**

*   **Accessibility:** Deliver teachings seamlessly across all devices (desktop, tablet, mobile) with a clean, responsive, and intuitive user interface.
*   **Performance:** Ensure fast load times and smooth media playback, even with very large files, by leveraging a distributed architecture and modern streaming protocols (HLS).
*   **Intelligence:** Automate the creation of subtitles for all Chinese-language media using a high-performance, CPU-optimized Whisper STT engine.
*   **Collaboration:** Enable a community of trusted users to suggest corrections to AI-generated subtitles, and provide administrators with an efficient workflow to review and approve these corrections.
*   **Scalability & Flexibility:** Design a system where the computationally intensive tasks (video conversion, STT) can be scaled independently, allowing for "burst" capacity using external resources without altering the core infrastructure.
*   **Personalization:** Provide users with a personalized experience, including access to specific content categories, viewing history, and the ability to resume playback from where they left off.

### **2. System Architecture**

The "Dharma Stream" platform is designed as a decoupled, service-oriented system. This separation of concerns ensures that each component is specialized for its task, leading to greater stability, maintainability, and scalability.

#### **2.1. Architectural Diagram**

```
                                     +--------------------------------+
                                     |      External WebDAV Server    |
                                     | (Stores all large media files: |
                                     |  .mp4, .m4a, HLS segments)     |
                                     +---------------+----------------+
                                                     ^
                                                     | 5. Uploads Processed Media
                                                     |
+-----------------+   1. API Calls   +---------------+----------------+   2. DB Queries/Auth   +-----------------+
|                 | (HTTPS/WSS)      |                                | (Secure Connection)    |                 |
| React Frontend  |----------------->|       Supabase Platform        |----------------------->|  PostgreSQL DB  |
| (Hosted on      |                  |                                |                        | (Metadata,      |
| Vercel/Netlify) |<-----------------| (Backend, Auth, Storage, APIs) |<-----------------------|  Users, Jobs)   |
|                 |   4. Streams     |                                |                        |                 |
+-------+---------+   Media From     +--------------------------------+                        +--------+--------+
        |             WebDAV                                                                           ^
        | 3. Fetches                                                                                   | 6. Updates Job
        |    Media URL                                                                                 |    Status &
        |    (Constructed)                                                                             |    Subtitles
        |                                                                                              |
        +----------------------------------------------------------------------------------------------+
                                                     | 7. Polls for Pending Jobs (SELECT...FOR UPDATE)
                                                     |
      +----------------------------------------------+----------------------------------------------+
      |                                                                                             |
      v                                                                                             v
+-----+--------------------+                                                            +-----------+------------+
|                          |                                                            |                        |
|  Batch Processing Pool   |                                                            | Batch Processing Pool  |
|  (Docker in Kubernetes)  |                                                            | (Docker on External VPS)|
|                          |                                                            |                        |
+--------------------------+                                                            +------------------------+
      |                                                                                             |
      +---------------------------------------------------------------------------------------------+
                                     | 8. Downloads media from WebDAV for processing
                                     v
```

#### **2.2. Core Components & Responsibilities**

1.  **Frontend (React SPA):**
    *   **Role:** The user's sole entry point to the platform.
    *   **Responsibilities:** Renders all user interfaces, manages client-side state, handles user authentication flows, plays media, and communicates with the Supabase backend via its auto-generated API. It is a "thin client" that contains no sensitive business logic.

2.  **Supabase (Backend-as-a-Service):**
    *   **Role:** The central nervous system and single source of truth for all data and business logic.
    *   **Responsibilities:**
        *   **Database (PostgreSQL):** Stores all structured data: user accounts, media metadata, categories, job queues, subtitles, user permissions, and view history.
        *   **Authentication:** Manages user sign-up, login, session management, and JWT issuance.
        *   **APIs:** Provides instant, secure RESTful and real-time APIs over the database, governed by strict Row Level Security (RLS) policies.
        *   **Storage:** Used for small, public assets like category thumbnails or speaker avatars (NOT for large media files).

3.  **Batch Processing Workers (Docker):**
    *   **Role:** The distributed, heavy-lifting workforce of the system.
    *   **Responsibilities:** Executes long-running, computationally expensive tasks completely independent of the user-facing application. This includes video/audio transcoding, HLS stream generation, and running Whisper.cpp for Speech-to-Text transcription.

4.  **WebDAV Server (External Storage):**
    *   **Role:** A simple, robust, and cost-effective "dumb" storage server for large files.
    *   **Responsibilities:** Stores and serves all large media assets. The application uses a strict, convention-based folder structure on this server to locate files, meaning the database does not need to store individual file URLs.

#### **2.3. Technology Stack**

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Frontend** | React.js, TypeScript, TanStack Query, Zustand | Modern, performant, and type-safe UI development with excellent state management capabilities. |
| **Backend** | Supabase (PostgreSQL, PostgREST, GoTrue) | Provides a complete backend solution, drastically reducing development time while ensuring security and scalability. |
| **Batch** | Python 3.10+, Docker, Whisper.cpp, FFmpeg | Python is ideal for scripting and orchestration. Docker ensures portability. Whisper.cpp and FFmpeg are best-in-class, high-performance tools for their respective tasks. |
| **Storage** | Any standard WebDAV-enabled server | A simple, open-standard protocol for file storage, separating large file costs from the primary application platform. |
| **Hosting** | Vercel/Netlify (Frontend), Supabase Cloud, Kubernetes/VPS (Batch) | Leverages managed services for ease of deployment, CI/CD, and scalability. |

### **3. Project & Code Organization**

To ensure a clean separation of concerns and facilitate parallel development, the project repository will be organized into three primary folders at the root level.

```
dharma-stream/
├── backend/
│   ├── api/                # Contains FastAPI code for admin APIs (correction approval, etc.)
│   └── supabase/
│       ├── migrations/     # SQL migration files for database schema changes
│       └── functions/      # Supabase Edge Functions (if used in the future)
│
├── frontend/
│   ├── public/
│   └── src/
│       ├── components/     # Reusable React components (Player, SubtitleList, etc.)
│       ├── pages/          # Top-level page components (HomePage, AdminDashboard, etc.)
│       ├── hooks/          # Custom React hooks
│       ├── services/       # Supabase client configuration and data fetching logic
│       └── state/          # Global state management (Zustand stores)
│
└── batch/
    ├── Dockerfile          # Defines the build process for the worker image
    ├── worker.py           # The main Python script for the job processing loop
    ├── requirements.txt    # Python dependencies
    └── .env.example        # Template for environment variables
```

