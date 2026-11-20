# OpenVINS for VIO-Benchmarks

This branch (`vio-benchmarks`) adds dataset configurations to OpenVINS. The source code is not changed.

- Upstream: [rpng/open_vins](https://github.com/rpng/open_vins), commit `6948812`.
- Project: [efekaanefe/VIO-Benchmarks](https://github.com/efekaanefe/VIO-Benchmarks). This repository is its submodule `open_vins_ws/src/open_vins/`.
- Mode: monocular camera and IMU (`ros1_serial_msckf`, `serial.launch`).
- Environment: ROS Noetic, Ubuntu 20.04, in Docker (`Dockerfile_ros1_20_04`).

The run, evaluation and plot scripts are in the project repository, in `open_vins_ws/`. The project README defines the datasets, the metrics and the failure criteria.

## 1. Configurations

| Folder | Dataset | Sequences |
|---|---|---|
| `config/kitti_2011_09_30_tuned/` | KITTI raw, date 2011-09-30 | odometry 04-10 |
| `config/kitti_2011_10_03_tuned/` | KITTI raw, date 2011-10-03 | odometry 00-02 |
| `config/kitti_2011_09_26_tuned/` | KITTI raw, date 2011-09-26 | odometry 03 (not used, drive not in the raw release) |
| `config/m3ed_car_urban_day/` | M3ED car, urban, day | 6 |
| `config/m3ed_car_urban_night/` | M3ED car, urban, night | 6 |
| `config/m3ed_car_forest/` | M3ED car, forest | 4 |
| `config/tartanground_diff/` | TartanGround, differential drive | 117 |

Each folder has `estimator_config.yaml`, `kalibr_imu_chain.yaml` and `kalibr_imucam_chain.yaml`.

Common settings: one camera, no stereo, online intrinsic and extrinsic calibration off, CLAHE histogram equalization on.

| Dataset | Specific settings |
|---|---|
| KITTI | 400 features, camera-IMU time offset calibration on, no zero-velocity update. One `estimator_config.yaml` for all dates. |
| M3ED | 250 features, zero-velocity update on, dynamic initialization on, IMU noise estimated from data. A less strict stationary detection is necessary for the night sequences. |
| TartanGround | 640x640 pinhole, f = 320 px. The camera-IMU transform is a rotation only. Dynamic initialization at 5 deg, gravity 9.8 m/s^2. |

## 2. Install

Do these steps in the project repository.

```bash
git clone --recurse-submodules https://github.com/efekaanefe/VIO-Benchmarks.git
cd VIO-Benchmarks/open_vins_ws
./ov_docker.sh image                                  # build the image ov_ros1_20_04
DATASETS_DIR=/abs/path/to/datasets ./ov_docker.sh true  # create and start the container "open_vins"
./ov_docker.sh build                                  # catkin build, Release
```

- Docker must be installed. Your user must be in the `docker` group.
- `JOBS=n` sets the number of compile jobs. The default is 4. Each job needs approximately 1.2 GB of RAM.
- The container mounts `open_vins_ws/` at `/catkin_ws` and `$DATASETS_DIR` at `/datasets`.
- `./ov_docker.sh` opens a shell. `./ov_docker.sh stop` removes the container. Remove the container before you change `DATASETS_DIR`.

## 3. Run one sequence

```bash
./ov_docker.sh roslaunch -p 11411 ov_msckf serial.launch config:=m3ed_car_urban_day dataset:=car_urban_day_horse \
  bag:=/datasets/VIO/M3ED/vins/car_urban_day_horse/data.bag bag_start:=0 max_cameras:=1 use_stereo:=false \
  dosave:=true path_est:=/catkin_ws/results/test/traj_estimate.txt dotime:=true path_time:=/catkin_ws/results/test/traj_timing.txt
```

`serial.launch` reads the bag one frame at a time. Thus the result does not depend on the computer speed.

The project scripts do the same for each dataset: `run_euroc.sh`, `run_kitti.sh`, `run_m3ed.sh`, `run_tartanground.sh`. Refer to `open_vins_ws/README.md` in the project repository.

## 4. Results

| Dataset | Runs X x N | Failed % | ATE SE(3) % | ATE Sim(3) % | RPE-t SE(3) % | RPE-t Sim(3) % | FPS |
|---|---|---|---|---|---|---|---|
| KITTI | 10 x 1 | 40.0 | 1.44 | 0.82 | 13.38 | 8.91 | 33.5 |
| M3ED | 16 x 10 | 0.0 | 0.72 | 0.51 | 4.46 | 3.43 | 48.0 |
| TartanGround | 117 x 5 | 7.5 | 0.31 | 0.21 | 1.58 | 1.32 | 45.8 |

Source: `open_vins_ws/results/table.tex` in the project repository. FPS is the processing rate (i7-5930K). The KITTI FPS is a lower limit, because 3 runs operated in parallel. A second PC made 10 x 10 KITTI runs: 30 % failed, ATE Sim(3) 3.00 %, 81.5 FPS. Those runs are not in the repository.

Observations:

- KITTI 04 (drive 0016) does not initialize. The drive is straight for 29 s, and the gyroscope measures only 4-5 deg of rotation. The configuration needs 10 deg (`init_dyn_min_deg`).
- KITTI 00, 01 and 02 (all 2011-10-03 drives) diverge. The scale error starts at initialization.
- The KITTI failures are deterministic. Three runs of drive 00 have the same initialization. Computing power is not the cause: each frame uses 24-32 ms of a 104 ms frame time.
- OpenVINS sets `cv::setRNGSeed(0)`. Thus runs differ only because of thread timing. On M3ED, 15 of 16 sequences give the same metrics (3 decimals) in all 10 runs.
- TartanGround: 44 of 585 runs diverge. Eight sequences diverge in all 5 runs. Usually the scale collapses to 0-0.05.
