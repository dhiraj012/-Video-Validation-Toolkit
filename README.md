# -Video-Validation-Toolkit
GitHub's Video Validation Toolkit streamlines video development, offering quality assessment and format validation tools for efficient, reliable applications.

# - Code -

import tkinter as tk
from tkinter import filedialog
import cv2
import mediapipe as mp
from moviepy.editor import VideoFileClip
import os
import shutil

# Initialize MediaPipe Face Detection and Face Mesh
mp_face_detection = mp.solutions.face_detection
mp_face_mesh = mp.solutions.face_mesh
mp_drawing = mp.solutions.drawing_utils

# Declare face_detection as a global variable
face_detection = mp_face_detection.FaceDetection(min_detection_confidence=0.5)
face_mesh = mp_face_mesh.FaceMesh(min_detection_confidence=0.5, min_tracking_confidence=0.5)

# Create a Tkinter window
window = tk.Tk()
window.title("Video Validation Toolkit")

# Get screen width and height
screen_width = window.winfo_screenwidth()
screen_height = window.winfo_screenheight()

# Set the dimensions for the button and center it on the screen
button_width = 200
button_height = 50
x_position = (screen_width - button_width) / 2
y_position = (screen_height - button_height) / 2

# Function to browse for a video file
def browse_video():
    file_path = filedialog.askopenfilename(filetypes=[("Video files", "*.mp4")])
    validate_video(file_path)

# Function to copy accepted video to a separate folder
def copy_accepted_video(video_path):
    # Define the name of the directory to store accepted videos
    accepted_videos_directory = "AcceptedVideos"  # You can change this to your desired directory name
    
    # Create the directory if it doesn't exist
    os.makedirs(accepted_videos_directory, exist_ok=True)
    
    # Get the filename from the full video path
    _, video_filename = os.path.split(video_path)
    
    # Construct the destination path for the accepted video in the specified directory
    destination_path = os.path.join(accepted_videos_directory, video_filename)
    
    # Copy the video file to the accepted videos directory
    shutil.copy(video_path, destination_path)


# Function to draw landmarks on the face and validate lip visibility
def validate_video(video_path):
    cap = cv2.VideoCapture(video_path)
    
    # Use MoviePy to get the video frame rate
    video_clip = VideoFileClip(video_path)
    frame_rate = video_clip.fps

    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break

        # Calculate the aspect ratio of the video frame
        frame_height, frame_width, _ = frame.shape
        frame_aspect_ratio = frame_width / frame_height

        # Calculate the maximum dimensions that fit within the screen
        max_width = screen_width
        max_height = int(max_width / frame_aspect_ratio)

        # If the calculated height is too large, use the screen height as the limit
        if max_height > screen_height:
            max_height = screen_height
            max_width = int(max_height * frame_aspect_ratio)

        # Resize the frame to fit within the screen dimensions
        frame = cv2.resize(frame, (max_width, max_height))

        # Convert the frame to RGB for MediaPipe
        frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)

        # Detect faces
        results = face_detection.process(frame_rgb)

        if results.detections:
            for detection in results.detections:
                # Get the exact bounding box dimensions
                bboxC = detection.location_data.relative_bounding_box
                h, w, c = frame.shape
                x, y, width, height = int(bboxC.xmin * w), int(bboxC.ymin * h), int(bboxC.width * w), int(bboxC.height * h)

                if width > 0 and height > 0:
                    # Draw a bounding box around the face
                    cv2.rectangle(frame, (x, y), (x + width, y + height), (0, 255, 0), 2)

                # Detect facial landmarks
                landmarks = face_mesh.process(frame_rgb)

                if landmarks.multi_face_landmarks:
                    for face_landmarks in landmarks.multi_face_landmarks:
                        # Draw facial landmarks (lips, eyes, nose)
                        mp_drawing.draw_landmarks(frame, face_landmarks, mp_face_mesh.FACEMESH_CONTOURS,
                                                 landmark_drawing_spec=mp_drawing.DrawingSpec(color=(0, 255, 0), thickness=1, circle_radius=1))

                    # You can access specific lip landmarks here
                    # For example, landmark 13 is the upper lip center, and landmark 14 is the lower lip center
                    upper_lip = face_landmarks.landmark[13]
                    lower_lip = face_landmarks.landmark[14]

                    # You can add validation logic for lip visibility here
                    if upper_lip and lower_lip:
                        if frame_rate == 30:
                            validation_result.config(text="Accepted (Frontal Face & Visible Lips & 30 FPS)")
                            copy_accepted_video(video_path)  # Copy the accepted video
                        elif frame_rate == 25:
                            validation_result.config(text="Accepted (Frontal Face & Visible Lips & 25 FPS)")
                            copy_accepted_video(video_path)  # Copy the accepted video
                        else:
                            validation_result.config(text="Rejected (Not 25 or 30 FPS)")
                    else:
                        validation_result.config(text="Rejected (No Visible Lips)")
                else:
                    validation_result.config(text="Rejected (No Frontal Face)")

        # Display the video frame
        cv2.imshow("Video Validation", frame)

        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

    cap.release()
    cv2.destroyAllWindows()

# Set the background image
background_image = tk.PhotoImage(file="pexels-lukas-317356.png")  # Specify the path to your background image

# Create a Canvas widget with the same dimensions as the image
canvas = tk.Canvas(window, width=background_image.width(), height=background_image.height())
canvas.create_image(0, 0, anchor=tk.NW, image=background_image)
canvas.pack()

background_label = tk.Label(window, image=background_image)
background_label.place(relwidth=1, relheight=1)


# Create GUI elements
browse_button = tk.Button(window, text="Browse Video", font=("Helvetica", 20), width=20, height=2, bg="lightblue", command=browse_video)
validation_result = tk.Label(window, text="Validation Result: ", font=("Helvetica", 20), bg="lightblue")

# Layout the GUI elements
browse_button.place(x=x_position, y=y_position)
validation_result.place(x=x_position, y=y_position + button_height + 90)  # Adjust the vertical position of the label

# Layout the GUI elements
#browse_button.place(x=x_position, y=y_position)
#validation_result.pack()

# Start the Tkinter main loop
window.mainloop()



