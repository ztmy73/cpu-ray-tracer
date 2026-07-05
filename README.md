# CPU Ray Tracer

C++로 밑바닥부터 구현하는 레이트레이서.
*Ray Tracing in One Weekend* (Peter Shirley)를 레퍼런스로 기초를 다지고,
이후 CUDA 포팅으로 GPU 가속을 측정하는 것을 목표로 한다.

> 🔨 **진행 중** — 현재 구(sphere) 교차까지 구현. 재질 단계로 진행 중.

**환경:** C++ / `g++` (CPU 단계) → 이후 `nvcc -arch=sm_89` (CUDA 포팅 예정)

## 진행 상황
- [x] `vec3` — 연산자 오버로딩, const-correctness, 헤더 `inline`/ODR 처리
- [x] `ray` — 멤버 이니셜라이저 리스트, `P(t) = A + t·b`
- [x] 첫 렌더 — 뷰포트 좌표계 + `lerp` 기반 `ray_color` → PPM 하늘 그라데이션
- [x] 구 교차 — 판별식 기반 `hit_sphere`
- [ ] 법선·음영 / 다중 물체(`hittable_list`)
- [ ] 안티에일리어싱 / 재질(diffuse·metal·dielectric)
- [ ] 카메라 · 디포커스 블러

## 렌더 결과
<!-- diffuse 이미지 확보되면 여기에 대표 이미지 -->

## 배운 점
헤더 온리 구조에서 `inline`이 성능이 아니라 ODR(단일 정의 규칙)을 위한
것이라는 점, `this`/`*this`와 연산자 오버로딩, 뷰포트→월드 좌표 매핑,
판별식으로 광선-구 교차를 판정하는 원리 등을 직접 구현하며 익혔다.

## 다음 단계: CUDA 포팅
CPU 완성 후 렌더 루프(픽셀별 `ray_color`)를 CUDA 커널로 이식할 예정.
미리 파악한 핵심 난관 3가지:
1. 상속 + 가상함수 **vtable** — host에서 생성한 객체를 device로 옮길 때의 문제
2. `shared_ptr` 부재 → **raw 포인터 수동 관리**
3. 재귀 `ray_color` → **반복문 + 바운스 깊이 제한** 변환

목표: 동일 씬을 CPU vs GPU로 렌더해 `cudaEvent`로 가속 배수 측정.

## 레퍼런스
Peter Shirley, *Ray Tracing in One Weekend*.
