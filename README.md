# <div align=center>duckietown_object_recognition</div>
#### <div align="center">" This repository is created for object recognition functionality of Duckiebot, </div>
#### <div align="center"> which reads the camera view of a duckiebot and detects the obstacle in the front. "</div>

***

# What you need to prepare
1. Linux environment
2. Image group for your own dataset

# I. Introduction to ROS

### A. How to install ROS Noetic
1. You can take a look at the ROS Wiki page to follow <a href="http://wiki.ros.org/noetic/Installation/Ubuntu">instructions to ROS Noetic installation</a>.

### B. How to create ROS Workspace and Package
1. Create a ROS workspace

       $ mkdir -p ~/catkin_ws/src
       $ cd ~/catkin_ws/
       $ catkin_make
       $ source devel/setup.bash

2. Get into your ROS workspace

       $ cd ~/catkin_ws/src
       
3. Copy a ROS Package

       $ git clone https://github.com/ldw200012/Duckietown_YOLOv5.git

4. Run below commands to configure your ROS Package

       $ cd ~/catkin_ws
       $ catkin_make
       $ source devel/setup.bash (This command must be run on every shell you are using for ROS from now on)
       
# II. Introduction to YOLOv5

### A. Custom the program (you can skip to B if you only want to detect duckie)

- Prepare a dataset 

1. Data preparation from duckiebot camera (You can skip this step if you already have your own images).
       
       $ rosrun image_transport republish compressed in:=/[duckiebot_name]/camera_node/image raw out:=/[duckiebot_name]/camera_node/image/raw
       $ cd [folder_directory_to_save_images]
       $ rosrun image_view image_saver image:=/JoudiDuck/camera_node/image/raw
     
2. Data Annotation

      Follow the instructions in Roboflow https://blog.roboflow.com/vott/
      
3. Dataset Creation
       
      Follow the instructions in Roboflow https://www.youtube.com/watch?v=MdF6x6ZmLAY until 15:40.
      
      From now on, you need to remember the "<b>link</b>" provided by Roboflow. (Choose 'Terminal' among 'Jupyter/Terminal/Raw URL')
  
- Train the dataset:

1. Go into the '/content' folder in this git repository and clean it.

       $ cd Duckietown_YOLOv5/content
       $ rm -rf *
       
2. Clone a git repository below

       $ git clone https://github.com/ultralytics/yolov5
       $ cd yolov5
       $ git reset --hard 886f1c03d839575afecb059accf74296fad395b6
       
3. Install all requirements (ignore errors)

       $ pip install -qr requirements.txt

4. Open Python and run below codes

       $ python
       >> import torch
       >> from IPython.display import Image, clear_output
       >> from utils.google_utils import gdrive_download
       >> 
       >> print('Setup complete. Using torch %s %s' % (torch.__version__, torch.cuda.get_device_properties(0) if torch.cuda.is_available() else 'CPU'))

5. Copy and paste the Terminal link from your Roboflow dataset.
       
   (If you see a question "replace data.yaml? [y]es, [n]o, [A]ll, [r]ename:", enter A for all)

       $ cd Duckietown_YOLOv5/content
       $ curl -L "https://..." > roboflow.zip; unzip roboflow.zip; rm roboflow.zip

      Then check the created data.yaml file
      
       $ cat data.yaml      
       
      Now you will see the data.yaml file created.
       
6. Open Python and run below codes

       $ python
       >> # define number of classes based on YAML
       >> import yaml
       >> with open("data.yaml", 'r') as stream:
       ..     num_classes = str(yaml.safe_load(stream)['nc'])
       
7. Create your own .yaml file

       $ cd Duckietown_YOLOv5content/yolov5/models/
       $ touch custom_yolov5s.yaml
       $ nano custom_yolov5s.yaml
       
      Then, paste the below cell into custom_yolov5s.yaml, save.
       
       # parameters
       nc: {num_classes}  # number of classes
       depth_multiple: 0.33  # model depth multiple
       width_multiple: 0.50  # layer channel multiple
       
       # anchors
       anchors:
         - [10,13, 16,30, 33,23]  # P3/8
         - [30,61, 62,45, 59,119]  # P4/16
         - [116,90, 156,198, 373,326]  # P5/32

       # YOLOv5 backbone
       backbone:
         # [from, number, module, args]
         [[-1, 1, Focus, [64, 3]],  # 0-P1/2
          [-1, 1, Conv, [128, 3, 2]],  # 1-P2/4
          [-1, 3, BottleneckCSP, [128]],
          [-1, 1, Conv, [256, 3, 2]],  # 3-P3/8
          [-1, 9, BottleneckCSP, [256]],
          [-1, 1, Conv, [512, 3, 2]],  # 5-P4/16
          [-1, 9, BottleneckCSP, [512]],
          [-1, 1, Conv, [1024, 3, 2]],  # 7-P5/32
          [-1, 1, SPP, [1024, [5, 9, 13]]],
          [-1, 3, BottleneckCSP, [1024, False]],  # 9
         ]

       # YOLOv5 head
       head:
         [[-1, 1, Conv, [512, 1, 1]],
          [-1, 1, nn.Upsample, [None, 2, 'nearest']],
          [[-1, 6], 1, Concat, [1]],  # cat backbone P4
          [-1, 3, BottleneckCSP, [512, False]],  # 13

          [-1, 1, Conv, [256, 1, 1]],
          [-1, 1, nn.Upsample, [None, 2, 'nearest']],
          [[-1, 4], 1, Concat, [1]],  # cat backbone P3
          [-1, 3, BottleneckCSP, [256, False]],  # 17 (P3/8-small)

          [-1, 1, Conv, [256, 3, 2]],
          [[-1, 14], 1, Concat, [1]],  # cat head P4
          [-1, 3, BottleneckCSP, [512, False]],  # 20 (P4/16-medium)

          [-1, 1, Conv, [512, 3, 2]],
          [[-1, 10], 1, Concat, [1]],  # cat head P5
          [-1, 3, BottleneckCSP, [1024, False]],  # 23 (P5/32-large)

          [[17, 20, 23], 1, Detect, [nc, anchors]],  # Detect(P3, P4, P5)
         ]
         
      Now you are ready to train your own data!

8. Train your own dataset

       $ cd Duckietown_YOLOv5/content/yolov5/
       $ python train.py --img 416 --batch 16 --epochs 100 --data '../data.yaml' --cfg ./models/custom_yolov5s.yaml --weights '' --name yolov5s_results  --cache
       
      You can change the epochs number as you want :-)

9. Check for trained weights

       $ ls runs/
       $ ls runs/train/yolov5s_results/weights
       
      You will see two * .pt files, which are your weights files. 

### B. How to run the program



***
# About the Project

#### Module Name: 
#### Instructors: 
#### Contributors: 


