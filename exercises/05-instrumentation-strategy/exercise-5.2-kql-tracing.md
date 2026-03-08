# Exercise 5.2: KQL Queries & Distributed Tracing

## Objective
Query logs using Kusto Query Language (KQL) and inspect distributed tracing in Application Insights.

## Skills Measured
- Analyze metrics by using collected telemetry, including usage and application performance
- Inspect distributed tracing by using Application Insights
- Interrogate logs using basic Kusto Query Language (KQL) queries

## Steps

### Part A: Basic KQL Queries

Open Azure Portal → Log Analytics workspace or Application Insights → Logs

1. **Requests — find slow requests**:
   ```kql
   requests
   | where timestamp > ago(24h)
   | where duration > 1000  // Slower than 1 second
   | project timestamp, name, url, duration, resultCode
   | order by duration desc
   | take 20
   ```

2. **Exceptions — find top errors**:
   ```kql
   exceptions
   | where timestamp > ago(7d)
   | summarize count() by type, outerMessage
   | order by count_ desc
   | take 10
   ```

3. **Performance — average response time per endpoint**:
   ```kql
   requests
   | where timestamp > ago(1h)
   | summarize avgDuration=avg(duration), count=count() by name
   | order by avgDuration desc
   ```

4. **Availability — uptime percentage**:
   ```kql
   availabilityResults
   | where timestamp > ago(24h)
   | summarize
       totalTests = count(),
       successfulTests = countif(success == true)
   | extend uptimePercent = round(100.0 * successfulTests / totalTests, 2)
   ```

5. **Server performance — CPU and memory**:
   ```kql
   performanceCounters
   | where timestamp > ago(1h)
   | where category == "Processor" and counter == "% Processor Time"
   | summarize avgCPU=avg(value) by bin(timestamp, 5m)
   | render timechart
   ```

### Part B: Intermediate KQL

1. **Join requests with exceptions** (find which requests caused errors):
   ```kql
   requests
   | where timestamp > ago(24h)
   | where success == false
   | join kind=leftouter (
       exceptions
       | where timestamp > ago(24h)
   ) on operation_Id
   | project timestamp, requestName=name, url, resultCode, exceptionType=type, exceptionMessage=outerMessage
   | take 50
   ```

2. **Dependency analysis** (slow external calls):
   ```kql
   dependencies
   | where timestamp > ago(1h)
   | summarize avgDuration=avg(duration), failRate=countif(success==false)*100.0/count() by target, name
   | order by avgDuration desc
   ```

3. **Time-based analysis** (requests per hour):
   ```kql
   requests
   | where timestamp > ago(24h)
   | summarize requestCount=count(), avgDuration=avg(duration) by bin(timestamp, 1h)
   | render columnchart
   ```

4. **Custom events** (track business metrics):
   ```kql
   customEvents
   | where timestamp > ago(7d)
   | where name == "UserLogin"
   | summarize logins=count() by bin(timestamp, 1d)
   | render barchart
   ```

5. **Alert query** — error rate spike:
   ```kql
   requests
   | where timestamp > ago(5m)
   | summarize
       totalRequests = count(),
       failedRequests = countif(resultCode >= 500)
   | extend errorRate = 100.0 * failedRequests / totalRequests
   | where errorRate > 5  // Alert if > 5% error rate
   ```

### Part C: Infrastructure Queries

1. **VM CPU usage**:
   ```kql
   Perf
   | where ObjectName == "Processor" and CounterName == "% Processor Time"
   | where InstanceName == "_Total"
   | summarize avgCPU=avg(CounterValue) by bin(TimeGenerated, 5m), Computer
   | render timechart
   ```

2. **VM memory usage**:
   ```kql
   Perf
   | where ObjectName == "Memory" and CounterName == "% Used Memory"
   | summarize avgMemory=avg(CounterValue) by bin(TimeGenerated, 5m), Computer
   | render timechart
   ```

3. **Disk space**:
   ```kql
   Perf
   | where ObjectName == "LogicalDisk" and CounterName == "% Free Space"
   | where InstanceName == "/"
   | summarize minFreeSpace=min(CounterValue) by Computer
   | where minFreeSpace < 20  // Less than 20% free
   ```

4. **Container performance** (AKS):
   ```kql
   KubePodInventory
   | where TimeGenerated > ago(1h)
   | where Namespace == "default"
   | summarize count() by PodStatus
   ```

### Part D: Distributed Tracing

1. **View end-to-end transaction**:
   - Application Insights → Transaction search
   - Click any request → "View all telemetry for this operation"
   - This shows the full trace: request → dependencies → exceptions

2. **Query distributed trace**:
   ```kql
   // Find a specific operation and all its telemetry
   union requests, dependencies, exceptions, traces
   | where operation_Id == "specific-operation-id"
   | order by timestamp asc
   | project timestamp, itemType, name, duration, success, message
   ```

3. **End-to-end latency breakdown**:
   ```kql
   requests
   | where timestamp > ago(1h)
   | where name == "GET /api/orders"
   | project operation_Id, requestDuration=duration
   | join kind=inner (
       dependencies
       | project operation_Id, depName=name, depDuration=duration, depTarget=target
   ) on operation_Id
   | summarize
       avgRequestDuration=avg(requestDuration),
       avgDepDuration=avg(depDuration)
       by depName, depTarget
   | order by avgDepDuration desc
   ```

4. **Application Map** query:
   ```kql
   // Understand service dependencies
   dependencies
   | where timestamp > ago(1h)
   | summarize
       calls=count(),
       failures=countif(success==false),
       avgDuration=avg(duration)
       by target, type
   | extend failRate = round(100.0 * failures / calls, 1)
   | order by calls desc
   ```

### Part E: Create Workbooks

1. In Azure Monitor → Workbooks → New
2. Add a KQL query tile: Request rate over time
3. Add a KQL query tile: Error rate
4. Add a KQL query tile: Top 5 slowest endpoints
5. Add a KQL query tile: Dependency health
6. Save and share with team

## Validation Checklist

- [ ] Basic KQL queries written (requests, exceptions, performance)
- [ ] Join queries used to correlate data
- [ ] Time-series queries with `bin()` and `render`
- [ ] Infrastructure queries for CPU, memory, disk
- [ ] Distributed tracing viewed in Application Insights
- [ ] End-to-end transaction trace query written
- [ ] KQL `summarize`, `where`, `project`, `join`, `extend` operators understood
- [ ] Azure Monitor Workbook created

## KQL Quick Reference

| Operator | Purpose | Example |
|----------|---------|---------|
| `where` | Filter rows | `where duration > 1000` |
| `project` | Select columns | `project name, duration` |
| `summarize` | Aggregate | `summarize count() by name` |
| `order by` | Sort | `order by duration desc` |
| `take` | Limit rows | `take 10` |
| `extend` | Add calculated column | `extend ms = duration/1000` |
| `join` | Combine tables | `join kind=inner other on key` |
| `bin()` | Time bucketing | `bin(timestamp, 1h)` |
| `ago()` | Relative time | `ago(24h)` |
| `render` | Visualize | `render timechart` |
| `count()` | Count rows | `summarize count()` |
| `avg()` | Average | `summarize avg(duration)` |
| `countif()` | Conditional count | `countif(success==false)` |

## Microsoft Learn References
- [KQL quick reference](https://learn.microsoft.com/en-us/azure/data-explorer/kql-quick-reference)
- [Application Insights log queries](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-query-overview)
- [Distributed tracing](https://learn.microsoft.com/en-us/azure/azure-monitor/app/distributed-trace-data)
