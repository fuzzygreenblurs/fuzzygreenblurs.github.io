---
layout: page
title: Projects
---

<br>

## Quadrotor Flight Controller 

### [Attitude Estimation](/projects/quadrotor/attitude-estimation)

Optional: A rotation stage test jig is used as a platform to evaluate the performance of various filters in real-time attitude measurement of a given workspace using a gyroscope-accelerometer sensor module.

The optimal filter identified by this test will be employed on the quadrotor controller thereafter. 

## A Simple MQTT Client/Server Scaffold

[This is a basic implementation](https://gist.github.com/fuzzygreenblurs/9e0606c7d4a300488b47a11673f59f96) of a Mosquitto based MQTT pipeline. MQTT is a lightweight M2M protocol designed with IoT applications in mind. For example, if we want to read the state of an assembly line and trigger changes to the system, we might use an MQTT based pipeline to do this.
{: style="text-align: justify"}

<br>

## 2D Physics Engine (Pikuma Course)

*Note: all demos in this section are built using vanilla C++17 and SDL2*
{: style="text-align: justify"}

Impulse Collision Resolution: [This demo](https://youtu.be/uDtQh0QGs48) showcases collision detection and resolution using a penetration and impulse calculation of circular objects of different masses at a different coefficients of resitution. In the showcased demo, one object is maintained in a static position (`invMass = 0`) while new objects are dynamically spawned in the environment, showcasing multiple simultaneous collisions. You can find the code for this specific demo in this [commit](https://github.com/fuzzygreenblurs/2d_game_physics_engine/commit/ba62d30afc7afab7a5d92c4412c0ffd2a9d1d162).
{: style="text-align: justify"}

<br>

## CUNY Emerging Scholars Program (ESP):  6 DoF SCARA Robotic Arm

In 2022, as a part-time student at CUNY, I was awarded the ESP Scholarship to showcase a 6 degree-of-freedom SCARA robotic arm that could perform a simple pick-and-place action (self-studying the [Angela Sodemann](https://www.robogrok.com/Robotics_1.php) Robotics 1 course and kit) and writing a [short paper](https://github.com/fuzzygreenblurs/3dof_control/blob/main/scara_esp_paper.pdf) summarizing the approach. This project was performed as a practical demonstration of linear transformations in the physical domain. The manipulator was made up of a Raspberry Pi that actuated 2 revolute servo joints and a prismatic joint (a rack-and-pinion controlled by a motor-encoder assembly). 
{: style="text-align: justify"}

<br>
Here is a summary of the perception and control [logic](https://github.com/fuzzygreenblurs/3dof_control):
0. A calibration measurement is performed to determine the pixel-to-cm mapping from image plane to world coordinates
1. A monocular camera is used to perform background subtraction to determine the location of the target in the image coordinates
2. A transformation is then used to convert these coordinates to 3D world coordinates
3. An inverse kinematics operation is then used to determine the requisite revolute joint angles
4. Finally, an RPi GPIO control signal is dispatched to the servos and motor-encoder to position the marker at the target
{: style="text-align: justify"}

I also put together a couple of [demo videos](https://www.youtube.com/playlist?list=PLoytQ1zm8QQB1rKg45-0EHJ8OXCNs55xg) to showcase this approach during the process of building the system.
{: style="text-align: justify"}
<br>
