---
layout: page
title: Attitude Measurement Test Jig
permalink: /projects/delta-robot-attitude-test-jig/
---

## High Level Goals

This rotation stage test jig is used as a platform to evaluate the performance of various filters in real-time attitude measurement of a given workspace using a `MPU6050` gyroscope-accelerometer sensor module. 

It leverages a delta-robot configuration to rotate the platform about various axes. It is a rebuild of the excellent project put together by [Aaed Musa](https://www.youtube.com/watch?v=v4F-cGDGiEw&t=400s&ab_channel=AaedMusa). While the mechanical setup is largely the same, the primarly goal of the platform is simply to achieve a given target attitude with a high degree of accuracy and stability.

This variant of the project emphasizes the following learning goals:

1. interfacing with multiple stepper motors using a custom motor driver
2. compare complementary, LKF and EKF variants to measure attitude
3. compare various attitude representations (euler, quaternion, DCM)
4. develop practical understanding of the kinematics of a Delta Robot
5. C++ based TDD (following the [JSF Standard](https://www.stroustrup.com/JSF-AV-rules.pdf)) 
6. implement Bluetooth-based communication between a pair of RP2040s

## Motor Driver

In this assembly, stepper motors are used in a coordinated fashion to orient the workspace at a desired angle. We can write a simple a motor driver, which will act as the software interface between the RP2040 and the physical motor circuit.

### Operating Principle: Stepper Motors

Let's first discuss the general operating interface of our stepper motor: `NEMA 17 Hybrid Synchronous Stepper Motor`. 

Firstly, stepper motors are brushless DC motors that rotate in discrete steps. This is highly useful for fine position control in open-loop. It consists of a rotor (permanent magnet) which is surrounded by a stator (which is essentially a set of coil windings). 

By activating the windings step-by-step in a particular order (letting current flow through them), the stator is magnetized, which creates a pair of corresponding poles that generate a coupling attractive-repulsive force on the rotor making it rotate for a step.

The rotor consists of 4 permanent magnetic sprockets of alternating polarity. Each sprocket has 50 teeth, and the alternating sprocket teeth are offsetted radially from each other as seen in the drawings. 

The stator consists of 8 physically separated coils that are driven by 4 wires. Thus, the 8 coils can be split up into two sets of 4 coils, and these set of 4 coils can simply be treated as 1 big wire with two ends. Thus, we can think the whole stator assembly as two wires which are spread apart from each other around the rotor in a 90 degree orientation. 

Note that for each set of 4 coils, the opposing coils on the stator will produce the same polarity, while the other two create a reverse magnetic polarity. 






