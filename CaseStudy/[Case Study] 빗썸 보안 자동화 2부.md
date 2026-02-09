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