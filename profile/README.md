## Gen-View 서비스 구조

``` mermaid
graph TB
    subgraph Frontend["프론트엔드 서비스"]
        F[gen-view-front<br>Svelte 4.2.8 + Vite 5.0.10<br>포트: 3001]
        S[gen-view-streamlit<br>Python + Streamlit 1.43.0<br>포트: 8120]
        SO[gen-view-streamlit-okx<br>OKX 전용 대시보드<br>포트: 8121]
    end

    subgraph Backend["백엔드 서비스"]
        subgraph API["gen-view-api"]
            API_Core[FastAPI 0.115.11<br>Python 3.12<br>포트: 8100]
            API_Worker[Worker 서비스<br>RQ 작업 처리<br>레플리카: 10개]
            API_Components[커스텀 컴포넌트<br>시뮬레이터/팩터/백테스트]
        end
    end

    subgraph DataPipeline["데이터 파이프라인"]
        AF_W[Airflow Webserver<br>v2.10.5<br>포트: 8280]
        AF_S[Airflow Scheduler]
        AF_P[Airflow PostgreSQL<br>메타데이터 저장소]
        AF_I[Airflow Init]
    end

    subgraph Storage["데이터 스토리지"]
        CH[ClickHouse<br>시계열 데이터<br>v25.2.1.3085<br>HTTP: 8123<br>Native: 9003]
        PG[PostgreSQL<br>메타데이터/권한관리<br>포트: 5433]
        MG[MongoDB<br>포트폴리오/전략 저장<br>포트: 27018]
        RD[Redis Alpine<br>캐싱 및 작업 큐<br>포트: 6379]
        ME[Mongo Express<br>MongoDB UI<br>포트: 8122]
        RC[Redis Commander<br>Redis UI<br>포트: 8281]
    end

    subgraph Security["보안"]
        V[HashiCorp Vault<br>시크릿 관리<br>v1.18.2<br>포트: 8201]
    end

    %% 프론트엔드 연결
    F --> API_Core
    S --> API_Core
    SO --> API_Core
    
    %% API 내부 연결
    API_Core --> API_Worker
    API_Core --> API_Components
    API_Worker --> RD
    
    %% Airflow 연결
    AF_W --> API_Core
    AF_W --> CH
    AF_S --> AF_P
    AF_I --> AF_P
    
    %% 데이터 저장소 연결
    API_Core --> V
    API_Core --> CH
    API_Core --> PG
    API_Core --> MG
    API_Core --> RD
    
    %% UI 도구 연결
    ME --> MG
    RC --> RD
    
    %% Vault 연결
    AF_W --> V
    AF_S --> V

    classDef frontendStyle fill:#f9f7e8,stroke:#333,stroke-width:1px
    classDef backendStyle fill:#e3f2fd,stroke:#333,stroke-width:1px
    classDef pipelineStyle fill:#e8f5e9,stroke:#333,stroke-width:1px
    classDef storageStyle fill:#fff3e0,stroke:#333,stroke-width:1px
    classDef securityStyle fill:#fce4ec,stroke:#333,stroke-width:1px
    
    class F,S,SO frontendStyle
    class API_Core,API_Worker,API_Components backendStyle
    class AF_W,AF_S,AF_P,AF_I pipelineStyle
    class CH,PG,MG,RD,ME,RC storageStyle
    class V securityStyle
```

## Gen-View 네트워크 및 볼륨 구성

``` mermaid
graph TB
    subgraph Networks["네트워크 구성"]
        GVN[gen-view-network<br>공용 네트워크]
        CHN[clickhouse-network]
        VN[vault-network]
        FN[frontend-network]
        STN[streamlit-network]
        STOKXN[streamlit-okx-network]
        AFN[airflow-network]
        PGN[postgresql-network]
        RDN[redis-network]
        MGN[mongodb-network]
    end

    subgraph Volumes["볼륨 구성"]
        CH_VOL[clickhouse-data<br>중요도: critical]
        V_VOL[vault-data<br>중요도: critical]
        PG_VOL[postgresql-data<br>중요도: critical]
        AF_PG_VOL[airflow-postgres-data<br>중요도: critical]
        RD_VOL[redis-data<br>중요도: medium]
        MG_VOL[mongodb-data<br>중요도: critical]
    end

    subgraph Services["주요 서비스"]
        API[API<br>gen-view-api]
        WORKER[Worker<br>10개 레플리카]
        FRONT[Frontend<br>gen-view-front]
        ST[Streamlit]
        ST_OKX[Streamlit OKX]
        CH_SVC[ClickHouse]
        MG_SVC[MongoDB]
        RD_SVC[Redis]
        V_SVC[Vault]
        AF_SVC[Airflow]
    end

    %% 서비스와 네트워크 연결
    API --> GVN
    API --> CHN
    API --> VN
    API --> FN
    API --> STN
    API --> RDN

    WORKER --> GVN
    WORKER --> CHN
    WORKER --> VN
    WORKER --> RDN

    FRONT --> FN

    ST --> STN

    ST_OKX --> GVN
    ST_OKX --> STOKXN

    CH_SVC --> GVN
    CH_SVC --> CHN

    MG_SVC --> GVN
    MG_SVC --> MGN

    RD_SVC --> GVN
    RD_SVC --> RDN

    V_SVC --> VN

    AF_SVC --> GVN
    AF_SVC --> AFN

    %% 서비스와 볼륨 연결
    CH_SVC --> CH_VOL
    V_SVC --> V_VOL
    MG_SVC --> MG_VOL
    RD_SVC --> RD_VOL
    AF_SVC --> AF_PG_VOL

    classDef networkStyle fill:#e1f5fe,stroke:#333,stroke-width:1px
    classDef volumeStyle fill:#f3e5f5,stroke:#333,stroke-width:1px
    classDef serviceStyle fill:#e8f5e9,stroke:#333,stroke-width:1px

    class GVN,CHN,VN,FN,STN,STOKXN,AFN,PGN,RDN,MGN networkStyle
    class CH_VOL,V_VOL,PG_VOL,AF_PG_VOL,RD_VOL,MG_VOL volumeStyle
    class API,WORKER,FRONT,ST,ST_OKX,CH_SVC,MG_SVC,RD_SVC,V_SVC,AF_SVC serviceStyle
```

## 서비스별 주요 기능 및 설명

``` mermaid
graph TB
    %% 서비스별 주요 기능
    subgraph gen-view-front["gen-view-front (프론트엔드)"]
        GVF_1[Svelte 4.2.8 기반<br>데이터 시각화 UI]
        GVF_2[Bootstrap 5.3.2<br>반응형 웹 디자인]
        GVF_3[Vite 5.0.10 기반<br>빠른 개발 환경]
        GVF_4[Chart.js 4.4.1<br>인터랙티브 차트]
    end
    
    subgraph gen-view-api["gen-view-api (백엔드 API)"]
        GVA_1[FastAPI 0.115.11<br>RESTful API]
        GVA_2[Python 3.12<br>고성능 백엔드]
        GVA_3[데이터 소스 통합<br>ClickHouse/MongoDB/PG]
        GVA_4[Redis Queue 기반<br>비동기 작업 처리]
    end
    
    subgraph gen-view-streamlit["gen-view-streamlit (데이터 대시보드)"]
        GVS_1[Streamlit 1.43.0<br>대화형 대시보드]
        GVS_2[Plotly 6.0.0<br>고급 시각화]
        GVS_3[Pandas 2.2.3<br>데이터 처리]
        GVS_4[Altair 5.5.0<br>선언적 시각화]
    end
    
    subgraph gen-view-streamlit-okx["gen-view-streamlit-okx (OKX 리서치)"]
        GVSO_1[OKX 암호화폐<br>시장 데이터 분석]
        GVSO_2[백테스트 결과<br>시각화 및 분석]
        GVSO_3[포트폴리오 성능<br>모니터링]
        GVSO_4[팩터 분석<br>및 리서치 도구]
    end
    
    subgraph gen-view-airflow["gen-view-airflow (데이터 파이프라인)"]
        GVAir_1[Airflow 2.10.5<br>워크플로우 오케스트레이션]
        GVAir_2[ETL 파이프라인<br>자동화]
        GVAir_3[데이터 수집<br>스케줄링]
        GVAir_4[ClickHouse<br>모니터링 API]
    end
    
    subgraph gen-view-clickhouse["gen-view-clickhouse (시계열 데이터)"]
        GVCH_1[ClickHouse 25.2.1<br>컬럼형 DBMS]
        GVCH_2[리전별 데이터 구성<br>KRX/US/OKX/Coinone]
        GVCH_3[고성능 분석<br>쿼리 처리]
        GVCH_4[시계열 데이터<br>연도별 파티셔닝]
    end
    
    subgraph gen-view-postgresql["gen-view-postgresql (관계형 데이터)"]
        GVPG_1[PostgreSQL<br>관계형 데이터베이스]
        GVPG_2[스키마 분리<br>auth/logs/stats]
        GVPG_3[API 키 관리<br>및 인증]
        GVPG_4[Vault 통합<br>보안 시크릿 관리]
    end
    
    subgraph gen-view-mongodb["gen-view-mongodb (NoSQL 데이터)"]
        GVMG_1[MongoDB<br>도큐먼트 데이터베이스]
        GVMG_2[전략 정보<br>유연한 저장]
        GVMG_3[백테스트 결과<br>히스토리 관리]
        GVMG_4[포트폴리오 구성<br>데이터 저장]
    end
    
    subgraph gen-view-redis["gen-view-redis (인메모리 데이터)"]
        GVR_1[Redis Alpine<br>인메모리 데이터베이스]
        GVR_2[Redis Queue<br>작업 대기열 관리]
        GVR_3[고성능 캐싱<br>및 세션 관리]
        GVR_4[Redis Commander<br>웹 UI 제공]
    end
    
    subgraph gen-view-vault["gen-view-vault (시크릿 관리)"]
        GVV_1[HashiCorp Vault 1.18.2<br>시크릿 관리]
        GVV_2[KV v2 시크릿 엔진<br>민감 정보 저장]
        GVV_3[접근 정책<br>및 권한 관리]
        GVV_4[AppRole 인증<br>서비스 간 보안 연결]
    end
    
    subgraph gen-view-init["gen-view-init (통합 환경)"]
        GVI_1[Docker Compose<br>멀티 서비스 오케스트레이션]
        GVI_2[네트워크 분리<br>보안 강화]
        GVI_3[볼륨 관리<br>데이터 영속성]
        GVI_4[환경 변수<br>중앙 관리]
    end
    
    %% 서비스별 스타일 지정
    classDef frontendStyle fill:#f9f7e8,stroke:#333,stroke-width:1px
    classDef backendStyle fill:#e3f2fd,stroke:#333,stroke-width:1px
    classDef streamlitStyle fill:#e1bee7,stroke:#333,stroke-width:1px
    classDef airflowStyle fill:#e8f5e9,stroke:#333,stroke-width:1px
    classDef clickhouseStyle fill:#ffecb3,stroke:#333,stroke-width:1px
    classDef postgresStyle fill:#bbdefb,stroke:#333,stroke-width:1px
    classDef mongoStyle fill:#b2dfdb,stroke:#333,stroke-width:1px
    classDef redisStyle fill:#ffccbc,stroke:#333,stroke-width:1px
    classDef vaultStyle fill:#fce4ec,stroke:#333,stroke-width:1px
    classDef initStyle fill:#d7ccc8,stroke:#333,stroke-width:1px
    
    class GVF_1,GVF_2,GVF_3,GVF_4 frontendStyle
    class GVA_1,GVA_2,GVA_3,GVA_4 backendStyle
    class GVS_1,GVS_2,GVS_3,GVS_4,GVSO_1,GVSO_2,GVSO_3,GVSO_4 streamlitStyle
    class GVAir_1,GVAir_2,GVAir_3,GVAir_4 airflowStyle
    class GVCH_1,GVCH_2,GVCH_3,GVCH_4 clickhouseStyle
    class GVPG_1,GVPG_2,GVPG_3,GVPG_4 postgresStyle
    class GVMG_1,GVMG_2,GVMG_3,GVMG_4 mongoStyle
    class GVR_1,GVR_2,GVR_3,GVR_4 redisStyle
    class GVV_1,GVV_2,GVV_3,GVV_4 vaultStyle
    class GVI_1,GVI_2,GVI_3,GVI_4 initStyle
```

## Gen-View API 내부 구조

``` mermaid
graph TB
    subgraph API["gen-view-api (FastAPI 0.115.11 + Python 3.12)"]
        subgraph Core["app/core"]
            CONFIG[config.py<br>환경설정 관리]
            VAULT[vault.py<br>Vault 시크릿 통합]
            
            subgraph DB["db"]
                CH_DB[clickhouse.py<br>ClickHouse 연결<br>clickhouse-driver 0.2.9]
                PG_DB[postgresql.py<br>PostgreSQL 연결<br>sqlalchemy 2.0.38]
                MG_DB[mongodb.py<br>MongoDB 연결]
                RD_DB[redis.py<br>Redis 연결]
            end
        end

        subgraph Components["components"]
            SIM[Simulator<br>시뮬레이션 엔진]
            SR[Simulator Result<br>결과 데이터 처리]
            FQ[Factor Quantile<br>팩터 분위수 분석]
            PC[Parser Cache<br>파싱 결과 캐싱]
            FE[Factor Exposure<br>팩터 노출도 분석]
            CE[Condition Exposure<br>조건 노출도 분석]
            CEV[Condition Evaluation<br>조건 평가 엔진]
            CC[Condition Crawler<br>조건 데이터 수집]
            BT[Back Test<br>백테스트 엔진]
        end

        subgraph App["app"]
            MAIN[main.py<br>애플리케이션 엔트리포인트]
            
            subgraph API_Routes["api"]
                API_V1[api_v1/api.py<br>API 라우터 설정]
                
                subgraph Endpoints["endpoints"]
                    HEALTH[health.py<br>헬스 체크 API]
                    DATA_CLICK[data_click.py<br>ClickHouse 데이터 API]
                    DATA_PG[data_pg.py<br>PostgreSQL 데이터 API]
                    DATA_MG[data_mongo.py<br>MongoDB 데이터 API]
                    WS[websocket.py<br>웹소켓 API]
                    DUMMY[dummy_api.py<br>테스트용 API]
                    RQ[rq_dashboard.py<br>RQ 대시보드 API]
                    
                    subgraph VIEW_OKX["view_okx"]
                        PORTFOLIO[portfolio_backtest_routes.py<br>포트폴리오 백테스트 API]
                        FACTOR_EXP[factor_exposure_routes.py<br>팩터 노출도 API]
                        EXPR[expression_routes.py<br>표현식 평가 API]
                        COND_SHAP[condition_shapley_routes.py<br>조건 샤플리 값 분석]
                        COND_EXP[condition_exposure_routes.py<br>조건 노출도 API]
                        SIMPLE_BT[simple_backtest_routes.py<br>단순 백테스트 API]
                        FACTOR_Q[factor_quantile_routes.py<br>팩터 분위수 API]
                        JOB[job_routes.py<br>작업 관리 API]
                        DATA[data_routes.py<br>데이터 조회 API]
                    end
                end
            end
            
            subgraph Service["service"]
                subgraph Worker["worker"]
                    WORKER[Worker Services<br>Redis Queue<br>비동기 작업 처리]
                end
                
                subgraph Engine["engines"]
                    SIM_ENGINE[engin_simulator<br>시뮬레이션 엔진<br>Numpy/Pandas 기반]
                    VECTOR_ENGINE[engin_vector<br>벡터 처리 엔진<br>고성능 연산]
                end
            end
            
            subgraph Models["models"]
                DATA_MODEL[data.py<br>공통 데이터 모델<br>Pydantic 2.10.6]
                CH_MODEL[clickhouse.py<br>ClickHouse 모델]
            end
        end

        subgraph External["external"]
            RD[(Redis<br>작업 큐/캐싱<br>포트: 6379)]
            CH[(ClickHouse<br>시계열 데이터<br>HTTP: 8123<br>Native: 9003)]
            PG[(PostgreSQL<br>메타데이터<br>포트: 5433)]
            MG[(MongoDB<br>전략 데이터<br>포트: 27018)]
            VT[Vault<br>시크릿 관리<br>포트: 8201]
        end
    end

    %% Core connections
    CONFIG --> DB
    CONFIG --> VAULT
    VAULT --> VT
    
    %% Database connections
    CH_DB --> CH
    PG_DB --> PG
    MG_DB --> MG
    RD_DB --> RD

    %% API flow
    Client --> MAIN
    MAIN --> API_V1
    API_V1 --> Endpoints
    
    %% Endpoint to service connections
    Endpoints --> Service
    VIEW_OKX --> Engine
    VIEW_OKX --> Worker
    
    %% Service connections
    Worker --> RD
    Engine --> Components
    
    %% Model usage
    Service --> Models
    Endpoints --> Models
    
    %% Component DB connections
    SIM --> CH
    BT --> CH
    FE --> PG
    CE --> MG
    PC --> RD
    
style API fill:#f5f5f5,stroke:#333,stroke-width:2px
style Core fill:#e1f5fe,stroke:#333,stroke-width:1px
style Components fill:#f3e5f5,stroke:#333,stroke-width:1px
style App fill:#e8f5e9,stroke:#333,stroke-width:1px
style External fill:#fff3e0,stroke:#333,stroke-width:1px
style VIEW_OKX fill:#ffecb3,stroke:#333,stroke-width:1px
style DB fill:#bbdefb,stroke:#333,stroke-width:1px
style Engine fill:#d1c4e9,stroke:#333,stroke-width:1px
style Worker fill:#c8e6c9,stroke:#333,stroke-width:1px
``` 

## 서비스 접속 정보

| 서비스 | URL | 설명 |
|-------|-----|------|
| API | http://localhost:8100 | FastAPI 백엔드 서비스 |
| 프론트엔드 | http://localhost:3001 | Svelte 프론트엔드 |
| Streamlit | http://localhost:8120 | 데이터 시각화 대시보드 |
| Streamlit OKX | http://localhost:8121 | OKX 리서치 대시보드 |
| Airflow | http://localhost:8280 | 데이터 파이프라인 관리 |
| ClickHouse HTTP | http://localhost:8123 | ClickHouse HTTP 인터페이스 |
| Vault | http://localhost:8201 | 시크릿 관리 서비스 |
| MongoDB Express | http://localhost:8122 | MongoDB 웹 인터페이스 |
| Redis Commander | http://localhost:8281 | Redis 웹 인터페이스 |
