
<h1 align="center">FingerCounter</h1>

<p align="center">
  <a href="https://www.youtube.com/watch?v=h8sp7vFeV7c"><img src="https://i.imgur.com/h7dbXn8.gif" alt="YouTube Demonstration" width="800"></a>
</p>

<h2>Description</h2>

<p>The goal of this project was to develop a program that used AI to recognize human hands through feedback of a webcamera and recognize how many fingers were being help up. This worked by identifying points in the hand and determining based on each finger if the tip of the finger was higher that another point of the finger.</p>

<h2>Languages and Utilities Used</h2>

<ul>
  <li><b>Python</b></li>
  <li><b>MediaPipe</b></li>
  <li><b>OpenCV</b></li>
</ul>

<h2>Environments Used</h2>

<ul>
  <li><b>Windows 11</b></li>
  <li><b>PyCharm</b></li>
</ul>

<h2>
<a href="https://github.com/pedromussi1/VolumeCV/blob/main/READCODE.md">Code Breakdown Here!</a>
</h2>

<h2>Project Walk-through</h2>

<p>Download files, install OpenCV and mediapipe into Python Interpreter. Run FingerCountingProject.py file.</p>

<h3>Hand Tracking</h3>

<p align="center">
  <kbd><img src="https://ai.google.dev/static/edge/mediapipe/images/solutions/hand-landmarks.png" alt="HandTracking"></kbd>
</p>

<p>The first step is to detect hands in the image and draw landmarks of that hands so that data can be compared and manipulated. This is possible with the help of a technology developed by Google called MediaPipe Hand Landmarker. Not only does this technology perfectly detect hands, it also adds unique points to each joint and enumerates them.  </p>

<h3>Manipulating Landmarks</h3>

<p align="center">
  <kbd><img src="https://i.imgur.com/qBrfm5J.png" alt="ManipulatingLandmarks"></kbd>
</p>

<p>Since there are multiple points in each finger, x and y axis of a finger can also be determined, thus enabling us to know if a finger is "up" or "down". In case a finger is up, it will be added to the total of fingers up.</p>

<h3>Handling the thumb</h3>

<p align="center">
  <kbd><img src="https://i.imgur.com/eF7KLZT.png" alt="VolumeSetting"></kbd>
</p>

<p>An important step in the process of counting all the fingers that were up was handling how to indentify a thumb as being "up". As we all know, when showing your hand your thumb points away form the palm of your hand, but not necessarily "up" in the plane graph. That is why it has to be handled differently. In my case, I handled it by comparing it in a horizontal sense, meaning that being on the right/left of the base of the thumb. Even so, there was still the need to tackle how to change that logic between right and left hands.</p>

<h3>HandType Differentiation</h3>

<p align="center">
  <kbd><img src="https://i.imgur.com/8RXf0z4.png" alt="VolumeSetting"></kbd>
</p>

<p>Luckily for us, MediaPipe has a built in function that care of that logic called handType. With handType, the program can determine if a hand is right or left, and we can identify if the thumb is up or down based on that information.</p>

