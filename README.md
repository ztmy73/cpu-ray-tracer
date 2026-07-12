# CPU Ray Tracer (진행 중)

C++로 밑바닥부터 짠 레이트레이서입니다. *Ray Tracing in One Weekend*(CPU 버전)을 따라가며
**직접 구현하고, 막힌 지점마다 저수준에서 이유를 파고든 학습 기록**을 함께 남깁니다.
최종 목표는 이 CPU 버전을 **CUDA로 포팅**해 "CPU 대비 GPU 가속" 결과를 만드는 것입니다.

> 이 저장소는 **완성작이 아니라 진행 중인 학습 기록**입니다.
> 현재 diffuse(람베르시안) 재질 + 감마 보정까지 구현해 음영이 있는 렌더링에 성공했고,
> 다음 단계는 컬러 재질(metal/dielectric)입니다.

관심 분야는 **실시간·인터랙티브 그래픽스와 GPU 저수준**이고, 이 프로젝트는
그래픽스 파이프라인 이해 + GPU 하드웨어/메모리 모델 이해를 하나의 결과물로 잇기 위한 것입니다.

---

## 결과 이미지

<!-- 대표 이미지: diffuse + 감마 적용한 음영 있는 공 -->
![diffuse](images/04_diffuse.png)
*diffuse(람베르시안) 재질 + 감마 보정. 바닥(거대한 구) 위에 놓인 매트한 공, 부드러운 음영과 그림자.*

### 단계별 진화

<!-- 아래 이미지들을 images/ 폴더에 넣고 링크를 활성화하세요 -->

| | |
|---|---|
| <img width="400" height="225" alt="image" src="https://github.com/user-attachments/assets/34be3074-8692-49ed-91c9-c54a9c664ba3" />
① 배경 그라데이션 (뷰포트 + lerp)* | <img width="400" height="225" alt="image" src="https://github.com/user-attachments/assets/646eb4af-eb04-44b3-a183-88be598c740c" />
*② 판별식 판정으로 그린 구* |
| <img width="600" height="338" alt="안티앨리어싱 전" src="https://github.com/user-attachments/assets/8adfec60-89f4-47ab-ae95-93bfa8d4a1cf" />
 />
*③ 표면 법선 시각화 (N → RGB)* |<img width="599" height="335" alt="diffuse후" src="https://github.com/user-attachments/assets/b229286f-a410-48e4-b11b-4ada944a44e0" />
*④ diffuse + 감마 (최종)* |

### 옵션: before / after 비교

이해한 내용을 시각적으로 보여주기 위한 비교 자료입니다.


## 지금까지 구현한 것

| 단계 | 내용 | 상태 |
|------|------|------|
| `vec3` | 3D 벡터/색/좌표. 연산자 오버로딩, const-correctness | ✅ |
| `ray` | `P(t) = A + t·b` 표현, `at(t)` | ✅ |
| 배경 렌더 | 뷰포트 좌표계 + 방향 기반 lerp → PPM 출력 | ✅ |
| 광선-구 교차 | 2차방정식 판별식 판정, 가까운 근(`t`) 선택 | ✅ |
| 표면 법선 시각화 | `N = unit(P − C)`를 색으로 매핑 | ✅ |
| `hittable` / `hittable_list` | 다중 객체용 추상화 + `hit_record`, 최근접 선택 | ✅ |
| `camera` + 안티에일리어싱 | 렌더 루프 캡슐화, 픽셀당 다중 샘플 평균 | ✅ |
| diffuse (람베르시안) | 랜덤 방향 반사 + 재귀 `ray_color`, 바운스 깊이 제한 | ✅ |
| 감마 보정 | 선형 → 감마 공간(sqrt) 변환 | ✅ |
| 컬러/금속/유리 재질 | `material` 추상 + lambertian/metal/dielectric | ⬜ 예정 |

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

렌더가 느리면 `camera`의 `samples_per_pixel`을 낮추세요(품질↓ 속도↑).

---

## 프로젝트 구조

헤더 온리(header-only) 구조입니다. 정의를 헤더에 두기 때문에 자유 함수에는 `inline`을 붙여
ODR(One Definition Rule) 위반을 피합니다.

```
.
├── rtweekend.h      # 공용 상수/유틸 (infinity, random_double)
├── vec3.h           # 벡터/색 (+ point3, color 별칭), 연산자 오버로딩, 랜덤 벡터
├── ray.h            # 광선 P(t) = A + t·b
├── color.h          # write_color: 0~1 색 → 0~255 PPM, 감마 보정
├── hittable.h       # hit_record + hittable 추상 베이스
├── sphere.h         # hittable 상속, 광선-구 교차 hit 구현
├── hittable_list.h  # 다중 객체 순회 + 가장 가까운 교차 선택
├── camera.h         # 뷰포트/렌더 루프 + 안티에일리어싱 + 재귀 ray_color
└── main.cpp         # 장면 구성 + cam.render(world)
```

렌더 루프의 뼈대는 이후로 바뀌지 않습니다. **"색 계산"만 점점 정교해집니다.**

```
픽셀마다 → 픽셀당 여러 광선을 랜덤 생성 → ray_color()로 색 계산(재귀 반사) → 평균 → PPM 출력
```

---

## 로드맵

CPU 버전을 끝까지 구현한 뒤 CUDA로 넘어갑니다.

- [x] `hittable` / `hittable_list` — 다중 객체
- [x] 안티에일리어싱 (픽셀당 다중 샘플)
- [x] **Diffuse 재질** — "레이트레이서다운" 첫 이미지 확보
- [ ] Metal / Dielectric 재질 (컬러 반사·굴절)
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
GPU에는 `shared_ptr`가 없습니다. `hittable_list`가 자식 객체들을 `shared_ptr`로 들고 있는데,
이를 raw 포인터 + 수동 관리로 바꿔야 합니다.

**3. 재귀 `ray_color` → 반복 루프**
현재 diffuse 렌더러는 `ray_color`가 자기를 부르는 재귀 구조입니다(바운스마다 한 번).
GPU에서는 깊은 재귀가 부담이므로, 이를 **바운스 깊이 상한을 둔 반복문**으로 변환합니다.

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

구현하면서 실제로 막히고 뚫은 개념들을 학습 노트에 정리했습니다.
연산자 오버로딩을 왜 자유 함수로 두는지, `inline`의 ODR 의미, 판별식 부호가 교차 개수를 정하는 이유,
변수 섀도잉으로 생긴 조용한 버그, diffuse에서 랜덤 방향을 뽑을 때 구 밖을 버리는 이유 등,
"코드는 됐지만 왜 되는지"를 파고든 기록입니다.

- [`LEARNING_NOTES.md`](./LEARNING_NOTES.md) — vec3부터 hittable까지 (1~6)
- [`LEARNING_NOTES_2.md`](./LEARNING_NOTES_2.md) — 다중 객체부터 diffuse·감마까지 (7~17)

---

## 참고

- Peter Shirley et al., *Ray Tracing in One Weekend*
  (https://raytracing.github.io/) — 구조와 알고리즘의 출발점으로 삼았습니다.
- 이 저장소의 코드는 위 교재를 따라 직접 타이핑·수정하며 구현한 것이고,
  학습 노트는 구현 과정에서 스스로 던진 질문들의 기록입니다.
