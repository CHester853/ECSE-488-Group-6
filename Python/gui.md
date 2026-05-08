```
import tkinter as tk
from tkinter import ttk
import customtkinter as ctk
from PIL import Image, ImageTk
import os
import pandas as pd
import cv2

# --- ROS IMPORTS ---
import rospy
from sensor_msgs.msg import Image as RosImage
from std_msgs.msg import Int32
from cv_bridge import CvBridge

# --- SYSTEM SETTINGS ---
ctk.set_appearance_mode("dark")
ctk.set_default_color_theme("blue")

bodyFont = 'Arial'
scrn_width = 1200
scrn_height = 700

# --- ROS GLOBALS & SETUP ---
bridge = CvBridge()
latest_gui_frame = None
current_gui_mode = 1
first_frame_received = False

# Variable to store our ROS connection so we can kill it later
camera_sub = None

def ros_video_callback(data):
	global latest_gui_frame, first_frame_received
	
	if not first_frame_received:
		print("[INFO] Successfully receiving ROS video feed!")
		first_frame_received = True
	try:
		latest_gui_frame = bridge.imgmsg_to_cv2(data, "rgb8")
	except Exception as e:
		print(f"[GUI ERROR] Video feed: {e}")

def ros_mode_callback(data):
	global current_gui_mode
	current_gui_mode = data.data

# Initialize ROS Node for the GUI
rospy.init_node('security_gui_monitor', anonymous=True)

# Save the subscriber to the 'camera_sub' variable
camera_sub = rospy.Subscriber("/camera/rgb/image_color", RosImage, ros_video_callback)
rospy.Subscriber("/system/current_mode", Int32, ros_mode_callback)

# --- GUI SETUP ---
GUI = ctk.CTk()
GUI.title("Security")
GUI.geometry(f"{scrn_width}x{scrn_height}")

GUI.grid_rowconfigure(0, weight=6)
GUI.grid_rowconfigure(1, weight=4)
GUI.grid_columnconfigure((0, 1, 2, 3), weight=1)

# ==========================================
# TOP LEFT: VIDEO FEED
# ==========================================
Camera_frame = ctk.CTkFrame(GUI, corner_radius=10, fg_color="#1e1e1e")
Camera_frame.grid(row=0, column=0, columnspan=2, padx=10, pady=10, sticky="nsew")

camera = ctk.CTkLabel(Camera_frame, text="Waiting for ROS Video...", fg_color="transparent")
camera.pack(expand=True, fill="both")

# ==========================================
# TOP RIGHT: LOG TABLE
# ==========================================
log_frame = ctk.CTkFrame(GUI, corner_radius=10, fg_color="#1e1e1e")
log_frame.grid(row=0, column=2, columnspan=2, padx=10, pady=10, sticky="nsew")

style = ttk.Style()
style.theme_use("default")
style.configure("Treeview.Heading", font=('Arial', 12, "bold"), background="#e0e0e0", foreground="black")
style.configure("Treeview", font=('Arial', 12), rowheight=25, background="white", foreground="blue", fieldbackground="white")

log = ttk.Treeview(log_frame, height=15)
log.pack(expand=True, fill="both", padx=5, pady=5)

def open_file():
	try:
		df = pd.read_csv(r'/home/SaPHaRI/chester/data/report_log/system_events.txt')
		log['column'] = list(df.columns)
		log['show'] = 'headings'
		for col in log['column']:
			log.heading(col, text=col)
		df_rows = df.to_numpy().tolist()
		for row in df_rows:
			log.insert("", "end", values=row)
	except Exception as e:
		print(f"Waiting for CSV data... ({e})")

# ==========================================
# BOTTOM COL 1: CAMERA SWITCHES
# ==========================================
cam_cntrl = ctk.CTkFrame(GUI, corner_radius=10, fg_color="#2b2b2b")
cam_cntrl.grid(row=1, column=0, columnspan=1, padx=10, pady=10, sticky="nsew")
  
cam_cntrl.grid_rowconfigure((0, 1, 2, 3), weight=1)
cam_cntrl.grid_columnconfigure(0, weight=1)
cam_cntrl.grid_columnconfigure(1, weight=1)

# Function to handle turning Camera 1 on and off
def toggle_camera1():
	global camera_sub, latest_gui_frame
	
	if switch_var1.get() == "on":
		# Reconnect to ROS if we aren't already
		if camera_sub is None:
			print("[INFO] Reconnecting to Camera 1...")
			camera_sub = rospy.Subscriber("/camera/rgb/image_color", RosImage, ros_video_callback)
	else:
		# Disconnect from ROS
		if camera_sub is not None:
			print("[INFO] Disconnecting from Camera 1...")
			camera_sub.unregister()
			camera_sub = None
			
			# Just set the frame to None. The update loop will draw the offline screen.
			latest_gui_frame = None

def create_switch_row(frame, row_num, command=None):
	lbl = ctk.CTkLabel(frame, text=f"Camera {row_num + 1}", font=(bodyFont, 18))
	lbl.grid(row=row_num, column=0, sticky='w', padx=20)
	
	switch_var = ctk.StringVar(value="on")
	switch = ctk.CTkSwitch(frame, text="", variable=switch_var, onvalue="on", offvalue="off", progress_color="#3a7ebf", command=command)
	switch.grid(row=row_num, column=1, sticky='e', padx=20)
	return switch_var

# Assign the toggle function to switch 1
switch_var1 = create_switch_row(cam_cntrl, 0, command=toggle_camera1)
switch_var2 = create_switch_row(cam_cntrl, 1)
switch_var3 = create_switch_row(cam_cntrl, 2)
switch_var4 = create_switch_row(cam_cntrl, 3)

# ==========================================
# BOTTOM COL 2: RESET BUTTON
# ==========================================
reset_frame = ctk.CTkFrame(GUI, corner_radius=10, fg_color="#2b2b2b")
reset_frame.grid(row=1, column=1, columnspan=1, padx=10, pady=10, sticky="nsew")

def buttonevent():
	print("Reset clicked")

reset_button = ctk.CTkButton(reset_frame, text='Reset', command=buttonevent, corner_radius=100, font=('arial', 24, 'bold'))
reset_button.pack(expand=True, fill="both", padx=20, pady=40)

# ==========================================
# BOTTOM COL 3: ALARM ICON
# ==========================================
alarm_color = "white"
alarm_frame = ctk.CTkFrame(GUI, fg_color=alarm_color, corner_radius=10)
alarm_frame.grid(row=1, column=2, columnspan=1, padx=10, pady=10, sticky="nsew")

try:
	raw_alarm_img = Image.open(r"/home/SaPHaRI/chester/resources/alarm transparent.png")
	alarm_pic = ctk.CTkImage(light_image=raw_alarm_img, size=(120, 120))
	alarm_label = ctk.CTkLabel(alarm_frame, image=alarm_pic, compound="center", text='Mode 1', text_color="black", font=('arial', 22,'bold'))
	alarm_label.pack(expand=True)
except Exception as e:
	alarm_label = ctk.CTkLabel(alarm_frame, text='Mode 1', text_color="black", font=('arial', 24,'bold'))
	alarm_label.pack(expand=True)

# ==========================================
# BOTTOM COL 4: PICTURES FOLDER
# ==========================================

def pics():
	os.system('xdg-open /home/SaPHaRI/chester/data/image')

pic_frame = ctk.CTkFrame(GUI, corner_radius=10, fg_color="#2b2b2b")
pic_frame.grid(row=1, column=3, columnspan=1, padx=10, pady=10, sticky="nsew")  

try:
	raw_folder_img = Image.open(r"/home/SaPHaRI/chester/resources/folder.PNG")
	folder_pic = ctk.CTkImage(light_image=raw_folder_img, size=(60, 60))
	folder_button = ctk.CTkButton(pic_frame, image=folder_pic, text='Pictures', compound="top", command=pics, font=('arial', 16), corner_radius=15)
	folder_button.pack(expand=True, fill="both", padx=30, pady=40)
except Exception as e:
	folder_button = ctk.CTkButton(pic_frame, text='Pictures', command=pics, font=('arial', 16), corner_radius=15)
	folder_button.pack(expand=True, fill="both", padx=30, pady=40)

# ==========================================
# UNIFIED UPDATE LOOP
# ==========================================
def update_gui_loop():
	global latest_gui_frame, current_gui_mode
	
	frame_w = Camera_frame.winfo_width()
	frame_h = Camera_frame.winfo_height()
	
	# Only draw if the window has fully loaded and has a size
	if frame > 50 and frame_h > 50:
		if latest_gui_frame is not None:
			# 1. Dynamically scale the ROS video feed to fit the box
			captured_image = Image.fromarray(latest_gui_frame)
			photo_image = ctk.CTkImage(light_image=captured_image, size=(frame_w - 20, frame_h - 20))
			camera.configure(image=photo_image, text="")
			
			# CRITICAL: This line stops Python from deleting the image from RAM!
			camera.image = photo_image
	
	else:
		# 2. Draw a blank dark-gray screen when offline instead of using "None"
		blank_image = Image.new('RGB', (frame_w - 20, frame_h - 20), color="#1e1e1e")
		photo_image = ctk.CTkImage(light_image=blank_image, size=(frame_w - 20, frame_h - 20))
		camera.configure(image=photo_image, text="Camera 1 Offline")
		
		# CRITICAL: Anchor the blank image too
		camera.image = photo_image
	
	# 3. Update Alarm Text and Color based on ROS Mode
	alarm_label.configure(text=f"Mode {current_gui_mode}")
	
	if current_gui_mode == 4:
		alarm_frame.configure(fg_color="#c92a2a")
		alarm_label.configure(text_color="white")
	else:
		alarm_frame.configure(fg_color="white")
		alarm_label.configure(text_color="black")
		
	GUI.after(50, update_gui_loop)

# Run initial setup
open_file()
update_gui_loop()

GUI.mainloop()
```