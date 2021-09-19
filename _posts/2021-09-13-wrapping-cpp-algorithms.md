---
layout: post
title:  "Wrap C++ algorithms in Python for pre-production evaluation and experimentation."
---

<!-- # Wrapping C++ code in Python for algorithm development & evaluation -->

## TLDR;

This article describes the benefits of calling production-focused low-level (C/C++) code in Python for functional evaluation and analysis. Assessment tools developed previously during the PoC stage can be reused. Furthermore, complexity for experimentation with various internal parameters is reduced, which can reduce costs and expensive debugging before the final deployment. In the end of the article the key benefits and limitations are presented.

## Motivation

Often, algorithms implemented in the automotive or other industrial domains are developed using the [model-based design](https://en.wikipedia.org/wiki/Model-based_design) (MbD) idiom [1]. Roughly the workflow involves:

1. Definition of *computation steps in a high level* (or even visual) language. E.g. `compute_mean(signal)` or `value = read_sensor()` etc.
2. *Simulation and experimentation* (with virtual/artificial data or system behaviour). Creating visualizations or performance assessments of the algorithm.
3. Finally, *code generation* for the target environment (e.g. C for embedded devices or C++ for a specialized server hardware).
4. *Testing* of the generated code on the target environment

Thus approach allows fast (rapid) prototyping **and** high quality code, since the generators are often well tested.

Companies like [Mathworks](https://de.mathworks.com/) provide specialized tools across the whole development chain. Often those tools are expensive for small projects so a smaller solution may be sutable.

<details> 
  <summary>FYI: See how tricky MbD can look like for large projects.</summary>
A lot of modelling, testing and project management tools are used in context of a complex industry application:

<img src="https://www.embitel.com/wp-content/uploads/Model-Based-Development-V-process.png"/>

Image credits by Embitel, 2019.
</details>

<br/>

However, for small projects (with a smaller budget) a developer may create a Proof of Concept (PoC) for the algorithm in a scripting language of her choice (like Python or Scala) before turning to C/C++. In addition to that, he or she calculates some quality metrics like mean squared error (MSE) or accuracy to show (off) how well the logic is performing.

The "architecture" on the development machine would look similar to the following diagram:

<img src="{{ site.baseurl }}/assets/blog/poc-arch.png" alt="poc-arch" width="90%"/>

After the stakeholders are satisfied, we may refactor the codebase or -- if the target platform (e.g. embedded device) requires a device specific implementation -- rewrite the routine to the target language. Sometimes we have to do this also for execution speed and scale.

But how to make sure that the final implementation is functionally as good as the PoC? A functional evaluation involves a pure evaluation focusing on the algorithmic part ignoring communication issues and other intergration effects.

A naive assessment approach would involve compiling generated code, transmit the build to the target device, generate some (artificial) input data, run the executable and simultaniously record the algorithms outputs:

<img src="{{ site.baseurl }}/assets/blog/target-arch.png" alt="target-arch" width="90%"/>

Even if we use an emulator for the target device or simply compile a regular executable on our develepoment machine, we always have to introduce some overhead "measuring" the algorithm's internals and its output. Of course, in return for the additional complexity we will be able to evaluate the real world performance including communication delays and performance fall-offs. Also we will be sure that system libraries on the target device are working as intended.

Let's summarize the key benefits and downsides of the approach above:

| Advantages | Drawbacks |
|------------|----------|
| Execution on the target device  | Data feed to the target device has to be implemented |
| Impact of execution speed, data transmission and network errors can be evaluated | Recording data introduces more complexity (and/or special debugging hardware)|

## Proposed solution

For the functional assessment following architecture is proposed. We wrap the low-level language (C++) implementation into a shared library and call it from the high level language (Python). We simulate time in a loop and record the results to a container (like `list()`) or save to disk. Finally we evaluate the results in the same way we did it in the PoC stage.

<img src="{{ site.baseurl }}/assets/blog/proposed-arch.png" alt="proposed-arch" width="90%"/>

Addionally to addressing the drawbacks from the last section we can achieve the following easily:

- change the internal parameters within the loop
- apply noise to input data
- observe and record internal state variables like buffers or counters

In the next section we will take a look at the basic skeleton required for the present approach.

## Software architecture

### C++

First we need an implementation in the low-level language. You will probably have the required methods like (getters and setters), `step()` or `run()` anyway, so the changes are expected to be small.

```cpp
// customalgo.hpp

#include <vector>

class CustomAlgo {
    public:
        CustomAlgo(float param);

        // the algorithm main routine for a single timestep
        float step(float input);

        void set_param(float p);
        void reset();

        // interface to the internals
        std::vector<float> get_state();

    // attributes and methods ...
    private:
        float _param;
        std::vector<float> _state;
};
```
### Python interface (pybind11)

In order to create a Python wrapper for a `C++` object you would do the following. Take a look at the official [pybind11](https://pybind11.readthedocs.io/en/stable/index.html) documentation.


```cpp
#include <pybind11/pybind11.h>
#include "customalgo.hpp"

namespace py = pybind11;

PYBIND11_MODULE(customalgo, m) {
    py::class_<CustomAlgo>(m, "CustomAlgo")
        .def(py::init<const float &>())  // constructor
        .def("step", &CustomAlgo::step)
        .def("set_param", &CustomAlgo::set_param)
        .def("get_state", &CustomAlgo::get_state);
}
```

After a successful compilation you have a `.so` (Unix) or `.dll` (Windows) which you can import in any Python script.

### Python

On the Python side you can do "whatever you want". You may call `step()` methon in a loop and record the results. You may instantiate the algorithm multiple times and run with same data and several parameters, parametrize it on the fly etc ...

```python
import random

# shared lib
import customalgo

# your evaluation code (e.g. from PoC)
from eval import evaluate

algo = customalgo.CustomAlgo(42.42)

# for storing outputs
results = list()

# for storing internal states
states = list()

# the loop simulates time
for t in range(10):
    inp = random.randn()  # example input
    
    # try out a parameter change within the loop
    if t == 5:
        algo.set_param(1234)

    # collect internal states for detailed analysis
    states.append(algo.get_state())

    res = algo.step(inp)  # run a single step
    results.append(res)

# evaluate and analyse
evaluate(results)
detailed_analysis(states, results)
```
## Key Benefits

Let's summarize the key benefits of making the logic accessible in a high level language like Python:

- Avoid data feeding and recording efforts for the target environment or hardware
- Evaluate your algorithm with various inputs and parameters without additional C++ code for data feeding and result conversion.
- Simulation, analysis and evaluation are directly in Python, libraries like `scikit-learn` for evaluation or `mlflow` for experiment tracking can be used effortlessly.
- Compare your PoC with the final implementation easily (even in the same `for`-loop)

## Limitations

Please note that the proposed approach has also some limitations which you have to be aware of, before starting intergrating the workflow in your project:

- Use it for functional validation only. It is not a replacement for a full system test.
- Target device specific routines relying on embedded circuits, e.g. ASICs for audio processing or TPUs for neural network inference. Those have to be reimplemented (or called) in C/C++ or in Python.
- Similar to the statement above, in case you have target device specific implementation of libraries you use, first check if they are reliable and behave like those on your development machine.

## You want more?

If you are interested in the concepts and challenges of algorithm development I recommend following resources:

- An [article](https://serge-sans-paille.github.io/pythran-stories/pythran-as-a-bridge-between-fast-prototyping-and-code-deployment.html) of an audio processing engineer bridging the gap between Python and C++ code with `pythran`. The library is not very mature but the charming idea is to skip the C++ coding entirely.
- Take a look how [Mathworks pitches](https://de.mathworks.com/solutions/model-based-design.html) model based design
- A [ROS1 tutorial](http://wiki.ros.org/ROS/Tutorials/Using%20a%20C%2B%2B%20class%20in%20Python) on how to feed a C++ node implementation from Python.

## References

[1] https://en.wikipedia.org/wiki/Model-based_design

[2] [pybind11 — Seamless operability between C++11 and Python](https://pybind11.readthedocs.io/en/stable/index.html)

