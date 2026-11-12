# 🐟 bungeoppang2 — 붕어빵 로봇 (Doosan E0509)

손님이 오면 인사하고, 음성으로 주문을 받아, 두산 E0509 로봇팔이 반죽 붓기부터 속재료 넣기, 뚜껑 닫기, 뒤집기, 뚜껑 열기까지 붕어빵을 굽는 ROS 2 패키지입니다.
[bungeoppang](https://github.com/SujinJi/bungeoppang)(M1013 · MoveIt 2 기반)을 실물 E0509에 맞춰 처음부터 다시 만든 버전으로, 모션은 두산 `DSR_ROBOT2` API(`movej` / `movel`)로 직접 제어합니다.

| 항목 | 내용 |
|---|---|
| 로봇 | Doosan E0509 |
| 그리퍼 | ROBOTIS RH-P12-RN (DRL로 제어) |
| ROS | ROS 2 Jazzy, `ament_python` |
| 모션 | `DSR_ROBOT2` (`movej`, `movel`, `posj`, `posx`) |
| 음성 인식 | faster-whisper (한국어, 오프라인) |
| 사람 인식 | YOLOv8 (ultralytics) + 웹캠 |
| 시각화 | RViz2 (그리퍼 URDF, 작업대·물체 마커) |

---

## 동작 흐름

```
대기 ──손님 인식──▶ ㄱ자 인사 + 안내 멘트 ──음성 주문──▶ 끄덕 (위아래) ──▶ 굽기
                        │                     └─ 못 알아들음 → 도리도리 (좌우) → 다시 듣기
                        └─ 30초 동안 주문 없음 → 대기
굽기 완료 → 남은 주문이 있으면 이어서 굽기 / 없으면 다음 손님 대기
```

틀이 2개라서 한 판에 2개씩 굽고, 3개 이상 주문하면 판을 나눠 이어서 굽습니다.

### 한 판 굽는 순서 (`bake_sequence`)

| 단계 | 내용 |
|---|---|
| 0. 홈 | 홈 자세, 그리퍼 열기 |
| 1. 1차 반죽 | 비커 손잡이를 옆에서 잡기 → 주둥이를 생선 칸 위로 → 25° 기울여 붓기 → 수평 복귀 → 제자리 |
| 2. 속재료 | 소스통(팥·슈크림) 옆면을 깊게 잡기 → 입구 위로 든 채 이동 → 입구를 아래로 뒤집기 → 조이면서 아래 → 위로 짜기 → 제자리 |
| 3. 2차 반죽 | 1차와 같고, 기울인 채 1초 |
| 4. 뚜껑 닫기 | 가까운 판 손잡이를 옆에서 잡고 경첩을 따라 호를 그리며 먼 판 위에 덮기 |
| 5. 굽기 대기 | 밑면 익히기 |
| 6. 뒤집기 | 겹친 손잡이 두 개를 잡고 들어 손잡이 축으로 180° → 내려놓고 대기 → 되돌리기 |
| 7. 뚜껑 열기 | 위 손잡이 양옆을 잡고 90°까지 세움 → 바깥쪽에서 다시 잡아 175°까지 눕히기 |

중간에 멈췄을 때는 원하는 단계부터 다시 시작할 수 있습니다. 반죽과 소스는 먼 판(마주보는 틀)에 넣습니다(`positions.POUR_PLATE`).

### 안전 장치

- 모든 `movel` 목표는 `positions.check_pose()`로 먼저 검사합니다. 틀 방향 한계, 베이스로부터 최소·최대 반경, 작업대 가장자리 여유, 최저·최고 높이를 벗어나면 `[안전 정지]`로 멈춥니다.
- 실물 모드(`--real`)는 속도를 0.2배로 낮추고, 단계마다 Enter로 확인한 뒤 움직입니다.
- J6 회전 전에 관절 한계를 확인해 넘지 않는 방향을 자동으로 고릅니다.
- 굽는 동안에는 마이크 입력을 버려서 작업 중 소리가 주문으로 잡히지 않게 합니다.

---

## 노드 구성

```
 vision_node ──/bungeoppang/person──────▶ ┌──────────────┐ ──/bungeoppang/start_bake──▶ ┌───────────────┐
 (웹캠 + YOLO)                             │ orchestrator │ ──/bungeoppang/gesture─────▶ │ bake_sequence │
                                          │ (상태 머신)    │ ◀──/bungeoppang/bake_done─── │   --serve     │
 voice_node ──/bungeoppang/order────────▶ │              │                              └───────┬───────┘
 (마이크)     /bungeoppang/voice_text ───▶ │              │ ──/bungeoppang/listening──▶ voice_node    │
                                          └──────┬───────┘                                     │
                                                 └─/bungeoppang/say (안내 멘트, TTS 연결용)       │
                                                                  /bungeoppang/gripper_width ◀──┘
                                                                          ▼
                                                                    gripper_viz ──▶ RViz2
```

| 실행 이름 | 파일 | 역할 |
|---|---|---|
| `bake_sequence` | `bake_sequence_node.py` | 전체 굽기 공정. 단독 1회 실행 또는 `--serve`로 주문 대기 |
| `flip_mold` | `flip_mold_node.py` | 틀 뒤집기 동작만 따로 테스트 |
| `orchestrator` | `orchestrator_node.py` | 상태 머신 `IDLE → LISTEN → ORDERED → BAKING`, 주문 큐 관리 |
| `voice_node` | `voice_node.py` | 소리 크기로 말소리 구간을 잘라 인식, 주문으로 해석되면 `팥x2,슈크림x1` 형식으로 발행 |
| `text_order` | `voice_node.py` | 마이크 없이 키보드로 문장을 입력해 같은 토픽으로 발행 (테스트용) |
| `vision_node` | `vision_node.py` | 사람이 연속으로 보이면 도착(`True`), 일정 시간 안 보이면 떠남(`False`) |
| `gripper_viz` | `gripper_viz.py` | 그리퍼 폭을 손가락 관절값으로 바꾸고, 작업대·안전 한계·물체를 마커로 표시 |

### 설정 파일

조절하는 값은 두 파일에 모여 있고, 바꿀 만한 값에는 `★` 표시가 있습니다.

| 파일 | 내용 |
|---|---|
| `positions.py` | 좌표 약속, 작업대·받침대 치수, 안전 영역, 물체 크기, 티칭 좌표, 굽는 시간, 속도, 제스처 자세 |
| `grip_params.py` | 물체별 그리퍼 접근 폭, 잡는 폭, 힘(current), 소스 짜는 폭·시간 |
| `workcell.py` | 예전 import 호환용 (위 두 파일을 다시 내보냄) |
| `gripper_drl.py` | 그리퍼 폭 → 위치값 변환, `DrlStart` 서비스로 DRL 코드 실행 |
| `gripper_ctrl.py` | 실물 모드에서는 DRL 실행, 가상 모드에서는 RViz 표시만 |

**좌표 약속:** 틀이 놓인 작업대 긴 방향이 로봇 ±X, 짧은 방향이 ±Y입니다. 실물에서 티치펜던트로 +X 조그를 눌러 틀 쪽으로 가면 `MOLD_SIGN = +1`, 반대면 `-1`로 바꿉니다. 베이스를 180° 돌려 설치했다면 `MOLD_SIGN`과 `SHORT_SIGN`을 함께 뒤집습니다.

### 음성 주문 인식

faster-whisper `small` 모델로 인식하고, 메뉴 힌트를 함께 줘서 '팥'과 '슈크림'을 더 잘 알아듣게 했습니다. 해석기는 여러 종류 동시 주문("팥 둘, 슈크림 하나"), 빼기("슈크림은 빼고"), "~씩", 그리고 자주 틀리는 발음까지 처리합니다. 주문으로 해석되지 않는 말이 들어오면 orchestrator가 도리도리 후 다시 묻습니다.

---

## 설치

```bash
# 시스템 패키지
sudo apt install -y libportaudio2

# Python 패키지
pip install faster-whisper sounddevice numpy ultralytics opencv-python --break-system-packages

# 워크스페이스에 두산 ROS 2 패키지(dsr_bringup2, dsr_msgs2, DSR_ROBOT2)와
# ROBOTIS rh_p12_rn_a_description이 있어야 합니다
cd ~/ros2_ws/src
git clone https://github.com/SujinJi/bungeoppang2.git
cd ~/ros2_ws
colcon build --packages-select bungeoppang2
source install/setup.bash
```

YOLO 모델(`yolov8n.pt`)은 `vision_node`를 처음 실행할 때 자동으로 내려받습니다.

---

## 실행

### 시뮬레이션: 공정만 (`sim.launch.py`)

두산 E0509 에뮬레이터와 전용 RViz를 띄우고, 20초 뒤 굽기 공정을 한 번 자동 실행합니다.

```bash
ros2 launch bungeoppang2 sim.launch.py
ros2 launch bungeoppang2 sim.launch.py f1:=슈크림 f2:=팥   # 틀 1·2 속재료 지정
ros2 launch bungeoppang2 sim.launch.py bake:=false         # 에뮬레이터 + RViz만
ros2 launch bungeoppang2 sim.launch.py bake_delay:=25.0    # 에뮬레이터 준비 대기 시간
```

### 시뮬레이션: 손님 응대까지 (`demo.launch.py`)

에뮬레이터, RViz, 공정(`--serve`), 총괄, 음성 인식을 터미널 하나에서 실행합니다.

```bash
ros2 launch bungeoppang2 demo.launch.py
```

다른 터미널에서 손님 도착을 보내고 마이크에 "팥 주세요"라고 말하면 인사 → 끄덕 → 굽기가 이어집니다.

```bash
ros2 topic pub --once /bungeoppang/person std_msgs/msg/Bool "{data: true}"
```

마이크 없이 테스트하려면 `voice:=false`로 실행한 뒤 텍스트로 주문합니다.

```bash
ros2 launch bungeoppang2 demo.launch.py voice:=false
ros2 run bungeoppang2 text_order          # 예: "팥 하나 슈크림 하나 주세요"
# 또는
ros2 topic pub --once /bungeoppang/order std_msgs/msg/String "{data: 팥x1}"
```

### 실물 로봇

```bash
ros2 run bungeoppang2 bake_sequence --real              # 단계마다 Enter로 확인
ros2 run bungeoppang2 bake_sequence --real --no-step    # 연속 실행
ros2 run bungeoppang2 bake_sequence --real --serve      # 주문 대기
ros2 launch bungeoppang2 interaction.launch.py          # 사람 인식 + 음성 인식 + 총괄
ros2 run bungeoppang2 flip_mold --real                  # 뒤집기만 테스트
```

### RViz 보기만

```bash
ros2 launch bungeoppang2 gripper_view.launch.py
```

로봇·그리퍼·작업대·물체가 미리 추가된 RViz 설정(`rviz/bungeoppang.rviz`)으로 열립니다. 그리퍼는 ROBOTIS 저장소에 RN-A 모델만 있어서 RN-A 메쉬로 표시하며, 외형만 조금 다릅니다.

---

## 주요 토픽

| 토픽 | 타입 | 발행 → 구독 | 내용 |
|---|---|---|---|
| `/bungeoppang/person` | `Bool` | vision_node → orchestrator | 손님 도착 / 떠남 |
| `/bungeoppang/person_count` | `Int32` | vision_node | 현재 보이는 사람 수 |
| `/bungeoppang/voice_text` | `String` | voice_node → orchestrator | 인식된 문장 전체 |
| `/bungeoppang/order` | `String` | voice_node → orchestrator | `팥x2,슈크림x1` |
| `/bungeoppang/listening` | `Bool` | orchestrator → voice_node | 지금 들어도 되는지 |
| `/bungeoppang/start_bake` | `String` | orchestrator → bake_sequence | `팥,슈크림@번호` |
| `/bungeoppang/gesture` | `String` | orchestrator → bake_sequence | `greet` / `yes` / `no` |
| `/bungeoppang/bake_done` | `Bool` | bake_sequence → orchestrator | 성공 / 안전 정지 |
| `/bungeoppang/say` | `String` | orchestrator | 안내 멘트 (TTS 연결용) |
| `/bungeoppang/gripper_width` | `Float64` | gripper_ctrl → gripper_viz | 그리퍼 폭 (mm) |

---

## 디렉터리 구조

```
bungeoppang2/
├── bungeoppang2_yammy/
│   ├── bake_sequence_node.py   # 전체 굽기 공정
│   ├── flip_mold_node.py       # 뒤집기 단독
│   ├── orchestrator_node.py    # 상태 머신
│   ├── voice_node.py           # 음성 인식 + 주문 해석
│   ├── vision_node.py          # 사람 인식
│   ├── gripper_viz.py          # RViz 표시
│   ├── gripper_ctrl.py         # 그리퍼 제어 래퍼
│   ├── gripper_drl.py          # DRL 그리퍼 제어
│   ├── positions.py            # ★ 위치·치수·안전·속도
│   ├── grip_params.py          # ★ 그리퍼 폭·힘
│   └── workcell.py             # 호환용
├── launch/
│   ├── sim.launch.py           # 에뮬레이터 + RViz + 공정 1회
│   ├── demo.launch.py          # 에뮬레이터 + 손님 응대 전체
│   ├── interaction.launch.py   # 사람 인식 + 음성 + 총괄
│   └── gripper_view.launch.py  # RViz 표시만
├── urdf/gripper_on_e0509.urdf.xacro
├── rviz/bungeoppang.rviz
├── resource/
├── package.xml  setup.py  setup.cfg
```

---

## 진행 상황

- [x] E0509 에뮬레이터에서 반죽 → 속재료 → 뚜껑 닫기 → 뒤집기 → 뚜껑 열기 공정
- [x] 손님 인식(YOLO), 음성 주문, 총괄 상태 머신, 인사·끄덕·도리도리 제스처
- [x] 안전 영역 검사, 실물 단계별 확인 모드
- [ ] 꼬챙이로 붕어빵 꺼내 손님 컵에 담기 (좌표·치수는 `positions.py`에 준비됨)
- [ ] 실물 RH-P12-RN DRL 코드 연결 (`gripper_drl.py`의 `DRL_TEMPLATE`)
- [ ] 임시값 실측 교체 (회전 손잡이 위치, 꼬챙이 꽂이 위치, 손가락 길이·두께 등)
- [ ] TTS로 안내 멘트 읽기

## License

MIT
