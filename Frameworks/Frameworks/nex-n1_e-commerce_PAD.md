# **PROJECT ARCHITECTURE DOCUMENT**
## **SINGAPORE SMB E-COMMERCE & INTEGRATED BUSINESS SYSTEM**

---

## **DOCUMENT CONTROL**

| **Version** | **Date** | **Author** | **Changes** |
|-------------|----------|------------|-------------|
| 1.0 | 2025-01-15 | AI Development Team | Initial comprehensive architecture document |
| 1.1 | TBD | [To be updated] | Future updates and revisions |

---

## **1. ARCHITECTURE OVERVIEW**

### **1.1 System Vision Statement**
Create a scalable, compliant, and performant e-commerce platform specifically designed for Singapore SMBs, integrating seamless inventory management, automated accounting, and full GST tax compliance within a Unified Business Management System (UBMS).

### **1.2 Architecture Principles**
1. **Compliance by Design**: Singapore regulatory compliance (PDPA, GST, IRAS) built into core architecture
2. **Scalability First**: Microservices architecture supporting 1,000+ concurrent transactions
3. **Mobile-First Philosophy**: Optimized for 70%+ mobile commerce traffic
4. **Real-Time Synchronization**: Event-driven architecture for seamless inventory-accounting integration
5. **Security by Default**: Zero-trust security model with comprehensive audit trails
6. **High Availability**: 99.9% uptime with automatic failover capabilities
7. **API-First Design**: Decoupled services enabling multi-channel expansion

### **1.3 System Context**
```
┌─────────────────────────────────────────────────────────────────┐
│                        EXTERNAL SYSTEMS                          │
├─────────────────────────────────────────────────────────────────┤
│ Customers │ Payment Gateways │ Logistics │ Accounting │ Social  │
│  (Mobile) │  (Stripe/PayNow) │ (NinjaVan)│  (Xero)   │  Media  │
└─────────────┬─────────────────┬───────────┬───────────┬─────────┘
              │                 │           │           │
              ▼                 ▼           ▼           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY & LOAD BALANCER                   │
│     Rate Limiting │ Authentication │ SSL Termination │ Caching   │
└──────────────────────────────┬─────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   E-COMMERCE    │  │    INVENTORY    │  │    ACCOUNTING   │
│   MICROSERVICE  │  │   MICROSERVICE  │  │   MICROSERVICE  │
│                 │  │                 │  │                 │
│ • Product Mgmt  │  │ • Multi-Warehouse│  │ • Automated GL  │
│ • Shopping Cart │  │ • Stock Control │  │ • GST Compliance│
│ • Order Mgmt    │  │ • Procurement   │  │ • Tax Reporting │
│ • Payment Int.  │  │ • PO Management │  │ • Financial Rpt  │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                              ▼
         ┌─────────────────────────────────────────┐
         │      SHARED DATA LAYER & MESSAGE BUS    │
         ├─────────────────────────────────────────┤
         │ PostgreSQL │ Redis │ Elasticsearch │ S3  │
         │ (Primary)  │(Cache)│   (Search)    │(Files)│
         └─────────────────────────────────────────┘
```

---

## **2. SYSTEM ARCHITECTURE**

### **2.1 High-Level Architecture**

The system follows a **Microservices Architecture Pattern** with:

#### **2.1.1 Communication Pattern: Event-Driven Architecture**
- **Event Bus**: Apache Kafka for async communication
- **Event Store**: Immutable audit trail for all business events
- **CQRS Pattern**: Separate read/write models for performance optimization
- **Saga Pattern**: Distributed transaction management across microservices

#### **2.1.2 Service Decomposition Strategy**
```
┌─────────────────────────────────────────────────────────────────┐
│                     API GATEWAY (Kong/AWS)                       │
│             Authentication │ Authorization │ Rate Limiting       │
└────────────────────────┬────────────────────────────────────────┘
                         │
    ┌────────────────────┼────────────────────┬──────────────────┐
    │                    │                    │                  │
    ▼                    ▼                    ▼                  ▼
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│E-COMMERCE│ │INVENTORY│ │ACCOUNTING│ │NOTIFICATION│ │AUDIT    │
│SERVICE  │ │SERVICE  │ │SERVICE   │ │SERVICE    │ │SERVICE   │
│         │ │         │ │          │ │           │ │          │
│Port:    │ │Port:    │ │Port:     │ │Port:      │ │Port:     │
│3001     │ │3002     │ │3003      │ │3004       │ │3005      │
└────┬────┘ └────┬────┘ └────┬─────┘ └─────┬────┘ └────┬─────┘
     │           │          │            │          │
     └───────────┼──────────┼────────────┼──────────┘
                 │          │            │
                 └──────────┼────────────┘
                            │
                 ┌──────────┴─────────┐
                 ▼                    ▼
         ┌──────────────┐    ┌──────────────┐
         │   KAFKA      │    │    REDIS     │
         │ EVENT BUS    │    │ CLUSTER      │
         └──────────────┘    └──────────────┘
                 │                    │
                 └────────────────────┘
                            │
                 ┌──────────┴─────────┐
                 ▼                    ▼
         ┌──────────────┐    ┌──────────────┐
         │ POSTGRESQL   │    │ ELASTICSEARCH│
         │ CLUSTER      │    │ CLUSTER      │
         └──────────────┘    └──────────────┘
```

### **2.2 Component Architecture**

#### **2.2.1 E-Commerce Service (Node.js + Express.js)**
```typescript
// E-Commerce Service Architecture
interface ECommerceService {
  // Core Domain Logic
  productCatalog: {
    getProducts(filters, pagination): Product[];
    getProductVariants(sku): Variant[];
    validateInventory(products): InventoryStatus[];
  };
  
  cartManagement: {
    createCart(userId): Cart;
    addItem(cartId, productId, quantity): Cart;
    applyPromotion(cartId, code): Cart;
    calculateTotals(cartId): CartTotals;
  };
  
  orderProcessing: {
    createOrder(cartId, checkoutDetails): Order;
    validatePayment(orderId, paymentDetails): PaymentResult;
    processOrder(orderId): OrderStatus;
  };
}
```

**Database Schema (E-Commerce):**
```sql
-- Products and Inventory
products (
  id UUID PRIMARY KEY,
  sku VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category_id UUID,
  brand VARCHAR(100),
  status product_status,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

product_variants (
  id UUID PRIMARY KEY,
  product_id UUID REFERENCES products(id),
  sku VARCHAR(50) UNIQUE NOT NULL,
  attributes JSONB, -- {color, size, weight}
  price DECIMAL(10,2),
  compare_price DECIMAL(10,2),
  cost_price DECIMAL(10,2),
  track_inventory BOOLEAN DEFAULT true,
  inventory_quantity INTEGER,
  created_at TIMESTAMP
);

-- Orders and Checkout
orders (
  id UUID PRIMARY KEY,
  order_number VARCHAR(20) UNIQUE,
  customer_id UUID,
  status order_status,
  subtotal DECIMAL(10,2),
  tax_amount DECIMAL(10,2),
  shipping_amount DECIMAL(10,2),
  total_amount DECIMAL(10,2),
  currency VARCHAR(3) DEFAULT 'SGD',
  gst_rate DECIMAL(5,2) DEFAULT 0.09,
  gst_amount DECIMAL(10,2),
  created_at TIMESTAMP
);

order_items (
  id UUID PRIMARY KEY,
  order_id UUID REFERENCES orders(id),
  product_variant_id UUID,
  quantity INTEGER,
  unit_price DECIMAL(10,2),
  line_total DECIMAL(10,2),
  gst_amount DECIMAL(10,2)
);
```

#### **2.2.2 Inventory Service (Node.js + Express.js)**
```typescript
// Inventory Service Architecture
interface InventoryService {
  // Multi-location Inventory Management
  warehouseManagement: {
    getStockLevels(location, skus): StockLevel[];
    updateInventory(sku, location, quantity): InventoryUpdate;
    transferStock(from, to, sku, quantity): TransferOrder;
    reserveInventory(orderId, items): ReservationResult;
  };
  
  // Procurement and Purchase Orders
  procurement: {
    createPurchaseOrder(vendorId, items): PurchaseOrder;
    receiveGoods(poId, items, location): ReceivingResult;
    trackVendorPerformance(vendorId): VendorMetrics;
  };
  
  // Reporting and Analytics
  analytics: {
    calculateTurnoverRate(location, period): number;
    identifySlowMoving(sku, threshold): SlowMovingItem[];
    forecastDemand(sku, period): DemandForecast;
  };
}
```

**Database Schema (Inventory):**
```sql
-- Multi-location Warehouse Management
warehouses (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  code VARCHAR(10) NOT NULL,
  address JSONB, -- Singapore address format
  contact_info JSONB,
  capacity_sqm DECIMAL(10,2),
  status warehouse_status,
  created_at TIMESTAMP
);

warehouse_locations (
  id UUID PRIMARY KEY,
  warehouse_id UUID REFERENCES warehouses(id),
  zone VARCHAR(50),
  aisle VARCHAR(10),
  shelf INTEGER,
  bin VARCHAR(10),
  is_active BOOLEAN DEFAULT true
);

inventory_items (
  id UUID PRIMARY KEY,
  product_variant_id UUID REFERENCES product_variants(id),
  warehouse_location_id UUID REFERENCES warehouse_locations(id),
  quantity INTEGER NOT NULL,
  reserved_quantity INTEGER DEFAULT 0,
  reorder_point INTEGER,
  max_stock_level INTEGER,
  last_count_date DATE,
  created_at TIMESTAMP,
  UNIQUE(product_variant_id, warehouse_location_id)
);

stock_movements (
  id UUID PRIMARY KEY,
  inventory_item_id UUID REFERENCES inventory_items(id),
  movement_type movement_type, -- IN, OUT, TRANSFER, ADJUSTMENT
  quantity INTEGER NOT NULL,
  reference_type VARCHAR(50), -- ORDER, PO, TRANSFER, ADJUSTMENT
  reference_id UUID,
  reason VARCHAR(255),
  created_by UUID,
  created_at TIMESTAMP
);

-- Procurement Management
suppliers (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  company_registration VARCHAR(50),
  payment_terms INTEGER,
  address JSONB,
  contact_person JSONB,
  performance_score DECIMAL(3,2),
  created_at TIMESTAMP
);

purchase_orders (
  id UUID PRIMARY KEY,
  po_number VARCHAR(20) UNIQUE NOT NULL,
  supplier_id UUID REFERENCES suppliers(id),
  status po_status,
  order_date DATE,
  expected_delivery DATE,
  total_amount DECIMAL(10,2),
  currency VARCHAR(3) DEFAULT 'SGD',
  created_by UUID,
  created_at TIMESTAMP
);

purchase_order_items (
  id UUID PRIMARY KEY,
  purchase_order_id UUID REFERENCES purchase_orders(id),
  product_variant_id UUID,
  quantity INTEGER,
  unit_cost DECIMAL(10,4),
  line_total DECIMAL(10,2),
  received_quantity INTEGER DEFAULT 0
);
```

#### **2.2.3 Accounting Service (Node.js + Express.js)**
```typescript
// Accounting Service Architecture
interface AccountingService {
  // Automated Bookkeeping
  transactionProcessing: {
    recordSale(orderId): JournalEntry;
    recordPurchase(poId): JournalEntry;
    recordPayment(paymentId): JournalEntry;
    performBankReconciliation(accountId, statement): ReconciliationResult;
  };
  
  // GST/SG Tax Compliance
  gstManagement: {
    calculateGST(saleAmount, taxCode): GSTCalculation;
    generateGSTReturn(period): GSTF5Return;
    reconcileGSTRReport(gstReturnId): ReconciliationResult;
  };
  
  // Financial Reporting
  reporting: {
    generatePAndL(period, departments): ProfitAndLoss;
    generateBalanceSheet(asOfDate): BalanceSheet;
    generateCashFlow(period): CashFlowStatement;
  };
}
```

**Database Schema (Accounting):**
```sql
-- Chart of Accounts (Singapore Specific)
accounts (
  id UUID PRIMARY KEY,
  account_code VARCHAR(10) NOT NULL,
  account_name VARCHAR(255) NOT NULL,
  account_type account_type, -- ASSET, LIABILITY, EQUITY, REVENUE, EXPENSE
  parent_account_id UUID REFERENCES accounts(id),
  gst_code VARCHAR(10), -- TX, TZ, ES, ZP, etc.
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP
);

-- General Ledger
journal_entries (
  id UUID PRIMARY KEY,
  entry_date DATE NOT NULL,
  reference_number VARCHAR(50),
  reference_type VARCHAR(30), -- ORDER, INVOICE, PAYMENT, etc.
  reference_id UUID,
  description VARCHAR(500),
  total_debit DECIMAL(12,2),
  total_credit DECIMAL(12,2),
  status entry_status,
  posted_at TIMESTAMP,
  created_at TIMESTAMP
);

journal_entry_items (
  id UUID PRIMARY KEY,
  journal_entry_id UUID REFERENCES journal_entries(id),
  account_id UUID REFERENCES accounts(id),
  debit_amount DECIMAL(12,2),
  credit_amount DECIMAL(12,2),
  gst_amount DECIMAL(10,2),
  tax_code VARCHAR(10),
  description VARCHAR(255)
);

-- GST Tracking and Reporting
gst_returns (
  id UUID PRIMARY KEY,
  return_period VARCHAR(7), -- YYYY-MM format
  filing_status filing_status,
  box_1_output_tax DECIMAL(12,2), -- Standard-rated supplies
  box_2_output_tax DECIMAL(12,2), -- Zero-rated supplies
  box_6_tax_due DECIMAL(12,2),
  box_7_input_tax DECIMAL(12,2),
  box_8_net_tax DECIMAL(12,2),
  box_9_refund DECIMAL(12,2),
  submitted_at TIMESTAMP,
  iras_reference VARCHAR(50),
  created_at TIMESTAMP
);

gst_transactions (
  id UUID PRIMARY KEY,
  gst_return_id UUID REFERENCES gst_returns(id),
  transaction_date DATE,
  transaction_type gst_transaction_type,
  document_number VARCHAR(50),
  amount DECIMAL(12,2),
  gst_amount DECIMAL(10,2),
  gst_rate DECIMAL(5,2),
  supplier_customer_name VARCHAR(255),
  gst_registration_number VARCHAR(20)
);
```

---

## **3. DATA ARCHITECTURE**

### **3.1 Database Architecture**

#### **3.1.1 Primary Database: PostgreSQL 15 Cluster**
```yaml
Database Configuration:
  Primary Cluster:
    - Primary Node: db-primary-sg-1a (Singapore)
    - Read Replicas: db-read-1b, db-read-1c (Singapore)
    - Standby: db-standby-malaysia (Malaysia DR site)
  
  Performance Settings:
    connection_pool_size: 200
    max_connections: 500
    shared_buffers: 4GB
    effective_cache_size: 12GB
    maintenance_work_mem: 1GB
    checkpoint_completion_target: 0.9
    wal_buffers: 16MB
    default_statistics_target: 100
  
  Replication:
    - Streaming replication (async)
    - Lag monitoring with < 100ms target
    - Automatic failover with Patroni
```

#### **3.1.2 Cache Layer: Redis Cluster**
```yaml
Redis Cluster Configuration:
  Cluster Setup:
    - 3 Master nodes (each with 1 replica)
    - 6 total nodes across 3 availability zones
  
  Use Cases:
    - Session storage (TTL: 24 hours)
    - Product catalog caching (TTL: 4 hours)
    - Shopping cart temporary storage (TTL: 7 days)
    - API rate limiting counters
    - Real-time inventory availability
  
  Performance:
    - < 1ms read/write latency (p99)
    - 99.95% availability
    - Automatic failover with Sentinel
```

#### **3.1.3 Search Engine: Elasticsearch Cluster**
```yaml
Elasticsearch Configuration:
  Cluster Setup:
    - 3 Master nodes (HA)
    - 6 Data nodes (hot/warm architecture)
    - 2 Coordinating nodes
  
  Indexing Strategy:
    products: Daily incremental, full sync weekly
    orders: Real-time indexing
    customers: Hourly sync
    inventory_search: Every 15 minutes
  
  Search Features:
    - Typo tolerance (Levenshtein distance: 2)
    - Synonym matching (Singapore English variations)
    - Faceted search with real-time filters
    - Auto-complete with analytics-driven suggestions
```

### **3.2 Data Partitioning Strategy**

#### **3.2.1 Horizontal Partitioning (Sharding)**
```sql
-- Sharding by Customer ID modulo 16
CREATE TABLE orders (
  id UUID,
  customer_id UUID,
  order_data JSONB,
  created_at TIMESTAMP
) PARTITION BY HASH(customer_id);

-- Create 16 partitions
CREATE TABLE orders_p0 PARTITION OF orders FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE orders_p1 PARTITION OF orders FOR VALUES WITH (MODULUS 16, REMAINDER 1);
-- ... continue for all 16 partitions
```

#### **3.2.2 Time-Based Partitioning**
```sql
-- Monthly partitions for audit logs
CREATE TABLE audit_logs (
  id UUID,
  entity_type VARCHAR(50),
  entity_id UUID,
  action VARCHAR(50),
  old_value JSONB,
  new_value JSONB,
  created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

-- Auto-create partitions with pg_partman
```

### **3.3 Data Integrity and Consistency**

#### **3.3.1 Eventual Consistency Strategy (CQRS)**
```typescript
// Event sourcing pattern for inventory consistency
class InventoryEventHandler {
  async handleStockReservation(e: OrderCreatedEvent) {
    // Optimistic locking with versioning
    await db.transaction(async (tx) => {
      const inventory = await tx.inventory_items.find({
        where: { product_variant_id: e.items[0].variantId },
        for: 'UPDATE'
      });
      
      if (inventory.quantity < e.items[0].quantity) {
        throw new OutOfStockError();
      }
      
      // Update read model optimistically
      await tx.inventory_items.update({
        where: { id: inventory.id },
        data: { quantity: { decrement: e.items[0].quantity } }
      });
      
      // Create compensating event for rollback
      await this.eventStore.append(new StockReservedEvent(
        e.orderId, inventory.id, e.items[0].quantity
      ));
    });
  }
}
```

---

## **4. API ARCHITECTURE**

### **4.1 RESTful API Design**

#### **4.1.1 API Gateway Configuration**
```yaml
# Kong API Gateway Configuration
services:
  - name: ecommerce-service
    url: http://ecommerce-service.internal
    routes:
      - name: products
        paths: /api/v1/products
        methods: [GET, POST, PUT, DELETE]
      
      - name: cart
        paths: /api/v1/cart
        methods: [GET, POST, PUT, DELETE]
        strip_path: true
        plugins:
          - name: rate-limiting
            config:
              minute: 100
              policy: local
              
          - name: jwt-auth
            config:
              claims_to_verify: [exp, nbf]
              
          - name: cors
            config:
              origins: ["https://*.shopify.com", "https://mystore.com"]
              methods: [GET, POST, PUT, DELETE, OPTIONS]
              headers: [Content-Type, Authorization, x-request-id]
```

#### **4.1.2 API Endpoint Specifications**

**Product Service Endpoints:**
```typescript
// Product Catalog
GET /api/v1/products
GET /api/v1/products/:id
GET /api/v1/categories/:id/products
POST /api/v1/products/search

// Product Variants
GET /api/v1/products/:id/variants
POST /api/v1/variants/:id/inventory-check

// Product Images and Media
POST /api/v1/products/:id/images
DELETE /api/v1/products/:id/images/:imageId

// Product Reviews
GET /api/v1/products/:id/reviews
POST /api/v1/products/:id/reviews
PUT /api/v1/reviews/:reviewId
```

**Inventory Service Endpoints:**
```typescript
// Stock Management
GET /api/v1/inventory/stock-levels
GET /api/v1/inventory/:productId/availability
POST /api/v1/inventory/reservations
PUT /api/v1/inventory/adjustments

// Multi-location Management
GET /api/v1/warehouses
GET /api/v1/warehouses/:id/locations
POST /api/v1/inventory/transfers

// Purchase Orders
GET /api/v1/purchase-orders
POST /api/v1/purchase-orders
GET /api/v1/purchase-orders/:id
PUT /api/v1/purchase-orders/:id/receive

// Suppliers
GET /api/v1/suppliers
POST /api/v1/suppliers
```

**Accounting Service Endpoints:**
```typescript
// Transaction Processing
POST /api/v1/transactions/sales
POST /api/v1/transactions/purchases
POST /api/v1/transactions/payments
GET /api/v1/transactions/:id

// GST and Tax Management
GET /api/v1/gst/returns
POST /api/v1/gst/returns
GET /api/v1/gst/transactions
POST /api/v1/gst/transactions

// Financial Reporting
GET /api/v1/reports/profit-loss
GET /api/v1/reports/balance-sheet
GET /api/v1/reports/cash-flow
GET /api/v1/reports/trial-balance
```

### **4.2 GraphQL Schema Design**

#### **4.2.1 Unified GraphQL Schema**
```graphql
type Query {
  # Product Queries
  products(filters: ProductFilters, pagination: Pagination): ProductConnection
  product(id: ID!): Product
  categories: [Category]
  
  # Inventory Queries
  inventoryStatus(productId: ID!): InventoryStatus
  warehouseStock(warehouseId: ID!): [StockItem]
  
  # Orders and Customers
  order(id: ID!): Order
  customer(id: ID!): Customer
  customerOrders(customerId: ID!, status: OrderStatus): [Order]
  
  # Accounting
  financialReport(type: ReportType!, period: DateRange!): FinancialReport
  gstReturn(period: String!): GSTReturn
}

type Mutation {
  # E-commerce
  addToCart(input: AddToCartInput!): Cart
  updateCartItem(input: UpdateCartItemInput!): Cart
  createOrder(input: CreateOrderInput!): Order
  
  # Inventory
  adjustInventory(input: InventoryAdjustmentInput!): StockMovement
  createPurchaseOrder(input: CreatePurchaseOrderInput!): PurchaseOrder
  
  # Accounting
  recordPayment(input: PaymentInput!): Payment
  reconcileTransaction(input: ReconciliationInput!): Reconciliation
}

type Subscription {
  # Real-time Updates
  orderStatusChanged(orderId: ID!): Order
  inventoryChanged(productId: ID!): StockUpdate
  paymentProcessed(customerId: ID!): Payment
}
```

---

## **5. SECURITY ARCHITECTURE**

### **5.1 Identity and Access Management**

#### **5.1.1 Authentication Strategy**
```typescript
// JWT-based Authentication with Refresh Tokens
interface AuthTokens {
  accessToken: string;  // 15-minute TTL
  refreshToken: string; // 30-day TTL
  tokenType: 'Bearer';
}

// Multi-Factor Authentication
interface MFAConfig {
  totpEnabled: boolean;
  smsEnabled: boolean;
  emailEnabled: boolean;
  backupCodes: string[];
}

// Password Policy (PDPA Compliant)
const passwordPolicy = {
  minLength: 12,
  requireUppercase: true,
  requireLowercase: true,
  requireNumbers: true,
  requireSpecialChars: true,
  noReuseOfLast5: true,
  complexity: 'high',
  expiryDays: 90
};
```

#### **5.1.2 RBAC (Role-Based Access Control)**
```yaml
# Role Definitions
roles:
  - name: customer
    permissions:
      - products.read
      - cart.create
      - order.create
      
  - name: warehouse_staff
    inherits: [customer]
    permissions:
      - inventory.read
      - inventory.update
      - stock_adjustments.create
      
  - name: accountant
    permissions:
      - transactions.read
      - transactions.create
      - financial_reports.read
      - gst_returns.create
      
  - name: admin
    inherits: [warehouse_staff, accountant]
    permissions:
      - users.manage
      - system.config
      - audit_logs.read
```

### **5.2 Data Protection and Encryption**

#### **5.2.1 Encryption Strategy**
```yaml
Encryption Layers:
  # At Rest
  Database:
    - PostgreSQL TDE (Transparent Data Encryption)
    - Column-level encryption for PII data
    - Encrypted backups with AES-256-GCM
  
  File Storage:
    - S3 Server-Side Encryption (SSE-KMS)
    - Client-side encryption for sensitive files
    - Key rotation every 90 days
  
  # In Transit
  Network:
    - TLS 1.3 for all connections
    - Certificate pinning for mobile apps
    - HSTS headers with 1-year max-age
```

#### **5.2.2 PDPA Compliance Features**
```typescript
// Personal Data Protection (Singapore PDPA)
interface PersonalDataProtection {
  // Consent Management
  customerConsents: {
    marketing: boolean;
    analytics: boolean;
    thirdPartySharing: boolean;
    timestamp: Date;
    version: string;
  };
  
  // Data Residency
  dataResidency: 'SG' | 'APAC' | 'GLOBAL';
  
  // Retention Policies
  retentionPolicies: {
    customerData: '7_years_after_last_transaction';
    auditLogs: '10_years';
    marketingData: '3_years';
    gdprCompliance: boolean;
  };
  
  // Right to Erasure
  dataPortability: {
    exportCustomerData(customerId: string): Promise<DataExport>;
    deleteCustomerData(customerId: string): Promise<DeletionResult>;
  };
}
```

### **5.3 Security Monitoring and Incident Response**

#### **5.3.1 Real-time Security Monitoring**
```yaml
Security Monitoring Stack:
  # SIEM and Log Analysis
  - ELK Stack (Elasticsearch, Logstash, Kibana)
  - AWS CloudTrail for API audit logs
  - CloudWatch for application logs
  
  # Real-time Alerting
  Critical Alerts:
    - Multiple failed login attempts (5 in 5 minutes)
    - Unusual API usage patterns
    - Database access from unfamiliar IPs
    - GDRP/PDPA data access violations
    - Unauthorized file access attempts
  
  Alert Channels:
    - PagerDuty for critical incidents
    - Slack for warnings
    - Email for informational
    - SMS for authenticated attacks
```

#### **5.3.2 Incident Response Procedures**
```yaml
Incident Response Flow:
  Level 1 - Alert Detection:
    - Automated monitoring detects anomaly
    - Initial threat classification
    - Alert sent to on-call engineer
    
  Level 2 - Investigation:
    - Incident declared by lead engineer
    - Security team notified
    - Relevant logs collected
    - Impact assessment conducted
    
  Level 3 - Response:
    - Immediate mitigation deployed
    - Affected services isolated
    - Customer communication prepared
    - Legal/Privacy team engaged
    
  Level 4 - Post-Incident:
    - Forensic analysis completed
    - Incident report published
    - Remediation plan implemented
    - Security controls updated
```

---

## **6. SCALABILITY AND PERFORMANCE**

### **6.1 Horizontal Scaling Strategy**

#### **6.1.1 Auto-Scaling Configuration**
```yaml
# EKS Auto-Scaling Configuration
autoScalingGroups:
  - name: ecommerce-service-asg
    minSize: 3
    maxSize: 50
    desiredSize: 5
    scalingPolicies:
      - name: cpu-scaling
        metric: CPUUtilization
        target: 70%
        scaleOut: { cooldown: 60, adjustment: 2 }
        scaleIn: { cooldown: 300, adjustment: -1 }
      
      - name: request-scaling
        metric: RequestCountPerTarget
        target: 1000
        scaleOut: { cooldown: 60, adjustment: 3 }
        scaleIn: { cooldown: 300, adjustment: -2 }
```

#### **6.1.2 Database Read Replicas**
```sql
-- Read Replica Distribution Strategy
-- Primary database (Singapore): Write operations only
-- Read Replicas:
--   - db-read-1a (Singapore AZ-1a): 40% read traffic
--   - db-read-1b (Singapore AZ-1b): 40% read traffic  
--   - db-read-1c (Malaysia): 20% read traffic (disaster recovery)

-- Application-level read distribution
CREATE POLICY distribute_reads ON ALL TABLES
FOR SELECT
USING (
  CASE 
    WHEN current_setting('session.read_only') = 'true' 
    THEN random() < 0.8  -- 80% to local replicas
    ELSE true            -- All to primary
  END
);
```

### **6.2 Caching Strategy**

#### **6.2.1 Multi-Layer Caching Architecture**
```yaml
Caching Layers:
  # Layer 1: CDN Cache (CloudFront)
  cdn:
    ttl: 86400  # 24 hours for static assets
    paths:
      - /static/*
      - /images/*
      - /css/*
      - /js/*
  
  # Layer 2: Application Cache (Redis)
  application:
    redis:
      ttl: 14400  # 4 hours for product data
      cached_entities:
        - products
        - categories
        - stock_status
        - pricing_rules
  
  # Layer 3: Database Cache (PostgreSQL)
  database:
    shared_buffers: 4GB
    effective_cache_size: 12GB
    query_cache:
      enabled: true
      min_cached_queries: 1000
      ttl: 3600
```

#### **6.2.2 Cache Invalidation Strategy**
```typescript
// Cache invalidation events
class CacheInvalidationService {
  async invalidateProductCache(productId: string) {
    const keys = [
      `product:${productId}`,
      `product:${productId}:variants`,
      `product:${productId}:reviews`,
      'product:search:*', // Invalidate all search caches
      'category:products:*'
    ];
    
    await this.redis.del(...keys);
    await this.cdn.purge(`/products/${productId}`);
  }
  
  async invalidateInventoryCache(stockMovement: StockMovement) {
    const keys = [
      `inventory:${stockMovement.productVariantId}`,
      `availability:${stockMovement.productVariantId}`,
      'warehouse:stock:*'
    ];
    
    await this.redis.del(...keys);
    await this.eventBus.publish(new InventoryChangedEvent(stockMovement));
  }
}
```

---

## **7. DEPLOYMENT ARCHITECTURE**

### **7.1 Containerization Strategy**

#### **7.1.1 Docker Container Configuration**
```dockerfile
# Multi-stage build for production optimization
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .

# Security hardening
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    chown -R nodejs:nodejs /app

USER nodejs
EXPOSE 3001
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3001/health || exit 1

CMD ["npm", "start"]
```

#### **7.1.2 Kubernetes Deployment Manifests**
```yaml
# ecommerce-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-service
  namespace: production
  labels:
    app: ecommerce-service
    version: v1.2.0
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ecommerce-service
  template:
    metadata:
      labels:
        app: ecommerce-service
        version: v1.2.0
    spec:
      containers:
      - name: ecommerce-service
        image: mycompany/ecommerce-service:1.2.0
        ports:
        - containerPort: 3001
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-service-svc
  namespace: production
spec:
  selector:
    app: ecommerce-service
  ports:
  - port: 80
    targetPort: 3001
    protocol: TCP
  type: LoadBalancer

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### **7.2 CI/CD Pipeline**

#### **7.2.1 GitHub Actions Workflow**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
    tags: [v*]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
    - uses: actions/checkout@v3
    - name: Install dependencies
      run: npm ci
    - name: Run unit tests
      run: npm run test:unit
    - name: Run integration tests
      run: npm run test:integration
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
    - name: Code coverage
      run: npm run test:coverage
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3

  security-scan:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Run Snyk to check for vulnerabilities
      run: |
        npx snyk auth ${{ secrets.SNYK_TOKEN }}
        npx snyk test --severity-threshold=high
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.IMAGE_NAME }}
        format: 'sarif'
        output: 'trivy-results.sarif'
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  build-and-deploy:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        platforms: linux/amd64
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
        cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  deploy-staging:
    needs: build-and-deploy
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - name: Deploy to staging
      uses: steebchen/kubectl@v2.0.0
      with:
        config: ${{ secrets.KUBE_CONFIG_STAGING }}
        command: |
          set -ex
          kubectl set image deployment/ecommerce-service-staging \
            ecommerce-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main
          kubectl rollout status deployment/ecommerce-service-staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
    - name: Deploy to production
      uses: steebchen/kubectl@v2.0.0
      with:
        config: ${{ secrets.KUBE_CONFIG_PRODUCTION }}
        command: |
          set -ex
          kubectl set image deployment/ecommerce-service \
            ecommerce-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main
          kubectl rollout status deployment/ecommerce-service
```

---

## **8. MONITORING AND OBSERVABILITY**

### **8.1 Monitoring Stack**

#### **8.1.1 Metrics Collection (Prometheus + Grafana)**
```yaml
# prometheus-config.yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
  
  - job_name: 'application'
    static_configs:
      - targets: ['ecommerce-service:3001/metrics']
    metrics_path: '/metrics'
```

#### **8.1.2 Application Performance Monitoring (New Relic)**
```typescript
// New Relic Integration
import newrelic from 'newrelic';

// Custom business metrics
newrelic.recordCustomEvent('EcommerceTransaction', {
  orderId: order.id,
  customerId: order.customerId,
  totalAmount: order.totalAmount,
  itemCount: order.items.length,
  currency: order.currency,
  salesChannel: order.salesChannel,
  gstAmount: order.gstAmount
});

// Custom metrics
newrelic.recordMetric('Custom/Inventory/TurnoverRate', inventoryTurnover);
newrelic.recordMetric('Custom/Sales/ConversionRate', conversionRate);
newrelic.recordMetric('Custom/GST/TaxCollected', totalGSTCollected);
```

### **8.2 Logging Architecture**

#### **8.2.1 Centralized Logging (ELK Stack)**
```yaml
# Filebeat Configuration
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - '/var/log/containers/*.log'
  json.keys_under_root: true
  json.add_error_key: true

output.elasticsearch:
  hosts: ['https://elasticsearch-logging:9200']
  username: 'filebeat'
  password: '${FILEBEAT_PASSWORD}'
  index: 'ecommerce-logs-%{+yyyy.MM.dd}'

processors:
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
  - drop_event:
      when:
        equals:
          kubernetes.namespace: 'kube-system'
```

#### **8.2.2 Structured Logging**
```typescript
// Pino logger with structured logging
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level(label, number) {
      return { level: label };
    },
    bindings(bindings) {
      return {
        pid: bindings.pid,
        hostname: bindings.hostname,
        node_version: process.version,
      };
    }
  },
  redact: {
    paths: ['password', '*.password', '*.token', 'authorization'],
    censor: '[REDACTED]'
  },
  timestamp: pino.stdTimeFunctions.isoTime
});

// Usage with context
logger.info(
  { orderId, customerId, totalAmount, gstAmount },
  'Order created successfully'
);

logger.error(
  { err: error, orderId, paymentId },
  'Payment processing failed'
);
```

---

## **9. IMPLEMENTATION ROADMAP**

### **Phase 1: Foundation Setup (Weeks 1-4)**

#### **Week 1-2: Infrastructure Provisioning**
- [ ] Set up AWS account with Singapore region
- [ ] Configure VPC with public/private subnets across 3 AZs
- [ ] Deploy EKS cluster with managed node groups
- [ ] Set up RDS PostgreSQL cluster with read replicas
- [ ] Configure Redis Elasticache cluster
- [ ] Deploy Elasticsearch cluster for search and logging
- [ ] Set up S3 buckets with proper IAM policies
- [ ] Configure CloudFront CDN distribution

#### **Week 3-4: CI/CD and Monitoring Setup**
- [ ] Set up GitHub Actions CI/CD pipeline
- [ ] Deploy Prometheus and Grafana
- [ ] Configure ELK stack for centralized logging
- [ ] Set up New Relic APM integration
- [ ] Deploy Kong API Gateway
- [ ] Configure SSL certificates with AWS Certificate Manager
- [ ] Set up automated backups and disaster recovery procedures

### **Phase 2: Core Services Development (Weeks 5-12)**

#### **Week 5-6: Database Schema and ORM Setup**
- [ ] Design and implement PostgreSQL database schemas
- [ ] Create Prisma schema for all services
- [ ] Implement database migrations with version control
- [ ] Set up database seeding scripts
- [ ] Configure read replica connection pools
- [ ] Implement Redis caching layer for products and inventory

#### **Week 7-8: E-Commerce Service**
- [ ] Implement product catalog API endpoints
- [ ] Build shopping cart management system
- [ ] Develop checkout and order processing logic
- [ ] Integrate Stripe and PayNow payment gateways
- [ ] Add user authentication and authorization
- [ ] Implement product search with Elasticsearch
- [ ] Add product image upload and CDN integration

#### **Week 9-10: Inventory Service**
- [ ] Develop multi-location warehouse management
- [ ] Build stock movement tracking system
- [ ] Implement purchase order management
- [ ] Add supplier management functionality
- [ ] Develop inventory reporting and analytics
- [ ] Integrate with Ninja Van and SingPost APIs
- [ ] Implement barcode/QR code scanning support

#### **Week 11-12: Accounting Service**
- [ ] Develop automated journal entry generation
- [ ] Implement GST calculation and tracking
- [ ] Build financial reporting system
- [ ] Add Xero and QuickBooks integration
- [ ] Implement tax invoice generation
- [ ] Develop reconciliation features
- [ ] Create IRAS-compliant GST return preparation

### **Phase 3: Frontend Development (Weeks 13-20)**

#### **Week 13-14: Next.js Setup and Design System**
- [ ] Set up Next.js 14 with App Router
- [ ] Create Tailwind CSS design system
- [ ] Implement shadcn/ui component library
- [ ] Set up TypeScript configuration
- [ ] Create reusable UI components
- [ ] Implement responsive design patterns
- [ ] Set up PWA capabilities

#### **Week 15-17: Customer-Facing Features**
- [ ] Develop product catalog pages with search
- [ ] Build shopping cart and checkout flow
- [ ] Implement product detail pages with variants
- [ ] Add user account management
- [ ] Create order history and tracking
- [ ] Develop mobile-optimized checkout
- [ ] Integrate payment gateway UIs

#### **Week 18-20: Admin Dashboard**
- [ ] Build inventory management interface
- [ ] Develop order management dashboard
- [ ] Create accounting and reporting UI
- [ ] Implement user management system
- [ ] Add analytics and reporting dashboards
- [ ] Create supplier and purchase order interface
- [ ] Add system configuration panels

### **Phase 4: Integration and Testing (Weeks 21-26)**

#### **Week 21-22: Third-Party Integrations**
- [ ] Complete payment gateway integrations (Stripe, PayNow)
- [ ] Finalize logistics provider APIs (Ninja Van, SingPost)
- [ ] Implement accounting software sync (Xero, QuickBooks)
- [ ] Add CRM integration with HubSpot
- [ ] Configure email marketing with Mailchimp
- [ ] Set up Google Analytics 4 and conversion tracking

#### **Week 23-24: Security and Compliance**
- [ ] Implement PDPA compliance features
- [ ] Add GDPR consent management
- [ ] Configure security headers and protections
- [ ] Set up multi-factor authentication
- [ ] Implement role-based access control
- [ ] Complete security audit and penetration testing
- [ ] Configure automated security scanning

#### **Week 25-26: Performance Optimization and Testing**
- [ ] Conduct comprehensive unit testing
- [ ] Complete integration testing across all services
- [ ] Perform load testing with Apache JMeter
- [ ] Optimize database queries and add indexes
- [ ] Fine-tune Redis caching strategy
- [ ] Optimize Core Web Vitals scores
- [ ] Complete user acceptance testing with stakeholders

### **Phase 5: Deployment and Launch (Weeks 27-32)**

#### **Week 27-28: Pre-Launch Preparation**
- [ ] Migrate existing data to new system
- [ ] Configure production DNS and SSL certificates
- [ ] Set up monitoring and alerting rules
- [ ] Create disaster recovery runbooks
- [ ] Train staff on new system functionality
- [ ] Prepare customer communication and support materials
- [ ] Execute disaster recovery testing

#### **Week 29-30: Production Launch**
- [ ] Deploy production environment with blue-green strategy
- [ ] Execute comprehensive smoke tests
- [ ] Monitor system performance and error rates
- [ ] Conduct final security and compliance checks
- [ ] Launch marketing and customer communication
- [ ] Provide 24/7 support during launch period
- [ ] Monitor all system metrics and user feedback

#### **Week 31-32: Post-Launch Stabilization**
- [ ] Address initial user feedback and issues
- [ ] Optimize performance based on real traffic
- [ ] Tune auto-scaling policies based on load
- [ ] Complete documentation and knowledge transfer
- [ ] Establish regular maintenance schedule
- [ ] Begin Phase 2 feature planning
- [ ] Conduct post-launch review and lessons learned

---

## **10. TECHNICAL SPECIFICATIONS**

### **10.1 System Requirements**

#### **10.1.1 Hardware Specifications**
```yaml
Production Environment:
  Application Servers:
    - Instance Type: t3.xlarge (4 vCPU, 16GB RAM)
    - Quantity: 5 nodes initially
    - Auto-scaling: 3-50 nodes
    - Storage: 100GB SSD per node
    
  Database Servers:
    - Primary: db.r5.2xlarge (8 vCPU, 64GB RAM)
    - Read Replicas: db.r5.xlarge (4 vCPU, 32GB RAM) × 3
    - Storage: 1TB SSD with Provisioned IOPS (3000)
    - Backup Storage: Automated with 30-day retention
    
  Cache/Search Layer:
    - Redis: cache.r6g.large × 6 nodes (cluster mode)
    - Elasticsearch: m5.large.search × 6 nodes
```

#### **10.1.2 Software Requirements**
```yaml
Runtime Environment:
  Node.js Version: 20.18.0 LTS
  NPM Version: 10.8.2
  PostgreSQL: 15.5
  Redis: 7.2.3
  Elasticsearch: 8.12.0
  
Container Platform:
  Kubernetes: 1.28
  Docker: 24.0.7
  Helm: 3.14.0
  
Frontend Stack:
  Next.js: 14.2.0
  React: 18.3.0
  TypeScript: 5.4.5
  Tailwind CSS: 3.4.7
```

### **10.2 Non-Functional Requirements**

#### **10.2.1 Performance Requirements**
- **Response Time**: < 200ms for 95% of API calls
- **Page Load Time**: < 3 seconds for all pages
- **Database Query Time**: < 50ms for 95% of queries
- **Concurrent Users**: Support 10,000 active users
- **Transactions Per Second**: Handle 500 orders per second

#### **10.2.2 Scalability Requirements**
- **Auto-scaling**: Scale horizontally within 2-3 minutes
- **Database Scalability**: Handle 10TB+ data volume
- **Peak Traffic**: Accommodate 10x normal traffic during sales
- **Geographic Distribution**: Multi-region deployment ready

#### **10.2.3 Maintainability Requirements**
- **Code Coverage**: > 80% unit test coverage
- **Documentation**: Comprehensive API documentation (OpenAPI 3.0)
- **Deployment Frequency**: Weekly deployments
- **Recovery Time**: < 15 minutes for service restoration

---

## **11. RISK MITIGATION STRATEGY**

### **11.1 Technical Risks**

| **Risk** | **Impact** | **Probability** | **Mitigation Strategy** |
|----------|------------|-----------------|------------------------|
| Database Performance Issues | High | Medium | Read replicas, query optimization, caching |
| Payment Gateway Downtime | High | Low | Multiple provider integration, retry logic |
| Microservices Communication Failure | Medium | Medium | Circuit breakers, retry mechanisms, queue-based processing |
| Kubernetes Infrastructure Issues | High | Low | Multi-AZ deployment, service mesh, container health checks |
| Cache Layer Failures | Medium | Low | Cache-aside pattern, fallback to database |

### **11.2 Security Risks**

| **Risk** | **Impact** | **Probability** | **Mitigation Strategy** |
|----------|------------|-----------------|------------------------|
| SQL Injection Attacks | High | Medium | Parameterized queries, ORM usage, input validation |
| DDoS Attacks | High | Medium | Rate limiting, WAF, CDN protection, auto-scaling |
| Data Breaches | Very High | Low | Encryption at rest/transit, PAM, MFA, audit logging |
| Malicious API Usage | Medium | Medium | API keys, rate limiting, request validation, anomaly detection |
| Insider Threats | High | Low | RBAC, audit trails, least privilege principle |

### **11.3 Compliance Risks**

| **Risk** | **Impact** | **Probability** | **Mitigation Strategy** |
|----------|------------|-----------------|------------------------|
| PDPA Compliance | Very High | Medium | Legal review, consent management, data residency controls |
| GST Calculation Errors | High | Low | Automated testing, reconciliation reports, quarterly audits |
| IRAS Reporting Issues | High | Low | Pre-submission validation, professional accounting review |
| Audit Trail Gaps | Medium | Low | Immutable event sourcing, comprehensive logging, retention policies |

---

## **12. SUCCESS METRICS AND KPIs**

### **12.1 Technical Performance Metrics**

| **Metric** | **Target** | **Measurement Method** |
|------------|------------|------------------------|
| API Response Time (p95) | < 200ms | Prometheus metrics |
| Page Load Time | < 3 seconds | Google Lighthouse |
| System Uptime | 99.9% (8.76 hours downtime/year) | New Relic APM |
| Database Performance | Query time < 50ms (p95) | PostgreSQL pg_stat_statements |
| Cache Hit Rate | > 90% | Redis monitoring |
| Error Rate | < 0.1% of requests | Application logs |
| Concurrent Users | 10,000 active users | Grafana dashboards |

### **12.2 Business Metrics**

| **Metric** | **Target** | **Measurement Method** |
|------------|------------|------------------------|
| Order Processing Time | < 5 minutes | Order creation to processing time |
| Inventory Accuracy | > 99% | Physical vs. system inventory comparison |
| GST Calculation Accuracy | > 99.5% | Automated reconciliation with accounting systems |
| Customer Satisfaction | > 4.5/5 stars | Product reviews and surveys |
| Conversion Rate | > 3.5% | Google Analytics 4 |
| Cart Abandonment Rate | < 60% | E-commerce analytics |
| Mobile User Experience | > 85 Lighthouse score | Regular performance audits |

---

## **13. DOCUMENT APPROVALS**

| **Role** | **Name** | **Signature** | **Date** |
|----------|----------|---------------|----------|
| Chief Technology Officer | [To be filled] | | |
| Head of Engineering | [To be filled] | | |
| Solutions Architect | [To be filled] | | |
| Security Lead | [To be filled] | | |
| DevOps Lead | [To be filled] | | |

---

**Document Status**: *READY FOR ARCHITECTURE REVIEW*  
**Review Meeting Date**: TBD  
**Implementation Start Date**: TBD  
**Target Go-Live Date**: Week 32 from project start date

---

This comprehensive architecture document provides the technical blueprint for building a world-class e-commerce platform for Singapore SMBs with integrated inventory and accounting systems. The architecture follows industry best practices and Singapore-specific compliance requirements while maintaining scalability, security, and performance standards.
