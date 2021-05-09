# Artifacts for Mapping Control Theory and Software Engineering Properties

[Repo Content](#contents) | [Replicate Experiments](#repl-exp) | [From Simulink to Model Checking CPP](#sim-cpp) | [Contact Us](#contact)

This repository is an online Appendix for versioning the artifacts produced in the ongoing research effort for mapping control theory and software engineering properties.

<h2 id="contents">Contents</h2>

- ./img: contains images used in this ReadMe.
- ./my_mpcACCsystem_grt_rtw: contains the cpp code automatically generated in simulink.
- ./myACC: contains the matlab + simulink model with a controller to the automatic cruise control problem.
- ./results: contains the report results for the scenarios executed in the paper.

<h2 id="repl-exp">Replicating our Experiments</h2>

### Pre-Requisites

- MATLAB (R2021a)
- [DIVINE 4](https://divine.fi.muni.cz/) (version 4.4.2+493f57f71694)

### Configuration

You must configure the makefile (my_mpcACCsystem.mk) to correctly run the experiments in your machine.

- START_DIR: line 27
- DIVINE_PATH: line 34

### Scenarios

![](img/scenarios.png?raw=true)

The values for v_0 and x_0 can be changed in the file my_mpcACCsystem_grt_rtw/my_mpcACCsystem_data.cpp.

- v_0 ego: line 67
- v_0 lead: line 72
- x_0 ego: line 77
- x_0 lead: line 82

For each scenario you must recompile the artifact using divine. We provide a makefile to help with this task.

```
cd my_mpcACCsystem_grt_rtw
make -f my_mpcACCsystem.mk divine-build 
```

If everything goes well, you should expect a new file my_mpcACCsystem.bc. Then, you can proceed and execute divine check.

```
make -f my_mpcACCsystem.mk check
```

As a result, the check will generate a report, named after my_mpcACCsystem.report.(unique label).

<h2 id="sim-cpp">From Simulink model to Model Checking with DIVINE4</h2>

### Pre-Requisites

- MATLAB (R2021a) with License.
- Simulink + Simulink Coder.
- Access to [Spot](https://spot.lrde.epita.fr/app/) 
- [DIVINE 4](https://divine.fi.muni.cz/) (version 4.4.2+493f57f71694)

### ACC Simulink Model

1) Open the folder ./myACC in your Matlab and open the file my_acc_example.m.
   
2) Configure (initial values x0, v0, amin, etc) and run the file. This should open the Simulink view and plot graphs with the vehicles behavior.
   
### Generating CPP code

For this step, we followed the [simulink tutorial for C code generation](https://se.mathworks.com/help/dsp/c-code-generation.html).

3) Make sure to select C++ as the language and tick the Generate code only configurations.

![](img/code_config.png?raw=true)

4) Open the Simulink Code view and click Generate Code.

### Instrumenting the Generated CPP code

5) According to [DIVINE instructions](https://divine.fi.muni.cz/manual.html#model-checking-c-and-c-code-via-llvm-bitcode), one needs to compile and then check the cpp code. You can use our provided makefile (./my_mpcACCsystem_grt_rtw/my_mpcACCsystem_grt_rtw.mk).
   
6) To have control of the generated code execution, you need to get the main function from the MATLAB folder and modify it. We suggest you to use our provided main (./my_mpcACCsystem_grt_rtw/main.cpp).
   
### Executing DIVINE for CT properties checking

7) Create the atomic propostions to be checked (see [my_mpcACCsystem.cpp](/my_mpcACCsystem_grt_rtw/my_mpcACCsystem.cpp) lines 195, 853 and [my_mpcACCsystem.h](/my_mpcACCsystem_grt_rtw/my_mpcACCsystem.h) line 42).
      
8) Open [Spot](https://spot.lrde.epita.fr/app/), type the LTL property to be checked and implement the generated Büchi Automata (see next function in [main.cpp](/my_mpcACCsystem_grt_rtw/main.cpp)).   

<h2 id="contents">Contact</h2>

For questions, discussions, or suggestions don't hesitate, contact me!

Ricardo Caldas (ricardo.caldas@chalmers.se)