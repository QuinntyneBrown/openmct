# Feature: Visualization Plugins

## Overview
Visualization Plugins provide different ways to display telemetry data including plots, tables, gauges, and layouts. These plugins are composable and configurable by users.

---

## Epic 1: Plot/Chart Visualization

### Story 1.1: Line Chart Component
**As a** user  
**I want** to visualize telemetry data as line charts  
**So that** I can see trends over time

**Acceptance Criteria:**
- Chart displays time-series data with responsive scaling
- Chart supports multiple Y-axes for different units
- Chart supports zoom and pan interactions
- Chart updates in real-time with streaming data
- Chart supports custom color schemes per series
- Chart shows legend with series names
- Chart shows tooltips on hover with values
- Chart exports to PNG/SVG

**Technical Implementation (Angular + Chart.js):**
```typescript
// components/plot-view/plot-view.component.ts
@Component({
  selector: 'app-plot-view',
  template: `
    <div class="plot-container">
      <div class="plot-toolbar">
        <button mat-icon-button (click)="resetZoom()">
          <mat-icon>zoom_out_map</mat-icon>
        </button>
        <button mat-icon-button (click)="exportChart()">
          <mat-icon>download</mat-icon>
        </button>
      </div>
      <canvas #chartCanvas></canvas>
    </div>
  `,
  styles: [`
    .plot-container {
      height: 100%;
      display: flex;
      flex-direction: column;
    }
    canvas {
      flex: 1;
    }
  `]
})
export class PlotViewComponent implements OnInit, OnDestroy {
  @ViewChild('chartCanvas') chartCanvas!: ElementRef<HTMLCanvasElement>;
  @Input() domainObject!: DomainObject;
  
  private chart?: Chart;
  private subscriptions: Subscription[] = [];
  
  constructor(
    private telemetryService: TelemetryService,
    private telemetryRealtimeService: TelemetryRealtimeService
  ) {}
  
  ngOnInit(): void {
    this.initializeChart();
    this.loadHistoricalData();
    this.subscribeToRealtime();
  }
  
  private initializeChart(): void {
    const ctx = this.chartCanvas.nativeElement.getContext('2d')!;
    
    this.chart = new Chart(ctx, {
      type: 'line',
      data: {
        datasets: []
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        interaction: {
          mode: 'index',
          intersect: false
        },
        plugins: {
          legend: {
            display: true,
            position: 'top'
          },
          tooltip: {
            enabled: true,
            callbacks: {
              label: (context) => {
                const label = context.dataset.label || '';
                const value = context.parsed.y.toFixed(2);
                return `${label}: ${value}`;
              }
            }
          },
          zoom: {
            pan: {
              enabled: true,
              mode: 'x'
            },
            zoom: {
              wheel: {
                enabled: true
              },
              pinch: {
                enabled: true
              },
              mode: 'x'
            }
          }
        },
        scales: {
          x: {
            type: 'time',
            time: {
              displayFormats: {
                millisecond: 'HH:mm:ss.SSS',
                second: 'HH:mm:ss',
                minute: 'HH:mm',
                hour: 'HH:mm'
              }
            },
            title: {
              display: true,
              text: 'Time'
            }
          },
          y: {
            title: {
              display: true,
              text: 'Value'
            }
          }
        }
      }
    });
  }
  
  private loadHistoricalData(): void {
    const request: TelemetryRequest = {
      objectId: this.domainObject.identifier,
      start: new Date(Date.now() - 3600000), // Last hour
      end: new Date(),
      strategy: 'all'
    };
    
    const sub = this.telemetryService.getHistoricalData(request).subscribe({
      next: (response) => {
        const data = response.data.map(d => ({
          x: d.timestamp,
          y: d.value as number
        }));
        
        this.chart!.data.datasets = [{
          label: this.domainObject.name,
          data: data,
          borderColor: 'rgb(75, 192, 192)',
          backgroundColor: 'rgba(75, 192, 192, 0.1)',
          borderWidth: 2,
          pointRadius: 0
        }];
        
        this.chart!.update();
      },
      error: (err) => console.error('Failed to load historical data:', err)
    });
    
    this.subscriptions.push(sub);
  }
  
  private subscribeToRealtime(): void {
    const sub = this.telemetryRealtimeService.subscribe(this.domainObject.identifier)
      .subscribe({
        next: (point) => {
          if (this.chart && this.chart.data.datasets.length > 0) {
            const dataset = this.chart.data.datasets[0];
            dataset.data.push({
              x: point.timestamp,
              y: point.value as number
            });
            
            // Keep only last 1000 points
            if (dataset.data.length > 1000) {
              dataset.data.shift();
            }
            
            this.chart.update('none'); // Update without animation for performance
          }
        },
        error: (err) => console.error('Realtime subscription error:', err)
      });
    
    this.subscriptions.push(sub);
  }
  
  resetZoom(): void {
    if (this.chart) {
      this.chart.resetZoom();
    }
  }
  
  exportChart(): void {
    if (this.chart) {
      const url = this.chart.toBase64Image();
      const link = document.createElement('a');
      link.download = `${this.domainObject.name}-chart.png`;
      link.href = url;
      link.click();
    }
  }
  
  ngOnDestroy(): void {
    this.subscriptions.forEach(sub => sub.unsubscribe());
    if (this.chart) {
      this.chart.destroy();
    }
  }
}
```

**Estimated Hours:** 32 hours (Junior Developer)

---

### Story 1.2: Multi-Series Plot Support
**As a** user  
**I want** to plot multiple telemetry series on one chart  
**So that** I can compare related metrics

**Acceptance Criteria:**
- Users can add/remove series from plot
- Each series has configurable color and style
- Series can use different Y-axes
- Legend shows all series with toggle visibility
- Series data updates independently in real-time

**Technical Implementation:**
```typescript
// components/multi-plot-view/multi-plot-view.component.ts
@Component({
  selector: 'app-multi-plot-view',
  template: `
    <div class="multi-plot-container">
      <mat-toolbar class="series-toolbar">
        <button mat-button (click)="addSeries()">
          <mat-icon>add</mat-icon> Add Series
        </button>
        <mat-chip-listbox>
          <mat-chip *ngFor="let series of seriesList" 
                    [removable]="true"
                    (removed)="removeSeries(series)">
            {{ series.name }}
            <mat-icon matChipRemove>cancel</mat-icon>
          </mat-chip>
        </mat-chip-listbox>
      </mat-toolbar>
      <canvas #chartCanvas></canvas>
    </div>
  `
})
export class MultiPlotViewComponent implements OnInit, OnDestroy {
  seriesList: TelemetrySeries[] = [];
  private chart?: Chart;
  private subscriptions = new Map<string, Subscription>();
  
  addSeries(): void {
    const dialogRef = this.dialog.open(SelectTelemetryDialog);
    
    dialogRef.afterClosed().subscribe((result: DomainObject | undefined) => {
      if (result) {
        this.addTelemetrySeries(result);
      }
    });
  }
  
  private addTelemetrySeries(domainObject: DomainObject): void {
    const seriesId = `${domainObject.identifier.namespace}:${domainObject.identifier.key}`;
    
    if (this.subscriptions.has(seriesId)) {
      return; // Already added
    }
    
    const color = this.generateColor(this.seriesList.length);
    const dataset: ChartDataset = {
      label: domainObject.name,
      data: [],
      borderColor: color,
      backgroundColor: color + '20',
      borderWidth: 2,
      yAxisID: this.shouldUseSecondaryAxis(domainObject) ? 'y1' : 'y'
    };
    
    this.chart!.data.datasets.push(dataset);
    
    // Load historical data
    this.loadSeriesHistoricalData(domainObject, dataset);
    
    // Subscribe to realtime
    const sub = this.telemetryRealtimeService
      .subscribe(domainObject.identifier)
      .subscribe(point => {
        dataset.data.push({
          x: point.timestamp,
          y: point.value as number
        });
        
        if (dataset.data.length > 1000) {
          dataset.data.shift();
        }
        
        this.chart!.update('none');
      });
    
    this.subscriptions.set(seriesId, sub);
    this.seriesList.push({
      id: seriesId,
      name: domainObject.name,
      object: domainObject
    });
    
    this.chart!.update();
  }
  
  removeSeries(series: TelemetrySeries): void {
    const sub = this.subscriptions.get(series.id);
    if (sub) {
      sub.unsubscribe();
      this.subscriptions.delete(series.id);
    }
    
    const datasetIndex = this.chart!.data.datasets.findIndex(
      ds => ds.label === series.name
    );
    
    if (datasetIndex >= 0) {
      this.chart!.data.datasets.splice(datasetIndex, 1);
      this.chart!.update();
    }
    
    this.seriesList = this.seriesList.filter(s => s.id !== series.id);
  }
  
  private generateColor(index: number): string {
    const colors = [
      'rgb(75, 192, 192)',
      'rgb(255, 99, 132)',
      'rgb(54, 162, 235)',
      'rgb(255, 206, 86)',
      'rgb(153, 102, 255)'
    ];
    return colors[index % colors.length];
  }
}

interface TelemetrySeries {
  id: string;
  name: string;
  object: DomainObject;
}
```

**Estimated Hours:** 24 hours (Junior Developer)

---

## Epic 2: Table Visualization

### Story 2.1: Telemetry Table Component
**As a** user  
**I want** to view telemetry data in a tabular format  
**So that** I can see precise values and perform analysis

**Acceptance Criteria:**
- Table displays latest values from multiple telemetry points
- Table columns are sortable and filterable
- Table supports column reordering
- Table updates in real-time with new data
- Table supports export to CSV
- Table supports pagination for large datasets
- Table highlights values exceeding thresholds

**Technical Implementation (Angular Material Table):**
```typescript
// components/telemetry-table/telemetry-table.component.ts
@Component({
  selector: 'app-telemetry-table',
  template: `
    <div class="table-container">
      <mat-toolbar>
        <mat-form-field>
          <mat-label>Filter</mat-label>
          <input matInput (keyup)="applyFilter($event)" placeholder="Search...">
        </mat-form-field>
        <span class="spacer"></span>
        <button mat-button (click)="exportToCsv()">
          <mat-icon>download</mat-icon> Export CSV
        </button>
      </mat-toolbar>
      
      <table mat-table [dataSource]="dataSource" matSort class="telemetry-table">
        <ng-container matColumnDef="name">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>Name</th>
          <td mat-cell *matCellDef="let row">{{ row.name }}</td>
        </ng-container>
        
        <ng-container matColumnDef="value">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>Value</th>
          <td mat-cell *matCellDef="let row" 
              [class.warning]="isOutOfRange(row)"
              [class.critical]="isCritical(row)">
            {{ formatValue(row.value, row.format) }}
          </td>
        </ng-container>
        
        <ng-container matColumnDef="units">
          <th mat-header-cell *matHeaderCellDef>Units</th>
          <td mat-cell *matCellDef="let row">{{ row.units }}</td>
        </ng-container>
        
        <ng-container matColumnDef="timestamp">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>Timestamp</th>
          <td mat-cell *matCellDef="let row">
            {{ row.timestamp | date:'medium' }}
          </td>
        </ng-container>
        
        <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
        <tr mat-row *matRowDef="let row; columns: displayedColumns;"></tr>
      </table>
      
      <mat-paginator [pageSizeOptions]="[10, 25, 50, 100]"></mat-paginator>
    </div>
  `,
  styles: [`
    .table-container {
      height: 100%;
      display: flex;
      flex-direction: column;
    }
    .telemetry-table {
      flex: 1;
      overflow: auto;
    }
    .warning {
      background-color: #fff3cd;
    }
    .critical {
      background-color: #f8d7da;
      font-weight: bold;
    }
  `]
})
export class TelemetryTableComponent implements OnInit, OnDestroy, AfterViewInit {
  @Input() domainObject!: DomainObject;
  @ViewChild(MatSort) sort!: MatSort;
  @ViewChild(MatPaginator) paginator!: MatPaginator;
  
  displayedColumns = ['name', 'value', 'units', 'timestamp'];
  dataSource: MatTableDataSource<TelemetryTableRow>;
  private subscriptions: Subscription[] = [];
  
  constructor(
    private telemetryService: TelemetryService,
    private telemetryRealtimeService: TelemetryRealtimeService
  ) {
    this.dataSource = new MatTableDataSource<TelemetryTableRow>([]);
  }
  
  ngOnInit(): void {
    this.loadTelemetryPoints();
  }
  
  ngAfterViewInit(): void {
    this.dataSource.sort = this.sort;
    this.dataSource.paginator = this.paginator;
  }
  
  private loadTelemetryPoints(): void {
    // Load composition (child telemetry points)
    this.domainObjectService.getComposition(this.domainObject.identifier)
      .subscribe({
        next: (children) => {
          const telemetryPoints = children.filter(c => c.type === 'telemetry');
          this.initializeTableRows(telemetryPoints);
          this.subscribeToAllPoints(telemetryPoints);
        }
      });
  }
  
  private initializeTableRows(points: DomainObject[]): void {
    const rows: TelemetryTableRow[] = points.map(point => ({
      id: `${point.identifier.namespace}:${point.identifier.key}`,
      name: point.name,
      value: null,
      units: point.metadata?.units || '',
      format: point.metadata?.format || '',
      timestamp: null,
      minValue: point.metadata?.minValue,
      maxValue: point.metadata?.maxValue,
      criticalMin: point.metadata?.criticalMin,
      criticalMax: point.metadata?.criticalMax
    }));
    
    this.dataSource.data = rows;
  }
  
  private subscribeToAllPoints(points: DomainObject[]): void {
    points.forEach(point => {
      const sub = this.telemetryRealtimeService.subscribe(point.identifier)
        .subscribe({
          next: (telemetryPoint) => {
            this.updateRow(point.identifier, telemetryPoint);
          }
        });
      
      this.subscriptions.push(sub);
    });
  }
  
  private updateRow(id: DomainObjectIdentifier, point: TelemetryPoint): void {
    const rowId = `${id.namespace}:${id.key}`;
    const rows = this.dataSource.data;
    const rowIndex = rows.findIndex(r => r.id === rowId);
    
    if (rowIndex >= 0) {
      rows[rowIndex] = {
        ...rows[rowIndex],
        value: point.value,
        timestamp: point.timestamp
      };
      
      this.dataSource.data = [...rows]; // Trigger change detection
    }
  }
  
  applyFilter(event: Event): void {
    const filterValue = (event.target as HTMLInputElement).value;
    this.dataSource.filter = filterValue.trim().toLowerCase();
  }
  
  formatValue(value: any, format: string): string {
    if (value === null || value === undefined) {
      return '-';
    }
    
    if (format) {
      // Apply custom format (e.g., "0.00", "0.0e+0")
      return numeral(value).format(format);
    }
    
    return value.toString();
  }
  
  isOutOfRange(row: TelemetryTableRow): boolean {
    if (!row.value || !row.minValue || !row.maxValue) {
      return false;
    }
    
    return row.value < row.minValue || row.value > row.maxValue;
  }
  
  isCritical(row: TelemetryTableRow): boolean {
    if (!row.value) {
      return false;
    }
    
    if (row.criticalMin && row.value < row.criticalMin) {
      return true;
    }
    
    if (row.criticalMax && row.value > row.criticalMax) {
      return true;
    }
    
    return false;
  }
  
  exportToCsv(): void {
    const csvData = this.dataSource.data.map(row => ({
      Name: row.name,
      Value: row.value,
      Units: row.units,
      Timestamp: row.timestamp?.toISOString()
    }));
    
    const csv = this.convertToCSV(csvData);
    const blob = new Blob([csv], { type: 'text/csv' });
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = `${this.domainObject.name}-data.csv`;
    link.click();
  }
  
  private convertToCSV(data: any[]): string {
    if (!data || !data.length) {
      return '';
    }
    
    const headers = Object.keys(data[0]);
    const csvRows = [
      headers.join(','),
      ...data.map(row => headers.map(h => row[h]).join(','))
    ];
    
    return csvRows.join('\n');
  }
  
  ngOnDestroy(): void {
    this.subscriptions.forEach(sub => sub.unsubscribe());
  }
}

interface TelemetryTableRow {
  id: string;
  name: string;
  value: any;
  units: string;
  format: string;
  timestamp: Date | null;
  minValue?: number;
  maxValue?: number;
  criticalMin?: number;
  criticalMax?: number;
}
```

**Estimated Hours:** 28 hours (Junior Developer)

---

## Epic 3: Gauge Visualization

### Story 3.1: Circular Gauge Component
**As a** user  
**I want** to display single telemetry values as gauges  
**So that** I can quickly assess status at a glance

**Acceptance Criteria:**
- Gauge displays current value with min/max range
- Gauge supports color zones (green, yellow, red)
- Gauge updates in real-time
- Gauge shows units and value label
- Gauge is configurable for size and appearance

**Technical Implementation:**
```typescript
// components/gauge-view/gauge-view.component.ts
@Component({
  selector: 'app-gauge-view',
  template: `
    <div class="gauge-container">
      <canvas #gaugeCanvas></canvas>
      <div class="gauge-value">
        <span class="value">{{ currentValue | number:'1.2-2' }}</span>
        <span class="units">{{ units }}</span>
      </div>
    </div>
  `,
  styles: [`
    .gauge-container {
      position: relative;
      width: 100%;
      height: 100%;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .gauge-value {
      position: absolute;
      text-align: center;
    }
    .value {
      display: block;
      font-size: 2em;
      font-weight: bold;
    }
    .units {
      display: block;
      font-size: 0.8em;
      color: #666;
    }
  `]
})
export class GaugeViewComponent implements OnInit, OnDestroy {
  @ViewChild('gaugeCanvas') canvasRef!: ElementRef<HTMLCanvasElement>;
  @Input() domainObject!: DomainObject;
  
  currentValue = 0;
  units = '';
  private subscription?: Subscription;
  
  ngOnInit(): void {
    this.units = this.domainObject.metadata?.units || '';
    this.initializeGauge();
    this.subscribeToTelemetry();
  }
  
  private initializeGauge(): void {
    const canvas = this.canvasRef.nativeElement;
    const ctx = canvas.getContext('2d')!;
    const minValue = this.domainObject.metadata?.minValue || 0;
    const maxValue = this.domainObject.metadata?.maxValue || 100;
    
    // Draw static gauge background
    this.drawGaugeBackground(ctx, minValue, maxValue);
  }
  
  private drawGaugeBackground(
    ctx: CanvasRenderingContext2D, 
    min: number, 
    max: number
  ): void {
    const centerX = ctx.canvas.width / 2;
    const centerY = ctx.canvas.height / 2;
    const radius = Math.min(centerX, centerY) - 20;
    
    // Draw arc segments with colors
    const greenEnd = min + (max - min) * 0.7;
    const yellowEnd = min + (max - min) * 0.9;
    
    // Green zone
    this.drawArc(ctx, centerX, centerY, radius, 
      this.valueToAngle(min, min, max),
      this.valueToAngle(greenEnd, min, max),
      '#4CAF50'
    );
    
    // Yellow zone
    this.drawArc(ctx, centerX, centerY, radius,
      this.valueToAngle(greenEnd, min, max),
      this.valueToAngle(yellowEnd, min, max),
      '#FFC107'
    );
    
    // Red zone
    this.drawArc(ctx, centerX, centerY, radius,
      this.valueToAngle(yellowEnd, min, max),
      this.valueToAngle(max, min, max),
      '#F44336'
    );
  }
  
  private drawArc(
    ctx: CanvasRenderingContext2D,
    x: number,
    y: number,
    radius: number,
    startAngle: number,
    endAngle: number,
    color: string
  ): void {
    ctx.beginPath();
    ctx.arc(x, y, radius, startAngle, endAngle);
    ctx.lineWidth = 20;
    ctx.strokeStyle = color;
    ctx.stroke();
  }
  
  private valueToAngle(value: number, min: number, max: number): number {
    const percentage = (value - min) / (max - min);
    const startAngle = Math.PI * 0.75; // Start at 135 degrees
    const endAngle = Math.PI * 2.25;   // End at 405 degrees
    return startAngle + percentage * (endAngle - startAngle);
  }
  
  private subscribeToTelemetry(): void {
    this.subscription = this.telemetryRealtimeService
      .subscribe(this.domainObject.identifier)
      .subscribe({
        next: (point) => {
          this.currentValue = point.value as number;
          this.updateNeedle();
        }
      });
  }
  
  private updateNeedle(): void {
    // Redraw gauge with needle pointing to current value
    const canvas = this.canvasRef.nativeElement;
    const ctx = canvas.getContext('2d')!;
    const min = this.domainObject.metadata?.minValue || 0;
    const max = this.domainObject.metadata?.maxValue || 100;
    
    // Clear previous needle
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    
    // Redraw background
    this.drawGaugeBackground(ctx, min, max);
    
    // Draw needle
    const centerX = canvas.width / 2;
    const centerY = canvas.height / 2;
    const radius = Math.min(centerX, centerY) - 20;
    const angle = this.valueToAngle(this.currentValue, min, max);
    
    ctx.beginPath();
    ctx.moveTo(centerX, centerY);
    ctx.lineTo(
      centerX + radius * 0.8 * Math.cos(angle),
      centerY + radius * 0.8 * Math.sin(angle)
    );
    ctx.lineWidth = 3;
    ctx.strokeStyle = '#333';
    ctx.stroke();
    
    // Draw center circle
    ctx.beginPath();
    ctx.arc(centerX, centerY, 10, 0, 2 * Math.PI);
    ctx.fillStyle = '#333';
    ctx.fill();
  }
  
  ngOnDestroy(): void {
    this.subscription?.unsubscribe();
  }
}
```

**Estimated Hours:** 20 hours (Junior Developer)

---

## Epic 4: Layout System

### Story 4.1: Flexible Layout Container
**As a** user  
**I want** to create custom layouts with multiple visualization components  
**So that** I can build dashboards tailored to my needs

**Acceptance Criteria:**
- Layout supports drag-and-drop positioning of components
- Layout supports resizing of components
- Layout saves configuration to domain object
- Layout supports nested layouts
- Layout adapts to container size (responsive)
- Layout supports grid snapping for alignment

**Technical Implementation (using gridstack.js):**
```typescript
// components/layout-view/layout-view.component.ts
@Component({
  selector: 'app-layout-view',
  template: `
    <div class="layout-container">
      <mat-toolbar>
        <button mat-button (click)="toggleEditMode()">
          <mat-icon>{{ editMode ? 'lock_open' : 'lock' }}</mat-icon>
          {{ editMode ? 'Lock' : 'Edit' }}
        </button>
        <button mat-button (click)="addWidget()" [disabled]="!editMode">
          <mat-icon>add</mat-icon> Add Widget
        </button>
        <button mat-button (click)="saveLayout()" [disabled]="!editMode">
          <mat-icon>save</mat-icon> Save
        </button>
      </mat-toolbar>
      
      <div class="grid-stack" #gridContainer></div>
    </div>
  `,
  styles: [`
    .layout-container {
      height: 100%;
      display: flex;
      flex-direction: column;
    }
    .grid-stack {
      flex: 1;
      background: #f5f5f5;
    }
  `]
})
export class LayoutViewComponent implements OnInit, OnDestroy, AfterViewInit {
  @ViewChild('gridContainer') gridContainer!: ElementRef;
  @Input() domainObject!: DomainObject;
  
  editMode = false;
  private grid?: GridStack;
  private widgets: LayoutWidget[] = [];
  
  constructor(
    private domainObjectService: DomainObjectService,
    private dialog: MatDialog,
    private componentFactoryResolver: ComponentFactoryResolver,
    private injector: Injector,
    private appRef: ApplicationRef
  ) {}
  
  ngAfterViewInit(): void {
    this.initializeGrid();
    this.loadLayout();
  }
  
  private initializeGrid(): void {
    this.grid = GridStack.init({
      float: true,
      cellHeight: 80,
      minRow: 1,
      disableOneColumnMode: true,
      animate: true
    }, this.gridContainer.nativeElement);
    
    this.grid.on('change', (event, items) => {
      if (this.editMode) {
        this.onLayoutChange(items);
      }
    });
  }
  
  toggleEditMode(): void {
    this.editMode = !this.editMode;
    
    if (this.grid) {
      if (this.editMode) {
        this.grid.enable();
      } else {
        this.grid.disable();
      }
    }
  }
  
  addWidget(): void {
    const dialogRef = this.dialog.open(SelectWidgetDialog);
    
    dialogRef.afterClosed().subscribe((result: WidgetSelection | undefined) => {
      if (result) {
        this.createWidget(result);
      }
    });
  }
  
  private createWidget(selection: WidgetSelection): void {
    const widget: LayoutWidget = {
      id: this.generateId(),
      type: selection.type,
      domainObject: selection.domainObject,
      x: 0,
      y: 0,
      width: selection.defaultWidth || 4,
      height: selection.defaultHeight || 3
    };
    
    this.widgets.push(widget);
    this.renderWidget(widget);
  }
  
  private renderWidget(widget: LayoutWidget): void {
    const gridItemEl = document.createElement('div');
    gridItemEl.className = 'grid-stack-item';
    gridItemEl.setAttribute('gs-id', widget.id);
    gridItemEl.setAttribute('gs-x', widget.x.toString());
    gridItemEl.setAttribute('gs-y', widget.y.toString());
    gridItemEl.setAttribute('gs-w', widget.width.toString());
    gridItemEl.setAttribute('gs-h', widget.height.toString());
    
    const contentEl = document.createElement('div');
    contentEl.className = 'grid-stack-item-content';
    
    // Create Angular component dynamically
    const component = this.createComponent(widget);
    contentEl.appendChild(component);
    
    gridItemEl.appendChild(contentEl);
    
    this.grid!.addWidget(gridItemEl);
  }
  
  private createComponent(widget: LayoutWidget): HTMLElement {
    const componentType = this.getComponentType(widget.type);
    const componentRef = this.componentFactoryResolver
      .resolveComponentFactory(componentType)
      .create(this.injector);
    
    // Set input properties
    (componentRef.instance as any).domainObject = widget.domainObject;
    
    this.appRef.attachView(componentRef.hostView);
    
    return (componentRef.hostView as EmbeddedViewRef<any>).rootNodes[0];
  }
  
  private getComponentType(type: string): Type<any> {
    const componentMap: Record<string, Type<any>> = {
      'plot': PlotViewComponent,
      'table': TelemetryTableComponent,
      'gauge': GaugeViewComponent
    };
    
    return componentMap[type] || PlotViewComponent;
  }
  
  private loadLayout(): void {
    const layoutConfig = this.domainObject.metadata?.layout;
    
    if (layoutConfig && layoutConfig.widgets) {
      this.widgets = layoutConfig.widgets;
      this.widgets.forEach(widget => this.renderWidget(widget));
    }
  }
  
  saveLayout(): void {
    const layoutConfig = {
      widgets: this.widgets.map(w => ({
        id: w.id,
        type: w.type,
        domainObject: w.domainObject,
        x: w.x,
        y: w.y,
        width: w.width,
        height: w.height
      }))
    };
    
    const updatedObject = {
      ...this.domainObject,
      metadata: {
        ...this.domainObject.metadata,
        layout: layoutConfig
      }
    };
    
    this.domainObjectService.update(updatedObject).subscribe({
      next: () => console.log('Layout saved'),
      error: (err) => console.error('Failed to save layout:', err)
    });
  }
  
  private onLayoutChange(items: any[]): void {
    items.forEach(item => {
      const widget = this.widgets.find(w => w.id === item.id);
      if (widget) {
        widget.x = item.x;
        widget.y = item.y;
        widget.width = item.w;
        widget.height = item.h;
      }
    });
  }
  
  private generateId(): string {
    return `widget-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
  
  ngOnDestroy(): void {
    if (this.grid) {
      this.grid.destroy();
    }
  }
}

interface LayoutWidget {
  id: string;
  type: string;
  domainObject: DomainObject;
  x: number;
  y: number;
  width: number;
  height: number;
}

interface WidgetSelection {
  type: string;
  domainObject: DomainObject;
  defaultWidth?: number;
  defaultHeight?: number;
}
```

**Estimated Hours:** 36 hours (Junior Developer)

---

## Epic 5: Unit Testing

### Story 5.1: Visualization Component Tests
**As a** developer  
**I want** unit tests for all visualization components  
**So that** rendering and interaction logic is validated

**Acceptance Criteria:**
- Tests for plot component rendering
- Tests for table component data binding
- Tests for gauge component value updates
- Tests for layout component widget management
- Test coverage > 80%

**Technical Implementation:**
```typescript
// components/plot-view/plot-view.component.spec.ts
describe('PlotViewComponent', () => {
  let component: PlotViewComponent;
  let fixture: ComponentFixture<PlotViewComponent>;
  let telemetryService: jasmine.SpyObj<TelemetryService>;
  let realtimeService: jasmine.SpyObj<TelemetryRealtimeService>;
  
  beforeEach(async () => {
    const telemetrySpy = jasmine.createSpyObj('TelemetryService', ['getHistoricalData']);
    const realtimeSpy = jasmine.createSpyObj('TelemetryRealtimeService', ['subscribe']);
    
    await TestBed.configureTestingModule({
      declarations: [PlotViewComponent],
      providers: [
        { provide: TelemetryService, useValue: telemetrySpy },
        { provide: TelemetryRealtimeService, useValue: realtimeSpy }
      ]
    }).compileComponents();
    
    telemetryService = TestBed.inject(TelemetryService) as jasmine.SpyObj<TelemetryService>;
    realtimeService = TestBed.inject(TelemetryRealtimeService) as jasmine.SpyObj<TelemetryRealtimeService>;
  });
  
  beforeEach(() => {
    fixture = TestBed.createComponent(PlotViewComponent);
    component = fixture.componentInstance;
    component.domainObject = {
      identifier: { namespace: 'test', key: '123' },
      name: 'Test Object',
      type: 'telemetry'
    } as DomainObject;
  });
  
  it('should create', () => {
    expect(component).toBeTruthy();
  });
  
  it('should load historical data on init', () => {
    const mockData: TelemetryResponse = {
      data: [
        { timestamp: new Date(), value: 42 }
      ],
      metadata: { start: new Date(), end: new Date(), count: 1, strategy: 'all' }
    };
    
    telemetryService.getHistoricalData.and.returnValue(of(mockData));
    realtimeService.subscribe.and.returnValue(EMPTY);
    
    fixture.detectChanges();
    
    expect(telemetryService.getHistoricalData).toHaveBeenCalled();
  });
  
  it('should update chart with realtime data', (done) => {
    telemetryService.getHistoricalData.and.returnValue(of({
      data: [],
      metadata: {} as any
    }));
    
    const realtimeSubject = new Subject<TelemetryPoint>();
    realtimeService.subscribe.and.returnValue(realtimeSubject.asObservable());
    
    fixture.detectChanges();
    
    // Emit realtime data
    realtimeSubject.next({
      timestamp: new Date(),
      value: 100
    });
    
    setTimeout(() => {
      expect(component['chart']?.data.datasets[0].data.length).toBeGreaterThan(0);
      done();
    }, 100);
  });
});
```

**Estimated Hours:** 32 hours (Junior Developer)

---

## Epic 6: E2E Testing with Playwright

### Story 6.1: Visualization E2E Tests
**As a** QA engineer  
**I want** E2E tests for visualization workflows  
**So that** user interactions are validated

**Acceptance Criteria:**
- Tests for creating and viewing plots
- Tests for table filtering and sorting
- Tests for layout creation and editing
- Tests for gauge value updates

**Technical Implementation:**
```typescript
// e2e/visualizations.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Visualizations', () => {
  test('should create and display a plot', async ({ page }) => {
    await page.goto('http://localhost:4200');
    
    // Create new plot
    await page.click('[aria-label="Create Object"]');
    await page.selectOption('select[name="type"]', 'plot');
    await page.fill('input[name="name"]', 'Test Plot');
    await page.click('button:has-text("Create")');
    
    // Navigate to plot
    await page.click('text=Test Plot');
    
    // Verify plot canvas is rendered
    await expect(page.locator('canvas')).toBeVisible();
  });
  
  test('should add series to plot', async ({ page }) => {
    await page.goto('http://localhost:4200/plot/test-plot');
    
    // Add series
    await page.click('button:has-text("Add Series")');
    await page.selectOption('select', 'test:telemetry-1');
    await page.click('button:has-text("Add")');
    
    // Verify series appears in legend
    await expect(page.locator('.legend-item:has-text("Telemetry 1")')).toBeVisible();
  });
});
```

**Estimated Hours:** 24 hours (Junior Developer)

---

## Summary of Hours for Visualization Plugins Feature

| Epic | Story Count | Total Hours |
|------|-------------|-------------|
| Epic 1: Plot/Chart | 2 | 56 |
| Epic 2: Table | 1 | 28 |
| Epic 3: Gauge | 1 | 20 |
| Epic 4: Layout System | 1 | 36 |
| Epic 5: Unit Testing | 1 | 32 |
| Epic 6: E2E Testing | 1 | 24 |
| **Total** | **7** | **196** |
