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