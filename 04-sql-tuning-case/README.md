## 04. SQL 튜닝 사례 분석

### 목표
이전 프로젝트에서 경험한 SQL 성능 개선 사례를 구조적으로 분석하고 기록으로 남기기 위해 정리했습니다.

### 1. 배경
|구분|내용|
|---|---|
|프로젝트|2024년 하반기 개인정보 포털 서비스 개선 용역 (에스온시스템)|
|환경|Java, JSP, jQuery, Cubrid|
|증상|운영 중 특정 조회 화면에서, 조회 버튼을 누르면 결과가 나오기까지 약 23초가 걸림|
|작업방식|선임과 함께 쿼리를 분석하고 구조를 수정|

화면 기능 자체는 정상 동작하였지만</br>사용자에게 23초는 사실상 응답이 없는 것과 같은 시간이었기 때문에 튜닝이 필수인 상황이었습니다.
> 이 문서에서 다루는 쿼리 튜닝은 선임과 함께 진행했습니다.</br>
> 그 중 직접 분석하고 수정한 것은</br>
> **(1) 필터링 조건의 위치 이동**</br>
> **(2) 상관 서브쿼리의 반복 실행 문제**</br>
> 두 가지 이며, 이 문서는 그 두 가지를 중심으로 정리합니다.</br>
> 인라인뷰 전체 재구성, `UNION` 제거 등 그 외 구조 변경은 선임이 담당하였습니다.

### 2. 원인 분석
#### 2.1. 늦은 필터링 시점: 필터링 조건(`group_no`)이 이후 단계에서 적용
- 이전 쿼리의 가장 큰 문제점은 데이터를 필터링 하는 시점이었습니다.
- 안쪽 인라인뷰에서 큰 테이블(`plesysitslevlt`, `pleinstitslevlt`)을 읽어낼 때, 조건이 `evli_no` 하나 뿐이었습니다.</br>
→ 읽어야 할 데이터의 양이 처음부터 불필요하게 커졌고, 커진 데이터가 이후 연산을 모두 무겁게  만들었습니다.
```sql
-- before: 안쪽 인라인뷰가 group_no 없이 evli_no만으로 데이터를 읽음
FROM plesysitslevlt AS self_sys
LEFT JOIN plesysevlrslttt AS ver_sys ON ...
WHERE self_sys.evli_no = #quantitativeIndicatorNo#
 
UNION
 
FROM pleinstitslevlt AS self_inst
LEFT JOIN pleinstevlrslttt AS ver ON ...
WHERE self_inst.evli_no = #quantitativeIndicatorNo#
```

#### 2.2. 행마다 반복실행 된 상관 서브쿼리
> **상관 서브쿼리란?**</br>
> 바깥 쿼리가 읽는 행마다 그 행의 값을 받아 다시 실행되는 서브쿼리이다.</br>
> 바깥 행이 1만 건이라면  서브쿼리도 1만 번을 돌게된다.
- `before` 쿼리에는 바깥 쿼리의 행마다 다시 실행되는 상관 서브쿼리가 있었습니다.
```sql
-- before: target_inst의 행마다 반복 실행되는 상관 서브쿼리
, (
    SELECT COUNT(inst_system.sys_no)
    FROM pleprvcprcssyst AS inst_system
    WHERE target_inst.group_no = inst_system.group_no       -- 바깥 행 참조
      AND target_inst.inst_mng_cd = inst_system.inst_mng_cd  -- 바깥 행 참조
      AND target_inst.inst_cd = inst_system.inst_cd          -- 바깥 행 참조
) AS system_count
```
-  원인 2.1과 2.2는 서로 맞물려 있다고 볼 수 있습니다. </br>바깥 쿼리의 행 수가 많았고, 그에 따라 상관 서브쿼리의 반복 횟수도 늘어날 수 밖에 없었습니다.

### 3. 개선 방법
#### 3.1. 필터링 조건을 인라인뷰 내부로 이동
- 가장 바깥에서 적용되던 그룹 조건(`group_no`)을 데이터를 읽어들이는 인라인뷰 안 쪽으로 이동하였습니다.</br>

**1) 시스템 쪽 인라인뷰: `plesysitslevlt`를 읽는 시점**
```diff
  FROM plesysitslevlt a
  LEFT JOIN plesysevlrsltt b ON ...
  WHERE a.evli_no = #quantitativeIndicatorNo#
+   AND a.group_no = #companyGroupNo#
```

**2) 기관 쪽 인라인뷰: `pleinstitslevlt`를 읽는 시점**
```diff
  FROM pleevlidtlm a
  LEFT JOIN pleinstitslevlt b
    ON b.evli_no = a.evli_no
    AND b.evli_dtl_no = a.evli_dtl_no
+   AND b.group_no = #companyGroupNo#
  LEFT JOIN pleinstevlrsltt c ON ...
  WHERE a.evli_no = #quantitativeIndicatorNo#
```

**3) 가장 바깥 쿼리(`t1`):결과를 한 번 더 필터링**
```diff
  FROM pletrgtinstt t1
  LEFT JOIN ( ... ) t2 ON ...
  LEFT JOIN ( ... ) t3 ON ...
+ WHERE t1.group_no = #companyGroupNo#
```

→ `before`에서는 `plesysitslevlt`과 `pleinstitslevlt`를 읽을 때 `evli_no` 조건 하나로 전체 데이터를 모두 읽고 결과를 다 만든 후에서야 그룹 조건을 통해 필터링 되었습니다.</br>
→ `after`에서는 1), 2)처럼 **`plesysitslevlt`과 `pleinstitslevlt`를 읽는 첫 시점부터 `group_no`로 데이터를 한정**하고, 3)에서 한 번 더 필터링하는 구조로 변경했습니다.</br>
→ 결과적으로 그룹으로 한정된 작은 데이터 위에서 이후의 조인·집계가 수행되도록 만들었습니다.

#### 3.2. 상관 서브쿼리의 반복 횟수 줄이기
- 상관 서브쿼리 자체는 튜닝 이후의 쿼리(`targetCompanyListAfter`)에도 남아있습니다.</br>다만 3.1.의 개선방법으로 바깥 쿼리의 행 수가 이미 줄어들었기 때문에 같은 서브쿼리라도 실행되는 횟수는 이전보다 크게 감소하였음을 확인할 수 있었습니다.

| 구분         | before | after             |
|------------|---|-------------------|
| 바깥 쿼리의 범위  | 해당 지표의 전체 데이터 | 해당 지표 + 해당 그룹 데이터 |
| 바깥 쿼리의 행 수 | 많음 | 적음                |
|상관 서브쿼리 실행 횟수|바깥 행 수만큼 (많이 반복)| 바깥 행 수만큼 (적게 반복)  |
> 즉, 원인 2.1을 해소하니 원인 2.2도 상당부분 해소되는 구조였습니다.</br>
> 필터링 조건의 시점을 옮기는 과정에서 단순히 데이터 양만 줄이는 게 아니라, 해당 데이터에 연동된 반복 연산까지 같이 줄일 수 있다는 것을 확인하게 된 시점이었습니다.


### 4. 결과
| 구분       | 변경 전 | 변경 후 |
|----------|-----|---|
| 조회 응답 시간 | 23초 |3초|

### 5. 배운 점
- 같은 결과를 내는 쿼리라도 데이터에 **필터가 적용되는 시점에 따라 성능이 크게 달라진다**는 것을 알게되었습니다.</br>
→ 필터 적용 시점을 가능한 한 선행 단계(데이터를 읽는 시점)에 두어야 이후에 따라오는 연산이 가벼워 집니다.
- 상관서브쿼리는 바깥 쿼리의 행 수만큼 반복되어 실행됩니다.</br>
→ 바깥 쿼리의 행 수를 먼저 줄이면 서브쿼리 부담도 함께 줄어들게 되므로 서로 맞물려 있던 문제임을 알게되었습니다.
- **성능 문제는 한 가지 원인으로 끝나지 않는다**는 것을 배웠습니다.</br>
→ 이번 쿼리도 조건의 위치, 상관서브쿼리, 인라인뷰 구조 등 여러 요인이 겹쳐져 있었고 그래서 선임과 함께 각 부분을 나눠 개선을 진행하였습니다.
- 직접 다룬 부분(조건의 위치, 상관서브쿼리)은 원리를 배우고 수정까지했지만 인라인뷰 전체 재구성처럼 선임이 담당한 영역은 아직 깊이 다뤄보지 못했습니다.</br>쿼리가 내부적으로 어떤 경로로 실행되는지(실행계획)를 직접 읽고 분석하는 것은 앞으로 더 학습해야 할 부분입니다.