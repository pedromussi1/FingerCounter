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

<p>The handDetector class encapsulates functionalities related to hand detection and tracking using the MediaPipe framework.
</p>

<p>mode: Specifies whether the detection should be static or dynamic (default: False).</p>

<p>maxHands: Maximum number of hands to detect (default: 2).</p>

<p>detectionCon: Minimum confidence threshold for detecting a hand (default: 0.5).</p>

<p>trackCon: Minimum confidence threshold for tracking a hand (default: 0.5).</p>

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

<p>It calculates the pixel coordinates of each landmark and optionally draws them on the image.</p>

<p>img: Input image (in BGR format).</p>

<p>draw: Boolean flag to enable/disable drawing landmarks on the image (default: True).</p>

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

<p>The findPosition method retrieves the positions of the hand landmarks and optionally draws them on the image.</p>

<p>handNo: Index of the hand to track (default: 0 for the first detected hand).</p>

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

<p>The handType method identifies whether the detected hand is left or right.</p>

<p>Returns: "Left" if the hand is identified as left-handed, "Right" otherwise.</p>


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

<p>Purpose: Calculates the Euclidean distance between two specified landmarks and optionally draws this distance on the image.</p>

<p>p1, p2: Indices of the landmarks (points) in self.lmList.</p>

<h2>FingerCountingProject</h2>

<p>The FingerCountringProject script captures video from the webcam and uses the handDetector class to detect hands and count fingers.</p>

```py
import cv2
import time
import os
import HandTrackingModule as htm

```

<p>Imports necessary libraries and the hand tracking module.</p>


```py
wCam, hCam = 640, 480
cap = cv2.VideoCapture(0)
cap.set(3, wCam)
cap.set(4, hCam)
pTime = 0

```

<p>Initializes the webcam settings and the frame size.
</p>

```py
detector = htm.handDetector(detectionCon=0.75)
tipIds = [4, 8, 12, 16, 20]

```

<p>Creates a hand detector object and sets the tip IDs of the fingers.</p>

```py

while True:
    success, img = cap.read()
    img = detector.findHands(img)
    lmList, bbox = detector.findPosition(img, draw=False)

    if len(lmList) >= 9:
        fingers = []
        handType = detector.handType()

        # Thumb
        if handType == "Right":
            if lmList[tipIds[0]][1] < lmList[tipIds[0] - 1][1]:
                fingers.append(1)
            else:
                fingers.append(0)
        else:  # Left hand
            if lmList[tipIds[0]][1] > lmList[tipIds[0] - 1][1]:
                fingers.append(1)
            else:
                fingers.append(0)

        # 4 Fingers
        for id in range(1, 5):
            if lmList[tipIds[id]][2] < lmList[tipIds[id]-2][2]:
                fingers.append(1)
            else:
                fingers.append(0)

        totalFingers = fingers.count(1)

        cv2.rectangle(img, (20, 255), (170, 425), (0, 255, 0), cv2.FILLED)
        cv2.putText(img, str(totalFingers), (45, 400), cv2.FONT_HERSHEY_PLAIN, 10, (255, 0, 0), 25)

    cTime = time.time()
    fps = 1/(cTime-pTime)
    pTime = cTime

    # cv2.putText(img, f'FPS: {int(fps)}',(400,70), cv2.FONT_HERSHEY_PLAIN,3,(255,0,0),3)

    cv2.imshow("Image", img)
    cv2.waitKey(1)


```

<p>Continuously captures frames from the webcam (cap.read()), processes them using the handDetector class methods (findHands, findPosition), counts the number of fingers held up, calculates the frame rate (fps), and displays the results on the image.</p>

