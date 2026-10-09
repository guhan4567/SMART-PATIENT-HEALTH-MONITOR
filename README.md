# SMART-PATIENT-HEALTH-MONITOR
ESP32 Based Smart Patient Health Monitoring System
Project Overview
The Smart Patient Health Monitor is an IoT-based healthcare system that monitors patient health parameters using sensors connected to an ESP32. It displays sensor readings on a web dashboard for real-time monitoring.
Objectives
Monitor heart rate and blood oxygen saturation (SpO₂).
Track patient movement and acceleration.
Display live sensor readings on a web dashboard.
Identify abnormal readings and provide alerts.
Hardware Components
ESP32 Development Board
Heart Rate Sensor
SpO₂ Sensor
MPU6050 Accelerometer and Gyroscope
OLED Display (optional)
Buzzer and LED Indicators
Software Requirements
Arduino IDE
Embedded C/C++
HTML, CSS, and JavaScript
Wi-Fi connectivity
Key Features
Real-time health monitoring
Patient movement detection
Live web dashboard
Sensor status monitoring
Alerts for abnormal readings, if implemented
Future Enhancements
Cloud-based health data storage
Mobile notifications
Patient history and reports
Remote monitoring for healthcare professionals
Applications
Patient monitoring
Home healthcare
Hospital support systems
Remote health monitoring prototypes
Disclaimer: This project is an educational prototype and is not intended to replace certified medical equipment.
3. Your GitHub Repository
Your repository is already created:
SMART-PATIENT-HEALTH-MONITOR⁠�
You have already uploaded sketch_oct8a.ino. Next, you can update README.md with the content above.
/*
  SMART PATIENT HEALTH MONITORING SYSTEM
  ESP32 + MAX30102 + MPU6050
  Integrated local web dashboard
  No cloud required
  No emojis in code
*/

#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include "MAX30105.h"
#include "spo2_algorithm.h"
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <math.h>

// ---------------- WI-FI SETTINGS ----------------

const char* WIFI_SSID = "vivot4x";
const char* WIFI_PASSWORD = "12345678";

const char* AP_SSID = "Patient-Monitor";
const char* AP_PASSWORD = "health123";

// ---------------- HARDWARE SETTINGS ----------------

#define I2C_SDA 21
#define I2C_SCL 22

MAX30105 particleSensor;
Adafruit_MPU6050 mpu;
WebServer server(80);

bool max30102Ready = false;
bool mpuReady = false;
bool usingAccessPoint = false;

float heartRateBPM = 0;
float spo2Percent = 0;

bool heartRateValid = false;
bool spo2Valid = false;

float ax = 0;
float ay = 0;
float az = 1;

float totalAccel = 1;
float pitchDeg = 0;
float rollDeg = 0;

long irValue = 0;

String motionStatus = "Waiting";
String motionEvent = "None detected";

const int BUFFER_LENGTH = 100;

uint32_t irBuffer[BUFFER_LENGTH];
uint32_t redBuffer[BUFFER_LENGTH];

int bufferIndex = 0;

unsigned long lastMotionUpdate = 0;

// ---------------- INTEGRATED DASHBOARD ----------------

const char DASHBOARD[] PROGMEM = R"HTML(
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Smart Patient Health Monitor</title>

<style>
:root {
  --bg:#061225;
  --panel:#0a1b33;
  --line:#17385b;
  --text:#edf5ff;
  --muted:#8eafd2;
  --cyan:#20d5df;
  --blue:#299cff;
  --pink:#ff477e;
  --green:#21d6a0;
  --orange:#ffab3d;
  --purple:#bb70ff;
}

* { box-sizing:border-box; }

body {
  margin:0;
  background:radial-gradient(circle at 70% 0%,#10284b,#061225 60%);
  font-family:Arial,Helvetica,sans-serif;
  color:var(--text);
}

.app {
  display:grid;
  grid-template-columns:220px 1fr;
  min-height:100vh;
}

.sidebar {
  padding:25px 17px;
  background:#06152a;
  border-right:1px solid var(--line);
  display:flex;
  flex-direction:column;
  gap:25px;
}

.brand {
  font-size:15px;
  font-weight:bold;
  line-height:1.5;
}

.mark {
  display:inline-grid;
  place-items:center;
  width:36px;
  height:36px;
  margin-right:8px;
  border:1px solid var(--cyan);
  border-radius:10px;
  color:var(--cyan);
}

.nav {
  display:grid;
  gap:8px;
}

.nav button {
  background:transparent;
  border:0;
  color:#b7cce4;
  text-align:left;
  padding:13px;
  border-radius:9px;
  font-size:13px;
  cursor:pointer;
}

.nav button.active,
.nav button:hover {
  background:#0d3158;
  color:white;
  box-shadow:inset 3px 0 var(--cyan);
}

.connection {
  margin-top:auto;
  border-top:1px solid var(--line);
  padding-top:17px;
  color:var(--muted);
  font-size:11px;
  line-height:1.9;
  overflow-wrap:anywhere;
}

.dot {
  display:inline-block;
  width:8px;
  height:8px;
  background:var(--green);
  border-radius:50%;
  margin-right:7px;
}

main {
  padding:25px;
  min-width:0;
}

.topbar {
  display:flex;
  justify-content:space-between;
  align-items:center;
  gap:14px;
  margin-bottom:22px;
}

.eyebrow {
  font-size:10px;
  letter-spacing:2px;
  color:var(--cyan);
  text-transform:uppercase;
}

h1 {
  font-size:24px;
  margin:7px 0;
}

.right {
  text-align:right;
  color:var(--muted);
  font-size:11px;
}

.pill {
  display:inline-flex;
  align-items:center;
  border:1px solid #1c6c61;
  background:#0a302f;
  color:#52edc3;
  padding:8px 10px;
  border-radius:25px;
  margin-bottom:7px;
}

.grid {
  display:grid;
  grid-template-columns:repeat(4,minmax(0,1fr));
  gap:13px;
}

.metric,
.panel {
  background:linear-gradient(145deg,#0c203b,#08182d);
  border:1px solid var(--line);
  border-radius:14px;
}

.metric {
  padding:16px;
  min-width:0;
}

.metric-head {
  display:flex;
  align-items:center;
  gap:9px;
  color:#c6d9ef;
  font-size:12px;
}

.icon-box {
  width:34px;
  height:34px;
  border-radius:10px;
  display:grid;
  place-items:center;
  font-weight:bold;
  background:var(--icon-bg);
  color:var(--accent);
  font-size:12px;
}

.pink { --accent:var(--pink); --icon-bg:#48213a; }
.blue { --accent:var(--blue); --icon-bg:#123b61; }
.orange { --accent:var(--orange); --icon-bg:#4b351c; }
.purple { --accent:var(--purple); --icon-bg:#392452; }

.value {
  font-size:32px;
  font-weight:750;
  margin:16px 0 5px;
}

.unit {
  font-size:12px;
  color:var(--muted);
  font-weight:normal;
}

.foot {
  display:flex;
  justify-content:space-between;
  gap:5px;
  align-items:center;
  font-size:10px;
  color:var(--muted);
}

.state {
  color:var(--green);
  background:#0b3a35;
  padding:4px 7px;
  border-radius:18px;
}

.middle {
  display:grid;
  grid-template-columns:1fr 1fr .85fr;
  gap:13px;
  margin-top:13px;
}

.panel {
  padding:16px;
  min-width:0;
}

.panel-title {
  display:flex;
  justify-content:space-between;
  align-items:center;
  font-size:13px;
  font-weight:bold;
  margin-bottom:14px;
}

.panel-title span {
  font-size:10px;
  color:var(--muted);
  font-weight:normal;
}

.chart {
  width:100%;
  height:175px;
  display:block;
}

.chart text {
  fill:#83a8cf;
  font-size:10px;
}

.gridline {
  stroke:#173454;
  stroke-width:1;
}

.chart-line {
  fill:none;
  stroke-width:2.5;
  stroke-linecap:round;
  stroke-linejoin:round;
}

.rows {
  display:grid;
  gap:12px;
}

.row {
  display:flex;
  justify-content:space-between;
  gap:8px;
  font-size:11px;
  color:var(--muted);
}

.row strong {
  color:var(--text);
  font-size:12px;
}

.subheading {
  border-top:1px solid var(--line);
  padding-top:12px;
  margin-top:14px;
  margin-bottom:11px;
  font-size:11px;
  font-weight:bold;
}

.bottom {
  display:grid;
  grid-template-columns:1fr 1fr 1.25fr;
  gap:13px;
  margin-top:13px;
}

.reading {
  display:grid;
  grid-template-columns:9px 1fr auto;
  align-items:center;
  gap:9px;
  padding:11px 0;
  border-bottom:1px solid #15304d;
  font-size:11px;
  color:#bfd3eb;
}

.reading:last-child {
  border:0;
}

.reading strong {
  color:white;
  font-size:12px;
}

.key {
  width:8px;
  height:8px;
  border-radius:50%;
}

.ring-wrap {
  display:flex;
  align-items:center;
  gap:15px;
  min-height:140px;
}

.ring {
  width:105px;
  height:105px;
  border-radius:50%;
  background:conic-gradient(var(--green) 0 94%,#17344f 94%);
  display:grid;
  place-items:center;
  flex-shrink:0;
}

.ring-inner {
  width:82px;
  height:82px;
  border-radius:50%;
  background:#0a1b33;
  display:grid;
  place-items:center;
  text-align:center;
  font-size:12px;
  font-weight:bold;
  color:var(--green);
  padding:8px;
}

.legend {
  display:grid;
  gap:11px;
  font-size:11px;
  color:#bfd3eb;
}

.legend span:before {
  content:"";
  display:inline-block;
  width:7px;
  height:7px;
  border-radius:50%;
  background:var(--color);
  margin-right:8px;
}

.visuals {
  display:grid;
  grid-template-columns:1fr 1fr;
  gap:9px;
}

.mini {
  border:1px solid var(--line);
  background:#07162b;
  border-radius:9px;
  padding:9px;
}

.mini-title {
  font-size:10px;
  color:#9fc5ed;
  margin-bottom:8px;
}

.mini svg {
  width:100%;
  height:95px;
}

.notice {
  margin-top:13px;
  color:#8caaca;
  font-size:10px;
  line-height:1.5;
}

.footer {
  text-align:center;
  color:#577da6;
  font-size:10px;
  padding:20px 0 4px;
}

@media(max-width:1050px) {
  .grid { grid-template-columns:repeat(2,minmax(0,1fr)); }
  .middle { grid-template-columns:1fr 1fr; }
  .motion { grid-column:span 2; }
  .bottom { grid-template-columns:1fr 1fr; }
  .sensor { grid-column:span 2; }
}

@media(max-width:680px) {
  .app { grid-template-columns:1fr; }
  .sidebar { padding:13px; gap:12px; }
  .nav { grid-template-columns:repeat(4,1fr); gap:3px; }
  .nav button { text-align:center; padding:10px 3px; font-size:10px; }
  .connection { display:none; }
  main { padding:13px; }
  .topbar h1 { font-size:19px; }
  .grid,.middle,.bottom { grid-template-columns:1fr; }
  .motion,.sensor { grid-column:auto; }
  .value { font-size:29px; }
}
</style>
</head>

<body>
<div class="app">

<aside class="sidebar">
  <div class="brand"><span class="mark">+</span>SMART PATIENT<br>HEALTH MONITOR</div>

  <nav class="nav">
    <button class="active" data-title="Real-Time Health Overview">Overview</button>
    <button data-title="Live Sensor Data">Live Data</button>
    <button data-title="Health Trends and History">History</button>
    <button data-title="System Information">System</button>
  </nav>

  <div class="connection">
    <div><span class="dot"></span><strong id="connection">Connecting</strong></div>
    <div id="ip">Local dashboard</div>
    <div id="clockSide"></div>
  </div>
</aside>

<main>

<header class="topbar">
  <div>
    <div class="eyebrow">Patient monitoring system</div>
    <h1 id="pageTitle">Real-Time Health Overview</h1>
  </div>
  <div class="right">
    <div class="pill"><span class="dot"></span><span id="overall">Starting sensors</span></div>
    <div id="clock"></div>
  </div>
</header>

<section class="grid">

  <article class="metric pink">
    <div class="metric-head"><div class="icon-box">HR</div>Heart Rate</div>
    <div class="value"><span id="hr">--</span> <span class="unit">BPM</span></div>
    <div class="foot"><span class="state" id="hrState">Waiting</span><span>Reference 60-100</span></div>
  </article>

  <article class="metric blue">
    <div class="metric-head"><div class="icon-box">O2</div>Blood Oxygen</div>
    <div class="value"><span id="spo2">--</span> <span class="unit">%</span></div>
    <div class="foot"><span class="state" id="spo2State">Waiting</span><span>Typical 95-100%</span></div>
  </article>

  <article class="metric orange">
    <div class="metric-head"><div class="icon-box">ACC</div>Acceleration</div>
    <div class="value"><span id="accel">--</span> <span class="unit">g</span></div>
    <div class="foot"><span class="state" id="motionState">Waiting</span><span>MPU6050</span></div>
  </article>

  <article class="metric purple">
    <div class="metric-head"><div class="icon-box">MOV</div>Movement</div>
    <div class="value" style="font-size:25px" id="activity">Waiting</div>
    <div class="foot"><span class="state" id="activityState">No event</span><span>Motion status</span></div>
  </article>

</section>

<section class="middle">

  <article class="panel">
    <div class="panel-title">Heart Rate Trend <span>BPM</span></div>
    <svg class="chart" id="hrChart" viewBox="0 0 340 175" preserveAspectRatio="none"></svg>
  </article>

  <article class="panel">
    <div class="panel-title">SpO2 Trend <span>Percent</span></div>
    <svg class="chart" id="spo2Chart" viewBox="0 0 340 175" preserveAspectRatio="none"></svg>
  </article>

  <article class="panel motion">
    <div class="panel-title">Motion Data <span>MPU6050</span></div>

    <div class="rows">
      <div class="row"><span>X acceleration</span><strong id="xval">-- g</strong></div>
      <div class="row"><span>Y acceleration</span><strong id="yval">-- g</strong></div>
      <div class="row"><span>Z acceleration</span><strong id="zval">-- g</strong></div>
    </div>

    <div class="subheading">Orientation</div>

    <div class="rows">
      <div class="row"><span>Pitch</span><strong id="pitch">--</strong></div>
      <div class="row"><span>Roll</span><strong id="roll">--</strong></div>
      <div class="row"><span>Motion event</span><strong id="event">None</strong></div>
    </div>
  </article>

</section>

<section class="bottom">

  <article class="panel">
    <div class="panel-title">Live Readings <span>Current values</span></div>

    <div class="reading"><i class="key" style="background:var(--pink)"></i><span>Heart rate</span><strong><span id="hr2">--</span> BPM</strong></div>
    <div class="reading"><i class="key" style="background:var(--blue)"></i><span>SpO2</span><strong><span id="spo22">--</span>%</strong></div>
    <div class="reading"><i class="key" style="background:var(--orange)"></i><span>Total acceleration</span><strong><span id="accel2">--</span> g</strong></div>
    <div class="reading"><i class="key" style="background:var(--green)"></i><span>Movement</span><strong id="activity2">Waiting</strong></div>
  </article>

  <article class="panel">
    <div class="panel-title">System Status <span>Sensor overview</span></div>

    <div class="ring-wrap">
      <div class="ring"><div class="ring-inner" id="ringText">Waiting</div></div>
      <div class="legend">
        <span style="--color:var(--green)">Heart rate sensor</span>
        <span style="--color:var(--blue)">SpO2 sensor</span>
        <span style="--color:var(--purple)">MPU6050</span>
        <span style="--color:var(--orange)">ESP32</span>
      </div>
    </div>
  </article>

  <article class="panel sensor">
    <div class="panel-title">Sensor Visualization <span>Signal view</span></div>

    <div class="visuals">
      <div class="mini">
        <div class="mini-title">Pulse signal</div>
        <svg viewBox="0 0 180 95" preserveAspectRatio="none">
          <path d="M0 50 L15 50 L23 15 L32 80 L42 50 L58 50 L66 15 L75 80 L85 50 L101 50 L109 15 L118 80 L128 50 L144 50 L152 15 L161 80 L171 50 L180 50" fill="none" stroke="#ff477e" stroke-width="2.3"/>
        </svg>
      </div>

      <div class="mini">
        <div class="mini-title">Motion axes</div>
        <svg viewBox="0 0 180 95" preserveAspectRatio="none">
          <path d="M0 24 Q10 7 20 24 T40 24 T60 24 T80 24 T100 24 T120 24 T140 24 T160 24 T180 24" fill="none" stroke="#21d6a0" stroke-width="2"/>
          <path d="M0 47 Q10 30 20 47 T40 47 T60 47 T80 47 T100 47 T120 47 T140 47 T160 47 T180 47" fill="none" stroke="#299cff" stroke-width="2"/>
          <path d="M0 72 Q10 55 20 72 T40 72 T60 72 T80 72 T100 72 T120 72 T140 72 T160 72 T180 72" fill="none" stroke="#bb70ff" stroke-width="2"/>
        </svg>
      </div>
    </div>
  </article>

</section>

<div class="notice">
  Educational prototype only. Sensor readings may be inaccurate and are not a substitute for medical equipment, professional assessment, or emergency care. Motion status is not a validated fall diagnosis.
</div>

<footer class="footer">ESP32 | MAX30102 PULSE OXIMETER | MPU6050 | LOCAL WI-FI DASHBOARD</footer>

</main>
</div>

<script>
const hrHistory = [];
const spo2History = [];

function setValue(id, value) {
  document.getElementById(id).textContent = value;
}

function setState(id, text, warning) {
  setValue(id, text);
  document.getElementById(id).style.color =
    warning ? "#ff8b9e" : "#21d6a0";
}

function drawChart(id, data, min, max, color) {
  const svg = document.getElementById(id);
  const w = 340, h = 175;
  const left = 30, right = 5, top = 10, bottom = 10;
  const pw = w - left - right;
  const ph = h - top - bottom;
  let output = "";

  for (let i = 0; i < 5; i++) {
    const y = top + ph * i / 4;
    const label = Math.round(max - (max - min) * i / 4);

    output += `<line class="gridline" x1="${left}" y1="${y}" x2="${w-right}" y2="${y}"/>`;
    output += `<text x="1" y="${y+3}">${label}</text>`;
  }

  if (data.length > 0) {
    const points = data.map((value, index) => [
      left + pw * index / Math.max(1, data.length - 1),
      top + ph - (value - min) / (max - min) * ph
    ]);

    const path = points.map((point, index) =>
      (index ? "L" : "M") +
      point[0].toFixed(1) + " " +
      point[1].toFixed(1)
    ).join(" ");

    output += `<path class="chart-line" stroke="${color}" d="${path}"/>`;
  }

  svg.innerHTML = output;
}

function updateDashboard(data) {
  setValue("hr", data.hrValid ? Math.round(data.hr) : "--");
  setValue("hr2", data.hrValid ? Math.round(data.hr) : "--");

  setValue("spo2", data.spo2Valid ? Math.round(data.spo2) : "--");
  setValue("spo22", data.spo2Valid ? Math.round(data.spo2) : "--");

  setValue("accel", Number(data.accel).toFixed(2));
  setValue("accel2", Number(data.accel).toFixed(2));

  setValue("xval", Number(data.ax).toFixed(2) + " g");
  setValue("yval", Number(data.ay).toFixed(2) + " g");
  setValue("zval", Number(data.az).toFixed(2) + " g");

  setValue("pitch", Number(data.pitch).toFixed(1) + " deg");
  setValue("roll", Number(data.roll).toFixed(1) + " deg");

  setValue("activity", data.motion);
  setValue("activity2", data.motion);
  setValue("event", data.event);

  setValue("connection", data.connected ? "ESP32 Connected" : "ESP32 Offline");
  setValue("ip", data.ip);
  setValue("overall", data.status);
  setValue("ringText", data.status);

  setState("hrState",
    data.hrValid ? "Reading available" : "No valid reading",
    !data.hrValid);

  setState("spo2State",
    data.spo2Valid ? "Reading available" : "No valid reading",
    !data.spo2Valid);

  setState("motionState",
    data.motion === "Normal" ? "Stable" : "Motion detected",
    data.motion !== "Normal");

  setState("activityState",
    data.motion === "Normal" ? "No event" : "Check movement",
    data.motion !== "Normal");

  if (data.hrValid) {
    hrHistory.push(Number(data.hr));
    if (hrHistory.length > 24) hrHistory.shift();
  }

  if (data.spo2Valid) {
    spo2History.push(Number(data.spo2));
    if (spo2History.length > 24) spo2History.shift();
  }

  drawChart("hrChart", hrHistory, 40, 140, "#ff477e");
  drawChart("spo2Chart", spo2History, 80, 100, "#299cff");
}

async function fetchData() {
  try {
    const response = await fetch("/data", {cache:"no-store"});
    if (!response.ok) throw new Error("Server unavailable");
    const data = await response.json();
    updateDashboard(data);
  } catch (error) {
    setValue("connection", "Waiting for ESP32");
    setValue("overall", "Connection unavailable");
  }
}

function updateClock() {
  const currentTime = new Date().toLocaleString();
  setValue("clock", currentTime);
  setValue("clockSide", currentTime);
}

document.querySelectorAll(".nav button").forEach(button => {
  button.addEventListener("click", () => {
    document.querySelectorAll(".nav button").forEach(item =>
      item.classList.remove("active"));
    button.classList.add("active");
    setValue("pageTitle", button.dataset.title);
  });
});

updateClock();
setInterval(updateClock, 1000);
fetchData();
setInterval(fetchData, 1000);
</script>
</body>
</html>
)HTML";

// ---------------- WI-FI INITIALIZATION ----------------

void startNetwork() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  Serial.print("Connecting to Wi-Fi...");

  unsigned long startTime = millis();

  while (WiFi.status() != WL_CONNECTED &&
         millis() - startTime < 12000) {
    delay(300);
    Serial.print(".");
  }

  Serial.println();

  if (WiFi.status() == WL_CONNECTED) {
    usingAccessPoint = false;

    Serial.print("Dashboard URL: http://");
    Serial.println(WiFi.localIP());
  } else {
    WiFi.disconnect(true);
    WiFi.mode(WIFI_AP);
    WiFi.softAP(AP_SSID, AP_PASSWORD);
    usingAccessPoint = true;

    Serial.println("ESP32 access point started.");
    Serial.print("Wi-Fi name: ");
    Serial.println(AP_SSID);
    Serial.print("Wi-Fi password: ");
    Serial.println(AP_PASSWORD);
    Serial.print("Dashboard URL: http://");
    Serial.println(WiFi.softAPIP());
  }
}

// ---------------- MPU6050 PROCESSING ----------------

void updateMotion() {
  if (!mpuReady) {
    motionStatus = "Sensor unavailable";
    motionEvent = "Check wiring";
    return;
  }

  sensors_event_t acceleration;
  sensors_event_t gyro;
  sensors_event_t temperature;

  mpu.getEvent(&acceleration, &gyro, &temperature);

  // Convert acceleration from m/s^2 to g.

  ax = acceleration.acceleration.x / 9.80665f;
  ay = acceleration.acceleration.y / 9.80665f;
  az = acceleration.acceleration.z / 9.80665f;

  totalAccel = sqrtf(ax * ax + ay * ay + az * az);

  pitchDeg = atan2f(
    -ax,
    sqrtf(ay * ay + az * az)
  ) * 180.0f / PI;

  rollDeg = atan2f(ay, az) * 180.0f / PI;

  // Basic motion classification only.
  // These thresholds do not provide validated fall detection.

  if (totalAccel > 2.5f || totalAccel < 0.45f) {
    motionStatus = "High motion";
    motionEvent = "Sudden acceleration";
  } else if (fabsf(ax) > 0.35f || fabsf(ay) > 0.35f) {
    motionStatus = "Moving";
    motionEvent = "Movement detected";
  } else {
    motionStatus = "Normal";
    motionEvent = "None detected";
  }
}

// ---------------- MAX30102 PROCESSING ----------------

void updatePulseOximeter() {
  if (!max30102Ready) return;

  particleSensor.check();

  while (particleSensor.available()) {
    uint32_t red = particleSensor.getRed();
    uint32_t ir = particleSensor.getIR();

    irValue = (long)ir;

    redBuffer[bufferIndex] = red;
    irBuffer[bufferIndex] = ir;

    bufferIndex++;

    particleSensor.nextSample();

    if (bufferIndex >= BUFFER_LENGTH) {
      int32_t calculatedSpO2 = 0;
      int8_t validSpO2 = 0;

      int32_t calculatedHR = 0;
      int8_t validHR = 0;

      maxim_heart_rate_and_oxygen_saturation(
        irBuffer,
        BUFFER_LENGTH,
        redBuffer,
        &calculatedSpO2,
        &validSpO2,
        &calculatedHR,
        &validHR
      );

      spo2Valid =
        validSpO2 == 1 &&
        calculatedSpO2 >= 70 &&
        calculatedSpO2 <= 100;

      if (spo2Valid) {
        spo2Percent = calculatedSpO2;
      }

      heartRateValid =
        validHR == 1 &&
        calculatedHR >= 40 &&
        calculatedHR <= 200;

      if (heartRateValid) {
        heartRateBPM = calculatedHR;
      }

      // Retain half the buffer for the next calculation.

      for (int i = 0; i < BUFFER_LENGTH / 2; i++) {
        redBuffer[i] = redBuffer[i + BUFFER_LENGTH / 2];
        irBuffer[i] = irBuffer[i + BUFFER_LENGTH / 2];
      }

      bufferIndex = BUFFER_LENGTH / 2;
    }

    // Basic finger-presence heuristic.
    // The threshold may need adjustment for your sensor module.

    if (irValue < 50000) {
      heartRateValid = false;
      spo2Valid = false;
    }
  }
}

// ---------------- JSON DATA ENDPOINT ----------------

void handleData() {
  String status = "Sensors active";

  if (!max30102Ready || !mpuReady) {
    status = "Check sensor wiring";
  } else if (!heartRateValid && !spo2Valid) {
    status = "Place finger on sensor";
  } else if (motionStatus == "High motion") {
    status = "Movement alert";
  }

  IPAddress address =
    usingAccessPoint ? WiFi.softAPIP() : WiFi.localIP();

  String json = "{";

  json += "\"connected\":true,";
  json += "\"ip\":\"" + address.toString() + "\",";

  json += "\"hr\":" + String(heartRateBPM, 1) + ",";
  json += "\"hrValid\":" + String(heartRateValid ? "true" : "false") + ",";

  json += "\"spo2\":" + String(spo2Percent, 1) + ",";
  json += "\"spo2Valid\":" + String(spo2Valid ? "true" : "false") + ",";

  json += "\"ir\":" + String(irValue) + ",";

  json += "\"ax\":" + String(ax, 3) + ",";
  json += "\"ay\":" + String(ay, 3) + ",";
  json += "\"az\":" + String(az, 3) + ",";

  json += "\"accel\":" + String(totalAccel, 3) + ",";
  json += "\"pitch\":" + String(pitchDeg, 2) + ",";
  json += "\"roll\":" + String(rollDeg, 2) + ",";

  json += "\"motion\":\"" + motionStatus + "\",";
  json += "\"event\":\"" + motionEvent + "\",";
  json += "\"status\":\"" + status + "\"";

  json += "}";

  server.sendHeader("Cache-Control", "no-store");
  server.send(200, "application/json", json);
}

// ---------------- SETUP ----------------

void setup() {
  Serial.begin(115200);
  delay(500);

  Wire.begin(I2C_SDA, I2C_SCL);

  Serial.println();
  Serial.println("Smart Patient Health Monitoring System");
  Serial.println("Initializing sensors...");

  // Initialize MAX30102.

  if (particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    max30102Ready = true;

    byte ledBrightness = 60;
    byte sampleAverage = 4;
    byte ledMode = 2;
    int sampleRate = 100;
    int pulseWidth = 411;
    int adcRange = 4096;

    particleSensor.setup(
      ledBrightness,
      sampleAverage,
      ledMode,
      sampleRate,
      pulseWidth,
      adcRange
    );

    particleSensor.setPulseAmplitudeRed(0x2A);
    particleSensor.setPulseAmplitudeIR(0x2A);
    particleSensor.setPulseAmplitudeGreen(0);

    Serial.println("MAX30102 detected.");
  } else {
    max30102Ready = false;
    Serial.println("MAX30102 not detected. Check SDA, SCL and power.");
  }

  // Initialize MPU6050.

  if (mpu.begin(0x68, &Wire)) {
    mpuReady = true;

    mpu.setAccelerometerRange(MPU6050_RANGE_4_G);
    mpu.setGyroRange(MPU6050_RANGE_500_DEG);
    mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);

    Serial.println("MPU6050 detected.");
  } else {
    mpuReady = false;
    Serial.println("MPU6050 not detected. Check wiring and address.");
  }

  startNetwork();

  server.on("/", HTTP_GET, []() {
    server.send_P(200, "text/html", DASHBOARD);
  });

  server.on("/data", HTTP_GET, handleData);

  server.onNotFound([]() {
    server.send(404, "text/plain", "Not found");
  });

  server.begin();

  Serial.println("Web server started.");
}

// ---------------- MAIN LOOP ----------------

void loop() {
  server.handleClient();

  updatePulseOximeter();

  if (millis() - lastMotionUpdate >= 100) {
    lastMotionUpdate = millis();
    updateMotion();
  }

  delay(2);
}
