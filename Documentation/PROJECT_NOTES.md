# Project notes and verification status

## Objective
Develop and explore a line-synchronized TRIAC phase-angle controller for the rectified supply of a DC motor in Proteus using Raspberry Pi Pico and MicroPython.

## Implemented
- Proteus schematic with transformer, rectifiers and 7805 control supply
- H11AA1 zero-crossing input connected to GP14
- 2N2222 and MOC3021 gate-driving stage controlled from GP15
- T410-800B TRIAC, motor bridge and simulated DC motor
- V3 experimental firmware with a 20-second commanded firing-angle profile

## Observed
Proteus simulation ran and produced screenshots of the motor circuit and voltage display. Some earlier attempts produced a SPICE convergence error around U5; the final screenshots were captured after wiring adjustments.

## Not independently validated
- Accurate TRIAC firing-angle waveforms and actual RPM vs firing angle
- Full automatic increase and decrease of motor RPM under controlled load
- EMI and harmonic measurements
- Long-term simulator and hardware stability
- Physical electrical safety and compliance

## Constraints
The available laptop limited comprehensive simulation testing. The simulation is an educational prototype, not a verified 230 V / 375 W motor controller.
