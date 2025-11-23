# AB3DMOT - Complete Explanation & Setup Guide

## What is AB3DMOT?

**AB3DMOT** (A Baseline for 3D Multi-Object Tracking) is a real-time 3D multi-object tracking system designed for autonomous driving applications. It tracks objects (cars, pedestrians, cyclists, etc.) in 3D space using LiDAR point cloud data.

### Key Features:
- **Real-time performance**: Runs at ~214 FPS (65x faster than state-of-the-art 2D MOT systems)
- **High accuracy**: State-of-the-art results on KITTI dataset
- **Simple architecture**: Combines standard methods (Kalman Filter + Hungarian Algorithm)
- **Supports multiple datasets**: KITTI and nuScenes

## How It Works

### Architecture Overview

The system follows a **tracking-by-detection** approach:

```
3D Detections → Kalman Filter Prediction → Data Association → Track Update/Birth/Death
```

### Step-by-Step Process:

1. **Input**: 3D bounding boxes from an object detector (e.g., PointRCNN)
   - Format: `[h, w, l, x, y, z, theta]` (height, width, length, position, rotation)
   - Each detection includes confidence score and other metadata

2. **State Prediction** (Kalman Filter):
   - Each tracked object has a 10D state: `[x, y, z, theta, l, w, h, dx, dy, dz]`
   - Uses constant velocity model to predict object positions in next frame
   - Predicts where existing tracks should be

3. **Ego Motion Compensation** (Optional):
   - Compensates for vehicle movement using GPS/IMU data
   - Transforms predicted tracks from previous frame's coordinate to current frame

4. **Data Association**:
   - Computes affinity/cost matrix between detections and predicted tracks
   - Uses distance metrics: 3D IoU, GIoU, 3D distance, Mahalanobis distance
   - Matches detections to tracks using:
     - **Hungarian Algorithm** (optimal assignment) or
     - **Greedy Matching** (faster, suboptimal)

5. **Track Management**:
   - **Update**: Matched detections update existing tracks via Kalman Filter
   - **Birth**: Unmatched detections create new tracks
   - **Death**: Tracks not matched for `max_age` frames are removed
   - **Output**: Only tracks with `hits >= min_hits` are output (prevents false positives)

6. **Output**: Tracked objects with consistent IDs across frames

### Key Components:

- **`AB3DMOT_libs/model.py`**: Main tracker class (`AB3DMOT`)
- **`AB3DMOT_libs/kalman_filter.py`**: Kalman Filter for state estimation
- **`AB3DMOT_libs/matching.py`**: Data association algorithms
- **`AB3DMOT_libs/dist_metrics.py`**: Distance/similarity metrics
- **`main.py`**: Entry point that processes sequences

## Project Structure

```
AB3DMOT/
├── main.py                 # Main entry point
├── configs/                # Dataset configuration files
│   ├── KITTI.yml
│   └── nuScenes.yml
├── AB3DMOT_libs/           # Core tracking library
│   ├── model.py           # Main tracker
│   ├── kalman_filter.py   # State estimation
│   ├── matching.py        # Data association
│   └── ...
├── data/                   # Dataset and detections
│   └── KITTI/
│       ├── detection/     # Pre-computed 3D detections
│       ├── tracking/      # KITTI dataset (if downloaded)
│       └── mini/          # Demo data
├── scripts/                # Evaluation and visualization
├── results/                # Output tracking results
└── Xinshuo_PyToolbox/      # Utility library (dependency)
```

## Getting It Working

### Prerequisites

- **Python 3.6+** (tested on Python 3.6, but you have 3.13.9)
- **Ubuntu 18.04+** (you're on Linux, which should work)
- Required packages (see `requirements.txt`)

### Installation Steps

#### 1. Set Up Python Environment

You already have a `venv` directory. Activate it:

```bash
cd /home/018218135/249/AB3DMOT
source venv/bin/activate
```

If packages aren't installed yet:
```bash
pip install -r requirements.txt
```

#### 2. Set Up PYTHONPATH

The code needs to find the `Xinshuo_PyToolbox` module. Add to your `~/.bashrc` or `~/.profile`:

```bash
export PYTHONPATH=${PYTHONPATH}:/home/018218135/249/AB3DMOT
export PYTHONPATH=${PYTHONPATH}:/home/018218135/249/AB3DMOT/Xinshuo_PyToolbox
```

Then reload:
```bash
source ~/.bashrc  # or source ~/.profile
```

Or set it temporarily in your current session:
```bash
export PYTHONPATH=${PYTHONPATH}:/home/018218135/249/AB3DMOT:/home/018218135/249/AB3DMOT/Xinshuo_PyToolbox
```

#### 3. Quick Demo (No Dataset Download Required)

The repository includes pre-computed detections for a quick demo:

```bash
# Run tracking on KITTI validation set with PointRCNN detections
python3 main.py --dataset KITTI --split val --det_name pointrcnn

# Post-process with confidence threshold
python3 scripts/post_processing/trk_conf_threshold.py --dataset KITTI --result_sha pointrcnn_val_H1

# Visualize results
python3 scripts/post_processing/visualization.py --result_sha pointrcnn_val_H1_thres --split val
```

This will:
- Process detections in `data/KITTI/detection/pointrcnn_*_val/`
- Generate tracking results in `results/KITTI/pointrcnn_val_H1/`
- Create visualizations

#### 4. Full Dataset Setup (Optional)

For full evaluation, download KITTI dataset:
- Download from: http://www.cvlibs.net/datasets/kitti/eval_tracking.php
- Extract to: `data/KITTI/tracking/`
- Structure should be:
  ```
  data/KITTI/tracking/
  ├── training/
  │   ├── calib/
  │   ├── image_02/
  │   ├── oxts/
  │   └── label_02/
  └── testing/
      ├── calib/
      ├── image_02/
      └── oxts/
  ```

### Running the Code

#### Basic Usage:

```bash
python3 main.py --dataset KITTI --split val --det_name pointrcnn
```

**Parameters:**
- `--dataset`: `KITTI` or `nuScenes`
- `--split`: `train`, `val`, or `test`
- `--det_name`: Detection method name (e.g., `pointrcnn`, `pvrcnn`, `centerpoint`)

#### Output Structure:

Results are saved in `results/KITTI/pointrcnn_val_H1/`:
- `data_0/`: Tracking results in KITTI format (for evaluation)
- `trk_withid_0/`: Tracking results with IDs (for visualization)
- `affinity/`: Affinity matrices between frames
- `log/`: Processing logs

### Evaluation

Evaluate 3D MOT performance:
```bash
python3 scripts/KITTI/evaluate.py pointrcnn_val_H1 1 3D 0.25
```

This computes metrics like:
- **MOTA**: Multiple Object Tracking Accuracy
- **MOTP**: Multiple Object Tracking Precision
- **IDS**: ID Switches
- **FPS**: Frames per second

### Visualization

Generate visualizations:
```bash
python3 scripts/post_processing/visualization.py --dataset KITTI --result_sha pointrcnn_val_H1_thres --split val
```

Outputs:
- `trk_image_vis/`: Per-frame visualizations
- `trk_video_vis/`: Video sequences

## Configuration

Edit `configs/KITTI.yml` to adjust:
- `score_threshold`: Filter low-confidence detections
- `num_hypo`: Number of hypotheses (1 = single hypothesis)
- `ego_com`: Enable/disable ego motion compensation
- `cat_list`: Object categories to track

## Troubleshooting

### Common Issues:

1. **Import errors**: Make sure PYTHONPATH is set correctly
2. **Missing dependencies**: Install from `requirements.txt`
3. **Version conflicts**: The code was tested with Python 3.6, but should work with 3.13
4. **No detections**: Check that detection files exist in `data/KITTI/detection/`

### Checking Installation:

```bash
# Test imports
python3 -c "from AB3DMOT_libs.model import AB3DMOT; print('OK')"
python3 -c "from xinshuo_io import mkdir_if_missing; print('OK')"
```

## Understanding the Code Flow

1. **`main.py`**:
   - Loads config from YAML
   - Loops through sequences and categories
   - Calls `main_per_cat()` for each category

2. **`main_per_cat()`**:
   - Loads detections for a sequence
   - Initializes tracker
   - Processes each frame:
     - Gets detections for frame
     - Calls `tracker.track()` to update tracks
     - Saves results

3. **`AB3DMOT.track()`** (in `model.py`):
   - Predicts track positions
   - Performs ego motion compensation
   - Matches detections to tracks
   - Updates/births/deaths tracks
   - Returns tracked objects

## Key Algorithms

- **Kalman Filter**: Constant velocity model for state prediction
- **Hungarian Algorithm**: Optimal assignment for data association
- **3D IoU/GIoU**: Distance metrics for matching
- **Ego Motion Compensation**: Transforms coordinates between frames

## Performance

On KITTI validation set with PointRCNN detections:
- **Car**: 86.47 MOTA, 108.7 FPS
- **Pedestrian**: 73.86 MOTA, 119.2 FPS
- **Cyclist**: 84.79 MOTA, 980.7 FPS

## Next Steps

1. Run the quick demo to verify installation
2. Explore the output results
3. Try different detection methods or parameters
4. Visualize tracking results
5. Evaluate on your own detections

## References

- Paper: "3D Multi-Object Tracking: A Baseline and New Evaluation Metrics" (IROS 2020)
- Project website: http://www.xinshuoweng.com/projects/AB3DMOT/
- Inspired by: SORT (Simple Online and Realtime Tracking)

