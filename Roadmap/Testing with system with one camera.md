## Current code
- main_system.py
```
import cv2
import freenect
import time
import os
from datetime import datetime

# IMPORT CUSTOM MODE FILES
import mode_1
import mode_2
import mode_3
import mode_4

# Setup paths to match your actual directories
DATA_DIR = os.path.expanduser("~/chester/data")
IMG_DIR = os.path.join(DATA_DIR, "image")
VID_DIR = os.path.join(DATA_DIR, "video")  

# Saving the text file inside your report_log folder
LOG_FILE = os.path.join(DATA_DIR, "report_log", "system_events.txt")

# Auto-create the folders just in case they ever get deleted or moved
os.makedirs(IMG_DIR, exist_ok=True)
os.makedirs(VID_DIR, exist_ok=True)
os.makedirs(os.path.join(DATA_DIR, "report_log"), exist_ok=True)

hog = cv2.HOGDescriptor()
hog.setSVMDetector(cv2.HOGDescriptor_getDefaultPeopleDetector())

# Dummy functions for Kinect hardware (use the ones from the previous code)
def get_video(dev_idx):
	array, _ = freenect.sync_get_video(dev_idx)
	return cv2.cvtColor(array, cv2.COLOR_RGB2BGR) if array is not None else None

def get_depth_in_meters(dev_index):
	"""Fetches depth array and calculates distance at the center of the frame."""
	try:
		# Get raw 11-bit depth from the Kinect
		depth_array, _ = freenect.sync_get_depth(dev_index, freenect.DEPTH_11BIT)
		if depth_array is not None:
			# Check the center pixel of the 640x480 frame
			raw_depth = depth_array[240, 320]
			
			if raw_depth < 2047: # 2047 is the Kinect's "blind spot" value
				# Standard Kinect V1 formula: convert raw depth to meters
				distance_m = 1.0 / (raw_depth * -0.0030711016 + 3.3309495161)
				return distance_m
	except TypeError:
		pass

	return 99.9 # Return a high distance if sensor fails to read

def main():
	cameras = [0, 1]
	current_cam_idx = 0
	last_swap_time = time.time()
	
	current_mode = 1
	video_writer = None

	while True:
		# 1. Hardware: Grab frame from the active camera
		active_cam = cameras[current_cam_idx]
		frame = get_video(active_cam)
		if frame is None:
			continue
			
		# 2. Logic: Detect Person
		process_frame = cv2.resize(frame, (320, 240))
		boxes, _ = hog.detectMultiScale(process_frame)
		
		person_detected = len(boxes) > 0
		timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
		
		# 3. Logic: Determine the correct mode
		if not person_detected:
			current_mode = 1
			# If we were recording in mode 4, stop the recording
			if video_writer is not None:
				video_writer.release()
				video_writer = None
		else:
			distance = get_depth_in_meters(active_cam)
			if distance > 4.0:
				current_mode = 2
			elif distance > 2.0:
				current_mode = 3
			else:
				current_mode = 4

		# 4. Execution: Pass data to the separate Mode Files
		if current_mode == 1:
			current_cam_idx, last_swap_time = mode_1.run(
				current_cam_idx, len(cameras), last_swap_time, swap_delay=3.0)
		
		elif current_mode == 2:
			mode_2.run(frame, timestamp, IMG_DIR)
		
		elif current_mode == 3:
			mode_3.run(frame, distance, timestamp, IMG_DIR, LOG_FILE)
		
		elif current_mode == 4:
			# We pass the video_writer in and get it back so the file stays open
			video_writer = mode_4.run(frame, distance, timestamp, video_writer, VID_DIR, LOG_FILE)

if __name__ == "__main__":
	main()
```
- mode_1.py
```
import time

def run(current_cam_index, total_cameras, last_swap_time, swap_delay):
	"""Mode 1: No threat. Swap cameras every few seconds."""
	
	if time.time() - last_swap_time > swap_delay:
		# Time to swap to the next camera
		next_cam = (current_cam_index + 1) % total_cameras
		print(f"[MODE 1] Swapping to Camera {next_cam}")
		return next_cam, time.time()
	
	# Otherwise, stay on current camera
	return current_cam_index, last_swap_time
```
- mode_2.py
```
import cv2
import os
import time

def run(frame, timestamp, img_dir):
	"""Mode 2: Far Threat. Save low-res image."""
	print("[MODE 2] Target detected FAR. Saving low-res image.")
	
	# Resize and compress to save SD card space
	low_res_frame = cv2.resize(frame, (320, 240))
	filename = os.path.join(img_dir, f"mode2_{timestamp}.jpg")
	cv2.imwrite(filename, low_res_frame, [cv2.IMWRITE_JPEG_QUALITY, 50])
	
	time.sleep(0.3) # Rate limit to prevent flooding the SD card
```
- mode_3.py
```
import cv2
import os
import time
  
def run(frame, distance, timestamp, img_dir, log_file):
	"""Mode 3: Medium Threat. Save high-res image and trigger low alarm."""
	print(f"[MODE 3] Target MEDIUM distance ({distance:.1f}m). Saving high-res.")
	
	# Save Image
	filename = os.path.join(img_dir, f"mode3_{timestamp}.jpg")
	cv2.imwrite(filename, frame, [cv2.IMWRITE_JPEG_QUALITY, 90])
	
	# Write to log
	with open(log_file, "a") as f:
		f.write(f"[{timestamp}] MODE 3 ALERT | Distance: {distance:.2f}m\n")
	
	# Trigger Alarm
	print("[ALARM] Sounding low-level alarm!")
	
	time.sleep(0.14) # Rate limit to ~7 frames a seconds
```
- mode_4.py
```
import cv2
import os

def run(frame, distance, timestamp, video_writer, vid_dir, log_file):
	"""Mode 4: Close Threat. Record video and trigger high alarm."""
	print(f"[MODE 4] Target CLOSE ({distance:.1f}m). Recording video!")
	
	# Write to log
	with open(log_file, "a") as f:
		f.write(f"[{timestamp}] MODE 4 ALERT | Distance: {distance:.2f}m\n")
	
	# Trigger High Alarm
	print("[ALARM] Sounding HIGH-LEVEL siren!")
	
	# Initialize video writer if it hasn't been started yet
	if video_writer is None:
		vid_filename = os.path.join(vid_dir, f"mode4_clip_{timestamp}.avi")
		fourcc = cv2.VideoWriter_fourcc(*'XVID')
		video_writer = cv2.VideoWriter(vid_filename, fourcc, 10.0, (640, 480))
		print(f"[MODE 4] Started recording: {vid_filename}")
	
	# Write the frame to the video file
	video_writer.write(frame)
	
	return video_writer # Return the writer back to the master so it keeps the file open
```

## Experiments
Try to run the Python script
- Navigate to the specific folder
	- `cd chester/python`
	- `python3 main_ssytem.py`
	- **Error:** No module named `freenect`

Try to install `freenect`
- `sudo apt install python3-freenect libfreenect-dev`
	- Failed
- `sudo apt install libfreenect-dev`
- `pip3 install Cython "setuptools<61.0.0" wheel`
- `pip3 install --no-build-isolation freenect`
	- Got some warning, but successfully installed
- **BUT** when I run the `main_system.py` again, it still fails

Change `def get_video` part in `main_system.py` 
```
def get_video(dev_idx):
	# Grab the raw data, but check if it actually exists first!
	video_data = freenect.sync_get_video(dev_idx)
	
	if video_data is None:
		print(f"[WARNING] Camera {dev_idx} failed to open or is missing.")
		return None
```

Fixing the USB permission
- `wget https://raw.githubusercontent.com/OpenKinect/libfreenect/master/platform/linux/udev/51-kinect.rules`
- `sudo mv 51-kinect.rules /etc/udev/rules.d/`
- `sudo udevadm control --reload-rules && sudo udevadm trigger`
- Still fails

I would like to change the logic
- Switch from pure Python and come back to ROS

Change `main_system.py`
```
import cv2
import rospy
from sensor_msgs.msg import Image
from cv_bridge import CvBridge, CvBridgeError
import time
import os
from datetime import datetime

# IMPORT CUSTOM MODE FILES
import mode_1
import mode_2
import mode_3
import mode_4

# Setup paths
DATA_DIR = os.path.expanduser("~/chester/data")
IMG_DIR = os.path.join(DATA_DIR, "image")
VID_DIR = os.path.join(DATA_DIR, "video")
LOG_FILE = os.path.join(DATA_DIR, "report_log", "system_events.txt")

os.makedirs(IMG_DIR, exist_ok=True)
os.makedirs(VID_DIR, exist_ok=True)
os.makedirs(os.path.join(DATA_DIR, "report_log"), exist_ok=True)

hog = cv2.HOGDescriptor()
hog.setSVMDetector(cv2.HOGDescriptor_getDefaultPeopleDetector())

# --- ROS GLOBALS & SETUP ---
bridge = CvBridge()
latest_rgb_frame = None
latest_depth_m = 99.9  # Default to far away

def rgb_callback(data):
    global latest_rgb_frame
    try:
        # Convert ROS image message to OpenCV BGR format
        latest_rgb_frame = bridge.imgmsg_to_cv2(data, "bgr8")
    except CvBridgeError as e:
        print(f"[ERROR] RGB Bridge: {e}")

def depth_callback(data):
    global latest_depth_m
    try:
        # Convert ROS depth message to OpenCV array
        depth_image = bridge.imgmsg_to_cv2(data, desired_encoding="passthrough")
        
        # Check the center pixel
        h, w = depth_image.shape
        center_depth = depth_image[h//2, w//2]
        
        # Handle ROS depth encodings (freenect usually outputs mm as 16UC1 or meters as 32FC1)
        if center_depth == 0 or center_depth != center_depth: # Handle 0 or NaN blind spots
            latest_depth_m = 99.9
        elif data.encoding == "32FC1":
            latest_depth_m = float(center_depth) # Already in meters
        elif data.encoding == "16UC1":
            latest_depth_m = float(center_depth) / 1000.0 # Convert mm to meters
            
    except CvBridgeError as e:
        print(f"[ERROR] Depth Bridge: {e}")

def main():
    global latest_rgb_frame, latest_depth_m
    
    # Initialize this script as a ROS Node
    rospy.init_node('surveillance_master', anonymous=True)
    
    # Subscribe to the camera topics (Matching your camera:=camera1 flag)
    rospy.Subscriber("/camera1/rgb/image_color", Image, rgb_callback)
    rospy.Subscriber("/camera1/depth/image_raw", Image, depth_callback)
    
    print("[INFO] Waiting for ROS camera feed...")
    
    cameras = [0, 1]
    current_cam_idx = 0
    last_swap_time = time.time()
    current_mode = 1
    video_writer = None

    # Replace "while True" with the ROS standard shutdown check
    while not rospy.is_shutdown():
        # 1. Hardware: Grab frame from the ROS callback
        if latest_rgb_frame is None:
            time.sleep(0.1) # Wait for the first frame to arrive
            continue
            
        # Copy the frame so we don't accidentally edit it while ROS updates it
        frame = latest_rgb_frame.copy() 
            
        # 2. Logic: Detect Person
        process_frame = cv2.resize(frame, (320, 240))
        boxes, _ = hog.detectMultiScale(process_frame)
        
        person_detected = len(boxes) > 0
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

        # 3. Logic: Determine the correct mode
        if not person_detected:
            current_mode = 1
            if video_writer is not None:
                video_writer.release()
                video_writer = None
        else:
            distance = latest_depth_m
            if distance > 4.0:
                current_mode = 2
            elif distance > 2.0:
                current_mode = 3
            else:
                current_mode = 4

        # 4. Execution: Pass data to the separate Mode Files
        if current_mode == 1:
            current_cam_idx, last_swap_time = mode_1.run(
                current_cam_idx, len(cameras), last_swap_time, swap_delay=3.0)
                
        elif current_mode == 2:
            mode_2.run(frame, timestamp, IMG_DIR)
            
        elif current_mode == 3:
            mode_3.run(frame, distance, timestamp, IMG_DIR, LOG_FILE)
            
        elif current_mode == 4:
            video_writer = mode_4.run(frame, distance, timestamp, video_writer, VID_DIR, LOG_FILE)

if __name__ == "__main__":
    main()
```

This time it works!
- 1st terminal
	- `ros core`
- 2nd terminal
	- `roslaunch ...`
- 3rd terminal
	- `cd chester/python`
	- `python3 main_system.py`
- 4th terminal
	- `web`

Before go to two cameras, I changed `mode_4.py` as well
```
import cv2
import os

def run(frame, distance, timestamp, video_writer, vid_dir, log_file):
	"""Mode 4: Close Threat. Record video and trigger high alarm."""
	print(f"[MODE 4] Target CLOSE ({distance:.1f}m). Recording video!")
	
	# Write to log
	with open(log_file, "a") as f:
		f.write(f"[{timestamp}] MODE 4 ALERT | Distance: {distance:.2f}m\n")
	
	# Trigger High Alarm
	print("[ALARM] Sounding HIGH-LEVEL siren!")
	
	# Initialize video writer if it hasn't been started yet
	if video_writer is None:
		vid_filename = os.path.join(vid_dir, f"mode4_clip_{timestamp}.mp4")
		fourcc = cv2.VideoWriter_fourcc(*'mp4v')
		video_writer = cv2.VideoWriter(vid_filename, fourcc, 10.0, (640, 480))
		print(f"[MODE 4] Started recording: {vid_filename}")
	
	# Write the frame to the video file
	video_writer.write(frame)
	
	return video_writer # Return the writer back to the master so it keeps the file open
```