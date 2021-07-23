# Artifacts for Mapping Control Theory and Software Engineering Properties

[Repo Content](#contents) | [Replicate Experiments](#repl-exp) | [From Simulink to Model Checking CPP](#sim-cpp) | [Contact Us](#contact)

This repository is an online Appendix for versioning the artifacts produced in the ongoing research effort for mapping control theory and software engineering properties.

<h2 id="contents">Contents</h2>

- ./img: contains images used in this ReadMe
- ./MATLAB: contains the matlab files necessary for embedded execution of the experiments.
- ./my_mpcACCsystem_grt_rtw: contains the cpp code automatically generated in simulink.
- ./myACC: contains the matlab + simulink model with a controller to the automatic cruise control problem.
- ./results: contains the report results for the scenarios executed in the paper.

<h2 id="repl-exp">Replicating our Experiments</h2>

### Pre-Requisites

- [DIVINE 4](https://divine.fi.muni.cz/) (version 4.4.2+493f57f71694)

### Configuration

First, you must configure the makefile (my_mpcACCsystem.mk).

- START_DIR: line 22
- DIVINE_PATH: line 35

<h3 id="scenarios">Scenarios</h3>

| ID             | v0 ego     | v0 lead    | x0 ego     | v0 lead    | Description|
| :------------- | :--------: | :--------: | :--------: | :--------: | :--------: |
| Case 1         | 10km/h     | 30km/h     | 10m        | 50m        | Ego is always at safe distance.|
| Case 2         | 20km/h     | 25km/h     | 3m         | 5m         | Ego recovers from unsafe distance.|
| Case 3         | 40km/h     | 15km/h     | 10m        | 20m        | Ego cannot recover from unsafe distance.|

The values for v_0 and x_0 can be changed in the file my_mpcACCsystem_grt_rtw/my_mpcACCsystem_data.cpp.

- v_0 ego: line 62
- v_0 lead: line 67
- x_0 ego: line 77
- x_0 lead: line 82

For each scenario, you must recompile the artifact using divine. We provide a makefile to help with this task.

```
cd my_mpcACCsystem_grt_rtw
make -f my_mpcACCsystem.mk divine-build 
```

If everything goes well, you should expect a new file my_mpcACCsystem.bc. Then, you can proceed and execute divine check.

```
make -f my_mpcACCsystem.mk divine-check
```

As a result, the check will generate a report, named after my_mpcACCsystem.report.(unique label).

<h2 id="sim-cpp">From Simulink model to Model Checking with DIVINE4</h2>

### Pre-Requisites

- MATLAB (R2021a).
- Simulink + Simulink Coder.
- Access to [Spot](https://spot.lrde.epita.fr/app/).
- [DIVINE 4](https://divine.fi.muni.cz/) (version 4.4.2+493f57f71694).

### Modeling the ACC solution in Simulink

1) Open the folder ./myACC in your Matlab and open the file my_acc_example.m.
   
2) Configure (initial values x0, v0, amin, etc) and run the file. This should open the Simulink view and plot graphs with the vehicles' behavior.
   
### Generating CPP code from Simulink Model

For this step, we followed the [simulink tutorial for C code generation](https://se.mathworks.com/help/dsp/c-code-generation.html).

3) Make sure to select C++ as the language and tick the Generate code only configurations.

- Solver: Type = Fixed-Step
- Hardware Implementation: Change it according to the Operating System (Device type).
- Code Generation: Language = C++; Generate Code Only = True;
- Interface: Nonreusable function.

4) Open the Simulink Code view and click Generate Code.

### Instrumenting the Generated CPP code for Model Checking with DIVINE4

Given a desired property &phi; and a model _M_, model checking decides whether _M_ ⊨ &phi; (_The model M satisfies the property &phi;_). Thus, for checking a property &phi; with DIVINE4, we need to setup the proeprty to be checked (5) and and the model (6). For this tutorial, we use the LTL property (&phi; ≡ <>[]safe) as an example.

5) Properties (&phi;) in DIVINE are encoded through büchi automata representation. Thus, we need to (5.1) translate our LTL properties to a büchi automaton, and (5.2) feed the automaton into DIVINE. 

    5.1) [Spot Online Tool](https://spot.lrde.epita.fr/app/) automatically translates formulae in temporal logic into büchi automata representation (either in HOA or in Neverclaim).  
    ![<>[]safe image](/img/safe_automaton.png "Generated automaton for <>[]safe")
    
    <sup><sub>*Components highlighted in green are accepting, in red are rejecting, and in grey are rejecting and useless</sub></sup>

    5.2) The easiest way to use the generated property in DIVINE is by implementing the automaton in C code. We used the Neverclaim representation to guide our implementation.
    ```cpp
    switch (current_state) {
        case 0:
            if (!AP[safe]) {return 0;}
            if (AP[safe])  {return __vm_choose( 2 ) ? 0 : 1}
        case 1:
            __vm_ctl_flag( 0, _VM_CF_Accepting );
            if (AP[safe]) {return 1;}
            if (!AP[safe]) {return 2;}
        case 2:
            return 2;   
    }
    ```
    
    <sup><sub>*We use macros from DIVINE to support the coding. The ```__vm_ctl_flag``` to mark an accepting states or transitions. We use the ```__vm_choose``` to emulate non-determinism</sub></sup>

6) The model (_M_) is the generated Simulink code. Since the model constraints will serve as guards to the transitions in the implemented automata (e.g., ```AP[safe]``` conditions in our previous snippet) we need to make such guards explicit in the model. To expose ```safe``` we have implemented it as an atomic proposition (AP) in two files:
   
    6.1) Declare them here: [my_mpcACCsystem.h](/my_mpcACCsystem_grt_rtw/my_mpcACCsystem.h) line 42).

    ```cpp
    enum APs {safe};
    ```

    6.2) Implement them here: [my_mpcACCsystem.cpp](/my_mpcACCsystem_grt_rtw/my_mpcACCsystem.cpp) lines 195, 853.

    ```cpp
    AP[safe] = (my_mpcACCsystem_B.d_rel - my_mpcACCsystem_B.safe_distance) > 0.05 * my_mpcACCsystem_B.safe_distance;
    ```

7) Finally, property and model need to be integrated before we compile and check the code with DIVINE. In our case, this is done in [main.cpp](/my_mpcACCsystem_grt_rtw/main.cpp) in two steps.

    7.1) Integrate the property (büchi automata) into the code. This can be done by implementing a ```next(state, AP)``` function
    (See [```int next(state, AP)```](/my_mpcACCsystem_grt_rtw/main.cpp), lines 384 - 423). 

    7.2) Then, change the lifecycle execution of the simulink model to alternate between computing transitions in the property and in the model. The following code snippet details the insertions in the execution (See [int main(argc, *argv)](/my_mpcACCsystem_grt_rtw/main.cpp), lines 430-499). 

    ```cpp
    int main (argc, *argv){
       ...
       while(true){
           __dios_reschedule();
           ...
           rt_OneStep(MODEL_INSTANCE);
           new_state = next(current_state, MODEL_INSTANCE.AP);
           ...
       }
    }
    ``` 

    <sup><sub>*The ```dios_reschedule()``` command is required by DIVINE to indicate a new cycle. The ```rt_OneStep(MODEL_INSTANCE);``` ticks the model once, updating the guards value. The ```next(state, AP)``` ticks the automata once to provoke a transition.</sub></sup>


### Executing DIVINE for CT properties checking

8) According to [DIVINE instructions](https://divine.fi.muni.cz/manual.html#model-checking-c-and-c-code-via-llvm-bitcode), one needs to compile and then check the cpp code. We provide the commands ```divine-build``` and ```divine-check``` in a makefile (./my_mpcACCsystem_grt_rtw/my_mpcACCsystem_grt_rtw.mk).
   
    ```
    make -f my_mpcACCsystem.mk divine-build
    make -f my_mpcACCsystem.mk divine-check
    ```

    <sup><sub>*Follow along the [scenario execution step](#scenarios) for a detailed tutorial on how to execute the scenarios.</sub></sup>

<h2 id="contact">Contact</h2>

Feel free to submit issues.

For questions, discussions, or suggestions, don't hesitate to contact us (;

Ricardo Caldas (ricardo.caldas@chalmers.se)