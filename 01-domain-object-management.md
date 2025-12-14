# Feature: Domain Object Management

## Overview
Domain Object Management provides the core framework for representing all entities in the system (telemetry points, displays, folders, layouts, etc.) with a hierarchical tree structure and persistent storage.

---

## Epic 1: Domain Object Core Infrastructure

### Story 1.1: Domain Object Model
**As a** system architect  
**I want** a flexible domain object model with metadata support  
**So that** all entities in the system can be represented uniformly

**Acceptance Criteria:**
- Domain object has identifier (namespace + key)
- Domain object has type, name, and location properties
- Domain object supports custom metadata via dictionary
- Domain object model is serializable to JSON
- Domain object supports composition (parent-child relationships)
- Domain object has created/modified timestamps
- Domain object has creator/modifier user tracking

**Technical Implementation (.NET Core):**
```csharp
// Models/DomainObjects/DomainObject.cs
public class DomainObject
{
    public DomainObjectIdentifier Identifier { get; set; }
    public string Type { get; set; }
    public string Name { get; set; }
    public string Location { get; set; }
    public DateTime CreatedOn { get; set; }
    public DateTime ModifiedOn { get; set; }
    public string CreatedBy { get; set; }
    public string ModifiedBy { get; set; }
    public List<DomainObjectIdentifier> Composition { get; set; }
    public Dictionary<string, object> Metadata { get; set; }
}

public class DomainObjectIdentifier
{
    public string Namespace { get; set; }
    public string Key { get; set; }
}
```

**Angular Implementation:**
```typescript
// models/domain-object.model.ts
export interface DomainObject {
  identifier: DomainObjectIdentifier;
  type: string;
  name: string;
  location: string;
  createdOn: Date;
  modifiedOn: Date;
  createdBy: string;
  modifiedBy: string;
  composition: DomainObjectIdentifier[];
  metadata: Record<string, any>;
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

### Story 1.2: Domain Object Repository
**As a** backend developer  
**I want** a repository pattern for domain object persistence  
**So that** domain objects can be stored and retrieved efficiently

**Acceptance Criteria:**
- Repository interface defines CRUD operations
- Azure Cosmos DB implementation for storage
- Support for querying by identifier, type, and parent
- Support for bulk operations
- Optimistic concurrency control with ETags
- Audit trail for all changes
- Soft delete support

**Technical Implementation (.NET Core):**
```csharp
// Repositories/Interfaces/IDomainObjectRepository.cs
public interface IDomainObjectRepository
{
    Task<DomainObject> GetAsync(DomainObjectIdentifier id);
    Task<DomainObject> CreateAsync(DomainObject obj);
    Task<DomainObject> UpdateAsync(DomainObject obj);
    Task DeleteAsync(DomainObjectIdentifier id);
    Task<IEnumerable<DomainObject>> GetByTypeAsync(string type);
    Task<IEnumerable<DomainObject>> GetCompositionAsync(DomainObjectIdentifier parentId);
    Task<IEnumerable<DomainObject>> SearchAsync(string query);
}

// Repositories/CosmosDbDomainObjectRepository.cs
public class CosmosDbDomainObjectRepository : IDomainObjectRepository
{
    private readonly Container _container;
    
    // Implementation using Cosmos DB SDK
}
```

**Testing Requirements:**
- Unit tests for repository interface
- Integration tests with Cosmos DB Emulator
- Performance tests for bulk operations

**Estimated Hours:** 24 hours (Junior Developer)

---

### Story 1.3: Domain Object Service Layer
**As a** backend developer  
**I want** a service layer for domain object business logic  
**So that** complex operations are encapsulated and testable

**Acceptance Criteria:**
- Service validates domain object before persistence
- Service enforces business rules (naming conventions, type restrictions)
- Service handles composition validation (no circular references)
- Service publishes events for object lifecycle changes
- Service supports transactions for multi-object operations
- Service implements caching strategy for frequently accessed objects

**Technical Implementation (.NET Core):**
```csharp
// Services/Interfaces/IDomainObjectService.cs
public interface IDomainObjectService
{
    Task<Result<DomainObject>> CreateAsync(CreateDomainObjectRequest request);
    Task<Result<DomainObject>> UpdateAsync(UpdateDomainObjectRequest request);
    Task<Result> DeleteAsync(DomainObjectIdentifier id);
    Task<Result<DomainObject>> GetAsync(DomainObjectIdentifier id);
    Task<Result<IEnumerable<DomainObject>>> GetCompositionAsync(DomainObjectIdentifier parentId);
}

// Services/DomainObjectService.cs
public class DomainObjectService : IDomainObjectService
{
    private readonly IDomainObjectRepository _repository;
    private readonly IValidator<CreateDomainObjectRequest> _createValidator;
    private readonly IMemoryCache _cache;
    private readonly IEventPublisher _eventPublisher;
    
    // Implementation with validation, caching, and events
}
```

**Estimated Hours:** 20 hours (Junior Developer)

---

### Story 1.4: Domain Object API Endpoints
**As a** frontend developer  
**I want** RESTful API endpoints for domain objects  
**So that** the Angular application can manage domain objects

**Acceptance Criteria:**
- GET /api/objects/{namespace}/{key} - retrieve object
- POST /api/objects - create object
- PUT /api/objects/{namespace}/{key} - update object
- DELETE /api/objects/{namespace}/{key} - delete object
- GET /api/objects/{namespace}/{key}/composition - get children
- GET /api/objects/search?q={query} - search objects
- All endpoints return proper HTTP status codes
- API uses standard REST conventions
- API supports CORS for Angular app
- API includes Swagger/OpenAPI documentation

**Technical Implementation (.NET Core):**
```csharp
// Controllers/DomainObjectsController.cs
[ApiController]
[Route("api/objects")]
[Authorize]
public class DomainObjectsController : ControllerBase
{
    private readonly IDomainObjectService _service;
    
    [HttpGet("{namespace}/{key}")]
    [ProducesResponseType(typeof(DomainObject), 200)]
    [ProducesResponseType(404)]
    public async Task<IActionResult> GetObject(string @namespace, string key)
    {
        var result = await _service.GetAsync(new DomainObjectIdentifier 
        { 
            Namespace = @namespace, 
            Key = key 
        });
        
        return result.Match<IActionResult>(
            success => Ok(success),
            error => NotFound(error)
        );
    }
    
    // Additional endpoints...
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

### Story 1.5: Angular Domain Object Service
**As a** frontend developer  
**I want** an Angular service for domain object operations  
**So that** components can easily interact with domain objects

**Acceptance Criteria:**
- Service methods for all CRUD operations
- Service uses HttpClient with proper error handling
- Service implements retry logic for failed requests
- Service caches frequently accessed objects
- Service exposes observables for reactive programming
- Service includes proper TypeScript typing

**Technical Implementation (Angular):**
```typescript
// services/domain-object.service.ts
@Injectable({
  providedIn: 'root'
})
export class DomainObjectService {
  private cache = new Map<string, Observable<DomainObject>>();
  
  constructor(private http: HttpClient) {}
  
  get(identifier: DomainObjectIdentifier): Observable<DomainObject> {
    const cacheKey = `${identifier.namespace}:${identifier.key}`;
    
    if (!this.cache.has(cacheKey)) {
      const request = this.http.get<DomainObject>(
        `/api/objects/${identifier.namespace}/${identifier.key}`
      ).pipe(
        retry(3),
        shareReplay(1),
        catchError(this.handleError)
      );
      
      this.cache.set(cacheKey, request);
    }
    
    return this.cache.get(cacheKey)!;
  }
  
  create(request: CreateDomainObjectRequest): Observable<DomainObject> {
    return this.http.post<DomainObject>('/api/objects', request).pipe(
      tap(() => this.invalidateCache()),
      catchError(this.handleError)
    );
  }
  
  // Additional methods...
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

## Epic 2: Object Tree Visualization

### Story 2.1: Tree Component
**As a** user  
**I want** to see all domain objects in a hierarchical tree  
**So that** I can navigate and organize my objects

**Acceptance Criteria:**
- Tree displays all root-level objects
- Tree supports expand/collapse for objects with composition
- Tree shows object icons based on type
- Tree supports lazy loading of children
- Tree supports drag and drop for reorganization
- Tree highlights currently selected object
- Tree supports keyboard navigation
- Tree shows loading indicators during fetch

**Technical Implementation (Angular):**
```typescript
// components/object-tree/object-tree.component.ts
@Component({
  selector: 'app-object-tree',
  template: `
    <mat-tree [dataSource]="dataSource" [treeControl]="treeControl">
      <mat-tree-node *matTreeNodeDef="let node" matTreeNodePadding>
        <button mat-icon-button disabled></button>
        <mat-icon>{{ getIcon(node.type) }}</mat-icon>
        <span (click)="selectNode(node)">{{ node.name }}</span>
      </mat-tree-node>
      
      <mat-tree-node *matTreeNodeDef="let node; when: hasChild" matTreeNodePadding>
        <button mat-icon-button 
                [attr.aria-label]="'Toggle ' + node.name"
                (click)="treeControl.toggle(node)">
          <mat-icon>
            {{ treeControl.isExpanded(node) ? 'expand_more' : 'chevron_right' }}
          </mat-icon>
        </button>
        <mat-icon>{{ getIcon(node.type) }}</mat-icon>
        <span (click)="selectNode(node)">{{ node.name }}</span>
      </mat-tree-node>
    </mat-tree>
  `
})
export class ObjectTreeComponent implements OnInit {
  treeControl = new FlatTreeControl<TreeNode>(
    node => node.level,
    node => node.expandable
  );
  
  dataSource: MatTreeFlatDataSource<DomainObject, TreeNode>;
  
  constructor(private domainObjectService: DomainObjectService) {}
  
  ngOnInit(): void {
    this.loadTree();
  }
  
  private loadTree(): void {
    // Implementation
  }
}
```

**Estimated Hours:** 24 hours (Junior Developer)

---

### Story 2.2: Object Creation Dialog
**As a** user  
**I want** to create new domain objects through a dialog  
**So that** I can add new objects to my tree

**Acceptance Criteria:**
- Dialog allows selecting object type from dropdown
- Dialog has form fields for name and location
- Dialog validates input before submission
- Dialog shows error messages for validation failures
- Dialog supports creating objects under selected parent
- Dialog closes and refreshes tree after successful creation

**Technical Implementation (Angular):**
```typescript
// components/create-object-dialog/create-object-dialog.component.ts
@Component({
  selector: 'app-create-object-dialog',
  template: `
    <h2 mat-dialog-title>Create New Object</h2>
    <mat-dialog-content>
      <form [formGroup]="form">
        <mat-form-field>
          <mat-label>Type</mat-label>
          <mat-select formControlName="type">
            <mat-option *ngFor="let type of objectTypes" [value]="type.key">
              {{ type.name }}
            </mat-option>
          </mat-select>
        </mat-form-field>
        
        <mat-form-field>
          <mat-label>Name</mat-label>
          <input matInput formControlName="name" required>
          <mat-error *ngIf="form.get('name')?.hasError('required')">
            Name is required
          </mat-error>
        </mat-form-field>
      </form>
    </mat-dialog-content>
    <mat-dialog-actions>
      <button mat-button (click)="cancel()">Cancel</button>
      <button mat-raised-button color="primary" 
              (click)="create()" 
              [disabled]="!form.valid">
        Create
      </button>
    </mat-dialog-actions>
  `
})
export class CreateObjectDialogComponent {
  form: FormGroup;
  objectTypes = [
    { key: 'folder', name: 'Folder' },
    { key: 'telemetry', name: 'Telemetry Point' },
    // ...more types
  ];
  
  constructor(
    private fb: FormBuilder,
    private dialogRef: MatDialogRef<CreateObjectDialogComponent>,
    private domainObjectService: DomainObjectService,
    @Inject(MAT_DIALOG_DATA) public data: { parent?: DomainObjectIdentifier }
  ) {
    this.form = this.fb.group({
      type: ['', Validators.required],
      name: ['', Validators.required]
    });
  }
  
  create(): void {
    if (this.form.valid) {
      const request: CreateDomainObjectRequest = {
        ...this.form.value,
        parent: this.data.parent
      };
      
      this.domainObjectService.create(request).subscribe({
        next: (obj) => this.dialogRef.close(obj),
        error: (err) => console.error(err)
      });
    }
  }
  
  cancel(): void {
    this.dialogRef.close();
  }
}
```

**Estimated Hours:** 20 hours (Junior Developer)

---

## Epic 3: Unit Testing

### Story 3.1: Backend Unit Tests
**As a** developer  
**I want** comprehensive unit tests for backend components  
**So that** code quality and correctness are ensured

**Acceptance Criteria:**
- Test coverage > 80% for services and repositories
- Tests use xUnit framework
- Tests use Moq for mocking dependencies
- Tests use FluentAssertions for readable assertions
- Tests follow AAA (Arrange-Act-Assert) pattern
- Tests are fast (< 100ms each)

**Technical Implementation:**
```csharp
// Tests/Services/DomainObjectServiceTests.cs
public class DomainObjectServiceTests
{
    private readonly Mock<IDomainObjectRepository> _repositoryMock;
    private readonly Mock<IValidator<CreateDomainObjectRequest>> _validatorMock;
    private readonly DomainObjectService _sut;
    
    public DomainObjectServiceTests()
    {
        _repositoryMock = new Mock<IDomainObjectRepository>();
        _validatorMock = new Mock<IValidator<CreateDomainObjectRequest>>();
        _sut = new DomainObjectService(_repositoryMock.Object, _validatorMock.Object);
    }
    
    [Fact]
    public async Task CreateAsync_WithValidRequest_ReturnsSuccess()
    {
        // Arrange
        var request = new CreateDomainObjectRequest { Name = "Test", Type = "folder" };
        var expectedObject = new DomainObject { Name = "Test" };
        
        _validatorMock.Setup(x => x.ValidateAsync(request, default))
            .ReturnsAsync(new ValidationResult());
        _repositoryMock.Setup(x => x.CreateAsync(It.IsAny<DomainObject>()))
            .ReturnsAsync(expectedObject);
        
        // Act
        var result = await _sut.CreateAsync(request);
        
        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().Be(expectedObject);
    }
    
    // More tests...
}
```

**Estimated Hours:** 32 hours (Junior Developer)

---

### Story 3.2: Frontend Unit Tests
**As a** developer  
**I want** comprehensive unit tests for Angular components  
**So that** UI behavior is predictable and reliable

**Acceptance Criteria:**
- Test coverage > 80% for components and services
- Tests use Jasmine and Karma
- Tests use Angular Testing utilities
- Tests mock HTTP requests with HttpClientTestingModule
- Tests verify component rendering and user interactions

**Technical Implementation:**
```typescript
// services/domain-object.service.spec.ts
describe('DomainObjectService', () => {
  let service: DomainObjectService;
  let httpMock: HttpTestingController;
  
  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [DomainObjectService]
    });
    
    service = TestBed.inject(DomainObjectService);
    httpMock = TestBed.inject(HttpTestingController);
  });
  
  afterEach(() => {
    httpMock.verify();
  });
  
  it('should retrieve domain object by identifier', (done) => {
    const mockObject: DomainObject = {
      identifier: { namespace: 'test', key: '123' },
      name: 'Test Object',
      type: 'folder'
    };
    
    service.get({ namespace: 'test', key: '123' }).subscribe(obj => {
      expect(obj).toEqual(mockObject);
      done();
    });
    
    const req = httpMock.expectOne('/api/objects/test/123');
    expect(req.request.method).toBe('GET');
    req.flush(mockObject);
  });
  
  // More tests...
});
```

**Estimated Hours:** 32 hours (Junior Developer)

---

## Epic 4: Integration Testing

### Story 4.1: API Integration Tests
**As a** developer  
**I want** integration tests for API endpoints  
**So that** the full request/response cycle is validated

**Acceptance Criteria:**
- Tests use WebApplicationFactory for in-memory testing
- Tests use test database (SQL Server LocalDB or Cosmos DB Emulator)
- Tests verify complete HTTP request/response
- Tests check proper status codes and headers
- Tests validate response JSON structure

**Technical Implementation:**
```csharp
// Tests/Integration/DomainObjectsControllerTests.cs
public class DomainObjectsControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    private readonly WebApplicationFactory<Program> _factory;
    
    public DomainObjectsControllerTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetObject_WithValidId_ReturnsObject()
    {
        // Arrange - seed database
        await SeedDatabaseAsync();
        
        // Act
        var response = await _client.GetAsync("/api/objects/test/123");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var obj = await response.Content.ReadFromJsonAsync<DomainObject>();
        obj.Should().NotBeNull();
        obj!.Identifier.Key.Should().Be("123");
    }
    
    // More tests...
}
```

**Estimated Hours:** 24 hours (Junior Developer)

---

## Epic 5: E2E Testing with Playwright

### Story 5.1: Object Tree E2E Tests
**As a** QA engineer  
**I want** end-to-end tests for object tree functionality  
**So that** user workflows are validated

**Acceptance Criteria:**
- Tests use Playwright for browser automation
- Tests verify tree loads and displays objects
- Tests verify object creation through UI
- Tests verify object selection and navigation
- Tests run against deployed application

**Technical Implementation:**
```typescript
// e2e/object-tree.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Object Tree', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:4200');
    await page.waitForSelector('app-object-tree');
  });
  
  test('should display root objects', async ({ page }) => {
    const treeNodes = await page.locator('mat-tree-node').count();
    expect(treeNodes).toBeGreaterThan(0);
  });
  
  test('should create new folder', async ({ page }) => {
    await page.click('[aria-label="Create Object"]');
    await page.selectOption('mat-select[formControlName="type"]', 'folder');
    await page.fill('input[formControlName="name"]', 'New Folder');
    await page.click('button:has-text("Create")');
    
    await expect(page.locator('text=New Folder')).toBeVisible();
  });
  
  test('should expand and collapse tree nodes', async ({ page }) => {
    await page.click('button[aria-label*="Toggle"]');
    await expect(page.locator('.mat-tree-node[level="1"]')).toBeVisible();
    
    await page.click('button[aria-label*="Toggle"]');
    await expect(page.locator('.mat-tree-node[level="1"]')).not.toBeVisible();
  });
});
```

**Estimated Hours:** 20 hours (Junior Developer)

---

## Epic 6: Azure Deployment

### Story 6.1: Infrastructure as Code
**As a** DevOps engineer  
**I want** ARM templates or Bicep files for Azure infrastructure  
**So that** resources can be deployed consistently

**Acceptance Criteria:**
- Bicep template for Azure App Service (backend)
- Bicep template for Azure Static Web Apps (frontend)
- Bicep template for Azure Cosmos DB
- Bicep template for Azure Application Insights
- Templates support multiple environments (dev, staging, prod)
- Templates use parameter files for configuration

**Technical Implementation:**
```bicep
// infrastructure/main.bicep
param location string = resourceGroup().location
param environment string = 'dev'
param appName string

resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' = {
  name: '${appName}-cosmos-${environment}'
  location: location
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: location
        failoverPriority: 0
      }
    ]
  }
}

resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: '${appName}-plan-${environment}'
  location: location
  sku: {
    name: 'B1'
    tier: 'Basic'
  }
}

resource appService 'Microsoft.Web/sites@2022-09-01' = {
  name: '${appName}-api-${environment}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      netFrameworkVersion: 'v8.0'
      appSettings: [
        {
          name: 'CosmosDb__Endpoint'
          value: cosmosDb.properties.documentEndpoint
        }
      ]
    }
  }
}
```

**Estimated Hours:** 16 hours (Junior Developer)

---

### Story 6.2: GitHub Actions CI/CD Pipeline
**As a** DevOps engineer  
**I want** automated deployment pipelines using GitHub Actions  
**So that** code changes are automatically deployed to Azure

**Acceptance Criteria:**
- Pipeline builds .NET backend on every commit
- Pipeline builds Angular frontend on every commit
- Pipeline runs all tests before deployment
- Pipeline deploys to Azure on merge to main branch
- Pipeline supports manual approval for production
- Pipeline creates deployment artifacts

**Technical Implementation:**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore
      
      - name: Test
        run: dotnet test --no-build --verbosity normal
      
      - name: Publish
        run: dotnet publish -c Release -o ./publish
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: backend
          path: ./publish
  
  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test -- --watch=false --browsers=ChromeHeadless
      
      - name: Build
        run: npm run build -- --configuration production
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: frontend
          path: ./dist
  
  deploy:
    needs: [build-backend, build-frontend]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - name: Download backend artifact
        uses: actions/download-artifact@v3
        with:
          name: backend
          path: ./backend
      
      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ secrets.AZURE_APP_SERVICE_NAME }}
          publish-profile: ${{ secrets.AZURE_APP_SERVICE_PUBLISH_PROFILE }}
          package: ./backend
      
      - name: Download frontend artifact
        uses: actions/download-artifact@v3
        with:
          name: frontend
          path: ./frontend
      
      - name: Deploy to Azure Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: "upload"
          app_location: "./frontend"
```

**Estimated Hours:** 20 hours (Junior Developer)

---

## Summary of Hours for Domain Object Management Feature

| Epic | Story Count | Total Hours |
|------|-------------|-------------|
| Epic 1: Core Infrastructure | 5 | 92 |
| Epic 2: Object Tree | 2 | 44 |
| Epic 3: Unit Testing | 2 | 64 |
| Epic 4: Integration Testing | 1 | 24 |
| Epic 5: E2E Testing | 1 | 20 |
| Epic 6: Azure Deployment | 2 | 36 |
| **Total** | **13** | **280** |

**Note:** These estimates are for a junior developer. An experienced developer could complete these stories 30-50% faster.
