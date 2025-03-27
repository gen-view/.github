## Gen-View 서비스 구조

``` mermaid
graph TB
    subgraph Frontend["프론트엔드 서비스"]
        F[gen-view-front]
        S[gen-view-streamlit]
        SO[gen-view-streamlit-okx]
    end

    subgraph Backend["백엔드 서비스"]
        API[gen-view-api]
        W[Worker]
    end

    subgraph DataPipeline["데이터 파이프라인"]
        AF_W[Airflow Webserver]
        AF_S[Airflow Scheduler]
        AF_P[Airflow Postgres]
    end

    subgraph Storage["데이터 스토리지"]
        CH[ClickHouse]
        PG[PostgreSQL]
        MG[MongoDB]
        RD[Redis]
        ME[Mongo Express]
        RC[Redis Commander]
    end

    subgraph Security["보안"]
        V[Vault]
    end

    F --> API
    S --> API
    SO --> API
    API --> W
    API --> RD
    W --> RD
    
    AF_W --> API
    AF_W --> CH
    AF_S --> AF_P
    
    API --> V
    API --> CH
    API --> PG
    API --> MG
    
    ME --> MG
    RC --> RD

```

## Gen-View API 내부 구조

``` mermaid
graph TB
    subgraph API["gen-view-api"]
        subgraph Core["core"]
            CONFIG[Config]
            DB[Database]
            SEC[Security]
            LOG[Logger]
        end

        subgraph Components["components"]
            SIM[Simulator]
            SR[Simulator Result]
            FQ[Factor Quantile]
            PC[Parser Cache]
            FE[Factor Exposure]
            CE[Condition Exposure]
            CEV[Condition Evaluation]
            CC[Condition Crawler]
            BT[Back Test]
        end

        subgraph App["app"]
            subgraph Routes["api"]
                RT[Routers]
            end
            
            subgraph Service["service"]
                SV[Services]
            end
            
            subgraph Models["models"]
                MD[Models]
            end
        end

        subgraph External["external"]
            RD[(Redis)]
            CH[(ClickHouse)]
            PG[(PostgreSQL)]
            MG[(MongoDB)]
            VT[Vault]
        end
    end

    %% Core connections
    CONFIG --> DB
    CONFIG --> SEC
    CONFIG --> LOG

    %% Route flow
    Client --> RT
    RT --> SV
    SV --> MD
    
    %% Component connections
    SV --> SIM
    SV --> SR
    SV --> FQ
    SV --> PC
    SV --> FE
    SV --> CE
    SV --> CEV
    SV --> CC
    SV --> BT

    %% External connections
    SIM --> CH
    BT --> CH
    FE --> PG
    CE --> MG
    PC --> RD
    
    %% Security
    SEC --> VT
    
style API fill:#f5f5f5,stroke:#333,stroke-width:2px
style Core fill:#e1f5fe,stroke:#333,stroke-width:1px
style Components fill:#f3e5f5,stroke:#333,stroke-width:1px
style App fill:#e8f5e9,stroke:#333,stroke-width:1px
style External fill:#fff3e0,stroke:#333,stroke-width:1px
``` 
