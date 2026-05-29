# AAR---AUTOMATED-ATTENDACE-REGISTER
The system takes class attendance registration using facial recognition with the help of an ESP32 CAM.
# AAR – Automated Attendance Register
### Copperbelt University | SICT | Computer Engineering | Group 3

---

## 📁 Repository Structure

```
AAR_Project/
│
├── esp32_firmware/
│   └── esp32_cam_server/
│       └── esp32_cam_server.ino     ← Flash this to the ESP32-CAM
│
├── mobile_app/
│   ├── database/
│   │   └── db_manager.py            ← SQLite Persistence Node
│   ├── recognition/
│   │   ├── recognition_engine.py    ← OpenCV + InsightFace pipeline
│   │   └── stream_service.py        ← MJPEG stream + session timing
│   ├── analytics/
│   │   └── export_service.py        ← PDF / CSV report generation
│   └── ui/
│       ├── main_window.py           ← Root Toga app
│       ├── attendance_tab.py        ← Tab 1: Live attendance
│       ├── register_tab.py          ← Tab 2: Student registration
│       └── records_tab.py           ← Tabs 3–6: Records/Analytics/Export/Manage
│
├── exports/                         ← Generated reports saved here
├── main.py                          ← Run this to start the app
├── requirements.txt
├── pyproject.toml                   ← BeeWare/Briefcase Android config
└── README.md
```

---

## ⚠️ CHANGES FROM ORIGINAL PROPOSAL

| Item | Proposal Said | Implementation | Reason |
|------|--------------|----------------|--------|
| UI Framework | Kivy | **BeeWare/Toga** | The System Design Specification (SDS 3.3.2, Layer I) explicitly specifies BeeWare/Toga. This was the design team's own revision. |
| Recognition | Generic "facial recognition libraries" | **InsightFace (buffalo_sc)** | InsightFace generates the 512-dim vectors specified in SDS 3.3.2. It is more accurate than alternatives like `face_recognition` (dlib). |
| Export | Excel/CSV | **CSV + PDF** | SDS 2.4 specifies CSV and PDF. PDF is added for formal submission to lecturers. |
| Camera placement | Front of room | **Doorway (ESP32-CAM)** | SDS 3.1.1 design trade-off table explicitly chose the doorway placement. |

---

## 🔧 PART 1: Flash the ESP32-CAM

### Hardware Connections (FTDI Programmer → ESP32-CAM)

| FTDI Pin | ESP32-CAM Pin | Notes |
|----------|---------------|-------|
| GND      | GND           |       |
| VCC (5V) | 5V            |       |
| TX       | UOR (GPIO3)   |       |
| RX       | UOT (GPIO1)   |       |
| GND      | **IO0**       | ⚠️ MUST be connected to GND during flashing only |

### Arduino IDE Setup
1. Install **Arduino IDE 2.x**
2. Add ESP32 board support:
   - Go to **File → Preferences**
   - Add to "Additional Board Manager URLs":
     ```
     https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
     ```
3. Go to **Tools → Board → Board Manager**, search `esp32`, install **esp32 by Espressif Systems**
4. Select Board: **AI Thinker ESP32-CAM** (under Tools → Board → ESP32 Arduino)
5. Set Upload Speed: **115200**

### Configure WiFi in the Sketch
Open `esp32_firmware/esp32_cam_server/esp32_cam_server.ino` and update:
```cpp
const char* WIFI_SSID     = "AAR_Hotspot";   // ← Your phone hotspot name
const char* WIFI_PASSWORD = "aar12345";      // ← Your phone hotspot password
```

### Flash Steps
1. Connect IO0 to GND (enables flash mode)
2. Click Upload in Arduino IDE
3. Watch the progress bar – when it says "Connecting...", the board enters flash mode automatically
4. After "Done uploading": **disconnect IO0 from GND**
5. Press the **RESET** button on the ESP32-CAM
6. Open Serial Monitor at **115200 baud** – you will see:
   ```
   [OK] WiFi connected!
   [INFO] ESP32-CAM IP Address: 192.168.43.XXX
   [INFO] Stream URL: http://192.168.43.XXX:81/stream
   ```
7. **Note down the IP address** – enter it in the mobile app.

---

## 📱 PART 2: Setup the Mobile Application

### Prerequisites
- Python 3.10 or 3.11 (recommended)
- pip

### Install Dependencies
```bash
cd AAR_Project
pip install -r requirements.txt
```

> **Note on InsightFace on Windows:** You may need Visual C++ Build Tools.
> Download from: https://visualstudio.microsoft.com/visual-cpp-build-tools/

### Run on Desktop (for development/testing)
```bash
python main.py
```
This opens a native desktop window that mimics the Android UI.

### Deploy to Android (Production)
```bash
pip install briefcase
briefcase create android
briefcase build android
briefcase run android
```
The app will be installed on a connected Android device.

---

## 📖 HOW TO USE (Step-by-Step)

### First-Time Setup
1. Open the **⚙️ Manage** tab
2. Add your course: e.g., Course ID = `CEE401`, Name = `Computer Engineering Design IV`

### Registering Students (do this before the first session)
1. Open the **➕ Register** tab
2. **Turn on your phone's hotspot** with the SSID/password you set in the ESP32 firmware
3. Power on the ESP32-CAM – it will connect to your hotspot
4. Enter the ESP32-CAM IP address in the "Camera IP" field
5. Fill in: SIN, Full Name, Programme
6. Click **📸 Capture Face** – student looks at camera
7. Click **💾 Register Student**
8. Repeat for all students

### Running an Attendance Session
1. Open the **📷 Attendance** tab
2. Select the course
3. Enter the ESP32-CAM IP address
4. Click **▶ Start Session**
5. Students walk past the camera as they enter the lecture
6. The system marks them present automatically
7. The 15-minute window closes automatically (CBU "15-minute rule")
8. Session is locked for 105 minutes (full lecture block)

### Viewing & Exporting
- **📋 Records tab**: View today's register or session history
- **📊 Analytics tab**: See attendance percentages and eligibility flags
- **📤 Export tab**: Generate PDF or CSV reports for lecturer submission

---

## 🗄️ Database Schema

Stored at: `mobile_app/database/aar_database.db` (SQLite)

```sql
courses(course_id PK, course_name)

students(sin, course_id FK, name, programme, vector_blob, attendance_pct)
  -- vector_blob: 512-dim float32 numpy array stored as binary

sessions(session_id PK, course_id FK, session_date, UNIQUE)

attendance_logs(log_id PK, course_id FK, sin, name, timestamp, session_date)
  -- Sparse: only PRESENT students are logged (SDS 3.3.4)
```

---

## 🔑 Key Configuration Values

| Parameter | Value | Location |
|-----------|-------|----------|
| WiFi SSID | `AAR_Hotspot` | `esp32_cam_server.ino` |
| WiFi Password | `aar12345` | `esp32_cam_server.ino` |
| ESP32 Stream Port | `81` | Fixed in firmware |
| Attendance Window | 15 minutes | `stream_service.py` |
| Session Lockout | 105 minutes | `stream_service.py` |
| Recognition Threshold | 0.45 (cosine dist) | `recognition_engine.py` |
| Eligibility Threshold | 80% | `db_manager.py` |
| Frame Processing Rate | 1.5 seconds | `stream_service.py` |

---

## 🧪 Testing Checklist

- [ ] ESP32-CAM connects to phone hotspot
- [ ] Serial monitor shows correct IP address
- [ ] `http://<IP>:81/stream` opens in browser (shows video)
- [ ] Desktop app launches: `python main.py`
- [ ] Course can be added in Manage tab
- [ ] Student registration captures face embedding
- [ ] Attendance session marks a registered student present
- [ ] Duplicate entry prevention works (same student not marked twice)
- [ ] 15-minute window auto-closes
- [ ] CSV export generates correctly
- [ ] PDF eligibility report shows 80% threshold flags

---

## 👥 Group 3 Members

| Name | SIN |
|------|-----|
| Chipego Phiri | 23133136 |
| Bwalya Mulenga | 23125615 |
| Bonaventure Chikoti | 23124504 |
| Lukundo Sikainga | 23123589 |
| Caleb Bwalya | 23127908 |

**Supervisor:** Prof. Kalezhi  
**Institution:** Copperbelt University – SICT, Computer Engineering Department
