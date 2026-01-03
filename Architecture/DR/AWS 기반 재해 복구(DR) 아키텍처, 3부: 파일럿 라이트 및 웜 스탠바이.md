# AWS DR 전략: 파일럿 라이트 vs 웜 스탠바이

데이터는 실시간으로 복제하되, 비용과 복구 속도 사이의 균형을 맞추는 두 가지 핵심 전략을 비교하고 구현 방법을 정리합니다.

---

## 1. 전략 비교: 핵심 차이점

| 구분 | 파일럿 라이트 (Pilot Light) | 웜 스탠바이 (Warm Standby) |
| --- | --- | --- |
| **비유** | 가스레인지의 **'불씨'**만 켜둠 | 자동차 엔진을 **'공회전(Idling)'** 시킴 |
| **핵심** | DB만 가동, 서버는 설정(AMI)만 유지 | 최소한의 서버 사양으로 항시 가동 |
| **RTO** | 수십 분 (서버 프로비저닝 시간 필요) | 수 분 (기존 서버 사양/대수 확장) |
| **비용** | **낮음** (서버 비용 거의 없음) | **중간** (항시 가동 비용 발생) |

---

## 2. 세부 전략 분석

### 2.1. 파일럿 라이트 (Pilot Light)

데이터는 준비되어 있지만 애플리케이션 계층은 비활성화된 상태입니다.

* **평상시:** DB만 복구 리전에서 동작하며, 서버는 AMI 형태로 저장만 해둡니다.
* **재해 시:** 저장된 AMI를 기반으로 EC2 인스턴스를 즉시 생성하고 트래픽을 수용합니다.
* **장점:** 백업 방식보다 빠르면서 서버 유지 비용을 획기적으로 절약할 수 있습니다.

### 2.2. 웜 스탠바이 (Warm Standby)

복구 리전에 실제 서비스가 가능한 최소한의 인프라가 이미 가동 중인 상태입니다.

* **평상시:** 서버 1~2대를 아주 작은 사양(Small Instance)으로 켜둡니다. 트래픽은 처리하지 않지만 모든 설정이 완료되어 있습니다.
* **재해 시:** 켜져 있는 서버의 대수를 늘리거나(Scale-out), 사양을 높여(Scale-up) 즉시 메인 리전의 역할을 물려받습니다.
* **장점:** 장애 조치(Failover)가 매우 빠르며, 평소에도 복구 리전의 작동 여부를 테스트하기 용이합니다.

---

## 3. 복구 프로세스 구현 (Detection & Restore)

### 3.1. 감지 (Detect) - 무엇을 지표로 삼을 것인가?

단순히 서버가 켜져 있는지(Ping) 확인하는 것만으로는 부족합니다.

* **API 지표:** 오류율 및 응답 시간을 모니터링합니다.
* **Synthetics:** `CloudWatch Synthetics`를 사용해 사용자 시나리오대로 스크립트를 실행하여 통계적 정확성을 확보합니다.
* **KPI 기반:** 전자 상거래의 경우 '주문 속도' 저하를 `Anomaly Detection`으로 감지하여 실제 비즈니스 임팩트를 파악합니다.

### 3.2. 복원 (Restore) - IaC를 활용한 스케일링

`CloudFormation`의 조건부 로직을 사용하면 단일 템플릿으로 운영/복구 리전을 모두 관리할 수 있습니다.

**CloudFormation 활용 예시 (Active/Passive 전환):**

```yaml
Parameters:
  ActiveOrPassive:
    Default: 'passive'
    Type: String
    AllowedValues: ['active', 'passive']

Conditions:
  IsActive: !Equals [!Ref "ActiveOrPassive", "active"]

Resources:
  WebAppAutoScalingGroup:
    Type: 'AWS::AutoScaling::AutoScalingGroup'
    Properties:
      # Active일 때는 설정한 대수만큼, Passive일 때는 0대로 유지
      MinSize: !If [IsActive, 3, 0] 
      MaxSize: 6
      DesiredCapacity: !If [IsActive, 3, 0]

```

**AWS CLI를 이용한 즉각적인 복구 실행:**
기존 스택의 파라미터만 `active`로 변경하여 서버를 즉시 프로비저닝합니다.

```bash
aws cloudformation update-stack \
    --stack-name SampleWebApp --use-previous-template \
    --parameters ParameterKey=ActiveOrPassive,ParameterValue=active

```

---

## 4. 장애 조치 (Failover) 전략

운영 트래픽을 복구 리전으로 전환하는 마지막 단계입니다.

### 4.1. Route 53 장애 조치 라우팅

* **자동 장애 조치:** 상태 검사(Health Check) 결과에 따라 트래픽을 자동 전환합니다.
* **주의사항:** 잘못된 경보로 인한 불필요한 전환을 막기 위해 **Route 53 Application Recovery Controller**를 사용하여 사람이 최종 승인(수동 개입)하는 구조가 권장됩니다.

### 4.2. AWS Global Accelerator

* 200개 이상의 엣지 로케이션을 통해 가장 가까운 엔드포인트로 연결합니다.
* **트래픽 다이얼(Traffic Dial):** 엔드포인트 간 트래픽 비율을 조절하여 신속하게 장애에 대응할 수 있습니다.

---

## 5. 결론: 어떤 전략을 선택해야 할까?

* **비용 절감이 우선**이며, 데이터는 실시간으로 보호하고 싶다 > **파일럿 라이트**
* **복구 시간(RTO) 단축**이 최우선이며, 평상시 모의 테스트가 중요하다 > **웜 스탠바이**