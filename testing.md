# aws webapp architecture diagram

```mermaid
%%{init: {"layout": "dagre", "look": "classic", "fontFamily": "Helvetica, Arial, sans-serif", "flowchart": {"curve": "step", "nodeSpacing": 30, "rankSpacing": 45, "padding": 10}}}%%
flowchart TB
    users(["Users<br/>web and mobile"])
    gha[["GitHub Actions<br/>CI/CD pipeline"]]
    oncall(["On-call team<br/>email and Slack"])

    subgraph cloud["AWS Cloud"]
        subgraph edge["Global edge"]
            dns{{"Route 53<br/>DNS and health checks"}}
            waf[\"AWS WAF<br/>managed rules, rate limits"/]
            cdn{{"CloudFront<br/>CDN"}}
        end

        subgraph region["Region · us-east-1"]
            subgraph regional["Regional services"]
                s3web[("S3<br/>static assets")]
                s3up[("S3<br/>user uploads")]
                fn>"Lambda<br/>image processing"]
                subgraph plink["Private access via interface endpoints"]
                    sqs[/"SQS<br/>job queue"/]
                    secrets("Secrets Manager<br/>DB credentials")
                    cw("CloudWatch<br/>logs, metrics, alarms")
                    ecr[("ECR<br/>container images")]
                end
                sns[/"SNS<br/>alert topic"/]
            end

            subgraph vpc["VPC · 10.0.0.0/16"]
                igw(("Internet<br/>Gateway"))

                subgraph public["Public subnets"]
                    subgraph pubA["AZ A · 10.0.1.0/24"]
                        natA(("NAT<br/>Gateway"))
                    end
                    alb{{"Application Load Balancer<br/>HTTPS 443 · ACM cert"}}
                    subgraph pubB["AZ B · 10.0.2.0/24"]
                        natB(("NAT<br/>Gateway"))
                    end
                end

                subgraph app["Private app subnets · ECS Fargate cluster"]
                    subgraph appA["AZ A · 10.0.11.0/24"]
                        apiA["API service<br/>REST"]
                        wrkA[["Worker service<br/>SQS consumer"]]
                    end
                    subgraph appB["AZ B · 10.0.12.0/24"]
                        apiB["API service<br/>REST"]
                        wrkB[["Worker service<br/>SQS consumer"]]
                    end
                end

                subgraph data["Private data subnets"]
                    subgraph dataA["AZ A · 10.0.21.0/24"]
                        dbW[("Aurora PostgreSQL<br/>writer")]
                        cacheP[("ElastiCache Redis<br/>primary")]
                    end
                    subgraph dataB["AZ B · 10.0.22.0/24"]
                        dbR[("Aurora PostgreSQL<br/>reader")]
                        cacheR[("ElastiCache Redis<br/>replica")]
                    end
                end

                subgraph endpoints["VPC endpoints"]
                    vpceS3(("S3 gateway<br/>endpoint"))
                    vpceIf(("Interface<br/>endpoints"))
                end
            end
        end
    end

    %% Edge order matters for layout, and linkStyle refers to edges by position (0 = first)
    users -.->|DNS| dns
    users ==>|HTTPS 443| waf
    waf ==> cdn
    cdn -->|/static/*| s3web
    cdn ==>|/api/*| igw
    igw -.- natA
    igw ==> alb
    igw -.- natB
    alb ==>|HTTP 8080| apiA
    alb ==>|HTTP 8080| apiB
    natA -.-|egress| appA
    natB -.-|egress| appB
    app -->|5432 · 6379| data
    dbW <-.->|replication| dbR
    cacheP <-.->|replication| cacheR
    app -->|AWS APIs, private| endpoints
    vpceS3 --> s3up
    vpceIf -->|PrivateLink| plink
    s3up -->|object created| fn
    gha -->|push image| ecr
    cw -->|alarm| sns
    sns -->|notify| oncall

    classDef ext fill:#232F3E,stroke:#5A6B7F,color:#fff
    classDef net fill:#8C4FFF,stroke:#6A3BC2,color:#fff
    classDef sec fill:#DD344C,stroke:#A8233A,color:#fff
    classDef compute fill:#ED7100,stroke:#B35500,color:#fff
    classDef db fill:#C925D1,stroke:#951B9B,color:#fff
    classDef storage fill:#7AA116,stroke:#5B7A10,color:#fff
    classDef integration fill:#E7157B,stroke:#B0105D,color:#fff
    class users,gha,oncall ext
    class dns,cdn,igw,alb,natA,natB,vpceS3,vpceIf net
    class waf,secrets sec
    class apiA,apiB,wrkA,wrkB,fn,ecr compute
    class dbW,dbR,cacheP,cacheR db
    class s3web,s3up storage
    class sqs,sns,cw integration

    style cloud fill:none,stroke:#7D8998,stroke-width:2px,color:#7D8998
    style edge fill:none,stroke:#8C4FFF,stroke-dasharray:4 3,color:#8C4FFF
    style region fill:none,stroke:#00A4A6,stroke-width:2px,stroke-dasharray:6 4,color:#00A4A6
    style regional fill:none,stroke:#7D8998,stroke-dasharray:4 3,color:#7D8998
    style vpc fill:none,stroke:#8C4FFF,stroke-width:2px,color:#8C4FFF
    style public fill:#F2F6E8,stroke:#7AA116,color:#3B5209
    style app fill:#E6F6F7,stroke:#00A4A6,color:#00626A
    style data fill:#E6F6F7,stroke:#00A4A6,color:#00626A
    style plink fill:none,stroke:#8C4FFF,stroke-dasharray:3 3,color:#8C4FFF
    style endpoints fill:#F4EDFF,stroke:#8C4FFF,color:#5A30B5
    style pubA fill:#fff,stroke:#7AA116,stroke-dasharray:4 3,color:#3B5209
    style pubB fill:#fff,stroke:#7AA116,stroke-dasharray:4 3,color:#3B5209
    style appA fill:#fff,stroke:#00A4A6,stroke-dasharray:4 3,color:#00626A
    style appB fill:#fff,stroke:#00A4A6,stroke-dasharray:4 3,color:#00626A
    style dataA fill:#fff,stroke:#00A4A6,stroke-dasharray:4 3,color:#00626A
    style dataB fill:#fff,stroke:#00A4A6,stroke-dasharray:4 3,color:#00626A

    %% thick purple = inbound requests, thin purple = private AWS access, orange = egress, pink = replication
    linkStyle 1,2,4,6,8,9 stroke:#8C4FFF,stroke-width:2.5px
    linkStyle 5,7,10,11 stroke:#ED7100
    linkStyle 13,14 stroke:#C925D1
    linkStyle 16,17 stroke:#8C4FFF
```

# example 2

```mermaid
%%{init: {"fontFamily": "Helvetica, Arial, sans-serif", "flowchart": {"curve": "step", "nodeSpacing": 40, "rankSpacing": 55, "padding": 14}}}%%
flowchart TB
    %% Actors outside AWS
    users(["Users<br/>web and mobile"])

    subgraph cloud["AWS Cloud"]
        subgraph edge["Global edge"]
            dns{{"Route 53<br/>DNS"}}
            waf[\"AWS WAF<br/>managed rules"/]
            cdn{{"CloudFront<br/>CDN"}}
        end

        subgraph region["Region · us-east-1"]
            %% Regional services that live outside the VPC go here
            s3web[("S3<br/>static assets")]

            subgraph vpc["VPC · 10.0.0.0/16"]
                igw(("Internet<br/>Gateway"))

                subgraph public["Public subnets"]
                    subgraph pubA["AZ A · 10.0.1.0/24"]
                        natA(("NAT<br/>Gateway"))
                    end
                    %% The ALB spans both public subnets, so it sits between them
                    alb{{"Application Load Balancer<br/>HTTPS 443"}}
                    subgraph pubB["AZ B · 10.0.2.0/24"]
                        natB(("NAT<br/>Gateway"))
                    end
                end

                subgraph app["Private app subnets · EKS cluster"]
                    subgraph appA["AZ A · 10.0.11.0/24"]
                        svcA["App deployment<br/>EKS pods"]
                    end
                    subgraph appB["AZ B · 10.0.12.0/24"]
                        svcB["App deployment<br/>EKS pods"]
                    end
                end

                subgraph data["Private data subnets"]
                    subgraph dataA["AZ A · 10.0.21.0/24"]
                        dbW[("Aurora PostgreSQL<br/>writer")]
                        cacheP[("ElastiCache Redis<br/>primary")]
                    end
                    subgraph dataB["AZ B · 10.0.22.0/24"]
                        dbR[("Aurora PostgreSQL<br/>read replica")]
                        cacheR[("ElastiCache Redis<br/>replica")]
                    end
                end
            end
        end
    end

    %% Edges. Arrow type sets color (see SKILL.md). Order sets left-to-right placement:
    %% the igw lines are interleaved natA, alb, natB so the ALB lands in the middle.
    users -.->|DNS| dns
    users ==>|HTTPS 443| waf
    waf ==> cdn
    cdn -->|/static/*| s3web
    cdn ==>|/api/*| igw
    igw -.- natA
    igw ==> alb
    igw -.- natB
    alb ==>|HTTP 8080| svcA
    alb ==>|HTTP 8080| svcB
    natA -.-|egress| appA
    natB -.-|egress| appB
    app -->|TCP 5432 · 6379| data
    dbW <-.->|replication| dbR
    cacheP <-.->|replication| cacheR

    %% Node colors: AWS Architecture Icons category palette
    classDef ext fill:#232F3E,stroke:#5A6B7F,color:#fff
    classDef net fill:#8C4FFF,stroke:#6A3BC2,color:#fff
    classDef sec fill:#DD344C,stroke:#A8233A,color:#fff
    classDef compute fill:#ED7100,stroke:#B35500,color:#fff
    classDef db fill:#C925D1,stroke:#951B9B,color:#fff
    classDef storage fill:#7AA116,stroke:#5B7A10,color:#fff
    classDef integration fill:#E7157B,stroke:#B0105D,color:#fff
    class users ext
    class waf sec
    class dns,cdn,igw,alb,natA,natB net
    class svcA,svcB compute
    class dbW,dbR,cacheP,cacheR db
    class s3web storage

    %% Box colors: AWS group conventions. fill:none boxes adapt to GitHub dark mode.
    style cloud fill:none,stroke:#7D8998,stroke-width:2px,color:#7D8998
    style edge fill:none,stroke:#8C4FFF,stroke-dasharray:4 3,color:#8C4FFF
    style region fill:none,stroke:#00A4A6,stroke-width:2px,stroke-dasharray:6 4,color:#00A4A6
    style vpc fill:none,stroke:#8C4FFF,stroke-width:2px,color:#8C4FFF
    style public fill:#F2F6E8,stroke:#7AA116,color:#3B5209
    style app fill:#E6F6F7,stroke:#00A4A6,color:#00626A
    style data fill:#E6F6F7,stroke:#00A4A6,color:#00626A
    style pubA fill:#fff,stroke:#7AA116,stroke-dasharray:4 3,color:#3B5209
    style pubB fill:#fff,stroke:#7AA116,stroke-dasharray:4 3,color:#3B5209
    style appA fill:#fff,stroke:#00A4A6,stroke-dasharray:4 3,color:#00626A
    style appB fill:#fff,stroke:#00A4A6,stroke-dasharray:4 3,color:#00626A
    style dataA fill:#fff,stroke:#00A4A6,stroke-dasharray:4 3,color:#00626A
    style dataB fill:#fff,stroke:#00A4A6,stroke-dasharray:4 3,color:#00626A

    %% linkStyle: generated by lint_mermaid.py --fix from arrow types; do not edit by hand
    linkStyle 1,2,4,6,8,9 stroke:#8C4FFF,stroke-width:2.5px
    linkStyle 5,7,10,11 stroke:#ED7100
    linkStyle 13,14 stroke:#C925D1
```




## Mermaid testing 2

```mermaid
%%{init: {"fontFamily": "Helvetica, Arial, sans-serif", "flowchart": {"curve": "step", "nodeSpacing": 40, "rankSpacing": 50, "padding": 14}}}%%
flowchart TB
    user@{ shape: person, label: "User" }
    dev@{ shape: person, label: "Developer" }
    web@{ shape: browser, label: "Web app<br/>React SPA" }
    internet@{ shape: cloud, label: "Internet" }
    repo@{ shape: folder, label: "CDK app<br/>infrastructure as code" }
    ci@{ shape: console, label: "GitHub Actions<br/>cdk deploy" }

    subgraph aws["AWS Cloud · us-east-1"]
        cdn{{"CloudFront<br/>CDN + WAF"}}
        site@{ shape: bucket, label: "S3 site<br/>static files" }

        subgraph api["API"]
            apigw{{"API Gateway<br/>HTTP API"}}
            cognito("Cognito<br/>user pool")
            fnApi>"Lambda<br/>API handlers"]
        end

        subgraph processing["Image processing"]
            uploads@{ shape: bucket, label: "S3 uploads<br/>original photos" }
            queue@{ shape: h-cyl, label: "SQS<br/>resize jobs" }
            fnResize>"Lambda<br/>thumbnailer"]
            thumbs@{ shape: bucket, label: "S3 thumbnails" }
        end

        ddb@{ shape: datastore, label: "DynamoDB<br/>photo metadata" }
    end

    user --> web
    web ==>|HTTPS 443| internet
    internet ==> cdn
    cdn -->|/*| site
    cdn -->|/img/*| thumbs
    cdn ==>|/api/*| apigw
    apigw ==>|invoke| fnApi
    apigw -.->|verify JWT| cognito
    fnApi -->|presigned URL| uploads
    fnApi -->|read and write| ddb
    web -->|PUT with presigned URL| uploads
    uploads -->|object created| queue
    queue -->|batch| fnResize
    fnResize -->|write| thumbs
    fnResize -->|update status| ddb
    dev --> repo
    repo -->|git push| ci
    ci -->|deploy| aws

    classDef ext fill:#232F3E,stroke:#5A6B7F,color:#fff
    classDef net fill:#8C4FFF,stroke:#6A3BC2,color:#fff
    classDef sec fill:#DD344C,stroke:#A8233A,color:#fff
    classDef compute fill:#ED7100,stroke:#B35500,color:#fff
    classDef db fill:#C925D1,stroke:#951B9B,color:#fff
    classDef storage fill:#7AA116,stroke:#5B7A10,color:#fff
    classDef integration fill:#E7157B,stroke:#B0105D,color:#fff
    class user,dev,web,internet,repo,ci ext
    class cdn,apigw net
    class cognito sec
    class fnApi,fnResize compute
    class ddb db
    class site,thumbs,uploads storage
    class queue integration

    style aws fill:none,stroke:#7D8998,stroke-width:2px,color:#7D8998
    style api fill:none,stroke:#8C4FFF,stroke-dasharray:4 3,color:#8C4FFF
    style processing fill:none,stroke:#ED7100,stroke-dasharray:4 3,color:#ED7100

    %% Edge colors (indexes from check_diagram.py): purple = request path,
    %% green = direct upload, which bypasses the API
    linkStyle 1,2,5,6 stroke:#8C4FFF,stroke-width:2.5px
    linkStyle 10 stroke:#7AA116,stroke-width:2px
```
