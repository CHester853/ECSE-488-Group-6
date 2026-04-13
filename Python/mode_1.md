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