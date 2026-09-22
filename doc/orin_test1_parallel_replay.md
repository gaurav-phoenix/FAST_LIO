# Orin Test 1 parallel replay, September 22 2026

Input: the original outdoor Test 1 bag at
`/home/orin/drone_ws/rosbags/lio_tests/20260921_143917_test1_682046616/bag`.
Only `/livox/point_cloud` and `/livox/imu` were replayed, from 110 seconds into
the bag for 100 seconds at normal speed. Each build ran in a separate install
tree under `/tmp/lio_match{1,2,3,4}_install`; output bags and logs are under
`/tmp/lio_replay_match{1,2,3,4}` inside `p30-edge`. `ROS_DOMAIN_ID=141` and
`ROS_LOCALHOST_ONLY=1` isolated the replay. No PX4 command topic was played.
The original bag and the existing installed executable were not used as output.

| Build | Sensor callback group | Odometry Hz | Age median / p95 | IMU callbacks | Max cloud queue | GPS-relative last difference |
|---|---|---:|---:|---:|---:|---:|
| 1 matching thread | Shared executor | 7.54 | 551 / 612 ms | 913 | 1 | 10.61 m |
| 2 matching threads | Shared executor | 9.97 | 82 / 84 ms | 14,465 | 1 | 1.85 m |
| 2 matching threads | Separate sensor group | 9.97 | 110 / 430 ms | 19,774 | 4 | 1.85 m |
| 3 matching threads | Separate sensor group | 9.98 | 66 / 68 ms | 19,672 | 1 | 1.85 m |

Age is replay `/clock` at odometry receipt minus the scan-end odometry stamp,
using the recorded 50 Hz replay clock. The one-thread run shows matching around
112 ms per scan and full processing sometimes over 200 ms. Three threads with
separate sensor callbacks held the observed cloud queue to one scan and had no
logged iKD-tree rebuild warning; the two-thread separated run emitted one such
warning and accumulated up to four clouds. The separate group consumed nearly
all of the recorded 200 Hz IMU stream. The two-thread shared-executor run
consumed about 145 Hz; one thread consumed about 9 Hz.

GPS-relative differences use raw Test 1 GPS UTC timestamps, reject GPS
brackets over 1 second, and align horizontal rotation and translation over
the first 30 airborne seconds without fitting scale. The later comparison covers
30 through 89 airborne seconds. Raw GPS is not surveyed truth, so the 1.85 m
endpoint is not an absolute LIO accuracy certificate. The 1-thread run also
produced fewer poses, which limits its direct endpoint comparison.

For the tested Orin configuration, build `fast_lio` with
`-DFAST_LIO_MATCH_THREADS=3` and use the hardware YAML parameter
`sensor_callbacks.concurrent: true`. Diagnostics are enabled in that YAML.
Both changes are opt-in outside the Orin hardware config: the CMake default
retains its architecture-specific behavior, and the callback group defaults to
false. Sensor subscriptions share one mutually exclusive group; the processing,
map publication, and map-save callbacks remain in the default group. The
shared input buffers and diagnostic state have locks; estimator updates remain
sequential.

This is an isolated replay result, not live flight or PX4 external-vision
acceptance. A disarmed bench run and subsequent controlled field validation are
still needed before relying on this configuration for navigation. Calibration,
sensor settings, timestamps, voxel sizes, and iteration count were unchanged.
