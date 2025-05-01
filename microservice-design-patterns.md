# microservice-design-patterns

# Microservices Design Patterns with Examples

Microservices architecture breaks applications into small, independent services that communicate over well-defined APIs. Here are key design patterns with examples:

## 1. Decomposition Patterns

### a. Business Capability Pattern
**Example**: E-commerce app decomposed into:
- Product Service (manages catalog)
- Order Service (handles orders)
- Payment Service (processes payments)
- Shipping Service (manages deliveries)

### b. Strangler Pattern
**Example**: Migrating a monolithic banking app:
1. Start by extracting "Account Balance" as a separate microservice
2. Redirect calls from monolith to new service
3. Gradually extract other features (transfers, statements)
4. Eventually retire the monolith

## 2. Integration Patterns

### a. API Gateway Pattern
**Example**: An e-commerce API gateway:
- Routes `/products` to Product Service
- Routes `/orders` to Order Service
- Aggregates data from multiple services for `/dashboard`
- Handles authentication for all requests

```javascript
// Example gateway routing configuration
app.get('/products/:id', apiGateway.auth, (req, res) => {
  productServiceClient.get(req.params.id).then(response => res.json(response));
});

app.post('/orders', apiGateway.auth, (req, res) => {
  orderServiceClient.create(req.body).then(response => res.json(response));
});
```

### b. Aggregator Pattern
**Example**: Dashboard service that combines:
- User profile (from User Service)
- Recent orders (from Order Service)
- Recommendations (from Recommendation Service)

```java
public DashboardData getDashboardData(String userId) {
    User user = userService.getUser(userId);
    List<Order> orders = orderService.getRecentOrders(userId);
    List<Product> recommendations = recommendationService.getForUser(userId);
    
    return new DashboardData(user, orders, recommendations);
}
```

## 3. Database Patterns

### a. Database per Service
**Example**:
- User Service uses PostgreSQL for user data
- Product Service uses MongoDB for product catalog
- Analytics Service uses Cassandra for time-series data

### b. Saga Pattern
**Example**: Order processing saga:
1. Order Service creates order (PENDING)
2. Payment Service processes payment
3. If payment succeeds, Inventory Service reserves items
4. If any step fails, compensating transactions undo previous steps

```python
def create_order(order_data):
    try:
        order = order_service.create(order_data)
        payment = payment_service.process(order)
        
        if payment.success:
            inventory_service.reserve(order.items)
            order_service.confirm(order.id)
            return "Order created"
        else:
            order_service.cancel(order.id)
            return "Payment failed"
            
    except Exception as e:
        order_service.cancel(order.id)
        payment_service.refund(payment.id)
        inventory_service.release(order.items)
        raise e
```

## 4. Observability Patterns

### a. Log Aggregation
**Example**: Centralized logging with ELK stack:
- Each service logs to stdout
- Filebeat collects logs
- Logstash processes logs
- Elasticsearch stores logs
- Kibana provides visualization

### b. Distributed Tracing
**Example**: Order request flow tracked across services:
1. User → API Gateway (trace-id: abc123)
2. API Gateway → Order Service (trace-id: abc123)
3. Order Service → Payment Service (trace-id: abc123)
4. Payment Service → Database (trace-id: abc123)

## 5. Cross-Cutting Concerns

### a. Circuit Breaker Pattern
**Example**: Product Service calls Inventory Service:

```java
@CircuitBreaker(fallbackMethod = "getDefaultInventory")
public Inventory getInventory(String productId) {
    return inventoryClient.get(productId);
}

public Inventory getDefaultInventory(String productId) {
    return new Inventory(productId, 0); // fallback when inventory service is down
}
```

### b. Service Mesh
**Example**: Istio service mesh handling:
- Service discovery (Product Service finds Inventory Service)
- Load balancing (distributes calls to Inventory Service instances)
- Retries (automatically retries failed calls)
- Security (mTLS between services)

## 6. Deployment Patterns

### a. Sidecar Pattern
**Example**: Service with logging sidecar:
- Main container runs business logic
- Sidecar container collects logs and ships to central system

```yaml
# Kubernetes deployment example
containers:
- name: order-service
  image: orders:latest
- name: log-sidecar
  image: fluentd:latest
  volumeMounts:
  - name: logs
    mountPath: /var/log
```

### b. Blue-Green Deployment
**Example**: Deploying User Service v2:
1. Deploy v2 alongside v1 (green)
2. Test v2 with internal traffic
3. Switch load balancer to route all traffic to v2
4. Monitor and decommission v1 if successful

These patterns help address common challenges in microservices architecture like service decomposition, distributed data management, inter-service communication, and resilience.

