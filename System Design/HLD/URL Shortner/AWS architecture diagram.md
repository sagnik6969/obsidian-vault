```
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%

  

flowchart LR

Client([User / Client Browser])

DNS[Amazon Route 53]

  

subgraph AWS Cloud [AWS Region - VPC]

ALB[Application Load Balancer<br/>Path-Based Routing]

  

subgraph Amazon EKS Cluster [EKS Compute & State]

subgraph Write Microservice [Shortener Service]

HPA_W[[HPA - Write]]

WritePod1[Write Pod 1<br/>ID Manager]

WritePod2[Write Pod 2<br/>ID Manager]

HPA_W -.- WritePod1 & WritePod2

end

  

subgraph Read Microservice [Redirect Service]

HPA_R[[HPA - Read]]

ReadPod1[Read Pod 1<br/>Cache Logic]

ReadPod2[Read Pod 2<br/>Cache Logic]

ReadPod3[Read Pod 3<br/>Cache Logic]

HPA_R -.- ReadPod1 & ReadPod2 & ReadPod3

end

  

subgraph Stateful Coordination [ZooKeeper Quorum]

ZK1[(ZK Pod 1)]

ZK2[(ZK Pod 2)]

ZK3[(ZK Pod 3)]

ZK1 --- ZK2 --- ZK3

end

  

end

  

subgraph Managed Data Services

Redis[(Amazon ElastiCache<br/>Redis)]

Dynamo[(Amazon DynamoDB)]

end

end

  

%% Routing Logic

Client -->|"1. DNS Lookup"| DNS

DNS -.-> ALB

ALB -->|"2a. POST /urls"| WritePod1 & WritePod2

ALB -->|"2b. GET /{alias}"| ReadPod1 & ReadPod2 & ReadPod3

  

%% Write Path Connections

WritePod1 & WritePod2 <-->|"3. Fetch ID Blocks"| ZK1

WritePod1 & WritePod2 -->|"4. Write Mapping"| Dynamo

  

%% Read Path Connections

ReadPod1 & ReadPod2 & ReadPod3 <-->|"5. Check Cache"| Redis

ReadPod1 & ReadPod2 & ReadPod3 <-->|"6. Fetch on Cache Miss"| Dynamo

  

%% Styling

classDef compute fill:#F58536,stroke:#232F3E,stroke-width:2px,color:white,font-weight:bold;

classDef database fill:#3B48CC,stroke:#232F3E,stroke-width:2px,color:white,font-weight:bold;

classDef network fill:#8C4FFF,stroke:#232F3E,stroke-width:2px,color:white,font-weight:bold;

classDef state fill:#1DA1F2,stroke:#232F3E,stroke-width:2px,color:white,font-weight:bold;

  

class DNS,ALB network;

class WritePod1,WritePod2,ReadPod1,ReadPod2,ReadPod3,HPA_W,HPA_R compute;

class Redis,Dynamo database;

class ZK1,ZK2,ZK3 state;    
```


