# Fullstack Engineer Challenge — System Design Review

Given the nature of demonstrating flows between system components, I chose to port the existing diagram to a sequence diagram and to point out the improvements in this new diagram.

## Problems in the original design

1. The full file (up to 1 GB) is transferred twice, from client to BFF and from BFF to API. With 500 concurrent uploads this kills the BFF.
2. There is no database, so there's no way to store video metadata or processing status. Polling doesn't work without it.
3. Processing is synchronous, the client waits for everything to finish before getting a response. Can't hit ≤15s for the first playable version.
4. Only 2 quality versions are generated (360p, 720p) but should be 3 (360p, 720p, 1080p).
5. Poster image is required but not generated.
6. Trick-play preview needs thumbnail sprites but the diagram only generates a single thumbnail.
7. Client needs an ID to poll for processing status but the diagram has no polling mechanism.
8. Diagram says "After video is completely download, user can playback." A 1 GB file on 10 Mbps takes ~13 min. Start time target is ≤2.0s.
9. CDN cache is set to 600s but deletion must propagate in ≤60s. Video stays playable from cache up to 10 min after deletion.
10. There is no delete flow in the diagram.
11. Optional .vtt caption upload/display is missing from the diagram.
12. Single BFF, single API, no load balancing. Can't hit 99.9% or 99.0% availability.

---

## Decisions and why

| Decision                                        | Why                                                                                                                                                                  |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Availability 99.0% for upload                   | Dedicated upload LB + multiple BFF/API instances; on the order of 500 concurrent uploads; path isolated from playback LB. Async queue/workers for heavy work.        |
| Client upload (progress, cancel, retry)         | axios onUploadProgress to track upload progress; cancel clears React state; retry on transient errors (e.g. timeout, 503).                                           |
| BFF uploads to S3, sends only the S3 key to API | Avoids sending the full 1 GB file through the API. Fewer network hops.                                                                                               |
| S3 object layout                                | Under `/videos/:video_id/`: `original.*`, `thumb.jpg`, `360p.mp4`, `720p.mp4`, `1080p.mp4`, `poster.jpg`, `sprites.jpg`; caption object when uploaded.               |
| Queue between API and Workers                   | Decouples upload from processing. Client gets the ID back fast and polls for status.                                                                                 |
| 20k uploads/day is fine                         | ~0.23/s average; even at high peak, upload LB + queue absorb load.                                                                                                   |
| Ingest job: scan, metadata, thumb, then 360p    | API enqueues one post-upload job. Worker runs virus scan and metadata first, then thumbnail, then 360p. Then enqueues 720p, 1080p, and poster/sprites jobs.          |
| One job per version (after ingest)              | 720p, 1080p, and poster/sprites are separate jobs after the post-upload job; they run in parallel. If one fails, only that job is retried.                           |
| First playable (360p) latency                   | Target ≤15s P95 from upload complete to first playable 360p (thumbnail and 360p both follow scan/metadata in the ingest job).                                        |
| Workers sized for CPU/RAM                       | P95 processing ≤10min for a 10-min source; parallel transcodes bounded by the slowest branch. Capacity matches both that target and the first-playable target above. |
| Availability 99.9% for playback                 | Dedicated playback LB + multiple Playback API instances; on the order of 5k concurrent viewers; path isolated from upload LB. CDN serves media files.                |
| Playback API serves the manifest, not CDN       | Deletion is immediate at the API: inactive record → no manifest. No reliance on CDN TTL for that decision.                                                           |
| Video files on CDN with 600s cache              | Immutable (versioned) object keys → safe long TTL. CDN edge reduces latency; supports the playback abandonment target.                                               |
| Progressive download, no HLS/DASH               | Adaptive streaming is not required. Progressive download + CDN supports ≤2.0s start time. Playback can start once a 360p URL from the manifest is used.              |
| Async cleanup on delete                         | Mark inactive first, then enqueue cleanup. Worker invalidates CDN and deletes objects; retry if cleanup fails.                                                       |

---

## Diagram

```mermaid
sequenceDiagram
    participant C as Client (Next.js)
    participant CP as Client playback
    participant BFF as LB + BFF (Next.js)
    participant DB as Database
    participant CDN as CDN
    participant B as Bucket
    participant Q as Queue
    participant API as LB + API (FastAPI)
    participant W as Workers
    participant PA as LB + Playback API

    rect rgba(240, 248, 255, 0.3)
        C->>BFF: upload video
        BFF->>BFF: Is file valid?<br/>Size ≤1GB, Type: MP4/MOV/WebM,<br/>Duration ≤10min
        alt No
            BFF-->>C: Send error to client
        else Yes
            BFF->>B: Upload file to S3
            BFF->>API: Send S3 reference (key/URL)
            API->>DB: Create video record (status: pending)
            API-)Q: Enqueue post-upload job<br/>(video ID + S3 ref)
            API-->>C: Video id, Status 201
        end
    end

    par Processing (async)
        Q-)W: Consume post-upload job
        W->>B: Download original from S3
        W->>DB: Update status: processing
        W->>W: Check file for virus and if file is corrupted
        W->>W: Metadata extraction (duration, dimensions)
        W->>W: Generate thumbnail
        W->>B: Store thumbnail on bucket
        W->>DB: Update status: thumbnail ready
        W->>W: Generate 360p quality version
        W->>B: Store 360p on bucket
        W->>DB: Update status: 360p ready
        W-)Q: Enqueue 720p job
        W-)Q: Enqueue 1080p job
        W-)Q: Enqueue poster and sprites job
        par Remaining jobs (parallel)
            Q-)W: Consume 720p job
            W->>W: Generate 720p quality version
            W->>B: Store 720p on bucket
            W->>DB: Update status: 720p ready
        and
            Q-)W: Consume 1080p job
            W->>W: Generate 1080p quality version
            W->>B: Store 1080p on bucket
            W->>DB: Update status: 1080p ready
        and
            Q-)W: Consume poster and sprites job
            W->>W: Generate poster and sprite strip
            W->>B: Store poster and sprites on bucket
            W->>DB: Update status: poster ready
            W->>DB: Update status: sprites ready
        end
        W->>DB: Update status: complete
    and Polling
        loop Until processing ends
            C->>API: Poll for processing status
            API->>DB: Query status
            DB-->>API: Status
            API-->>C: Status (updates UI)
        end
    end

    rect rgba(240, 255, 248, 0.3)
        C->>BFF: POST /videos/:video_id/captions (.vtt)
        BFF->>API: Send .vtt file
        API->>B: Store .vtt file on bucket
        API->>DB: Register caption URL in video record
        API-->>BFF: 200 OK
        BFF-->>C: 200 OK
    end

    rect rgba(248, 240, 255, 0.3)
        CP->>PA: GET /videos/:video_id/manifest
        PA->>DB: Query video metadata
        DB-->>PA: Video metadata
        PA-->>CP: Playback manifest JSON
        CP->>CDN: GET selected quality URL from manifest (e.g. 720p.mp4)
        CDN->>B: fetch
        B-->>CDN: file
        CDN-->>CP: video file
    end

    rect rgba(255, 240, 240, 0.3)
        C->>BFF: DELETE /videos/:video_id
        BFF->>API: forward delete request
        API->>DB: Mark video record as inactive
        API-)Q: Enqueue cleanup job
        API-->>BFF: 200 OK
        BFF-->>C: 200 OK
        Q-)W: Consume cleanup job
        W->>CDN: Invalidate cache
        W->>B: Remove files from bucket
        W->>DB: Delete video record
    end
```
