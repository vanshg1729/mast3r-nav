# MASt3R-Nav

## Setup

### Clone Repository

Clone the repository with submodules:

```bash
# Clone with submodules
git clone --recursive https://github.com/vanshg1729/mast3r-nav.git
cd mast3r-nav/
```

If you already cloned without `--recursive`, initialize submodules:

```bash
# Initialize and update submodules
git submodule update --init --recursive
```

The repository includes two submodules:
- **libs/matcher/mast3r** - MASt3R for 3D scene reconstruction
- **libs/control/visualnav_transformer** - Visual Navigation Transformer for learned control

### Environment Setup with Pixi

This project uses [Pixi](https://pixi.sh) for reproducible environment management with Habitat-Sim (headless, Bullet physics), PyTorch, and all dependencies.

#### Install Pixi

```bash
# Install pixi (if not already installed)
curl -fsSL https://pixi.sh/install.sh | bash
``` 
#### Create Environment
```bash
# 1. Clean any previous installations
pixi clean
# 2. Install base environment (Python 3.10, NumPy 1.26, PyTorch, OpenCV, etc.)
pixi install

# 3. Build and install everything (habitat-sim, habitat-lab, PyG, MASt3R deps, etc.)
pixi run init
```
**Expected time**: 20 minutes (mostly for habitat-sim compilation)

## Checkpoints
Download the `MASt3R` checkpoint and our controller pre-trained model checkpoint:

```bash
mkdir -p checkpoints/

# download MASt3R_ViTLarge_BaseDecoder_512_catmlpdpt_metric.pth
wget https://download.europe.naverlabs.com/ComputerVision/MASt3R/MASt3R_ViTLarge_BaseDecoder_512_catmlpdpt_metric.pth -P checkpoints/

# download our controller checkpoint
mkdir -p checkpoints/gnm_mast3r_nav
wget https://huggingface.co/vanshg1729/mast3r-nav/resolve/main/latest.pth -P checkpoints/gnm_mast3r_nav
```

### Data
In `./data/`, sym link the following downloads as subdirs: `hm3d_v0.2`, `hm3d_iin_train` and `hm3d_iin_val`:

<details>
<summary> Data paths on Spectre </summary>

- **hm3d_v0.2** : `/scratch2/public_scratch/toponav/indoor-topo-loc/datasets/hm3d_navigation/hm3d_v0.2`
- **hm3d_iin_val** : `/scratch2/public_scratch/toponav/indoor-topo-loc/datasets/hm3d_navigation/hm3d_iin_val_320x240`
- **hm3d_iin_train** : `/scratch2/public_scratch/toponav/indoor-topo-loc/datasets/hm3d_navigation/hm3d_iin_train`

</details>

## Mapping

Create topological maps from RGB image sequences using MASt3R for 3D reconstruction:

```bash
pixi run python -m libs.mapper.create_topomap
# Or using pixi tasks
pixi run map
```

### Configuration

The mapper uses Hydra configuration from `configs/mapper/mapper_config.yaml`. Key parameters to modify:

**Important**: Update the model path in `mapper_config.yaml` to point to your downloaded MASt3R checkpoint:

```yaml
# Model settings
model:
  name: "mast3r"
  # Update this path to where you downloaded the checkpoint
  path: "/path/to/your/checkpoints/MASt3R_ViTLarge_BaseDecoder_512_catmlpdpt_metric.pth"
  device: "cuda"
  subsample_or_initxy1: 8

scenes:
  # Parent directory containing scene subdirectories with images/ folder
  base_dir: "/path/to/your/scenes"
  
  # Output directory for generated maps (optional, defaults to scene dir)
  base_out_dir: "/path/to/output"
  
  # Single scene mode
  multi_scene: false
  scene_name: "your_scene_name"
  
  # Multi-scene mode (set multi_scene: true)
  scene_list_file: "path/to/scene_list.txt"  # Optional: txt file with scene names
  start_idx: 0
  end_idx: -1  # -1 = all scenes
  step: 1
```

### Single Scene Mapping

To map a single scene:

1. Set `multi_scene: false` in config
2. Specify `scene_name` and `base_dir`
3. Run the mapper:

```bash
pixi run python -m libs.mapper.create_topomap scenes.base_dir=/path/to/scenes scenes.scene_name=my_scene_folder scenes.base_out_dir=/path/to/output
```

### Multi-Scene Mapping

To map multiple scenes:

1. Set `multi_scene: true` in config
2. Either provide a `scene_list_file` (txt file with one scene name per line) or use `start_idx/end_idx/step` to slice scene list
3. Run the mapper:

```bash
pixi run python -m libs.mapper.create_topomap scenes.multi_scene=true scenes.base_dir=/path/to/scenes scenes.base_out_dir=/path/to/output
```

### Expected Input Structure

Each scene should have the following structure:
```
scene_name/
├── images/           # RGB images (named sequentially)
│   ├── 0000.png
│   ├── 0001.png
│   └── ...
```

### Outputs

The mapper generates the following files in the output directory:

- `graph_base_*.pkl.b2s` - Compressed base topological graph (nodes + edges)
- `graph_with_distances_to_goal_*.pkl.b2s` - Graph with shortest path distances to goal
- `costmaps_*.npz` - Per-pixel distance-to-goal maps for all images
- `nodes_mast3r_points.npz` - 3D point clouds from MASt3R

Filename suffixes indicate configuration (image resolution, edge culling mode, node culling factor, etc.).

## Navigation

Run visual navigation episodes using pre-computed topological maps and costmaps:

```bash
pixi run python run_nav.py
# Or using pixi tasks
pixi run nav
```

### Configuration

The navigation system uses Hydra configuration from `configs/config.yaml`. Key parameters to modify:

**Important**: Update the controller checkpoint path in `configs/controller/gnm_waypixel.yaml` to point to your downloaded checkpoint:

```yaml
# Update this path to where you downloaded the checkpoint
load_run: "/path/to/your/checkpoints/gnm_mast3r_nav"
```

Other key parameters to modify in `configs/config.yaml`:

```yaml
# Episode paths
episode_path: "/path/to/single/episode"  # For single episode mode
episodes_dir: "/path/to/episodes"        # For multi-episode mode
hm3d_root_path: "/path/to/hm3d_v0.2/val"

# Costmap configuration
costmap_base_dir: "/path/to/costmaps"    # Directory containing generated costmaps
costmap_filename: "costmaps_320x240_EC_NONE_NC_NONE_NCF_10.npz"

# Results
results_dirpath: "/path/to/save/results"

# Episode selection
multi_episode: false                     # true for batch processing
episode_list: []                         # Optional: specific episodes to run
episode_list_file: null                  # Optional: txt file with episode names
```

### Single Episode Navigation

To run a single navigation episode:

1. Set `multi_episode: false` in config
2. Specify `episode_path`, `costmap_base_dir`, and `costmap_filename`
3. Run navigation:

```bash
pixi run python run_nav.py episode_path=/path/to/episode costmap_base_dir=/path/to/costmaps costmap_filename=costmaps_320x240.npz results_dirpath=/path/to/results
```

### Multi-Episode Navigation

To run multiple episodes in batch:

1. Set `multi_episode: true` in config
2. Specify `episodes_dir` containing episode subdirectories
3. Optionally provide `episode_list_file` (txt file with episode names, one per line) or use `episode_list` array
4. Run navigation:

```bash
pixi run python run_nav.py multi_episode=true episodes_dir=/path/to/episodes costmap_base_dir=/path/to/costmaps results_dirpath=/path/to/results
```
