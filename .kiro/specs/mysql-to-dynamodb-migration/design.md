# Design Document

## Overview

This design outlines the migration from MySQL to DynamoDB using a single table design pattern with blue-green deployment strategy. The solution maintains zero downtime while transforming a relational billing system into a NoSQL structure optimized for DynamoDB's strengths.

## Architecture

### Migration Architecture

```mermaid
graph TB
    subgraph "On-Premises Environment"
        subgraph "Blue Environment"
            App1[Application Layer]
            MySQL[(MySQL Database)]
            App1 --> MySQL
        end
        
        subgraph "Migration Bridge"
            DMS[AWS DMS Instance]
            VPN[VPN Gateway]
            DX[Direct Connect]
            DMS --> MySQL
        end
    end
    
    subgraph "AWS Cloud Environment"
        subgraph "Network Layer"
            VPC[VPC]
            PrivSub[Private Subnets]
            PubSub[Public Subnets]
            NAT[NAT Gateway]
        end
        
        subgraph "Migration Infrastructure"
            MS[Migration Service]
            VS[Validation Service]
            S3[S3 Staging Bucket]
            MS --> S3
            VS --> S3
        end
        
        subgraph "Target Green Environment"
            App2[Application Layer]
            DDB[(DynamoDB Single Table)]
            App2 --> DDB
        end
        
        subgraph "Load Balancer"
            LB[Application Load Balancer]
            LB --> App1
            LB -.-> App2
        end
        
        subgraph "Monitoring"
            CW[CloudWatch]
            MS --> CW
            VS --> CW
            DMS --> CW
        end
    end
    
    VPN -.-> VPC
    DX -.-> VPC
    DMS --> S3
    MS --> DDB
    VS --> DDB
    VS --> MySQL
```

### On-Premises to AWS Migration Flow

```mermaid
sequenceDiagram
    participant OnPrem as On-Premises MySQL
    participant DMS as AWS DMS
    participant S3 as S3 Staging
    participant MS as Migration Service
    participant DDB as DynamoDB
    participant LB as Load Balancer
    participant App as Application
    
    Note over OnPrem,DDB: Phase 1: Network Setup & Initial Sync
    MS->>DMS: Configure DMS replication instance
    DMS->>OnPrem: Establish secure connection
    DMS->>S3: Full load data export
    MS->>S3: Validate exported data
    
    Note over OnPrem,DDB: Phase 2: Data Transformation & Loading
    MS->>S3: Read exported MySQL data
    MS->>MS: Transform to DynamoDB format
    MS->>DDB: Batch load transformed data
    MS->>MS: Validate data integrity
    
    Note over OnPrem,DDB: Phase 3: CDC and Incremental Sync
    DMS->>OnPrem: Enable change data capture
    DMS->>S3: Stream incremental changes
    MS->>S3: Process change events
    MS->>DDB: Apply incremental updates
    
    Note over OnPrem,DDB: Phase 4: Blue-Green Switch
    MS->>MS: Final validation check
    LB->>App: Switch traffic to DynamoDB
    MS->>OnPrem: Monitor for rollback window
    
    Note over OnPrem,DDB: Phase 5: Cleanup
    MS->>DMS: Stop replication
    MS->>S3: Archive migration data
```

## On-Premises to AWS Connectivity

### Network Architecture Options

#### Option 1: VPN Connection
- **Use Case**: Lower data volumes, cost-sensitive migrations
- **Bandwidth**: Up to 1.25 Gbps per tunnel
- **Latency**: Variable, internet-dependent
- **Security**: IPSec encryption
- **Cost**: Lower setup and operational costs

#### Option 2: AWS Direct Connect
- **Use Case**: High data volumes, consistent performance requirements
- **Bandwidth**: 1 Gbps to 100 Gbps dedicated
- **Latency**: Consistent, predictable
- **Security**: Private connection, optional encryption
- **Cost**: Higher setup cost, predictable data transfer costs

#### Hybrid Approach
- Primary: Direct Connect for bulk data transfer
- Backup: VPN for redundancy and smaller incremental updates

### Data Transfer Strategy

#### AWS Database Migration Service (DMS) Integration
```mermaid
graph LR
    subgraph "On-Premises"
        MySQL[(MySQL)]
        DMSAgent[DMS Replication Agent]
    end
    
    subgraph "AWS"
        DMSInstance[DMS Replication Instance]
        S3Bucket[S3 Staging Bucket]
        Lambda[Lambda Processor]
        DDB[(DynamoDB)]
    end
    
    MySQL --> DMSAgent
    DMSAgent --> DMSInstance
    DMSInstance --> S3Bucket
    S3Bucket --> Lambda
    Lambda --> DDB
```

#### Data Transfer Phases

1. **Full Load Phase**
   - DMS performs complete data export from on-premises MySQL
   - Data staged in S3 with compression and encryption
   - Parallel processing for large tables
   - Checksum validation for data integrity

2. **Change Data Capture (CDC) Phase**
   - Real-time capture of changes from MySQL binary logs
   - Incremental updates streamed to S3
   - Event-driven processing with Lambda triggers
   - Minimal impact on source database performance

3. **Cutover Phase**
   - Final sync of remaining changes
   - Application traffic switch with minimal downtime
   - Rollback capability maintained

### Security Considerations

#### Data in Transit
- TLS 1.2+ encryption for all connections
- VPN IPSec tunneling for internet-based connections
- Direct Connect with MACsec encryption option
- Certificate-based authentication

#### Data at Rest
- S3 server-side encryption (SSE-S3 or SSE-KMS)
- DynamoDB encryption at rest
- Encrypted EBS volumes for migration instances

#### Access Control
- IAM roles with least privilege principles
- VPC security groups restricting network access
- Database user accounts with minimal required permissions
- CloudTrail logging for all API calls

## Components and Interfaces

### DynamoDB Single Table Design

#### Table Structure
- **Table Name**: `BillingSystem`
- **Partition Key**: `PK` (String)
- **Sort Key**: `SK` (String)
- **Global Secondary Indexes**:
  - GSI1: `GSI1PK` / `GSI1SK` - For subscriber-based queries
  - GSI2: `GSI2PK` / `GSI2SK` - For time-based queries (bills, charges)

#### Entity Mapping

| Entity | PK Pattern | SK Pattern | GSI1PK | GSI1SK | GSI2PK | GSI2SK |
|--------|------------|------------|---------|---------|---------|---------|
| BILLING_ACCOUNT | `BAN#{BAN}` | `ACCOUNT` | - | - | - | - |
| CUSTOMER | `BAN#{BAN}` | `CUSTOMER` | - | - | - | - |
| SUBSCRIBER | `BAN#{BAN}` | `SUB#{subscriber_no}` | `SUB#{subscriber_no}` | `ACCOUNT` | - | - |
| SERVICE_AGREEMENT | `BAN#{BAN}` | `SA#{subscriber_no}#{SOC}` | `SUB#{subscriber_no}` | `SA#{SOC}` | - | - |
| BILL | `BAN#{BAN}` | `BILL#{bill_seq_number}` | - | - | `BILL#{cycle_code}` | `{cycle_close_date}` |
| CHARGE | `BAN#{BAN}` | `CHARGE#{charge_seq}#{activity_num}` | `SUB#{subscriber_no}` | `CHARGE#{charge_creation_date}` | `CHARGE#{feature_code}` | `{charge_creation_date}` |

#### DynamoDB Single Table Design Visualization

```mermaid
graph TB
    subgraph "DynamoDB Single Table: BillingSystem"
        subgraph "Table Structure"
            PK[PK - Partition Key]
            SK[SK - Sort Key]
            GSI1[GSI1PK / GSI1SK]
            GSI2[GSI2PK / GSI2SK]
            ET[EntityType]
        end
        
        subgraph "Entity Examples with Key Patterns"
            BA["BILLING_ACCOUNT<br/>PK: BAN#12345<br/>SK: ACCOUNT<br/>EntityType: BILLING_ACCOUNT"]
            
            CUST["CUSTOMER<br/>PK: BAN#12345<br/>SK: CUSTOMER<br/>EntityType: CUSTOMER"]
            
            SUB["SUBSCRIBER<br/>PK: BAN#12345<br/>SK: SUB#SUB001<br/>GSI1PK: SUB#SUB001<br/>GSI1SK: ACCOUNT<br/>EntityType: SUBSCRIBER"]
            
            SA["SERVICE_AGREEMENT<br/>PK: BAN#12345<br/>SK: SA#SUB001#SOC123<br/>GSI1PK: SUB#SUB001<br/>GSI1SK: SA#SOC123<br/>EntityType: SERVICE_AGREEMENT"]
            
            BILL["BILL<br/>PK: BAN#12345<br/>SK: BILL#001<br/>GSI2PK: BILL#MONTHLY<br/>GSI2SK: 2024-01-31<br/>EntityType: BILL"]
            
            CHARGE["CHARGE<br/>PK: BAN#12345<br/>SK: CHARGE#001#ACT001<br/>GSI1PK: SUB#SUB001<br/>GSI1SK: CHARGE#2024-01-15<br/>GSI2PK: CHARGE#VOICE<br/>GSI2SK: 2024-01-15<br/>EntityType: CHARGE"]
        end
    end
    
    PK --> BA
    PK --> CUST
    PK --> SUB
    PK --> SA
    PK --> BILL
    PK --> CHARGE
    
    style BA fill:#e1f5fe
    style CUST fill:#f3e5f5
    style SUB fill:#e8f5e8
    style SA fill:#fff3e0
    style BILL fill:#fce4ec
    style CHARGE fill:#f1f8e9
```

#### Access Patterns with Key Structures

```mermaid
flowchart TD
    MainTable[DynamoDB Single Table<br/>BillingSystem]
    
    subgraph "Main Table Queries"
        AP1[Get All Data for Billing Account<br/>Query: PK = BAN#12345<br/>Returns: All entities for account]
        AP2[Get Specific Entity<br/>Query: PK = BAN#12345 AND SK = CUSTOMER<br/>Returns: Customer details]
    end
    
    subgraph "GSI1: Subscriber-Based Queries"
        GSI1_1[Get Subscriber + Service Agreements<br/>Query: GSI1PK = SUB#SUB001<br/>Returns: Subscriber and SAs]
        GSI1_2[Get Subscriber Charges<br/>Query: GSI1PK = SUB#SUB001<br/>Filter: GSI1SK begins_with CHARGE<br/>Returns: All subscriber charges]
    end
    
    subgraph "GSI2: Time-Based Queries"
        GSI2_1[Get Bills by Cycle<br/>Query: GSI2PK = BILL#MONTHLY<br/>Range: GSI2SK between dates<br/>Returns: Bills in date range]
        GSI2_2[Get Charges by Feature<br/>Query: GSI2PK = CHARGE#VOICE<br/>Range: GSI2SK between dates<br/>Returns: Voice charges in range]
    end
    
    MainTable --> AP1
    MainTable --> AP2
    MainTable --> GSI1_1
    MainTable --> GSI1_2
    MainTable --> GSI2_1
    MainTable --> GSI2_2
    
    style AP1 fill:#e3f2fd
    style AP2 fill:#e3f2fd
    style GSI1_1 fill:#e8f5e8
    style GSI1_2 fill:#e8f5e8
    style GSI2_1 fill:#fff3e0
    style GSI2_2 fill:#fff3e0
```

#### Key Design Patterns

1. **Get Billing Account with all related data**
   - Query: `PK = BAN#{BAN}` 
   - Returns: Account, Customer, Subscribers, Bills, Charges

2. **Get Subscriber with Service Agreements**
   - Query GSI1: `GSI1PK = SUB#{subscriber_no}`
   - Returns: Subscriber and all Service Agreements

3. **Get Bills by Cycle**
   - Query GSI2: `GSI2PK = BILL#{cycle_code}` with date range on `GSI2SK`

4. **Get Charges by Feature Code and Date Range**
   - Query GSI2: `GSI2PK = CHARGE#{feature_code}` with date range on `GSI2SK`

### Migration Service Components

#### S3 Data Processing Layer
```typescript
interface S3DataProcessor {
  processFullLoadFiles(s3Prefix: string): Promise<ProcessingResult>
  processCDCEvents(s3Key: string): Promise<CDCResult>
  validateS3Data(s3Key: string): Promise<ValidationResult>
  archiveProcessedData(s3Key: string): Promise<void>
}

interface DMSEventProcessor {
  parseFullLoadRecord(record: S3Record): Promise<MySQLEntity>
  parseCDCRecord(record: S3Record): Promise<CDCEvent>
  handleInsertEvent(event: CDCEvent): Promise<void>
  handleUpdateEvent(event: CDCEvent): Promise<void>
  handleDeleteEvent(event: CDCEvent): Promise<void>
}
```

#### Data Extraction Layer (Legacy - for validation)
```typescript
interface DataExtractor {
  extractBillingAccounts(): Promise<BillingAccount[]>
  extractSubscribers(ban: string): Promise<Subscriber[]>
  extractCharges(ban: string, dateRange?: DateRange): Promise<Charge[]>
  extractBills(ban: string): Promise<Bill[]>
  extractCustomers(ban: string): Promise<Customer[]>
  extractServiceAgreements(ban: string): Promise<ServiceAgreement[]>
}
```

#### Data Transformation Layer
```typescript
interface DataTransformer {
  transformToDynamoDBItem(entity: MySQLEntity): DynamoDBItem
  generatePartitionKey(entity: MySQLEntity): string
  generateSortKey(entity: MySQLEntity): string
  generateGSIKeys(entity: MySQLEntity): GSIKeys
}
```

#### Data Loading Layer
```typescript
interface DataLoader {
  batchWriteItems(items: DynamoDBItem[]): Promise<BatchWriteResult>
  validateWrite(item: DynamoDBItem): Promise<boolean>
  handleWriteErrors(errors: WriteError[]): Promise<void>
}
```

## Data Models

### DynamoDB Item Structure

```typescript
interface DynamoDBItem {
  PK: string           // Partition Key
  SK: string           // Sort Key
  GSI1PK?: string      // GSI1 Partition Key
  GSI1SK?: string      // GSI1 Sort Key
  GSI2PK?: string      // GSI2 Partition Key
  GSI2SK?: string      // GSI2 Sort Key
  EntityType: string   // ACCOUNT, CUSTOMER, SUBSCRIBER, etc.
  CreatedAt: string    // ISO timestamp
  UpdatedAt: string    // ISO timestamp
  TTL?: number         // For temporary migration tracking items
  [key: string]: any   // Entity-specific attributes
}
```

### Migration Tracking

```typescript
interface MigrationStatus {
  PK: string           // "MIGRATION#STATUS"
  SK: string           // "BATCH#{batch_id}"
  BatchId: string
  EntityType: string
  Status: 'PENDING' | 'IN_PROGRESS' | 'COMPLETED' | 'FAILED'
  RecordsProcessed: number
  RecordsFailed: number
  StartTime: string
  EndTime?: string
  ErrorDetails?: string[]
  TTL: number          // Auto-cleanup after migration
}
```

## Error Handling

### Migration Error Strategies

1. **Batch Processing Errors**
   - Implement exponential backoff for throttling
   - Dead letter queue for failed items
   - Detailed logging with correlation IDs

2. **Data Validation Errors**
   - Pre-migration validation checks
   - Post-migration data integrity verification
   - Rollback procedures for critical failures

3. **Application Layer Errors**
   - Circuit breaker pattern for DynamoDB calls
   - Graceful degradation during migration
   - Health check endpoints for monitoring

### Network Resilience and Failover

#### Connection Monitoring
```typescript
interface NetworkMonitor {
  checkVPNHealth(): Promise<ConnectionStatus>
  checkDirectConnectHealth(): Promise<ConnectionStatus>
  monitorBandwidthUtilization(): Promise<BandwidthMetrics>
  detectNetworkPartition(): Promise<boolean>
}

interface FailoverManager {
  switchToBackupConnection(): Promise<void>
  pauseMigrationOnNetworkIssue(): Promise<void>
  resumeMigrationAfterRecovery(): Promise<void>
  validateDataIntegrityAfterFailover(): Promise<ValidationResult>
}
```

#### Data Consistency During Network Issues
- **Checkpoint Management**: Regular checkpoints stored in S3
- **Resume Capability**: Ability to resume from last successful checkpoint
- **Duplicate Detection**: Handling of duplicate records during network recovery
- **Ordering Guarantees**: Maintaining CDC event ordering across network interruptions

### Rollback Strategy

```mermaid
graph LR
    A[Detect Issue] --> B[Stop DMS Replication]
    B --> C[Preserve On-Prem MySQL State]
    C --> D[Switch Load Balancer Back]
    D --> E[Validate Rollback]
    E --> F[Monitor System]
    F --> G[Clean Up AWS Resources]
```

## Testing Strategy

### Migration Testing Phases

1. **Unit Testing**
   - Data transformation logic
   - Key generation algorithms
   - Validation functions

2. **Integration Testing**
   - End-to-end migration pipeline
   - DynamoDB read/write operations
   - Application layer compatibility

3. **Performance Testing**
   - Migration throughput benchmarks
   - DynamoDB query performance
   - Load testing during blue-green switch

4. **Data Validation Testing**
   - Record count verification
   - Data integrity checks
   - Relationship preservation validation

### Monitoring and Alerting

#### Key Metrics
- Migration progress (records/minute)
- Error rates by entity type
- DynamoDB consumed capacity
- Application response times
- Data consistency checks

#### Alert Conditions
- Migration failure rate > 1%
- DynamoDB throttling events
- Data validation failures
- Application error rate increase
- Blue-green switch failures

### Performance Optimization for On-Premises Migration

#### Bandwidth Management
- **Compression**: Enable compression for DMS data transfer
- **Parallel Processing**: Multiple DMS tasks for large tables
- **Throttling**: Rate limiting to avoid overwhelming on-premises network
- **Off-Peak Scheduling**: Schedule bulk transfers during low-usage periods

#### Data Transfer Optimization
```typescript
interface TransferOptimizer {
  calculateOptimalBatchSize(networkBandwidth: number): number
  scheduleTransferWindows(businessHours: TimeWindow[]): Schedule
  monitorTransferRates(): Promise<TransferMetrics>
  adjustTransferParameters(metrics: TransferMetrics): Promise<void>
}
```

### Deployment Checklist

1. **Pre-Migration**
   - [ ] Network connectivity established (VPN/Direct Connect)
   - [ ] DMS replication instance configured and tested
   - [ ] S3 staging bucket created with proper permissions
   - [ ] DynamoDB table created with proper capacity
   - [ ] Security groups and IAM roles configured
   - [ ] Migration scripts tested in staging environment
   - [ ] Monitoring dashboards configured
   - [ ] Rollback procedures documented and tested

2. **During Migration**
   - [ ] Network connectivity monitoring active
   - [ ] DMS replication monitoring active
   - [ ] Real-time data validation running
   - [ ] Performance metrics tracked (bandwidth, latency, throughput)
   - [ ] Error handling and retry mechanisms verified
   - [ ] CDC lag monitoring for incremental sync

3. **Post-Migration**
   - [ ] Data integrity validated between on-premises and DynamoDB
   - [ ] Performance benchmarks met
   - [ ] Application functionality verified
   - [ ] Network connections maintained for rollback window
   - [ ] On-premises MySQL preserved for rollback period
   - [ ] Migration infrastructure cleanup scheduled