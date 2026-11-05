# Sherpa RP HMI — System Architecture

This document describes the actual frontend/backend/robot integration structure documented in the supplied `sherpa_hmi_complete_v2.zip`. It is intentionally an architecture document: it describes components, boundaries, responsibilities and data flows, but does not publish implementation source code, secrets, deployment scripts or private configuration files.

## 1. System context

The Sherpa RP HMI is a browser-based operator console. The operator connects from a laptop or tablet over the local network. The on-robot Jetson Orin Nano hosts the web entry point and supporting services, while the ROS 2 Humble robot workspace runs in a Docker container.

The browser is deliberately isolated from ROS 2 DDS/RTPS. It uses ordinary web protocols through the Jetson-hosted services, so the operator device does not need ROS 2, a DDS middleware installation, a colcon workspace or ROS domain configuration.

## 2. Logical layers

### A. Operator / presentation layer

A static browser frontend provides the operator-facing pages and shared dashboard components.

| Frontend area | Responsibility |
|---|---|
| Login | Authenticated entry to the operator interface |
| Home | Robot overview, connectivity, battery and mode indicators |
| Mapping | Mapping workflow controls, live occupancy-grid display and map saving |
| Navigation | Saved-map selection, localization initialization, goal setting, route and workflow status |
| Teleoperation | Manual drive controls, keyboard/pad input and HMI safety latch |
| Sensors | Sensor and robot diagnostic information |
| Camera | Live camera viewing through the video service |
| Settings | Operator-facing configuration/status area |
| Developer | Diagnostic and developer-facing interface area |

The UI is structured around reusable state and workflow interfaces, rather than embedding robot launch details in each page.

### B. Web edge layer

Nginx is the single browser-facing entry point on the Jetson. Its responsibilities are:

- Serve the static HMI frontend and its assets.
- Reverse-proxy browser API requests to the FastAPI backend.
- Forward the ROS WebSocket path to rosbridge.
- Forward camera video requests to the MJPEG video service without buffering the stream.
- Forward the optional Foxglove WebSocket path to foxglove_bridge.

The supplied deployment documentation also describes local-network discovery through Avahi/mDNS using a `.local` hostname, so an operator can discover the HMI without manually looking up the Jetson's current IP address.

### C. Application / coordination backend

A Python FastAPI service runs on the Jetson and provides the application boundary between the browser and privileged robot-side operations.

| Backend responsibility | Role in the system |
|---|---|
| Authentication and sessions | Protect the operator interface and its application functions |
| Robot status and diagnostics | Aggregate and expose available robot/ROS status to the frontend |
| Workflow coordination | Start, stop, restart and report the state of approved robot workflows |
| Launch lifecycle management | Coordinate ROS launch processes inside the Docker container and report readiness/failure |
| Map service | List/select maps, provide map previews and coordinate map saving |
| Configuration/status | Present supported HMI configuration and diagnostic information |
| Network coordination | Validate supported network requests and forward them to a separate local service |
| Bluetooth coordination | Validate supported Bluetooth requests and forward them to a separate local service |

Workflow definitions are held in a registry/configuration layer. This creates one consistent lifecycle path for Teleoperation, Mapping and Navigation, including precondition checks, mutual exclusion, readiness reporting and stop/failure handling.

### D. Local device-service layer

Network and Bluetooth operations are separated from the main HMI API into dedicated local helper services. The browser cannot call these privileged helpers directly. The HMI backend forwards supported requests over local-only service communication, and the helper services perform their own validation before interacting with the host operating system.

This separation reduces the privileges and responsibilities exposed through the browser-facing API. It is an architectural boundary, not a claim that the HMI alone is a safety-rated or security-certified product.

### E. Robotics runtime layer

The ROS 2 Humble workspace runs in Docker on the Jetson Orin Nano. The HMI coordinates approved workflows; the robot's ROS 2 packages remain responsible for sensing, state estimation, mapping, navigation, velocity arbitration, safety gating and hardware control.

The robot-side stack documented by the project includes:

- Robot bringup and hardware/control interfaces.
- Sensor drivers and sensor processing, including LiDAR and camera data.
- Odometry/state-estimation components.
- Cartographer-based mapping.
- Map server and AMCL localization.
- Nav2 planning, controller and behavior components.
- Velocity muxing and the command-velocity safety gate.
- HAL / motor-control interfaces.

## 3. Communication paths

| Path | Protocol / data | Purpose |
|---|---|---|
| Browser ↔ Nginx | HTTP or HTTPS | Load the HMI and access the single web entry point |
| Frontend ↔ FastAPI | HTTP REST | Authentication, status, workflows, maps, configuration and supported device operations |
| Frontend ↔ rosbridge | WebSocket with ROS messages represented for web clients | Subscribe to robot state and publish supported operator inputs |
| Frontend ↔ video service | HTTP MJPEG | Stream camera frames efficiently without packaging images into the ROS JSON WebSocket path |
| External 3D tool ↔ foxglove_bridge | Foxglove WebSocket protocol | Optional 3D/robot-data visualization through an external viewer such as Lichtblick |
| FastAPI ↔ ROS runtime | Docker-controlled workflow lifecycle and status checks | Start/stop approved ROS workflows and inspect their readiness |
| FastAPI ↔ local network/Bluetooth helpers | Local-only authenticated service requests | Perform supported host network and Bluetooth operations |

### Why the browser does not use DDS directly

The browser has no direct DDS participant. This avoids requiring ROS 2 and DDS discovery on operator laptops/tablets, reduces dependence on multicast discovery across different Wi-Fi/VPN/NAT networks, and gives the HMI a web/API boundary for authentication and input validation. ROS 2 distribution-specific details remain on the robot-side Jetson/container.

### Why camera video has its own path

Camera images are streamed through the dedicated MJPEG service. This avoids the extra overhead of transporting video frames as base64 image payloads inside a JSON WebSocket stream and gives the camera page a direct video-oriented transport.

## 4. Robot workflow architecture

### Teleoperation

1. The operator requests the Teleoperation workflow from the HMI.
2. The backend checks workflow state and starts the approved ROS launch inside the ROS 2 container.
3. The frontend enables manual input only when the operator has enabled the HMI safety control, the robot is in the expected manual state and the ROS WebSocket connection is healthy.
4. Joystick/keyboard input is sent through the dedicated manual velocity input (`/cmd_vel_joy`).
5. The downstream robot command chain arbitrates manual and autonomous velocity sources through the velocity mux and safety gate before the final motor-control interface.
6. Releasing the control, losing browser focus or losing the ROS WebSocket connection results in a zero-velocity command / stop of the teleoperation publish loop.

The HMI does not publish directly to the final `/cmd_vel` motor command topic. The HMI's browser-side stop logic supplements, but does not replace, the robot's downstream safety gate or physical emergency stop.

### Mapping

1. The operator selects Mapping and requests Start.
2. The backend checks whether a conflicting workflow is already active, then starts the approved mapping launch inside the ROS 2 container.
3. The mapping stack brings up the required sensor/processing, odometry and Cartographer components.
4. The live occupancy grid (`/map`) is delivered to the browser and rendered in the mapping view with the robot marker.
5. The operator uses the shared, safety-gated manual drive interface to move the robot while mapping.
6. The map-save operation invokes the ROS map saver through the backend, validates the saved map artifacts and makes the resulting map available to the navigation map store.
7. Stopping Mapping tears down the workflow and locks the associated manual-drive controls.

**Documented operational limitation:** the supplied project notes state that Livox obstacle avoidance (the publisher of the obstacle-stop signal) is not included in the Mapping launch. Therefore automatic obstacle stopping must not be assumed active while Mapping is running. This limitation should be made clear to operators and addressed in the robot's approved operating procedure.

### Navigation

1. The operator selects a saved map and requests Navigation.
2. The backend checks that the map is available and starts the approved Nav2 launch inside the ROS 2 container.
3. The ROS 2 stack provides map server, AMCL localization and Nav2 planning/control components.
4. The browser can set the robot's initial pose and publish a navigation goal; the UI displays localization, route/plan, goal status, workflow readiness and reported obstacle-stop state.
5. Nav2 velocity commands pass through the robot's velocity mux and safety gate before reaching the motor-control layer.
6. The operator can cancel a goal or stop the workflow through the HMI.

### Sensor and robot-state monitoring

The frontend subscribes to supported ROS state streams through rosbridge and/or reads aggregated backend diagnostics. The project documentation describes map, localization, plan, goal-status, odometry, battery, controller and sensor data as HMI-visible information. Individual topic names and message mappings must follow the target robot's verified interface configuration.

### Camera viewing

The browser camera page requests an MJPEG stream from the robot-side video service through Nginx. This path is separate from rosbridge and is intended for efficient live video delivery.

## 5. Motion-command and safety boundaries

The project documents the following command separation:

- Manual teleoperation uses the HMI's dedicated manual command input.
- Nav2 uses its own autonomous velocity input.
- A downstream velocity mux arbitrates command sources.
- The safety gate is downstream of the mux and upstream of the hardware-control interface.
- The HMI does not write directly to the final motor command interface.

The frontend applies additional guards: the safety latch must be enabled, the expected mode must be active and the ROS WebSocket must be connected. Losing the WebSocket stops the frontend's teleoperation publishing path.

**Safety qualification:** this software architecture is not a certified safety controller. The robot must retain an independent physical emergency stop and the validated robot-side safety mechanisms. The obstacle-stop topic shown in the UI must not be confused with a safety-rated E-stop circuit.

## 6. Deployment topology

| Component | Deployment location | Role |
|---|---|---|
| HMI browser | Operator laptop/tablet | User interface only; no ROS 2 installation required |
| Nginx | Jetson Orin Nano host | Static serving and reverse proxy |
| FastAPI HMI backend | Jetson host | Authenticated API and workflow coordination |
| Network helper | Jetson host, local-only | Supported network operations |
| Bluetooth helper | Jetson host, local-only | Supported Bluetooth operations |
| ROS 2 Humble workspace | Docker container on Jetson | Robot drivers, state estimation, SLAM, Nav2, control and hardware interfaces |
| rosbridge | ROS 2 container | Browser-facing ROS WebSocket bridge |
| web_video_server | ROS 2 container | Camera MJPEG endpoint |
| foxglove_bridge | ROS 2 container | Optional external 3D diagnostics data path |
| system services / mDNS | Jetson host | Process startup and local-network discovery |

The HMI frontend is described as static content and does not require a frontend compilation/build step. The robot-side ROS 2 workspace remains separately managed and built in its ROS 2 environment.

## 7. Configuration and integration contract

The supplied project uses a central configuration surface for:

- ROS launch workflow registry and expected-node readiness.
- ROS topic/service interface mappings.
- ROS workspace and Docker runtime information.
- HMI service ports and robot model metadata.

The source documentation explicitly marks some settings as assumptions requiring verification on the target robot. These include selected safety/battery/controller/mode/diagnostic interfaces, ROS container naming, bridge service ports and teleoperation speed limits. They must be reconciled with the actual Jetson deployment before use. This public document does not publish the private configuration files or their deployment values.

## 8. Validation record and known caveats

The supplied project README reports:

- 73 backend automated tests passing in its documented test environment.
- A static workflow/launch-registry consistency check.
- 46 browser end-to-end tests using test doubles for Docker and rosbridge.

Those checks are useful software-level evidence, but they do not equal a complete acceptance test on the physical Jetson/Sherpa RP. The same supplied README distinguishes target-hardware checks as requiring verification on the Jetson.

The supplied architecture notes also contain the following implementation caveats which should be checked before production use:

1. The vendored `roslibjs` file is described in the archive's architecture notes as a minimal stub rather than the official ROSLIB distribution. Confirm/replace this with the official compatible library for a production deployment.
2. The optional Foxglove/Lichtblick 3D-viewer integration is described in the architecture notes, but the frontend file inventory in the supplied archive does not include the referenced 3D-view page. Verify the operator-facing page and external viewer connection in the deployed build before advertising it as a completed UI feature.
3. Some ROS interface names, service ports and teleoperation limits are explicitly marked as requiring integration verification.
4. The Mapping launch notes state that the Livox obstacle-avoidance publisher is not part of the Mapping workflow; this affects which automatic obstacle-stop behavior is available during Mapping.

## 9. Public showcase boundary

This repository is intentionally limited to architecture-level material and selected screenshots. It excludes:

- Frontend HTML/CSS/JavaScript implementation.
- Backend Python application and routers.
- ROS 2 workspace packages and launch source.
- Private configuration and environment files.
- Deployment/systemd scripts and Nginx implementation configuration.
- TLS private keys, certificates used for deployment, tokens, passwords and other secrets.
- Historical backups and archive copies.

**For the complete source code, contact E MADAN KUMAR.**
