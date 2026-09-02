# oci-arm-grab

![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-workflow-2088FF?logo=githubactions&logoColor=white)
![OCI](https://img.shields.io/badge/Oracle%20Cloud-Always%20Free%20ARM-F80000?logo=oracle&logoColor=white)
![Shell](https://img.shields.io/badge/Shell-bash%20%2B%20oci--cli-4EAA25?logo=gnubash&logoColor=white)
![Notify](https://img.shields.io/badge/알림-Telegram-26A5E4?logo=telegram&logoColor=white)

> Oracle Cloud 무료(Always Free) ARM 인스턴스의 "Out of capacity" 를 뚫을 때까지 **GitHub Actions가 대신 재시도**해 주고, 성공하면 텔레그램으로 알려주는 워크플로.

## 무엇을 해결하나

OCI 의 무료 ARM(Ampere A1) 인스턴스는 인기가 많아 생성 시도가 대부분 "용량 부족"으로 실패합니다.
성공할 때까지 콘솔에서 사람이 반복 클릭하는 대신, 이 저장소는 **GitHub Actions 워크플로가 스스로를 다시 실행하며** OCI Resource Manager 스택 적용(apply)을 계속 시도합니다. 내 PC 를 켜 둘 필요도, 서버도 필요 없습니다.

## 동작 방식

1. `workflow_dispatch` 로 워크플로를 한 번 실행하면 `oci-cli` 를 설치하고, 미리 만들어 둔 **Resource Manager 스택**에 apply job 을 생성합니다.
2. job 상태를 15초 간격으로 폴링해 `SUCCEEDED` / `FAILED` 를 판정합니다 (최대 40회, 약 10분).
3. **성공** → 텔레그램으로 소리 나는 알림 전송 후 종료.
4. **실패(자리 없음)** → 무음 텔레그램 로그를 남기고 120초 뒤 `gh workflow run` 으로 **자기 자신을 다시 트리거** → 성공할 때까지 반복.

인스턴스 스펙(모양·OCPU·이미지 등)은 워크플로가 아니라 OCI 콘솔의 Resource Manager 스택에 정의해 둡니다. 이 저장소는 "그 스택을 될 때까지 apply 하는 재시도 루프"만 담당합니다.

## 사용 방법

1. OCI 콘솔에서 원하는 ARM 인스턴스 구성을 **Resource Manager 스택**으로 만들어 둡니다 (스택 OCID 확보).
2. 이 저장소를 포크(또는 복제)하고, 저장소 **Settings → Secrets and variables → Actions** 에 아래 시크릿을 등록합니다.

   | Secret | 내용 |
   |---|---|
   | `OCI_CLI_USER` | OCI 사용자 OCID |
   | `OCI_CLI_TENANCY` | 테넌시 OCID |
   | `OCI_CLI_FINGERPRINT` | API 키 지문 |
   | `OCI_CLI_KEY_CONTENT` | API 개인키(PEM) 내용 |
   | `OCI_CLI_REGION` | 리전 (예: `ap-chuncheon-1`) |
   | `STACK_ID` | Resource Manager 스택 OCID |
   | `TG_TOKEN` | 텔레그램 봇 토큰 |
   | `TG_CHAT` | 텔레그램 chat id |

3. 워크플로가 자기 자신을 재실행할 수 있도록 **Settings → Actions → General → Workflow permissions** 를 *Read and write* 로 설정합니다.
4. **Actions 탭 → oci-arm-grab → Run workflow** 로 시작. 이후는 성공 알림이 올 때까지 자동으로 돌아갑니다.
5. 성공 알림을 받으면 OCI 콘솔 **Compute → Instances** 에서 인스턴스를 확인하고, 더 이상 돌지 않게 워크플로를 중지/비활성화하면 됩니다.


## 처음부터 따라하기 (상세 가이드)

### 1) OCI API 키 만들기 — 시크릿 5종 확보

1. OCI 콘솔 오른쪽 위 프로필 → **My profile → API keys → Add API key**
2. **Generate API key pair** 선택 → **Download private key** (PEM 파일 저장) → Add
3. 생성 직후 나오는 **Configuration file preview** 에 필요한 값이 다 있습니다:
   - `user=ocid1.user.oc1..…` → `OCI_CLI_USER`
   - `tenancy=ocid1.tenancy.oc1..…` → `OCI_CLI_TENANCY`
   - `fingerprint=aa:bb:…` → `OCI_CLI_FINGERPRINT`
   - `region=ap-chuncheon-1` → `OCI_CLI_REGION`
4. 내려받은 PEM 파일을 메모장으로 열어 `-----BEGIN PRIVATE KEY-----` 부터 끝까지 통째로 복사 → `OCI_CLI_KEY_CONTENT`

### 2) Resource Manager 스택 만들기 — `STACK_ID` 확보

1. 콘솔 **Compute → Instances → Create instance** 에서 원하는 구성을 잡습니다:
   - Shape: **Ampere / VM.Standard.A1.Flex** (합계 4 OCPU · 24GB 까지 무료)
   - Image: Ubuntu / Oracle Linux 등, **SSH 공개키 등록 필수**
2. 화면 하단 **Save as stack** 을 누르면 인스턴스를 만드는 대신 Resource Manager 스택으로 저장됩니다.
3. **Developer Services → Resource Manager → Stacks** 에서 방금 만든 스택 클릭 → **OCID 복사** → `STACK_ID`
4. (검증) 스택 화면에서 **Plan** 을 한 번 돌려 오류가 없는지 확인해 두면 좋습니다.

### 3) 텔레그램 봇 — `TG_TOKEN` / `TG_CHAT`

1. 텔레그램에서 **@BotFather** → `/newbot` → 봇 이름 지정 → 발급된 토큰이 `TG_TOKEN`
2. 만든 봇에게 아무 메시지나 하나 보낸 뒤, 브라우저에서
   `https://api.telegram.org/bot<토큰>/getUpdates` 를 열면 `"chat":{"id":123456789…}` 가 보입니다 → `TG_CHAT`

### 4) 저장소 설정과 실행

1. 이 저장소를 **Fork** (또는 Use this template)
2. **Settings → Secrets and variables → Actions → New repository secret** 으로 위 8개 시크릿 등록
3. **Settings → Actions → General → Workflow permissions** → **Read and write permissions** 체크 후 저장
4. **Actions 탭 → grab → Run workflow** — 이후는 자동. 성공하면 소리 나는 텔레그램 알림이 옵니다.
5. 성공 후: **Actions → grab → ⋯ → Disable workflow** 로 루프를 꼭 멈추세요.

### 문제 해결

| 증상 | 확인할 것 |
|---|---|
| `NotAuthenticated` / 401 | API 키 5종 값 재확인 — 특히 `OCI_CLI_KEY_CONTENT` 에 BEGIN/END 줄 포함 여부, 지문 일치 여부 |
| `NotAuthorizedOrNotFound` | `STACK_ID` 오타이거나 리전이 다른 경우 (스택은 리전 종속) |
| apply 가 곧바로 FAILED | 스택 자체 문제 — Resource Manager 에서 수동 Plan/Apply 로 로그 확인 (SSH 키 누락, 서브넷 삭제 등) |
| `Out of capacity` 반복 | 정상입니다 — 그걸 뚫으려고 도는 중. 수일 걸릴 수 있고, 새벽 시간대 성공률이 높은 편 |
| 워크플로가 1회만 돌고 멈춤 | Workflow permissions 가 Read and write 인지, `gh workflow run` 단계 로그 확인 |
| 텔레그램 알림이 안 옴 | 봇에게 먼저 말을 걸었는지, `TG_CHAT` 이 숫자 id 인지 확인 |

## 기술적 특징

서버·크론 없이 **GitHub Actions 만으로 무한 재시도 루프**를 구성한 것이 핵심입니다. Actions 의 job 시간 제한을 피하기 위해 한 run 안에서 오래 기다리는 대신, run 마지막에 `gh workflow run` 으로 다음 run 을 예약하는 **자기 재귀 트리거** 패턴을 사용합니다. 모든 자격 증명(OCI API 키, 텔레그램 토큰)은 GitHub Secrets 로만 주입되어 코드·로그에 남지 않고, 알림은 성공 시에만 소리가 나도록(실패 로그는 `disable_notification`) 구분해 수백 번의 재시도에도 피로하지 않게 설계했습니다.

## 주의

- Always Free 한도 내 구성이라도 생성 성공까지 수일이 걸릴 수 있습니다.
- 성공 후에는 반드시 루프를 중지하세요 (이미 만들어진 스택에 apply 를 반복할 필요가 없습니다).
- Public 저장소의 Actions 사용량 정책과 OCI 이용약관 범위 안에서 사용하세요.
