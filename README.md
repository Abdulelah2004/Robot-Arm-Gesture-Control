# Robot Arm Gesture Control

Control a five-servo robotic hand using hand gestures detected by a webcam. The Python application uses OpenCV to segment a hand by HSV color, identify its contour, estimate the number of extended fingers, and send a compact command to an Arduino over serial communication.

> **Hardware project:** This repository contains the Python controller, Arduino sketch, and the supplied STL model files. The exact wiring, servo model, and mechanical assembly details should be verified against your own build before powering the robot.

## Features

- Real-time webcam input using OpenCV.
- Adjustable HSV thresholds through trackbars.
- Hand segmentation with Gaussian blur and morphological filtering.
- Contour and convexity-defect-based finger-count estimation.
- Five-finger command packets sent to an Arduino over serial.
- Optional detection-only mode without an Arduino connection.
- Supplied 3D-printable STL files.

## Project structure

```text
robot-arm-gesture-control/
├── main.py                 # Webcam loop and application entry point
├── utils.py                # Image processing, gesture detection, serial helper
├── ArduinoSerial.ino       # Arduino servo controller
├── requirements.txt        # Python dependencies
├── .gitignore
├── README.md
└── models/
    ├── Base.STL
    └── BaseCover.STL
```

## How it works

1. The webcam captures a frame.
2. The frame is blurred and converted from BGR to HSV.
3. HSV trackbars define the color range used to isolate the hand.
4. Morphological opening and closing reduce image noise.
5. The largest contour above the configured area threshold is treated as the hand.
6. Convexity defects are used to estimate the number of extended fingers.
7. A five-bit command is sent to the Arduino.
8. The Arduino maps each bit to a servo position.

### Gesture command mapping

The serial packet contains a `$` prefix followed by five bits. The bit order is **pinky, ring, middle, index, thumb**.

| Gesture estimate | Packet | Meaning |
|---|---|---|
| 0 fingers | `$00000` | All fingers closed |
| 1 finger | `$00010` | Index servo open |
| 2 fingers | `$00110` | Middle and index open |
| 3 fingers | `$01110` | Ring, middle, and index open |
| 4 fingers | `$11110` | Pinky, ring, middle, and index open |
| 5 fingers | `$11111` | All fingers open |

The gesture classifier is a contour-based heuristic. Its accuracy depends on camera angle, lighting, background, hand orientation, and HSV threshold calibration.

## Requirements

- Python 3.9 or newer
- A webcam
- Arduino-compatible board
- Five hobby servos and a compatible mechanical hand
- USB serial connection between the computer and Arduino
- Arduino IDE to upload the sketch

Python packages:

- `opencv-python`
- `numpy`
- `pyserial`

## Installation

Clone the repository:

```bash
git clone https://github.com/<your-username>/robot-arm-gesture-control.git
cd robot-arm-gesture-control
```

Create and activate a virtual environment (recommended):

**Windows**
```bash
python -m venv .venv
.venv\Scripts\activate
```

**macOS / Linux**
```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

## Hardware setup

1. Assemble the robotic hand using the STL files as a reference.
2. Connect the five servo signal wires to the Arduino pins configured in `ArduinoSerial.ino`.
3. Connect the Arduino to your computer using USB.
4. Upload `ArduinoSerial.ino` using the Arduino IDE.
5. Identify the serial port assigned to the Arduino.
6. **Power the servos using a suitable external supply.** Do not power multiple servos directly from the Arduino 5 V pin. Connect the external supply ground to Arduino GND.

### Default servo pin mapping

| Finger | Arduino pin |
|---|---:|
| Thumb | 9 |
| Index | 10 |
| Pinky | 11 |
| Ring | 12 |
| Middle | 13 |

The sketch's packet order is pinky, ring, middle, index, thumb. Check the actual servo-to-finger linkage before operation.

### Mode input

The supplied sketch retains the original A0 mode-selection behavior:

- A0 reads LOW at startup: serial gesture-control mode.
- A0 reads non-zero at startup: manual mode, where A0 is read to open or close all fingers.

The original circuit's switch or input wiring was not included with the source files. Configure A0 with an appropriate switch and pull-up/pull-down arrangement for your hardware. Do not leave a digital input floating.

## Run the application

Windows example:

```bash
python main.py --camera 0 --port COM4
```

macOS / Linux example:

```bash
python main.py --camera 0 --port /dev/ttyACM0
```

Run without connecting an Arduino (gesture detection only):

```bash
python main.py --camera 0 --no-serial
```

### Command-line options

| Option | Default | Description |
|---|---|---|
| `--camera` | `0` | Webcam index |
| `--port` | `COM4` | Arduino serial port |
| `--baud` | `9600` | Serial baud rate |
| `--width` | `640` | Camera frame width |
| `--height` | `480` | Camera frame height |
| `--brightness` | `230` | Camera brightness setting |
| `--crop X Y WIDTH HEIGHT` | `100 100 400 300` | Region of interest |
| `--min-contour-area` | `1000` | Minimum contour area |
| `--no-serial` | off | Disable Arduino connection |
| `--debug` | off | Enable debug logs |

Press **Q** or **Esc** in the video window to quit.

## HSV calibration

The application opens an **HSV Controls** window with six sliders:

- HUE MIN / HUE MAX
- SAT MIN / SAT MAX
- VALUE MIN / VALUE MAX

Adjust these sliders until the hand is isolated in the binary mask and most of the background is removed. The default values are only starting points; they may not work for every skin tone, camera, or lighting condition.

For more reliable segmentation:

- Use consistent, diffuse lighting.
- Keep the background visually different from the selected hand color.
- Avoid strong shadows and reflections.
- Keep the hand inside the green region-of-interest rectangle.
- Adjust the crop and minimum contour area if needed.

## Troubleshooting

**Camera does not open**
- Try another camera index, such as `--camera 1`.
- Close other applications that may be using the webcam.
- Check operating-system camera permissions.

**Arduino connection fails**
- Confirm the correct port in Arduino IDE or your operating system.
- Close the Arduino Serial Monitor before running the Python program.
- Check the USB cable and board connection.
- Confirm that both sides use the same baud rate (`9600` by default).

**Hand is not detected**
- Adjust the HSV sliders.
- Improve lighting and background contrast.
- Keep the hand within the region of interest.
- Lower `--min-contour-area` if the detected hand contour is small.

**Servos move in the wrong direction or fingers do not match**
- Check the servo order and mechanical linkage.
- Change the servo angle mapping in `ArduinoSerial.ino` if your mechanism requires different positions.
- Test with the hand unloaded and keep fingers clear of pinch points.

**Serial port is busy**
- Close Arduino Serial Monitor and any other program using the same port.

## Safety

- Test the mechanism without placing fingers near moving joints.
- Use a regulated external power supply sized for the servos.
- Ensure all grounds are connected together.
- Add suitable mechanical stops to prevent servo over-travel.
- Disconnect power before changing wiring.

## Limitations

- The current vision method is color-based, not a trained hand-pose or gesture-recognition model.
- Finger counting relies on contour geometry and may fail for rotated, occluded, or overlapping fingers.
- The supplied source does not specify the exact servo models, power-supply rating, or complete circuit schematic.
- Servo endpoints are currently configured as 0° and 180°; calibrate these values for the mechanical design.

## 3D model files

The `models/` directory contains the supplied `Base.STL` and `BaseCover.STL` files. Inspect dimensions and fit in your preferred slicer/CAD tool before printing.

## Acknowledgements

This project builds on a webcam-based OpenCV hand-contour approach. The original README referenced the Robot Arm Gesture Control project by Murtaza Hassan. Adapt the acknowledgement and attribution to match the source and license terms of the code and models you use.

## License

No license was included with the supplied files. Add a license only after confirming that you have the right to distribute the source code and 3D model files under that license.
