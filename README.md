# 대미지 계산식

## 0. 전체 요약

전체 계산 구조를 간략화하면 다음과 같다.

**대미지 ≈ floor( D × [(100 - Guard) / 100] × [1 + Dom / 100] )**

정상 대미지 영역에서는:

**D ≈ floor( [기본 공격값 × 방어/저항·관통 계수 + FD + ADD - F] × [K / 100] × [Q / 100] )**

* `기본 공격값` = 공격 타입 및 직타/소환에 따라 결정
* `방어/저항·관통 계수` = 물리는 방어력, 마법은 저항력 기준
* `FD` = 고정 대미지 증가
* `ADD` = 몬스터 추가 대미지
* `F` = 피해 감소
* `K` = 크리티컬 및 조건부 대미지 계수
* `Q` = 직타는 `Q_d`, 소환은 `Q_s`
* `Guard` = 대미지 감소(%)
* `Dom` = 일반/보스 몬스터 지배력

직타와 소환, 비크리와 크리의 차이는 주로 `기본 공격값`, `방어/저항·관통 계수`, `K`, `Q`의 계산 방식에서 발생한다.

실제 계산에서는 각 단계별 `floor`와 `f32` 변환이 존재하므로, 위 식은 전체 계산 구조를 보여주기 위한 요약식이다.<br>
정확한 계산 순서는 아래 세부 공식을 따른다.

또한 `T_raw <= 0`인 경우에는 계산된 `T_raw` 대신 `R_min ∈ {1, 2}`를 사용한 뒤 대미지 감소(%)와 몬스터 지배력을 적용한다.

**미검증된 항목**

현재 다음 항목은 계산 모델에는 반영되어 있으나 직접적인 검증이 완료되지 않았다.

* 물리 직타 무기공격력 난수 `W_d`가 정확한 균등분포인지 여부
* 최소\~최대 공격력 `W_d`와 최소\~최대 대미지 `Q_d`가 서로 독립적인 난수인지 여부
* 소환 공격력 계수 `SC = 42`가 모든 소환에 공통으로 적용되는 고정값인지 여부

## 1. 약자

`STR` = 근력<br>
`STR_eff` = EFF 적용 후 실효 근력

`MAG` = 마법력<br>
`MAG_eff` = EFF 적용 후 실효 마법력

`EFF` = 주 능력치 효율

* 물리: 근력 효율
* 마법: 마법력 효율

`WMIN` = 최소 무기공격력<br>
`WMAX` = 최대 무기공격력<br>
`W_d` = 물리 직타 무기공격력 난수<br>
`W_s` = 물리 소환 무기공격력 값

`ELE` = 속성력<br>
`FD` = 해당 공격 타입의 고정 대미지 증가<br>
`F` = 해당 공격 타입의 피해 감소

`ADD` = 몬스터 추가 대미지

* 일반 몬스터: 일반 몬스터 추가 대미지
* 보스 몬스터: 보스 몬스터 추가 대미지

`L` = 공격자 레벨<br>
`ARMOR` = 물리 방어력<br>
`RES` = 마법 저항력<br>
`PEN` = 해당 공격 타입의 관통력<br>
`P_s` = 소환 실효 관통력<br>
`Guard` = 대미지 감소(%)<br>
`ELASTICITY` = 크리티컬 저항

`C` = 직타 스킬 계수<br>
`S` = 소환 능력치 적용 배율(%)<br>
`SC` = 소환 공격력 계수 (42 고정값으로 추정됨)

`BD` = 해당 공격 타입의 백어택 대미지<br>
`MD` = 근거리 대미지<br>
`SD` = 상태이상 대미지

`rawMIN` = 해당 공격 타입의 최소 대미지 고정값(+)<br>
`FMIN` = 해당 공격 타입의 최종 최소 대미지<br>
`MIN` = FMIN 적용 후 최소 대미지

`rawMAX` = 해당 공격 타입의 최대 대미지 고정값(+)<br>
`FMAX` = 해당 공격 타입의 최종 최대 대미지<br>
`MAX` = FMAX 적용 후 최대 대미지

`rawCD` = 해당 공격 타입의 크리티컬 대미지 고정값(+)<br>
`FCD` = 해당 공격 타입의 최종 크리티컬 대미지<br>
`CD` = FCD 적용 후 크리티컬 대미지

`Dom` = 일반/보스 몬스터 지배력

`Q_d` = 직타 최소\~최대 대미지 난수 계수<br>
`Q_s` = 소환 최소\~최대 대미지 난수 계수

`COND_d` = 직타에 실제 적용되는 조건부 대미지 합<br>
`BACK_s` = 소환 백어택 대미지

`R_min` = 방어 초과 시 대미지 출력 내부값 `{1, 2}`

`f32(x)` = IEEE 754 float32 변환<br>
`floor(x)` = 소수점 이하 버림

# 2. 기본 스탯 및 난수 계산

## 실효 근력

```text
STR_eff =
floor(
    STR * (1 + EFF / 100)
)
```

물리 직타에는 `STR_eff`를 사용한다.

물리 소환에는 `STR_eff`가 아닌 `STR`를 직접 사용한다.

## 실효 마법력

```text
MAG_eff =
floor(
    MAG * (1 + EFF / 100)
)
```

마법 직타에는 `MAG_eff`를 사용한다.

마법 소환에는 `MAG_eff`가 아닌 `MAG`를 직접 사용한다.

## 최소 대미지

```text
MIN =
floor(
    rawMIN * (100 + FMIN) / 100
)
```

## 최대 대미지

```text
MAX =
floor(
    rawMAX * (100 + FMAX) / 100
)
```

## 크리티컬 대미지

```text
CD =
floor(
    rawCD * (100 + FCD) / 100
)
```

## 물리 직타 무기공격력 난수

```text
W_d,min =
min(
    WMIN,
    WMAX
)

W_d,max =
WMAX

W_d =
W_d,min ~ W_d,max 사이의 고해상도 균등 난수
```

일반적으로 `WMIN <= WMAX`이면 `WMIN ~ WMAX` 사이에서 결정된다.

`WMIN > WMAX`이면 하한도 `WMAX`로 고정되어 무기공격력 난수 폭이 사라진다.

## 물리 소환 무기공격력

```text
W_s,min =
min(
    WMIN,
    WMAX
)

W_s,max =
WMAX

W_s =
W_s,min ~ W_s,max 사이의 무기공격력 값
```

일반적으로 `WMIN <= WMAX`이면 `WMIN ~ WMAX` 범위를 사용한다.

`WMIN > WMAX`이면 하한도 `WMAX`로 고정되어 무기공격력 범위가 사라진다.

## 직타 난수

```text
Q_d,min =
min(
    MIN,
    MAX
)

Q_d,max =
MAX

Q_d =
Q_d,min ~ Q_d,max 사이의 고해상도 균등 난수
```

일반적으로 `MIN <= MAX`이면 `MIN ~ MAX` 사이에서 결정된다.

`MIN > MAX`이면 하한도 `MAX`로 고정되어 난수 폭이 사라진다.

물리 직타는 `W_d`와 `Q_d`가 각각 독립적인 난수로 적용된다.

## 소환 난수

```text
Q_s,min_raw =
floor(
    MIN * S / 100
)
+ 95

Q_s,max =
floor(
    MAX * S / 100
)
+ 105

Q_s,min =
min(
    Q_s,min_raw,
    Q_s,max
)

Q_s =
Q_s,min ~ Q_s,max 사이의 고해상도 균등 난수
```

`Q_s,min_raw > Q_s,max`이면 하한도 `Q_s,max`로 고정된다.

## 직타 조건부 대미지

```text
COND_d =
(백어택이면 BD, 아니면 0)
+ (근거리이면 MD, 아니면 0)
+ (상태이상이면 SD, 아니면 0)
```

백어택, 근거리, 상태이상은 서로 곱하지 않고 같은 계수에 가산된다.

# 3. 직타 비크리 — 물리

방어 상수(레벨):

```text
a =
30 * L + 200
```

기본 공격값:

```text
B_0 =
STR_eff
+ floor(
    W_d * (C + 100) / 50
)
```

방어력 및 관통력 계수:

```text
m_ARMOR =
f32(
    1
    - ARMOR / (ARMOR + a)
      * (100 - PEN) / 100
)
```

즉 물리 방어력의 분모는:

```text
ARMOR + 30 * L + 200
```

방어력 적용:

```text
B_1 =
f32(
    f32(B_0) * m_ARMOR
)
```

물리 고정 대미지 및 추가 대미지 적용:

```text
B_2 =
f32(
    B_1 + FD + ADD
)
```

피해 감소 적용:

```text
U_d =
f32(
    B_2 - F
)
```

조건부 계수:

```text
K_d =
100 + COND_d
```

조건부 대미지 적용:

```text
U_d_cond =
f32(
    U_d * f32(K_d / 100)
)
```

MIN\~MAX 난수 적용:

```text
T_raw =
floor(
    f32(
        U_d_cond * Q_d
    ) / 100
)
```

정상 영역:

```text
T_raw > 0 이면

D_direct_noncrit =
T_raw
```

방어가 공격을 완전히 상쇄한 영역:

```text
T_raw <= 0 이면

R_min ∈ {1, 2}

D_direct_noncrit =
R_min
```

# 4. 직타 크리 — 물리

방어 상수(레벨):

```text
a =
30 * L + 200
```

기본 공격값:

```text
B_0 =
STR_eff
+ floor(
    W_d * (C + 100) / 50
)
```

방어력 및 관통력 계수:

```text
m_ARMOR =
f32(
    1
    - ARMOR / (ARMOR + a)
      * (100 - PEN) / 100
)
```

즉 물리 방어력의 분모는:

```text
ARMOR + 30 * L + 200
```

방어력 적용:

```text
B_1 =
f32(
    f32(B_0) * m_ARMOR
)
```

물리 고정 대미지 및 추가 대미지 적용:

```text
B_2 =
f32(
    B_1 + FD + ADD
)
```

피해 감소 적용:

```text
U_d =
f32(
    B_2 - F
)
```

크리티컬 저항 적용 크리티컬 대미지:

```text
CD_e =
floor(
    CD * (1000 - ELASTICITY) / 1000
)
```

크리티컬 및 조건부 계수:

```text
K_d =
100
+ CD_e
+ COND_d
```

즉:

```text
K_d =
100
+ floor(
    CD * (1000 - ELASTICITY) / 1000
)
+ (백어택이면 BD, 아니면 0)
+ (근거리이면 MD, 아니면 0)
+ (상태이상이면 SD, 아니면 0)
```

백어택, 근거리, 상태이상 대미지에는 크리티컬 저항이 적용되지 않는다.

크리티컬 및 조건부 대미지 적용:

```text
U_d_crit =
f32(
    U_d * f32(K_d / 100)
)
```

MIN\~MAX 난수 적용:

```text
T_raw =
floor(
    f32(
        U_d_crit * Q_d
    ) / 100
)
```

정상 영역:

```text
T_raw > 0 이면

D_direct_crit =
T_raw
```

방어가 공격을 완전히 상쇄한 영역:

```text
T_raw <= 0 이면

R_min ∈ {1, 2}

D_direct_crit =
R_min
```

# 5. 소환 비크리 — 물리

방어 상수(레벨):

```text
a =
30 * L + 200
```

기본 공격값:

```text
B_0,s =
floor(
    STR * S / 100
)
+ (W_s + 1) * SC
+ 1
```

소환 실효 물리 관통력:

```text
P_s =
99
```

현재 검증된 범위에서는 표시 물리 관통력과 관계없이 실효 99%로 계산한다.

방어력 및 관통력 계수:

```text
m_ARMOR,s =
f32(
    1
    - ARMOR / (ARMOR + a)
      * (100 - P_s) / 100
)
```

즉 현재 운용식에서는:

```text
m_ARMOR,s =
f32(
    1
    - ARMOR / (ARMOR + a)
      * 0.01
)
```

즉 물리 방어력의 분모는:

```text
ARMOR + 30 * L + 200
```

방어력 적용:

```text
B_1,s =
f32(
    f32(B_0,s) * m_ARMOR,s
)
```

물리 고정 대미지 및 추가 대미지 적용:

```text
B_2,s =
f32(
    B_1,s + FD + ADD
)
```

피해 감소 적용:

```text
U_s =
f32(
    B_2,s - F
)
```

소환은 근거리 대미지와 상태이상 대미지가 적용되지 않는다.

캐릭터의 물리 백어택 대미지 `BD`도 사용하지 않으며, 백어택일 때 고정 20이 적용된다.

```text
BACK_s =
백어택이면 20
아니면 0
```

비크리 및 백어택 계수:

```text
K_s =
100 + BACK_s
```

백어택 계수 적용:

```text
U_s_cond =
f32(
    U_s * f32(K_s / 100)
)
```

소환 난수 적용:

```text
T_raw =
floor(
    f32(
        U_s_cond * Q_s
    ) / 100
)
```

정상 영역:

```text
T_raw > 0 이면

D_summon_noncrit =
T_raw
```

방어가 공격을 완전히 상쇄한 영역:

```text
T_raw <= 0 이면

R_min ∈ {1, 2}

D_summon_noncrit =
R_min
```

# 6. 소환 크리 — 물리

방어 상수(레벨):

```text
a =
30 * L + 200
```

기본 공격값:

```text
B_0,s =
floor(
    STR * S / 100
)
+ (W_s + 1) * SC
+ 1
```

소환 실효 물리 관통력:

```text
P_s =
99
```

방어력 및 관통력 계수:

```text
m_ARMOR,s =
f32(
    1
    - ARMOR / (ARMOR + a)
      * (100 - P_s) / 100
)
```

즉 현재 운용식에서는:

```text
m_ARMOR,s =
f32(
    1
    - ARMOR / (ARMOR + a)
      * 0.01
)
```

즉 물리 방어력의 분모는:

```text
ARMOR + 30 * L + 200
```

방어력 적용:

```text
B_1,s =
f32(
    f32(B_0,s) * m_ARMOR,s
)
```

물리 고정 대미지 및 추가 대미지 적용:

```text
B_2,s =
f32(
    B_1,s + FD + ADD
)
```

피해 감소 적용:

```text
U_s =
f32(
    B_2,s - F
)
```

소환용 크리티컬 대미지 변환:

```text
R_s =
floor(
    (rawCD - 50) * S / 100
)
+ 50
```

```text
CD_s =
floor(
    R_s * (100 + FCD) / 100
)
```

크리티컬 저항 적용:

```text
CD_s,e =
floor(
    CD_s * (1000 - ELASTICITY) / 1000
)
```

소환은 근거리 대미지와 상태이상 대미지가 적용되지 않는다.

백어택은 캐릭터의 `BD` 대신 고정 20을 사용한다.

```text
BACK_s =
백어택이면 20
아니면 0
```

크리티컬 및 백어택 계수:

```text
K_s =
100
+ CD_s,e
+ BACK_s
```

즉:

```text
K_s =
100
+ floor(
    CD_s * (1000 - ELASTICITY) / 1000
)
+ (백어택이면 20, 아니면 0)
```

백어택 +20에는 크리티컬 저항이 적용되지 않는다.

크리티컬 및 백어택 계수 적용:

```text
U_s_crit =
f32(
    U_s * f32(K_s / 100)
)
```

소환 난수 적용:

```text
T_raw =
floor(
    f32(
        U_s_crit * Q_s
    ) / 100
)
```

정상 영역:

```text
T_raw > 0 이면

D_summon_crit =
T_raw
```

방어가 공격을 완전히 상쇄한 영역:

```text
T_raw <= 0 이면

R_min ∈ {1, 2}

D_summon_crit =
R_min
```

# 7. 직타 비크리 — 마법

저항 상수(레벨):

```text
a =
20 * L + 200
```

기본 공격값:

```text
B_0 =
MAG_eff
+ floor(
    ELE * (C + 100) / 50
)
```

저항력 및 관통력 계수:

```text
m_RES =
f32(
    1
    - RES / (RES + a)
      * (100 - PEN) / 100
)
```

즉 마법 저항력의 분모는:

```text
RES + 20 * L + 200
```

저항력 적용:

```text
B_1 =
f32(
    f32(B_0) * m_RES
)
```

마법 고정 대미지 및 추가 대미지 적용:

```text
B_2 =
f32(
    B_1 + FD + ADD
)
```

피해 감소 적용:

```text
U_d =
f32(
    B_2 - F
)
```

조건부 계수:

```text
K_d =
100 + COND_d
```

조건부 대미지 적용:

```text
U_d_cond =
f32(
    U_d * f32(K_d / 100)
)
```

MIN\~MAX 난수 적용:

```text
T_raw =
floor(
    f32(
        U_d_cond * Q_d
    ) / 100
)
```

정상 영역:

```text
T_raw > 0 이면

D_direct_noncrit =
T_raw
```

방어가 공격을 완전히 상쇄한 영역:

```text
T_raw <= 0 이면

R_min ∈ {1, 2}

D_direct_noncrit =
R_min
```

# 8. 직타 크리 — 마법

저항 상수(레벨):

```text
a =
20 * L + 200
```

기본 공격값:

```text
B_0 =
MAG_eff
+ floor(
    ELE * (C + 100) / 50
)
```

저항력 및 관통력 계수:

```text
m_RES =
f32(
    1
    - RES / (RES + a)
      * (100 - PEN) / 100
)
```

즉 마법 저항력의 분모는:

```text
RES + 20 * L + 200
```

저항력 적용:

```text
B_1 =
f32(
    f32(B_0) * m_RES
)
```

마법 고정 대미지 및 추가 대미지 적용:

```text
B_2 =
f32(
    B_1 + FD + ADD
)
```

피해 감소 적용:

```text
U_d =
f32(
    B_2 - F
)
```

크리티컬 저항 적용 크리티컬 대미지:

```text
CD_e =
floor(
    CD * (1000 - ELASTICITY) / 1000
)
```

크리티컬 및 조건부 계수:

```text
K_d =
100
+ CD_e
+ COND_d
```

즉:

```text
K_d =
100
+ floor(
    CD * (1000 - ELASTICITY) / 1000
)
+ (백어택이면 BD, 아니면 0)
+ (근거리이면 MD, 아니면 0)
+ (상태이상이면 SD, 아니면 0)
```

백어택, 근거리, 상태이상 대미지에는 크리티컬 저항이 적용되지 않는다.

크리티컬 및 조건부 대미지 적용:

```text
U_d_crit =
f32(
    U_d * f32(K_d / 100)
)
```

MIN\~MAX 난수 적용:

```text
T_raw =
floor(
    f32(
        U_d_crit * Q_d
    ) / 100
)
```

정상 영역:

```text
T_raw > 0 이면

D_direct_crit =
T_raw
```

방어가 공격을 완전히 상쇄한 영역:

```text
T_raw <= 0 이면

R_min ∈ {1, 2}

D_direct_crit =
R_min
```

# 9. 소환 비크리 — 마법

저항 상수(레벨):

```text
a =
20 * L + 200
```

기본 공격값:

```text
B_0,s =
floor(
    MAG * S / 100
)
+ (ELE + 1) * SC
+ 1
```

소환 실효 마법 관통력:

```text
P_s =
99
```

현재 검증된 범위에서는 표시 마법 관통력과 관계없이 실효 99%로 계산한다.

저항력 및 관통력 계수:

```text
m_RES,s =
f32(
    1
    - RES / (RES + a)
      * (100 - P_s) / 100
)
```

즉 현재 운용식에서는:

```text
m_RES,s =
f32(
    1
    - RES / (RES + a)
      * 0.01
)
```

즉 마법 저항력의 분모는:

```text
RES + 20 * L + 200
```

저항력 적용:

```text
B_1,s =
f32(
    f32(B_0,s) * m_RES,s
)
```

마법 고정 대미지 및 추가 대미지 적용:

```text
B_2,s =
f32(
    B_1,s + FD + ADD
)
```

피해 감소 적용:

```text
U_s =
f32(
    B_2,s - F
)
```

소환은 근거리 대미지와 상태이상 대미지가 적용되지 않는다.

캐릭터의 마법 백어택 대미지 `BD`도 사용하지 않으며, 백어택일 때 고정 20이 적용된다.

```text
BACK_s =
백어택이면 20
아니면 0
```

비크리 및 백어택 계수:

```text
K_s =
100 + BACK_s
```

백어택 계수 적용:

```text
U_s_cond =
f32(
    U_s * f32(K_s / 100)
)
```

소환 난수 적용:

```text
T_raw =
floor(
    f32(
        U_s_cond * Q_s
    ) / 100
)
```

정상 영역:

```text
T_raw > 0 이면

D_summon_noncrit =
T_raw
```

방어가 공격을 완전히 상쇄한 영역:

```text
T_raw <= 0 이면

R_min ∈ {1, 2}

D_summon_noncrit =
R_min
```

# 10. 소환 크리 — 마법

저항 상수(레벨):

```text
a =
20 * L + 200
```

기본 공격값:

```text
B_0,s =
floor(
    MAG * S / 100
)
+ (ELE + 1) * SC
+ 1
```

소환 실효 마법 관통력:

```text
P_s =
99
```

저항력 및 관통력 계수:

```text
m_RES,s =
f32(
    1
    - RES / (RES + a)
      * (100 - P_s) / 100
)
```

즉 현재 운용식에서는:

```text
m_RES,s =
f32(
    1
    - RES / (RES + a)
      * 0.01
)
```

즉 마법 저항력의 분모는:

```text
RES + 20 * L + 200
```

저항력 적용:

```text
B_1,s =
f32(
    f32(B_0,s) * m_RES,s
)
```

마법 고정 대미지 및 추가 대미지 적용:

```text
B_2,s =
f32(
    B_1,s + FD + ADD
)
```

피해 감소 적용:

```text
U_s =
f32(
    B_2,s - F
)
```

소환용 크리티컬 대미지 변환:

```text
R_s =
floor(
    (rawCD - 50) * S / 100
)
+ 50
```

```text
CD_s =
floor(
    R_s * (100 + FCD) / 100
)
```

크리티컬 저항 적용:

```text
CD_s,e =
floor(
    CD_s * (1000 - ELASTICITY) / 1000
)
```

소환은 근거리 대미지와 상태이상 대미지가 적용되지 않는다.

백어택은 캐릭터의 `BD` 대신 고정 20을 사용한다.

```text
BACK_s =
백어택이면 20
아니면 0
```

크리티컬 및 백어택 계수:

```text
K_s =
100
+ CD_s,e
+ BACK_s
```

즉:

```text
K_s =
100
+ floor(
    CD_s * (1000 - ELASTICITY) / 1000
)
+ (백어택이면 20, 아니면 0)
```

백어택 +20에는 크리티컬 저항이 적용되지 않는다.

크리티컬 및 백어택 계수 적용:

```text
U_s_crit =
f32(
    U_s * f32(K_s / 100)
)
```

소환 난수 적용:

```text
T_raw =
floor(
    f32(
        U_s_crit * Q_s
    ) / 100
)
```

정상 영역:

```text
T_raw > 0 이면

D_summon_crit =
T_raw
```

방어가 공격을 완전히 상쇄한 영역:

```text
T_raw <= 0 이면

R_min ∈ {1, 2}

D_summon_crit =
R_min
```

# 11. 공통 대미지 감소 및 몬스터 지배력 적용

일반 몬스터는 일반 몬스터 지배력,<br>
보스 몬스터는 보스 몬스터 지배력을 사용한다.

## 직타

직타는 방어 초과 여부 판정 후 대미지 감소(%)와 지배력을 적용한다.

지배력 계수:

```text
g =
f32(
    1 + f32(Dom / 100)
)
```

최종 대미지:

```text
damage =
floor(
    D
    * (100 - Guard) / 100
    * g
)
```

여기서 `D`는 다음 중 하나다.

```text
직타 비크리: D_direct_noncrit
직타 크리:   D_direct_crit
```

## 소환

소환도 방어 초과 여부 판정 후 대미지 감소(%)와 지배력을 적용한다.

지배력 계수:

```text
g =
f32(
    1 + f32(Dom / 100)
)
```

최종 대미지:

```text
damage =
floor(
    D
    * (100 - Guard) / 100
    * g
)
```

여기서 `D`는 다음 중 하나다.

```text
소환 비크리: D_summon_noncrit
소환 크리:   D_summon_crit
```
