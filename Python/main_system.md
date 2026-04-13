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