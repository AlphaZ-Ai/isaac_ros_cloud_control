# Setup ros cloud control

This readme will setup the environment for you, if you want to look at where these steps were taken from, you can check the official docs, which are linked in each section.
start here -> https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_cloud_control/isaac_ros_mission_client/index.html

## setup isaacsim in docker

pull the image

```bash
docker pull nvcr.io/nvidia/isaac-sim:5.1.0
```

run the container, change the `USD_DIR` to where you store usd assets to be able to access them from the container.

```bash
USD_DIR=~/.cache/
```

```bash
docker run --name isaac-sim -it --rm --gpus all --network host \
  -e ACCEPT_EULA=Y \
  -e PRIVACY_CONSENT=Y \
  -e ROS_DOMAIN_ID=0 \
  -v "$USD_DIR":/host_usd:rw \
  -v ~/docker/isaac-sim/cache/kit:/isaac-sim/kit/cache:rw \
  -v ~/docker/isaac-sim/cache/ov:/root/.cache/ov:rw \
  -v ~/docker/isaac-sim/cache/pip:/root/.cache/pip:rw \
  -v ~/docker/isaac-sim/cache/glcache:/root/.cache/nvidia/GLCache:rw \
  -v ~/docker/isaac-sim/cache/computecache:/root/.nv/ComputeCache:rw \
  -v ~/docker/isaac-sim/logs:/root/.nvidia-omniverse/logs:rw \
  -v ~/docker/isaac-sim/data:/root/.local/share/ov/data:rw \
  -v ~/docker/isaac-sim/documents:/root/Documents:rw \
  nvcr.io/nvidia/isaac-sim:5.1.0 \
  ./runheadless.sh -v
```
outside the container, download and run the webrtc client

```bash
wget https://download.isaacsim.omniverse.nvidia.com/isaacsim-webrtc-streaming-client-1.1.4-linux-x64.AppImage
chmod +x isaacsim-webrtc-streaming-client-1.1.4-linux-x64.AppImage
./isaacsim-webrtc-streaming-client-1.1.4-linux-x64.AppImage
```

to run the demo, go to  `Window` -> `Examples` -> then check `Robot Examples`
that will open a tab at the bottom, in that tab go to `ROS2` -> `ISAAC ROS` -> `Sample Scene` and `Load Sample Scene`

## setup isaac sim dev env

the following steps install the isaac-ros cli, the docs can be found here -> https://nvidia-isaac-ros.github.io/getting_started/index.html

first setup the env workspace

```bash
mkdir -p  ~/workspaces/isaac_ros-dev/src
echo 'export ISAAC_ROS_WS="${ISAAC_ROS_WS:-${HOME}/workspaces/isaac_ros-dev/}"' >> ~/.bashrc
source ~/.bashrc
```

then install the cli

```bash
locale  # check for UTF-8

sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

locale  # verify settings
```

```bash
sudo apt update && sudo apt install curl gnupg
sudo apt install software-properties-common
sudo add-apt-repository universe
k="/usr/share/keyrings/nvidia-isaac-ros.gpg"
curl -fsSL https://isaac.download.nvidia.com/isaac-ros/repos.key | sudo gpg --dearmor \
    | sudo tee -a $k > /dev/null
f="/etc/apt/sources.list.d/nvidia-isaac-ros.list"
sudo touch $f
s="deb [signed-by=$k] https://isaac.download.nvidia.com/isaac-ros/release-4.2 noble main"
grep -qxF "$s" $f || echo "$s" | sudo tee -a $f

sudo apt-get update
pip install termcolor --break-system-packages
sudo apt-get install isaac-ros-cli
```

setup docker 

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
sudo isaac-ros init docker
```

then activate 

```bash
isaac-ros activate
```

install the needed packages, note you will need to do this every time you restart the container


```bash
sudo apt-get update
sudo apt-get install -y ros-${ROS_DISTRO}-rmw-cyclonedds-cpp
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

after the above install, you should be able to run the example env and see the ros2 topics being published

```bash
sudo apt update
sudo apt-get install -y ros-jazzy-isaac-ros-mission-client
```



## setup the mqtt broker

these docs can be found here -> https://nvidia-isaac-ros.github.io/concepts/missions/isaac_ros_mission_client.html

outside the container 

```bash
touch ~/mosquitto.sh
echo "CONFIG_FILE=/mosquitto.conf
if [ $# != 2 ] ; then
    echo "usage: $0 <tcp_port> <websocket_port>"
    exit 1
fi
PORT=$1
PORT_WEBSOCKET=$2
echo "allow_anonymous true" >> $CONFIG_FILE
echo "listener $PORT 0.0.0.0" >> $CONFIG_FILE
echo "listener $PORT_WEBSOCKET" >> $CONFIG_FILE
echo "protocol websockets" >> $CONFIG_FILE
mosquitto -c $CONFIG_FILE" > ~/mosquitto.sh
chmod +x ~/mosquitto.sh
```

then run the broker

```bash
docker run -it --network host -v ~/mosquitto.sh:/mosquitto.sh -d eclipse-mosquitto:latest sh mosquitto.sh 1883 9001
```

## run the mission client

run this command in the isaac-ros-dev workspace, run `isaac-ros activate` if you haven't already

```bash
ros2 launch isaac_ros_vda5050_client_bringup isaac_ros_vda5050_client_nav2.launch.py init_pose_x:=-2.0 init_pose_yaw:=3.14159
```

## start mission dispatch

start the postfgres database

```bash
export POSTGRES_PASSWORD=test
docker run --rm --name postgres \
  --network host \
  -p 5432:5432 \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD \
  -e POSTGRES_DB=mission \
  -d postgres:14.5
```

start api and database server

```bash
docker run -it --network host nvcr.io/nvidia/isaac/mission-database:4.1.0
```

start the mission dispatch server

```bash
docker run -it --network host nvcr.io/nvidia/isaac/mission-dispatch:4.1.0
```

go to `http://localhost:5000/docs`


## test mission control
This assumes you have the example scene running with the carter robot.

first you need to register the robot in the `POST /register` endpoint.

```json
{
  "labels": [],
  "battery": {
    "critical_level": 10
  },
  "heartbeat_timeout": 30,
  "switch_teleop": false,
  "name": "carter01"
}
```

then you need to send the `POST /mission` endpoint with the following body

```json
{
  "robot": "carter01",
  "mission_tree": [
    {
      "name": "travel_to_point",
      "parent": "root",
      "route": {
        "waypoints": [
          {
            "x": 0,
            "y": 3,
            "theta": 0,
            "map_id": "",
            "allowedDeviationXY": 0.1,
            "allowedDeviationTheta": 0
          }
        ]
      }
    }
  ],
  "name": "mission_001"
}
```

