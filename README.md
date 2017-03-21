# ViveTrackedDevice

This repo contains all of the documentation and source code for a
SteamVR-compatible tracked device. This implementation is derived from reverse
engineering work I performed during my EE113D design project to determine the
operating principles behind the SteamVR tracking system. My project report for
that design project is available [here]().

## The SteamVR System

The SteamVR tracking system consists of two main components:
- a _lighthouse_ (or _lighthouses_)
- a _tracked device_ (or _tracked devices_)

### The Lighthouse

The _lighthouse_ is a device which produces a spatiotemporally indexing light
field that sensors on the _tracked device_ can measure to determine position
and orientation with low latency and high accuracy. The first generation
lighthouses that ship with the HTC Vive virtual reality system consist of two
rotary line laser guns and an LED panel, all of which emit infrared light. The
two rotary guns are rotating at 60 revolutions per second in planes
perpendicular to one another, and are phase-locked so as to take turns sweeping
across the field of view of the device (180 degrees out-of-phase). The LED
panel flashes at 120Hz, and is phase-locked with the rotors so as to produce
short flashes in between rotary gun sweeps. A diagram of the lighthouse
construction is given below:
![Drawing of the HTC Vive lighthouse, depicting the rotor and LED panel
positions and orientations.](
https://github.com/fughilli/ViveTrackedDevice/raw/master/lighthouse-schematic.png)
Note that the vertical rotor is facing the back of the device when the
horizontal rotor is facing forwards. This phase condition is enforced by the
control loop running the two rotor motors; if the lighthouse is rotated while
it is operating (thus imparting an angular moment that disturbs the phase lock),
the laser guns and LED panel will stop emitting until the system has regained
this condition.

As a result of this design, an infrared sensor placed somewhere in the field
of view of the lighthouse will produce a signal with the following shape (given
in blue):
![Timing diagram of the signal as seen from a single sensor of the lighthouse
lightfield.](
https://github.com/fughilli/ViveTrackedDevice/raw/master/pulse-shape-annotated.png)

The green line in the plot above is the output of the internal 120Hz PLL. It is
phase-locked against the thin synchronization pulses. ```Δt_horiz``` and
```Δt_vert``` are the elapsed times from the phase-locked internal 120Hz pulse
to the sweep pulse (wider blue pulse) for each horizontal and vertical timing
window, respectively. A successive pair of these measurements for each sensor
describes a set of rays, projected from the lighthouse, upon which each
corresponding sensor of the device must lie. The pitch and yaw angles of these
rays are calculated, in radians, as:
![Expressions for pitch and yaw angles as a function of delta t horizontal and
delta t vertical.](
https://github.com/fughilli/ViveTrackedDevice/raw/master/angle-calc.png)

When a sensor is dead front and center, the angles for that sensor's ray are
both ```π/2```.

The SteamVR system, and the HTC Vive for that matter, is capable of using more
than one lighthouse simultaneously in the tracking space. In this scenario, one
lighthouse is designated as the "master", and the additional lighthouses are
"slaves". The master sets the timebase (the other lighthouses align their 120Hz
sync pulses against the master optically), and then each lighthouse takes turns
round-robin-style sweeping their rotors through their FOV. In a two-lighthouse
setup, the rotors are spinning at 30Hz apiece in order to achieve this.

### The Tracked Device

The tracked device is a set of light sensors, as described above, arranged on
the surface of some physical object. In the case of the HTC Vive system, the
tracked devices are the HMD (Head-Mounted Display) and the two handheld
controllers. The small indentations covering the surfaces of these devices are
apertures behind which sensors are located:
![Small indentations on the HTC Vive handheld controller, under which infrared
sensors are hiding.](
https://github.com/fughilli/ViveTrackedDevice/raw/master/vive-controller-annotated.png)

The arrangement of the sensors on the device is known _a priori_, and as such,
the problem of determining the position and orientation of the device is to find
a position and orientation for which the linear error (Euclidean distance)
between the estimated positions of the sensors and the nearest points on their
corresponding rays is as small as possible.

In the HTC Vive system, each device also has an IMU (Inertial Measurement Unit)
which is used to determine the high frequency pose changes that are too fast or
minute to reliably capture with the light-based tracking method.

## Problem Definition

In order to track an object, the following problems have to be solved:
- detect the light pulses from the lighthouse
- determine ```Δt_horiz``` and ```Δt_vert```
- solve for the transformation that minimizes the error between the inferred
device state and the measured rays

The optimization problem can be described more specifically as:
![Formal problem definition.](
https://github.com/fughilli/ViveTrackedDevice/raw/master/problem-statement.svg)

## System Architecture

The system consists of three main components:
- a sensor array with an analog frontend
- an FPGA, implementing a digital frontend
- a microcontroller, implementing the tracking algorithm

The sensor array captures the light pulses from the lighthouse using
photodiodes, and the analog frontend thresholds these signals dynamically to
produce clean digital signals for the digital frontend that are not disturbed by
varying ambient light levels.

The circuit for one sensor of this sensor array is given below:
![Schematic of a single sensor and its analog frontend.](
https://github.com/fughilli/ViveTrackedDevice/raw/master/analog-frontend-part.png)

The FPGA implements a soft PLL which phase-aligns an internal 120Hz clock
against the lighthouse's 120Hz sync pulse. Once the desired accuracy has been
achieved, timer capture blocks begin counting on the internal 120Hz clock, and
stop counting at the falling edge of each of the sweep pulses. These captured
values are buffered on the FPGA, and an interrupt line to the MCU flags that
the data is ready.

The microcontroller implements an interative solver which minimizes the linear
error of the inferred device state against the measurements from the frontend.
A visualization of the steps of this algorithm can be found in the
```ray_solver``` branch of the [ViveVisualizer](
https://github.com/fughilli/ViveVisualizer) repository.

The sensor array consists of at least four sensors and their analog frontends.
At least four sensors is required for the algorithm to overcome the degenerate
case where two sensors lie on the same ray from the lighthouse. In this
situation, there are two possible solutions that minimize the error function,
which are related by a 180-degree rotation that swaps the position of the two
collinear sensors. An additional condition on overcoming this case is that the
fourth sensor is non-coplanar with the others.

This tracked device implementation does not make use of an IMU. However,
accuracy and latency stand to be significantly improved by the incorporation of
an IMU.
