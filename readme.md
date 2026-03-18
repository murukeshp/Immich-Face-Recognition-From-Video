# 🕵️ Immich Video Face Recognition Pipeline
**Codebase Analysis Report**

---

## 1. Abstract
This document provides a comprehensive analysis of the Video Face Recognition implementation within the Immich codebase (specifically focusing on `person.service` and `machine-learning.repository.ts`). It outlines the complete architectural pipeline, key thresholds, and detailed steps involved in identifying and clustering faces across video frames to ensure accurate person detection.

---

## 2. General Architecture & Entry Points

The facial recognition module is built on an event-driven architecture using job queues:

1.  **Job Queuing:** The process begins with `AssetDetectFacesQueueAll` (Queue: `FaceDetection`), queuing individual `AssetDetectFaces` jobs.
2.  **Asset Dispatching:** Inside `handleDetectFaces`, the asset type is determined. For images, a single preview frame is used. For videos, the workload is handed off to `handleVideoFaceDetection()`.
3.  **Machine Learning Repository:** `MachineLearningRepository.predict()` acts as an abstraction layer for making HTTP POST requests with `FormData` to external ML servers running detection/recognition models.

---

## 3. The Video Face Detection Pipeline (`handleVideoFaceDetection`)

Video face recognition follows a sophisticated multi-stage pipeline designed to accurately map faces temporally across a moving video.

### Step 1: Multi-Frame Extraction
Instead of analyzing every single frame, the system extracts a subset of frames at a controlled FPS rate (calculated based on `videoDuration`). These temporary frames are stored in an OS-level temp directory. 
*   **Purpose:** Ensures individuals appearing at different timestamps are caught while saving compute resources.

### Step 2: Individual Frame Detection
The system iterates through every extracted frame and calls the remote ML service (`detectFaces`).
*   **Normalization Check:** The engine automatically detects whether the bounding box coordinates returned by the ML model are normalized `[0, 1]` or absolute pixel values (by checking if `x2` or `y2` > `1.05`).
*   **Quality & Size Filters:** 
    *   Minimum bounding box width/height: **20 pixels**
    *   Minimum face confidence score: **0.75 (75%)**
    If a detection falls below these limits, it is discarded immediately.

### Step 3: Temporal Clustering (`clusterFaces`)
Because a person's face will appear across multiple frames, Immich uses temporal clustering to avoid registering the same person 50 times.
*   **Algorithm:** It builds a graph where nodes are detections and edges are drawn if the **Cosine Similarity** between two face embeddings exceeds the threshold of **0.85**. 
*   It then groups connected components together to form "clusters" (representing a single person's appearances throughout the video).

### Step 4: Cluster Qualification & Voting
Once clusters are formed, the system evaluates them to ensure they represent a valid person rather than a temporary anomaly or false positive.
*   **Adaptive Minimum Faces:**
    *   If video duration $\leq$ 30s: Minimum **2** occurrences required.
    *   If video duration $>$ 30s: Defaults to **3** (or config settings).
*   **Best Face Selection (Composite Score):** From a valid cluster, the system picks the single "best" frame to serve as a thumbnail. It uses a composite weighting formula:
    *   `60%`: Confidence Score from AI (`score * 0.6`)
    *   `30%`: Relative Area Size (`(area / maxAreaInCluster) * 0.3`)
    *   `10%`: Centrality penalty (Faces closer to the center of the frame get a boost).

### Step 5: Database Persistence (`processFaces`)
The single best representation of each cluster is added to the database. It carefully bounds the coordinates so they remain strictly within the exact pixel dimensions of the specific individual frame where they were found. 

---

## 4. Key Thresholds & Configurations

| Parameter Name                  | Value / Formula                                                | Description                                                                 |
|:--------------------------------|:---------------------------------------------------------------|:----------------------------------------------------------------------------|
| **Frame Extraction FPS**        | `getVideoFps(duration)`                                        | Dynamically determined based on video length.                               |
| **Min Bounding Box Size**       | `20px` x `20px`                                                | Minimum area required to acknowledge a face in video.                       |
| **Min ML Confidence Score**     | `0.75` (Initial) / `0.80` (Final)                              | Initial loose filter vs Final strict check for Video assets.                |
| **Cosine Similarity Threshold** | `0.85`                                                         | To consider two face embeddings as the "same person" across frames.         |
| **Adaptive Minimum Clusters**   | `2` (Short Video) / `3` (Long Video)                           | The number of times a face must appear in the video to count as legitimate. |

---

## 5. Built-in Debugging Support
The codebase includes dedicated debug routines for video pipeline visibility:
*   `copyFramesToDebugFolder`: Dumps all extracted RAW frames into `/tmp/immich-debug-frames/<asset_id>`.
*   `drawBoundingBoxesToDebugFolder`: Uses `sharp` and SVG overlays to re-draw green bounding boxes and confidence scores over the raw frames and saves them to `/tmp/immich-debug-faces/<asset_id>/`. This is immensely useful for visual verification of the ML server's accuracy.

---
*Report generated based on `c:/Users/Nived/Desktop/IMMICH` source files.*
