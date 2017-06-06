# CarND-Controls-MPC
Self-Driving Car Engineer Nanodegree Program

---

[//]: # (Image References)

[image1]: ./images/State_Error.png "STATE"
[image2]: ./images/Actuators.png "ACT"
[image3]: ./images/UpdateEq_Error.png "UPEQ"
[image4]: ./images/Latency.png "LATENCY"
[image5]: ./images/Transform.png "TRANSFORM"

# Write Up

## Kinematic Model
First we create a model of they system. This was provided in Lecture 18. 

There are 4 states as shown in the equation below.The goal will be to model how these states evolve through time.

![alt text][image1]

Next we identify 2 actuators, or inputs that change the states, shown in the equation below. These are the steering angle and throttle/brake. 

![alt text][image2]

The update equations are shown below in the equation below. These equations tell us how the state evolves with time given the value of other states, some geometry and the previous value of that particular state.

![alt text][image3]

## Setting MPC Timestep Length (N) and Elapsed Duration (dt)
As stated in Lesson 19 the prediction horizon, T is the duration over which future predictions are made. T is the product of N and dt. T = N * dt. N is the number of timesteps in the horizon. dt is the timestep between actuations.

N, dt and T are hyperparameters we must determine. From the Lesson 19 we recall some guidelines:
* T should be as large as possible, implying we should try to predict as far into the future as possible.
* dt should be as small as possible to make actuations as smooth as possible
* T should at most be a few seconds for a car, the environment doesnt change much beyond this
* N determines the number of variables optimized by MPC. Larger N imcreases computational load.

The approach I took was similar to what was suggested in Lesson 19:
Set N,dt and T to find a reasonable range for T and then tune dt and N. I observed the following in this process:
* dt < 0.05 made the steering much smoother. 
* Larger N > 20 seemed to be too large. The car seemed like it was trying to steer too slowly. It would often oscillate uncontrollably if N was too high.

After some trial and error I settled on N = 17, dt=0.023s and T = 0.391s. This in concert with the gains applied to the costs seemed to work best.

## Polynomial Fitting and MPC Preprocessing
The simulator returns x and y global position coordinates of the car as well as the waypoints. We need to translate these into the vehicle body coordinate system first before sending to the MPC solver as the kinematic equations are mechanized in that frame of reference.

![alt text][image5]

Once the waypoints of the car are in the vehicle body frame we apply the polyfit function with 3rd order. The polyfit function will fit a polynomial of 3rd order. 3rd order was selected because of the suggestion in Lesson 18 that they work best at fitting most roads.

## MPC
For the actual MPC itself, I started with the mpc_to_line example from Lesson 19. I started by adjusting N, dt and T. This did not seem to be enough to get the car driving smoothly. To improve this I used the suggestion in Lecture 19 to apply gains/cost modifiers for each variable we are attempting to minimize the cost. These "lambda" variables are hand tuned to the performance of the MPC. For example if I want the car to prioritize minimizing CTE, I increase the gain on CTE which increases the overall cost. These parameters were primarily hand tuned. The best gains I used are shown below:

- lambda_cte    	 = 10.0 
- lambda_epsi  	  = 50.0 
- lambda_v  		    = 1.0  
- lambda_thr      = 20.0 
- lambda_str      = 20.0 
- lambda_dif_thr  = 30.0  
- lambda_dif_str  = 1000.0

## MPC With Latency

The simulator artifically adds 100 ms of latency between command and actuator response in order to mimic real life actuator delay. To deal with the latency I used the suggestion in Lesson 19 to create a simple dynamic model of the system and propogated the state from the simulator for the duration of the latency. This model is shown in the equations below. This new state could then be used by the MPC solver as its new initial state.

![alt text][image4]

## Results
Final results are shown in the video below.

## Further Work
Further tuning could be done to improve performance. Only one speed was tested, other speeds should be tested to verify robustness of the controller. 

## Dependencies

* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets) == 0.14, but the master branch will probably work just fine
  * Follow the instructions in the [uWebSockets README](https://github.com/uWebSockets/uWebSockets/blob/master/README.md) to get setup for your platform. You can download the zip of the appropriate version from the [releases page](https://github.com/uWebSockets/uWebSockets/releases). Here's a link to the [v0.14 zip](https://github.com/uWebSockets/uWebSockets/archive/v0.14.0.zip).
  * If you have MacOS and have [Homebrew](https://brew.sh/) installed you can just run the ./install-mac.sh script to install this.
* Fortran Compiler
  * Mac: `brew install gcc` (might not be required)
  * Linux: `sudo apt-get install gfortran`. Additionall you have also have to install gcc and g++, `sudo apt-get install gcc g++`. Look in [this Dockerfile](https://github.com/udacity/CarND-MPC-Quizzes/blob/master/Dockerfile) for more info.
* [Ipopt](https://projects.coin-or.org/Ipopt)
  * Mac: `brew install ipopt`
  * Linux
    * You will need a version of Ipopt 3.12.1 or higher. The version available through `apt-get` is 3.11.x. If you can get that version to work great but if not there's a script `install_ipopt.sh` that will install Ipopt. You just need to download the source from the Ipopt [releases page](https://www.coin-or.org/download/source/Ipopt/) or the [Github releases](https://github.com/coin-or/Ipopt/releases) page.
    * Then call `install_ipopt.sh` with the source directory as the first argument, ex: `bash install_ipopt.sh Ipopt-3.12.1`. 
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [CppAD](https://www.coin-or.org/CppAD/)
  * Mac: `brew install cppad`
  * Linux `sudo apt-get install cppad` or equivalent.
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/CarND-MPC-Project/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions


1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

## Tips

1. It's recommended to test the MPC on basic examples to see if your implementation behaves as desired. One possible example
is the vehicle starting offset of a straight line (reference). If the MPC implementation is correct, after some number of timesteps
(not too many) it should find and track the reference line.
2. The `lake_track_waypoints.csv` file has the waypoints of the lake track. You could use this to fit polynomials and points and see of how well your model tracks curve. NOTE: This file might be not completely in sync with the simulator so your solution should NOT depend on it.
3. For visualization this C++ [matplotlib wrapper](https://github.com/lava/matplotlib-cpp) could be helpful.

## Editor Settings

We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style

Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric

Note: regardless of the changes you make, your project must be buildable using
cmake and make!

More information is only accessible by people who are already enrolled in Term 2
of CarND. If you are enrolled, see [the project page](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/f1820894-8322-4bb3-81aa-b26b3c6dcbaf/lessons/b1ff3be0-c904-438e-aad3-2b5379f0e0c3/concepts/1a2255a0-e23c-44cf-8d41-39b8a3c8264a)
for instructions and the project rubric.

## Hints!

* You don't have to follow this directory structure, but if you do, your work
  will span all of the .cpp files here. Keep an eye out for TODOs.

## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to we ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make./
