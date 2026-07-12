# 학습 노트 2 — 다중 객체부터 diffuse까지

(이전 노트가 `hittable` 예습에서 끝났으니 그 다음부터. 구 여러 개 → 카메라/안티에일리어싱 → diffuse → 감마 순서로 실제 구현하면서 막힌 것들 정리.)

---

## 전체 구조 — 사실 수업 때 짠 거랑 같음

컴그 수업에서 "광선마다 sphere 리스트 돌면서 부딪히나 검사하고, 가장 가까운 거 고르기"를 짰었는데,
RTIOW도 알고리즘은 완전히 똑같다. 다만 그걸 세 조각으로 나눴을 뿐:

- `sphere::hit` — "나(구 하나) 이 광선에 맞아?" 판별식 풀고 맞으면 정보 채움. **자기 하나만** 앎.
- `hittable_list::hit` — "내 리스트의 모든 물체 중 가장 가까운 건?" for로 순회 + `closest_so_far`로 최근접 고름. → **이게 수업 때 짠 그 for 루프.**
- `hittable` — 위 둘의 공통 인터페이스(추상).

수업 때 "이 구 맞나?"랑 "이게 제일 먼저 맞나?"가 한 덩어리로 엉켜서 찜찜했는데,
여기선 sphere가 "맞나?"만, list가 "가장 가까운가?"만 담당하게 분리돼서 깔끔해짐.

**`hittable_list`도 `hittable`을 상속**한다는 게 포인트. 그래서 `ray_color`는 `world.hit()` 하나만 부르면 됨.
world가 구 1개든 100개든 신경 안 씀. **물체 하나여도 리스트에 넣는다** — 인터페이스를 하나로 고정하려고.
(리스트도 hittable이라, 나중에 리스트 안에 리스트 넣는 식으로 BVH 트리까지 확장되는 씨앗. 지금은 안 씀.)

---

## hit_record는 어떻게 전달되나

`hit`은 `bool`만 반환한다(맞았냐/아니냐). 실제 정보(교차점 `p`, 법선 `normal`, `t`, `front_face`)는
`hit_record& rec`을 **참조로 받아서 함수 안에서 채운다.** `operator+=`에서 참조로 원본 고치던 거랑 같은 패턴.
호출하는 쪽이 빈 `hit_record`를 만들어 넘기면, `hit`이 그 안을 채워주는 방식.

`normal`은 따로 떠도는 변수가 아니라 `rec`의 멤버 한 칸(`rec.normal`)이다. 여기서 잠깐 헷갈렸음.

### rec은 광선당 하나. 구 개수랑 무관.

- `rec` — `ray_color`가 광선 하나당 하나 만듦. 최종 승자 자리.
- `temp_rec` — 리스트 for 루프 안, 각 구가 매번 덮어쓰는 후보 그릇.

구가 100개여도 `hit_record`는 **딱 2개**(rec, temp_rec)만 존재하고 재사용함.
`temp_rec`을 따로 두는 이유: 구 하나 검사한 결과를 바로 `rec`에 쓰면, 나중에 더 먼 구가 `rec`을 덮어써서
좋은 정보를 날림. 그래서 임시로 받고 "더 가까울 때만" `rec`으로 옮김. (후보 → 검증 → 커밋. 트랜잭션 느낌.)

---

## sphere::hit 짜면서 낸 버그 3개 (기록용, 다 배울 만했음)

`hit_sphere`의 판별식 로직을 클래스 안 `hit`으로 옮기면서 낸 실수들.

**1. `override` + 끝 `const` 누락 → 컴파일 에러**
부모 `hittable::hit`은 끝에 `const`가 붙어 있는데 내 sphere 쪽엔 없었음.
시그니처가 하나라도 다르면 "부모 계약을 채운 것"으로 인정 안 됨 → sphere가 여전히 추상 클래스 →
객체 생성 불가 → 에러. `override` 붙이면 이런 시그니처 불일치를 컴파일러가 바로 잡아줌.
**앞으로 override할 땐 무조건 `override` 붙이기.**

**2. 연산자 우선순위 — `/ 2*a`**
`(-b - sqrt(D)) / 2*a` 라고 썼는데, `/`랑 `*`는 우선순위 같고 왼→오라서
실제론 `((-b - sqrt(D)) / 2) * a`로 계산됨. `2*a`로 나누는 게 아님.
`a`가 1이 아니면(광선 방향이 단위벡터가 아니면 a≠1) t가 완전히 틀림.
→ `/ (2*a)`로 괄호. **의심되면 괄호.**

**3. 변수 섀도잉 — 이게 제일 위험 (컴파일 되는데 논리 틀림)**
```cpp
auto root = (-b - sqrt(D)) / (2*a);      // 바깥 root (첫 근)
if (범위 밖) {
    auto root = (-b + sqrt(D)) / (2*a);  // ← auto 붙여서 새 변수! 안쪽 스코프
    ...
}                                        // ← 안쪽 root 여기서 소멸
rec.p = r.at(root);                      // ← 이 root는 바깥(첫 근). 두 번째 근 안 씀!
```
안쪽 `if`에서 `auto root`로 또 선언하면 **새 변수**가 됨(이름만 같음). `}` 나가면 사라짐.
그래서 두 번째 근을 제대로 계산해놓고도 그걸 못 쓰고 첫 근(범위 밖)으로 rec 채움.
→ 안쪽에서 `auto` 빼고 그냥 `root = ...`로 **대입**. 바깥 root 갱신.

`auto root`(새 변수) vs `root`(기존에 대입) — 이 차이가 핵심. 컴파일러가 안 잡아주는 버그라 제일 조심.

### 섀도잉 원리 (수업 때 느낌으로만 알던 거 확실히)
안쪽 블록 변수는 그 블록 진입할 때 스택에 잡히고 `}`에서 되감기며 사라진다.
이름 찾기는 항상 "가장 안쪽 스코프 → 바깥"으로 훑어서, 안쪽에 같은 이름 있으면 거기서 멈춤(바깥 가림).
안쪽이 소멸하면 그 이름 없어지니 다시 바깥에서 찾음. → 버그 2번의 정체가 이거였음.
두 번째 근이 틀리게 계산된 게 아니라, **계산 결과가 스택과 함께 증발해서 도달을 못 한** 버그.

---

## hittable_list for 루프 — 여기서도 삽질

```cpp
for (const auto& object : objects) {
    if (object->hit(r, ray_tmin, closest_so_far, temp_rec)) {
        hit_anything = true;
        closest_so_far = temp_rec.t;   // ← 이게 정답
        rec = temp_rec;
    }
}
```

처음에 `closest_so_far`에 거리(`length()`)를 계산해서 넣으려다 틀림. 이유 두 개:

- `closest_so_far`는 `ray_tmax` 자리에 들어가서 `sphere::hit` 안에서 `t`랑 비교됨. 그러니 **t 단위**여야 함.
  근데 우리 광선은 방향을 정규화 안 해서(`pixel_center - camera_center` 그대로) **t ≠ 실제 거리**. length 넣으면 스케일 안 맞음.
- `rec.p`를 쓰려 했는데, `rec`은 그 아래 줄 `rec = temp_rec`에서야 갱신됨. 방금 맞은 정보는 `temp_rec`에 있음.

→ `t`는 이미 `sphere::hit`이 `temp_rec.t`에 계산해뒀으니 **그냥 재활용**. 거리 새로 구할 필요 없음(sqrt도 안 함).
"필요한 값이 이미 계산돼 있진 않나?" 먼저 보는 습관. 픽셀마다 수백만 번 도는 코드라 sqrt 하나가 쌓임.

`object->`인 이유: objects가 포인터(shared_ptr)라서. `virtual` 덕에 각자 실제 타입(sphere)의 hit이 불림.

---

## main / 장면 구성

```cpp
world.add(make_shared<sphere>(point3(0, 0, -1), 0.5));       // 가운데 구
world.add(make_shared<sphere>(point3(0, -100.5, -1), 100));  // 바닥
```

- **바닥 트릭**: 반지름 100짜리 거대한 구를 아래(`y=-100.5`)에 둠. 표면 꼭대기가 `y=-0.5` 근처.
  너무 커서 곡률이 안 느껴져 **평면처럼** 보임. 진짜 평면 클래스 안 만들고 큰 구로 바닥 흉내.
- `world.hit(r, 0, infinity, rec)`: `0`이 카메라 뒤(`t<0`) 거름. sphere에서 못 걸렀던 문제 여기서 해결.
- **바닥이 연두색인 이유** (버그 아님): 바닥 표면 법선이 거의 위(0,1,0) →
  `0.5*((0,1,0)+(1,1,1)) = (0.5, 1.0, 0.5)` → G 최대 = 연두. 정상.

---

## 상속 / virtual 다시 정리 (vtable까지)

- 부모(`hittable`) = 계약만 선언(`= 0`, 순수 가상). 자식(`sphere`) = 구체적 구현. 매개변수 다 똑같이(끝 const까지).
- `virtual`: `hittable*` 포인터로 불러도 **실제 가리키는 객체(sphere)의 hit**이 불리게 하는 스위치.
  없으면 포인터 타입만 보고 부모 걸 부르려 함.
- 메커니즘: virtual 함수 있는 객체는 내부에 vtable 포인터를 품음. `obj->hit()` = "obj가 품은 vtable 따라가서
  거기 적힌 hit 주소로 점프". 그래서 포인터 타입이 hittable*여도 객체가 품은 vtable이 sphere 거라 sphere::hit으로 감.
  컴파일 타임이 아니라 런타임에 결정(동적 디스패치).

> **CUDA 북마크**: host에서 만든 sphere의 vtable 포인터는 host 메모리 vtable을 가리킴.
> 이걸 그냥 GPU로 복사하면 포인터가 여전히 host를 가리켜서 device에서 깨짐 → GPU 안에서 객체를 새로 생성해야.

---

## camera 클래스 + 안티에일리어싱

main에 흩어져 있던 뷰포트 계산/렌더 루프를 `camera`로 캡슐화. main은 world + 카메라 설정만 갖고 `cam.render(world)`.

**계단(jaggies) 원인**: 픽셀마다 정중앙으로 광선 하나만 쏨 → 그 픽셀은 "구" 아니면 "배경" 이분법. 중간이 없음.
경계 픽셀은 실제로 반은 구 반은 배경인데 흑백으로만 찍혀서 계단.

**해법**: 픽셀 하나에 광선 여러 개를 **랜덤 위치**로 쏘고 색을 **평균**. 경계 픽셀은
"광선 10개 중 4개는 구, 6개는 배경" → 평균이 중간색 → 부드럽게 섞임. (몬테카를로: 랜덤 샘플 평균으로 참값 근사.)

```cpp
color pixel_color(0, 0, 0);                    // 0에서 시작
for (sample) {
    ray r = get_ray(i, j);                     // 픽셀 안 랜덤 지점
    pixel_color += ray_color(r, world);        // 누적
}
write_color(out, pixel_samples_scale * pixel_color);  // 1/samples 곱해 평균
```

- `get_ray`가 `sample_square()`로 `[-0.5, 0.5)` 오프셋 줘서 `(i + offset)`이 픽셀 영역 안 랜덤 지점.
- samples 늘리면(50, 100) 더 매끈해지지만 그만큼 느려짐. 10이면 충분.

---

## rtweekend.h — 공용 헤더

`infinity`, `random_double`, `pi` 등 여기저기서 쓰는 유틸을 한 곳에 모음.
**`rtweekend.h`가 `vec3.h`/`ray.h`를 include하고, 나머지 헤더(camera, sphere 등)는 `rtweekend.h`를 include**하는 구조.
그러면 공용 상수/함수가 어디서든 보임.

```cpp
const double infinity = std::numeric_limits<double>::infinity();
inline double random_double() { return std::rand() / (RAND_MAX + 1.0); }  // [0,1)
```

---

## diffuse — 여기가 목표였던 "레이트레이서다운 이미지"

**diffuse = 거친 무광 표면**(벽, 종이, 흙). 매끈한 거울이 한 방향 반사라면, diffuse는 미세 요철 때문에 빛이 사방으로 흩어짐.
그걸 "랜덤 방향 반사"로 근사.

핵심: 광선이 표면 닿으면 **랜덤 방향으로 새 광선**을 만들어 튕김 → 그게 또 어딘가 닿아 또 튕김 → ...
→ 하늘(배경)에 닿으면 하늘색을 얻고, **튕길 때마다 색이 감쇠(×0.5)**되며 되돌아옴.
→ 이걸 **재귀**로 구현. `ray_color`가 자기를 부름.

```cpp
color ray_color(const ray& r, int depth, const hittable& world) const {
    if (depth <= 0) return color(0,0,0);                 // 너무 많이 튕김 → 빛 소진

    hit_record rec;
    if (world.hit(r, 0.001, infinity, rec)) {            // t_min = 0.001 주의!
        vec3 direction = rec.normal + random_unit_vector();  // 람베르시안
        return 0.5 * ray_color(ray(rec.p, direction), depth-1, world);  // 재귀, ×0.5 감쇠
    }
    // 하늘색 (기존 배경 lerp)
    ...
}
```

- `0.5 *` = 반사율(albedo). 받은 빛의 50%만 반사. 나중에 재질마다 이게 색이 됨(빨간 공은 빨강만 반사).
- 두 번 튕기면 `0.5×0.5 = 0.25`. 빛이 여러 번 갇히는 구석(두 구 접점, 오목한 데)이 더 어두워짐 = **부드러운 그림자/음영**. 이게 diffuse의 입체감.
- **depth 제한**(`max_depth = 50`): 하늘에 영영 안 닿으면 재귀가 무한히 돌아 스택 오버플로. depth를 재귀마다 -1, 0이면 검정 반환하고 멈춤.

### shadow acne — t_min = 0.001 (절대 빼먹지 말 것)
튕긴 새 광선의 시작점 `rec.p`가 표면에 딱 붙어 있어서, 부동소수점 오차로 그 광선이
**자기가 방금 출발한 표면을 t≈0에서 다시 맞았다**고 착각함 → 표면을 못 벗어나고 그 자리서 계속 튕겨 → 검은 얼룩 범벅.
`t_min = 0.001`로 "출발점 바로 근처(t < 0.001) 교차 무시" → 자기 표면 오인 제거.

> **CUDA 북마크**: 이 재귀 `ray_color`가 GPU에선 **반복문 + 깊이 제한**으로 바뀜(깊은 재귀가 GPU에 부담).
> diffuse의 랜덤성 + 재귀가 포팅 때 제일 손볼 곳.

---

## random_unit_vector — 왜 구 안만 채택하나 (rejection method)

diffuse 튕김은 **모든 방향이 공평한** 랜덤이어야 함. 특정 방향이 더 자주 나오면 음영이 부자연스러움.

```cpp
inline vec3 random_unit_vector() {
    while (true) {
        auto p = random_vec(-1, 1);            // [-1,1]³ 정육면체 안 랜덤 점
        auto lensq = p.length_squared();
        if (1e-160 < lensq && lensq <= 1.0)     // 단위 구 안이면 채택
            return p / sqrt(lensq);             // 표면으로 정규화
    }
}
```

**왜 정육면체를 그냥 안 쓰고 구 밖을 버리나:**
정육면체는 축 방향보다 **모서리(대각선) 방향이 더 멀다**(3D면 √3 ≈ 1.73 vs 1). 그래서 균일하게 점을 뿌리면
모서리 쪽에 점이 더 몰림 → 그 방향이 과대표집됨(편향). 구는 모든 방향이 등거리(반지름)라 공평.
→ 정육면체에서 뽑고(쉬움) → **구 밖은 버리고**(모서리 편향 제거) → 정규화. 다트판에 원 그려놓고 원 밖은 무효 처리하는 느낌.
`1e-160` 체크는 0에 너무 가까운 벡터 정규화(0/0) 방지 안전장치.

**중요한 구분**: 여기 "구"는 우리가 렌더링하는 sphere가 아니라, **방향을 뽑기 위한 수학적 단위 구**(반지름 1).
화면에 안 그려짐. 나침반 같은 도구. 실제 튕김은 교차점 `rec.p`에서 일어나고, 단위 구는 방향 계산에만 씀.

**람베르시안**: `rec.normal + random_unit_vector()`. 법선(바깥 방향)에 랜덤 벡터를 더하면
법선 쪽으로 치우친 랜덤 방향이 됨. 이 분포가 실제 diffuse 물리(코사인 법칙)랑 맞아서 자연스러운 음영.

---

## 감마 보정 — 결과가 어두운 이유

diffuse 돌리면 이미지가 예상보다 어두운데 버그 아님.
우리가 계산한 색은 **선형**(0.5 = 빛의 양 절반). 근데 모니터는 입력을 **비선형(대략 제곱)**으로 어둡게 출력함.
그래서 0.5를 넣어도 화면엔 훨씬 어둡게 나옴.

→ 미리 **sqrt**를 취해 밝혀서 보내면 모니터의 제곱과 상쇄됨.

```cpp
inline double linear_to_gamma(double x) {
    if (x > 0) return std::sqrt(x);
    return 0;
}
// write_color에서 r, g, b 각각 linear_to_gamma 통과시킨 뒤 0~255 변환
```

이거 넣으면 회색 공이 눈에 띄게 밝아지고 음영이 자연스러워짐. → 목표 이미지 완성.

---

## 지금까지 만든 파일

```
rtweekend.h        # 공용 상수/유틸 (infinity, random_double)
vec3.h             # 벡터/색 + random_unit_vector
ray.h
color.h            # write_color + 감마 보정
hittable.h         # hit_record + hittable 추상
sphere.h           # hittable 상속, hit 구현
hittable_list.h    # 다중 객체 순회 + 최근접
camera.h           # 뷰포트/렌더 루프 + 안티에일리어싱 + 재귀 ray_color
main.cpp           # 장면 구성 + cam.render
```

## 다음 (아직 안 함)
- 색/재질: `material` 추상 + `lambertian`(albedo를 색으로) → 컬러 공
- `metal`(반사), `dielectric`(굴절/유리)
- 이동 카메라, 디포커스 블러
- CUDA 포팅: 재귀→반복, device 객체 생성, shared_ptr→raw 포인터
