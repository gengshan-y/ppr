# PPR: Physically Plausible Reconstruction from Monocular Videos

This repo contains instructions for executing "PPR: Physically Plausible Reconstruction from Monocular Videos (ICCV 23)", which builds 4D models of the object and the environment by coupling differentiable rendering and differentiable physics simulation.
For differentiable physics simulation (4D to physics), please refer to our dedicated repo at [ppr-diffphysics](https://github.com/gengshan-y/ppr-diffphys).
For differentiable rendering (video to 4D), please see [lab4d](https://github.com/lab4d-org/lab4d).

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

### Get data
We show how PPR works using the `cat-pikachu-0` sequence. Begin by downloading the pre-processed video data.
```
bash scripts/download_unzip.sh "https://www.dropbox.com/scl/fi/j2bztcv49mqc4a6ngj8jp/cat-pikachu-0.zip?rlkey=g2sw8te19bsirr5srdl17kzt1&dl=0"
```
Note: Processing follows the [pre-processing pipeline](https://lab4d-org.github.io/lab4d/tutorials/preprocessing.html) of lab4d, except frames are extracted with a constant rate of 10 fps (no flow-guided filtering).

### Training
Training has 2 stages. We first optimize background and object. The following can be executed in parallel. 
To reconstruct the background scene:
```
# Args: gpu-id, sequence name, hyper-parameters defined in lab4d/config.py
bash scripts/train.sh lab4d/train.py 0 --seqname cat-pikachu-0 --logname bg --field_type bg --data_prefix full --num_rounds 60 --alter_flow --mask_wt 0.01 --normal_wt 1e-2 --reg_eikonal_wt 0.01
```
To reconstruct the object:
```
# Args: gpu-id, sequence name, hyper-parameters defined in lab4d/config.py
bash scripts/train.sh lab4d/train.py 1 --seqname cat-pikachu-0 --logname fg-urdf --fg_motion urdf-quad --num_rounds 20 --feature_type cse
```

The second stage is a physics-informed optimization that couples the object and the scene:
```
# Args: gpu-id sequence name, hyper-parameters in both lab4d/config.py and and projects/ppr/config.py
bash scripts/train.sh projects/ppr/train.py 0 --seqname cat-pikachu-0 --logname ppr --field_type comp --fg_motion urdf-quad --feature_type cse --num_rounds 20 --learning_rate 1e-4 --pixels_per_image 12 --iters_per_round 100 --secs_per_wdw 2.4 --noreset_steps  --noabsorb_base --load_path logdir/cat-pikachu-0-fg-urdf/ckpt_latest.pth --load_path_bg logdir/cat-pikachu-0-bg/ckpt_latest.pth
```
You may find the physics simulation results at `logdir/cat-pikachu-0-ppr/all-phys-00-05d.mp4`. 

Visualization at 0 iteration. Left to right: target, simulated, control reference, "distilled" simulation (to regularize diff-rendering).

https://github.com/gengshan-y/ppr/assets/13134872/22279f93-3e28-4e30-bbc8-f771f38257f0

Visualization at 1000 iteration:

https://github.com/gengshan-y/ppr/assets/13134872/3a769bb5-00d3-4f06-8d1c-6b3ffbdc37e2

Note: Multi-gpu physics-informed optimization is not supported as of now. 

### Visualization

To visualize the intermediate simulated trajectories over the course of optimization:
```
python projects/ppr/render_intermediate.py --testdir logdir/cat-pikachu-0-ppr/ --data_class sim
```
https://github.com/gengshan-y/ppr/assets/13134872/b1b18209-bd41-4f5d-a2d7-093b1b1dc8e2

To export meshes and visualize results, run
```
python projects/ppr/export.py --flagfile=logdir/cat-pikachu-0-ppr/opts.log --load_suffix latest --inst_id 0 --vis_thresh -10 --extend_aabb
```
https://github.com/gengshan-y/ppr/assets/13134872/1987a40f-1450-4cb9-a529-527a1b347fb3

There are a few other modes supported by the mesh renderer, such as bird's eye view, and ghosting.
```
python lab4d/render_mesh.py --testdir logdir/cat-pikachu-0-ppr/export_0000/ --view bev --ghosting
```
https://github.com/gengshan-y/ppr/assets/13134872/354ff079-627f-406b-bab6-32b5f53fa43c

### Interactive Visualization

We use [viser](https://github.com/nerfstudio-project/viser) for interactive visualization of the 4D scene.
Install viser by `pip install viser`. Once the result meshes have been exported to `logdir/$logname/export_$inst_id`, run
```
python lab4d/mesh_viewer.py --testdir logdir/cat-pikachu-0-ppr/export_0000/
```
https://github.com/gengshan-y/ppr/assets/13134872/fdf057dc-d97d-4b4c-ba17-a5fd5b9fdaff

### Simulation After Training

To run physics simulation with the optimized parameters (control reference, body mass, PD gains):
```
python projects/ppr/simulate.py --flagfile=logdir/cat-pikachu-0-ppr/opts.log --load_suffix latest --load_suffix_phys latest --inst_id 0
```

To visualize the output in the interactive mode:
```
python lab4d/mesh_viewer.py --testdir logdir//cat-pikachu-0-ppr-exp2/simulate_0000/sim_traj/
```
https://github.com/gengshan-y/ppr/assets/13134872/3b9e7adb-7529-4b2e-a199-a9214d43de28


## Results on AMA and Evaluation
### Training
Run the following script to get results on [AMA](https://people.csail.mit.edu/drdaniel/mesh_animation/) samba and bouncing sequences.
```
bash projects/ppr/run_ama.sh
```
It downloads the pre-processed data and then runs training and visualization routines. We suggest reading the comments in the script to understand what happens in each step. Expected results:



https://github.com/gengshan-y/ppr/assets/13134872/0849d538-7176-4465-b391-485f3b440b1b


https://github.com/gengshan-y/ppr/assets/13134872/ddd342bb-2bc3-4aad-ba7a-ad02d43672c1


### Evaluation
First download the [AMA data](https://people.csail.mit.edu/drdaniel/mesh_animation/) and place it at `database/ama/T_samba/{meshes/, calibration/}`. 
For running evaluation scripts, a few extra dependencies need to be installed:
```
pip intall git+https://github.com/facebookresearch/pytorch3d
```

Once the result meshes have been exported to `logdir/$logname/export_$inst_id`, execute
```
python projects/ppr/eval/compute_metrics.py --testdir logdir//ama-bouncing-4v-ppr-exp/export_0000/ --gt_seq D_bouncing-1 --pred_prefix "" --fps 30
```
This computes chamfer distance and f-score on the mesh sequence by comparing it against the ground truth.

Results of DiffRen optimization-only:
```
  Avg chamfer dist: 10.90cm
  Avg f-score at d=10cm: 85.6%
  Avg f-score at d=5cm:  61.4%
  Avg f-score at d=2cm:  25.3%
```
Results of DiffRen+DiffSim optimization:
```
  Avg chamfer dist: 8.85cm
  Avg f-score at d=10cm: 90.6%
  Avg f-score at d=5cm:  70.0%
  Avg f-score at d=2cm:  32.3%
```

Note: The paper uses the old codebase so the the numbers might differ. We plan to revise the paper so it matches this codebase.


## Custom video
See [lab4d pre-process doc](https://lab4d-org.github.io/lab4d/tutorials/preprocessing.html).
