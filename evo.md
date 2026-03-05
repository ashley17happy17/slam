# evo evaluation toolkit

# Run
After installation with pip, the following executables can be called globally from your command-line:

**Metrics:**

* `evo_ape` - absolute pose error
```
evo_ape tum ./output/08.tum ./kitti_transformation3.tum -va --plot --plot_mode xz --save_results results/ORB.zip -a
```
* `evo_rpe` - relative pose error

**Tools:**

* `evo_traj` - tool for analyzing, plotting or exporting one or more trajectories
```
# Utilize Umeyama's method to conduct time sync and output synchronized tum file
evo_traj tum ./output/kitti_transformation3.tum --ref ./output/08.tum -a --correct_scale --sync -p --plot_mode=xz --save_as_tum
```

* `evo_res` - tool for comparing one or multiple result files from `evo_ape` or `evo_rpe`
```
# Plot the evo_ape results into charts
evo_res ./results/ORB.zip -p --save_table ./results/table.csv
```

* `evo_config` - tool for global settings and config file manipulation

Call the commands with `--help` to see the options, e.g. `evo_ape --help`. Tab-completion of command line parameters is available on UNIX-like systems.

**More documentation**
Check out the [Wiki on GitHub](https://github.com/MichaelGrupp/evo/wiki).


# Reference
1. https://github.com/MichaelGrupp/evo
