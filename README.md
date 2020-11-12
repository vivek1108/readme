+ Licensed Materials - Property of IBM
+ “Restricted Materials of IBM”
+ 5765-R17
+ © Copyright IBM Corp. 2020 All Rights Reserved.
+ US Government Users Restricted Rights - Use, duplication or disclosure restricted by GSA ADP Schedule Contract with IBM Corp


**Description**
---

```
This example implements a possible instrumentation of a basic CFD simulation, run with OpenFOAM, with BOA.

The simulation models the flow around a cylinder, where the control variables are the flow's inlet velocity,
initial pressure, turbulent kinetic energy, dissipation rate, and wall velocity. The objective is to minimize
the turbulent kinetic energy at a point in the cylinder's wake (coordinates x=2, y=0=z).

```


**Input Values**
---


```
  boaServiceURL = input("Enter BOA services URL : ")
  userId = input("Enter your USER ID to run the BOA experiment : ")
  password = input("Enter password to run the BOA experiment : ")
  baseDir = input("Enter Directory that holds the Template folder :  ")

```

While running this Interface_function , It ask for user to input 4 values . Those values are :

1. BOA service URL .
2. UserId of the User trying to run the experiment
3. password of the account from which the experiment is going to be executed
4. Directory that holds the template folder


**Experiment configuration**
---

BOA uses a configuration JSON file or an equivalent Python dictionary to configure the optimization. This
topic describes the parameters used for configuration.

1. name

    + The name of your optimization experiment.

2. domain


      ```
      "domain": [
          {
            "name": "x1",
            "min": -2,
            "max": 2,
            "step": 0.01
      }, {
            "name": "x2",
            "min": -1,
            "max": 1,
            "step": 0.01
      }
      ]
      ```

    + The domain is the set of parameters that you want to search through to find the optimum parameters.
      For an engineering problem this might be the list of possible designs for a component, such as the
      engine of a car. For a chemical manufacturing problem, this could be the list of possible combinations
      of ingredients.
    + To define the domain as a grid, we specify the name of each parameter, a minimum value, a
      maximum value, and a step size.

3. model

    ```
    "model":{
      "gaussian_process": {
      "kernel_func": "Matern52",
      "scale_y": True,
      "scale_x": False,
      "noise_kernel": True,
      "use_scikit": True
      }
    }

    ```

    + Defines the surrogate model to use


4. optimization_type

    + Defines the type of optimization technique : min (for minimization ) max (for maximization )

5. initialization

    ```
    "optimization_type": "min",
       "initialization": {
         "type": "random",
         "random": {
           "no_samples": 1,
           "seed": None
         }
       }
    ```

    + Specifies how to initialize the optimizer. There are two options: initialization by random samples
      (random) or by uploading observations (observations). Random initialization randomly selects
      values from the domain, whereas the observation-based initialization allows you to specify a list of
      parameter values to initialize the optimizer.
      Example of specifying random initialization:

    + no_samples
        - Specifies the number of samples to use for initialization.
    + seed
        - Specifies whether to set the NumPy random seed for the initialization.

6. sampling_function

    ```
    "sampling_function": {
      "type": "expected_improvement",
      "epsilon": 0.03,
      "optimize_acq": False,
      "outlier": False,
      "bounds": None,
      "scale": False,
      "explain": {
        "feature_importance": True,
        "feature_interaction": ['PDP', 'H_statistic'],
        "features_idx": [0,1]
      }
    }

    ```

    + Specifies how the optimizer will sample from the domain
    + type
       - Specifies what type of acquisition function to use.
       - The acquisition function is core to how Bayesian optimization functions,
         and different acquisition functions will result in different optimizer
         behaviors.
       - It is typically advised to use one of the following acquisition functions:
         expected_improvement, adaptive_expected_improvement, probablity_improvement,
         or adaptive_probability_improvement
       - BOA optimizer also supports, epsilon_greedy, maximum_entropy and random_sampler sampler
         types but only when design variables domain is defined as Grid(Discrete variables).
     + epsilon
        - This variable controls the degree to which the optimizer will tend to
          exploit known 'good' areas of the domain (low epsilon), or favor exploring
          less well-known areas of the domain (high epsilon).
     + scale
        - Depending on version of BOA used, the scale parameter may not be required.
        - If required, this should always be set to False.
     + bounds
        - Contains the upper and lower bound for each parameter in the list.
     + explain
        - Defines the explainability features computed for BOA. If the explain field
          isn't used, BOA will run without explainability. Its parameters are :
        - feature_importance . Whether to compute the feature importance.
        - feature_interaction . A list of feature interactions to use, one or both of
          PDP and H_statistic can populate the list.
        - features_idx

**BOA Services**
---


1. Login API

    ```
      boaas = BOaaSClient(host=boaServiceURL)
      user = {"_id": userId , "password": password }
      user_login = boaas.login(user)
      user_token = user_login["logged_in"]["token"]

    ```
    
    + Set BOA instance
    + create UserId and password dictionary using data shared by User at run time
    + Use the BOA's login API
    + Fetch user token

2. Construct BOA experiment

    ```
    exp_user_object = { "_id": user["_id"], "token": user_token}
    experiment_res = boaas.create_experiment(exp_user_object, openFOAM_experiment)
    experiment_id = experiment_res["experiment"]["_id"]

    ```
    
    + Define experiment user object
    + Use the BOA's create_experiment in order to create an experiment .
        - API takes in the experiment user object created above and Experiment Configuration created above .
    + Fetch the experiment_id from the experiment created above .

3. Run BOA experiment

    ```
    boaas.run(experiment_id=experiment_id, user_token=user_token, func=objective_func, no_epochs=3, explain=True)
    best_observation = boaas.best_observation(experiment_id, user_token)
    print("best observation:")
    print(best_observation)
    boaas.stop_experiment(experiment_id=experiment_id, user_token=user_token)

    ```
    
    + Use the BOA's run API to execute the experiment .
       - it takes the Experiment_id , user token , objective_function , number of epochs
    + execute the best_observation API to get the best output .
       - for the number of epoch it returns the best value depending upon the optimization_type
    + prints the final output value .
    + Run the stop_experiment API to change the experiment status and stop the activity .


**objective_function**
---
An objective function is the output that you want to maximize or minimize. It is
what we will measure designs against to decide which option is best. The objective function can be
thought of as the goal of your generative design process.

```

def objective_func(x):
    """CFD flow optimisation
    This function takes in a vector feeded by the BOA for an epoch
    It has 4 sub process inside it and returns a float Y value/result of the simulation for that epoch
    PARAMS : x is a vector of 5 values
    x1 - FVEL Flow Velocity
    x2 - PRESS Pressure
    x3 - TKE Turbulent Kinetic energy
    x4 - TEPS Turbulent Epsilon
    x5 - U wall Velocity

    Takes in these 5 parameters and perform Preprocessing , set boundary consitons ,
    simulation process for the epoch and then postprocess the result for the epoch
    and return the result.
    """

    # Create simulation skeleton
    args=[str(el) for el in x]
    cmd=args
    args.insert(0, baseDir+"/preprocess")
    subprocess.call(cmd, cwd=baseDir)

    # Deform the geometry based on this function's arguments
    args=[str(el) for el in x]
    cmd=args
    args.insert(0, baseDir+"/setBCs")
    subprocess.call(cmd, cwd=baseDir)

    # Run an OpenFOAM simulation to compute the flow around the cylinder
    args=[str(el) for el in x]
    cmd=args
    args.insert(0, baseDir+"/simCFD")
    subprocess.call(cmd, cwd=baseDir)

    # Collect the output and pass it to this function
    args=[str(el) for el in x]
    cmd=args
    args.insert(0, baseDir+"/postprocess")
    stdout = subprocess.check_output(cmd, cwd=baseDir).decode('utf-8').strip()

    print("Epoch ran at : " + str(datetime.datetime.now()) + " with following Deformation parameters : " + str(args))
    print("Epoch result = " + str(stdout))

    return float(stdout)

```


BOA uses this function to perform 4 differnet process to prepare the data , populate with respective values ,
run simulations and prepare the output variable and return the optimized value.

+ Takes in 5 arguments (Flow Velocity, Pressure, Turbulent Kinetic energy, Turbulent Epsilon, wall Velocity)
+ these 5 arguments in the form of vector is shared by BOA during every epoch run
+ It returns a value that is taken as the putput.
+ This function internally executes 4 processess
+ preprocess  :     Prepare the data for every epoch by creating different folder for each epoch
+ setBCs      :     Sets up the boundary condition using the parameters value shared by BOA
+ simCFD      :     run the simulation process within that epoch folder
+ postprocess :     feteches the output from the output file and store in a variable
