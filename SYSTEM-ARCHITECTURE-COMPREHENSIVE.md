# 🔥 Fire Alarm System - Comprehensive Architecture Documentation

## 📋 Table of Contents
1. [System Overview](#system-overview)
2. [Architecture Design](#architecture-design)
3. [Technology Stack](#technology-stack)
4. [Database Design](#database-design)
5. [API Architecture](#api-architecture)
6. [MQTT Integration](#mqtt-integration)
7. [Real-time Features](#real-time-features)
8. [Security Implementation](#security-implementation)
9. [Deployment & Infrastructure](#deployment--infrastructure)
10. [Testing Strategy](#testing-strategy)
11. [Performance & Scalability](#performance--scalability)
12. [Monitoring & Analytics](#monitoring--analytics)

---

## 🏗️ System Overview

### Core Purpose
The Fire Alarm System is a comprehensive IoT-based fire detection and alert system that provides real-time monitoring, instant notifications, and intelligent response mechanisms for fire safety.

### Key Features
- **Real-time Fire Detection**: MQTT-based sensor communication
- **Instant Notifications**: FCM push notifications to nearby users
- **Spatial Intelligence**: Location-based alert distribution
- **Multi-tenant Architecture**: Enterprise-grade tenant management
- **WebSocket Real-time**: Live status updates and alerts
- **Advanced Analytics**: Performance monitoring and insights
- **Security & Access Control**: Role-based permissions and audit logs

---

## 🏛️ Architecture Design

### High-Level Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   IoT Sensors   │    │    Gateways     │    │   MQTT Broker   │
│   (Smoke, etc.) │────│   (LoRaWAN)     │────│   (EMQX)        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Clients   │    │  Backend API    │    │   Database      │
│   (React/Vue)   │────│   (Go/Gin)      │────│   (PostgreSQL)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Mobile Apps   │    │   FCM Service   │    │   Redis Cache   │
│   (Android/iOS) │────│   (Firebase)    │────│   (Session)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Microservices Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                              │
│                    (Nginx + SSL)                                │
└─────────────────────────────────────────────────────────────────┘
                                │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Auth Service  │    │  Core API       │    │  Notification   │
│   (JWT/OTP)     │    │  (CRUD)         │    │  Service        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   MQTT Bridge   │    │  WebSocket      │    │   Analytics     │
│   (IoT Data)    │    │  Manager        │    │   Service       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## 🛠️ Technology Stack

### Backend Technologies
- **Language**: Go (Golang) 1.21+
- **Framework**: Gin Web Framework
- **Database**: PostgreSQL 15+ with PostGIS extension
- **Cache**: Redis 7+ for session management
- **Message Broker**: EMQX MQTT Broker 5.3+
- **Authentication**: JWT + OTP (SMS/Email)
- **Push Notifications**: Firebase Cloud Messaging (FCM)

### Infrastructure & DevOps
- **Containerization**: Docker & Docker Compose
- **Reverse Proxy**: Nginx with SSL/TLS
- **SSL Certificates**: Let's Encrypt (production) / Self-signed (dev)
- **Monitoring**: Built-in health checks and metrics
- **Deployment**: Automated scripts for VPS deployment

### Development Tools
- **Testing**: Postman CLI (Newman) for API testing
- **Documentation**: Markdown-based comprehensive docs
- **Version Control**: Git with structured commits
- **Code Quality**: Go modules and linting

---

## 🗄️ Database Design

### Core Tables Structure

#### 1. Tenant Management (Enterprise Features)
```sql
-- Multi-tenant architecture
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    domain TEXT UNIQUE,
    status TEXT DEFAULT 'active',
    plan TEXT DEFAULT 'basic',
    max_users INTEGER DEFAULT 100,
    max_devices INTEGER DEFAULT 1000,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### 2. User Management
```sql
-- Users with tenant support
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    phone TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    email TEXT,
    password TEXT NOT NULL,
    fcm_token TEXT,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### 3. Spatial Data (PostGIS)
```sql
-- Houses with geographic location
CREATE TABLE houses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    address TEXT NOT NULL,
    location GEOMETRY(Point, 4326), -- PostGIS point
    user_id UUID REFERENCES users(id),
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### 4. IoT Device Management
```sql
-- Gateways (LoRaWAN concentrators)
CREATE TABLE gateways (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    serial_number TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    house_id UUID REFERENCES houses(id),
    location GEOMETRY(Point, 4326),
    status TEXT DEFAULT 'offline',
    battery_voltage REAL,
    signal_strength INTEGER,
    last_heartbeat TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- Sensors (Smoke detectors, etc.)
CREATE TABLE sensors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    serial_number TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    type TEXT NOT NULL, -- smoke, heat, co2
    gateway_id UUID REFERENCES gateways(id),
    house_id UUID REFERENCES houses(id),
    location GEOMETRY(Point, 4326),
    status TEXT DEFAULT 'active',
    battery_level INTEGER,
    last_maintenance TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### 5. Fire Alarm Events
```sql
-- Fire alarm incidents
CREATE TABLE fire_alarms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id UUID REFERENCES sensors(id),
    type TEXT NOT NULL, -- smoke, heat, test, maintenance
    severity TEXT NOT NULL, -- low, medium, high, critical
    status TEXT DEFAULT 'active', -- active, resolved, acknowledged
    data TEXT,
    triggered_at TIMESTAMPTZ DEFAULT now(),
    resolved_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### 6. Notification System
```sql
-- Notification history for audit
CREATE TABLE notification_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    alarm_id UUID REFERENCES fire_alarms(id),
    notification_type TEXT NOT NULL, -- fcm, email, sms
    status TEXT NOT NULL, -- sent, failed, pending
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

### Database Features
- **Spatial Queries**: PostGIS for location-based operations
- **JSON Support**: JSONB for flexible data storage
- **UUID Primary Keys**: Distributed-friendly identifiers
- **Audit Trails**: Comprehensive history tracking
- **Soft Deletes**: Data preservation with is_deleted flags

---

## 🔌 API Architecture

### RESTful API Design
```
Base URL: /api/v1
Authentication: Bearer JWT Token
Content-Type: application/json
```

### API Categories

#### 1. Authentication & Authorization
```
POST   /auth/register              # User registration
POST   /auth/request-otp           # Request OTP (SMS/Email)
POST   /auth/request-otp-phone     # Phone-specific OTP
POST   /auth/verify-otp            # Verify OTP
POST   /auth/login-password        # Password-based login
POST   /auth/refresh               # Refresh JWT token
GET    /users/me                   # Get user profile
```

#### 2. House Management
```
GET    /houses                     # List user's houses
POST   /houses                     # Create new house
GET    /houses/nearby              # Spatial query - nearby houses
GET    /houses/:id                 # Get house details
PUT    /houses/:id                 # Update house
DELETE /houses/:id                 # Delete house
GET    /houses/:id/sensors         # Get sensors by house
GET    /houses/:id/alerts          # Get alerts by house
GET    /houses/:id/members         # Get house members
```

#### 3. Gateway Management
```
GET    /gateways                   # List gateways
POST   /gateways                   # Create gateway
GET    /gateways/nearby            # Spatial query - nearby gateways
GET    /gateways/:id               # Get gateway details
PUT    /gateways/:id               # Update gateway
DELETE /gateways/:id               # Delete gateway
GET    /gateways/:id/sensors       # Get sensors by gateway
```

#### 4. Sensor Management
```
GET    /sensors                    # List sensors
POST   /sensors                    # Create sensor
GET    /sensors/:id                # Get sensor details
PUT    /sensors/:id                # Update sensor
DELETE /sensors/:id                # Delete sensor
GET    /sensors/:id/alerts         # Get alerts by sensor
GET    /sensors/pairing/:gateway_serial/:sensor_serial  # Pairing status
POST   /sensors/mock-mqtt          # Mock MQTT message
```

#### 5. Fire Alarm Management
```
GET    /fire-alarms                # List fire alarms
POST   /fire-alarms                # Create fire alarm
GET    /fire-alarms/:id            # Get alarm details
PATCH  /fire-alarms/:id/resolve    # Resolve alarm
POST   /fire-alarms/:id/acknowledge # Acknowledge alarm
POST   /fire-alarms/:id/escalate   # Escalate alarm
GET    /fire-alarms/:id/history    # Get alarm history
POST   /fire-alarms/:id/notify-emergency # Emergency notification
```

#### 6. Real-time Features
```
GET    /realtime/status            # Get real-time status
POST   /realtime/test-fire-alarm   # Test fire alarm
POST   /realtime/test-fire-alarm-with-sensor # Test with sensor
POST   /realtime/test-sensor-data  # Test sensor data
```

#### 7. Analytics & Monitoring
```
GET    /analytics/dashboard        # Live dashboard data
GET    /analytics/sensors          # Sensor performance
GET    /analytics/gateways         # Gateway health
GET    /analytics/performance      # Performance metrics
GET    /analytics/errors           # Error tracking
```

#### 8. Enterprise Features
```
# Tenant Management
POST   /tenants                    # Create tenant
GET    /tenants                    # List tenants
GET    /tenants/:id                # Get tenant details
POST   /tenants/:id/users          # Add user to tenant
GET    /tenants/:id/users          # Get tenant users

# Security & Access Control
POST   /security/audit             # Log audit event
GET    /security/audit             # Get audit logs
POST   /security/rate-limit        # Set rate limit
GET    /security/rate-limit        # Check rate limit

# Integration & Webhooks
POST   /integration/webhooks       # Create webhook
GET    /integration/webhooks       # List webhooks
POST   /integration/webhooks/:id/test # Test webhook
GET    /integration/webhooks/:id/history # Webhook history
```

### API Response Format
```json
{
  "success": true,
  "data": {
    // Response data
  },
  "message": "Operation successful",
  "timestamp": "2025-01-27T10:30:00Z"
}
```

---

## 📡 MQTT Integration

### MQTT Architecture
```
IoT Sensors → LoRaWAN Gateway → MQTT Broker (EMQX) → Backend Service
```

### MQTT Topics Structure
```
uplink/{gateway_id}/power_on      # Gateway power-on message
uplink/{gateway_id}/heartbeat      # Gateway heartbeat
uplink/{gateway_id}/smoke_alarm    # Fire alarm from sensor
uplink/{gateway_id}/smoke_register # Sensor registration
uplink/{gateway_id}/delete_response # Deletion confirmation
uplink/{gateway_id}/self_test      # Self-test message
uplink/{gateway_id}/low_battery    # Low battery warning
```

### Message Types

#### 1. Power-On Message
```json
{
  "gateway_id": "GW-001-2024",
  "hw_version": "1.2.0",
  "sw_version": "2.1.0",
  "battery_voltage": 12.5,
  "signal_strength": -45
}
```

#### 2. Smoke Alarm Message
```json
{
  "gateway_id": "GW-001-2024",
  "smoke_id": "SENSOR-001-2024",
  "battery_voltage": 3.2,
  "signal_strength": -52
}
```

#### 3. Heartbeat Message
```json
{
  "gateway_id": "GW-001-2024",
  "hw_version": "1.2.0",
  "sw_version": "2.1.0",
  "battery_voltage": 12.3,
  "signal_strength": -48
}
```

### MQTT Processing Flow
1. **Message Reception**: MQTT client receives messages
2. **Parsing**: Parse gateway-specific message format
3. **Validation**: Validate message structure and data
4. **Database Update**: Update sensor/gateway status
5. **Fire Alarm Creation**: Create fire alarm record if needed
6. **Notification Dispatch**: Send notifications to nearby users
7. **WebSocket Broadcast**: Real-time updates to connected clients

---

## ⚡ Real-time Features

### WebSocket Implementation
```go
// WebSocket connection management
type Manager struct {
    clients    map[string]*Client
    broadcast  chan *Message
    register   chan *Client
    unregister chan *Client
    mutex      sync.RWMutex
}
```

### Real-time Events
- **Fire Alarm Triggered**: Instant notification to all connected clients
- **Sensor Status Change**: Real-time sensor status updates
- **Gateway Health**: Live gateway connectivity status
- **System Alerts**: Maintenance and system notifications

### WebSocket Message Types
```json
{
  "type": "fire_alarm",
  "data": {
    "alarm_id": "uuid",
    "sensor_id": "uuid",
    "severity": "high",
    "location": "Kitchen",
    "timestamp": "2025-01-27T10:30:00Z"
  }
}
```

---

## 🔐 Security Implementation

### Authentication Methods
1. **JWT Token Authentication**: Stateless token-based auth
2. **OTP Verification**: SMS/Email one-time passwords
3. **Password-based Login**: Traditional username/password
4. **Token Refresh**: Automatic token renewal

### Security Features
- **Rate Limiting**: API endpoint protection
- **CORS Configuration**: Cross-origin request handling
- **Input Validation**: Request data sanitization
- **SQL Injection Prevention**: Parameterized queries
- **XSS Protection**: Content security headers
- **Audit Logging**: Comprehensive security audit trail

### Access Control
```go
// Role-based access control
type Role struct {
    ID   int    `json:"id"`
    Name string `json:"name"` // owner, fire_officer, police
}

type UserRole struct {
    ID     uuid.UUID `json:"id"`
    UserID uuid.UUID `json:"user_id"`
    RoleID int       `json:"role_id"`
}
```

---

## 🚀 Deployment & Infrastructure

### Docker Compose Architecture
```yaml
version: '3.8'
services:
  postgres:
    image: postgis/postgis:15-3.3
    environment:
      POSTGRES_DB: firealarm
      POSTGRES_USER: firealarm
      POSTGRES_PASSWORD: firealarm123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  emqx:
    image: emqx/emqx:5.3.0
    ports:
      - "1883:1883"    # MQTT
      - "8883:8883"    # MQTT/SSL
      - "8083:8083"    # MQTT/WebSocket
      - "18083:18083"  # Dashboard

  app:
    build: .
    ports:
      - "28080:8080"
    depends_on:
      - postgres
      - redis
      - emqx

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
```

### SSL/TLS Configuration
- **Development**: Self-signed certificates
- **Production**: Let's Encrypt certificates
- **Automatic Renewal**: Certbot integration
- **HTTP to HTTPS Redirect**: Secure by default

### Nginx Configuration
```nginx
# SSL configuration
server {
    listen 443 ssl http2;
    server_name api-test.theio.vn;
    
    ssl_certificate /etc/nginx/ssl/api-test.theio.vn.crt;
    ssl_certificate_key /etc/nginx/ssl/api-test.theio.vn.key;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    
    # API proxy
    location /api/ {
        proxy_pass http://app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # WebSocket support
    location /ws {
        proxy_pass http://app:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 🧪 Testing Strategy

### API Testing
- **Postman Collection**: Comprehensive API testing
- **Newman CLI**: Automated API testing
- **Test Data Management**: Realistic test scenarios
- **Performance Testing**: Load and stress testing

### Test Categories
1. **Unit Tests**: Individual function testing
2. **Integration Tests**: API endpoint testing
3. **End-to-End Tests**: Complete workflow testing
4. **Performance Tests**: Load and scalability testing

### Testing Tools
- **Postman**: API testing and documentation
- **Newman**: CLI-based test execution
- **Custom Scripts**: Automated test scenarios
- **Mock Data**: Realistic test data generation

---

## 📈 Performance & Scalability

### Performance Optimizations
- **Database Indexing**: Optimized queries with proper indexes
- **Caching Strategy**: Redis for session and data caching
- **Connection Pooling**: Database connection management
- **Async Processing**: Non-blocking operations

### Scalability Features
- **Horizontal Scaling**: Stateless API design
- **Load Balancing**: Nginx reverse proxy
- **Database Sharding**: Multi-tenant architecture
- **Microservices Ready**: Modular service design

### Monitoring Metrics
- **Response Times**: API performance tracking
- **Error Rates**: System reliability monitoring
- **Resource Usage**: CPU, memory, disk monitoring
- **Business Metrics**: User activity and alarm statistics

---

## 📊 Monitoring & Analytics

### Built-in Analytics
```go
// Analytics endpoints
GET /analytics/dashboard    # Live dashboard data
GET /analytics/sensors      # Sensor performance metrics
GET /analytics/gateways     # Gateway health status
GET /analytics/performance  # System performance metrics
GET /analytics/errors       # Error tracking and reporting
```

### Key Metrics
- **Fire Alarm Response Time**: Time from detection to notification
- **Sensor Uptime**: Sensor availability and reliability
- **Gateway Health**: Connectivity and battery status
- **User Engagement**: Notification delivery rates
- **System Performance**: API response times and throughput

### Health Monitoring
```go
// Health check endpoint
GET /health
Response: {
  "status": "healthy",
  "timestamp": "2025-01-27T10:30:00Z",
  "services": {
    "database": "connected",
    "redis": "connected",
    "mqtt": "connected"
  }
}
```

---

## 🎯 Conclusion

The Fire Alarm System represents a comprehensive IoT-based fire safety solution with:

### Key Strengths
- **Real-time Monitoring**: Instant fire detection and alerting
- **Spatial Intelligence**: Location-based notification distribution
- **Enterprise Ready**: Multi-tenant architecture with security
- **Scalable Design**: Microservices-based architecture
- **Comprehensive Testing**: Automated testing and validation
- **Production Ready**: SSL, monitoring, and deployment automation

### Technology Highlights
- **Modern Stack**: Go, PostgreSQL, Redis, MQTT
- **Real-time Capabilities**: WebSocket and MQTT integration
- **Security First**: JWT, OTP, audit logging, rate limiting
- **DevOps Ready**: Docker, Nginx, automated deployment
- **API First**: RESTful API with comprehensive documentation

### Future Enhancements
- **Machine Learning**: Predictive maintenance and anomaly detection
- **Mobile Apps**: Native iOS/Android applications
- **Advanced Analytics**: Business intelligence and reporting
- **Third-party Integrations**: Emergency services and building management systems
- **Edge Computing**: Local processing for reduced latency

This architecture provides a solid foundation for a production-ready fire alarm system that can scale from small installations to enterprise deployments. 