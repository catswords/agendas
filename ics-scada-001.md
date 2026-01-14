# ics-scada-001

## 개발 명세서 (Detailed Engineering Specification)

**부제:**
**Multi-Vector Attacker Model 기반
MQTT · Modbus/TCP · ONVIF/RTSP 동시 공격 시뮬레이션**

---

## 문서 정보

* Version: **0.3**
* Date: **2026-01-14**
* Owner: **Catswords Research**
* Scope: **MVP + 확장 기반**
* Status: **Implementation-Ready**

---

## 1. 시스템 목적

본 시스템은 산업 제어 환경(ICS/OT)을 3D 공간으로 구현하고,
장비 제어·관측·검증 경로가 **서로 다른 네트워크 프로토콜(MQTT / Modbus / CCTV)** 로 분리된 현실을 반영한다.

특히 본 시스템은 다음 질문에 답하도록 설계된다.

> **“공격자가 여러 제어·관측 경로에 동시에 접근할 수 있다면,
> 운영자는 무엇을 믿어야 하는가?”**

→ **정답: 카메라(현장 시야)**

---

## 2. 설계 원칙 (Design Principles)

### 2.1 Single Physical Truth (SPT)

* 실제 상태는 **Unity 3D 공간의 물리 객체**만이 진실이다.
* MQTT / Modbus 상태는 *논리적 관측치*일 뿐이다.

### 2.2 Multiple Logical Interfaces (MLI)

* 하나의 물리 장비는 다음 인터페이스를 가진다:

  * MQTT (IT/OT 제어)
  * Modbus/TCP (Legacy OT)
  * CCTV(ONVIF/RTSP) (물리 검증)

### 2.3 Multi-Vector Attacker Assumption

* 공격자는 **동시에** 다음에 접근 가능하다고 가정한다:

  * MQTT Broker
  * Modbus Gateway
  * ONVIF / RTSP

---

## 3. 네트워크 및 존 모델

### 3.1 논리 존 구분

| Zone       | 설명                               |
| ---------- | -------------------------------- |
| IT / Corp  | 운영자, VMS, 공격자 접근 용이              |
| DMZ        | 표준 인터페이스(MQTT/ONVIF/RTSP/Modbus) |
| OT / Plant | 3D 물리 진실(Unity)                  |

> MVP에서는 물리적 분리는 선택사항이나,
> **논리적 경계는 반드시 코드 구조에 반영**해야 한다.

---

## 4. 주요 구성 요소 명세

---

## 4.1 3D Space Server (Unity)

### 4.1.1 책임

* 물리 장비 시뮬레이션 (Single Physical Truth)
* 카메라 시뮬레이션
* MQTT / Modbus 명령 적용
* 상태/이벤트 발행
* 카메라 영상 생성

---

### 4.1.2 핵심 내부 모듈

```
Unity
 ├─ DeviceRegistry        // 장비 SoT
 ├─ CameraRegistry        // 카메라 SoT
 ├─ MqttClientModule
 ├─ ModbusSyncModule
 ├─ PtzControlModule
 ├─ StateMismatchDetector
 └─ FrameExportModule
```

---

### 4.1.3 장비(Device) 모델

#### 공통 필드

* `deviceId`
* `deviceType`
* `physicalState`        ← 진실
* `logicalState.mqtt`
* `logicalState.modbus`
* `alarmState`

#### 예시: Pump

```json
{
  "deviceId": "pump-01",
  "physical": { "power": true, "rpm": 1800, "temp": 85.3 },
  "mqtt":     { "power": true, "rpm": 1800, "temp": 42.1 },
  "modbus":   { "rpm": 1800, "temp": 42.0 },
  "alarm": "CRITICAL"
}
```

> 물리값과 논리값의 **불일치가 의도적으로 발생 가능**해야 한다.

---

### 4.1.4 카메라(Camera) 모델

#### 구조

```
CameraRoot
 └─ Yaw (Pan)
     └─ Pitch (Tilt)
         └─ Lens (Unity Camera)
```

#### 속성

* `cameraId`
* `ptzLimits`
* `renderTexture`
* `overlayEnabled`

---

## 4.2 MQTT 명세

### 4.2.1 토픽 구조

```
site/{siteId}/device/{deviceId}/cmd
site/{siteId}/device/{deviceId}/state
site/{siteId}/device/{deviceId}/evt
```

### 4.2.2 QoS 정책

| Topic | QoS | Retain |
| ----- | --- | ------ |
| cmd   | 1   | false  |
| state | 1   | true   |
| evt   | 1   | false  |

---

### 4.2.3 Command Payload

```json
{
  "ts": "ISO-8601",
  "op": "set",
  "k": "power",
  "v": 1,
  "ttl_ms": 2000,
  "req_id": "uuid"
}
```

* TTL 만료 시 Unity는 **명령 무시**
* 처리 결과는 `evt`로 ACK/NACK 발행

---

## 4.3 Modbus/TCP 명세

### 4.3.1 역할

* 레거시 OT 제어/관측 인터페이스
* 공격자가 **직접 레지스터 조작 가능**

### 4.3.2 매핑 규칙 (예: pump-01)

| Address | Type | Meaning   |
| ------- | ---- | --------- |
| 00001   | Coil | Power     |
| 00002   | Coil | Alarm     |
| 40001   | HR   | RPM       |
| 40002   | HR   | Temp × 10 |

### 4.3.3 동작 규칙

* Modbus Write → Unity `logicalState.modbus` 갱신
* 물리 상태 반영 여부는 **시나리오에 따라 다름**
* Polling 지연/위조 시뮬레이션 가능

---

## 4.4 ONVIF / RTSP 명세

### 4.4.1 ONVIF Server

#### 지원 서비스

* Device

  * GetDeviceInformation
  * GetCapabilities
* Media

  * GetProfiles (Multi-Channel)
  * GetStreamUri
* PTZ

  * ContinuousMove
  * Stop
  * GetStatus

#### Discovery

* WS-Discovery (UDP/3702)

---

### 4.4.2 RTSP 스트리밍

* Codec: H.264
* Source: Unity RenderTexture
* 경로:

```
rtsp://<rtsp-server>/cam/{cameraId}
```

* 인코딩:

  * Unity → FFmpeg(stdin rawvideo)
  * FFmpeg → MediaMTX (RTSP Push)

---

## 5. 공격자 모델 (Threat Model)

### 5.1 공격 가정

공격자는 **동일 시간대에** 다음을 수행 가능:

* MQTT 명령 주입 / 상태 위조
* Modbus 레지스터 쓰기
* ONVIF PTZ 강제 이동
* RTSP 스트림 차단/대체

---

## 6. 대표 공격 시나리오 (시스템 필수 지원)

---

### 시나리오 A

### MQTT 제어 + CCTV 은폐

* MQTT로 펌프 가동
* ONVIF PTZ로 카메라 시야 회피
* 논리 상태 정상, 물리 위험

---

### 시나리오 B

### Modbus 위조 + 카메라만 진실

* Modbus로 정상 온도 지속 주입
* 실제 과열 → 연기/VFX
* 카메라만 이상 감지

---

### 시나리오 C

### 카메라 마비 + 제어 지속

* RTSP DoS
* MQTT/Modbus 제어 정상
* “제어는 되나 검증 불가”

---

## 7. 상태 불일치 탐지 (필수)

### 7.1 Mismatch Detector

Unity 내부에서 다음 비교 수행:

```
physicalState != mqttState
physicalState != modbusState
cameraVisibility == false
```

### 7.2 결과

* UI 경고
* 이벤트 로그 생성
* 교육용 힌트 제공

---

## 8. UX / 시각화 요구사항

### 8.1 카메라 중심 UX

* 카메라 영상은 **최종 판단 근거**
* 논리 상태 패널과 분리

### 8.2 시각 효과 규칙

| 상태       | 효과                 |
| -------- | ------------------ |
| Normal   | Neutral            |
| Warning  | Yellow blink       |
| Critical | Red beacon / smoke |

---

## 9. MVP 범위

### 포함

* 장비 3종
* 카메라 2대
* MQTT + Modbus + ONVIF/RTSP
* 공격 시나리오 ≥ 2
* 상태 불일치 표현

### 제외

* 인증/권한
* 실제 PLC
* 녹화/재생

---

## 10. 핵심 메시지 (프로젝트 아이덴티티)

> **“제어는 논리적일 수 있지만,
> 사고는 물리적으로 발생한다.”**

이 프로젝트는 단순 시뮬레이터가 아니라
**ICS 보안 사고의 본질을 ‘보여주는’ 도구**이다.

---

## 11. 다음 단계 (권장)

1. **Unity 프로젝트 구조 + 클래스 다이어그램**
2. **MQTT / Modbus / ONVIF 시퀀스 다이어그램**
3. **공격 시나리오 DSL 정의**
4. **훈련자/공격자 UI 분리 설계**


## 다이어그램

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                          Industrial Cyber Range 3D (Detailed Diagram)                                                     │
│                              MQTT + Modbus/TCP + ONVIF/RTSP  (Multi-Vector Attacker Model)                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

LEGEND
  [ ]  Component / Process                     -->  Primary data flow
  ( )  Protocol / Port / Notes                 ==>  Attack path (attacker can access directly)
  { }  Key data / contracts                    ~~>  Internal control / state sync

NETWORK ZONES (logical separation; may be on one host in MVP)
  IT / Corp Network     |     DMZ (Services)     |      OT / Plant Network (3D Physical Truth)

=================================================================================================================================================
IT / CORP NETWORK (Operator / Attacker access is easiest here)
=================================================================================================================================================

                  ┌───────────────────────────┐
                  │       Operator / VMS      │
                  │  - ONVIF client (discover)│
                  │  - RTSP viewer (live)     │
                  │  - Optional Web UI        │
                  └─────────────┬─────────────┘
                                │
                                │  (ONVIF WS-Discovery UDP/3702 + SOAP HTTP/80|8080)
                                │  (RTSP TCP/554|8554)
                                v
        ===========================================================================================================
        ||  ATTACKER (Multi-Vector)                                                                           ||
        ||  - Can hit MQTT, Modbus, ONVIF, RTSP simultaneously                                                 ||
        ||                                                                                                     ||
        ||   ==> MQTT (1883/8883)       ==> Modbus/TCP (502)      ==> ONVIF (3702/80|8080)     ==> RTSP (554|8554)||
        ===========================================================================================================
                                │                     │                       │                     │
                                │                     │                       │                     │
                                v                     v                       v                     v

=================================================================================================================================================
DMZ (SERVICES) — where “real-world interfaces” live (best practice); MVP may co-locate
=================================================================================================================================================

┌───────────────────────────────┐          ┌───────────────────────────────────┐          ┌───────────────────────────────┐
│        MQTT Broker            │          │            ONVIF Server           │          │          RTSP Server          │
│   (Mosquitto / EMQX)          │          │     (Device + Media + PTZ)        │          │          (MediaMTX)           │
│   Ports: 1883 / 8883(TLS)     │          │  WS-Discovery: UDP/3702           │          │   RTSP: TCP/8554 (or 554)     │
│                               │          │  SOAP/HTTP: 80 or 8080            │          │   Paths: /cam/{cameraId}      │
│  Topics:                      │          │                                   │          │                               │
│   - site/{sid}/device/{did}/cmd   <== attacker inject/DoS                     │          │  Receives RTSP PUSH from FFmpeg|
│   - site/{sid}/device/{did}/state (retain)                                    │          │  Serves RTSP PLAY to clients   │
│   - site/{sid}/device/{did}/evt                                             │          │                               │
└───────────────┬───────────────┘          └───────────────┬───────────────────┘          └───────────────┬───────────────┘
                │                                          │                                              │
                │ MQTT cmd/state/evt                       │ ONVIF calls                                   │ RTSP URL referenced by ONVIF
                │                                          │  - GetCapabilities                             │  (GetStreamUri returns this)
                │                                          │  - GetProfiles (multi-channel)                 │
                │                                          │  - GetStreamUri  --> rtsp://.../cam/{id}       │
                │                                          │  - PTZ: ContinuousMove/Stop/GetStatus          │
                │                                          │                                              │
                v                                          v                                              v
┌───────────────────────────────┐          ┌───────────────────────────────────┐          ┌───────────────────────────────┐
│       Modbus Gateway          │          │   Unity Bridge (Control Plane)    │          │   Unity Bridge (Video Plane)  │
│   (Slave/Adapter)             │          │   (PTZ + Admin + Telemetry)       │          │   (Frame export to FFmpeg)    │
│   Port: TCP/502               │          │   Proto: WS/TCP/NamedPipe(local)  │          │   stdin rawvideo -> FFmpeg    │
│                               │          │                                   │          │                               │
│  Maps Modbus registers <~~>   │          │ Receives PTZ from ONVIF Server     │          │ Manages FFmpeg per camera     │
│  Unity Physical Truth         │          │ Sends PTZ to Unity camera objects  │          │ Pushes RTSP to MediaMTX       │
│                               │          │                                   │          │                               │
│  Example mapping (pump-01):   │          │  PTZ cmd (normalized):             │          │  FFmpeg per camera:           │
│   - Coil 00001 power          │          │   {camId, pan, tilt, zoom, speed}  │          │   -f rawvideo -pix_fmt rgba    │
│   - Coil 00002 alarm          │          │                                   │          │   -s WxH -r FPS -i -           │
│   - HR 40001 rpm              │          │                                   │          │   -> H.264 -> rtsp://...       │
│   - HR 40002 temp*10          │          │                                   │          │                               │
└───────────────┬───────────────┘          └───────────────┬───────────────────┘          └───────────────┬───────────────┘
                │                                          │                                              │
                │ Modbus R/W                                │ PTZ control                                   │ raw frames (RGBA)
                │ (attacker can spoof/write)                │ (attacker can force PTZ)                      │
                v                                          v                                              v

=================================================================================================================================================
OT / PLANT NETWORK (3D “Physical Truth” lives here)
=================================================================================================================================================

┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                               3D Space Server (Unity)                                                                      │
│                                  “Single Physical Truth” + “Multiple Logical Interfaces”                                                    │
│   - Owns device objects (pump/valve/door/...)                                                                                                  │
│   - Owns virtual PTZ camera objects (N)                                                                                                        │
│   - Applies MQTT + Modbus commands to physical simulation                                                                                      │
│   - Emits state + events back to MQTT / Modbus                                                                                                 │
│   - Renders camera views (RenderTexture) and exports frames                                                                                    │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                 ┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
                 │                                            UNITY SCENE GRAPH                                                 │
                 │                                                                                                              │
                 │   ┌──────────────────────────────┐                ┌────────────────────────────────────────────────────┐     │
                 │   │   Device Registry (SoT)       │                │      Camera Registry (SoT)                        │     │
                 │   │  - deviceId -> controller     │                │  - cameraId -> PTZ rig + RenderTexture            │     │
                 │   │  - physicalState (truth)      │                │  - streamPath: /cam/{cameraId}                    │     │
                 │   │  - logicalState.MQTT          │                │  - PTZ limits + smoothing                          │     │
                 │   │  - logicalState.Modbus        │                │  - overlay feed: device status + alarms            │     │
                 │   └───────────────┬───────────────┘                └───────────────────────────┬────────────────────────┘     │
                 │                   │                                                        │                              │
                 │                   │                                                        │                              │
                 │                   │                                                        │                              │
                 │                   v                                                        v                              │
                 │   ┌──────────────────────────────────────────┐               ┌─────────────────────────────────────────┐  │
                 │   │            Device Objects (examples)     │               │          Virtual PTZ Cameras            │  │
                 │   │                                          │               │                                         │  │
                 │   │ [pump-01] PumpController + MqttAgent      │               │ [cam-001] PTZ Rig                       │  │
                 │   │  - cmd: power, rpmTarget                  │               │   CameraRoot -> Yaw -> Pitch -> Lens    │  │
                 │   │  - state: power, rpm, temp, alarm         │               │   RenderTexture RT_001 (WxH@FPS)        │  │
                 │   │  - VFX: rotation, vibration, heat/smoke    │               │   Overlay: selected device state + evt  │  │
                 │   │                                          │               │                                         │  │
                 │   │ [valve-01] ValveController + MqttAgent     │               │ [cam-002] PTZ Rig                       │  │
                 │   │  - cmd: openPercent                        │               │   RenderTexture RT_002                  │  │
                 │   │  - state: openPercent, flow, alarm         │               │                                         │  │
                 │   │                                          │               │  PTZ INPUTS (from ONVIF):               │  │
                 │   │ [door-01] DoorController + MqttAgent        │               │   - ContinuousMove(pan,tilt,zoom,spd)   │  │
                 │   │  - cmd: lock/unlock/open                   │               │   - Stop()                               │  │
                 │   │  - state: locked, open, forcedEntryAlarm    │               │   - GetStatus()                          │  │
                 │   └──────────────────────────┬─────────────────┘               └──────────────────────┬──────────────────┘  │
                 │                              │                                                      │                     │
                 │                              │ publishes                                             │ renders              │
                 │                              │ MQTT state/evt (retain for state)                     │ RT -> frame export    │
                 │                              v                                                      v                     │
                 │                 ┌──────────────────────────────┐                    ┌───────────────────────────────────┐  │
                 │                 │   MQTT Publisher (Unity)     │                    │   Frame Export (Unity)            │  │
                 │                 │  - state retained             │                    │  - AsyncGPUReadback / ReadPixels │  │
                 │                 │  - evt non-retained           │                    │  - per-camera pacing (FPS)        │  │
                 │                 └──────────────────────────────┘                    └───────────────────┬───────────────┘  │
                 │                                                                                         │ raw RGBA bytes     │
                 └─────────────────────────────────────────────────────────────────────────────────────────┼────────────────────┘
                                                                                                           │
                                                                                                           v
                                                                                           ┌──────────────────────────────┐
                                                                                           │          FFmpeg             │
                                                                                           │  stdin rawvideo (RGBA)      │
                                                                                           │  -> H.264 (zerolatency)     │
                                                                                           │  -> RTSP PUSH to MediaMTX   │
                                                                                           └──────────────────────────────┘


=================================================================================================================================================
DATA CONTRACTS (what must match for interoperability)
=================================================================================================================================================

MQTT Topics (control + visibility)
  {cmd}   site/{sid}/device/{did}/cmd    (QoS1, ttl_ms, req_id)
  {state} site/{sid}/device/{did}/state  (QoS1, retain=true, physical truth snapshot)
  {evt}   site/{sid}/device/{did}/evt    (QoS1, non-retained, alarms/acks)

Modbus Mapping (legacy OT view/control)
  {coils}  00001.. : booleans (power, alarm, interlock)
  {holding}40001.. : numerics (rpm, temp*10, flow*10, ...)
  NOTE: Gateway must map R/W operations ~~> Unity SoT, and can simulate spoofing/latency.

ONVIF (camera discovery/control) + RTSP (video)
  {onvif}  WS-Discovery UDP/3702  -> device found
  {device} GetCapabilities / GetDeviceInformation / GetSystemDateAndTime
  {media}  GetProfiles (multi-channel) + GetStreamUri -> rtsp://<rtsp-server>/cam/{cameraId}
  {ptz}    ContinuousMove / Stop / GetStatus -> forwarded to Unity PTZ rig
  {rtsp}   H.264 live stream from MediaMTX path /cam/{cameraId}


=================================================================================================================================================
ATTACK SURFACE (what the attacker can do in the same time window)
=================================================================================================================================================

A) MQTT + CCTV concealment (Scenario #1)
  ==> MQTT inject:   site/demo/device/pump-01/cmd {power=1}
  ==> ONVIF PTZ:     ContinuousMove(cam-001, pan=..., tilt=...)  (turn camera away)
  RESULT: logical says “normal”, camera can’t see pump; physical pump runs.

B) Modbus spoof + MQTT normal + camera reveals truth (Scenario #2)
  ==> Modbus write:  40002(temp*10)=420 (fake normal temp)
  ==> MQTT state:    still “ok” (or attacker spoofs retained state)
  RESULT: camera shows smoke/overheat VFX; only “visual verification” catches it.

C) Camera denial + control continues (Scenario #3)
  ==> RTSP disrupt:  block/DoS stream or replace with stale feed
  ==> MQTT/Modbus:   keep controlling devices
  RESULT: operator can send commands but cannot verify on site (safety failure).


=================================================================================================================================================
MVP IMPLEMENTATION NOTES (so diagram is also a build plan)
=================================================================================================================================================

1) Start small: 3 devices + 2 cameras
   - pump-01, valve-01, door-01
   - cam-001, cam-002

2) Services (recommended process split)
   - MQTT Broker: Mosquitto
   - RTSP Server: MediaMTX
   - ONVIF Server: .NET (ASP.NET Core)
   - Modbus Gateway: .NET/Go/Python (maps to Unity)
   - Unity: physical truth + rendering

3) Ports (typical)
   - MQTT: 1883 / 8883
   - Modbus: 502
   - ONVIF: 3702 UDP + 80/8080 TCP
   - RTSP: 8554 (or 554)

4) “Camera is final authority” UX
   - CCTV overlay shows selected device live state + alarms
   - mismatch detector highlights when MQTT/Modbus view != physical truth
```
