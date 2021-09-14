---
layout: post
title:  "DRAFT: Wrapping C++ in Python for algorithm development & evaluation"
---

<!-- # Wrapping C++ code in Python for algorithm development & evaluation -->

## TLDR;

This article describes the the workflow benefits of wrapping C/C++ (or other languages) code to Python for algorithm development and evaluation. In particular, robotics-related implementations for ROS can heavily benefit from this approach.

## Motivation

A lot of algorithms implemented in the automotive or other industrial domains are implemented using the [model-based design](https://en.wikipedia.org/wiki/Model-based_design) approach [1]. Roughly this workflow involves:

1. Definition of processing steps in a high level (or even visual) language
2. Simulation
3. Finally, a code generation for the target language (e.g. C/C++ or Assembly)
4. Testing of the generated code

Companies like [Mathworks](https://de.mathworks.com/) provide commercial tools across the whole development chain.

<details> 
  <summary>FYI: See how complex it can look like for large projects.</summary>
A lot of modelling, testing and project management tools are used in context of a complex industry application:

<img src="https://www.embitel.com/wp-content/uploads/Model-Based-Development-V-process.png"/>

Image credits by Embitel, 2019.
</details>

<br/>

However, for smaller projects (with a smaller budget) a developer may create a Proof of Concept (PoC) for the algorithm in a scripting language of her choice (like Python or Scala). In addition to that, he or she calculates some quality metrics like mean squared error (MSE) or accuracy to show how well the logic is performing.

The "architecture" on the development machine would look similar to the following diagram:

<img src="assets/blog/poc-arch.png" alt="me" width="500"/>

After the stakeholders are satisfied, we may refactor the codebase or -- if the target platform (e.g. embedded device) requires a device specific implementation -- rewrite the routine to the target language. Sometimes we have to do this for execution speed and scale.

But how to make sure that the target implementation is as good as the PoC?

A naive approach would involve compiling generated code, transmit the build to the target device, exectute it and simultaniously record the algorithms outputs.

## Software architecture

### C++

```cpp
// myalgorithm.hpp

#include <vector>

class MyAlgorithm {
    public:
        MyAlgorithm(float param);

        // the algorithm main routine
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

### Interface (pybind11)

```cpp
#include <pybind11/pybind11.h>
#include "myalgorithm.hpp"

namespace py = pybind11;

PYBIND11_MODULE(myalgorithm, m) {
    py::class_<MyAlgorithm>(m, "MyAlgorithm")
        .def(py::init<const float &>())  // constructor
        .def("step", &MyAlgorithm::step)
        .def("set_param", &MyAlgorithm::set_param)
        .def("step", &MyAlgorithm::step);
}
```

### Python

```python
import random

import myalgorithm
from eval import evaluate  # your evaluation routine

algo = myalgorithm.MyAlgorithm(42.42)

results = list()

# the loop simulates time
for t in range(10):
    inp = random.randn()  # example input
    res = algo.step(inp)  # run a single step
    results.append(res)

accuracy = evaluate(results)
```

## Key Benefits

- Avoid data recording overhead on the target hardware
- Try your algorithm with various inputs and parameters without additional C++ code for data feeding and result conversion.
- Simulation and evaluation directly in Python, libraries like `scikit-learn` for evaluation or `mlflow` for experiment tracking can be used directly.
- Compare your PoC with the final implementation

## Limitations

Please note that the proposed approach has also some limitations which you have to be aware of before starting intergrating the workflow in your project:

- Target device specific routines using embedded circuits, e.g. TPU units or ASICs for audio processing or neural network inference. Those have to be reimplemented (or called) in C/C++ or in Python.
- Real time applications, simulation time loop has to be realistic!

## References

[1] https://en.wikipedia.org/wiki/Model-based_design

