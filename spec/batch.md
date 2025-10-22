## **Project Specification: Dharma Stream**

### **Part 4: Batch Processing Specification (Docker)**

**Version:** 1.3
**Date:** October 21, 2025

---

### **1. Philosophy & Architectural Role**

The batch processing worker is the engine of the Dharma Stream platform. It is a **stateless, distributed, and location-agnostic** application whose sole purpose is to execute computationally expensive, long-running tasks. It operates asynchronously from the user-facing application, communicating and receiving tasks exclusively through the central PostgreSQL database.

*   **Decoupled:** The worker knows nothing about the frontend or user sessions. Its world consists of the job queue in the database and the media files on the WebDAV server.
*   **Scalable:** The system's processing power can be scaled horizontally by simply running more instances of the worker container. Each new instance will automatically start competing for jobs from the queue.
*   **Resilient:** The worker is designed to be fault-tolerant. If a worker crashes mid-process, the job will remain in the `processing` state and can be manually reset or automatically timed out for another worker to pick up.
*   **Portable:** Encapsulation within a Docker container ensures that the worker runs identically on a developer's laptop, a Kubernetes pod, or a temporary cloud VPS.

### **2. Core Technologies & Binaries**

| Technology | Source / Tool | Role |
| :--- | :--- | :--- |
| **Orchestration** | Python 3.10+ | The primary language for the main worker script, handling database communication, file I/O, and process management. |
| **Containerization** | Docker | Packages the application, all its dependencies, and the compiled binaries into a single, portable image. |
| **STT Engine** | **Whisper.cpp** | A high-performance C++ port of OpenAI's Whisper model. It is compiled with CPU optimizations (OpenBLAS) to provide fast transcription without requiring expensive GPUs. |
| **Media Toolkit** | **FFmpeg** (Static Build) | The industry-standard tool for all media manipulation. It will be used for format conversion (e.g., WEBM to MP4), audio extraction (MP4 to M4A), and HLS stream generation. |
| **DB Driver** | `psycopg2-binary` | A robust and performant Python driver for connecting to the PostgreSQL database. |
| **Supabase Client** | `supabase-py` | A helper library for simpler interaction with Supabase, though direct `psycopg2` usage is preferred for complex, transactional queries. |

### **3. Worker Logic & Job Lifecycle**

The `worker.py` script will execute an infinite loop that represents the lifecycle of a job.

**Step 1: Claim a Job (Atomic Operation)**
The worker will execute a single, atomic SQL query to find and claim a job. This prevents race conditions where multiple workers might grab the same job.

```python
# worker.py (pseudocode)
import psycopg2

# ... (connect to database)
cursor = conn.cursor()

claim_job_sql = """
    UPDATE dharma.stt_jobs
    SET
        status = 'processing',
        processing_started_at = now()
    WHERE id = (
        SELECT id
        FROM dharma.stt_jobs
        WHERE status = 'pending'
        ORDER BY created_at
        FOR UPDATE SKIP LOCKED
        LIMIT 1
    )
    RETURNING id, media_id;
"""

cursor.execute(claim_job_sql)
job_data = cursor.fetchone()
conn.commit()

if not job_data:
    # No pending jobs found, sleep and try again
    time.sleep(30)
    continue # Go to the next loop iteration
```

**Step 2: Prepare Environment & Paths**
Once a job is claimed, the worker fetches the necessary metadata and constructs all the paths it will need.

```python
# ...
job_id, media_id = job_data
media_metadata = fetch_media_metadata(media_id) # Fetches title, category_id, created_at

# Create a temporary local directory for processing
local_temp_dir = f"/tmp/{job_id}"
os.makedirs(local_temp_dir, exist_ok=True)

# Construct WebDAV paths based on convention
year_month = media_metadata['created_at'].strftime('%y%m')
base_dav_path = f"/{media_metadata['category_id']}/{year_month}/{media_id}"
video_dav_path = f"{base_dav_path}/video.mp4"
audio_dav_path = f"{base_dav_path}/audio.m4a"
hls_dav_path = f"{base_dav_path}/hls/"
```

**Step 3: Download & Pre-process Media**
The worker downloads the source file (which was temporarily uploaded by the admin) and uses FFmpeg to standardize it.

```python
# Download the source file from its temporary location on WebDAV
source_file_path = download_source_file(media_id, local_temp_dir)

# Define local paths for processed files
local_video_path = f"{local_temp_dir}/video.mp4"
local_audio_path = f"{local_temp_dir}/audio.m4a"
local_hls_path = f"{local_temp_dir}/hls"

# Use FFmpeg to convert to standard formats
# This command overwrites, ensures a specific codec, and extracts audio
run_subprocess(f"ffmpeg -i {source_file_path} -y -c:v libx264 -c:a aac {local_video_path}")
run_subprocess(f"ffmpeg -i {local_video_path} -y -vn -c:a copy {local_audio_path}")
```

**Step 4: Generate HLS Stream**
Using FFmpeg again, the worker creates an HLS stream for adaptive bitrate streaming on the frontend.

```python
# HLS parameters from environment variables
hls_bitrate = os.getenv("HLS_BITRATE", "2000k")
hls_shard_seconds = os.getenv("HLS_SHARD_SECONDS", "6")

os.makedirs(local_hls_path, exist_ok=True)

# This command creates the HLS playlist and video segments
run_subprocess(f"ffmpeg -i {local_video_path} -y -b:v {hls_bitrate} -g 48 -hls_time {hls_shard_seconds} -hls_playlist_type vod -hls_segment_filename '{local_hls_path}/segment%03d.ts' {local_hls_path}/playlist.m3u8")
```

**Step 5: Run Speech-to-Text (Whisper.cpp)**
The worker calls the compiled `whisper` binary to perform the transcription.

```python
# Run whisper.cpp against the extracted audio file
# -l zh = language Chinese
# -m = path to the model file (e.g., ggml-base.bin)
# -oj = output in JSON format
run_subprocess(f"whisper -m /models/ggml-base.bin -f {local_audio_path} -l zh -oj")

# The output will be a file named local_audio_path.json
json_output_path = f"{local_audio_path}.json"
with open(json_output_path, 'r') as f:
    transcription_result = json.load(f)
```

**Step 6: Process Results & Update Database**
The worker parses the JSON output from Whisper, formats it, and batch-inserts it into the `subtitles` table.

```python
# Parse the JSON and prepare for batch insert
subtitles_to_insert = []
for segment in transcription_result['transcription']:
    subtitles_to_insert.append({
        'media_id': media_id,
        'start_time_ms': int(segment['timestamps']['from']),
        'end_time_ms': int(segment['timestamps']['to']),
        'text': segment['text'].strip()
    })

# Perform a batch insert for efficiency
batch_insert_subtitles(subtitles_to_insert)```

**Step 7: Upload Artifacts & Finalize Job**
The worker uploads all the generated files to their final, permanent location on the WebDAV server and updates the job status.

```python
# Upload processed files to WebDAV
upload_to_dav(local_video_path, video_dav_path)
upload_to_dav(local_audio_path, audio_dav_path)
upload_directory_to_dav(local_hls_path, hls_dav_path)

# Update the media_files table with the duration
video_duration = get_video_duration(local_video_path)
update_media_duration(media_id, video_duration)

# Finally, mark the job as completed
update_job_status(job_id, 'completed')
```

**Error Handling:** The entire process (Steps 2-7) must be wrapped in a `try...except` block. If any exception occurs, the `except` block will log the error and call `update_job_status(job_id, 'failed', error_message)`.

**Cleanup:** A `finally` block will ensure that the temporary local directory (`/tmp/{job_id}`) is always deleted, regardless of success or failure.

### **4. Dockerfile Configuration**

The `Dockerfile` will be a multi-stage build to keep the final image lean.

```dockerfile
# Stage 1: Builder for Whisper.cpp
# This stage compiles whisper.cpp with OpenBLAS for CPU optimization.
FROM debian:bullseye-slim AS builder
RUN apt-get update && apt-get install -y build-essential git cmake libopenblas-dev
WORKDIR /build
RUN git clone https://github.com/ggerganov/whisper.cpp.git
WORKDIR /build/whisper.cpp
# Compile with OpenBLAS support
RUN WHISPER_OPENBLAS=1 make

# Stage 2: Final Production Image
FROM python:3.10-slim
WORKDIR /app

# Install system dependencies: wget for ffmpeg, xz-utils to decompress it,
# and libopenblas-base to satisfy the runtime dependency for whisper.cpp.
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    wget \
    xz-utils \
    libopenblas-base \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Download and install the static FFmpeg build
RUN wget https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz && \
    tar -xf ffmpeg-release-amd64-static.tar.xz && \
    mv ffmpeg-*-static/ffmpeg /usr/local/bin/ && \
    rm -rf ffmpeg-*

# Copy the compiled whisper.cpp binary and a pre-downloaded model from the builder stage
COPY --from=builder /build/whisper.cpp/main /usr/local/bin/whisper
# Models should be downloaded during the build process for inclusion in the image
RUN mkdir /models && \
    wget -O /models/ggml-base.bin https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the worker script
COPY worker.py .

# Set the entrypoint
CMD ["python", "worker.py"]
```

### **5. Environment Configuration (`.env.example`)**

This file defines the contract for all environment variables the container expects.

```ini
# Supabase Configuration
SUPABASE_URL=https://your-project-ref.supabase.co
SUPABASE_SERVICE_KEY=your-secret-service-role-key

# PostgreSQL Direct Connection (for psycopg2)
# This is often more robust for long-running server processes.
DB_HOST=db.your-project-ref.supabase.co
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres
DB_PASSWORD=your-database-password

# WebDAV Server Configuration
WEBDAV_BASE_URL=https://dav.your-server.com
WEBDAV_USER=your-webdav-user
WEBDAV_PASSWORD=your-webdav-password

# HLS Processing Configuration
HLS_BITRATE=2000k
HLS_SHARD_SECONDS=6
```
