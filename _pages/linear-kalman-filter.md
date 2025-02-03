---
layout: page
title: Linear Kalman Filter
permalink: /concepts/linear-kalman-filter/
---

### High Level Context

The Kalman Filter (KF) is an algorithm used to estimate the state of a given system based some measured data. While this notion of "state" is specific to the system we care about, the KF workflow can be agnostically employed across a variety of applications: computer vision, guidance and navigation, industrial process control, etc. 

In this writeup, we will use it to accurately estimate the 2D position of a self-driving car in real-time, given noisy GPS sensor measurements. 

Note: The Kalman Filter follows much of the same approach of the [Recursive LSE workflow](/concepts/recursive-least-squares). For more intuition, follow along with that writeup first. 

### Workflow

The whole point of the Kalman Filter is to measure the state of the system in a "fuzzy" fashion. The term "fuzzy" is used here to mean that the state is represented in a probablistic fashion: it has a _mean_ and some _uncertainty_. By representing the state this way, we can account for inherent inaccuracies in any models we use to represent the system as well as the stochastic nature of noisy sensors.

This improved estimate of state can thereafter be used downstream as the feedback component of a high-level control loop. For example, the autopilot of an airplane might want to maintain some atitude, altitude and heading velocity. The output of a Kalman Filter in this application would be an improved estimate of the current attitude, altitude and heading velocity, based on a set of noisy/inaccurate proprioceptive and exteroceptive sensors. 

The inputs and outputs of the KF for our specific self-driving car example are discussed in greater depth in following sections. 

The Kalman Filter is a recursive algorithm that can be represented as a loop:

![Range of Uncertainty](/assets/images/recursive-least-squares/recursive_lse_cycle.png)
![Range of Uncertainty](/assets/images/recursive-least-squares/recursive_lse_cycle.png)

where: 
- $$\hat{X}_{k-1}$$: the estimated state at the end of the previous timestep
- $$y_k$$: incoming sensor measurement for the current timestep 
- $$\hat{X}_k$$:  newly estimated state at the end of the current timestep

Thus, the loop displayed on the right represents one such timestep in an endless infinite loop of timesteps.

Let's now explore the workflow and its components a bit further:

![KF Workflow](/assets/images/linear-kalman-filter/kf_workflow.png)

First, lets establish the nomenclature used in this diagram and all subsequent sections:

1. Variables:
    - $$X$$ represents the mean component of the state
    - $$P$$ represents the uncertainty component of the state
    - $$z$$ represents the mean component of the sensor measurement
    - $$R$$ represents the uncertainty component of the sensor measurement (sensor noise)


1. `estimated` vs `true` state notation:
    - the _hat_ notation is used to represent _estimated_ state: $$\hat{X}$$
    - $$X$$ alone represents the true state (which we cannot perfectly measure)


2. `a-priori` state estimate: 
    - _"using principles to decide the probable effects or result of something."_
    - **"predicted"** state estimate _before_ a new measurement
    - denoted with a superscript `-` such as $$\hat{X}^{-}$$

3. `a-posteriori` state estimate:
    - _"knowledge which proceeds from observations to deduce probable causes."_
    - **"updated"** or **"corrected"** state estimate _after_ a new measurement
    - denoted with a superscript `+` such as $$\hat{X}^{+}$$

4. Timestep notation:
    - a subscript $$_k$$ is used to denote the timestep
    - $$X_k$$ is the state in the current timestep
    - $$X_{k-1}$$ is the state in the previous timestep

Thus, if we consider the KF workflow diagram (for an arbitrary timestep $$k$$) more closely, we are performing the following steps:

1. Start with the `a-posteriori` estimated state $$\hat{X}^{+}_{k-1}$$ and uncertainty $$P^{+}_{k-1}$$ from the previous timestep $$k-1$$.

2. **Prediction Step:** Feed this into a model (a kinematics-based motion model in our example) that represents the expected dynamics of the system to generate the new `a-priori` (predicted) state $$\hat{X}^{-}_{k}$$ and uncertainty $$P^{-}_{k}$$ for the current timestep $$k$$. 

3. Ingest a new measurement $$z_k$$ (sensor reading) of the current state. Note that this measurement is noisy and therefore has an uncertainty $$R_k$$ associated with it.

4. **Correction Step:** The measurement model takes in the current measurement $$z_k$$ and uncertainty $$R_k$$ along with the `a-priori` state $$\hat{X}^{-}_{k}$$ and uncertainty $$P^{-}_{k}$$ and outputs the `a-posteriori` state estimate $$\hat{X}^{+}_{k}$$ and uncertainty $$P^{+}_{k}$$.

5. These `a-posteriori` components then become the new inputs to the KF for the next timestep cycle.

Note: for the purposes of the Linear Kalman Filter, the motion and measurement models must both be linear models. We will explore why this is important in subsequent sections.

### Desired State & Motion Model 

Now that we have discussed the high level workflow of the Kalman Filter, let us consider application-specific details related to tracking the position of our self-driving car. 

First, let's begin with the set of assumptions we make about this problem:
1. The vehicle is constrained to traverse 2D space only, represented here by the XY plane.
2. The vehicle generally moves at a constant speed longitudinally.
3. The vehicle could turn randomly: this is not a controlled action but needs to be accomodated in our estimation of the state. 

Next, let's consider what our desired state should be. For this application, we are only really interested in tracking the real-time position and velocity of our self-driving car with respect the inertial frame. 

Aside:

Recall the inertial/global frame can be defined as a frame whose origin is fixed at a specific point on the 2D plane. 

In contrast, there is also the relative/body frame whose origin is set to the centroid of the car (and therefore moves as the car moves). In addition, the orientation of its axes is relative to the longitudinal heading of the vehicle. For the purposes of our current problem, all our state variables will be measured with respect to an inertial frame. 

Thus, our state can be represented as a vector $$\bold{X} = [p_x p_y v_x v_y]^T$$, where each component is with respect to the global frame.

In the below diagram, $$v$$ is the vector along the longitudinal axis of the vehicle, representing its longitudinal velocity based with respect to the vehicle's relative frame. For our purposes, this vector can be decomposed into component $$v_x$$ and $$v_y$$ vectors, along the direction of the world frame's basis axes.

![Range of Uncertainty](/assets/images/linear-kalman-filter/motion_model.png)

To consider how the position and velocity of the vehicle evolve with time, we can apply these point-mass kinematics equations (assuming a constant acceleration component):












