---
title: "KPI Framework"
date: 2022-03-08T12:00:00+01:00
draft: true
math: true
---

## Introduction

Evaluating complex systems or algorithms is hard. Especially if you are dealing with dynamic behaviour over time, the requirements to the evaluation codebase grow with each new insight. First you want to compute just a single number describing algorithm's performance in one scalar (like mean squared error - *MSE*), but over the projects lifetime your expecations to the evaluation framework grow: you also want to generate static visualizations, generate reports over a time range, look at some specific situations or simply render both the algorithms outputs and the corresponding ground-truth in a video file.

In this post I will describe a software archecture which proved one's worth across multiple projects I was involved in. The focus of this architecture is the evaluation of are dynamical algorithms (or systems), which commonly deployed into the "real" physical world (such as robots or sensors) and run there for a specific time duration. 

## Basics

For the sake of better unterstanding, I'll introduce an artificial setting which helps to internalize the proposed concept. Let's stick with the logistics domain and focus on a mobile robot picking and delivering small boxes in a production site. This warehouse is also filled with employees, pallet trucks and other moving objects which are to detect and to avoid.

![audi-smart-factory](audi-smart-factory.jpg)

### Descrete systems

In particular, we focus on evaluating *discrete* systems, which output an action or prediction at particular points in time (timesteps). Moreover we will assume that we deal with *equidistant* outputs, means that the outputs are generated in a fixed output rate. This assumption often does not hold in real-world implementations, where the output generation time depends on system's load, other processes and the state of the environment. Thus, it is often needed to *resample* or *interpolate* the signal to equidistant timesteps.

In addition, for each timestep, there is ground-truth data, representing the desired behaviour of the system at that point in time. For a mobile robot in logistics it can be a safe path, for an aerial surveillance system it can be the collection of ground truth objects on the ground, which have to be tracked.

The system's output, the resampled version and the corresponding ground truth can be visualized in a timeline:

![discrete-timeseries](discrete-timeseries.png)

### Evaluation schemes

In this section a collection of various metrics and visualizations is outlined to motivate the need for an extensible, application-independent evaluation tooling.

#### Example 1: Localization

In the first example, the system's output can be its perceived position which has to be compared with it's actual position (with heading):

$$
y_1(t) = \begin{bmatrix} x_1(t) \\
    x_2(t) \\
    \varphi(t)
\end{bmatrix} 
= \begin{bmatrix} \mathbf x(t) \\
    \varphi(t)
\end{bmatrix}
$$

This assessment can be accomplished with a *mean euclidean distance*  $e(t) = ||\mathbf{\hat x}(t) - \mathbf{x}(t)||_2$ over evaluation duration $T$:

$$
MED = \frac{1}{T}\sum_{t=1}^T e(t)
$$

If you are interested in other localization metrics in ADAS context, please refer to [[2]](https://doi.org/10.3390/s21175855), where more sophisticated metrics (also considering bounding boxes) are defined.

#### Example 2: Object tracking

Another interesting assessment can be whether the system is detecting all objects (e.g. humans, pallets) within its own perception area:

$$
y_2(t) = \mathcal{O}_t = \{o_1, o_2, \dots, o_n \}_t
$$

Here, metrics like *multiple object tracking precision* (MOTP) or the *multiple object tracking accuracy* (MOTA) from [1] can be employed.

![motp](motp.png)

Small circles represent the actual objects (ground-truth) the large ones are the objects detected by the system (also called *object hypotheses*). MOTP considers the number of misdetections, false positives etc. over time. The MOTA metric computes the mean length of black arrows (i.e. the distances between detected and actual objects).

In addition to numeric metrics, for a deeper scene understanding you probably want to visualize the object bounding boxes (red: estimated, blue: ground-truth) and sensor data for a particular timestep, similar to the image below[3]:

![kitti-example](kitti-example.png)

### Summary

So far, many possible assessment methods and visualizations were introduced. You see, that on the first glance they are all different and share little between each other. For an evaluation environment those examples shall motivate a need for a generic architecture which is easy to extend and easy to undestand. This will be done in the next section.

## Architecture

- erkennen, dass es t und T basierte artefakte gibt
## Summary

## References

[1] Bernardin, K., Stiefelhagen, R. *Evaluating Multiple Object Tracking Performance: The CLEAR MOT Metrics.* EURASIP Journal on Image and Video Processing (2008)

[2] Rehrl, K.; Gröchenig, S. *Evaluating Localization Accuracy of Automated Driving Systems.* Sensors 2021, 21, 5855.

[3] Yang, Bin & Luo, Wenjie & Urtasun, Raquel. (2019). *PIXOR: Real-time 3D Object Detection from Point Clouds.*