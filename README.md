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
it is operating, the laser guns and LED panel will stop emitting until the
system has regained this condition.

As a result of this design, an infrared sensor placed somewhere in the field
of view of the lighthouse will produce a signal with the following shape:
![Timing diagram of the signal as seen from a single sensor of the lighthouse
lightfield.](
https://github.com/fughilli/ViveTrackedDevice/raw/master/pulse-shape-annotated.png)

```Δt_horiz``` and ```Δt_vert``` are the elapsed times from the phase-locked
internal 120Hz pulse to the sweep pulse for each horizontal and vertical timing
window, respectively. A successive pair of these measurements for each sensor
describes a set of rays, projected from the lighthouse, upon which each
corresponding sensor of the device must lie. The pitch and yaw angles of these
rays are calculated, in radians, as:
![Expressions for pitch and yaw angles as a function of delta t horizontal and
delta t vertical.](
https://github.com/fughilli/ViveTrackedDevice/raw/master/angle-calc.png)

When the sensor is dead front and center, these angles are both ```π/2```.

## System Architecture

The system consists of three main components:
- a sensor array with an analog frontend
- an FPGA, implementing a digital frontend
- a microcontroller, implementing the tracking algorithm

The sensor array captures the light pulses from the lighthouse using
photodiodes, and the analog frontend thresholds these signals dynamically to
produce clean digital signals for the digital frontend that are not disturbed by
varying ambient light levels.

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
case where two sensors lie on the same ray from the lighthouse. An additional
condition on overcoming this case is that the fourth sensor is non-coplanar with
the others.


