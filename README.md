# Projectphase-01
# TITLE OF THE PROJECT :

# Computer Vision-Based Early Warning System for Potential Theft

## Small Description

The Computer Vision-Based Early Warning System for Potential Theft is a Python-based surveillance application designed to monitor a selected area through a camera, detect possible object removal, and send real-time warning notifications through Telegram.

## About

The Computer Vision-Based Early Warning System is designed to improve surveillance monitoring by identifying significant changes in a selected area of a live camera feed. Traditional CCTV systems often require continuous human observation to identify suspicious situations. This project helps reduce that effort by automatically monitoring an object's designated area and sending alerts when a possible removal is detected.

The system uses Python, OpenCV, and image-processing techniques to compare the selected region with a reference image captured when the object is present. When the visual difference remains above a defined threshold for a specified duration, the system generates a possible object-removal warning and sends it through a Telegram bot.

The application also detects when the selected area appears to return to its original visual state and sends a return notification. This allows monitoring to resume after the object returns. The system is intended as an early warning aid and does not independently confirm theft.

---

## Features

1. Live camera monitoring using OpenCV
2. Manual selection of a monitored object area
3. Reference-image-based change detection
4. Automatic warning when possible object removal is detected
5. Telegram notifications for object removal
6. Telegram notifications when the object appears to return
7. Repeated monitoring after the object returns
8. Configurable change threshold and confirmation duration
9. Simple keyboard controls to select the area, reset the reference, and exit
10. Support for webcam input and compatible camera streams

---

## Requirements

- **Operating System**
  - Windows 7 / 8 / 8.1 / 10 / 11 (64-bit)

- **Development Environment**
  - Python 3.8 or later

- **Programming Language**
  - Python

- **Libraries and Tools**
  - OpenCV (`opencv-python`) – Camera access and image processing
  - NumPy – Image-difference calculations
  - Requests – Telegram Bot API integration
  - Telegram Bot – Warning and status notifications

- **IDE**
  - Visual Studio Code

- **Hardware**
  - Webcam or compatible CCTV/IP camera
  - Internet connection for Telegram notifications

---

## System Architecture

The system follows a modular, camera-based monitoring architecture.

1. **Video Acquisition Module:** Captures frames from a webcam or compatible camera stream.
2. **Region Selection Module:** Allows the user to select a specific area around the object to be monitored.
3. **Reference Image Module:** Stores the initial appearance of the selected area as a baseline.
4. **Image Processing Module:** Converts frames to grayscale, applies Gaussian blur, and calculates image differences.
5. **Change Detection Module:** Measures the proportion of changed pixels and checks whether the change persists beyond the configured confirmation duration.
6. **Alert Management Module:** Determines when a possible removal or return event has occurred and manages alert timing.
7. **Telegram Notification Module:** Sends removal warnings and return updates using the Telegram Bot API.
8. **Display Module:** Shows the camera feed, selected area, change percentage, and current monitoring status.

---

## Results and Impact

The system demonstrates automated monitoring of a selected camera region and provides timely notifications when significant visual changes occur.

- **Object Removal Detection:** Identifies sustained visual changes that may indicate object removal.
- **Telegram Alerts:** Sends a warning message for a possible removal event.
- **Object Return Detection:** Sends a notification when the selected area becomes similar to its original reference again.
- **Continuous Monitoring:** Re-arms removal detection after a confirmed return.
- **Reduced Manual Monitoring:** Helps users notice potential events without continuously watching the camera feed.

The current implementation uses image-change detection rather than a trained theft-classification model. Its performance depends on camera stability, lighting, object placement, and the selected thresholds. Camera movement, shadows, lighting changes, or other objects entering the monitored region may cause false alerts.

Accuracy, precision, and recall have not been established through a formal evaluation and should be reported only after testing against labeled data.

---

## How to Run the Project

1. Download or clone this repository.
2. Install the required libraries:

   ```bash
   pip install -r requirements.txt
   ```

3. Open `config.py` and configure your Telegram bot token and chat ID. Keep these credentials private.
4. Set `CAMERA_SOURCE = 0` for a default webcam, or configure a compatible camera stream.
5. Start the application:

   ```bash
   python main.py
   ```

6. When the camera window opens, place the object in view and press **C** to select the area around it.
7. Remove the object to test the removal alert.
8. Return the object to the same area to test the return notification.

### Keyboard Controls

- **C** – Select the monitored area and establish a reference image.
- **R** – Reset the reference to the current camera view.
- **Q** – Quit the application.

---

## Applications

- Retail store surveillance
- Monitoring unattended bags or personal belongings
- Office and workplace security
- Laboratory and equipment monitoring
- Home and small-business surveillance

The system should be used as a supplementary warning tool, with human review of alerts before drawing conclusions.

---

# CODE IMPLEMENTATION:
```
import time
from datetime import datetime
import cv2
import numpy as np
import requests
import config

SOURCE = config.CAMERA_SOURCE
TOKEN = config.TELEGRAM_BOT_TOKEN
CHAT_ID = config.TELEGRAM_CHAT_ID
CHANGE_THRESHOLD = 28
CHANGED_FRACTION = 0.35
CONFIRM_SECONDS = 2.0
ALERT_COOLDOWN = 30.0


def telegram(message):
    if not TOKEN or not CHAT_ID or "PASTE_" in str(TOKEN) or "PASTE_" in str(CHAT_ID):
        print("Telegram not configured: edit config.py first.")
        return False
    try:
        response = requests.post(
            f"https://api.telegram.org/bot{TOKEN}/sendMessage",
            data={"chat_id": CHAT_ID, "text": message}, timeout=10)
        response.raise_for_status()
        data = response.json()
        if not data.get("ok"):
            print("Telegram API error:", response.text)
            return False
        print("Telegram message sent.")
        return True
    except Exception as exc:
        print("Telegram send failed:", exc)
        return False


def timestamp():
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")


def main():
    cap = cv2.VideoCapture(SOURCE)
    if not cap.isOpened():
        raise SystemExit("Could not open camera. Check CAMERA_SOURCE in config.py.")

    roi = None
    reference = None
    changed_since = None
    returned_since = None
    last_alert = 0.0
    object_state = "present"

    print("Keep camera fixed. Press C to select the bag area; R to reset baseline; Q to quit.")

    while True:
        ok, frame = cap.read()
        if not ok:
            print("No video frame received. Check camera/stream.")
            break

        view = frame.copy()
        status = "Press C to select area around bag" if reference is None else f"Bag state: {object_state.upper()}"
        color = (0, 200, 255) if reference is None else ((0, 0, 255) if object_state == "removed" else (0, 200, 0))

        if roi is not None:
            x, y, w, h = roi
            cv2.rectangle(view, (x, y), (x + w, y + h), color, 2)

        if reference is not None:
            x, y, w, h = roi
            crop = frame[y:y+h, x:x+w]
            if crop.size:
                current = cv2.cvtColor(crop, cv2.COLOR_BGR2GRAY)
                current = cv2.GaussianBlur(current, (5, 5), 0)
                diff = cv2.absdiff(current, reference)
                _, mask = cv2.threshold(diff, CHANGE_THRESHOLD, 255, cv2.THRESH_BINARY)
                fraction = np.count_nonzero(mask) / mask.size
                now = time.time()

                if object_state == "present":
                    if fraction >= CHANGED_FRACTION:
                        if changed_since is None:
                            changed_since = now
                        elif now - changed_since >= CONFIRM_SECONDS:
                            object_state = "removed"
                            changed_since = None
                            returned_since = None
                            status = "POSSIBLE REMOVAL - REVIEW"
                            color = (0, 0, 255)
                            if now - last_alert >= ALERT_COOLDOWN:
                                message = (
                                    "🚨 CCTV EARLY WARNING: Possible object removal detected.\n"
                                    f"Time: {timestamp()}\n"
                                    "Please review the footage. This is not proof of theft."
                                )
                                if telegram(message):
                                    last_alert = now
                    else:
                        changed_since = None
                        status = "Bag state: PRESENT"
                        color = (0, 200, 0)
                else:
                    if fraction < CHANGED_FRACTION:
                        if returned_since is None:
                            returned_since = now
                        elif now - returned_since >= CONFIRM_SECONDS:
                            object_state = "present"
                            returned_since = None
                            changed_since = None
                            status = "BAG RETURNED"
                            color = (0, 200, 0)
                            telegram(
                                "✅ CCTV UPDATE: The object appears to have returned to the monitored area.\n"
                                f"Time: {timestamp()}\n"
                                "Please verify the camera view."
                            )
                    else:
                        returned_since = None
                        status = "BAG REMOVED - WAITING FOR RETURN"
                        color = (0, 0, 255)

                cv2.putText(view, f"Area change: {fraction:.0%}", (10, 55),
                            cv2.FONT_HERSHEY_SIMPLEX, 0.65, color, 2)

        cv2.putText(view, status, (10, 28), cv2.FONT_HERSHEY_SIMPLEX, 0.65, color, 2)
        cv2.imshow("CCTV Object-Removal Early Warning", view)
        key = cv2.waitKey(1) & 0xFF

        if key in (ord("c"), ord("C")):
            ok2, frame2 = cap.read()
            if not ok2:
                continue
            selected = cv2.selectROI("Select tightly around the bag, then press ENTER",
                                     frame2, showCrosshair=True, fromCenter=False)
            cv2.destroyWindow("Select tightly around the bag, then press ENTER")
            x, y, w, h = map(int, selected)
            if w > 0 and h > 0:
                roi = (x, y, w, h)
                crop = frame2[y:y+h, x:x+w]
                reference = cv2.cvtColor(crop, cv2.COLOR_BGR2GRAY)
                reference = cv2.GaussianBlur(reference, (5, 5), 0)
                changed_since = None
                returned_since = None
                object_state = "present"
                last_alert = 0.0
                print("Baseline saved. Remove the bag to test.")
                telegram("CCTV object-removal monitor started. Selected area is being monitored.")

        elif key in (ord("r"), ord("R")) and roi is not None:
            ok2, frame2 = cap.read()
            if ok2:
                x, y, w, h = roi
                crop = frame2[y:y+h, x:x+w]
                reference = cv2.cvtColor(crop, cv2.COLOR_BGR2GRAY)
                reference = cv2.GaussianBlur(reference, (5, 5), 0)
                changed_since = None
                returned_since = None
                object_state = "present"
                print("Baseline reset to current view.")
                telegram("🔄 CCTV monitor update: Baseline reset to the current camera view.\n"
                         f"Time: {timestamp()}")

        elif key in (ord("q"), ord("Q")):
            break

    cap.release()
    cv2.destroyAllWindows()


if __name__ == "__main__":
    main()

```

# OUTPUT:

<img width="830" height="712" alt="Screenshot 2026-11-02 213715" src="https://github.com/user-attachments/assets/00ac90cd-4844-449d-a17f-36c80972818f" />

<img width="876" height="747" alt="Screenshot 2026-11-02 213858" src="https://github.com/user-attachments/assets/f4f072ca-9fbf-42d3-9eca-d2c0b1d8714d" />

<img width="852" height="767" alt="Screenshot 2026-11-02 213950" src="https://github.com/user-attachments/assets/5a47b7f9-38aa-409c-908f-e7ce7c7e7dce" />

<img width="717" height="1600" alt="WhatsApp Image 2026-11-02 at 9 43 36 PM" src="https://github.com/user-attachments/assets/bb6188bb-1552-48c2-a014-cd4639ed0116" />


## References

1. OpenCV Documentation – https://docs.opencv.org/
2. NumPy Documentation – https://numpy.org/doc/
3. Python Documentation – https://docs.python.org/3/
4. Telegram Bot API Documentation – https://core.telegram.org/bots/api
5. [1] G. A. Martínez-Mascorro et al., “Suspicious Behavior Detection on Shoplifting Cases for Crime Prevention by Using 3D Convolutional Neural Networks,” arXiv:2005.02142, 2020.
6. [2] U.-J. Gim et al., “An Automatic Shoplifting Detection from Surveillance Videos,” AAAI, vol. 34, no. 10, pp. 13795–13796, 2020.
7. [3] N. Rashvand et al., “Shopformer: Transformer-Based Framework for Detecting Shoplifting via Human Pose,” CVPR Workshops, pp. 5798–5807, 2025.


