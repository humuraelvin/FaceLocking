# FACE_LOCKING_GUIDE.md

# Face Locking: Advanced Behavior Tracking

## Overview

**Face Locking** extends the face recognition system with **intelligent tracking and behavior detection**. Once a specific enrolled identity is recognized, the system:

1. **Locks onto that face** and maintains focus
2. **Tracks movements** (head position changes)
3. **Detects actions** (blinks, smiles, expressions)
4. **Records a timeline** of all observed behaviors

This is ideal for:
- Security systems that need to track intruders
- Attention-monitoring systems
- Interview/interrogation recordings
- Human-computer interaction (HCI) research
- Behavioral analysis

---

## Architecture

### Core Components

#### 1. **FaceLockSystem** (`face_lock.py`)
Main orchestrator that manages the face locking state machine.

**State Machine:**
```
SEARCHING → (target found + high confidence) → LOCKED
    ↑                                            ↓
    ← (face lost > timeout)                  (track actions)
                                                 ↓
                                    (face disappears < timeout)
                                                 ↓
                                               LOST
                                                 ↓
                                    (face reappears) → LOCKED
```

#### 2. **ActionDetector** (`action_detector.py`)
Detects face actions from landmarks:
- **Eye Blinks**: rapid eye closure/opening
- **Head Movement**: left/right motion tracking via nose position
- **Smiles/Laughs**: mouth height increase
- **Face Scale Changes**: closer/farther from camera

#### 3. **FaceHistoryLogger** (`face_history_logger.py`)
Persistent logging of all detected actions to timestamped text files.

**File Format:**
```
<face>_history_<timestamp>.txt

Example:
gabi_history_1706549940123.txt
```

**Log Entry Format:**
```
[HH:MM:SS.mmm] ACTION_TYPE | description | confidence | value
[00:00:05.234] BLINK       | Eye blink detected (open -> closed -> open) | 0.85 | 0.45
[00:00:06.567] MOVE_RIGHT  | Head movement right (12.5px) | 0.92 | 12.5
[00:00:07.890] SMILE       | Smile/laugh detected (mouth height ratio: 1.15) | 0.88 | 1.15
```

---

## Installation & Setup

### Prerequisites
- Working face recognition system (completed enrollment with at least 1 enrolled face)
- Python 3.9+
- All dependencies from `requirements.txt`
- ArcFace ONNX model in `models/embedder_arcface.onnx`

### File Structure
```
data/
├── enroll/          # Enrollment crops (must be populated)
│   ├── Person1/
│   ├── Person2/
│   └── ...
├── db/
│   ├── face_db.npz  # Recognition database
│   └── face_db.json # Metadata
└── face_histories/  # Auto-created, stores action logs
    ├── person1_history_1706549940123.txt
    ├── person2_history_1706549950456.txt
    └── ...
```

---

## Quick Start

### Step 1: Enroll Faces (if not already done)
```bash
python -m src.enroll
```

Follow prompts to capture 15+ samples per person, then save.

### Step 2: Evaluate Threshold (optional but recommended)
```bash
python -m src.evaluate
```

This helps determine optimal distance threshold for your setup.

### Step 3: Run Face Locking
```bash
python -m src.face_lock
```

**Interactive Prompts:**
1. System shows available enrolled faces
2. You select the target identity (e.g., "Gabi")
3. System enters search mode
4. When target appears with high confidence → lock acquired
5. System tracks face and records actions in real-time

**Controls:**
- `r`: Release lock manually
- `q`: Quit and save history

### Step 4: Review Action History
After session ends, check `data/face_histories/`:
```bash
cat data/face_histories/gabi_history_1706549940123.txt
```

---

## Detailed Usage

### Configuration Parameters

Edit `face_lock.py` or pass parameters to `FaceLockSystem`:

```python
from pathlib import Path
from src.face_lock import FaceLockSystem

system = FaceLockSystem(
    db_path=Path("data/db/face_db.npz"),
    model_path="models/embedder_arcface.onnx",
    distance_threshold=0.34,        # Cosine distance for recognition
    lock_timeout_frames=30,          # Frames to hold lock if face lost
    min_lock_confidence=0.85,        # Min confidence to acquire lock
)
```

### Action Detection Sensitivity

Adjust `action_detector.py` parameters:

```python
detector = ActionDetector(
    blink_threshold=0.15,            # Eye opening ratio (0-1)
    smile_threshold=0.08,            # Mouth height increase ratio
    movement_threshold_px=8.0,       # Pixels for left/right detection
    scale_change_threshold=0.12,     # Face scale change ratio
)
```

**Tuning Guide:**
- **Lower** values = more sensitive, more false positives
- **Higher** values = less sensitive, may miss subtle actions

---

## Detected Actions Reference

### 1. **Blink**
- **Detection**: Vertical eye closure/opening using eye openness metric
- **Threshold**: `blink_threshold` (default 0.15)
- **Confidence**: 0.85 (fixed for stable blinks)
- **False Positives**: Squinting, sunglasses

### 2. **Move Left / Move Right**
- **Detection**: Horizontal nose position change
- **Threshold**: `movement_threshold_px` pixels (default 8.0)
- **Confidence**: Scales with movement distance (0-0.95)
- **False Positives**: None (pure position tracking)

### 3. **Smile**
- **Detection**: Increase in mouth height ratio
- **Threshold**: `smile_threshold` ratio increase (default 0.08 = 8%)
- **Confidence**: Scales with mouth height increase
- **False Positives**: Yawning, wide speech

### 4. **Face Closer / Face Farther**
- **Detection**: Change in eye distance (proxy for camera distance)
- **Threshold**: `scale_change_threshold` ratio (default 0.12 = 12%)
- **Confidence**: Scales with distance change
- **False Positives**: Head tilt (less reliable than others)

---

## Output Format: Action History Files

### File Naming
```
<face>_history_<timestamp>.txt

Examples:
- gabi_history_1706549940123.txt
- fani_history_1706549950456.txt
- alice_history_1706549960789.txt
```

### File Structure

**Header:**
```
================================================================================
Face Locking Session History
================================================================================
Face Name: GABI
Session Start: 2024-01-29 12:30:45.123
File: gabi_history_1706549940123.txt
================================================================================
Format: [HH:MM:SS.mmm] ACTION_TYPE | description | confidence | value
================================================================================
```

**Action Entries:**
```
[00:00:05.234] BLINK           | Eye blink detected (open -> closed -> open) | conf=0.85 | val=0.4500
[00:00:06.567] MOVE_RIGHT      | Head movement right (12.5px) | conf=0.92 | val=12.5000
[00:00:07.890] SMILE           | Smile/laugh detected (mouth height ratio: 1.15) | conf=0.88 | val=1.1500
[00:00:08.123] FACE_CLOSER     | Face moved closer to camera (scale ratio: 1.08) | conf=0.80 | val=1.0800
[00:00:09.456] STATUS          | Lock ACQUIRED for GABI (confidence=0.92)
[00:00:10.789] STATUS          | Lock LOST (face disappeared)
[00:00:11.012] STATUS          | Lock RE-ACQUIRED for GABI
```

**Footer:**
```
================================================================================
Session ended at 2024-01-29 12:32:10.456
Total actions recorded: 47
================================================================================
```

---

## Advanced Usage

### Programmatic Access

```python
from pathlib import Path
from src.face_lock import FaceLockSystem

# Initialize system
system = FaceLockSystem(
    db_path=Path("data/db/face_db.npz"),
    distance_threshold=0.34,
)

# Select target
if not system.select_target("Gabi"):
    print("Face not found in database")
    exit(1)

# Process frames
cap = cv2.VideoCapture(0)
while True:
    ret, frame = cap.read()
    if not ret:
        break

    # Process single frame
    result = system.process_frame(frame)

    # Access state
    print(f"State: {result['state']}")
    print(f"Locked: {result['locked_identity']}")
    print(f"Actions: {result['actions']}")

    # Draw custom visualization
    # ... your code ...

    if should_quit:
        break

# Save history
history_file = system.finalize_session()
print(f"Saved to: {history_file}")

cap.release()
cv2.destroyAllWindows()
```

### Custom Action Processing

```python
from src.action_detector import ActionDetector

detector = ActionDetector()

# In your main loop:
actions = detector.detect(landmarks_5x2)

for action in actions:
    print(f"Time: {action.timestamp}")
    print(f"Type: {action.action_type}")
    print(f"Confidence: {action.confidence}")
    print(f"Value: {action.value}")
    print(f"Description: {action.description}")
```

---

## Troubleshooting

### Issue: "Lock never acquired"

**Cause**: Recognition confidence too low or face not enrolled properly

**Solutions:**
1. Check enrollment: `python -m src.evaluate`
2. Lower `min_lock_confidence` threshold (default 0.85)
3. Ensure lighting is similar to enrollment
4. Re-enroll with more varied poses

### Issue: "Lock lost immediately"

**Cause**: Face disappears or confidence drops

**Solutions:**
1. Increase `lock_timeout_frames` (default 30)
2. Check camera stability
3. Ensure better lighting
4. Move closer to camera

### Issue: "Actions not detecting"

**Cause**: Action thresholds too high or landmarks poor quality

**Solutions:**
1. Verify landmarks are visible: `python -m src.landmarks`
2. Lower action thresholds in `ActionDetector`
3. Check camera resolution and lighting
4. Ensure face is properly aligned (run `python -m src.align`)

### Issue: "Empty or missing history file"

**Cause**: Session ended before logging completed

**Solutions:**
1. Ensure normal exit (press 'q' to quit)
2. Check `data/face_histories/` directory exists
3. Verify write permissions: `ls -la data/face_histories/`

---

## Performance Optimization

### CPU-Only Considerations

**Frame Processing Timing:**
- Detection: ~50-100ms (Haar cascade)
- Alignment: ~10-20ms (5-point landmarks)
- Embedding: ~100-150ms (ArcFace ONNX)
- Action Detection: ~5-10ms (simple math)

**Total**: ~200-300ms per frame ≈ 3-5 FPS

**Optimization Tips:**

1. **Skip frames for detection** (detect every N frames):
```python
if frame_count % 5 == 0:
    faces = detector.detect(frame)
```

2. **Reduce input resolution**:
```python
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
```

3. **Use ROI-based detection** (search within previous face area)

---

## Example: Integration with Existing Code

### Minimal Integration
```python
# In your existing recognize.py, add:
from src.face_lock import FaceLockSystem

system = FaceLockSystem()
system.select_target("Gabi")

# In main loop:
result = system.process_frame(frame)
if result["actions"]:
    print(f"Actions detected: {[a.action_type for a in result['actions']]}")
```

### Advanced Integration
```python
# Replace recognize.py pipeline with:
from src.face_lock import FaceLockSystem
from src.action_detector import ActionDetector

system = FaceLockSystem()
system.select_target("Gabi")

# Add custom visualization
while True:
    ret, frame = cap.read()
    result = system.process_frame(frame)

    # Custom drawing
    if result["locked_identity"]:
        x1, y1, x2, y2 = result["face_box"]
        cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 3)

        # Show actions as overlay
        for action in result["actions"]:
            cv2.putText(
                frame, action.action_type, (x1, y1 - 10),
                cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 255), 2
            )

    cv2.imshow("Locked", frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

system.finalize_session()
```

---

## Data Analysis

### Parse History Files

```python
from pathlib import Path

history_file = Path("data/face_histories/gabi_history_1706549940123.txt")
content = history_file.read_text()

# Extract actions (skip headers/status)
actions = {}
for line in content.split('\n'):
    if '|' in line and not '=====' in line and '[' in line:
        parts = line.split('|')
        if len(parts) >= 2:
            action_type = parts[0].split(']')[1].strip()
            actions[action_type] = actions.get(action_type, 0) + 1

print(actions)
# Output: {'BLINK': 47, 'MOVE_RIGHT': 12, 'MOVE_LEFT': 8, 'SMILE': 3}
```

### Statistics

```python
# Count actions
total_actions = sum(actions.values())
print(f"Total actions: {total_actions}")

# Most common
most_common = max(actions, key=actions.get)
print(f"Most common: {most_common} ({actions[most_common]} times)")

# Session duration
# (extract from header and footer timestamps)
```

---

## Common Workflows

### Workflow 1: Security Monitoring
```
1. Enroll suspect/intruder faces
2. Run face_lock for each person
3. Generate incident reports with action timelines
4. Use action history as evidence
```

### Workflow 2: User Attention Tracking
```
1. Enroll user
2. Run face_lock during task
3. Analyze blinks and movement patterns
4. Correlate with task performance
```

### Workflow 3: Interview Recording
```
1. Enroll interview subject
2. Start face_lock
3. Record actions during interview
4. Analyze behavioral cues (smiles, nervousness)
5. Cross-reference with audio/video
```

---

## Technical Notes

### Landmark Ordering (Critical)
The action detector expects landmarks in this order:
```
0: left_eye
1: right_eye
2: nose_tip
3: left_mouth
4: right_mouth
```

This order is enforced throughout the pipeline (see `haar_5pt.py`).

### State Machine Guarantees
- **SEARCHING** → **LOCKED**: When target found with `confidence >= min_lock_confidence`
- **LOCKED** → **LOST**: When face not detected in current frame
- **LOST** → **SEARCHING**: When face absent for `lock_timeout_frames`
- **LOST** → **LOCKED**: When target reappears with high confidence

### Thread Safety
Current implementation is single-threaded. For multi-threaded use:
```python
import threading

lock = threading.Lock()

def process_frame(frame):
    with lock:
        result = system.process_frame(frame)
    return result
```

---

## Limitations & Future Work

### Current Limitations
- Single-face locking (no multi-face support)
- Landmarks from MediaPipe (not as robust as 3D models)
- Simple action detection (no complex gestures)
- CPU-only (limited FPS)

### Potential Improvements
- Multi-face locking with identity switching
- 3D pose estimation for better head movement
- Gesture recognition (thumbs up, peace sign, etc.)
- GPU acceleration for higher FPS
- Neural network-based action detection
- Integration with emotion recognition
- Real-time dashboard with metrics

---

## References

- **Face Recognition**: Deng et al. (2019) - ArcFace
- **Landmarks**: MediaPipe Face Mesh (Lugaresi et al., 2019)
- **Alignment**: Hartley & Zisserman (2003) - Multiple View Geometry
- **Detection**: Viola & Jones (2001) - Haar Cascades

---

## Support & Questions

For issues or questions:
1. Check **Troubleshooting** section
2. Review action detector confidence scores
3. Verify enrollment quality
4. Test individual components (`python -m src.align`, etc.)
5. Check logs in `data/face_histories/`

---

**Last Updated:** January 2025  
**Author:** Gabriel Baziramwabo (Rwanda Coding Academy)
