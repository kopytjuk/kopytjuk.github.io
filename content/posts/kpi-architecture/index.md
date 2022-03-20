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

Another interesting assessment can be whether the system is detecting all objects (e.g. humans, pallets) within its own perception area. The system's output is a list of detected objects, i.e.:

$$
y_2(t) = \mathcal{O}_t = \lbrace o_1, o_2, \dots, o_n \rbrace_t
$$

Each object in the list is described by its unique ID, coordinates and heading, i.e. $o_i = [\text{id}, \mathbf x, \varphi]$.

For the object tracking use-case, metrics like *multiple object tracking precision* (MOTP) or the *multiple object tracking accuracy* (MOTA) from [1] can be employed.

![motp](motp.png)

Small circles represent the actual objects (ground-truth) the large ones are the objects detected by the system (also called *object hypotheses*). MOTP considers the number of misdetections, false positives etc. over time. The MOTA metric computes the mean length of black arrows (i.e. the distances between detected and actual objects).

In addition to numeric metrics, for a deeper scene understanding you probably want to visualize the object bounding boxes (red: estimated, blue: ground-truth) and sensor data for a particular timestep, similar to the image below[3]:

![kitti-example](kitti-example.png)

### Summary

So far, many possible assessment methods and visualizations were introduced. On the first glance they are all different and share little between each other. For an evaluation environment the variety of possible artifacts shall motivate a need for a generic architecture which is easy to understand and extend. In the next session we will try to to reach this goal.

## Architecture

First, we will summarize all possible plots, metrics, visualizations and derivates derived from data as *artifacts*. Further we will group them into two categories:

- *Timestep artifacts* can be generated from data available at a particular timestep $t$. For example, to compute the distance $e(t)$ between perceived position and it's actual position we only need the data from $t$. Another example is the previous image - you only need sensor data and the objects from timestep $t$. Artifacts which can be created (computed) iteratively would fall in that category - an example is the sum of all distances or writing into a video buffer.
- *Timeseries artifacts* can only be generated from multiple timesteps. For example, in order to compute the mean distance (MED) you need the distances $e(t)$ from all $t$s. Animations which involve multiple frames also involve data from multiple timesteps. For the sake of simplicity, I will focus on artifacts which involve all timesteps, i.e. a data list of length $T$.

Further we will generalize both the output data from the system/algorithm under test and the ground-truth data as `EvaluationData`. Instances of this class have a method to retrieve the data from a particular timestep $t$.

The evaluation loop iterates over all timesteps, retrieves data from available data sources and generates timeseries-artifacts. Additionally, for the second artifact group the data has to be collected - you can think of a recording tape. After all timesteps are processed, the timeseries artifacts are computed.

![sequence-diagram](sequence-diagram.png)

Note that `EvaluationData` collection stands for the available data sources, e.g. algorithm and ground-truth in the previous examples.

As you can see, the methods `sample(t)`, `generate()` and `record()` are the core of the architecture. The following class diagram shows multiple implementations for `EvaluationData`, `TimestepArtifact` and `TimeseriesArtifact`.

![class-diagram](class-diagram.png)

You can extend your codebase with new metrics and plots and pass the list of desired artifacts to the evaluation loop without ever touching loop's codebase itself. The majority of code changes happen in the implementations of the artifacts and data sources.

Moreover you can replace the algorithm under test by providing a different implementation as it was done with `AlgorithmOutputImpl1` and `AlgorithmOutputImpl2` - you can even have them both at the same time for comparison. The only thing to extend, is to create a family of artifacts supporting two algorithm outputs.

Note that you have full access to the whole data after feedng the `Tape` instance via `record()` , which simply logs the data to memory or disk.

## Summary

The proposed architecture with a main loop over evaluation timesteps, a collection of data sources and two artifact groups allows easy replacement, straightforward extensibility and modularity by introducing clean interfaces to retreive and pass data.

You want to use this architecture template if following is true:

- you have a discrete and dynamic system under test
- you have different algorithm implementation for the same problem
- you have a large number of evaluation artifacts and they surely will grow over time
- you work in Python or similar language supporting *duck typing*.

## Possible extensions

Probably you already recognized some insufficiences of the architecture. Those have to be addressed to fully uncover the potental:

- *Timeseries artifacts* for variable sized (<T) lists.
- Configuration management for data sources and artifact collections. Serializing data source and artifact instances allows reproduce the same plots and metrics at a different point of time.
- Manage passing arguments to `generate`. Havin multiple data sources allows a combination of different data constellations. Currently the `generate` signature solely accepts a list of objects equal to the number of data sources.

## References

[1] Bernardin, K., Stiefelhagen, R. *Evaluating Multiple Object Tracking Performance: The CLEAR MOT Metrics.* EURASIP Journal on Image and Video Processing (2008)

[2] Rehrl, K.; Gröchenig, S. *Evaluating Localization Accuracy of Automated Driving Systems.* Sensors 2021, 21, 5855.

[3] Yang, Bin & Luo, Wenjie & Urtasun, Raquel. (2019). *PIXOR: Real-time 3D Object Detection from Point Clouds.*