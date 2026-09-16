from pathlib import Path
import textwrap, zipfile, os, json

root = Path("/mnt/data/smart_helmet")
if root.exists():
    import shutil
    shutil.rmtree(root)

files = {}

files["README.md"] = r"""# Raspberry Pi Smart Helmet

A modular, safety-focused Raspberry Pi prototype for a hackathon/college project.

## Architecture

Sensors -> sensor abstraction -> EventBus -> centralized SafetyManager
Camera -> vision modules -> EventBus
GPS -> location service -> speed/emergency services
EventBus -> logging/database/audio/emergency/dashboard

The Raspberry Pi is the primary computer. ESP32 is not required.

## Recommended hardware

- Raspberry Pi 5, preferably 8 GB for a multi-module prototype
- Active Cooler
- Raspberry Pi Camera Module 3 or Raspberry Pi AI Camera
- MPU6050 on I2C, or replace `sensors/imu.py` with another IMU driver
- USB/UART GNSS receiver
- Optional SIM800/SIM900-class GSM modem for SMS (requires a suitable modem power supply and SIM/service)
- USB microphone
- USB audio adapter/speaker or powered speaker
- Momentary push button
- LED and resistor
- Optional buzzer through a suitable transistor/driver if its current exceeds GPIO capability
- Good quality regulated 5 V power source; do not power a cellular modem from a GPIO pin

For heavier object detection, Raspberry Pi 5 + Raspberry Pi AI HAT+ is the recommended expansion. Current Raspberry Pi documentation says AI HAT+ provides Hailo acceleration and integrates with rpicam/Picamera2-supported vision workloads.

## Safety

This is a prototype, not a medical, automotive, emergency-service, or certified crash-detection product.
Do not test by deliberately crashing, riding dangerously, or creating hazardous conditions.
Use the simulation/demo commands for presentations.

## Important calibration

Thresholds in `.env.example` are starting values only. Real calibration must be performed with safe, controlled tests and compared with normal helmet movement. A single acceleration spike never directly triggers an alert: the safety manager uses multiple signals and a confirmation countdown.

## Quick start

1. Install Raspberry Pi OS 64-bit.
2. Run:
   `sudo bash scripts/install.sh`
3. Copy `.env.example` to `.env` and edit it.
4. Create/activate the venv if install.sh did not already do so:
   `source .venv/bin/activate`
5. Run hardware self-test:
   `python scripts/health_check.py`
6. Start:
   `python main.py`
7. Demo:
   `python -m scripts.demo crash`
   `python -m scripts.demo overspeed`
   `python -m scripts.demo drowsiness`

## Current implementation

Core functionality is implemented without requiring every optional device:
- centralized configuration
- event bus
- SQLite event database
- IMU driver
- NMEA GPS parser
- GPS-based speed monitor
- crash/fall state machine
- emergency countdown/cancel
- GSM SMS notifier
- camera abstraction
- OpenCV Haar-based drowsiness prototype
- optional OpenCV DNN object detection
- priority audio queue
- health monitor
- local Flask dashboard
- simulation/demo mode
- unit tests
- systemd service

Optional AI acceleration can be added later without changing the safety core.
"""

files[".env.example"] = r"""DEBUG=false
LOG_LEVEL=INFO
DB_PATH=data/smart_helmet.db

# GPIO - BCM numbering
CANCEL_BUTTON_GPIO=17
BUZZER_GPIO=18
STATUS_LED_GPIO=23

# Interfaces
I2C_BUS=1
GPS_PORT=/dev/serial0
GPS_BAUD=9600
GSM_PORT=/dev/ttyUSB0
GSM_BAUD=9600

# Feature switches
ENABLE_IMU=true
ENABLE_GPS=true
ENABLE_CAMERA=true
ENABLE_DROWSINESS=true
ENABLE_OBJECT_DETECTION=false
ENABLE_GSM=false
ENABLE_AUDIO=true
ENABLE_DASHBOARD=true

# Safety thresholds - MUST be calibrated safely
IMPACT_THRESHOLD_G=3.5
DECELERATION_THRESHOLD_G=2.5
ROTATION_THRESHOLD_DPS=250.0
FALL_ANGLE_THRESHOLD_DEG=60.0
INACTIVITY_SECONDS=5.0
CONFIRMATION_SECONDS=15.0

# Speed
CITY_SPEED_LIMIT_KMH=50
HIGHWAY_SPEED_LIMIT_KMH=80
SPEED_WARNING_MARGIN_KMH=5
SPEED_WARNING_COOLDOWN_SECONDS=30

# Drowsiness prototype
EAR_THRESHOLD=0.22
EYE_CLOSED_DURATION_SECONDS=2.0
DROWSINESS_WARNING_SECONDS=4.0
DROWSINESS_CRITICAL_SECONDS=8.0
VISION_FPS=8

# Camera
CAMERA_WIDTH=640
CAMERA_HEIGHT=480
CAMERA_FPS=15
OBJECT_INFERENCE_FPS=3
VEHICLE_MODEL_PATH=models/vehicle.onnx
VEHICLE_CLASSES_PATH=models/classes.txt
VEHICLE_CONFIDENCE=0.45

# Emergency
EMERGENCY_CONTACTS=+910000000000
EMERGENCY_MESSAGE_PREFIX=SMART HELMET EMERGENCY ALERT

# Audio
AUDIO_VOLUME=0.9
TTS_RATE=165

# Storage
EVENT_RETENTION_DAYS=30
LOW_DISK_MB=500
EVIDENCE_ENABLED=false
EVIDENCE_SECONDS=10

# Dashboard
DASHBOARD_HOST=127.0.0.1
DASHBOARD_PORT=8080

# Demo switches
SIMULATE_CRASH=false
SIMULATE_OVERSPEED=false
SIMULATE_DROWSINESS=false
SIMULATE_GPS=false
"""

files["requirements.txt"] = r"""python-dotenv>=1.0,<2
smbus2>=0.5,<1
pyserial>=3.5,<4
gpiozero>=2.0,<3
psutil>=6,<8
Flask>=3,<4
numpy>=1.26,<3
opencv-python-headless>=4.10,<5
pyttsx3>=2.90,<3

# Optional voice recognition:
# vosk>=0.3.45
# sounddevice>=0.5,<1

# Picamera2 should normally be installed with Raspberry Pi OS packages,
# not pip, because it is integrated with libcamera.
"""

files["config.py"] = r'''from __future__ import annotations

import os
from dataclasses import dataclass, field
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent


def _bool(name: str, default: bool) -> bool:
    value = os.getenv(name)
    if value is None:
        return default
    return value.strip().lower() in {"1", "true", "yes", "on"}


def _float(name: str, default: float) -> float:
    try:
        return float(os.getenv(name, default))
    except (TypeError, ValueError):
        return default


def _int(name: str, default: int) -> int:
    try:
        return int(os.getenv(name, default))
    except (TypeError, ValueError):
        return default


@dataclass(frozen=True)
class Config:
    debug: bool = _bool("DEBUG", False)
    log_level: str = os.getenv("LOG_LEVEL", "INFO")
    db_path: Path = Path(os.getenv("DB_PATH", "data/smart_helmet.db"))

    cancel_button_gpio: int = _int("CANCEL_BUTTON_GPIO", 17)
    buzzer_gpio: int = _int("BUZZER_GPIO", 18)
    status_led_gpio: int = _int("STATUS_LED_GPIO", 23)

    i2c_bus: int = _int("I2C_BUS", 1)
    gps_port: str = os.getenv("GPS_PORT", "/dev/serial0")
    gps_baud: int = _int("GPS_BAUD", 9600)
    gsm_port: str = os.getenv("GSM_PORT", "/dev/ttyUSB0")
    gsm_baud: int = _int("GSM_BAUD", 9600)

    enable_imu: bool = _bool("ENABLE_IMU", True)
    enable_gps: bool = _bool("ENABLE_GPS", True)
    enable_camera: bool = _bool("ENABLE_CAMERA", True)
    enable_drowsiness: bool = _bool("ENABLE_DROWSINESS", True)
    enable_object_detection: bool = _bool("ENABLE_OBJECT_DETECTION", False)
    enable_gsm: bool = _bool("ENABLE_GSM", False)
    enable_audio: bool = _bool("ENABLE_AUDIO", True)
    enable_dashboard: bool = _bool("ENABLE_DASHBOARD", True)

    impact_threshold_g: float = _float("IMPACT_THRESHOLD_G", 3.5)
    deceleration_threshold_g: float = _float("DECELERATION_THRESHOLD_G", 2.5)
    rotation_threshold_dps: float = _float("ROTATION_THRESHOLD_DPS", 250)
    fall_angle_threshold_deg: float = _float("FALL_ANGLE_THRESHOLD_DEG", 60)
    inactivity_seconds: float = _float("INACTIVITY_SECONDS", 5)
    confirmation_seconds: float = _float("CONFIRMATION_SECONDS", 15)

    city_speed_limit_kmh: float = _float("CITY_SPEED_LIMIT_KMH", 50)
    highway_speed_limit_kmh: float = _float("HIGHWAY_SPEED_LIMIT_KMH", 80)
    speed_warning_margin_kmh: float = _float("SPEED_WARNING_MARGIN_KMH", 5)
    speed_warning_cooldown_seconds: float = _float("SPEED_WARNING_COOLDOWN_SECONDS", 30)

    ear_threshold: float = _float("EAR_THRESHOLD", 0.22)
    eye_closed_duration_seconds: float = _float("EYE_CLOSED_DURATION_SECONDS", 2)
    drowsiness_warning_seconds: float = _float("DROWSINESS_WARNING_SECONDS", 4)
    drowsiness_critical_seconds: float = _float("DROWSINESS_CRITICAL_SECONDS", 8)
    vision_fps: int = _int("VISION_FPS", 8)

    camera_width: int = _int("CAMERA_WIDTH", 640)
    camera_height: int = _int("CAMERA_HEIGHT", 480)
    camera_fps: int = _int("CAMERA_FPS", 15)
    object_inference_fps: int = _int("OBJECT_INFERENCE_FPS", 3)
    vehicle_model_path: Path = Path(os.getenv("VEHICLE_MODEL_PATH", "models/vehicle.onnx"))
    vehicle_classes_path: Path = Path(os.getenv("VEHICLE_CLASSES_PATH", "models/classes.txt"))
    vehicle_confidence: float = _float("VEHICLE_CONFIDENCE", 0.45)

    emergency_contacts: tuple[str, ...] = field(
        default_factory=lambda: tuple(
            x.strip() for x in os.getenv("EMERGENCY_CONTACTS", "").split(",") if x.strip()
        )
    )
    emergency_message_prefix: str = os.getenv(
        "EMERGENCY_MESSAGE_PREFIX", "SMART HELMET EMERGENCY ALERT"
    )

    audio_volume: float = _float("AUDIO_VOLUME", 0.9)
    tts_rate: int = _int("TTS_RATE", 165)

    event_retention_days: int = _int("EVENT_RETENTION_DAYS", 30)
    low_disk_mb: int = _int("LOW_DISK_MB", 500)
    evidence_enabled: bool = _bool("EVIDENCE_ENABLED", False)
    evidence_seconds: int = _int("EVIDENCE_SECONDS", 10)

    dashboard_host: str = os.getenv("DASHBOARD_HOST", "127.0.0.1")
    dashboard_port: int = _int("DASHBOARD_PORT", 8080)

    simulate_crash: bool = _bool("SIMULATE_CRASH", False)
    simulate_overspeed: bool = _bool("SIMULATE_OVERSPEED", False)
    simulate_drowsiness: bool = _bool("SIMULATE_DROWSINESS", False)
    simulate_gps: bool = _bool("SIMULATE_GPS", False)

    def validate(self) -> list[str]:
        errors = []
        if self.confirmation_seconds <= 0:
            errors.append("CONFIRMATION_SECONDS must be > 0")
        if self.impact_threshold_g <= 0:
            errors.append("IMPACT_THRESHOLD_G must be > 0")
        if self.fall_angle_threshold_deg <= 0 or self.fall_angle_threshold_deg >= 180:
            errors.append("FALL_ANGLE_THRESHOLD_DEG must be between 0 and 180")
        if self.city_speed_limit_kmh <= 0 or self.highway_speed_limit_kmh <= 0:
            errors.append("speed limits must be > 0")
        if self.enable_gsm and not self.emergency_contacts:
            errors.append("ENABLE_GSM=true requires EMERGENCY_CONTACTS")
        if not 0 <= self.audio_volume <= 1:
            errors.append("AUDIO_VOLUME must be 0..1")
        return errors
'''

files["core/event_bus.py"] = r'''from __future__ import annotations

import queue
import threading
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any, Callable


@dataclass(slots=True)
class Event:
    event_type: str
    severity: str = "INFO"
    payload: dict[str, Any] = field(default_factory=dict)
    timestamp: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )


class EventBus:
    """Thread-safe publish/subscribe bus. Subscribers must be quick or queue work."""

    def __init__(self) -> None:
        self._subscribers: dict[str, list[Callable[[Event], None]]] = {}
        self._lock = threading.RLock()

    def subscribe(self, event_type: str, callback: Callable[[Event], None]) -> None:
        with self._lock:
            self._subscribers.setdefault(event_type, []).append(callback)

    def publish(self, event: Event) -> None:
        with self._lock:
            callbacks = list(self._subscribers.get(event.event_type, []))
            callbacks += list(self._subscribers.get("*", []))
        for callback in callbacks:
            try:
                callback(event)
            except Exception:
                # A faulty subscriber must never stop the publisher.
                continue


class EventQueue:
    """Useful for components that need asynchronous event consumption."""

    def __init__(self, maxsize: int = 500) -> None:
        self.queue: queue.Queue[Event] = queue.Queue(maxsize=maxsize)

    def put(self, event: Event) -> None:
        try:
            self.queue.put_nowait(event)
        except queue.Full:
            # Dropping a low-level event is preferable to deadlocking safety logic.
            try:
                self.queue.get_nowait()
                self.queue.put_nowait(event)
            except queue.Empty:
                pass

    def get(self, timeout: float | None = None) -> Event:
        return self.queue.get(timeout=timeout)
'''

files["core/state_manager.py"] = r'''from __future__ import annotations

import threading
from dataclasses import dataclass
from enum import Enum


class SafetyState(str, Enum):
    BOOT = "BOOT"
    SELF_TEST = "SELF_TEST"
    NORMAL = "NORMAL"
    WARNING = "WARNING"
    DROWSY = "DROWSY"
    OVERSPEED = "OVERSPEED"
    IMPACT_SUSPECTED = "IMPACT_SUSPECTED"
    FALL_SUSPECTED = "FALL_SUSPECTED"
    EMERGENCY_COUNTDOWN = "EMERGENCY_COUNTDOWN"
    EMERGENCY_ACTIVE = "EMERGENCY_ACTIVE"
    EMERGENCY_CANCELLED = "EMERGENCY_CANCELLED"
    RECOVERY = "RECOVERY"
    FAULT = "FAULT"


@dataclass
class StateSnapshot:
    state: SafetyState
    reason: str = ""


class StateManager:
    def __init__(self) -> None:
        self._state = SafetyState.BOOT
        self._reason = ""
        self._lock = threading.RLock()

    def set(self, state: SafetyState, reason: str = "") -> None:
        with self._lock:
            self._state = state
            self._reason = reason

    def snapshot(self) -> StateSnapshot:
        with self._lock:
            return StateSnapshot(self._state, self._reason)
'''

files["core/logger.py"] = r'''from __future__ import annotations

import logging
import sys


def configure_logging(level: str = "INFO") -> logging.Logger:
    logger = logging.getLogger("smart_helmet")
    if logger.handlers:
        return logger
    logger.setLevel(getattr(logging, level.upper(), logging.INFO))
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(
        logging.Formatter("%(asctime)s | %(levelname)s | %(name)s | %(message)s")
    )
    logger.addHandler(handler)
    return logger
'''

files["core/safety_manager.py"] = r'''from __future__ import annotations

import threading
import time
import uuid
from dataclasses import dataclass
from typing import Any

from config import Config
from core.event_bus import Event, EventBus
from core.state_manager import SafetyState, StateManager


@dataclass
class MotionEvidence:
    acceleration_g: float = 1.0
    deceleration_g: float = 0.0
    angular_velocity_dps: float = 0.0
    tilt_deg: float = 0.0
    inactivity: bool = False


class SafetyManager:
    """Centralized safety decision engine.

    A possible accident needs corroborating evidence and a confirmation window.
    It never treats one noisy sample as proof of a crash.
    """

    def __init__(self, config: Config, bus: EventBus, state: StateManager) -> None:
        self.config = config
        self.bus = bus
        self.state = state
        self._lock = threading.RLock()
        self._countdown_thread: threading.Thread | None = None
        self._cancel_event = threading.Event()
        self._last_motion = MotionEvidence()
        self._impact_time: float | None = None
        self._last_speed_warning = 0.0
        self._event_id: str | None = None

        bus.subscribe("IMU_SAMPLE", self._on_imu)
        bus.subscribe("BUTTON_CANCEL", self._on_cancel)
        bus.subscribe("DROWSINESS_WARNING", lambda e: self._set_warning(SafetyState.DROWSY, e))
        bus.subscribe("DROWSINESS_CRITICAL", lambda e: self._set_warning(SafetyState.DROWSY, e))
        bus.subscribe("OVERSPEED", self._on_overspeed)

    def _set_warning(self, state: SafetyState, event: Event) -> None:
        current = self.state.snapshot().state
        if current not in {
            SafetyState.EMERGENCY_COUNTDOWN,
            SafetyState.EMERGENCY_ACTIVE,
        }:
            self.state.set(state, event.event_type)

    def _on_overspeed(self, event: Event) -> None:
        now = time.monotonic()
        if now - self._last_speed_warning < self.config.speed_warning_cooldown_seconds:
            return
        self._last_speed_warning = now
        self.state.set(SafetyState.OVERSPEED, "speed threshold exceeded")

    def _on_cancel(self, event: Event) -> None:
        if self.state.snapshot().state == SafetyState.EMERGENCY_COUNTDOWN:
            self._cancel_event.set()
            self.state.set(SafetyState.EMERGENCY_CANCELLED, "rider cancelled")
            self.bus.publish(
                Event("EMERGENCY_CANCELLED", "INFO", {"event_id": self._event_id})
            )

    def _on_imu(self, event: Event) -> None:
        p = event.payload
        evidence = MotionEvidence(
            acceleration_g=float(p.get("acceleration_g", 1.0)),
            deceleration_g=float(p.get("deceleration_g", 0.0)),
            angular_velocity_dps=float(p.get("angular_velocity_dps", 0.0)),
            tilt_deg=abs(float(p.get("tilt_deg", 0.0))),
            inactivity=bool(p.get("inactivity", False)),
        )
        with self._lock:
            self._last_motion = evidence

        impact = (
            evidence.acceleration_g >= self.config.impact_threshold_g
            or evidence.deceleration_g >= self.config.deceleration_threshold_g
        )
        rotation = evidence.angular_velocity_dps >= self.config.rotation_threshold_dps
        fall = evidence.tilt_deg >= self.config.fall_angle_threshold_deg

        if impact and (rotation or fall):
            self._begin_confirmation("multi-signal impact + abnormal motion")
        elif fall and evidence.inactivity:
            self._begin_confirmation("abnormal orientation + inactivity")

    def _begin_confirmation(self, reason: str) -> None:
        with self._lock:
            if self.state.snapshot().state in {
                SafetyState.EMERGENCY_COUNTDOWN,
                SafetyState.EMERGENCY_ACTIVE,
            }:
                return
            if self._countdown_thread and self._countdown_thread.is_alive():
                return
            self._event_id = str(uuid.uuid4())
            self._impact_time = time.monotonic()
            self._cancel_event.clear()
            self.state.set(SafetyState.EMERGENCY_COUNTDOWN, reason)
            event_id = self._event_id

        self.bus.publish(
            Event(
                "EMERGENCY_COUNTDOWN",
                "CRITICAL",
                {
                    "event_id": event_id,
                    "reason": reason,
                    "countdown_seconds": self.config.confirmation_seconds,
                },
            )
        )
        self._countdown_thread = threading.Thread(
            target=self._countdown_worker, args=(event_id,), daemon=True
        )
        self._countdown_thread.start()

    def _countdown_worker(self, event_id: str) -> None:
        deadline = time.monotonic() + self.config.confirmation_seconds
        while time.monotonic() < deadline:
            if self._cancel_event.wait(timeout=0.25):
                return
        if self.state.snapshot().state != SafetyState.EMERGENCY_COUNTDOWN:
            return
        self.state.set(SafetyState.EMERGENCY_ACTIVE, "confirmation window expired")
        self.bus.publish(
            Event(
                "EMERGENCY_TRIGGER",
                "EMERGENCY",
                {
                    "event_id": event_id,
                    "motion": self._last_motion.__dict__,
                },
            )
        )

    def demo_crash(self) -> None:
        self._begin_confirmation("SIMULATED crash event")
'''

files["sensors/imu.py"] = r'''from __future__ import annotations

import math
import time
from threading import Event as ThreadEvent, Thread
from typing import Callable

from core.event_bus import Event, EventBus

try:
    from smbus2 import SMBus
except ImportError:
    SMBus = None


class MPU6050:
    ADDRESS = 0x68
    PWR_MGMT_1 = 0x6B
    ACCEL_XOUT_H = 0x3B
    GYRO_XOUT_H = 0x43

    def __init__(self, bus_number: int, bus: object | None = None) -> None:
        self.bus = bus or (SMBus(bus_number) if SMBus else None)
        if self.bus is None:
            raise RuntimeError("smbus2 is not installed")
        self.bus.write_byte_d
