---
layout: page
title: Projects
---

### [Vertical Farm Orchestrator (Python)](https://github.com/fuzzygreenblurs/vfarm)
A scalable on-prem service-oriented architecture for vertical farm automation that ingests sensor data via MQTT, processes it via stream-based analytics, and coordinates actuator control using distributed locking for thread-safe resource management. It includes simulation, alerting, and comprehensive testing, with built-in support for horizontal scaling and a migration path to Kafka for larger deployments.
<br><br>

### [Thread-Safe OS Virtual Memory Simulator (C)](https://github.com/fuzzygreenblurs/threadsafe_vmem/tree/main)
A thread-safe virtual memory management library that implements a two-level page table and TLB cache to translate virtual addresses into physical addresses efficiently. It is validated through single and multithreaded matrix multiplication benchmarks to ensure correctness and measure translation performance.
<br><br>
### [FPGA: General Purpose Processor (VHDL)](https://github.com/fuzzygreenblurs/fpga_asip)
An FPGA-based processor with application-specific instructions for VGA video and UART communication designed around a fully detailed custom ISA and integrated from ground-up IP blocks in Vivado. Verified via a full-system simulator running assembly programs produced by a custom assembler.
<br><br>
### [MagTile (Python)](https://ieeexplore.ieee.org/document/11400697) 
A modular electromagnetic planar motor for fine position control of small ferromagnetic payloads, designed to mimic the swimming action of zebrafish. The platform is built from physically chained custom PCBs driven by PCA9685 PWM controllers over I²C. Applied force is profiled using visual feedback and its non-linear dynamics characterized through system identification. Multi-agent trajectory control with collision avoidance via vision-based custom Dijkstra and steering-angle strategies.

_Published: IEEE Transactions for Control Systems Technology, Feb 2026._
<br><br>
### [6-DOF Attitude Estimator (C++)](https://github.com/fuzzygreenblurs/ahrs)
A [first-principles](https://github.com/fuzzygreenblurs/ahrs/blob/main/6dof_attitude_estimation.pdf) bare-metal AHRS on an STM32F446RE that fuses MPU9250 gyro and accelerometer data over I²C (using CMSIS-CORE) to estimate quaternion-based pitch and roll, benchmarking a Madgwick filter against an Extended Kalman Filter (EKF). Visualized using Three.js. 
<br><br>
### [A Simple MQTT Client/Server Scaffold (Python)](https://gist.github.com/fuzzygreenblurs/9e0606c7d4a300488b47a11673f59f96) 

A skeleton implementation of a secure Mosquitto based MQTT pipeline running on EC2. MQTT is a lightweight M2M protocol designed for IoT. 
{: style="text-align: justify"}

<br>

