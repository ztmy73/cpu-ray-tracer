# CPU Ray Tracer (진행 중)

C++로 밑바닥부터 짠 레이트레이서입니다. *Ray Tracing in One Weekend*(CPU 버전)을 따라가며
**직접 구현하고, 막힌 지점마다 저수준에서 이유를 파고든 학습 기록**을 함께 남깁니다.
최종 목표는 이 CPU 버전을 **CUDA로 포팅**해 "CPU 대비 GPU 가속" 결과를 만드는 것입니다.

> 이 저장소는 **완성작이 아니라 진행 중인 학습 기록**입니다.
> 현재 광선-구 교차와 표면 법선 시각화까지 렌더링에 성공했고,
> 다중 객체를 다루기 위한 `hittable` 추상화로 리팩터링하는 중입니다.

관심 분야는 **실시간·인터랙티브 그래픽스와 GPU 저수준**이고, 이 프로젝트는
그래픽스 파이프라인 이해 + GPU 하드웨어/메모리 모델 이해를 하나의 결과물로 잇기 위한 것입니다.

---

## 지금까지 구현한 것

| 단계 | 내용 | 상태 |
|------|------|------|
| `vec3` | 3D 벡터/색/좌표. 연산자 오버로딩, const-correctness | ✅ |
| `ray` | `P(t) = A + t·b` 표현, `at(t)` | ✅ |
| 배경 렌더 | 뷰포트 좌표계 + 방향 기반 lerp → PPM 출력 | ✅ |
| 광선-구 교차 | 2차방정식 판별식 판정, 가까운 근(`t`) 선택 | ✅ |
| 표면 법선 시각화 | `N = unit(P − C)`를 색으로 매핑 | ✅ |
| `hittable` 추상화 | 다중 객체용 인터페이스 + `hit_record` | 🔨 진행 중 |

렌더 결과 (3장):

```
images/
├── 01_gradient.png    # 하늘색 그라데이션 배경
├── 02_red_sphere.png  # 판별식 판정으로 그린 빨간 원
└── 03_normals.png     # 법선을 색으로 시각화한 입체 구
```

> `images/` 폴더를 만들고 위 3장을 넣은 뒤, 아래에 이미지 링크를 붙이면 됩니다.
> 예: `![normals](images/03_normals.png)`

---

## 빌드 & 실행

CUDA가 아니라 순수 C++입니다. `g++`만 있으면 됩니다.

```bash
g++ -std=c++17 -o main main.cpp
./main > image.ppm
```

- 픽셀 데이터는 `std::cout` → 파일로 리다이렉트
- 진행률은 `std::clog` → 터미널에 그대로 출력 (이 분리 때문에 `> image.ppm`이 깔끔하게 동작)
- PPM 확인: VS Code PPM 뷰어 확장, 또는 `convert image.ppm image.png` (ImageMagick)

---

## 프로젝트 구조

헤더 온리(header-only) 구조입니다. 정의를 헤더에 두기 때문에 자유 함수에는 `inline`을 붙여
ODR(One Definition Rule) 위반을 피합니다.

```
.
├── vec3.h       # 벡터/색 (+ point3, color 별칭), 연산자 오버로딩
├── ray.h        # 광선 P(t) = A + t·b
├── color.h      # write_color: 0~1 색 → 0~255 PPM
├── hittable.h   # (진행 중) hit_record + hittable 추상 베이스
└── main.cpp     # 카메라/뷰포트 셋업 + 렌더 루프 + ray_color
```

렌더 루프의 뼈대는 이후로 바뀌지 않습니다. **"색 계산"만 점점 정교해집니다.**

```
픽셀마다 → 그 픽셀을 향하는 광선 생성 → ray_color()로 색 계산 → PPM 출력
```

---

## 로드맵

CPU 버전을 끝까지 구현한 뒤 CUDA로 넘어갑니다.

- [ ] `hittable` / `hittable_list` — 다중 객체
- [ ] 안티에일리어싱 (픽셀당 다중 샘플)
- [ ] **Diffuse 재질** ← "레이트레이서다운" 첫 이미지 확보 지점
- [ ] Metal / Dielectric 재질
- [ ] 이동 가능한 카메라, 디포커스 블러
- [ ] **CUDA 포팅** (아래 계획 참고)

---

## CUDA 포팅 계획

CPU 구현을 GPU로 옮길 때 **미리 예상되는 3가지 마찰점**입니다.
포팅 전에 이걸 짚어두는 이유는, RTIOW → CUDA에서 흔히 깨지는 지점이 정확히 여기이기 때문입니다.

**1. 상속 + 가상 함수 (vtable)**
`hittable`/`material`은 CPU에선 자연스러운 다형성이지만, host에서 만든 객체를 그대로
`cudaMemcpy`하면 vptr이 **host의 vtable**을 가리켜 device에서 깨집니다.
→ device 커널 안에서 객체를 생성해 vtable이 device 쪽을 가리키게 해야 합니다.

**2. `shared_ptr` 부재**
GPU에는 `shared_ptr`가 없습니다. 객체 소유권을 raw 포인터 + 수동 관리로 바꿔야 합니다.
(`hittable_list`가 자식 객체들을 어떻게 들고 있느냐가 그대로 영향받는 부분)

**3. 재귀 `ray_color` → 반복 루프**
GPU에서는 깊은 재귀가 부담이므로, `ray_color`의 재귀를
**바운스 깊이 상한을 둔 반복문**으로 변환합니다.

### 측정에 대한 태도

이 프로젝트의 목표 중 하나는 "CPU 대비 GPU 몇 배"라는 정량 결과입니다.
다만 별도 CUDA 워밍업에서, **이론상 우월한 최적화(tiled matmul)가 특정 하드웨어·문제 크기에서는
naive 대비 1.07~1.35배에 그친** 경험을 했습니다. 여러 가설을 측정으로 하나씩 반증했지만,
프로파일러 없이는 원인을 확정할 수 없다는 한계까지 확인했습니다.
그래서 가속 결과는 **측정값 + 측정 조건**을 함께 기록하고, 확정할 수 없는 부분은 그대로 명시할 계획입니다.

> 환경 메모: 워밍업은 WSL2에서 진행했고, `ncu`/`compute-sanitizer`가 GPU 패스스루 한계로
> 동작하지 않아 `cudaEvent` 기반 타이밍 + 수동 가설검증을 사용했습니다.
> 정밀 프로파일링은 네이티브 Linux/Windows 환경에서 이어갈 예정입니다.

---

## 학습 기록

구현하면서 실제로 막히고 뚫은 개념들을 [`LEARNING_NOTES.md`](./LEARNING_NOTES.md)에 정리했습니다.
연산자 오버로딩을 왜 자유 함수로 두는지, `inline`의 ODR 의미, 판별식 부호가 교차 개수를 정하는 이유 등,
"코드는 됐지만 왜 되는지"를 파고든 기록입니다.

## 참고

- Peter Shirley et al., *Ray Tracing in One Weekend*
  (https://raytracing.github.io/) — 구조와 알고리즘의 출발점으로 삼았습니다.
- 이 저장소의 코드는 위 교재를 따라 직접 타이핑·수정하며 구현한 것이고,
  학습 노트는 구현 과정에서 스스로 던진 질문들의 기록입니다.
