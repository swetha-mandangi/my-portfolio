# flask@dashboard.py
import time
import threading
from flask import Flask, render_template, request, jsonify, Response
from flask_socketio import SocketIO
from xarm.wrapper import XArmAPI   # Lite6 SDK
import cv2

ROBOT_IP = "192.168.1.152"

app = Flask(__name__)
app.config['SECRET_KEY'] = 'lite6secret'
socketio = SocketIO(app, cors_allowed_origins="*", async_mode='threading')

arm = None
arm_lock = threading.Lock()

# ------------------ Camera globals ------------------
camera = None
camera_lock = threading.Lock()
latest_frame = None
camera_running = False

def camera_thread_func():
    """Continuously read frames from the MacBook webcam into latest_frame."""
    global camera, latest_frame, camera_running
    with camera_lock:
        if camera is None:
            camera = cv2.VideoCapture(0)
            camera.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
            camera.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)
    camera_running = True
    while camera_running:
        with camera_lock:
            ok, frame = camera.read()
        if not ok or frame is None:
            time.sleep(0.1)
            continue
        frame = cv2.flip(frame, 1)
        ret, jpeg = cv2.imencode('.jpg', frame)
        latest_frame = jpeg.tobytes() if ret else None
        time.sleep(0.03)  # ~30 FPS

def gen_video_stream():
    """Generator function for MJPEG streaming."""
    global latest_frame
    while True:
        frame = latest_frame
        if frame is None:
            # tiny black frame
            frame = b'\xff\xd8' + b'\xff'*100 + b'\xff\xd9'
        yield (b'--frame\r\n'
               b'Content-Type: image/jpeg\r\n\r\n' + frame + b'\r\n')
        time.sleep(0.03)

# ------------------ Robot connection + polling ------------------
def connect_robot():
    """Connect to Lite6 at startup."""
    global arm
    try:
        with arm_lock:
            arm = XArmAPI(ROBOT_IP)
        print(f"[OK] Connected to robot at {ROBOT_IP}")
    except Exception as e:
        print("[ERROR] Could not connect:", e)
        arm = None

def poll_robot():
    """Continuously broadcast robot angles + position to dashboard."""
    while True:
        try:
            if arm is not None:
                with arm_lock:
                    # Joint angles
                    try:
                        angles = arm.angles
                    except:
                        ok, angles = arm.get_servo_angle()
                        if not ok:
                            angles = [0]*6

                    # TCP Position
                    try:
                        pos = arm.position
                    except:
                        ok2, pos = arm.get_position()
                        if not ok2:
                            pos = [0]*6

                    # Moving state
                    try:
                        moving = arm.get_is_moving()[1]
                    except:
                        moving = False

                # Broadcast to dashboard
                socketio.emit("robot_state", {
                    "angles": angles,
                    "position": pos,
                    "moving": bool(moving)
                })
        except Exception as e:
            socketio.emit("robot_error", {"error": str(e)})
        time.sleep(0.15)  # ~7 updates/sec

# ------------------ Flask routes ------------------
@app.route('/')
def index():
    return render_template("index.html")

@app.route('/voice', methods=['POST'])
def voice_cmd():
    """Receive speech text from browser and move robot joints."""
    data = request.json or {}
    spoken = data.get('text', '').lower()
    
    heard = f"Executing: {spoken}"
    
    # Emit log to dashboard
    socketio.emit("voice_log", {"said": spoken, "heard": heard})

    # --- Parse joint command ---
    import re
    from word2number import w2n

    joint_num = angle = None

    # Format: "move joint 2 to 45"
    match = re.search(r"joint (\d+) to (\d+)", spoken)
    if match:
        joint_num = int(match.group(1))
        angle = int(match.group(2))
    else:
        # Parse numbers as words: "joint two to forty five"
        nums = re.findall(r"\b\w+\b", spoken)
        numbers = []
        for n in nums:
            try:
                numbers.append(int(n))
            except:
                try:
                    numbers.append(w2n.word_to_num(n))
                except:
                    continue
        if len(numbers) >= 2:
            joint_num, angle = numbers[0], numbers[1]

    # Move the joint if valid
    if joint_num and angle:
        with arm_lock:
            current_angles = arm.get_servo_angle()[1][:6]
            current_angles[joint_num-1] = angle
            arm.set_servo_angle(servo_id=None, angle=current_angles, is_radian=False)

    return jsonify({"status": "ok", "heard": heard})



@app.route('/joints')
def joints():
    global arm, arm_lock
    if arm is None:
        return jsonify({"joints": [0]*6})
    with arm_lock:
        try:
            angles = arm.angles
        except:
            ok, angles = arm.get_servo_angle()
            if not ok:
                angles = [0]*6
    return jsonify({"joints": angles})

@app.route('/video_feed')
def video_feed():
    return Response(gen_video_stream(),
                    mimetype='multipart/x-mixed-replace; boundary=frame')

# ------------------ SocketIO events ------------------
@socketio.on("connect")
def handle_connect():
    print("[Client Connected]")

@socketio.on("move_joint")
def handle_move_joint(data):
    """Receive joint movement from dashboard sliders."""
    joint = int(data.get("joint", 1))
    angle = float(data.get("angle", 0))
    if arm is not None:
        with arm_lock:
            try:
                arm.set_servo_angle(joint, angle, is_radian=False)
            except Exception as e:
                print(f"[ERROR] Failed to move joint {joint}: {e}")

# ------------------ App startup ------------------
if __name__ == "__main__":
    connect_robot()
    threading.Thread(target=poll_robot, daemon=True).start()
    threading.Thread(target=camera_thread_func, daemon=True).start()
    socketio.run(app, host="0.0.0.0", port=5050)
