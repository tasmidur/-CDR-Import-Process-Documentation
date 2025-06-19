# CDR Import Process Documentation

## Overview
This system imports Call Detail Records (CDRs) from a database for multiple enterprises, processes the records, and distributes them to appropriate destinations based on enterprise and group configurations.

## Main Components

### 1. Core Process (`DoImport` in `CDRImportProcess.cs`)
The main workflow that orchestrates the CDR import cycle for all configured enterprises.

### 2. Supporting Functions
- `GetEnterpriseGroups`: Retrieves group configurations for each enterprise
- `Import`: Handles the actual CDR data retrieval and processing
- `HandlerError`: Centralized error handling and notification
- File operations for writing and moving CDR files

## Workflow Description

```mermaid
flowchart TD
    A[Start DoImport] --> B[DisableTimer]
    B --> C[GetEnterprises]
    C --> D[Initialize CdrImporter]
    D --> E{For each enterprise}
    E -->|Next enterprise| F[CreateEnterprise]
    F --> G{Enterprise Enabled?}
    G -->|Yes| H[LogInfo: Importing CDR]
    H --> I[GetEnterpriseGroups]
    I --> J[Set importer properties]
    J --> K[Import CDR]
    K --> L{Import successful?}
    L -->|Yes| M[Update LastImportedCdrID]
    M --> N[LogInfo: Records found]
    L -->|No| O[HandlerError]
    G -->|No| P[LogInfo: Enterprise Disabled]
    E -->|All enterprises processed| Q[LogInfo: NEW CYCLE]
    Q --> R{stopProcess?}
    R -->|No| S[EnableTimer]
    R -->|Yes| T[End]
    
    subgraph Import CDR Process
    K --> K1[Initialize save directory]
    K1 --> K2[Build SQL query]
    K2 --> K3[SetupLog]
    K3 --> K4[Execute query]
    K4 --> K5{Has rows?}
    K5 -->|Yes| K6[Process each row]
    K6 --> K7[Convert time fields]
    K7 --> K8[Update NextLastProcessedCdrID]
    K8 --> K9{Emergency number?}
    K9 -->|Yes| K10[AddEmergencyCDRToGroup]
    K9 -->|No| K11{ImportOnlyStartAST?}
    K11 -->|Yes| K12{acctStatusType=stop?}
    K12 -->|Yes| K13[AddCDRToGroup]
    K11 -->|No| K13
    K13 --> K14[DoASTCounterLogic]
    K5 -->|No| K15[Set zero records]
    K14 --> K6
    K6 -->|All rows| K16[WriteAndMoveCDRFiles]
    end
    
    subgraph Error Handling
    O --> O1[Build error message]
    O1 --> O2[LogError]
    O2 --> O3[SendEmail]
    end
    
    subgraph File Operations
    K16 --> K17[Create directories]
    K17 --> K18[Write files]
    K18 --> K19[Move files]
    K19 --> K20{Network error?}
    K20 -->|Yes| K21[ConnectToNetworkDrive]
    K21 --> K19
    end
```

## Key Features

1. **Enterprise-Specific Processing**:
   - Handles each enterprise independently
   - Maintains last processed CDR ID per enterprise
   - Supports enterprise-specific configurations

2. **Error Handling**:
   - Comprehensive error logging
   - Email notifications for critical errors
   - Enterprise-specific error tracking

3. **File Operations**:
   - Temporary file staging
   - Secure file transfer to network locations
   - Automatic retry for network issues

4. **Performance Tracking**:
   - Record counters
   - Processing statistics
   - Detailed logging

## Configuration Requirements

### Database Tables
- `ImportCdrEnterprises`: Contains enterprise configurations
  - Fields: EnterpriseID, LastImportedCdrID, MaxRecordsToFetch, Enabled
- `ImportCdrGroups`: Contains group destinations
  - Fields: GroupID, CDRSavePath, CDREmergencySavePath, UserName, Password

### App Settings
- Database connection strings
- Email notification settings
- Timer intervals
- File paths for temporary storage

## Usage Examples

### Starting the Import Process
```csharp
// Initialize and start the import process
var importer = new CDRImportProcess();
importer.Start();
```

### Sample Enterprise Configuration
```sql
INSERT INTO ImportCdrEnterprises 
(EnterpriseID, LastImportedCdrID, MaxRecordsToFetch, Enabled)
VALUES 
('Ent00001', 9301025599, 1000, 1);
```

### Sample Group Configuration
```sql
INSERT INTO ImportCdrGroups
(GroupID, EnterpriseID, CDRSavePath, CDREmergencySavePath, UserName, Password)
VALUES
('709000001', 'Ent00001', 'F:\CDR\Ent00001', F:\CDR\Ent00001', 'user1', 'encryptedpass');
```

## Error Handling

The system provides multiple layers of error handling:

1. **Enterprise-level errors**: Failures for one enterprise don't affect others
2. **Network retries**: Automatic retry for network share access issues
3. **Notification system**: Email alerts for critical failures
