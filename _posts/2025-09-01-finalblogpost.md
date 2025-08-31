---
layout: post
title: "Final Blog Post - On-the-fly Analysis of WESTPA Segments using Trajectory Streaming (WIP)"
date: 2025-08-27 00:00:00 +0100
description: Week 10-11 focused on benchmarking and improving the test systems.
# img: gsocheader.png # Add image post (optional)
tags: [GSOC, Molecular dynamics] # add tag
---

My Google Summer of Code project has been an amazing experience, and I'm deeply grateful to my mentors: Oliver Beckstein ([@orbeckst](https://github.com/orbeckst)), Jeremy Leung ([@jeremyleung521](https://github.com/jeremyleung521)), Lawson Woods ([@ljwoods2](https://github.com/ljwoods2)), and Lillian Chong ([@ltchong](https://github.com/ltchong)). In this final post, I'll describe the work I accomplished during the program.

# Why On-the-fly Analysis?
The goal of implementing on-the-fly analysis of WESTPA segments using trajectory streaming is to improve the efficiency of weighted ensemble simulations. Weighted ensemble simulations operate by running many short simulations (potentially thousands) in parallel. These short simulations are often called 'segments.' The segments are propagated in iterations with the progress of the segment being tracked using a progress coordinate, which is some function of the atomic positions. Currently, the WESTPA workflow separates the simulation and the calculation of the progress coordinate into two distinct steps. The simulation runs first, and the progress coordinate is calculated afterward. This can be inefficient because if the analysis takes a significant amount of time, the overall time for the segment to complete increases. This project uses Interactive Molecular Dynamics (IMD), along with the [IMDClient](https://github.com/Becksteinlab/imdclient) and MDAnalysis, to stream trajectory data while the simulations run. This allows the analysis to be executed in parallel with the simulation, increasing the efficiency of the WESTPA run. Analyzing on-the-fly also allows for more complex analyses during the WESTPA run without additional costs. 

# Creating a Framework for Streaming WESTPA segments
My primary goal was to create a framework for streaming and analyzing segment simulation data on-the-fly. Initially, I tried to integrate this directly into WESTPA by creating a new executable in `executable.py` (see WESTPA [PR #495](https://github.com/westpa/westpa/pull/495)). The aim was to create an executable that would be launched alongside the executable for the segment and run the analysis. However, this approach proved tricky because a port had to be assigned to each segment and communicated to both the segment executable and the analysis executable. While WESTPA could find and assign an available port to the segment, ensuring the port remained open long enough for the simulation engine to bind to it was difficult. Instead, I opted for a simpler approach: using a Python script to launch both the simulation and analysis for each segment. This script is run within the executable for the segment. First, the script launches the simulation engine with IMD enabled, instructing it to use port 0. By assigning the port to 0, the simulation engine automatically finds an available port. The script then reads the output of the simulation engine to identify which port has been assigned. Once the port has been identified, an MDAnalysis universe can be created that connects to the simulation engine using the IMDClient. The analysis can then be performed on-the-fly as the simulation is running. The contents of this script are now encapsulated in the `TrajectoryStreamer` class (WESTPA [PR #501](https://github.com/westpa/westpa/pull/501)).

# Using the TrajectoryStreamer Class
The `TrajectoryStreamer` class offers a convenient interface for streaming trajectory data from the simulation engine to the analysis code. It manages port assignment and connects to the simulation engine using the IMDClient, providing an MDAnalysis universe for analyzing the streamed data. The `TrajectoryStreamer` requires three inputs: the simulation engine name (currently supporting GROMACS, LAMMPS, and NAMD), an MDAnalysis-compatible template file, and a string used to launch the simulation. A typical usage of the TrajectoryStreamer class is as follows:
```python
from westpa.tools.trajectory_streaming import TrajectoryStreamer
stream = TrajectoryStreamer("gromacs", "template.gro", "gmx_imd mdrun -s seg.tpr -o seg.trr -c seg.gro -e seg.edr -cpo seg.cpt -g seg.log -nt 5 -imdwait -imdport 0")
# The following function starts the simulation and returns the corresponding MDAnalysis universe
u = stream.start_sim_and_get_universe(timeout=30)
# We can create an atom group from the universe
ag = u.select_atoms("protein and name CA")
for ts in u.trajectory:
    # Perform your analysis here
    dists = self_distance_array(ag, box=ts.dimensions)
# Once the analysis is complete the end_sim function is used to get the simulation output
# and end the simulation if it is still running
stream.end_sim()
```
The `timeout` keyword is important if the frequency of trajectory frames sent for analysis is low. This value should be greater than the time between frames. The Python file containing this code should be called within the WESTPA runseg.sh script (see the [WESTPA tutorials](https://github.com/westpa/westpa_tutorials) for more information on running weighted ensemble simulations in WESTPA), replacing the lines that normally launch the simulation and run the analysis. For example this snippet from a runseg.sh script:
```bash
############################## Run the dynamics ################################
# Propagate the segment using gmx mdrun
$GMX mdrun -s seg.tpr -o seg.trr -c seg.gro -e seg.edr \
  -cpo seg.cpt -g seg.log -nt 6

########################## Calculate and return data ###########################

# Calculate the progress coordinate and return it to westpa
python $WEST_SIM_ROOT/common_files/get_pairwise_dists.py
cat pcoord_out.dat > $WEST_PCOORD_RETURN
```
will now become:

```bash
############### Run the dynamics and calculate progress coordinate #############
# This script needs to contain the creation of the TrajectoryStreamer object
python $WEST_SIM_ROOT/common_files/run_sim_and_get_pairwise_dists.py
cat pcoord_out.dat > $WEST_PCOORD_RETURN
```

# When is it Appropriate to Stream the Segments?
By benchmarking WESTPA simulations, I have determined when streaming segments is beneficial. My benchmarks (detailed in [my previous post](https://jpkrowe.github.io/posts/week10-11)) suggest that streaming provides the best performance boost when the time to analyze a segment's simulation is similar to the simulation time. This is shown in the below graph:
![Realised performance boost as a function of analysis/simulation time ratio](/assets/img/realised_performance_boost.png)Therefore, I recommend first benchmarking both the simulation time for one segment and the time to analyze that segment's simulation. If these times are comparable, then streaming the segments is likely to be beneficial. It is important to note that the overhead of streaming is relatively low, and it enables significantly more analysis. This is shown in this graph:
![Plot of iteration time against number of points](/assets/img/Iteration_times.png)


# What Next?
The upcoming IMDClient release will significantly change its interaction with MDAnalysis. The `IMDReader` functionality, previously part of IMDClient, has moved into MDAnalysis itself. My immediate plan is to update the `TrajectoryStreamer` class to use this new approach. During testing, the analysis sometimes ended prematurely, even while the simulation was running. This was due to IMDClient timeouts, which occur when there's a long delay between frames. This issue can be resolved by increasing the stream_timeout parameter. However, as noted in [this GitHub issue](https://github.com/Becksteinlab/imdclient/issues/111), the reason for the analysis failure isn't always clear to users. I aim to improve how IMDClient handles timeouts to provide clearer feedback. Additionally, I plan to add more comprehensive tests and documentation for TrajectoryStreamer to make it easier to use and understand. Currently, the analysis and simulation of a segment must occur on the same computing node. However, the IMD protocol can transmit data across non-local hosts. In the future, I plan to enable running the analysis and simulation on separate nodes, offering users greater flexibility.

# What Have I Learned?
I learned a great deal during the GSOC process. These things include:
- The importance of benchmarking and profiling code to identify performance bottlenecks.
- Learning to be practical about what is achievable.
- Focusing on implementations that work first before trying to do things perfectly.
- Learning to build upon existing code and frameworks.
- Using git and GitHub effectively for version control and collaboration.
