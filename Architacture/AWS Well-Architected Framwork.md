# Well-Architected Framework

## 운영 우수성

- 코드형 운영
    - Systems Manager 의 패치 매니저 사용하면 일정 지정하고 여러 인스턴스를 패치 가능
    - MSA 기능별 코드 분리에서 AWS CodePipeline 사용하여 CI/CD 사용
        
        ![image.png](attachment:e56ba7ec-6e6c-4933-a989-43b6a2726b35:image.png)
        
        - CodeCommit = git
        - CodeDeploy 는 젠킨스 역할
    - Session Manager 을 사용해 관리형 노드를 사용
- 로드벨런서를 통해 트래픽 분산화 엔드포인트 역할 수행
    - RDS 는 고가용성을 위해 멀티 에이지 구조 구조
    
    ![image.png](attachment:3c3630c2-c4a4-44d0-a676-0c3c5dd4f81f:image.png)
    

## 보안

- AWS Organizations 를 사용해서 파급효과(Blast Radius) 최소화 할 수 있다.

![image.png](attachment:09173ac7-2e47-4180-9134-1b3dac4347bc:image.png)

- 탐지 제어를 사용하면 계정 활동을 파악할 수 있음

![image.png](attachment:0e1248f2-81b8-45e8-a69f-1dd937f39e33:image.png)

- 모든 계층에 보안 적용
    - Amazon CloudFront 활용해 DDoS 완화 지원하고 VPC 내 애플리케이션 층까지 ALB 를 사용해 공격 표면을 줄인다.
    - NACL 및 보안 그룹을 사용해 최소권한을 부여하여 휴먼에러로 인한 파급효과 최소화

![image.png](attachment:0e1a3b89-64a8-40f5-a928-c59fd97341b2:image.png)

- 전송 중 및 저장 시 데이터 보호
    - 민감도 수준에 따라 조직 데이터를 분류하는 방법을 제공
    - 암호화는 무단 액세스가 데이터를 이해할 수 없도록 하여 데이터를 보호

![image.png](attachment:a4d3dccf-0d68-4c4a-a243-b7581bcfb613:image.png)

- 데이터 수동처리 최소화
    - 데이터 수동처리 최소화하면 데이터 원본이 손상될 가능성 줄어듦
    - 아래가 베스트 프렉티스

![image.png](attachment:e390d7f7-83ab-45e1-a7e5-6c1583d515cb:image.png)

## 안정성

![image.png](attachment:43370c66-6434-459a-9e18-06072883cd7e:image.png)

- DR, 복구 절차 테스트, 수평적 확장, 변경관리 자동화
- 장애 페일오버 자동화
    
    ![image.png](attachment:15aea146-4d69-4d91-9fdb-d8dbf4011736:image.png)
    
    - 오토스케일링그룹
    - Amazon ElastiCache 사용
- AWS FIS(Fault Injection Simulator) : 오류 주입 실험을 수행하기 위한 완전 관리형 서비스
    - EC2 를 terminate 해 확인 = 넷플릭스의 카오스 몽키와 유사

![image.png](attachment:f00a3a7a-f9ec-40f6-975c-b5818b4d8e8e:image.png)

![image.png](attachment:97c8ea81-c53b-4341-b15d-232b3028b3a5:image.png)

## 성능 효율성

- 수요가 변화하고 기술이 발전함에 따라 리소스가 비즈니스 및 기술 요구사항을 모두 충족하고 효율성을 유지하는 것
    - NoSQL 업데이트 → DynamoDB 사용
    - AWS CloudFormation 으로 몇분만에 배포
    - 서버리스 아키텍처 사용
        - 인스턴스 ASG 는 인스턴스 시작과 용량 추가 사이에 지연이 있다
        - 서버리스는 사라므이 개입이나 사용자가 생성한 규칙을 모니터링하지 않아도 동적으로 확장됨
            - 하지만 Cold Start 문제가 있다.
    - 실험 빈도 증가
        - 새로운 인스턴스를 테스트해서 변경
    - 기계적 동조 고려
        - 최적의 작동방식에 대한 이해를 바탕으로 도구를 잘 활용하는 것
        
        ![image.png](attachment:74e5e888-32b4-49fc-8085-10890818bc4f:image.png)
        
    

## 비용 최적화

- 가장 낮은 가격으로 비즈니스 가치를 제공하는 시스템 운영 능력
    - 사용하지 않는 리소스를 종료, 일시중단
    - 사용한만큼만 지불
    - 서버리스 아키텍처로 전환