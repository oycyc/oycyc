## ML workflow architecture

```mermaid
%%{init: {"layout": "dagre", "look": "classic", "fontFamily": "Helvetica, Arial, sans-serif", "flowchart": {"curve": "stepAfter", "nodeSpacing": 30, "rankSpacing": 40, "padding": 14}}}%%
flowchart TB
    %% End-to-end ML workflow in two rows: Build (prepare data, train with SageMaker Pipelines)
    %% above Run (release, serve predictions, monitor for drift and retrain).

    %% Actors outside AWS
    sources@{ shape: cloud, label: "Data sources<br/>apps, DBs, logs" }
    scientists@{ shape: person, label: "Data scientists" }
    apps@{ shape: browser, label: "Client apps" }

    subgraph cloud["AWS Cloud"]
        subgraph region["Region · us-east-1"]
            subgraph build["Build · data to model"]
                direction LR
            subgraph prep["1 · Data preparation"]
                direction TB
                raw@{ shape: bucket, label: "S3<br/>raw data" }
                glue[["Glue<br/>ETL jobs"]]
                fs[("SageMaker<br/>Feature Store")]
            end

            subgraph train["2 · Training · SageMaker Pipelines"]
                direction TB
                studio("SageMaker Studio<br/>pipeline authoring")
                proc[["Processing<br/>job"]]
                trn[["Training<br/>job"]]
                evalstep[["Evaluate<br/>metrics"]]
            end

            end

            subgraph run["Run · release, serve, monitor"]
                direction LR
            subgraph deploy["3 · Release"]
                direction TB
                registry[("SageMaker<br/>Model Registry")]
                cp@{ shape: procs, label: "CodePipeline<br/>approve, deploy" }
            end

            subgraph serve["4 · Inference"]
                direction TB
                apigw{{"API Gateway<br/>REST API"}}
                fnInfer>"Lambda<br/>inference handler"]
                ep["SageMaker endpoint<br/>real-time"]
            end

            subgraph monitor["5 · Monitoring"]
                direction TB
                mm("Model Monitor<br/>drift checks")
                cw("CloudWatch<br/>metrics, alarms")
                bus[/"EventBridge<br/>retrain trigger"/]
            end
            end
        end
    end

    %% Edges. External edges point at boxes, not nodes inside them, so each box is laid out
    %% on its own: the stages run left to right inside each row, rows stack top to bottom.
    %% The retraining loop (EventBridge starts a new pipeline run) is left as a label rather
    %% than a line back up the diagram, which would cross every stage.
    sources --> build
    scientists -.-> build
    prep e1@--> train
    build -->|register model| run
    apps ==>|HTTPS 443| run
    deploy e2@--> serve
    serve e3@--> monitor
    %% Straight horizontal arrows between stages: the default stepAfter curve tilts their arrowheads
    e1@{ curve: linear }
    e2@{ curve: linear }
    e3@{ curve: linear }

    raw -->|S3 event| glue
    glue -->|features| fs
    studio --> proc
    proc --> trn
    trn --> evalstep
    registry -->|approved| cp
    apigw ==>|invoke| fnInfer
    fnInfer ==>|InvokeEndpoint| ep
    mm --> cw
    cw -->|alarm| bus

    %% Node colors: AWS Architecture Icons category palette, plus ml for SageMaker
    classDef ext fill:#232F3E,stroke:#5A6B7F,color:#fff
    classDef net fill:#8C4FFF,stroke:#6A3BC2,color:#fff
    classDef compute fill:#ED7100,stroke:#B35500,color:#fff
    classDef db fill:#C925D1,stroke:#951B9B,color:#fff
    classDef storage fill:#7AA116,stroke:#5B7A10,color:#fff
    classDef integration fill:#E7157B,stroke:#B0105D,color:#fff
    classDef cicd fill:#3B48CC,stroke:#2A3399,color:#fff
    classDef ml fill:#01A88D,stroke:#017A66,color:#fff
    class sources,scientists,apps ext
    class apigw,glue net
    class fnInfer compute
    class fs,registry db
    class raw storage
    class mm,cw,bus integration
    class cp cicd
    class studio,proc,trn,evalstep,ep ml

    %% Box colors
    style cloud fill:none,stroke:#7D8998,stroke-width:2px,color:#7D8998
    style region fill:none,stroke:#00A4A6,stroke-width:2px,stroke-dasharray:6 4,color:#00A4A6
    style prep fill:none,stroke:#7D8998,stroke-dasharray:4 3,color:#7D8998
    style train fill:none,stroke:#01A88D,stroke-dasharray:4 3,color:#01A88D
    style deploy fill:none,stroke:#7D8998,stroke-dasharray:4 3,color:#7D8998
    style serve fill:none,stroke:#7D8998,stroke-dasharray:4 3,color:#7D8998
    style monitor fill:none,stroke:#7D8998,stroke-dasharray:4 3,color:#7D8998
    style build fill:none,stroke:#7D8998,stroke-dasharray:2 2,color:#7D8998
    style run fill:none,stroke:#7D8998,stroke-dasharray:2 2,color:#7D8998

    %% Edge colors: indexes come from the edge list printed by scripts/check_diagram.py
    linkStyle 4,13,14 stroke:#8C4FFF,stroke-width:2.5px
```

The workflow runs in two rows: **Build** (stages 1 and 2) turns raw data into a trained model, and **Run** (stages 3 to 5) releases, serves and monitors it.

1. **Data preparation:** source data lands in S3. Glue ETL jobs turn it into features and store them in SageMaker Feature Store.
2. **Training:** data scientists build and run a SageMaker Pipeline from SageMaker Studio (processing, training and evaluation). A model that passes evaluation is registered in the Model Registry.
3. **Release:** an approved model version in the Model Registry triggers CodePipeline to deploy it.
4. **Inference:** client apps call the real-time SageMaker endpoint through API Gateway and a Lambda handler.
5. **Monitoring:** Model Monitor checks captured requests for drift and publishes metrics to CloudWatch. A drift alarm fires an EventBridge rule that starts a new pipeline run (the retraining loop back to stage 2, shown as a label rather than a line so it doesn't cross the diagram).

**Shapes**

| Shape | Meaning |
|---|---|
| Person | People |
| Browser window | Client applications |
| Cloud | Data sources outside AWS |
| Hexagon | Request routing (API Gateway) |
| Flag | Event-driven functions (Lambda) |
| Rectangle with side bars | Multi-step jobs: Glue ETL, pipeline steps |
| Rectangle | Long-running compute (SageMaker endpoint) |
| Rounded rectangle | Managed services: Studio, Model Monitor, CloudWatch |
| Stacked cards (blue) | CI/CD pipeline |
| Bucket | S3 buckets |
| Cylinder | Stores and registries: Feature Store, Model Registry |
| Parallelogram | Event bus (EventBridge) |
| Dashed box | One stage of the workflow |
| Dotted box | A row of stages: Build or Run |

**Colors:** teal = SageMaker (machine learning) · green = storage · magenta = data stores · purple = networking and analytics · orange = compute · pink = monitoring and events · blue = CI/CD · dark = outside AWS

**Lines:** thick purple = inbound inference requests · solid = data flow and service calls, including the plain arrows from one stage to the next · dotted = people working in the system
