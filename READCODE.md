<h1>Code Breakdown</h1>

<p>The hand-tracking and finger-counting program aims to identify human hands using a webcam and determine how many fingers are being held up. This application is implemented using OpenCV for video processing and MediaPipe for hand detection and landmark recognition. The report provides an overview of the program's functionality, detailed code explanation, and potential improvements.</p>

<h2>HandTrackingModule</h2>

<p>
The handDetector class encapsulates the hand detection and tracking functionalities. Key methods include:
</p>

<p>__init__: Initializes the hand detector with parameters for mode, maximum hands, detection confidence, and tracking confidence.
</p>

<p>findHands: Processes an image to detect hands and draw landmarks.
</p>

<p>findPosition: Identifies and returns the positions of hand landmarks.
</p>

<p>handType: Determines whether the detected hand is left or right.
</p>

<p>fingersUp: Determines which fingers are up based on landmark positions.
</p>

<p>findDistance: Calculates the distance between two landmarks and optionally draws this distance on the image.
</p>


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

<p>The handDetector class constructor initializes the hand tracking model with the specified parameters.
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

<p>The findHands method processes the input image to detect hands and optionally draw the landmarks on the image.</p>

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

<p>The findPosition method retrieves the positions of the hand landmarks and optionally draws them on the image.
</p>

```py
    def handType(self):
        if self.results.multi_handedness:
            for hand_info in self.results.multi_handedness:
                if hand_info.classification[0].label == "Left":
                    return "Left"
                else:
                    return "Right"
        return "Right"  # Default to right if not detected

```

<p>The handType method identifies whether the detected hand is left or right.
</p>

```py

    def findDistance(self, p1, p2, img, draw=True):
        x1, y1 = self.lmList[p1][1], self.lmList[p1][2]
        x2, y2 = self.lmList[p2][1], self.lmList[p2][2]
        cx, cy = (x1 + x2) // 2, (y1 + y2) // 2

        if draw:
            cv2.circle(img, (x1, y1), 15, (255, 0, 255), cv2.FILLED)
            cv2.circle(img, (x2, y2), 15, (255, 0, 255), cv2.FILLED)
            cv2.line(img, (x1, y1), (x2, y2), (255, 0, 255), 3)
            cv2.circle(img, (cx, cy), 15, (255, 0, 255), cv2.FILLED)

        length = math.hypot(x2 - x1, y2 - y1)
        return length, img, [x1, y1, x2, y2, cx, cy]

```

<h2>FingerCountingProject</h2>

<p>The FingerCountringProject script captures video from the webcam and uses the handDetector class to detect hands and count fingers.</p>

```py
import cv2
import time
import os
import HandTrackingModule as htm

```

<p>Imports necessary libraries and the hand tracking module.</p>

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

