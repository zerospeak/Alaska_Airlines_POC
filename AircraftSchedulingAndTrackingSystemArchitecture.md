# Aircraft Scheduling and Tracking System Architecture

## Attribution

This document was authored by Leeroy D'Souza

## Table of Contents

- Introduction
- System Architecture Overview
- Backend API Implementation
- Database Design
- Event-Driven Architecture
- Container Orchestration
- CI/CD Pipeline Implementation
- Testing Strategy
- UX/UI Design Principles
- Deployment Guidelines
- Conclusion
- Glossary
- Appendix

## Introduction

This comprehensive technical document outlines the architecture and implementation details of the Aircraft Scheduling and Tracking System. The system is designed to provide a robust, scalable, and maintainable solution for managing aircraft operations, with particular emphasis on reliability and performance.

## System Architecture Overview

The system follows a modular architecture with separate components for different responsibilities:

- `AircraftSchedulingApi`: Backend API logic
- `AircraftSchedulingDb`: Database schema and scripts
- `AircraftSchedulingEDA`: Event-driven architecture components
- `AircraftSchedulingWeb`: Frontend web application
- `AircraftSchedulingCI`: CI/CD pipeline scripts
- `docker-compose.yml`: Docker Compose configuration
- `deployment.yaml`: Kubernetes deployment configuration

```mermaid
flowchart TD
    classDef client fill:#90CAF9,stroke:#1976D2,color:#000
    classDef frontend fill:#81C784,stroke:#388E3C,color:#000
    classDef backend fill:#FFB74D,stroke:#F57C00,color:#000
    classDef database fill:#CE93D8,stroke:#7B1FA2,color:#000
    classDef event fill:#FF8A65,stroke:#D84315,color:#000

    subgraph ClientLayer["Client Layer"]
        Web[Web Browser]:::client
        Mobile[Mobile App]:::client
    end

    subgraph FrontendLayer["Frontend Layer"]
        WebApp[AircraftSchedulingWeb]:::frontend
    end

    subgraph BackendLayer["Backend Layer"]
        API[AircraftSchedulingApi]:::backend
        Service[AircraftService]:::backend
    end

    subgraph DataLayer["Data Layer"]
        DB[(SQL Server)]:::database
    end

    subgraph EventLayer["Event Layer"]
        EventBus[RabbitMQ/Azure SB]:::event
        Events[AircraftEvents]:::event
    end

    Web --> WebApp
    Mobile --> WebApp
    WebApp --> API
    API --> Service
    Service --> DB
    Service --> EventBus
    EventBus --> Events

    %% Legend
    subgraph Legend["Legend"]
        C[Client Components]:::client
        F[Frontend Components]:::frontend
        B[Backend Components]:::backend
        D[(Database)]:::database
        E[Event System]:::event
    end
```

The system architecture diagram above illustrates the layered design of our aircraft scheduling system. Each color represents a distinct responsibility layer:

- Blue components represent client interfaces
- Green shows frontend services
- Orange indicates backend processing
- Purple denotes database storage
- Red highlights event-driven components

Data flows from clients through the frontend layer to the backend services, which handle both database operations and event publishing. This separation ensures clean boundaries between concerns and enables independent scaling of components.

## Backend API Implementation

The AircraftSchedulingApi directory contains the backend API logic developed using C#/.NET Core:

```csharp
[ApiController]
[Route("api/[controller]")]
public class AircraftController : ControllerBase
{
    private readonly IAircraftService _service;

    public AircraftController(IAircraftService service)
    {
        _service = service;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Aircraft>>> GetAircraft()
    {
        return await _service.GetAircraft();
    }

    [HttpPost]
    public async Task<ActionResult<Aircraft>> CreateAircraft(Aircraft aircraft)
    {
        return await _service.CreateAircraft(aircraft);
    }
}

public class Aircraft
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Model { get; set; }
}

public interface IAircraftService
{
    Task<IEnumerable<Aircraft>> GetAircraft();
    Task<Aircraft> CreateAircraft(Aircraft aircraft);
}

public class AircraftService : IAircraftService
{
    private readonly AircraftDbContext _context;

    public AircraftService(AircraftDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Aircraft>> GetAircraft()
    {
        return await _context.Aircrafts.ToListAsync();
    }

    public async Task<Aircraft> CreateAircraft(Aircraft aircraft)
    {
        _context.Aircrafts.Add(aircraft);
        await _context.SaveChangesAsync();
        return aircraft;
    }
}

public class AircraftDbContext : DbContext
{
    public AircraftDbContext(DbContextOptions<AircraftDbContext> options) : base(options)
    {
    }

    public DbSet<Aircraft> Aircrafts { get; set; }
}
```

```mermaid
sequenceDiagram
    participant Client
    participant Controller as AircraftController
    participant Service as AircraftService
    participant DbContext as AircraftDbContext
    participant EventBus as AircraftEventPublisher
    
    Client->>+Controller: POST /api/aircraft
    Note over Client,Controller: JSON payload:<br/>{"name": "...",<br/>"model": "..."}
    
    Controller->>+Service: CreateAircraft(aircraft)
    Service->>+DbContext: Add(aircraft)
    DbContext-->>-Service: Added
    Service->>+DbContext: SaveChangesAsync()
    DbContext-->>-Service: Saved
    Service->>+EventBus: PublishAircraftEvent(event)
    EventBus-->>-Service: Published
    Service-->>-Controller: Created Aircraft
    Controller-->>-Client: 201 Created + Location header
```

The sequence diagram above illustrates the flow of creating an aircraft record. The solid arrows represent method calls, while dashed arrows show responses. Notice how the operation is fully asynchronous, ensuring non-blocking I/O operations. After successfully saving the aircraft data, an event is published to notify other parts of the system about the change.

## Database Design

The AircraftSchedulingDb directory contains the database schema and scripts for SQL Server:

```sql
CREATE TABLE Aircrafts (
    Id INT PRIMARY KEY IDENTITY(1,1),
    Name NVARCHAR(100),
    Model NVARCHAR(100)
);
```

## Event-Driven Architecture

The AircraftSchedulingEDA directory contains components for event-driven architecture using Azure Service Bus or RabbitMQ:

```csharp
using Azure.Messaging.ServiceBus;

public class AircraftEventPublisher
{
    private readonly ServiceBusClient _serviceBusClient;
    private readonly ServiceBusSender _sender;

    public AircraftEventPublisher(string connectionString, string queueName)
    {
        _serviceBusClient = new ServiceBusClient(connectionString);
        _sender = _serviceBusClient.CreateSender(queueName);
    }

    public async Task PublishAircraftEvent(AircraftEvent @event)
    {
        var message = new ServiceBusMessage(@event.ToString());
        await _sender.SendMessageAsync(message);
    }
}

public class AircraftEvent
{
    public string EventType { get; set; }
    public Aircraft Aircraft { get; set; }
}
```

## Container Orchestration

The system supports both Docker Compose and Kubernetes deployments:

```mermaid
flowchart LR
    classDef docker fill:#2496ED,stroke:#164B78,color:#fff
    classDef k8s fill:#326CE5,stroke:#19355D,color:#fff
    
    subgraph DockerCompose["Docker Compose Environment"]
        direction TB
        WebDC[Docker Web]:::docker
        ApiDC[Docker API]:::docker
        DbDC[(SQL Server)]:::docker
        BusDC[RabbitMQ]:::docker
        
        WebDC -->|"http:80->80"| ApiDC
        ApiDC -->|"depends_on: db"| DbDC
        ApiDC -->|"amqp://bus:5672"| BusDC
    end
    
    subgraph Kubernetes["Kubernetes Environment"]
        direction TB
        WebKS[Deployment Web]:::k8s
        ApiKS[Deployment API]:::k8s
        DbKS[StatefulSet DB]:::k8s
        BusKS[StatefulSet Bus]:::k8s
        
        WebKS -->|"Service: api:5000"| ApiKS
        ApiKS -->|"Service: db:1433"| DbKS
        ApiKS -->|"Service: bus:5672"| BusKS
    end
    
    %% Legend
    subgraph Legend["Container Orchestration Types"]
        DC[Docker Container]:::docker
        KS[Kubernetes Pod]:::k8s
    end
```

The diagram above illustrates the container orchestration strategies for both Docker Compose and Kubernetes environments. In Docker Compose, containers communicate directly using port mappings and dependencies. In Kubernetes, services are exposed through cluster IPs, and stateful sets are used for databases and message queues to maintain persistent storage and ordered deployment.

## CI/CD Pipeline Implementation

```yaml
name: Build and Deploy

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        run: |
          docker build -t aircraft-scheduling-api .
          docker tag aircraft-scheduling-api:latest ${{ secrets.DOCKER_USERNAME }}/aircraft-scheduling-api:latest
          docker push ${{ secrets.DOCKER_USERNAME }}/aircraft-scheduling-api:latest

      - name: Deploy to Kubernetes
        uses: kubernetes/deploy-action@v1
        with:
          kubeconfig: ${{ secrets.KUBECONFIG }}
          deployment: aircraft-scheduling-api
```

## Testing Strategy

The system implements a comprehensive testing strategy:

1. Unit Testing
  - Backend components using NUnit/xUnit
  - Frontend components using Jest


2. Integration Testing
  - API endpoints using Postman
  - Database operations
  - Event-driven interactions


3. User Acceptance Testing (UAT)
  - Functional requirements validation
  - User interface testing
  - Business workflow verification



## UX/UI Design Principles

The system implements comprehensive UX/UI design principles:

### 1. User-Centric Design

- Thorough user research to understand pilot and crew member requirements
- Design prototyping for flight scheduling and crew management systems
- Regular usability testing for feedback collection and design refinement

### 2. Consistency and Hierarchy

- Standardized component library across all applications
- Clear navigation structure with intuitive information architecture
- Consistent visual design language throughout the system

### 3. Context and Accessibility

- Responsive design for varying screen sizes and devices
- Screen reader support and keyboard navigation
- High contrast mode and font size adjustment options

### 4. User Control and Feedback

- Intuitive controls for workflow management
- Clear confirmation messages for successful operations
- Progress indicators for ongoing processes

### 5. Security and Privacy

- Two-factor authentication for all users
- Data encryption both in transit and at rest
- Role-based access control for sensitive operations

### 6. Scalability and Performance

- Scalable architecture using microservices
- Optimized resource loading
- Caching strategies for frequently accessed data

### 7. Responsive Design

- Cross-device compatibility
- Adaptive layouts for different screen sizes
- Touch-friendly controls for mobile devices

### 8. Iterative Design and Feedback

- Regular usability testing sessions
- User feedback collection and analysis
- Iterative design updates based on user input

## Example Applications

### Flight Scheduling Dashboard

The dashboard provides:

- Interactive real-time updates for flight management
- Data visualization for complex flight data
- Efficient management of flight assignments

### Crew Management System

The system offers:

- User-friendly interface for scheduling and training
- Real-time notifications for crew updates
- Performance tracking and management tools

### Passenger Services App

Passengers benefit from:

- Intuitive navigation system
- Real-time flight status updates
- Gate change notifications and relevant travel information

By integrating these UX/UI design principles into Alaska Airlines' internal applications, the airline can enhance operational efficiency, improve user satisfaction, and maintain a competitive edge in the aviation industry.

## Glossary

- Responsive Design: Design approach that adapts to different screen sizes and devices
- User-Centric Design: Design methodology focusing on user needs and behaviors
- Microservices: Small, independent services that communicate with each other
- Data Visualization: Presentation of complex data in an easily understandable format
- Accessibility: Design features that ensure all users can access and use the system
- Iterative Design: Continuous refinement of design based on user feedback
- Cross-Device Compatibility: Ability of applications to work seamlessly across different devices

## Appendix

### A. Implementation Guidelines

1. Development Process
  - Follow agile methodology
  - Implement continuous integration
  - Conduct regular code reviews


2. Testing Requirements
  - Unit testing for all components
  - Integration testing for service interactions
  - User acceptance testing for business requirements


3. Deployment Strategy
  - Automated deployment pipelines
  - Rollback procedures
  - Monitoring and logging setup



### B. Technical Considerations

1. Performance Optimization
  - Caching strategies
  - Database indexing
  - Resource optimization


2. Security Measures
  - Authentication protocols
  - Data encryption methods
  - Access control implementation


3. Scalability Factors
  - Horizontal scaling capabilities
  - Load balancing strategies
  - Resource allocation planning



### C. Maintenance Guidelines

1. Regular Updates
  - Security patches
  - Feature enhancements
  - Performance optimizations


2. Monitoring Requirements
  - System performance metrics
  - User activity tracking
  - Error logging and analysis


3. Backup Procedures
  - Data backup schedules
  - Recovery procedures
  - Disaster recovery planning
