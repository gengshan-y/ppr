# PPR: Physically Plausible Reconstruction from Monocular Videos

This repo contains instructions for running "PPR: Physically Plausible Reconstruction from Monocular Videos (ICCV 23)". 
If you are interested in differentiable physics simulation (4D to physics), see the standalone repo [ppr-diffphysics](https://github.com/gengshan-y/ppr-diffphys).
If you are interested in differentiable rendering (video to 4D), see [lab4d](https://github.com/lab4d-org/lab4d).
Read on if you are interested in coupling DiffRen and DiffSim.

**[[Project page]](https://gengshan-y.github.io/ppr/)**

## Installation
Follow instructions to install [lab4d](https://lab4d-org.github.io/lab4d/get_started/). 
Inside lab4d root directory, checkout the ppr branch and update the diffphys submodule.
```
git checkout ppr
git submodule update --recursive
```
Activate lab4d conda environment and install additional dependencie:
```
pip install -r projects/ppr/ppr-diffphys/requirements.txt
pip install urdfpy==0.0.22 --no-deps
pip install open3d==0.17.0
pip install geomloss==0.2.6
```

## Video to 4D and Physics
We show how PPR works using the `cat-pikachu-0` sequence. Begin by downloading the pre-processed video data.
```
bash scripts/download_unzip.sh "https://www.dropbox.com/scl/fi/j2bztcv49mqc4a6ngj8jp/cat-pikachu-0.zip?rlkey=g2sw8te19bsirr5srdl17kzt1&dl=0"
```
Note: Processing follows the [pre-processing pipeline](https://lab4d-org.github.io/lab4d/tutorials/preprocessing.html) of lab4d, except frames are extracted with a constant rate of 10 fps (no flow-guided filtering).

Training has 2 stages. We first optimize background and object. The following can be executed in parallel. 
To reconstruct the background scene:
```
# Args: gpu-id, sequence name, hyper-parameters defined in lab4d/config.py
bash scripts/train.sh lab4d/train.py 0 --seqname cat-pikachu-0 --logname bg --field_type bg --data_prefix full --num_rounds 60 --alter_flow --mask_wt 0.0 --normal_wt 1e-3 --reg_eikonal_wt 0.01
```
To reconstrut the object:
```
# Args: gpu-id, sequence name, hyper-parameters defined in lab4d/config.py
bash scripts/train.sh lab4d/train.py 1 --seqname cat-pikachu-0 --logname fg-urdf --fg_motion urdf-quad --num_rounds 20 --feature_type cse
```

Second stage is a physics-informed optimization that couples the object and the scene:
```
# Args: gpu-id sequence name, hyper-parameters in both lab4d/config.py and and projects/ppr/config.py
bash scripts/train.sh projects/ppr/train.py 0 --seqname cat-pikachu-0 --logname ppr --field_type comp --fg_motion urdf-quad --feature_type cse --num_rounds 20 --learning_rate 1e-4 --pixels_per_image 12 --iters_per_round 100 --ratio_phys_cycle 0.5 --phys_vis_interval 20 --secs_per_wdw 2.4 --noreset_steps  --noabsorb_base --load_path logdir/cat-pikachu-0-fg-urdf/ckpt_latest.pth --load_path_bg logdir/cat-pikachu-0-bg/ckpt_latest.pth
```
You may find the physics simulation results at `logdir/cat-pikachu-0-ppr/all-phys-00-05d.mp4`. 

Visualization at 0 iteration. Left to right: target, simulated, control reference, "distilled" simulation (to regularize diff-rendering).

https://github.com/gengshan-y/ppr/assets/13134872/22279f93-3e28-4e30-bbc8-f771f38257f0

Visualization at 1000 iteration:

https://github.com/gengshan-y/ppr/assets/13134872/3a769bb5-00d3-4f06-8d1c-6b3ffbdc37e2

Note: Multi-gpu physics-informed optimization is not supported as of now. 

## Visualization

To visualize the intermediate simulated trajectories over the course of optimization:
```
python projects/ppr/render_intermediate.py --testdir logdir/cat-pikachu-0-ppr/ --data_class sim
```
https://github.com/gengshan-y/ppr/assets/13134872/b1b18209-bd41-4f5d-a2d7-093b1b1dc8e2

To export meshes and visualize results, run
```
python projects/ppr/export.py --flagfile=logdir/cat-pikachu-0-ppr/opts.log --load_suffix 0060 --inst_id 0 --vis_thresh -10 --extend_aabb
```
https://github.com/gengshan-y/ppr/assets/13134872/1987a40f-1450-4cb9-a529-527a1b347fb3

There are a few other modes supported by our mesh renderer, such as bird's eye view, and ghosting.
```
python lab4d/render_mesh.py --testdir logdir/cat-pikachu-0-ppr/export_0000/ --view bev --ghosting
```
https://github.com/gengshan-y/ppr/assets/13134872/354ff079-627f-406b-bab6-32b5f53fa43c


We use [viser](https://github.com/nerfstudio-project/viser) for interactive visualization of the 4D scene.
Install viser by `pip install viser`. Once the result meshes have been exported to `logdir/$logname/export_$inst_id`, run
```
python lab4d/mesh_viewer.py --testdir logdir/cat-pikachu-0-ppr/export_0000/
```
https://github.com/gengshan-y/ppr/assets/13134872/fdf057dc-d97d-4b4c-ba17-a5fd5b9fdaff


## Evaluation (WIP)
First download the [AMA data](https://people.csail.mit.edu/drdaniel/mesh_animation/) and place it at `database/ama/T_samba/{meshes/, calibration/}`. Then install a few extra dependencies:
```
pip intall git+https://github.com/facebookresearch/pytorch3d
```

Once the result meshes have been exported to `logdir/$logname/export_$inst_id`, execute
```
python projects/ppr/eval/compute_metrics.py --testdir logdir//ama-bouncing-4v-ppr-exp/export_0000/ --gt_seq D_bouncing-1 --pred_prefix "" --fps 30
```
This comptues chamfer distance and f-score on the mesh sequence by comparing against the ground-truth.

Results of DiffRen optimization-only:
```
  Avg chamfer dist: 8.28cm
  Avg f-score at d=10cm: 92.5%
  Avg f-score at d=5cm:  72.1%
  Avg f-score at d=2cm:  33.9%  
```
Results of DiffRen+DiffSim optimization:
```
  Avg chamfer dist: 7.85cm
  Avg f-score at d=10cm: 93.4%
  Avg f-score at d=5cm:  74.3%
  Avg f-score at d=2cm:  35.3%
```

Note: The paper uses the old codebase so the the numbers might differ. We plan to revise the paper so it matches this codebase.


## Custom video
See [lab4d pre-process doc](https://lab4d-org.github.io/lab4d/tutorials/preprocessing.html).
