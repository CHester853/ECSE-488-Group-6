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