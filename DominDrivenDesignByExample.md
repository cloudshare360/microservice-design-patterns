# Breaking Down Domains into Subdomains and Bounded Contexts in DDD

Let me explain this process with a comprehensive example of an **E-Commerce Platform** to show how to identify subdomains, bounded contexts, and their boundaries.

## Step 1: Identify Core Domains and Subdomains

### Core Domain (Business Differentiator)
**Order Fulfillment**: The unique capability that makes our e-commerce platform stand out (e.g., 2-hour delivery guarantee)

### Supporting Subdomains (Essential but Not Unique)
1. **Product Catalog**
2. **Inventory Management**
3. **Payment Processing**
4. **Customer Management**

### Generic Subdomains (Common Solutions)
1. **User Authentication**
2. **Notification System**
3. **Reporting**

## Step 2: Define Bounded Contexts

Each subdomain becomes a bounded context with explicit boundaries:

### 1. Order Processing Context
**Responsibility**: Manage the entire order lifecycle
**Key Operations**:
- Create order
- Process payment
- Coordinate fulfillment
- Handle returns

**Boundary Rules**:
- Owns order data model (different from inventory's view of products)
- Communicates with payment context via anti-corruption layer

### 2. Inventory Context
**Responsibility**: Track product availability
**Key Operations**:
- Check stock levels
- Reserve items for orders
- Receive new inventory
- Update product locations

**Boundary Rules**:
- Uses its own product SKU system (different from catalog's product IDs)
- Publishes "InventoryUpdated" events

### 3. Catalog Context
**Responsibility**: Product information management
**Key Operations**:
- Manage product hierarchy
- Handle pricing rules
- Apply discounts
- Search products

**Boundary Rules**:
- Maintains rich product descriptions (unlike inventory's minimal product data)
- Shares read-only product info via published language

## Step 3: Context Mapping (How They Relate)

```
[Catalog Context] --(Published Language)--> [Order Context]
       ↑
       | (RPC)
       ↓
[Inventory Context] --(Event-Driven)--> [Fulfillment Context]
```

## Detailed Example: Order Processing Breakdown

### Order Context Operations:
```java
public class OrderService {
    public Order createOrder(CustomerId customerId, Map<ProductId, Integer> items) {
        // 1. Verify products exist (catalog context)
        catalogService.validateProductsExist(items.keySet());
        
        // 2. Check inventory (inventory context)
        inventoryService.reserveItems(items);
        
        // 3. Calculate pricing (catalog context)
        Money total = catalogService.calculateTotal(items);
        
        // 4. Create order (this context)
        Order order = new Order(orderId, customerId, items, total);
        orderRepository.save(order);
        
        // 5. Publish event
        eventBus.publish(new OrderCreatedEvent(order));
        
        return order;
    }
}
```

### Boundary Enforcement:

1. **Database Separation**:
   - Order context has its own `orders` table
   - References products by ID only (no JOINs to catalog tables)

2. **Communication Protocols**:
   ```mermaid
   sequenceDiagram
       Order->>+Catalog: Get product details (RPC)
       Catalog-->>-Order: ProductDTO
       Order->>+Inventory: Reserve items (RPC)
       Inventory-->>-Order: Reservation confirmation
       Order->>EventBus: OrderCreatedEvent
       Fulfillment->>EventBus: Subscribe to OrderCreatedEvent
   ```

3. **Model Differences**:
   - **Catalog's Product**:
     ```java
     public class Product {
         private ProductId id;
         private String name;
         private String description;
         private List<Category> categories;
         private Price price;
     }
     ```
     
   - **Inventory's Product**:
     ```java
     public class InventoryItem {
         private Sku sku;  // Different identifier!
         private WarehouseLocation location;
         private int quantityOnHand;
         private int reservedQuantity;
     }
     ```

## Key Boundary Decisions

1. **Order ↔ Catalog Boundary**:
   - Order context only needs basic product info (ID, price)
   - Sync via periodic data exports rather than real-time queries

2. **Order ↔ Inventory Boundary**:
   - Strong consistency required for inventory reservations
   - Uses saga pattern for reservation management

3. **Shared Kernel** (for common concepts):
   ```java
   // Shared kernel module
   public class CustomerId {
       private UUID id;
       // Common validation rules
   }
   ```

## Identifying Operations Through Event Storming

1. **Domain Events**:
   - `OrderPlaced`
   - `PaymentProcessed`
   - `InventoryReserved`
   - `ShippingLabelGenerated`

2. **Commands** that trigger events:
   - `PlaceOrder`
   - `ProcessPayment`
   - `AllocateInventory`

3. **Aggregates** that handle commands:
   - `Order` (root aggregate)
   - `InventoryItem`
   - `Payment`

## Practical Boundary Implementation

### Order Context API:
```typescript
// Order service exposes these operations
POST /orders          // Create order
GET  /orders/{id}     // Get order status
POST /orders/{id}/cancel  // Cancel order

// But intentionally doesn't expose:
// - Product management (catalog's job)
// - Inventory adjustments (inventory's job)
```

### Context Integration:
```java
// Instead of direct DB access between contexts
public class CatalogAdapter {
    public ProductInfo getProductInfo(ProductId id) {
        // Makes API call to catalog context
        // Translates catalog's model to order context's model
        // Implements caching
    }
}
```

This breakdown ensures each context:
1. Has a single responsibility
2. Maintains its own model integrity
3. Communicates through well-defined boundaries
4. Evolves independently while working together to fulfill business capabilities