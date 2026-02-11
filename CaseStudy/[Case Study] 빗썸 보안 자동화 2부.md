출처 : https://aws.amazon.com/ko/blogs/tech/building-ec2-diagnostics-automation-using-systems-manager-part2/

## 1. SSM Command 문서 작성 및 중앙관리 방안

- EC2 내 수행이 필요한 스크립트를 중앙 계정에서 SSM 에서 Command 유형의 문서 스키마에 따라 작성한 후 멤버 계정들에 공유하여 배포하는 방식을 채택한다.
- Command 문서에 대한 중앙 관리가 가능해졌고 명령 수행 단계를 세분화해 명령 수행이 실패한 경우 어느 과정에서 실패했는지 가시성을 확보할 수 있게 됨

- 또한 Command 문서(AWS원격 스크립트 실행 파일)의 `parameters` 필드를 통해 Command 수행 시 취약성 점검 스크립트의 URL 을 인스턴스에 전달할 수 있도록 구성
- `allowPattern`을 통해 허용된 위치에서 생성된 URL 만 인스턴스 단에 전달 가능하도록 통제하는 절차 구현
    - 중앙 계정에서 AWS SSM 문서에서 제공하는 aws:runShellScript 유형의 동작을 활용해 커맨드 작성
    - 중앙 계정에서 AWS SSM 문서를 멤버 계정에 공유
        - 중앙 계정에 한번만 SSM 문서를 작성하면 모든 멤버계정의 서버에 공유해서 사용할 수 있음
- 파라미터 선언 부분
    - `allowedPattern` 에 특정 S3 버킷에서 오는 주소만 허용토록 함

```json
{
  "parameters": {
    "scriptUrl": {
      "type": "String",
      "description": "URL for the script to be downloaded",
      "allowedPattern": "^https:\\/\\/bucketname\\.s3\\.ap-northeast-2\\.amazonaws\\.com\\/.*$"
    },
    "uploadUrl": {
      "type": "String",
      "description": "URL for the script to be uploaded",
      "allowedPattern": "^https:\\/\\/bucketname\\.s3\\.ap-northeast-2\\.amazonaws\\.com\\/.*$"
    },
    "workingDirectory": {
      "type": "String",
      "description": "(Optional) The path to the working directory on your instance.",
      "allowedPattern": "^\\/tmp\\/runcommand_script./*$"
    }
  }
}
```

- 점검 스크립트 다운로드 관계
    - "action": "aws:runShellScript" 이 부분으로 쉘 스크립트 실행

```json
{
  "action": "aws:runShellScript",
  "name": "Download_CCE_Script",
  "precondition": {
    "StringEquals": [
      "platformType",
      "Linux"
    ]
  },
  "maxAttempts": 3,
  "inputs": {
    "workingDirectory": "{{ workingDirectory }}",
    "runCommand": [
      "#!/bin/bash",
      "#################################################",
      "# Get CCE Script",
      "#################################################",
      "curl -o script.tar '{{ scriptUrl }}'",
      "curl -sS -w \"%{http_code}\" -o {{ workingDirectory }}/script.tar '{{ scriptUrl }}'",
      "tar -xf script.tar"
    ]
  }
}
```

- 점검 스크립트 실행 단계
    - 실제 서버의 취약점을 점검하는 단계
    - precondition: 리눅스 서버인지 확인하고 실행

```json
{
  "action": "aws:runShellScript",
  "name": "Run_CCE_Script",
  "maxAttempts": 1,
  "precondition": {
    "StringEquals": [
      "platformType",
      "Linux"
    ]
  },
  "inputs": {
    "workingDirectory": "{{ workingDirectory }}",
    "runCommand": [
      "#!/bin/bash",
      "#################################################",
      "# RUN CCE Script",
      "#################################################",
      "source script.sh"
    ]
  }
}
```

- 점검 결과 S3 업로드 단계
    - 점검이 끝난 결과 파일을 S3로 올리는 단계
    - PUT 방식으로 S3의 사전 서명된 URL 방식을 사용해 별도의 S3 접근 권한이 없어도 안전하게 파일을 업로드할 수 있음

```json
{
  "action": "aws:runShellScript",
  "name": "Upload_result",
  "maxAttempts": 3,
  "precondition": {
    "StringEquals": [
      "platformType",
      "Linux"
    ]
  },
  "inputs": {
    "workingDirectory": "{{ workingDirectory }}",
    "runCommand": [
      "#!/bin/bash",
      "#################################################",
      "# Upload Result",
      "#################################################",
      "hostname=$(hostname)",
      "result=\"${hostname}.tar\"",
      "curl -sS -w \"%{http_code}\" -X PUT -T \"$result\" \"{{ uploadUrl }}\""
    ]
  }
}
```

단계를 세분화해 오류 추적 모니터링 용이

## 2. 중앙 계정의 Lambda 함수에서 멤버 계정으로 RunCommand 수행하기

- SSM 자동화 기능을 통해 AWS Organizations 서비스를 통해 구성된 OU 별로 Lambda 함수 구현 없이 RunCommand 수행이 가능
- 다만 각 인스턴스의 점검 결과를 S3의 Presigned URL 을 통해 S3로 업로드하는 방식을 채택하여 각 인스턴스별로 각기 다른 URL 파리미터를 주입해야 한다는 점에서 Lambda함수를 통한 RunCommand를 수행하게 되었음
    
    <aside>
    💡
    
    이거를 IAM 역할로 하게 된다면 서버가 해킹됐을 때 발생 범위가 S3까지 확산될 수 있음
    
    사전서명된 URL(Presigned URL)을 사용하면 특정 시간 동안만 특정 이름의 파일만 허용하는 주소를 던저주기 때문에 여러 인스턴스를 관리하는데 유리
    
    </aside>
    
    - Signed URL 과 Presigned URL 의 차이점
        
        
        | 구분 | Presigned URL | Signed URL |
        | --- | --- | --- |
        | 관련 서비스 | Amazon S3 | Amazon Cloudfront |
        | 인증 주체 | IAM 사용자 또는 역할 | Cloudfront 키 페어 |
        | 주요 용도 | 업로드/다운로드 모두 가능 | 주로 콘텐츠 배포/ 다운로드
        넷플릭스나 유료 강의 사이트에서 결제한 사람에게만 특정 시간동안 영상 보여주기 |
        | 특징 | S3버킷에 직접 연동 | 엣지 로케이션을 거쳐서 오는 전용 통로 |
        | 비용 | URL 자체는 무료지만 
          1. S3 에서 인터넷으로 파일을 보낼 때 발생하는 비용
          2. S3 요청 비용(PUT/GET 요청) : 1000건당 약 0.005달러 수준으로 저렴
          3. Lambda 실행 비용 |  |
        | 트레이드 오프 | Presigned URL 은 코드로 직접 구현해야 하지만 커스터마이징의 유연성이 크다 |  |

### Presigned URL 만드는 Lambda 예시코드

```python
import boto3
from botocore.exceptions import ClientError

def create_presigned_url(bucket_name, object_name, expiration=600):
    s3_client = boto3.client('s3', region_name='ap-northeast-2')
    try:
        # 'put_object'는 업로드, 'get_object'는 다운로드 권한입니다.
        response = s3_client.generate_presigned_url('put_object',
                                                    Params={'Bucket': bucket_name,
                                                            'Key': object_name},
                                                    ExpiresIn=expiration) # 유효 기간(초)
    except ClientError as e:
        return None
    return response

# 사용 예시: 10분 동안 'my-bucket'에 'server-01.tar'를 올릴 수 있는 주소 생성
url = create_presigned_url("my-bucket", "server-01.tar", 600)
print(url)
```

- 모든 EC2 인스턴스 대상으로 앞서 작성한 SSM Document를 전달하여 명령을 수행하거나
- 인스턴스를 직접 선택하여 명령을 수행할 수 있도록 상황에 맞는 취약점 점검을 수행하기 위해 Lambda 함수를 역할에 맞게 배치하여 RunCommand 수행 과정을 자동화 하였음
- 이 과정에서 다른 계정에서 RunCommand 를 실행할 수 있는 권한을 부여받기 위해 AWS Security Token Service 의 AssumeRole API 를 사용하여 수행
    - STS(Security Token Service) 의 AssumeRole 을 사용하면 잠시동안 다른 계정의 권한을 빌려 쓸 수 있음
- 먼저 모든 멤버 계정 내 테라폼을 통해 필요한 권한을 가진 IAM 역할을 작성 후 배포
    - 해당 IAM 역할은 신뢰관계 내 보안 주체에 Lambda 함수에 부착된 IAM 역할의 ARN 을 입력해두어 Lambda 함수에서 해당 권한을 정상적으로 획득할 수 있도록 구현
    - ARN이란? (Amazon Resource Name)
        
        AWS의 모든 자원에 붙혀지는 고유 식별 번호
        
        - ARN 템플릿
            - `arn:` 으로 시작하며 `:` 으로 구분된 일정한 규칙을 따른다
            - 형식 : `arn:aws:서비스:리전:계정번호:자원유형/자원이름`
    
    ![image.png](attachment:4f62fc84-4e91-4596-98c0-97ab06f4299f:image.png)
    
1. 모든 멤버 계정에 똑같은 이름의 IAM 역할을 만든다
    1. 정책 : ssm:SendCommand를 할 수 있는 역할
    2. 신뢰관계 : 오직 중앙계정의 Lambda ARN 을 함수만 역할을 빌려갈 수 있다.
2. `Lamdba 함수가 실행되면 멤버 계정의 STS에 연결
    1. STS 는 Lambda의 신분을 확인하고 1시간 정도만 유효한 임시 보안 토큰 전달
3. RunCommand 실행
    1. Lambda 함수는 임시토큰으로 멤버계정의 SSM 서비스에게 명령을 내린다.
1. 앞서 작업 대상 계정에 생성했던 IAM 역할에 대한 AssumeRole 요청을 수행하여 자격증명을 획득
2. SSM 서비스의 DescribeInstancesInformation API를 호출하여 InstanceID 와 SSM Document Parameter에 활용될 Instance Name, Instance IP를 획득, 진단이 필요한 OS 타입을 지정하거나 SSM 서비스와의 연동 상태가 정상인 인스턴스를 확인하는 등 필터 옵션을 통해 특정 조건을 만족하는 인스턴스를 선별할 수 있음
3. SSM 서비스의 Send Command API 를 통해 2번 과정에서 획득한 InstanceID 리스트 대상으로 미리 작성한 SSM 문서 실행
    1. SSM Run Command 내 태그, 리소스 그룹등을 바탕으로 명령을 대규모로 실행하는 기능 지원, 이를 통해 동시 실행 가능한 범위 지정, 오류 발생 시 중단 기능을 사용할 수 있음. 위 다이어그램에선 인스턴스마다 각기 다른 파라미터를 주입하기 위해 Lambda 함수를 통해 반복 처리하도록 구성
    2. 인스턴스가 다수인 경우 쓰로틀링을 방지하기 위해 예시와 같은 요청 속도를 제한하는 옵션 사용을 고려할 수 있음
        - 쓰로틀링
            - 시스템이 감당할 수 있는 수준까지 트래픽을 받고 그 이상의 요청은 잠시 멈추거나 차단하여 시스템 전체의 다운을 막는 방어 기술
            
            핵심 목적
            
            1. 공정성 : 한 명의 사용자가 트래픽을 독점해서 다른 사용자들이 서비스를 못 쓰는 상황을 방지
            2. 시스템 보호 : 서버나 DB 처리 한계치를 넘지 않게 조절하여 시스템이 느려지거나 다운되는 것을 방지
            3. 비용 통제 : Lambda나 SSM 처럼 호출 당 비용이 발생하는 서비스에서 예기치 못한 호출 폭주로 인한 비용을 방지
            - 쓰로틀링이 발생하면 서버는 HTTP 429 (Too Many Requests) 상태코드를 보냄
            
            쓰로틀링 해결책
            
            - 쓰로틀링이 있을때 바로 다시 요청하면 거절 당할 확률이 높으니 지수 백오프를 사용해 지수적으로 대기시간을 늘려 요청을 수행
                - AWS SDK(Boto3등)은 이걸 내부적으로 자동으로 해줌
            - 
        
        ```go
        func GetDefaultClient(ctx context.Context, accountID string) (*SSMClient, error) {
        	cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion(TargetRegion), config.WithRetryer(func() aws.Retryer {
        		return retry.NewStandard(func(o *retry.StandardOptions) {
        			o.RateLimiter = ratelimit.NewTokenRateLimit(20)// 쓰로틀링 방지용 rate limit
        			o.RetryCost = 1
        			o.RetryTimeoutCost = 3
        			o.NoRetryIncrement = 10
        		})
        	}))
        
        	if err != nil {
        		return nil, err
        	}
        
        	stsClient := sts.NewFromConfig(cfg)// AWS 보안 토큰 서비스(STS) 호출
        	creds := stscreds.NewAssumeRoleProvider(stsClient, fmt.Sprintf(AssumeRoleARN, accountID, AssumeRoleName)) // 다른 계정 열쇠 빌리기
          cfg.Credentials = aws.NewCredentialsCache(creds) // 임시 신분증 보관
        	cfg.Credentials = aws.NewCredentialsCache(creds)
        
        	ssmClient := ssm.NewFromConfig(cfg)// 리모컨 완성
        
        	return &SSMClient{ssmClient}, nil
        }
        ```
        
    - SendCommand API 호출 시 완료/실패 수를 저장한 뒤, 모든 로직이 마무리되고 결과 보고서를 메신저로 전달하도록 구성
    - 빗썸에서는 Slack에서 제공하는 API 를 활용해 아래와 같은 템플릿으로 결과가 전달되도록 처리
        
        ![image.png](attachment:0271b173-ccb7-4a24-9a25-37a325f4bb3f:image.png)
        
    
    ### IAM 역할 및 정책 작성 후 배포하기
    
    - Lambda 함수에서 사용하고 있는 IAM 역할 ARM 값을 신뢰관계의 principal 부분에 명시함으로써 Lambda 함수에서 해당 권한을 정상적으로 획득할 수 있도록 구현
    - IAM 정책
    
    ```hcl
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "ssm:SendCommand",
            "ssm:ListCommandInvocations",
            "ssm:ListCommands",
            "ssm:DescribeInstanceInformation"
          ],
          "Resource": "*"
        },
        {
          "Effect": "Allow",
          "Action": "ec2:DescribeInstances",
          "Resource": "*"
        }
      ]
    }
    ```
    
    - IAM 신뢰 관계
    
    ```hcl
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "AWS": [
              "arn:aws:iam::123456789012:role/service-role/runcommand-lambda-single-role-******",
              "arn:aws:iam::123456789012:role/service-role/run-command-lambda-role-******"
            ]
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
    ```
    

## 3. S3 Presigned GET URL 을 통한 EC2 내 점검 스크립트 전달

- 인스턴스의 보안설정 오류(misconfiguration) 취약점을 확인할 수 있는 쉘 스크립트를 인스턴스에 전달하기 위해, 먼저 중앙 S3 버킷에 점검 스크립트를 업로드 하였음
- 이후 Lambda 함수에서 일정 시간 동안만 접근할 수 있는 Presigned URL을 생성하여 SSM문서에서 정의한 downloadURL 파라미터에 해당 URL 주소를 전달함으로써 인스턴스에서 S3 객체에 대한 접근 권한이 없어도 스크립트를 다운로드 받을 수 있게 구성
- S3 버킷에 업로드 하는 방안은 다음과 같이 여러가지가 있으나 문서 수정 없이도 점검 스크립트를 유연하게 변경할 수 있어야 하고 EC2 의 인스턴스 프로필 내 명시적인 정책 추가 없이 취약점 점검을 해야하는 요구사항을 충족시키기 위해 Presigned URL 을 활용하여 업로드하는 방안을 선택
    1. **Command 문서에 스크립트를 직접 추가하는 방법**
        1. 장점 
            1. **권한 설정 필요 없이** 문서 하나 실행 가능
            2. 속도가 가장 **빠르다**
        2. 단점
            1. 스크립트가 길어지면 **가독성이 처참**해지고 **수정할 때마다 문서 버전까지 업데이트** 해줘야한다.
            2. 보안 취약 : 문서에 접근 권한이 있는 사람은 누구나 점검 로직을 볼 수 있어 **노출 위험도**가 크다
            3. **재사용성 낮음** : 다른 자동화 도구에서 이 스크립트만 따로 쓰고 싶어도 문서에서 떼어내기가 어려움
    2. **EC2 의 인스턴스 프로필에 특정 S3 버킷만 접근 가능하도록 정책을 추가하고, 해당 버킷의 버킷 정책으로 접근 제어하는 방법**
        1. 장점
            1. **대용량 처리** : 스크립트가 매우 크거나 여러 개의 파일을 가져와야 할 때 유리
            2. **표준화** : AWS 가 권장하는 가장 일반적인 자원 관리 방식
        2. 단점
            1. **권한 관리 오버헤드** : 수천 대의 EC2 인스턴스 프로필에 S3 접근 권한을 일일이 넣어야 함
            2. **보안 리스크** : 서버가 해킹당하면 해당 서버는 언제든 S3 버킷에 다시 들어가서 정보를 빼오거나 조작할 수 있음
            3. **복잡한 버킷 정책** : 멀티 계정 환경에서는 S3 버킷 정책이 엄청나게 길어지고 복잡해짐
    3. **S3의 스크립트를 다운로드할 수 있는 Presigned URL 을 통해 일정 시간만 접근할 수 있는 URL을 생성하여 전달하는 방법 (Zero Trust)**
        1. 장점
            1. **최고의 보안성** : EC2 에 S3 접근 권한을 줄 필요가 없고 주소는 일정 시간이 지나면 무효화되어 서버가 해킹되어도 s3 는 안전
            2. 어느 계정의 서버든 url 만 있으면 접근 가능하므로 **복잡한 교차 계정 권한 설정을 대폭 줄일 수 있다**.
            3. **입력값 검증** : `allowedPattern` 을 통해 특정 경로의 URL 만 허용하도록 이중 잠금을 할 수 있음
        2. 단점
            1. **구현 복잡도** : URL 을 생성하고 전달하는 **Lambda 로직을 직접 코딩**해야 함
            2. **네트워크 의존** : 인스턴스가 S3 엔드포인트에 접속할 수 있는 **네트워크 환경(인터넷 또는 PrivateLink)이 반드시 갖춰있어야 함** 

### PresignedURL 생성 코드

- 다음은 Lambda 함수에 부여된 IAM 역할을 통해 특정 버킷 내 객체를 다운로드 할 수 있는 PresignedURL을 생성하는 예제 코드이다.
- 지정한 time. Duration(분) 시간 만큼 유효한 URL 을 생성해 인스턴스에 전달함으로써, 인스턴스에 s3 관련 정책을 부여하지 않고, 특정 S3 버킷에 업로드돼 있는 객체의 다운로드가 가능하다.
    - Presigned URL 생성코드
    
    ```go
    func CreateURLToGet(ctx context.Context, key string, time_expire int64) (string, error) {
    	cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion(TargetRegion))
    	if err != nil {
    		return "", err
    	}
    	s3Client := s3.NewFromConfig(cfg)
    	PresignClient := s3.NewPresignClient(s3Client)
    
    	resp, err := PresignClient.PresignGetObject(ctx, &s3.GetObjectInput{
    		Bucket: aws.String(BucketName),
    		Key:    aws.String(key),
    	}, func(o *s3.PresignOptions) {
    		o.Expires = time.Minute * time.Duration(time_expire)
    	})
    
    	if err != nil {
    		return "", err
    	}
    
    	return resp.URL, nil
    }
    ```
    
    - Presigned URL을 활용한 SSM 문서 단계
    
    ```json
    {
      "action": "aws:runShellScript",
      "name": "Download_CCE_Script",
      "precondition": {
        "StringEquals": [
          "platformType",
          "Linux"
        ]
      },
      "maxAttempts": 3,
      "inputs": {
        "workingDirectory": "{{ workingDirectory }}",
        "runCommand": [
          "#!/bin/bash",
          "#################################################",
          "# Get CCE Script",
          "#################################################",
          "curl -o script.tar '{{ scriptUrl }}'",
          "curl -sS -w \"%{http_code}\" -o {{ workingDirectory }}/script.tar '{{ scriptUrl }}'",
          "tar -xf script.tar"
        ]
      }
    }
    ```
    

## 4. S3 Presigned PUT URL 을 통한 점검결과 수집

- 취약점 점검 결과는 `tar` 형태로 서버 내 생성됨
- 생성된 결과파일을 S3 버킷으로 업로드하기 위해 특정 Key로 객체를 업로드할 수 있는 권한이 부여된 Presigned URL을 생성했고, SSM 문서에 정의한 `uploadURL` 파라미터로 해당 URL 주소를 전달함으로써 인스턴스에서 S3 객체에 대한 접근 권한이 명시적으로 없어도 결과 파일을 업로드할 수 있도록 구성
    
    ![image.png](attachment:88374778-be08-4996-a178-d9c4bc51e31b:image.png)
    
    계정별로 Prefix 지정하여 결과 수집
    
    ![image.png](attachment:3fa2062c-cc34-4b83-8d53-c72ae13db351:image.png)
    
    한 계정의 전체 인스턴스 점검 결과가 수집된 모습
    
    ### PresignedURL 생성 코드
    
    - 업로드 Presigned URL 생성 예제 코드
    - 지정한 time.Duration시간만큼 유효한 URL을 싱성해 인스턴스 단에 전달하므로 인스턴스 내 s3 관련 권한을 부여하지 않고 특정 s3로 파일 업로드가 가능
    - 이 경우 Lambda에 부여된 Role의 Policy를 따르므로 생성한 URL 에 IP ACL과 같은 보안 정책 적용 가능
    - Presigned URL 생성 코드
    
    ```go
    func CreateURLToPut(ctx context.Context, key string, time_expire int64) (string, error) {
    	cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion(TargetRegion))
    	if err != nil {
    		return "", err
    	}
    	s3Client := s3.NewFromConfig(cfg)
    	PresignClient := s3.NewPresignClient(s3Client)
    
    	resp, err := PresignClient.PresignPutObject(ctx, &s3.PutObjectInput{
    		Bucket: aws.String(BucketName),
    		Key:    aws.String(fmt.Sprintf("%s/%s/%s", "Result", Target, key)),
    	}, func(o *s3.PresignOptions) {
    		o.Expires = time.Minute * time.Duration(time_expire)
    	})
    
    	if err != nil {
    		return "", err
    	}
    
    	return resp.URL, nil
    }
    ```
    
    - PresignedURL을 활용한 SSM 문서 단계
    
    ```go
    {
      "action": "aws:runShellScript",
      "name": "Upload_result",
      "maxAttempts": 3,
      "precondition": {
        "StringEquals": [
          "platformType",
          "Linux"
        ]
      },
      "inputs": {
        "workingDirectory": "{{ workingDirectory }}",
        "runCommand": [
          "#!/bin/bash",
          "#################################################",
          "# Upload Result",
          "#################################################",
          "hostname=$(hostname)",
          "result=\"${hostname}.tar\"",
          "curl -sS -w \"%{http_code}\" -X PUT -T \"$result\" \"{{ uploadUrl }}\""
        ]
      }
    }
    ```
    

## 5. 수집된 결과 파일 분석 및 시각화

![image.png](attachment:53cd48ef-7530-4215-8357-c3eca2c18734:image.png)

- 취약점 점검 결과 파일은 중앙 s3 버킷에 저장됨
- 위 아키텍처를 통해 인스턴스 취약점 점검 결과는 정기적으로 수집되어 최신 보안 상태를 가진다.
- 이를 분석하여 시각화하는 아키텍처 구성
- AWS에는 S3버킷에 저장된 데이터를 분석할 수 있는 다양한 도구들이 제공되지만 이 아키텍처에는 별도의 분석 서버를 운영하는 방식을 채택함
    - 분석 작업이 데이터 처리량이 많고, 특정 라이브러리나 환경이 요구되는 경우가 있기 때문
- → 이 요구 사항을 충족하기 위해 S3 에서 분석 서버로 S3 sync 명령을 통해 결과 파일을 일괄적으로 전달하여 분석 작업을 수행

```bash
aws s3 sync s3://<버킷명>/results/ /path/to/local/analysis/
```

- 분석 서버에선 전달받은 파일 기준으로 취약점 내용을 파싱하고, 필요한 정보를 추출
- 분석 결과는 SIEM 솔루션에서 쉽게 처리할 수있도록 구조화 시키는 목적으로 JSON 형식으로 반환
- 분석이 완료된 JSON 파일들은 다시 S3 버킷으로 업로드

```bash
aws s3 sync s3://<버킷명>/results/ /path/to/local/analysis/
```

- SIEM 솔루션에선 S3 버킷 내 특정 Prefix에 저장된 JSON 파일을 모니터링하며 새로운 파일이 업로드되면 이를 자동으로 수집
- 수집된 데이터는 시각화 도구를 통해 분석되고 검색 쿼리를 통해 필요한 정보를 손쉽게 조회할 수 있음
- 위 과정으로 시각화된 결과물을 관련 부서들과 함께 확인할 수 있는 환경을 만듦으로써 효율적인 취약점 관리 및 개선방안 도출 가능해짐