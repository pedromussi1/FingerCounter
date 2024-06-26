<h1>Code Breakdown</h1>

<p>This report covers the functionality and implementation of a hand gesture-based volume control program. The program leverages computer vision and hand tracking technology to allow users to control the volume of a Windows computer using hand gestures. The main components of the program include a hand tracking module and a volume control script.</p>

<h2>HandTrackingModule</h2>

<p>
This module is responsible for detecting and tracking hand landmarks using MediaPipe and OpenCV.
</p>


<p>The handDetector class initializes the MediaPipe hand tracking model with specified parameters: mode, maxHands, detectionCon, and trackCon.</p>

<p>MediaPipe's hands module is configured for hand detection and tracking, and its drawing utilities are set up for visualizing the hand landmarks.</p>

```py
import cv2
import mediapipe as mp
import time
import math

class handDetector():
    def __init__(self, mode=False, maxHands=2, detectionCon=0.5, trackCon=0.5):
        self.mode = mode
        self.maxHands = maxHands
        self.detectionCon = detectionCon
        self.trackCon = trackCon

        self.mpHands = mp.solutions.hands
        self.hands = self.mpHands.Hands(static_image_mode=self.mode,
                                        max_num_hands=self.maxHands,
                                        min_detection_confidence=self.detectionCon,
                                        min_tracking_confidence=self.trackCon)
        self.mpDraw = mp.solutions.drawing_utils

```

<p>The findHands method processes an image to detect hands and optionally draws landmarks on the image.
</p>

<p>The image is converted to RGB format, and the hand detection model processes it to find hand landmarks.
</p>

<p>If landmarks are detected, they are drawn on the image.
</p>

```py
def findHands(self, img, draw=True):
    imgRGB = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    self.results = self.hands.process(imgRGB)
    if self.results.multi_hand_landmarks:
        for handLms in self.results.multi_hand_landmarks:
            if draw:
                self.mpDraw.draw_landmarks(img, handLms, self.mpHands.HAND_CONNECTIONS)
    return img
```

<p>The findPosition method identifies and returns the positions of the landmarks of the detected hand.</p>

<p>It calculates the pixel coordinates of each landmark and optionally draws them on the image.
</p>

```py
def findPosition(self, img, handNo=0, draw=True):
    xList = []
    yList = []
    bbox = []
    self.lmList = []
    if self.results.multi_hand_landmarks:
        myHand = self.results.multi_hand_landmarks[handNo]
        for id, lm in enumerate(myHand.landmark):
            h, w, c = img.shape
            cx, cy = int(lm.x * w), int(lm.y * h)
            xList.append(cx)
            yList.append(cy)
            self.lmList.append([id, cx, cy])
            if draw:
                cv2.circle(img, (cx, cy), 5, (255, 0, 255), cv2.FILLED)
        xmin, xmax = min(xList), max(xList)
        ymin, ymax = min(yList), max(yList)
        bbox = xmin, ymin, xmax, ymax
        if draw:
            cv2.rectangle(img, (bbox[0] - 20, bbox[1] - 20), (bbox[2] + 20, bbox[3] + 20), (0, 255, 0), 2)
    return self.lmList, bbox
```

<p>The findDistance method calculates the Euclidean distance between two specified landmarks.
</p>

<p>It also draws lines and circles between the landmarks to visually indicate the measurement.
</p>

```py
def findDistance(self, p1, p2, img, draw=True):
    x1, y1 = self.lmList[p1][1], self.lmList[p1][2]
    x2, y2 = self.lmList[p2][1], self.lmList[p2][2]
    cx, cy = (x1 + x2) // 2, (y1 + y2) // 2
    length = math.hypot(x2 - x1, y2 - y1)
    if draw:
        cv2.line(img, (x1, y1), (x2, y2), (255, 0, 255), 3)
        cv2.circle(img, (x1, y1), 10, (255, 0, 255), cv2.FILLED)
        cv2.circle(img, (x2, y2), 10, (255, 0, 255), cv2.FILLED)
        cv2.circle(img, (cx, cy), 10, (255, 0, 255), cv2.FILLED)
    return length, img, [x1, y1, x2, y2, cx, cy]
```

<p>The fingersUp method determines which fingers are up by analyzing the positions of the landmarks.
</p>

<p>It compares the positions of the fingertip landmarks with those of their corresponding lower joints.
</p>

```py

def fingersUp(self):
    fingers = []
    if self.lmList[self.tipIds[0]][1] > self.lmList[self.tipIds[0] - 1][1]:
        fingers.append(1)
    else:
        fingers.append(0)
    for id in range(1, 5):
        if self.lmList[self.tipIds[id]][2] < self.lmList[self.tipIds[id] - 2][2]:
            fingers.append(1)
        else:
            fingers.append(0)
    return fingers
```

<h2>VolumeHandControlAdvanced</h2>

<p>The script sets up webcam capture and initializes the hand detector.
</p>

<p>It configures the audio system using the pycaw library to adjust the system volume.
</p>

<p>The script initializes the webcam capture and the hand detector.
</p>

```py
import cv2
import numpy as np
import time
import ctypes
import comtypes
from ctypes import cast, POINTER
from comtypes import CLSCTX_ALL
from pycaw.pycaw import AudioUtilities, IAudioEndpointVolume

import HandTrackingModule as htm

cap = cv2.VideoCapture(0)
detector = htm.handDetector(detectionCon=0.7, maxHands=1)

devices = AudioUtilities.GetSpeakers()
interface = devices.Activate(IAudioEndpointVolume._iid_, CLSCTX_ALL, None)
volume = cast(interface, POINTER(IAudioEndpointVolume))
volRange = volume.GetVolumeRange()
minVol = volRange[0]
maxVol = volRange[1]
vol = 0
volBar = 400
volPer = 0
```

<p>The main loop reads frames from the webcam and processes them using the hand detector.
</p>

<p>It identifies hand landmarks, calculates the area of the detected hand, and filters frames based on the hand size.
</p>

<p>The script measures the distance between the thumb and index finger, maps this distance to a volume level, and adjusts the system volume accordingly.
</p>

<p>Visual feedback is provided by drawing on the webcam feed.
</p>

```py
pTime = 0
while True:
    success, img = cap.read()
    img = detector.findHands(img)
    lmList, bbox = detector.findPosition(img, draw=True)
    
    if len(lmList) != 0:
        area = (bbox[2] - bbox[0]) * (bbox[3] - bbox[1]) // 100
        
        if 250 < area < 1000:
            length, img, lineInfo = detector.findDistance(4, 8, img)
            volBar = np.interp(length, [50, 200], [400, 150])
            volPer = np.interp(length, [50, 200], [0, 100])
            
            smoothness = 10
            volPer = smoothness * round(volPer / smoothness)
            
            fingers = detector.fingersUp()
            if not fingers[4]:
                volume.SetMasterVolumeLevelScalar(volPer / 100, None)
                cv2.circle(img, (lineInfo[4], lineInfo[5]), 15, (0, 255, 0), cv2.FILLED)
                colorVol = (0, 255, 0)
            else:
                colorVol = (255, 0, 0)
    
    cv2.rectangle(img, (50, 150), (85, 400), (255, 0, 0), 3)
    cv2.rectangle(img, (50, int(volBar)), (85, 400), (255, 0, 0), cv2.FILLED)
    cv2.putText(img, f'{int(volPer)} %', (40, 450), cv2.FONT_HERSHEY_COMPLEX, 1, (255, 0, 0), 3)
    
    cVol = int(volume.GetMasterVolumeLevelScalar() * 100)
    cv2.putText(img, f'Vol Set: {int(cVol)}', (400, 50), cv2.FONT_HERSHEY_COMPLEX, 1, colorVol, 3)
    
    cTime = time.time()
    fps = 1 / (cTime - pTime)
    pTime = cTime
    cv2.putText(img, f'FPS: {int(fps)}', (40, 50), cv2.FONT_HERSHEY_COMPLEX, 1, (255, 0, 0), 3)
    
    cv2.imshow("Img", img)
    cv2.waitKey(1)

```

<h2>Hand Tracking Module (HandTrackingModule.py):</h2>

<h3>Initialization (__init__ method):</h3>

<p>This method sets up the MediaPipe hands solution with specified parameters such as mode, maxHands, detectionCon, and trackCon.
</p>

<p>The self.mpHands initializes the MediaPipe hands solution.
</p>

<p>The self.hands object is configured with the provided parameters to detect and track hand landmarks.
</p>

<p>self.mpDraw is set up to draw the detected hand landmarks.
</p>

<h3>Hand Detection (findHands method):</h3>

<p>The input image is converted from BGR to RGB format.
</p>

<p>The self.hands.process method processes the RGB image to detect hand landmarks.
</p>

<p>If hand landmarks are detected, they are drawn on the image using self.mpDraw.
</p>

<h3>Position Detection (findPosition method):</h3>

<p>This method iterates over the detected landmarks and calculates their pixel coordinates.
</p>

<p>It appends the coordinates to self.lmList and optionally draws them on the image.
</p>

<p>The bounding box (bbox) around the hand is calculated based on the minimum and maximum coordinates of the landmarks.
</p>

<h3>Distance Calculation (findDistance method):</h3>

<p>This method calculates the Euclidean distance between two specified landmarks.
</p>

<p>It draws lines and circles between the landmarks to visually indicate the distance.
</p>

<h3>Finger Status (fingersUp method):</h3>

<p>This method determines the status of the fingers (up or down) by comparing the positions of the fingertip landmarks with their corresponding lower joints.
</p>

<p>It returns a list indicating the status of each finger.
</p>

<h2>Volume Control Script (VolumeHandControlAdvanced.py):</h2>

<h3>Initialization:</h3>

<p>The script captures video from the webcam and initializes the hand detector.</p>

<p>The pycaw library is used to control the system volume. It retrieves the audio endpoint and sets up the volume control interface.</p>

<p>The volume range (minVol and maxVol) is obtained from the audio endpoint.</p>

<h3>Main Loop:</h3>

<p>The script continuously captures frames from the webcam and processes them using the findHands and findPosition methods of the handDetector class.</p>

<p>If hand landmarks are detected, the area of the bounding box around the hand is calculated.</p>

<p>The script filters frames based on the hand size to ensure reliable detection.</p>

<p>The distance between the thumb and index finger is measured using the findDistance method, and this distance is mapped to a volume level.</p>

<p>If the pinky finger is not up, the system volume is adjusted based on the mapped volume level.</p>

<p>The volume level is displayed on the webcam feed, and visual feedback is provided to the user.</p>

