# Autonomous Robog: Vision-Guided Robot

  Our "goosebot" is a small autonomous ground vehicle that races around a track using only a camera and a trained object-detection model. It was built by Jacob Brescia, Ethan Beasley, and myself for an Autonomous Robotic Systems course. The challenge was to complete three laps in under 60 seconds while staying between white boundary lines and following yellow guidance lines, with no human control. Our final robot finished in 53 seconds, 7 seconds under the requirement, and managed to recover on its own from overshoots on the long stretches.

  The project combined machine learning, computer vision, control theory, embedded hardware, and a lot of hands-on lab tuning. Its main lesson was that performance came less from adding complexity than from the quality of the training data, the tuning process, and knowing when to simplify.

# Hardware
  The robot was built on a 3D-printed frame with four DC motors driven by two L298N motor drivers. A Radxa Rock5c Lite single-board computer ran the software, connected to a PWM I2C bridge for motor control. Perception came from a USB wide-field-of-view webcam. Power came from a 3S 11.1V 2200mAh LiPo battery through an adjustable DC/DC converter. We started from an existing open-source robot codebase provided by our professor and modified it heavily.

# Software 

  Our starting point was 42 images provided by our professor. The original model, built for an earlier lane-following assignment, detected the white lines poorly. We re-annotated the entire dataset from scratch after the first two retrained models were still not consistent enough. We took our own photos in the lab and built a set of 65 images. With augmentations that varied saturation, brightness, exposure, and blur, the dataset grew to 169 images. That let the model train on the imperfect lighting and image quality it would face on the real robot. Because 169 images is small for object detection, we used a 92% train, 4% validation, 4% test split so nearly all the data went to training.

We trained several YOLOv11 object-detection models in Colab and compared them. The one we chose reached 0.83 precision, 0.75 recall, 0.75 mAP50, and 0.57 mAP50-95 across both classes. The white line was harder to detect (0.68 mAP50) than the yellow (0.82 mAP50), matching what we saw on the track. We first built the full system in ROS2, with each node handling one job:

• A perception node that read the camera, ran the YOLO model, and computed lane error
• A control node that ran a PD controller on the lane error and produced drive commands
• A motor node that turned those commands into hardware signals
• A keyboard node for manual override and shutdown
• A web-stream node that streamed the camera feed for debugging

It worked, and it gave us a clean, modular design. On the track, though, it performed poorly for a speed competition. Config values were hard to change on the fly, and there was a long delay between launching from the terminal and the robot moving. Because of this we went to a lighter, single-script Python approach, and it proved faster and smoother with fewer stutters. Deciding to drop the more sophisticated architecture turned out to be one of our best engineering decisions. In drive.py, we were able to tune:

• Confidence threshold: After noticing the robot ignoring some white lines, we found our model's confidence values were below the script's cutoff. We lowered the threshold to 0.2, since the model rarely produced false lines.
• Vertical cutoff: The robot sometimes reacted to lines too far away or missed nearby ones. Adjusting the cutoff to 0.7 fixed this.
• PD controller gains and speed: This had the biggest effect on lap time. Other groups told us a base speed of 0.15 barely met the 60-second limit, so we started at 0.20. After hours of lab testing, we settled on Kp = 0.0008 and Kd = 0.00015. When the robot became unstable late in lap three, we dropped the base speed to 0.19, which made runs consistent.

  For added ease and efficiency, we added:

• A Ctrl+C shutdown handler that stops all motors safely and lets us edit configs without powering the robot off and on
• An "Enter to start" gate that lets the model and camera load before the robot moves, cutting about 5 seconds from each timed run.
• Web stream for debugging. Watching the camera's live detection feed showed us exactly what the model saw, which is how we caught the confidence-threshold problem.

# Results

  In the lab , we recorded the robot before competition day. It managed to complete three laps in 53 seconds. The timer started only when Enter was pressed. The robot sometimes over- or undershoots the long straightaways, but it corrects itself before leaving the track, a good sign that the detection model is reliable. We also ran it from different starting positions to check that it could consistently finish under a minute. Battery voltage, starting position, and camera angle all affected results, so we tested for consistency and not just one good run. 

Youtube Video: https://www.youtube.com/watch?v=-edryXxe_Rs


# What I Learned
  Quality beats quantity. Collecting our own images and using image augmentation mattered more than model tweaks. Sometimes simpler can be faster, Modular ROS2 was elegant, but the lighter python script won when accuracy and speed mattered.

  Tuning goes a long way, even small, careful adjustments to PD gains and speed produced most of our speed improvement. you cannot test enough because there will always be little variables to change. Developer experience also matters, Fast restarts and safe shutdowns let us iterate far more in the same amount of lab time.
  
# Future Improvements

The robot still tends to touch the white line on straightaways and occasionally crosses the yellow line, so finer PD tuning could cut a few more seconds and let the robot handle lane guidance better. A larger, more varied dataset would also help the model handle lighting and surface conditions it hasn't seen. 
