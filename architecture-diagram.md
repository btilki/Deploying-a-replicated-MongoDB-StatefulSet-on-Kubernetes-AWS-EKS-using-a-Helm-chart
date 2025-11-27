# Architecture Diagram

```mermaid
flowchart TB
    subgraph Internet
        User[("👤 User Browser")]
    end

    subgraph AWS["AWS Cloud"]
        subgraph EKS["EKS Cluster"]
            subgraph IngressNS["ingress-nginx namespace"]
                LB["AWS Load Balancer<br/>(EXTERNAL-IP)"]
                IC["nginx Ingress Controller"]
            end
            
            subgraph DefaultNS["default namespace"]
                Ingress["Ingress<br/>(mongo-express)"]
                
                subgraph MongoExpress["Mongo Express"]
                    MES["mongo-express-service<br/>(ClusterIP:8081)"]
                    MEP["Deployment<br/>mongo-express"]
                end
                
                subgraph MongoDB["MongoDB Replica Set"]
                    MHS["mongodb-headless<br/>(Headless Service)"]
                    M0["Pod: mongodb-0<br/>(Primary)"]
                    M1["Pod: mongodb-1<br/>(Secondary)"]
                    M2["Pod: mongodb-2<br/>(Secondary)"]
                end
                
                Secret["Secret<br/>mongodb-root-password"]
            end
            
            subgraph Storage["Persistent Storage"]
                PVC0["PVC: mongodb-0"]
                PVC1["PVC: mongodb-1"]
                PVC2["PVC: mongodb-2"]
            end
        end
        
        subgraph EBS["AWS EBS (gp3)"]
            EBS0["EBS Volume 0"]
            EBS1["EBS Volume 1"]
            EBS2["EBS Volume 2"]
        end
    end

    User -->|"HTTP/HTTPS<br/>YOUR_HOST_DNS_NAME"| LB
    LB --> IC
    IC --> Ingress
    Ingress --> MES
    MES --> MEP
    MEP -->|"mongodb://..."| MHS
    MEP -.->|"Read Password"| Secret
    MHS --> M0
    MHS --> M1
    MHS --> M2
    M0 <-->|"Replication"| M1
    M1 <-->|"Replication"| M2
    M0 <-->|"Replication"| M2
    M0 --> PVC0
    M1 --> PVC1
    M2 --> PVC2
    PVC0 --> EBS0
    PVC1 --> EBS1
    PVC2 --> EBS2

    classDef aws fill:#FF9900,stroke:#232F3E,color:#232F3E
    classDef k8s fill:#326CE5,stroke:#fff,color:#fff
    classDef mongo fill:#00ED64,stroke:#023430,color:#023430
    classDef storage fill:#7B68EE,stroke:#fff,color:#fff
    
    class LB,EBS0,EBS1,EBS2 aws
    class IC,Ingress,MES,MEP,MHS,Secret k8s
    class M0,M1,M2 mongo
    class PVC0,PVC1,PVC2 storage
```

## Component Overview

| Component | Description |
|-----------|-------------|
| **AWS Load Balancer** | Automatically provisioned by nginx Ingress Controller to handle external traffic |
| **nginx Ingress Controller** | Routes HTTP traffic based on host/path rules to internal services |
| **Ingress Resource** | Defines routing rules to direct traffic to mongo-express-service |
| **Mongo Express** | Web UI for MongoDB administration (Deployment + Service) |
| **MongoDB Replica Set** | 3-node StatefulSet with primary/secondary replication |
| **Headless Service** | Enables direct pod-to-pod communication for replica set |
| **Kubernetes Secret** | Stores MongoDB root password securely |
| **PersistentVolumeClaims** | Request storage from AWS EBS for each MongoDB replica |
| **AWS EBS (gp3)** | Persistent block storage for MongoDB data |

## Traffic Flow

1. User accesses `http(s)://YOUR_HOST_DNS_NAME/` in browser (HTTPS recommended for production)
2. DNS resolves to AWS Load Balancer external IP
3. Load Balancer forwards to nginx Ingress Controller
4. Ingress Controller matches host rule and routes to mongo-express-service
5. Mongo Express connects to MongoDB via headless service (`mongodb-0.mongodb-headless:27017`)
6. MongoDB replica set maintains data consistency across 3 pods
7. Each MongoDB pod persists data to its own AWS EBS volume
