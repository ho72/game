# 실행 환경과 테스트 안내

현재 저장소에서 확인할 수 있는 범위를 게임 규칙 테스트, 서버 이미지 빌드, 클러스터 연결로 나누었습니다. 전체 서비스 실행에는 외부 인프라와 AI 모델 준비가 필요합니다.

이 문서의 명령은 저장소 파일과 실행 진입점을 대조해 작성했습니다. 2026년 10월 5일 문서 정리에서는 설치·테스트·이미지 빌드·클러스터 배포를 실행하지 않았습니다.

## 저장소 받기

```bash
git clone https://github.com/ho72/game.git
cd game
```

## 게임 규칙 테스트

서버·DB·RabbitMQ를 띄우지 않고 게임 규칙 테스트부터 확인할 수 있습니다. Game Server Dockerfile의 Python 기준은 3.12이며, 아래 세 테스트의 게임 구현은 표준 라이브러리와 NumPy를 사용합니다.

```bash
cd tacticai/game_server
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install pytest numpy
python -m pytest \
  tests/test_chess.py \
  tests/test_othello.py \
  tests/test_tictactoe.py
```

위 명령은 게임 규칙 테스트만 선택합니다. `test_db.py`, `test_rabbitmq.py`, `test_game_manager.py`는 서비스 설정과 외부 자원 준비 범위를 확인한 뒤 별도로 실행합니다. 세 규칙 테스트의 통과 여부도 이 문서 정리에서 새로 검증하지 않았습니다.

## 서버별 역할과 진입점

| 서비스 | 진입점 | 컨테이너 실행 방식 | 메트릭 |
| --- | --- | --- | --- |
| API Gateway | [main.py](../tacticai/mq_api_gateway/main.py) | `uvicorn main:app --host 0.0.0.0 --port 8000` | 8000의 `/metrics` · 인증 사용 |
| Game Server | [main.py](../tacticai/game_server/main.py) | `python main.py` | 8002 · Prometheus HTTP 서버 |
| AI Server | [main.py](../tacticai/ai_server/main.py) | `python main.py` | 8001 · Prometheus HTTP 서버 |

Game·AI Server는 메시지 소비와 처리 루프를 실행합니다. 두 서비스를 FastAPI 애플리케이션으로 가정해 `uvicorn`으로 시작하지 않습니다.

## 컨테이너 이미지 빌드

저장소 루트에서 각 서비스 폴더를 빌드 컨텍스트로 사용합니다.

```bash
docker build -t tacticai-gateway:local tacticai/mq_api_gateway
docker build -t tacticai-game:local tacticai/game_server
docker build -t tacticai-ai:local tacticai/ai_server
```

AI Dockerfile은 PyTorch 2.5.1·CUDA 12.4 기반 이미지를 사용합니다. 이미지 빌드 완료와 실제 GPU 추론 가능 여부는 별도로 확인해야 합니다. Git에 없는 모델 가중치·모델 배치 경로는 실행 환경에 맞춰 준비합니다.

## 전체 서비스 준비

| 준비 항목 | 연결 범위 |
| --- | --- |
| RabbitMQ | Gateway·Game·AI 서버의 메시지 전달 |
| Redis | 서비스별 집계 지표 공유와 조회 |
| MongoDB·MySQL | Game Server의 데이터 저장·조회 |
| 모델 가중치 | AI Server가 읽을 수 있는 모델 파일과 경로 |
| Kubernetes·Helm | 서버 Deployment·Service 배포 |
| GPU 노드와 NVIDIA device plugin | AI Deployment의 GPU 요청 조건 충족 |
| Prometheus Operator·Prometheus·Grafana | 모니터링 수집·표시 |

각 서비스의 설정과 환경변수를 실행 환경에 맞게 준비하고, 메시지 브로커·DB·Redis 접속 정보를 일치시킵니다. 문서에는 특정 개발 환경의 인증값을 제공하지 않습니다.

[`tacticai`](../tacticai)에 Docker Compose 파일도 있지만, 서비스 연결·모델 파일·외부 인프라 조건을 포함한 전체 스택 기동은 이번 문서 정리에서 검증하지 않았습니다.

## Helm 배포와 모니터링 연결

현재 [Helm 차트](../tacticai/helm/tacticai)는 Gateway·Game·AI 서버의 Deployment와 Service를 정의합니다. 메시지 브로커·DB·Prometheus·Grafana를 함께 설치하는 차트는 아닙니다.

레지스트리에 사용할 서버 이미지를 준비하고, 실행 환경에 맞는 로컬 values 파일에 이미지, 네임스페이스, 노드, 서비스 포트, 외부 인프라 연결 설정을 구성합니다. 아래 `local-values.yaml`은 사용자가 별도로 준비하는 파일입니다.

```bash
helm lint tacticai/helm/tacticai --values local-values.yaml
helm upgrade --install tacticai tacticai/helm/tacticai \
  --namespace tacticai --create-namespace \
  --values local-values.yaml
kubectl -n tacticai get deployments,pods,services
```

**현재 Helm 템플릿에는 ServiceMonitor에 필요한 Service 라벨과 이름 있는 포트가 없습니다.** Helm 배포 준비 과정에서 [모니터링 연결 조건](MONITORING.md#servicemonitor-연결-조건)을 반영한 뒤 `monitor.yaml`을 적용해야 합니다. 현재 차트에 values 파일만 지정하는 것으로 누락된 Service 필드가 자동 추가되지는 않습니다.

Prometheus·Grafana 설치와 Gateway 메트릭 인증 설정은 별도로 준비합니다. 서버 실행, 모니터 연결, Pod 확장 검증 순서는 [모니터링 문서](MONITORING.md)에 있습니다.

## 현재 재현 범위

- 게임 규칙과 서비스·모델 코드를 확인할 수 있습니다.
- Dockerfile과 Helm 템플릿으로 실행·배포 구조를 확인할 수 있습니다.
- 현재 서비스의 메트릭 선언과 ServiceMonitor의 선택 조건을 확인할 수 있습니다.
- 과거 Mock-up 모니터링 화면을 확인할 수 있습니다.

당시 사용한 모델 가중치, Grafana 대시보드 JSON, PodMonitor와 전체 Mock-up 환경은 포함되어 있지 않습니다. 과거 검증 결과와 현재 환경에서의 재실행 결과를 구분합니다.
