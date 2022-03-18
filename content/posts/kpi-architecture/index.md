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

For the sake of better unterstanding, I'll introduce an artificial setting which helps to internalize the proposed concept. Let's stick with the logistics domain and focus on a mobile robot picking and delivering small boxes in a production site. This warehouse is also filled with employees, pallet trucks and other moving objects.

![audi-smart-factory](audi-smart-factory.jpg)

### Descrete systems

In particular, we focus on evaluating *discrete* systems, which output an action or prediction at particular points in time (timesteps). Moreover we will assume that we deal with *equidistant* outputs, means that the outputs are generated in a fixed output rate. This assumption often does not hold in real-world implementations, where the output generation time depends on system's load, other processes and the state of the environment. Thus, it is often needed to *resample* or *interpolate* the signal to equidistant timesteps.

In addition, for each timestep, there is ground-truth data, representing the desired behaviour of the system at that point in time. For a mobile robot in logistics it can be a safe path, for an aerial surveillance system it can be the collection of ground truth objects on the ground, which have to be tracked.

The system's output, the resampled version and the corresponding ground truth can be visualized in a timeline:

![discrete-timeseries](discrete-timeseries.png)

### Data format examples

In the initial example, the system's output can be it's perceived position which has to be compared with it's actual position.


Another interesting assessment can be whether the system is detecting all humans in it's perception field.

### Evaluation metrics


## Architecture

## Summary