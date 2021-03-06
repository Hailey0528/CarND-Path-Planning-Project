# CarND-Path-Planning-Project
In this project a path planning algorithm is implemented to drive the vehicle on the highway. It includes three parts: prediction, behavior planning and path planning. In the prediction part, the behavior of other vehicles on the road from sensor fusion will be predicted. Then in the behavior planning the ego-vehicle's behavior is planned. in this simplified highway scenario, the vehicle can only execute three hehaviors: keep in the current lane, change to the left lane and change to the right lane. And of course we need to brake, if the ego-vehicle is near to the other vehicle, and accelerate to the maximal allowed velocity, if their distance is not far. At last, except the waypoints from the previous trajectory, the rest points in 50 points are calculated.

### Prediction
The basic rule is that the distance of ego-vehicle to other vehicle should be greater than 30 m. So we need to know if we follow one of the movements (change to the left lane, change to the right lane, keep in the current lane) after finish executing the current trajectory, this requirement will be violated or not. For this purpose, we have created three booleans.

                bool car_straight = false;
                bool car_left = false;
                bool car_right = false;

For every vehicle the sensor fusion has observed, the lane number is obtained.

                //decide the maneuver the car can do 
                for (int i = 0; i < sensor_fusion.size(); i++){
                  float d = sensor_fusion[i][6];
                  int car_lane = -1;
                  if (d > 0 && d < 4){
                    car_lane = 0;
                  }else if (d > 4 && d < 8){
                     car_lane = 1;
                  }else if (d > 8 && d < 12){
                     car_lane = 2;
                  }

Then we assume that the vehicle will drive with the constant speed during the whole previous trajectory. The speed will be calculated with the checked vehicle's current velocity in x and y direction. And the s in Frenet coordinate after executing the previous trajectory will be calculated.

                  //calculate the vehicle's speed
                  double vx = sensor_fusion[i][3];
                  double vy = sensor_fusion[i][4];
                  double check_speed = sqrt(vx*vx + vy*vy);
                  double check_car_s = sensor_fusion[i][5];

                  //estimate the vehicle's postion after executing the previous trajectory
                  check_car_s += (double)(prev_size*0.02*check_speed);
                 
Now let's check if it violates the 30 m distance rule in each lane.

                if (car_lane == lane)
                  {
                    car_straight |= check_car_s - car_s > 0 && check_car_s - car_s < 30;
                  }
                  else if(car_lane - lane == -1)
                  {
                    car_left |= check_car_s - car_s < 30 && check_car_s - car_s > -30;
                  }
                  else if(car_lane - lane == 1)
                  {
                    car_right |= check_car_s - car_s < 30 && check_car_s - car_s > -30;
                  }

### Behavior Planning
In this part we need to decide which decision the vehicle will make. So we need to solve two problems: in which lane will the ego-vehicle be, and what is the reference velocity for the ego-vehicle. As in a simplified highway situatation, the vehicle can only choose one of the decisions for lane: keep the current lane, change to the left lane and change to the right lane. If it violates the 30 m distance rule in the current lane, then we consider if we can change the lane. Left lane has priority compare with right lane. If it doesn't violate the 30 m rule, then we change the lane. if changing lane is not possible, then we brake the vehicle with the allowed maximal acceleration. If in the current lane there is no potential danger, then we keep in the current lane. Besides, if the velocity of the ego-vehicle is less than the expected velocity, we accelerate the vehicle with the allowed maximal acceleration.

                if(car_straight)
                {
                  if(!car_left && lane>0)
                  {
                    lane--; 
                  }
                  else if(!car_right && lane<2)
                  {
                    lane++;
                  }
                  else
                  {
                    ref_vel -= MAX_ACC;
                  }
                }
                else if(ref_vel < MAX_SPEED)
                {
                  
                    ref_vel += MAX_ACC;
                }

### Trajectory Planning
If the previous trajectory has no point or only one point, then set the refence point to the current position of the ego-vehicle and set the reference yaw rate to the current yaw rate of the ego vehicle. 

                  //Use two points that make the path tangent to the car
                  double prev_car_x = car_x - cos(car_yaw);
                  double prev_car_y = car_y - sin(car_yaw);

                  ptsx.push_back(prev_car_x);
                  ptsx.push_back(car_x);

                  ptsy.push_back(prev_car_y);
                  ptsy.push_back(car_y);
                  
If the previous trajectory has more or equal to 2 points, the reference rate will be caiculated based on the last two points of the previous trajectory.

                  ref_x = previous_path_x[prev_size -1];
                  ref_y = previous_path_y[prev_size -1];

                  double ref_x_prev = previous_path_x[prev_size -2];
                  double ref_y_prev = previous_path_y[prev_size -2];

                  ref_yaw = atan2(ref_y-ref_y_prev, ref_x-ref_x_prev);  

                  ptsx.push_back(ref_x_prev);
                  ptsx.push_back(ref_x);

                  ptsy.push_back(ref_y_prev);
                  ptsy.push_back(ref_y);
                  
As we need more points, so based on the current Frenet position we use Function getXY to obtain position in cartesian coordinate after 30 m, 60 m, 90 m in Frenet Coordinate. So bascially we have 5 points to obtain the spline.

In order to keep the consistency, all the points in previous trajectory will be used in the new trajectory. Always 50 points will be created for the next trajectory. So how do we get the rest points besides the points from the previous trajectory. It is assumed that the ego-vehicle will move with reference velocity for 30 m in the horizon. The target distance can be calculated as follows:

                double target_x = 30.0;
                double target_y = s(target_x);
                double target_dist = sqrt(target_x*target_x + target_y*target_y);
                
If we know the x_point, then we know the corresponing y_point.

                double x_add_on = 0;

                // Fill up the rest of our path planner after filling it with previous points, 
                for (int i = 1; i <= 50-previous_path_x.size(); i++)
                {
                  double N = (target_dist/(.02*ref_vel/2.24));
                  double x_point = x_add_on + target_x/N;
                  double y_point = s(x_point);

                  x_add_on = x_point;

                  double x_ref = x_point;
                  double y_ref = y_point;

                  // rotate back to normal after rotating it earlier
                  x_point = (x_ref * cos(ref_yaw)-y_ref*sin(ref_yaw));
                  y_point = (x_ref * sin(ref_yaw)+y_ref*cos(ref_yaw));

                  x_point += ref_x;
                  y_point += ref_y;

                  next_x_vals.push_back(x_point);
                  next_y_vals.push_back(y_point);
                }
                
### Simulator.
You can download the Term3 Simulator which contains the Path Planning Project from the [releases tab (https://github.com/udacity/self-driving-car-sim/releases/tag/T3_v1.2).  

To run the simulator on Mac/Linux, first make the binary file executable with the following command:
```shell
sudo chmod u+x {simulator_file_name}
```

### Goals
In this project your goal is to safely navigate around a virtual highway with other traffic that is driving +-10 MPH of the 50 MPH speed limit. You will be provided the car's localization and sensor fusion data, there is also a sparse map list of waypoints around the highway. The car should try to go as close as possible to the 50 MPH speed limit, which means passing slower traffic when possible, note that other cars will try to change lanes too. The car should avoid hitting other cars at all cost as well as driving inside of the marked road lanes at all times, unless going from one lane to another. The car should be able to make one complete loop around the 6946m highway. Since the car is trying to go 50 MPH, it should take a little over 5 minutes to complete 1 loop. Also the car should not experience total acceleration over 10 m/s^2 and jerk that is greater than 10 m/s^3.

#### The map of the highway is in data/highway_map.txt
Each waypoint in the list contains  [x,y,s,dx,dy] values. x and y are the waypoint's map coordinate position, the s value is the distance along the road to get to that waypoint in meters, the dx and dy values define the unit normal vector pointing outward of the highway loop.

The highway's waypoints loop around so the frenet s value, distance along the road, goes from 0 to 6945.554.

## Basic Build Instructions

1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./path_planning`.

Here is the data provided from the Simulator to the C++ Program

#### Main car's localization Data (No Noise)

["x"] The car's x position in map coordinates

["y"] The car's y position in map coordinates

["s"] The car's s position in frenet coordinates

["d"] The car's d position in frenet coordinates

["yaw"] The car's yaw angle in the map

["speed"] The car's speed in MPH

#### Previous path data given to the Planner

//Note: Return the previous list but with processed points removed, can be a nice tool to show how far along
the path has processed since last time. 

["previous_path_x"] The previous list of x points previously given to the simulator

["previous_path_y"] The previous list of y points previously given to the simulator

#### Previous path's end s and d values 

["end_path_s"] The previous list's last point's frenet s value

["end_path_d"] The previous list's last point's frenet d value

#### Sensor Fusion Data, a list of all other car's attributes on the same side of the road. (No Noise)

["sensor_fusion"] A 2d vector of cars and then that car's [car's unique ID, car's x position in map coordinates, car's y position in map coordinates, car's x velocity in m/s, car's y velocity in m/s, car's s position in frenet coordinates, car's d position in frenet coordinates. 

## Details

1. The car uses a perfect controller and will visit every (x,y) point it recieves in the list every .02 seconds. The units for the (x,y) points are in meters and the spacing of the points determines the speed of the car. The vector going from a point to the next point in the list dictates the angle of the car. Acceleration both in the tangential and normal directions is measured along with the jerk, the rate of change of total Acceleration. The (x,y) point paths that the planner recieves should not have a total acceleration that goes over 10 m/s^2, also the jerk should not go over 50 m/s^3. (NOTE: As this is BETA, these requirements might change. Also currently jerk is over a .02 second interval, it would probably be better to average total acceleration over 1 second and measure jerk from that.

2. There will be some latency between the simulator running and the path planner returning a path, with optimized code usually its not very long maybe just 1-3 time steps. During this delay the simulator will continue using points that it was last given, because of this its a good idea to store the last points you have used so you can have a smooth transition. previous_path_x, and previous_path_y can be helpful for this transition since they show the last points given to the simulator controller with the processed points already removed. You would either return a path that extends this previous path or make sure to create a new path that has a smooth transition with this last path.

## Tips

A really helpful resource for doing this project and creating smooth trajectories was using http://kluge.in-chemnitz.de/opensource/spline/, the spline function is in a single hearder file is really easy to use.

---

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
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets 
    cd uWebSockets
    git checkout e94b6e1
    ```

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


## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to ensure
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

## How to write a README
A well written README file can enhance your project and portfolio.  Develop your abilities to create professional README files by completing [this free course](https://www.udacity.com/course/writing-readmes--ud777).

