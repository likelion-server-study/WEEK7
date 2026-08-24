# WEEK7 — Kubernetes로 "여러 컨테이너를 손으로 관리하지 않기"

2주차에 우리는 컨테이너 하나를 `docker run` 한 줄로 띄웠다. 4주차엔 Nginx를 앞에 세웠고, 5주차엔 app을 두 개(app1·app2)로 늘려 Nginx가 나눠 보내게 했고, 6주차엔 그 위에 Prometheus·Grafana로 관측을 붙였다. 여기까지는 전부 `docker-compose.yml` 파일 하나로 **사람이 구조를 다 적어서** 관리했다.

7주차는 그 반대 방향의 도구를 만난다. Kubernetes(K8s)는 "몇 개를 띄울지, 어떻게 연결할지"를 사람이 매번 손으로 확인하는 대신, **선언한 상태를 클러스터가 스스로 유지**하게 만드는 도구다. 컨테이너가 죽으면 누가 `docker ps`로 확인하고 다시 켜는 게 아니라, 클러스터가 알아서 다시 켠다. 이번 주 핵심 질문은 하나다 — **"컨테이너 여러 개를, 사람이 계속 지켜보지 않아도 스스로 굴러가게 하려면 어떻게 해야 하는가?"**

이 문서는 발표를 따라오지 않고 **혼자 읽어도 이해되도록** 썼다. 개념 → 왜 필요한가 → 실습에서 어떻게 확인하는가 순서로 이어진다.

## 문서

| 문서 | 내용 |
| --- | --- |
| [01-환경세팅.md](01-환경세팅.md) | 모임 전에 각자 끝내 두는 준비. Docker Desktop Kubernetes 활성화, kubectl 확인, 6주차 이미지 재사용 확인 |
| [02-이론.md](02-이론.md) | VM → 컨테이너 → 오케스트레이션으로 이어지는 흐름과, Pod·Deployment·Service·ConfigMap/Secret 등 핵심 개념 |
| [03-실습.md](03-실습.md) | PetClinic + PostgreSQL을 쿠버네티스 위에 올리고, 자가치유·스케일링·무중단 재배포를 직접 확인하는 실습 |
| [04-트러블슈팅.md](04-트러블슈팅.md) | 막혔을 때 — Pending, ImagePullBackOff, CrashLoopBackOff 등 |

## 설정 파일

`files/k8s/` 폴더에 이번 주 실습에서 그대로 적용할 매니페스트 4개가 들어 있다.

```
files/
└── k8s/
    ├── 01-configmap.yaml    비밀이 아닌 설정값 (SPRING_PROFILES_ACTIVE 등)
    ├── 02-secret.yaml       DB 계정·비밀번호
    ├── 03-postgres.yaml     PostgreSQL Deployment + Service
    └── 04-petclinic.yaml    PetClinic Deployment(replicas: 2) + Service
```

`files/` 폴더를 통째로 내려받은 뒤, [03-실습.md](03-실습.md) 1절에서 다음처럼 경로를 잡는다.

```bash
WEEK7_FILES=~/Downloads/WEEK7-main/files
```

> **참고**: `petclinic` 프로젝트에는 이미 `k8s/db.yml`, `k8s/petclinic.yml`이라는 공식 예제가 들어 있다(Spring PetClinic 오픈소스 저장소가 기본 제공하는 파일). 다만 이 파일들은 Service Binding이라는 좀 더 고급 패턴을 쓰고 있어서, 우리 프로젝트에는 아직 없는 라이브러리(`spring-cloud-bindings`)가 있어야 실제로 동작한다. 이번 주 실습은 우리가 3주차부터 써온 `POSTGRES_URL`/`POSTGRES_USER`/`POSTGRES_PASS` 환경변수 방식 그대로 쿠버네티스로 옮기는, 더 쉬운 버전이다. 여유가 되면 공식 예제도 열어보고 02-이론.md 5장의 Service Binding 설명과 비교해 보면 좋다.

## 이번 주차에 사용하는 도구

| 도구 | 역할 |
| --- | --- |
| Docker Desktop Kubernetes | 로컬 컴퓨터 안에 단일 노드 쿠버네티스 클러스터를 띄운다 |
| kubectl | 클러스터에 명령을 내리는 CLI (Docker의 `docker` 명령과 같은 위치) |
| Deployment | Pod를 몇 개 띄울지 선언하고, 죽으면 다시 만들고, 무중단으로 교체한다 |
| Service | 여러 Pod 앞에서 로드밸런싱하고, 클러스터 내부 DNS 이름을 만들어 준다 |
| ConfigMap / Secret | 컨테이너 밖에서 설정값·비밀번호를 주입한다 |

## 이번 주에 꼭 가져갈 것

- 컨테이너 오케스트레이션은 "컨테이너를 하나하나 관리"하는 게 아니라 "원하는 상태를 선언하면 클러스터가 그 상태를 유지"하는 방식이다.
- Pod는 쿠버네티스의 최소 실행 단위, Deployment는 Pod를 몇 개·어떻게 유지할지에 대한 선언이다.
- Service는 5주차의 Nginx가 하던 로드밸런싱 + 4주차부터 써온 Compose 내부 DNS, 이 두 가지를 함께 대신한다.
- `docker run --restart` 한 컨테이너가 죽으면 다시 켜지는 것처럼, 쿠버네티스에서는 Pod를 지워도 Deployment가 즉시 새 Pod를 만든다(자가치유).
- `--memory`/`--cpus`가 하던 자원 제한은 `resources.requests`/`resources.limits`로 옮겨간다.
- ConfigMap은 공개돼도 되는 설정, Secret은 민감한 값 — 단, Secret도 암호화가 아니라 base64 인코딩일 뿐이다.

## 참고 자료

- [Kubernetes 공식 문서 — Concepts](https://kubernetes.io/docs/concepts/)
- [Docker Desktop — Kubernetes](https://docs.docker.com/desktop/kubernetes/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- Spring PetClinic 공식 저장소 `k8s/` 폴더 — Service Binding 방식 예제
- 6주차 자료: `likelion-server-study/WEEK6` — Prometheus·Grafana 관측 구조 (이번 주 Service/Deployment와 비교하며 보면 좋음)
