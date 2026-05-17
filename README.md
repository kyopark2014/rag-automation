# Bedrock Data Automation (BDA)를 이용한 RAG 구현

## 1. 개요

Amazon Bedrock Data Automation(BDA)은 문서, 이미지, 영상, 오디오 등 비정형 콘텐츠에서 가치 있는 인사이트를 추출하는 완전 관리형 클라우드 서비스입니다. 생성형 AI를 활용하여 멀티모달 데이터를 구조화된 형식으로 변환하며, 복잡한 AI 모델 오케스트레이션 없이 단일 API로 처리할 수 있습니다.

**Knowledge Bases에서의 Parser 역할:**
Amazon Bedrock Knowledge Bases의 데이터 소스 수집(Ingestion) 단계에서 BDA를 파서(Parser)로 지정하면, PDF, 이미지, 오디오, 비디오 등 멀티모달 콘텐츠를 텍스트 표현으로 변환하거나 멀티모달 임베딩을 위한 원본 파일로 저장하여 RAG(Retrieval Augmented Generation) 기반 질의응답 애플리케이션의 품질을 크게 향상시킬 수 있습니다.


## 2. 파서 옵션 비교

Knowledge Bases에서 사용할 수 있는 파서는 세 가지입니다.

| 구분 | 기본 파서 (Default) | BDA 파서 | 파운데이션 모델 파서 |
|------|---------------------|----------|----------------------|
| 지원 형식 | .txt, .md, .html, .doc/.docx, .xls/.xlsx, .pdf (텍스트만) | PDF, JPEG, PNG, 오디오, 비디오 | PDF, JPEG, PNG, 구조화 문서 |
| 멀티모달 처리 | 불가 | 가능 (이미지, 도표, 표, 오디오, 비디오) | 가능 (이미지, 도표, 표) |
| 프롬프트 커스터마이징 | 불가 | 불가 | 가능 |
| 비용 구조 | 무료 | 페이지/이미지 수 기준 과금 | 입출력 토큰 수 기준 과금 |
| 파일 크기 합계 제한 | - | - | 최대 100 GB |

> **중요:** BDA 또는 파운데이션 모델을 파서로 선택하면, 해당 데이터 소스의 모든 PDF 파일에 해당 파서가 적용됩니다. 텍스트만 포함된 PDF도 예외 없이 과금 대상이 됩니다.



## 3. BDA 파서가 처리하는 데이터 유형

### 문서 (Documents)

- 지원 형식: PDF, TIFF, JPEG, PNG, DOCX
- 텍스트 추출, 도표/차트 설명, 표 구조 인식, 손글씨 인식 포함
- DOCX 파일은 내부적으로 PDF로 변환하여 처리 (이 경우 페이지 번호 매핑 불가)

### 이미지 (Images)

- 지원 형식: JPEG, PNG
- 이미지 내 텍스트 추출(OCR), 시각적 설명 생성

### 오디오 (Audio)

- 지원 형식: AMR, FLAC, M4A, MP3, Ogg, WAV
- 지원 언어: 영어, 독일어, 스페인어, 프랑스어, 이탈리아어, 포르투갈어, 일본어, 한국어, 중국어 (대만, 광둥어 포함)
- 음성을 텍스트 트랜스크립트로 변환

### 비디오 (Video)

- 지원 형식: MP4, MOV, AVI, MKV, WEBM (H.264, H.265/HEVC, VP8, VP9 등 코덱 지원)
- 장면 요약, 텍스트 추출, 콘텐츠 분류, 오디오 트랜스크립트 생성



## 4. 파일 처리 제한 사항

### 비동기(Async) 처리 제한

| 항목 | 제한 |
|------|------|
| 최대 페이지 수 (문서 분할 활성화 시) | 3,000 페이지 |
| 최대 파일 크기 | 500 MB |
| 최대 비디오 길이 | 240분 |
| 최대 오디오 길이 | 240분 |
| 최대 이미지 해상도 | 8K |
| 최소 텍스트 감지 높이 | 15픽셀 (150 DPI 기준 8pt 폰트) |

### 동기(Sync) 처리 제한

| 항목 | 제한 |
|------|------|
| 최대 페이지 수 | 10 페이지 |
| 최대 파일 크기 | 50 MB |

### 추가 제약

- 수직 방향 텍스트 (일본어, 중국어 등 세로쓰기) 인식 미지원
- 비밀번호로 보호된 PDF 처리 불가
- PDF 최대 높이/너비: 40인치 / 9,000포인트

### 프로젝트/블루프린트 제한

| 항목 | 제한 |
|------|------|
| 프로젝트당 최대 블루프린트 수 | 40개 |
| 계정당 최대 프로젝트 수 | 100개 |
| 계정당 최대 블루프린트 수 | 1,000개 |
| 블루프린트 최대 이름 길이 | 60자 |
| 블루프린트 최대 크기 | 100,000자 (JSON 형식) |



## 5. BDA 파서 동작 방식

BDA는 크게 두 가지 출력 방식을 제공합니다.

### 표준 출력 (Standard Output)

별도의 블루프린트(Blueprint)나 프로젝트 없이 파일을 전송하면 해당 파일 유형에 맞는 기본 출력을 반환합니다.

| 데이터 유형 | 기본 출력 내용 |
|-------------|----------------|
| 문서 | 텍스트 추출, 문서 요약 |
| 오디오 | 전체 트랜스크립트, 요약 |
| 비디오 | 장면 요약, 감지된 텍스트, 콘텐츠 분류 |
| 이미지 | 텍스트 추출, 시각적 설명 |

### 커스텀 출력 (Custom Output)

블루프린트를 사용하여 추출할 필드를 정확하게 정의합니다. 문서, 오디오, 이미지에 적용 가능하며 비즈니스 워크플로에 특화된 정보 추출이 가능합니다.

### 프로젝트 (Projects)

표준 출력 및 커스텀 출력 구성을 하나의 리소스로 관리합니다. `InvokeDataAutomationAsync` API 호출 시 프로젝트 ARN을 지정하면 해당 구성에 따라 파일이 처리됩니다.

- 프로젝트당 최대 블루프린트 수: 40개
- 계정당 최대 프로젝트 수: 100개
- `LIVE` / `DEVELOPMENT` 두 가지 스테이지 지원
- `DEVELOPMENT` 스테이지는 콘솔에서 접근 불가, API를 통해서만 변경 및 호출 가능



## 6. API 구성 방법

Knowledge Bases 데이터 소스 생성 시 파싱 전략을 `BEDROCK_DATA_AUTOMATION`으로 설정합니다.

### ParsingConfiguration 구조

```json
{
  "parsingStrategy": "BEDROCK_DATA_AUTOMATION",
  "bedrockDataAutomationConfiguration": {
    "parsingModality": "MULTIMODAL"
  }
}
```

- `parsingStrategy`: `BEDROCK_FOUNDATION_MODEL` 또는 `BEDROCK_DATA_AUTOMATION` 중 선택
- `parsingModality`: `MULTIMODAL` 지정 시 텍스트와 이미지를 모두 포함한 멀티모달 파싱 활성화
- BDA 또는 파운데이션 모델 파서가 파일 파싱에 실패하면, 자동으로 기본 파서로 폴백(fallback) 처리

### AWS CLI를 이용한 Knowledge Base 생성 예시

```bash
aws bedrock-agent create-knowledge-base \
  --cli-input-json file://kb-bda-parser.json
```

`kb-bda-parser.json` 파일 내용 (플레이스홀더를 실제 값으로 교체):

```json
{
  "knowledgeBaseConfiguration": {
    "vectorKnowledgeBaseConfiguration": {
      "embeddingModelArn": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-2-multimodal-embeddings-v1:0",
      "supplementalDataStorageConfiguration": {
        "storageLocations": [
          {
            "type": "S3",
            "s3Location": {
              "uri": "s3://<multimodal-storage-bucket>/"
            }
          }
        ]
      }
    },
    "type": "VECTOR"
  },
  "storageConfiguration": {
    "opensearchServerlessConfiguration": {
      "collectionArn": "arn:aws:aoss:us-east-1:<account-id>:collection/<collection-id>",
      "vectorIndexName": "<index-name>",
      "fieldMapping": {
        "vectorField": "<vector-field>",
        "textField": "<text-field>",
        "metadataField": "<metadata-field>"
      }
    },
    "type": "OPENSEARCH_SERVERLESS"
  },
  "name": "<knowledge-base-name>",
  "description": "Knowledge base with BDA parser"
}
```



## 7. Cross-Region Inference (CRIS) 필수 요건

BDA는 크로스 리전 추론(Cross-Region Inference)을 반드시 사용해야 합니다. 이를 통해 요청된 지리적 경계 내에서 최적의 리전을 자동으로 선택하여 처리하며, 추가 비용은 발생하지 않습니다.

데이터는 원본 소스 리전에만 저장되며, 전송 중에는 AWS의 암호화된 보안 네트워크를 통해 처리됩니다.

| 소스 리전 | ARN 패턴 | 처리 가능 리전 |
|-----------|----------|----------------|
| US East (N. Virginia) | `arn:aws:bedrock:us-east-1:account-id:data-automation-profile/us.data-automation-v1` | us-east-1, us-east-2, us-west-1, us-west-2 |
| US West (Oregon) | `arn:aws:bedrock:us-west-2:account-id:data-automation-profile/us.data-automation-v1` | us-east-1, us-east-2, us-west-1, us-west-2 |
| Europe (Frankfurt) | `arn:aws:bedrock:eu-central-1:account-id:data-automation-profile/eu.data-automation-v1` | eu-central-1, eu-north-1, eu-south-1, eu-south-2, eu-west-1, eu-west-3 |
| Europe (Ireland) | `arn:aws:bedrock:eu-west-1:account-id:data-automation-profile/eu.data-automation-v1` | eu-central-1, eu-north-1, eu-south-1, eu-south-2, eu-west-1, eu-west-3 |
| Europe (London) | `arn:aws:bedrock:eu-west-2:account-id:data-automation-profile/eu.data-automation-v1` | eu-west-2 |
| Asia Pacific (Mumbai) | `arn:aws:bedrock:ap-south-1:account-id:data-automation-profile/apac.data-automation-v1` | ap-northeast-1/2/3, ap-south-1/2, ap-southeast-1/2/4 |
| Asia Pacific (Sydney) | `arn:aws:bedrock:ap-southeast-2:account-id:data-automation-profile/apac.data-automation-v1` | ap-northeast-1/2/3, ap-south-1/2, ap-southeast-1/2/4 |
| AWS GovCloud (US-West) | `arn:aws:bedrock:gov-cloud:account-id:data-automation-profile/us-gov.data-automation-v1` | us-gov-west-1 |

### CRIS IAM 정책 예시

```json
{
  "Effect": "Allow",
  "Action": ["bedrock:InvokeDataAutomationAsync"],
  "Resource": [
    "arn:aws:bedrock:us-east-1:<account_id>:data-automation-profile/us.data-automation-v1",
    "arn:aws:bedrock:us-east-2:<account_id>:data-automation-profile/us.data-automation-v1",
    "arn:aws:bedrock:us-west-1:<account_id>:data-automation-profile/us.data-automation-v1",
    "arn:aws:bedrock:us-west-2:<account_id>:data-automation-profile/us.data-automation-v1"
  ]
}
```



## 8. IAM 권한 구성

BDA를 Knowledge Bases의 파서로 사용하려면 아래 IAM 권한이 필요합니다.

### BDA 파서 기본 권한

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeDataAutomationAsync",
    "bedrock:GetDataAutomationStatus"
  ],
  "Resource": [
    "arn:aws:bedrock:us-east-1:<account_id>:data-automation-profile/us.data-automation-v1",
    "arn:aws:bedrock:us-west-2:<account_id>:data-automation-profile/us.data-automation-v1"
  ]
}
```

### 멀티모달 스토리지 S3 권한

멀티모달 스토리지 대상 버킷에 대한 S3 읽기/쓰기 권한이 필요합니다.

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::<multimodal-storage-bucket>",
    "arn:aws:s3:::<multimodal-storage-bucket>/*"
  ]
}
```

### 고객 관리형 KMS 키 사용 시

KMS 키 작업 및 그랜트 생성 권한을 추가로 구성해야 합니다.

> **참고:** AWS Management Console에서 Knowledge Base를 생성하는 경우, Amazon Bedrock Knowledge Bases가 필요한 권한을 자동으로 구성합니다.



## 9. 멀티모달 스토리지 설정

| 구분 | Nova 멀티모달 임베딩 | BDA 파서 |
|------|----------------------|----------|
| 스토리지 설정 | 필수 | 선택 사항 |
| 스토리지 미설정 시 | 멀티모달 처리 불가 | 텍스트 파싱만 가능 |
| 스토리지 설정 시 | 이미지/오디오/비디오 직접 검색 가능 | 이미지/오디오/비디오 멀티모달 파싱 가능 |

### 스토리지 구성 권장 사항

- **별도 버킷 사용 (권장):** 데이터 소스 버킷과 멀티모달 스토리지 버킷을 분리하여 구성합니다. 설정이 단순하며 충돌 방지에 효과적입니다.
- **동일 버킷 사용 시:** 데이터 소스에 포함 접두사(inclusion prefix)를 반드시 지정하여 추출된 미디어 파일이 재수집되지 않도록 해야 합니다.
- **`aws/` 접두사 사용 금지:** 동일 버킷 사용 시, `aws/`로 시작하는 포함 접두사는 사용할 수 없습니다. 해당 경로는 추출된 미디어 저장용으로 예약되어 있습니다.

### S3 라이프사이클 정책 권장

Nova 멀티모달 임베딩 사용 시, Amazon Bedrock은 처리 완료 후 임시 데이터 삭제를 시도합니다. 임시 데이터 경로에 S3 라이프사이클 정책을 적용하여 정상적인 정리가 이루어지도록 설정할 것을 권장합니다.



## 10. 임베딩 모델과의 조합

BDA 파서는 두 가지 임베딩 모델 접근 방식과 함께 사용할 수 있습니다.

### 텍스트 임베딩 + BDA 파서

- Titan Text Embeddings v2 등 텍스트 임베딩 모델과 함께 사용
- BDA가 멀티모달 콘텐츠를 텍스트로 변환하여 저장
- 텍스트 기반 검색만 가능하나, 멀티모달 파싱 결과가 검색에 활용됨

### Nova 멀티모달 임베딩 + BDA 파서

- BDA 파싱이 먼저 수행된 후 Nova 멀티모달 임베딩이 적용됨
- 이 경우 Nova가 이미지/오디오/비디오에 대한 네이티브 멀티모달 임베딩을 생성하지 않고, BDA의 텍스트 변환 결과를 사용함

### 임베딩 모델 선택 가이드

| 상황 | 권장 구성 |
|------|-----------|
| 텍스트 문서 위주, 멀티모달 불필요 | 기본 파서 + 텍스트 임베딩 |
| PDF/이미지 포함, 텍스트 기반 검색 | BDA 파서 + 텍스트 임베딩 |
| 이미지/오디오/비디오 직접 시각 검색 | BDA 파서 + Nova 멀티모달 임베딩 |
| 음성 콘텐츠 검색 필요 | BDA 파서 (Nova 멀티모달 임베딩은 음성 검색 지원 제한적) |
| 이미지 전용 데이터셋 검색 | Titan Multimodal Embeddings G1 + 기본 파서 |


## 11. 주요 활용 사례

- **지능형 문서 처리 (IDP):** 계약서, 청구서, 양식 등에서 구조화된 데이터 추출 및 분류, 복잡한 오케스트레이션 없이 대규모 자동화 가능
- **멀티미디어 분석:** 영상에서 장면 요약, 부적절 콘텐츠 감지, 광고/브랜드 분류, 지능형 영상 검색 지원
- **RAG 강화:** 문서/이미지/오디오/비디오를 모두 포함하는 지식 베이스 구축으로 질의응답 정확도 향상
- **회의록/강의 분석:** 오디오 및 비디오 파일의 트랜스크립트 및 요약 자동 생성
- **복합 문서 검색:** PDF 내 도표, 차트, 표, 이미지가 포함된 문서의 시맨틱 검색


## 12. 참고 문서 링크

| 문서 | URL |
|------|-----|
| Parsing options for your data source | https://docs.aws.amazon.com/bedrock/latest/userguide/kb-advanced-parsing.html |
| What is Bedrock Data Automation | https://docs.aws.amazon.com/bedrock/latest/userguide/bda.html |
| How Bedrock Data Automation works | https://docs.aws.amazon.com/bedrock/latest/userguide/bda-how-it-works.html |
| Prerequisites for using BDA | https://docs.aws.amazon.com/bedrock/latest/userguide/bda-limits.html |
| Cross Region support for BDA | https://docs.aws.amazon.com/bedrock/latest/userguide/bda-cris.html |
| Standard output in BDA | https://docs.aws.amazon.com/bedrock/latest/userguide/bda-standard-output.html |
| Bedrock Data Automation projects | https://docs.aws.amazon.com/bedrock/latest/userguide/bda-projects.html |
| ParsingConfiguration API Reference | https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent_ParsingConfiguration.html |
| BedrockDataAutomationConfiguration API | https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent_BedrockDataAutomationConfiguration.html |
| Create a knowledge base for multimodal content | https://docs.aws.amazon.com/bedrock/latest/userguide/kb-multimodal-create.html |
| Prerequisites for multimodal knowledge bases | https://docs.aws.amazon.com/bedrock/latest/userguide/kb-multimodal-prerequisites.html |
