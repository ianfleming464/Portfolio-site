# Real-Time Cart Synchronization Engine
## Solving E-Commerce Pricing Discrepancies at Scale

---

## Overview

**Role:** Lead Developer & Architect  
**Timeline:** 12+ months (development + production)  
**Deployment:** 9 International E-Commerce Storefronts  
**Tech Stack:** JavaScript (ES6+) · Shopify APIs · Liquid · DOM Manipulation · Event-Driven Architecture

**Impact:**
- ✅ Eliminated "price changed at checkout" support tickets
- ✅ Enabled complex multi-discount promotional campaigns
- ✅ Zero-downtime international deployment across 9 markets
- ✅ 12+ months production-stable handling thousands of daily customer interactions

---

## The Challenge

### The Problem

E-commerce platforms often create customer trust issues when discount systems don't communicate properly. At Bears with Benefits, customers frequently saw different prices between their cart and checkout:

```
Customer sees:     Cart shows €100 - 10% = €90
Customer pays:     Checkout charges €76.50 (15% auto + 10% manual)
Customer thinks:   "Why did the price change? Is this a scam?"
```

### Business Impact

**Before the solution:**
- Regular support tickets about "price changed at checkout" (weekly occurrence)
- Limited promotional capabilities (couldn't run complex campaigns without confusing customers)
- Cart abandonment when prices didn't match expectations
- Manual intervention required to honor displayed cart prices

### Technical Root Cause

The platform uses three independent discount systems that don't coordinate:

1. **Shopify:** Applies automatic discounts at checkout (invisible to cart)
2. **Rebuy (third-party cart widget):** Handles manual discount codes at cart layer
3. **URL parameters:** Enable promotional campaigns with pre-applied codes

When multiple discounts activate simultaneously, each system operates independently, creating pricing inconsistencies and customer confusion.

### Additional Complexity

- **Bundle promotions** (3-for-2, Buy X Get Y) conflict with percentage discounts
- **Free gift triggers** based on cart thresholds
- **Multiple simultaneous discounts** (automatic + manual + URL) with unclear precedence
- **International markets** (9 stores with different currencies, tax rules, formatting conventions)

---

## The Solution

### High-Level Approach

Rather than fighting platform limitations, I architected a **real-time synchronization layer** that bridges the gap between independent discount systems, ensuring customers always see accurate pricing before proceeding to checkout.

### Core Architecture: Detect → Allocate → Render

```
User Interaction (quantity change, code entry)
    ↓
[Event Hub] Single cart fetch, debounced
    ↓
[Signal Detection] Aggregate discount info from multiple sources
    ↓
[Discount Resolution] Normalize, validate, enforce precedence
    ↓
[Mode Determination] Select pricing strategy
    ↓
[Proportional Allocation] Distribute discounts mathematically
    ↓
[Locale Formatting] Currency symbols, decimals per market
    ↓
[Renderer] Update cart UI idempotently
```

### Key Technical Decisions

#### 1. Multi-Source Signal Aggregation

Since no single API reliably provides discount state, the system synthesizes information from:
- Shopify's `discount_code` cookie (URL discounts)
- Rebuy's cart API (`cart_level_discount_applications`)
- DOM visual indicators (discount badges, messages)
- Custom cookies (free gift tracking)

**Why this matters:** Redundant data sources compensate for platform API unreliability.

#### 2. Proportional Allocation Algorithm

Percentage and fixed-amount discounts must be distributed across line items to show accurate per-item pricing. The challenge: floating-point arithmetic creates 1-cent rounding discrepancies.

**Solution:**
```javascript
// Simplified concept
function allocateDiscount(items, totalDiscount) {
  const subtotal = items.reduce((sum, item) => sum + item.price, 0);
  let remaining = totalDiscount;
  
  items.forEach((item, index) => {
    const proportionalShare = (item.price / subtotal) * totalDiscount;
    
    // Last item absorbs rounding remainder
    item.discount = (index === items.length - 1) 
      ? remaining 
      : Math.round(proportionalShare);
    
    remaining -= item.discount;
  });
  
  // Invariant maintained: sum(discounts) === totalDiscount
  return items;
}
```

#### 3. Mode-Based Rendering Strategies

Different discount combinations require different display approaches:

| Mode | Strategy | Example |
|------|----------|---------|
| **Bundle Exclusive** | Hide manual input, show bundle message | "Buy 3, Get 1 Free" |
| **Manual Preview** | Use Rebuy's custom pricing | Customer-entered code |
| **Checkout Only** | Mirror Shopify's automatic discount | URL/auto discounts |
| **Combo Mode** | Coordinate 2 discounts, show combined savings | Auto + Manual |

#### 4. International Locale Support

Each market has unique pricing conventions:

| Market | Format | Example |
|--------|--------|---------|
| Spain (ES) | Symbol before, comma decimal | €12,34 |
| Germany (DE) | Symbol after, comma decimal | 12,34€ |
| France (FR) | Symbol after, space thousands | 12 345,67€ |
| Switzerland (CH) | Symbol before, period decimal | CHF12.34 |

---

## Implementation Journey

### Incremental Refactoring (M0-M7)

Rather than a risky "big bang" rewrite, I implemented a phased migration strategy:

**M0: Scope Definition**
- Locked priority hierarchy for discount conflicts
- Defined scenario matrix for all discount combinations
- Established locale formatting rules

**M1: Canonical State Snapshot**
- Single cart fetch per event (eliminated race conditions)
- Centralized signal detection (cookie reads, DOM parsing)
- Built `CartState` model with normalized data

**M2: Pricing Core**
- Extracted allocation algorithm into reusable module
- Added comprehensive unit tests (rounding, formatting)
- Centralized locale-aware price formatting

**M3: Renderer Layer**
- Single idempotent renderer (eliminated DOM collisions)
- Preserved original HTML for reversion capability
- Conditional subtotal override (only when necessary)

**M4: Coordinator**
- Explicit mode resolution with deterministic precedence
- Enforced "maximum 2 discounts" business rule
- Ordered execution pipeline

**M5: Performance Optimization**
- Cached cart fetch promise with TTL
- Batched DOM writes via `requestAnimationFrame`
- Reduced unnecessary API calls

**M6-M7: Hardening & Testing**
- Removed duplicate detection logic
- Aligned formatting across all handlers
- Manual QA across all discount scenarios
- Production monitoring and validation

---

## Technical Challenges & Solutions

### Challenge 1: Unreliable Platform APIs

**Problem:** Shopify's discount APIs return inconsistent data depending on discount type, application method, and timing. Some discount information simply doesn't appear in expected API responses.

**Solution:**
- Multi-source validation with cross-referencing
- Fallback detection strategies (cookies → cart API → DOM)
- Defensive programming with null checks and validation

### Challenge 2: Asynchronous Update Cycles

**Problem:** Rebuy and Shopify have different API response times, creating scenarios where cart shows stale data or updates arrive out of order.

**Solution:**
- Debounced event hub (waits for state quiescence before processing)
- Snapshot-based rendering (compute full state, then single DOM update)
- State versioning (ignore updates from older snapshots)

### Challenge 3: Complex Discount Precedence

**Problem:** Customers can trigger multiple discounts simultaneously (URL + manual + automatic + bundle). Which takes priority? How to display clearly?

**Solution:**
- Explicit hierarchy: Bundle → Multiples → Gift → Manual → Automatic
- Maximum 2 discounts enforced by coordinator
- Mode-specific rendering for each valid combination
- Clear visual indicators for active discounts

### Challenge 4: International Deployment

**Problem:** 9 storefronts with different currencies, decimal formats, tax rules, and cultural expectations around price display.

**Solution:**
- Configuration-driven locale system
- Market-specific formatting rules (symbol position, separators)
- Extensible architecture for new markets
- Sequential rollout strategy (validate in Spain, then expand)

---

## Results & Impact

### Business Outcomes

**Customer Experience:**
- ✅ **Eliminated pricing discrepancy complaints** (previously recurring weekly issue)
- ✅ **Transparent pricing** throughout shopping journey
- ✅ **Complex campaigns enabled** without customer confusion

**Operational Efficiency:**
- ✅ **No manual price reconciliation** (previously handled by support team)
- ✅ **Promotional flexibility** (2-3 simultaneous discounts supported)
- ✅ **Self-service progressive discount app** (reduced setup time significantly)

### Technical Achievements

**Performance:**
- ✅ **<100ms cart update latency** (real-time user experience)
- ✅ **Single cart fetch per event** (optimized API usage)
- ✅ **Batched DOM rendering** (no visual flicker)

**Reliability:**
- ✅ **12+ months production-stable** (zero critical incidents)
- ✅ **Zero-downtime deployment** across all markets
- ✅ **Handles peak seasonal traffic** (thousands of concurrent shoppers)

**Scalability:**
- ✅ **9 international markets** with locale-specific formatting
- ✅ **Modular architecture** (independent feature updates)
- ✅ **Edge case coverage** (gift cards, subscriptions, split payments, international VAT)

---

## System Evolution

### Architecture Progression

**Phase 1: Event-Driven Monolith (Months 1-6)**
- Multiple handlers listening to cart events
- Duplicated detection logic
- Overlapping DOM updates
- Functional but fragile

**Phase 2: Modular Coordinator (Months 7-12)**
- Single event hub with cart state snapshot
- Normalized discount representation
- Mode-based rendering strategies
- Deterministic execution order

**Phase 3: Optimized Production System (Current)**
- Performance optimizations (caching, batching)
- Comprehensive edge case handling
- International locale support
- Integration with progressive discount app

### Lessons Learned

**What Worked Well:**
- Incremental refactoring maintained production stability throughout
- Multi-source signal detection compensated for unreliable APIs
- Unit tests caught rounding edge cases early
- Mode-based approach provided flexibility for new discount types

**What I'd Do Differently:**
- **Earlier automated testing:** Manual QA caught issues but was time-intensive
- **State machine formalization:** Implicit mode detection evolved into tech debt
- **Performance monitoring upfront:** Added reactively rather than proactively
- **Documentation from day one:** Rebuilt context multiple times

---

## Related Systems

### Progressive Discount Application

Mid-project, the business required **tiered discounting** (single code that increases discount as cart value rises). I developed a complementary Shopify app that integrates with the cart synchronization engine:

**Features:**
- Single trigger code (e.g., "GETMORE") with multiple cart-value thresholds
- Merchant self-service UI for tier configuration (no developer needed)
- Real-time tier visualization in checkout
- Automatic discount code creation via GraphQL mutations

**Tech Stack:** Remix · Shopify Functions · GraphQL · Neon Postgres · Vercel

**Integration:** Cart sync engine detects progressive discounts via cart attributes and coordinates with other active promotions, maintaining pricing accuracy when quantity changes cross tier thresholds.

---

## Technology Deep Dive

**Core Technologies:**
- **JavaScript (ES6+)** - Event handling, mathematical algorithms, DOM manipulation
- **Shopify Storefront API** - Cart state management
- **Shopify Admin API** - Discount validation
- **Rebuy API** - Third-party cart widget coordination
- **Liquid** - Theme templating and data attribute injection
- **Custom Event System** - Cross-component communication

**Architecture Patterns:**
- Event-driven coordination with debouncing
- Snapshot-based state management
- Idempotent rendering with state preservation
- Proportional allocation with rounding correction
- Locale-aware formatting strategies

**Testing Strategy:**
- Unit tests (allocation algorithms, formatting rules)
- Manual QA checklists (all discount scenarios)
- Playwright E2E (optional, cart UI flows)
- Production monitoring and error tracking

---

## Project Context

This system is production-critical for **Bears with Benefits**, a European beauty/skincare startup. The cart synchronization engine processes thousands of customer interactions daily, enabling revenue-driving promotional campaigns while maintaining pricing transparency and customer trust.

**Why This Matters:** E-commerce success depends on customer trust. When customers see unexpected price changes, they abandon carts. This system ensures what customers see is what they pay, enabling complex marketing strategies without sacrificing user experience.

---

## Portfolio Notes

**Confidentiality:** The production codebase is proprietary to Bears with Benefits. This case study describes architecture, challenges, and solutions without revealing business-sensitive implementation details.

**Documentation:** Comprehensive [technical documentation](https://github.com/ianfleming464/cart-sync-engine-docs) available including architecture diagrams, API patterns, and example implementations.

---

## Skills Demonstrated

**System Architecture:**
- Multi-platform integration design
- State synchronization across independent systems
- Event-driven coordination with conflict resolution
- Performance optimization (caching, batching, debouncing)

**Technical Problem Solving:**
- Worked around platform limitations rather than fighting them
- Mathematical algorithm development (proportional allocation, rounding)
- Edge case identification and handling
- Incremental refactoring strategies

**Production Engineering:**
- Zero-downtime international deployment
- Long-term production stability (12+ months)
- Performance optimization at scale
- Comprehensive error handling

**Business Impact:**
- Eliminated customer support burden
- Enabled revenue-driving promotional campaigns
- Improved operational efficiency
- Maintained system through business growth

---

**Author:** Ian Fleming  
**Role:** Full-Stack JavaScript Developer  
**Specialization:** E-Commerce Integration Architecture, Production Systems

[Portfolio](https://www.ianflemingdeveloper.com) | [LinkedIn](https://linkedin.com/in/i-fleming) | [GitHub](https://github.com/ianfleming464) | [Technical Docs](https://github.com/ianfleming464/cart-sync-engine-docs)

---

*Last Updated: January 2026*
