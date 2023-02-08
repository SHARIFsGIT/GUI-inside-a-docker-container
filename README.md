# How to open a GPU application inside a docker container in ubuntu

Hello dockers! 

Recently I worked on a docker file for ROBOKUDO and tried to open a GUI inside from that docker container. Here is a brief note that might be helpful for you as well.

Docker file for robokudo:

```
# Base image with ROS Noetic installed
FROM osrf/ros:noetic-desktop-full
SHELL ["/bin/bash", "-c"]
# Installing necessary packages and dependencies
RUN apt-get update && apt-get install -y \
    python3-tk \
    python3-pip \
    python3-catkin-tools \
    python3-catkin-pkg \
    python3-rosdep \
    python3-wstool \
    ros-noetic-rqt-py-trees \
    ros-noetic-ddynamic-reconfigure-python \
    ros-noetic-py-trees-ros \
    git \
    ros-noetic-cv-bridge \
    ros-noetic-tf \
    && rm -rf /var/lib/apt/lists/* # Cleanup apt cache & reduce image size \
# Creates a new directory and clones the Robokudo repository
RUN mkdir -p /robokudo_ws/src
WORKDIR /robokudo_ws/src
RUN git clone --recurse-submodules https://gitlab.informatik.uni-bremen.de/robokudo/robokudo.git
WORKDIR /robokudo_ws
# Configures the catkin workspace and builds
RUN catkin config --extend /opt/ros/noetic #extended for catkin WS
RUN catkin build
# Installs required python packages listed in the requirements.txt file using pip
RUN python3 -m pip install --upgrade pip
RUN pip install --requirement /robokudo_ws/src/robokudo/requirements.txt --user
# Adds a source to bashrc file to run whenever a new terminal is started
RUN echo "source /robokudo_ws/devel/setup.bash" >> ~/.bashrc
```

To build this docker:
```
sudo docker build --no-cache --pull -t robokudo_container .
```

Wait until download the packages and finish all the steps. If you got an error like:
>Could not find a version that satisfies the requirement open3d~=0.15.2(from -r requirements.txt)


add this line `RUN python3 -m pip install --no-cache-dir --upgrade open3d` to your docker file after `RUN python3 -m pip install --upgrade pip` and now build the docker again with `sudo docker build --no-cache --pull -t robokudo_container .`
If so far works perfectly then our next step is to go inside of the docker and check whether open3d works properly or not. To go inside of our build image:
```
sudo docker run --rm -it robokudo_container
```

Now you are inside of the docker image and for checking open3d works or not just write `python3` then `import open3d`. If you see libraries required then install the libraries inside the docker with 
```
apt-get update && apt-get install --no-install-recommends -y \
libgl1 \
libgomp1 \
&& rm -rf /var/lib/apt/lists/*
```

Now try again `python3` then `import open3d` and lastly `open3d.__version__`. This should work now and you can see the version of the open3d package.
If you want to run a 3D GPU application inside from the docker image then you need to run your built docker image a bit differently. For that use
```
docker run -it --net=host --gpus all --env="NVIDIA_DRIVER_CAPABILITIES=all" --env="DISPLAY" --env="QT_X11_NO_MITSHM=1" --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" robokudo_container bash
```

If so far works perfectly then I believe you are now inside of the docker container. Now just open a new terminal in your machine and run `roscore` then `rosrun robokudo main.py` inside of your docker container.
Don't worry! You are almost near if you got an error like:
>Failed to open X display.


Open a new terminal and run `xhost +` and try again `rosrun robokudo main.py` inside of your docker container.
If still facing the same problem then you may check the display value is equal or not. 
Try `echo $DISPLAY` outside of the container or on your machine, which will return a value ':1' or ':0' and do the same command inside of the container. If the value is different then run `export DISPLAY=:1` (if :1 is the value of the inside container).
Now try again `roscore` in your local terminal then 
```
docker run -it --net=host --gpus all --env="NVIDIA_DRIVER_CAPABILITIES=all" --env="DISPLAY" --env="QT_X11_NO_MITSHM=1" --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" robokudo_container bash
```

to your docker file and `xhost +`  to a new local terminal and finally `rosrun robokudo main.py` inside of your docker container.
Again if you fall into another error like:
>could not connect to display :0. This application failed to start because no Qt platform plugin could be initialized. Reinstalling the application may fix this problem.

Then the solution can be switched between graphics chips. For that just open a terminal on your local machine and then run 
```
prime-select query
sudo prime-select intel
sudo prime-select nvidia
```

Then restart your machine and try `xhost +` in a local terminal then try again `roscore` in your local terminal. 
```
docker run -it --net=host --gpus all --env="NVIDIA_DRIVER_CAPABILITIES=all" --env="DISPLAY" --env="QT_X11_NO_MITSHM=1" --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" robokudo_container bash
```

to your docker file and finally `rosrun robokudo main.py` inside of your docker container.
