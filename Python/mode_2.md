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