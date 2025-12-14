# Feature: Telemetry System

## Overview
The Telemetry System provides real-time and historical data streaming, storage, and retrieval capabilities for monitoring system metrics, sensors, and measurements.

---

## Epic 1: Telemetry Metadata Management

### Story 1.1: Telemetry Metadata Model
**As a** system architect  
**I want** a flexible telemetry metadata model  
**So that** telemetry points can be described with units, formats, and constraints

**Acceptance Criteria:**
- Telemetry metadata includes key, name, units, format
- Metadata supports multiple value types (number, string, enum, boolean)
- Metadata includes min/max ranges and default values
- Metadata supports custom formatters for display
- Metadata is stored as part of domain object
- Metadata supports telemetry point aggregation rules

**Technical Implementation (.NET Core):**
```csharp
// Models/Telemetry/TelemetryMetadata.cs
public class TelemetryMetadata
{
    public string Key { get; set; }
    public string Name { get; set; }
    public TelemetryValueType ValueType { get; set; }
    public string Units { get; set; }
    public string Format { get; set; }
    public object MinValue { get; set; }
    public object MaxValue { get; set; }
    public object DefaultValue { get; set; }
    public List<TelemetryHint> Hints { get; set; }
    public Dictionary<string, object> CustomProperties { get; set; }
}

public enum TelemetryValueType
{
    Number,
    String,
    Boolean,
    Enum,
    Timestamp
}

public class TelemetryHint
{
    public string Key { get; set; }
    public object Value { get; set; }
}
```

**Angular Implementation:**
```typescript
// models/telemetry-metadata.model.ts
export interface TelemetryMetadata {
  key: string;
  name: string;
  valueType: TelemetryValueType;
  units?: string;
  format?: string;
  minValue?: any;
  maxValue?: any;
  defaultValue?: any;
  hints?: TelemetryHint[];
  customProperties?: Record<string, any>;
}

export enum TelemetryValueType {
  Number = 'number',
  String = 'string',
  Boolean = 'boolean',
  Enum = 'enum',
  Timestamp = 'timestamp'
}
```

**Estimated Hours:** 12 hours (Junior Developer)

---

### Story 1.2: Telemetry Provider Registry
**As a** backend developer  
**I want** a registry for telemetry providers  
**So that** multiple data sources can be plugged into the system

**Acceptance Criteria:**
- Registry allows registration of telemetry providers by type
- Registry supports lookup of providers by domain object type
- Registry validates provider implements required interfaces
- Registry supports provider metadata (name, description, version)
- Registry supports hot-reload of providers

**Technical Implementation (.NET Core):**
```csharp
// Services/Telemetry/ITelemetryProvider.cs
public interface ITelemetryProvider
{
    string Name { get; }
    string Description { get; }
    bool SupportsRealtime { get; }
    bool SupportsHistorical { get; }
    
    Task<TelemetryMetadata> GetMetadataAsync(DomainObjectIdentifier id);
    Task<IEnumerable<TelemetryDatum>> GetHistoricalDataAsync(
        DomainObjectIdentifier id, 
        TelemetryRequest request);
    IObservable<TelemetryDatum> SubscribeToRealtime(DomainObjectIdentifier id);
}

// Services/Telemetry/TelemetryProviderRegistry.cs
public class TelemetryProviderRegistry : ITelemetryProviderRegistry
{
    private readonly Dictionary<string, ITelemetryProvider> _providers;
    
    public void RegisterProvider(string objectType, ITelemetryProvider provider)
    {
        if (!provider.SupportsRealtime && !provider.SupportsHistorical)
        {
            throw new ArgumentException("Provider must support at least one mode");
        }
        
        _providers[objectType] = provider;
    }
    
    public ITelemetryProvider GetProvider(string objectType)
    {
        return _providers.TryGetValue(objectType, out var provider) 
            ? provider 
            : null;
    }
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

## Epic 2: Historical Telemetry

### Story 2.1: Telemetry Storage in Azure Data Explorer
**As a** backend developer  
**I want** to store telemetry data in Azure Data Explorer  
**So that** high-volume time-series data can be efficiently queried

**Acceptance Criteria:**
- ADX cluster provisioned with appropriate SKU
- Database and tables created for telemetry storage
- Telemetry data includes timestamp, value, and source identifier
- Support for batch ingestion of telemetry points
- Support for streaming ingestion via Event Hubs
- Data retention policies configured
- Materialized views for common aggregations

**Technical Implementation (.NET Core):**
```csharp
// Services/Telemetry/AdxTelemetryStore.cs
public class AdxTelemetryStore : ITelemetryStore
{
    private readonly ICslAdminProvider _adminProvider;
    private readonly ICslQueryProvider _queryProvider;
    private readonly string _databaseName;
    
    public async Task IngestAsync(IEnumerable<TelemetryDatum> data)
    {
        var csvData = ConvertToCsv(data);
        
        var ingestCommand = CslCommandGenerator.GenerateTableIngestPushCommand(
            "TelemetryData",
            compress: false,
            csvData
        );
        
        await _adminProvider.ExecuteControlCommandAsync(_databaseName, ingestCommand);
    }
    
    public async Task<IEnumerable<TelemetryDatum>> QueryAsync(TelemetryRequest request)
    {
        var kql = $@"
            TelemetryData
            | where ObjectId == '{request.ObjectId}'
            | where Timestamp >= datetime({request.Start})
            | where Timestamp <= datetime({request.End})
            | order by Timestamp asc
            | project Timestamp, Value, Metadata
        ";
        
        var results = await _queryProvider.ExecuteQueryAsync(
            _databaseName, 
            kql, 
            new ClientRequestProperties()
        );
        
        return ParseResults(results);
    }
}

// Models/Telemetry/TelemetryDatum.cs
public class TelemetryDatum
{
    public DomainObjectIdentifier ObjectId { get; set; }
    public DateTime Timestamp { get; set; }
    public object Value { get; set; }
    public Dictionary<string, object> Metadata { get; set; }
}

public class TelemetryRequest
{
    public DomainObjectIdentifier ObjectId { get; set; }
    public DateTime Start { get; set; }
    public DateTime End { get; set; }
    public string Strategy { get; set; } // "latest", "minmax", "all"
    public int? Size { get; set; }
}
```

**Estimated Hours:** 28 hours (Junior Developer)

---

### Story 2.2: Historical Telemetry API Endpoints
**As a** frontend developer  
**I want** API endpoints for querying historical telemetry  
**So that** charts and graphs can display historical data

**Acceptance Criteria:**
- GET /api/telemetry/historical/{namespace}/{key} - query historical data
- Query parameters: start, end, strategy, size
- Response includes timestamp and value arrays
- Support for data decimation/reduction strategies
- Support for pagination of large result sets
- Response includes metadata about the query

**Technical Implementation (.NET Core):**
```csharp
// Controllers/TelemetryController.cs
[ApiController]
[Route("api/telemetry")]
[Authorize]
public class TelemetryController : ControllerBase
{
    private readonly ITelemetryService _service;
    
    [HttpGet("historical/{namespace}/{key}")]
    [ProducesResponseType(typeof(TelemetryResponse), 200)]
    public async Task<IActionResult> GetHistoricalData(
        string @namespace,
        string key,
        [FromQuery] DateTime start,
        [FromQuery] DateTime end,
        [FromQuery] string strategy = "all",
        [FromQuery] int? size = null)
    {
        var request = new TelemetryRequest
        {
            ObjectId = new DomainObjectIdentifier { Namespace = @namespace, Key = key },
            Start = start,
            End = end,
            Strategy = strategy,
            Size = size
        };
        
        var result = await _service.GetHistoricalDataAsync(request);
        
        return Ok(new TelemetryResponse
        {
            Data = result.Select(d => new TelemetryPoint
            {
                Timestamp = d.Timestamp,
                Value = d.Value
            }),
            Metadata = new QueryMetadata
            {
                Start = start,
                End = end,
                Count = result.Count(),
                Strategy = strategy
            }
        });
    }
}

public class TelemetryResponse
{
    public IEnumerable<TelemetryPoint> Data { get; set; }
    public QueryMetadata Metadata { get; set; }
}

public class TelemetryPoint
{
    public DateTime Timestamp { get; set; }
    public object Value { get; set; }
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

### Story 2.3: Angular Historical Telemetry Service
**As a** frontend developer  
**I want** a service for fetching historical telemetry  
**So that** visualization components can easily access historical data

**Acceptance Criteria:**
- Service method accepts telemetry request parameters
- Service returns observable of telemetry points
- Service handles date range validation
- Service caches recent queries
- Service supports cancellation of in-flight requests
- Service retries failed requests with exponential backoff

**Technical Implementation (Angular):**
```typescript
// services/telemetry.service.ts
@Injectable({
  providedIn: 'root'
})
export class TelemetryService {
  private cache = new Map<string, Observable<TelemetryResponse>>();
  
  constructor(private http: HttpClient) {}
  
  getHistoricalData(request: TelemetryRequest): Observable<TelemetryResponse> {
    const cacheKey = this.getCacheKey(request);
    
    if (!this.cache.has(cacheKey)) {
      const params = new HttpParams()
        .set('start', request.start.toISOString())
        .set('end', request.end.toISOString())
        .set('strategy', request.strategy || 'all');
      
      if (request.size) {
        params.set('size', request.size.toString());
      }
      
      const url = `/api/telemetry/historical/${request.objectId.namespace}/${request.objectId.key}`;
      
      const request$ = this.http.get<TelemetryResponse>(url, { params }).pipe(
        retry({
          count: 3,
          delay: (error, retryCount) => timer(Math.pow(2, retryCount) * 1000)
        }),
        shareReplay(1),
        catchError(this.handleError)
      );
      
      this.cache.set(cacheKey, request$);
      
      // Clear cache after 5 minutes
      timer(5 * 60 * 1000).subscribe(() => this.cache.delete(cacheKey));
    }
    
    return this.cache.get(cacheKey)!;
  }
  
  private getCacheKey(request: TelemetryRequest): string {
    return `${request.objectId.namespace}:${request.objectId.key}:${request.start.getTime()}:${request.end.getTime()}`;
  }
  
  private handleError(error: HttpErrorResponse): Observable<never> {
    console.error('Telemetry request failed:', error);
    return throwError(() => new Error('Failed to fetch telemetry data'));
  }
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

## Epic 3: Real-time Telemetry

### Story 3.1: SignalR Hub for Real-time Streaming
**As a** backend developer  
**I want** a SignalR hub for streaming real-time telemetry  
**So that** clients can receive live data updates

**Acceptance Criteria:**
- SignalR hub accepts subscription requests for telemetry points
- Hub broadcasts new telemetry data to subscribed clients
- Hub supports multiple concurrent subscriptions per client
- Hub handles client disconnection gracefully
- Hub supports authentication and authorization
- Hub includes heartbeat/keepalive mechanism

**Technical Implementation (.NET Core):**
```csharp
// Hubs/TelemetryHub.cs
[Authorize]
public class TelemetryHub : Hub
{
    private readonly ITelemetrySubscriptionManager _subscriptionManager;
    private readonly ILogger<TelemetryHub> _logger;
    
    public TelemetryHub(
        ITelemetrySubscriptionManager subscriptionManager,
        ILogger<TelemetryHub> logger)
    {
        _subscriptionManager = subscriptionManager;
        _logger = logger;
    }
    
    public async Task Subscribe(string @namespace, string key)
    {
        var identifier = new DomainObjectIdentifier 
        { 
            Namespace = @namespace, 
            Key = key 
        };
        
        var groupName = GetGroupName(identifier);
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        
        await _subscriptionManager.AddSubscriptionAsync(
            Context.ConnectionId,
            identifier
        );
        
        _logger.LogInformation(
            "Client {ConnectionId} subscribed to {Namespace}:{Key}",
            Context.ConnectionId,
            @namespace,
            key
        );
    }
    
    public async Task Unsubscribe(string @namespace, string key)
    {
        var identifier = new DomainObjectIdentifier 
        { 
            Namespace = @namespace, 
            Key = key 
        };
        
        var groupName = GetGroupName(identifier);
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        
        await _subscriptionManager.RemoveSubscriptionAsync(
            Context.ConnectionId,
            identifier
        );
    }
    
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        await _subscriptionManager.RemoveAllSubscriptionsAsync(Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }
    
    private string GetGroupName(DomainObjectIdentifier id)
    {
        return $"telemetry:{id.Namespace}:{id.Key}";
    }
}

// Services/Telemetry/TelemetrySubscriptionManager.cs
public class TelemetrySubscriptionManager : ITelemetrySubscriptionManager
{
    private readonly ConcurrentDictionary<string, HashSet<DomainObjectIdentifier>> _subscriptions;
    private readonly IHubContext<TelemetryHub> _hubContext;
    
    public async Task AddSubscriptionAsync(string connectionId, DomainObjectIdentifier id)
    {
        _subscriptions.AddOrUpdate(
            connectionId,
            new HashSet<DomainObjectIdentifier> { id },
            (_, set) => { set.Add(id); return set; }
        );
    }
    
    public async Task BroadcastTelemetryAsync(
        DomainObjectIdentifier id, 
        TelemetryDatum datum)
    {
        var groupName = $"telemetry:{id.Namespace}:{id.Key}";
        
        await _hubContext.Clients.Group(groupName).SendAsync(
            "telemetryUpdate",
            new
            {
                id = id,
                timestamp = datum.Timestamp,
                value = datum.Value
            }
        );
    }
}
```

**Estimated Hours:** 24 hours (Junior Developer)

---

### Story 3.2: Telemetry Data Generator/Simulator
**As a** developer  
**I want** a telemetry simulator for testing  
**So that** I can develop without real data sources

**Acceptance Criteria:**
- Simulator generates random telemetry data
- Simulator supports sine wave, random walk, and constant patterns
- Simulator publishes data at configurable intervals
- Simulator can simulate multiple telemetry points
- Simulator can be started/stopped via API
- Simulator supports realistic value ranges and units

**Technical Implementation (.NET Core):**
```csharp
// Services/Telemetry/TelemetrySimulator.cs
public class TelemetrySimulator : BackgroundService
{
    private readonly ITelemetrySubscriptionManager _subscriptionManager;
    private readonly ILogger<TelemetrySimulator> _logger;
    private readonly List<SimulatedTelemetryPoint> _points;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            foreach (var point in _points)
            {
                var value = point.Generator.GenerateNext();
                
                var datum = new TelemetryDatum
                {
                    ObjectId = point.Identifier,
                    Timestamp = DateTime.UtcNow,
                    Value = value
                };
                
                await _subscriptionManager.BroadcastTelemetryAsync(
                    point.Identifier,
                    datum
                );
            }
            
            await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
        }
    }
}

// Services/Telemetry/Generators/SineWaveGenerator.cs
public class SineWaveGenerator : ITelemetryValueGenerator
{
    private readonly double _amplitude;
    private readonly double _period;
    private readonly double _offset;
    private int _step;
    
    public SineWaveGenerator(double amplitude, double period, double offset)
    {
        _amplitude = amplitude;
        _period = period;
        _offset = offset;
    }
    
    public object GenerateNext()
    {
        var radians = 2 * Math.PI * (_step / _period);
        var value = _offset + _amplitude * Math.Sin(radians);
        _step++;
        return value;
    }
}
```

**Estimated Hours:** 20 hours (Junior Developer)

---

### Story 3.3: Angular SignalR Integration
**As a** frontend developer  
**I want** SignalR integration in Angular  
**So that** components can receive real-time telemetry updates

**Acceptance Criteria:**
- Service establishes SignalR connection on initialization
- Service provides method to subscribe to telemetry points
- Service emits observable streams for subscribed points
- Service handles reconnection on connection loss
- Service properly cleans up subscriptions
- Service supports multiple simultaneous subscriptions

**Technical Implementation (Angular):**
```typescript
// services/telemetry-realtime.service.ts
@Injectable({
  providedIn: 'root'
})
export class TelemetryRealtimeService implements OnDestroy {
  private hubConnection: signalR.HubConnection;
  private subjects = new Map<string, Subject<TelemetryPoint>>();
  private subscriptionCounts = new Map<string, number>();
  
  constructor() {
    this.initializeConnection();
  }
  
  private initializeConnection(): void {
    this.hubConnection = new signalR.HubConnectionBuilder()
      .withUrl('/hubs/telemetry', {
        accessTokenFactory: () => this.getAccessToken()
      })
      .withAutomaticReconnect({
        nextRetryDelayInMilliseconds: (retryContext) => {
          return Math.min(1000 * Math.pow(2, retryContext.previousRetryCount), 30000);
        }
      })
      .build();
    
    this.hubConnection.on('telemetryUpdate', (data: any) => {
      const key = `${data.id.namespace}:${data.id.key}`;
      const subject = this.subjects.get(key);
      
      if (subject) {
        subject.next({
          timestamp: new Date(data.timestamp),
          value: data.value
        });
      }
    });
    
    this.hubConnection.start().catch(err => {
      console.error('Failed to start SignalR connection:', err);
    });
  }
  
  subscribe(identifier: DomainObjectIdentifier): Observable<TelemetryPoint> {
    const key = `${identifier.namespace}:${identifier.key}`;
    
    if (!this.subjects.has(key)) {
      this.subjects.set(key, new Subject<TelemetryPoint>());
      this.subscriptionCounts.set(key, 0);
    }
    
    const count = this.subscriptionCounts.get(key)! + 1;
    this.subscriptionCounts.set(key, count);
    
    if (count === 1) {
      this.hubConnection.invoke('Subscribe', identifier.namespace, identifier.key)
        .catch(err => console.error('Failed to subscribe:', err));
    }
    
    return this.subjects.get(key)!.asObservable().pipe(
      finalize(() => this.unsubscribe(identifier))
    );
  }
  
  private unsubscribe(identifier: DomainObjectIdentifier): void {
    const key = `${identifier.namespace}:${identifier.key}`;
    const count = this.subscriptionCounts.get(key)! - 1;
    
    this.subscriptionCounts.set(key, count);
    
    if (count === 0) {
      this.hubConnection.invoke('Unsubscribe', identifier.namespace, identifier.key)
        .catch(err => console.error('Failed to unsubscribe:', err));
      
      this.subjects.get(key)?.complete();
      this.subjects.delete(key);
      this.subscriptionCounts.delete(key);
    }
  }
  
  private getAccessToken(): string {
    // Return JWT token from auth service
    return localStorage.getItem('access_token') || '';
  }
  
  ngOnDestroy(): void {
    this.hubConnection.stop();
    this.subjects.forEach(subject => subject.complete());
  }
}
```

**Estimated Hours:** 20 hours (Junior Developer)

---

## Epic 4: Unit Testing

### Story 4.1: Telemetry Service Unit Tests
**As a** developer  
**I want** unit tests for telemetry services  
**So that** data retrieval logic is validated

**Acceptance Criteria:**
- Tests for ADX query generation
- Tests for data transformation
- Tests for error handling
- Tests for subscription management
- Mock ADX provider for testing
- Test coverage > 80%

**Technical Implementation:**
```csharp
// Tests/Services/AdxTelemetryStoreTests.cs
public class AdxTelemetryStoreTests
{
    private readonly Mock<ICslQueryProvider> _queryProviderMock;
    private readonly Mock<ICslAdminProvider> _adminProviderMock;
    private readonly AdxTelemetryStore _sut;
    
    public AdxTelemetryStoreTests()
    {
        _queryProviderMock = new Mock<ICslQueryProvider>();
        _adminProviderMock = new Mock<ICslAdminProvider>();
        _sut = new AdxTelemetryStore(_queryProviderMock.Object, _adminProviderMock.Object, "TestDb");
    }
    
    [Fact]
    public async Task QueryAsync_WithValidRequest_ReturnsData()
    {
        // Arrange
        var request = new TelemetryRequest
        {
            ObjectId = new DomainObjectIdentifier { Namespace = "test", Key = "1" },
            Start = DateTime.UtcNow.AddHours(-1),
            End = DateTime.UtcNow
        };
        
        var mockResults = CreateMockDataReader();
        _queryProviderMock.Setup(x => x.ExecuteQueryAsync(
            It.IsAny<string>(),
            It.IsAny<string>(),
            It.IsAny<ClientRequestProperties>()
        )).ReturnsAsync(mockResults);
        
        // Act
        var results = await _sut.QueryAsync(request);
        
        // Assert
        results.Should().NotBeEmpty();
        results.Should().AllSatisfy(d => 
        {
            d.Timestamp.Should().BeOnOrAfter(request.Start);
            d.Timestamp.Should().BeOnOrBefore(request.End);
        });
    }
    
    [Fact]
    public async Task IngestAsync_WithValidData_ExecutesCommand()
    {
        // Arrange
        var data = new List<TelemetryDatum>
        {
            new TelemetryDatum
            {
                ObjectId = new DomainObjectIdentifier { Namespace = "test", Key = "1" },
                Timestamp = DateTime.UtcNow,
                Value = 42.5
            }
        };
        
        // Act
        await _sut.IngestAsync(data);
        
        // Assert
        _adminProviderMock.Verify(x => x.ExecuteControlCommandAsync(
            "TestDb",
            It.IsAny<string>()
        ), Times.Once);
    }
}
```

**Estimated Hours:** 24 hours (Junior Developer)

---

### Story 4.2: SignalR Hub Unit Tests
**As a** developer  
**I want** unit tests for SignalR hub  
**So that** subscription logic is validated

**Acceptance Criteria:**
- Tests for subscribe/unsubscribe operations
- Tests for client disconnection handling
- Tests for broadcast functionality
- Mock hub context for testing
- Test coverage > 80%

**Technical Implementation:**
```csharp
// Tests/Hubs/TelemetryHubTests.cs
public class TelemetryHubTests
{
    private readonly Mock<ITelemetrySubscriptionManager> _subscriptionManagerMock;
    private readonly Mock<HubCallerContext> _contextMock;
    private readonly Mock<IGroupManager> _groupManagerMock;
    private readonly TelemetryHub _sut;
    
    public TelemetryHubTests()
    {
        _subscriptionManagerMock = new Mock<ITelemetrySubscriptionManager>();
        _contextMock = new Mock<HubCallerContext>();
        _groupManagerMock = new Mock<IGroupManager>();
        
        _sut = new TelemetryHub(_subscriptionManagerMock.Object, Mock.Of<ILogger<TelemetryHub>>())
        {
            Context = _contextMock.Object,
            Groups = _groupManagerMock.Object
        };
        
        _contextMock.Setup(x => x.ConnectionId).Returns("test-connection-id");
    }
    
    [Fact]
    public async Task Subscribe_AddsToGroupAndRegisters()
    {
        // Arrange
        var @namespace = "test";
        var key = "123";
        
        // Act
        await _sut.Subscribe(@namespace, key);
        
        // Assert
        _groupManagerMock.Verify(x => x.AddToGroupAsync(
            "test-connection-id",
            "telemetry:test:123",
            default
        ), Times.Once);
        
        _subscriptionManagerMock.Verify(x => x.AddSubscriptionAsync(
            "test-connection-id",
            It.Is<DomainObjectIdentifier>(id => id.Namespace == @namespace && id.Key == key)
        ), Times.Once);
    }
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

### Story 4.3: Angular Telemetry Service Unit Tests
**As a** developer  
**I want** unit tests for Angular telemetry services  
**So that** frontend data handling is validated

**Acceptance Criteria:**
- Tests for HTTP request formation
- Tests for response parsing
- Tests for caching behavior
- Tests for error handling
- Mock HttpClient for testing
- Test coverage > 80%

**Technical Implementation:**
```typescript
// services/telemetry.service.spec.ts
describe('TelemetryService', () => {
  let service: TelemetryService;
  let httpMock: HttpTestingController;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [TelemetryService]
    });
    
    service = TestBed.inject(TelemetryService);
    httpMock = TestBed.inject(HttpTestingController);
  });
  
  afterEach(() => {
    httpMock.verify();
  });
  
  it('should fetch historical data', (done) => {
    const request: TelemetryRequest = {
      objectId: { namespace: 'test', key: '123' },
      start: new Date('2024-01-01'),
      end: new Date('2024-01-02'),
      strategy: 'all'
    };
    
    const mockResponse: TelemetryResponse = {
      data: [
        { timestamp: new Date('2024-01-01T12:00:00Z'), value: 42 }
      ],
      metadata: {
        start: request.start,
        end: request.end,
        count: 1,
        strategy: 'all'
      }
    };
    
    service.getHistoricalData(request).subscribe(response => {
      expect(response.data.length).toBe(1);
      expect(response.data[0].value).toBe(42);
      done();
    });
    
    const req = httpMock.expectOne(r => 
      r.url.includes('/api/telemetry/historical/test/123')
    );
    expect(req.request.method).toBe('GET');
    expect(req.request.params.get('start')).toBeTruthy();
    req.flush(mockResponse);
  });
  
  it('should cache repeated requests', () => {
    const request: TelemetryRequest = {
      objectId: { namespace: 'test', key: '123' },
      start: new Date('2024-01-01'),
      end: new Date('2024-01-02'),
      strategy: 'all'
    };
    
    // First request
    service.getHistoricalData(request).subscribe();
    const req1 = httpMock.expectOne(r => r.url.includes('/api/telemetry/historical'));
    req1.flush({ data: [], metadata: {} });
    
    // Second request should be cached (no HTTP request)
    service.getHistoricalData(request).subscribe();
    httpMock.expectNone(r => r.url.includes('/api/telemetry/historical'));
  });
});
```

**Estimated Hours:** 20 hours (Junior Developer)

---

## Epic 5: Integration Testing

### Story 5.1: End-to-End Telemetry Flow Tests
**As a** QA engineer  
**I want** integration tests for complete telemetry flow  
**So that** data flows correctly from ingestion to retrieval

**Acceptance Criteria:**
- Tests ingest telemetry data into ADX
- Tests query telemetry data via API
- Tests verify data integrity through pipeline
- Tests use test ADX cluster or emulator
- Tests clean up test data after execution

**Technical Implementation:**
```csharp
// Tests/Integration/TelemetryIntegrationTests.cs
[Collection("Sequential")]
public class TelemetryIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    private readonly IAdxTestHelper _adxHelper;
    
    [Fact]
    public async Task CompleteFlow_IngestAndQuery_ReturnsData()
    {
        // Arrange
        var testData = new List<TelemetryDatum>
        {
            new TelemetryDatum
            {
                ObjectId = new DomainObjectIdentifier { Namespace = "test", Key = "int-test-1" },
                Timestamp = DateTime.UtcNow.AddMinutes(-5),
                Value = 100.5
            }
        };
        
        // Act - Ingest
        var ingestResponse = await _client.PostAsJsonAsync("/api/telemetry/ingest", testData);
        ingestResponse.EnsureSuccessStatusCode();
        
        // Wait for ADX ingestion
        await Task.Delay(TimeSpan.FromSeconds(10));
        
        // Act - Query
        var queryUrl = $"/api/telemetry/historical/test/int-test-1" +
                       $"?start={DateTime.UtcNow.AddHours(-1):O}" +
                       $"&end={DateTime.UtcNow:O}";
        var queryResponse = await _client.GetAsync(queryUrl);
        queryResponse.EnsureSuccessStatusCode();
        
        var result = await queryResponse.Content.ReadFromJsonAsync<TelemetryResponse>();
        
        // Assert
        result.Should().NotBeNull();
        result!.Data.Should().Contain(d => Math.Abs((double)d.Value - 100.5) < 0.01);
    }
}
```

**Estimated Hours:** 20 hours (Junior Developer)

---

## Epic 6: E2E Testing with Playwright

### Story 6.1: Real-time Telemetry Display E2E Tests
**As a** QA engineer  
**I want** E2E tests for real-time telemetry visualization  
**So that** live data updates are validated in the UI

**Acceptance Criteria:**
- Tests verify chart updates with new data
- Tests verify SignalR connection establishment
- Tests verify reconnection on connection loss
- Tests verify multiple simultaneous subscriptions
- Tests run against deployed application

**Technical Implementation:**
```typescript
// e2e/telemetry-realtime.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Real-time Telemetry', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:4200/telemetry/test/sine-wave-1');
    await page.waitForSelector('app-telemetry-chart');
  });
  
  test('should display live telemetry updates', async ({ page }) => {
    // Wait for initial connection
    await page.waitForSelector('.connection-status.connected');
    
    // Get initial value
    const initialValue = await page.locator('.current-value').textContent();
    
    // Wait for value to change (indicating real-time update)
    await expect(page.locator('.current-value')).not.toHaveText(initialValue!, {
      timeout: 5000
    });
  });
  
  test('should reconnect on connection loss', async ({ page, context }) => {
    // Simulate network offline
    await context.setOffline(true);
    
    // Verify disconnected state
    await expect(page.locator('.connection-status')).toHaveClass(/disconnected/);
    
    // Restore network
    await context.setOffline(false);
    
    // Verify reconnection
    await expect(page.locator('.connection-status')).toHaveClass(/connected/, {
      timeout: 10000
    });
  });
  
  test('should handle multiple subscriptions', async ({ page }) => {
    // Subscribe to first telemetry point
    await page.click('[data-testid="add-telemetry"]');
    await page.selectOption('select', 'test:sine-wave-1');
    
    // Subscribe to second telemetry point
    await page.click('[data-testid="add-telemetry"]');
    await page.selectOption('select', 'test:sine-wave-2');
    
    // Verify both are updating
    const charts = page.locator('app-telemetry-chart');
    await expect(charts).toHaveCount(2);
    
    // Verify both show live data
    for (let i = 0; i < 2; i++) {
      const chart = charts.nth(i);
      const initialValue = await chart.locator('.current-value').textContent();
      await expect(chart.locator('.current-value')).not.toHaveText(initialValue!, {
        timeout: 5000
      });
    }
  });
});
```

**Estimated Hours:** 24 hours (Junior Developer)

---

## Epic 7: Azure Deployment

### Story 7.1: Azure Data Explorer Provisioning
**As a** DevOps engineer  
**I want** Bicep templates for ADX cluster  
**So that** telemetry storage can be deployed consistently

**Acceptance Criteria:**
- Bicep template creates ADX cluster
- Template creates database and tables
- Template configures retention policies
- Template creates materialized views
- Template supports different SKUs per environment

**Technical Implementation:**
```bicep
// infrastructure/adx.bicep
param location string = resourceGroup().location
param environment string
param clusterName string

var skuMap = {
  dev: {
    name: 'Dev(No SLA)_Standard_D11_v2'
    tier: 'Basic'
    capacity: 1
  }
  prod: {
    name: 'Standard_D16d_v5'
    tier: 'Standard'
    capacity: 2
  }
}

resource adxCluster 'Microsoft.Kusto/clusters@2023-08-15' = {
  name: clusterName
  location: location
  sku: skuMap[environment]
  properties: {
    enableStreamingIngest: true
    enablePurge: true
  }
}

resource database 'Microsoft.Kusto/clusters/databases@2023-08-15' = {
  parent: adxCluster
  name: 'TelemetryDb'
  location: location
  kind: 'ReadWrite'
  properties: {
    softDeletePeriod: 'P365D'
    hotCachePeriod: 'P31D'
  }
}

resource telemetryTable 'Microsoft.Kusto/clusters/databases/scripts@2023-08-15' = {
  parent: database
  name: 'create-telemetry-table'
  properties: {
    scriptContent: '''
      .create table TelemetryData (
        ObjectId: string,
        Timestamp: datetime,
        Value: dynamic,
        Metadata: dynamic
      )
      
      .create table TelemetryData ingestion json mapping 'TelemetryMapping'
      '['
      '  {"column":"ObjectId","path":"$.objectId","datatype":"string"},'
      '  {"column":"Timestamp","path":"$.timestamp","datatype":"datetime"},'
      '  {"column":"Value","path":"$.value","datatype":"dynamic"},'
      '  {"column":"Metadata","path":"$.metadata","datatype":"dynamic"}'
      ']'
    '''
  }
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

### Story 7.2: Event Hubs for Streaming Ingestion
**As a** backend developer  
**I want** Event Hubs integration for streaming telemetry  
**So that** high-volume data can be ingested efficiently

**Acceptance Criteria:**
- Event Hub namespace and hub created
- Connection to ADX data connection configured
- Dead letter queue for failed messages
- Monitoring and alerting configured
- Auto-scaling enabled

**Technical Implementation:**
```bicep
// infrastructure/event-hub.bicep
param location string = resourceGroup().location
param namespaceName string
param hubName string = 'telemetry-stream'

resource eventHubNamespace 'Microsoft.EventHub/namespaces@2023-01-01-preview' = {
  name: namespaceName
  location: location
  sku: {
    name: 'Standard'
    tier: 'Standard'
    capacity: 1
  }
  properties: {
    isAutoInflateEnabled: true
    maximumThroughputUnits: 20
  }
}

resource eventHub 'Microsoft.EventHub/namespaces/eventhubs@2023-01-01-preview' = {
  parent: eventHubNamespace
  name: hubName
  properties: {
    messageRetentionInDays: 7
    partitionCount: 4
  }
}

resource consumerGroup 'Microsoft.EventHub/namespaces/eventhubs/consumergroups@2023-01-01-preview' = {
  parent: eventHub
  name: 'adx-ingestion'
}
```

**Estimated Hours:** 12 hours (Junior Developer)

---

## Summary of Hours for Telemetry System Feature

| Epic | Story Count | Total Hours |
|------|-------------|-------------|
| Epic 1: Metadata Management | 2 | 28 |
| Epic 2: Historical Telemetry | 3 | 60 |
| Epic 3: Real-time Telemetry | 3 | 64 |
| Epic 4: Unit Testing | 3 | 60 |
| Epic 5: Integration Testing | 1 | 20 |
| Epic 6: E2E Testing | 1 | 24 |
| Epic 7: Azure Deployment | 2 | 28 |
| **Total** | **15** | **284** |

**Note:** These estimates are for a junior developer. An experienced developer could complete these stories 30-50% faster.
