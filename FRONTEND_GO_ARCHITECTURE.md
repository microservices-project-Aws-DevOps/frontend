# Frontend Service Architecture - Go + HTML/CSS/JavaScript Relationship

## Overview: Why Go for Frontend?

**This is NOT a Node.js/React/Vue SPA (Single Page Application)!**

Instead, this is a **Server-Side Rendered (SSR) web application** where:
- **Go** = Backend server (business logic, service orchestration, rendering)
- **HTML/CSS/JS** = Frontend templates (UI presentation, minimal interactivity)

---

## Architecture Pattern: Server-Side Rendering (SSR)

### Traditional SPA Approach (NOT used here):
```
Browser (React/Vue) ─► API Server
     │
     ├─ JavaScript runs in browser
     ├─ Fetches JSON from API
     ├─ Renders UI dynamically
     └─ Heavy JS bundle (~200-500KB)
```

### Frontend Service Approach (USED here):
```
Browser ◄─────────── Go Server (Frontend Service)
                         │
                         ├─ Go HTTP Server
                         ├─ HTML Template Rendering
                         ├─ Service Orchestration
                         │  ├─ Call Product Catalog (gRPC)
                         │  ├─ Call Cart Service (gRPC)
                         │  ├─ Call Payment Service (gRPC)
                         │  └─ Call Other Services (gRPC)
                         │
                         ├─ Serve CSS files
                         ├─ Serve JS files
                         └─ Serve Static Images
```

---

## Component Breakdown

### 1. Go Backend (Server-Side)

**File: `main.go`**

```go
// 1. Start HTTP server on port 8080
func main() {
    svc := new(frontendServer)
    
    // Initialize gRPC connections to all backend services
    mustConnGRPC(ctx, &svc.currencySvcConn, svc.currencySvcAddr)
    mustConnGRPC(ctx, &svc.productCatalogSvcConn, svc.productCatalogSvcAddr)
    mustConnGRPC(ctx, &svc.cartSvcConn, svc.cartSvcAddr)
    // ... more services
    
    // Setup HTTP routes
    r := mux.NewRouter()
    r.HandleFunc("/", svc.homeHandler).Methods(http.MethodGet)
    r.HandleFunc("/product/{id}", svc.productHandler).Methods(http.MethodGet)
    r.HandleFunc("/cart", svc.viewCartHandler).Methods(http.MethodGet)
    r.HandleFunc("/cart", svc.addToCartHandler).Methods(http.MethodPost)
    r.HandleFunc("/login", svc.loginHandler).Methods(http.MethodGet, http.MethodPost)
    r.PathPrefix("/static/").Handler(http.StripPrefix("/static/", http.FileServer(http.Dir("./static/"))))
    
    // Start server
    http.ListenAndServe(":8080", handler)
}
```

**Key Responsibilities:**
- Initialize gRPC connections to 8+ backend services
- Route HTTP requests to handlers
- Serve static files (CSS, JS, images)
- Setup middleware (logging, auth, tracing)

---

### 2. Go Handlers (Request Processing)

**File: `handlers.go`**

#### Example: `homeHandler` - Renders Home Page

```go
func (fe *frontendServer) homeHandler(w http.ResponseWriter, r *http.Request) {
    // Step 1: Call backend services via gRPC
    currencies, err := fe.getCurrencies(r.Context())      // Call Currency Service
    products, err := fe.getProducts(r.Context())          // Call Product Catalog Service
    cart, err := fe.getCart(r.Context(), sessionID(r))    // Call Cart Service
    
    // Step 2: Process and format data for template
    type productView struct {
        Item  *pb.Product
        Price *pb.Money
    }
    ps := make([]productView, len(products))
    for i, p := range products {
        price, err := fe.convertCurrency(r.Context(), p.GetPriceUsd(), currentCurrency(r))
        ps[i] = productView{p, price}
    }
    
    // Step 3: Render HTML template with data
    templates.ExecuteTemplate(w, "home", map[string]interface{}{
        "products":      ps,
        "currencies":    currencies,
        "cart_size":     len(cart),
        "banner_color":  os.Getenv("BANNER_COLOR"),
    })
}
```

#### Example: `addToCartHandler` - POST Request

```go
func (fe *frontendServer) addToCartHandler(w http.ResponseWriter, r *http.Request) {
    // Step 1: Parse form data from HTML form
    productID := r.FormValue("product_id")
    quantity := r.FormValue("quantity")
    
    // Step 2: Call Cart Service via gRPC
    err := fe.insertCart(r.Context(), sessionID(r), productID, qty)
    
    // Step 3: Redirect back to cart view
    http.Redirect(w, r, "/cart", http.StatusSeeOther)
}
```

**What handlers do:**
1. ✅ Extract user input (form data, URL params)
2. ✅ Call backend services (gRPC)
3. ✅ Transform/process response data
4. ✅ Render HTML templates with processed data
5. ✅ Send HTML response to browser

---

### 3. HTML Templates (Go's `html/template`)

**File: `templates/home.html`**

```html
{{ define "home" }}
{{ template "header" . }}

<section class="fs-hero">
  <div class="container">
    <h1>Your Style, Your Statement</h1>
    <a href="#catalog" class="fs-btn-hero">Shop Now →</a>
    <a href="{{ $.baseUrl }}/cart" class="fs-btn-hero-outline">View Cart</a>
  </div>
</section>

<!-- Products section - rendered by Go template engine -->
<section id="catalog">
  <div class="container">
    <h2>Featured Products</h2>
    
    <!-- Loop through products array passed from Go handler -->
    {{range .products}}
    <div class="product-card">
      <img src="/static/images/{{.Item.Id}}.jpg" alt="{{.Item.Name}}">
      <h3>{{.Item.Name}}</h3>
      <p class="price">
        {{renderMoney .Price}}  <!-- Custom Go function to format price -->
      </p>
      
      <!-- Form to add to cart - POST to Go handler -->
      <form method="POST" action="/cart">
        <input type="hidden" name="product_id" value="{{.Item.Id}}">
        <input type="hidden" name="session_id" value="{{$.sessionID}}">
        <select name="quantity">
          <option value="1">Qty 1</option>
          <option value="2">Qty 2</option>
          <option value="5">Qty 5</option>
        </select>
        <button type="submit">Add to Cart</button>
      </form>
    </div>
    {{end}}
  </div>
</section>

{{ template "footer" . }}
{{ end }}
```

**What Go templates do:**
1. ✅ Loop through data (`.products`)
2. ✅ Conditionally render content (`{{if}}{{end}}`)
3. ✅ Call custom Go functions (`renderMoney`, `renderCurrencyLogo`)
4. ✅ Inject dynamic data into HTML (`{{.Item.Name}}`)
5. ✅ Generate HTML forms that POST back to Go handlers

**Key Functions:**
```go
templates = template.Must(template.New("").
    Funcs(template.FuncMap{
        "renderMoney": renderMoney,              // Format price: ₹1,299.99
        "renderCurrencyLogo": renderCurrencyLogo, // Show $ € ₹ symbols
    }).
    ParseGlob("templates/*.html"))
```

---

### 4. CSS Files (Static Assets)

**File: `static/styles/styles.css`**

```css
/* Main stylesheet - served as static file */

.fs-hero {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 80px 20px;
  text-align: center;
  color: white;
}

.fs-btn-hero {
  display: inline-block;
  padding: 12px 32px;
  background: #f5a623;
  color: black;
  border-radius: 8px;
  text-decoration: none;
  font-weight: 600;
  transition: all 0.3s ease;
}

.fs-btn-hero:hover {
  transform: translateY(-2px);
  box-shadow: 0 10px 20px rgba(0,0,0,0.2);
}

.product-card {
  border: 1px solid #eee;
  border-radius: 8px;
  padding: 16px;
  margin: 16px;
  background: white;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.product-card img {
  width: 100%;
  height: 300px;
  object-fit: cover;
  border-radius: 4px;
  margin-bottom: 12px;
}

.price {
  font-size: 24px;
  font-weight: 700;
  color: #f5a623;
  margin: 12px 0;
}

form button {
  width: 100%;
  padding: 12px;
  background: #667eea;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 600;
}

form button:hover {
  background: #764ba2;
}
```

**Purpose:**
- ✅ Style HTML elements
- ✅ Responsive design (mobile, tablet, desktop)
- ✅ Animations and transitions
- ✅ Served as static files (no processing needed)

**Directory structure:**
```
static/
├── styles/
│   ├── styles.css          (main stylesheet)
│   ├── cart.css
│   ├── itk-fashion.css
│   └── bot.css
├── images/
│   ├── product-1.webp
│   ├── product-2.webp
│   └── blog-hero-2.webp
├── icons/
│   └── ... (SVG/PNG icons)
└── vendor/
    └── aos/ (external JS library)
```

---

### 5. JavaScript Files (Minimal Interactivity)

**File: `static/vendor/aos/aos.js`**

This is AOS (Animate On Scroll) - a lightweight animation library.

```javascript
// When page loads, enable scroll animations
document.addEventListener('DOMContentLoaded', function() {
    AOS.init({
        duration: 1000,
        once: true,
        offset: 100
    });
});
```

**How it's used in HTML:**
```html
<div class="col-12 col-lg-6" data-aos="fade-right">
    <h1>Your Style, Your Statement</h1>
</div>
```

**Why minimal JavaScript?**
- ✅ No heavy React/Vue bundle
- ✅ Server renders complete HTML
- ✅ Browser just needs to animate (light work)
- ✅ Faster page load times
- ✅ Better SEO (HTML sent to browser, not JS)

---

## Data Flow Diagram

### User Opens Home Page

```
┌─────────────────────────────────────────────────────────────────┐
│ Browser: User clicks on https://shop.example.com/              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ Go Frontend Server (homeHandler)                                │
│                                                                  │
│ 1. Parse HTTP GET request for "/"                              │
│ 2. Extract sessionID from cookies                              │
│ 3. Call gRPC services:                                         │
│    - ProductCatalogService.ListProducts()                      │
│    - CartService.GetCart(sessionID)                            │
│    - CurrencyService.GetSupportedCurrencies()                 │
│ 4. Process response data:                                       │
│    - Format prices for current currency                        │
│    - Count cart items                                          │
│    - Build productView objects                                 │
│ 5. Execute HTML template with data:                            │
│    templates.ExecuteTemplate(w, "home", dataMap)              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ HTML Response (Complete HTML string)                            │
│                                                                  │
│ <!DOCTYPE html>                                                 │
│ <html>                                                          │
│   <head>                                                        │
│     <link href="/static/styles/styles.css">                    │
│     <script src="/static/vendor/aos/aos.js"></script>         │
│   </head>                                                       │
│   <body>                                                        │
│     <h1>Your Style, Your Statement</h1>                        │
│     <div class="products">                                      │
│       <div class="product-card">                               │
│         <h3>Premium Shirt</h3>                                 │
│         <p class="price">₹1,299.99</p>                         │
│         <form method="POST" action="/cart">                    │
│           <input type="hidden" name="product_id" value="1">   │
│           <select name="quantity">                             │
│             <option value="1">Qty 1</option>                   │
│           </select>                                            │
│           <button type="submit">Add to Cart</button>           │
│         </form>                                                │
│       </div>                                                   │
│       ... (more products)                                      │
│     </div>                                                      │
│   </body>                                                       │
│ </html>                                                         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ Browser Rendering                                               │
│                                                                  │
│ 1. Parse HTML                                                   │
│ 2. Load CSS from /static/styles/styles.css                     │
│ 3. Load JS from /static/vendor/aos/aos.js                      │
│ 4. Render styled HTML                                           │
│ 5. Execute JavaScript (animations on scroll)                   │
│ 6. Display complete page to user                               │
└─────────────────────────────────────────────────────────────────┘
```

---

### User Adds Product to Cart

```
┌─────────────────────────────────────────────────────────────────┐
│ Browser: User clicks "Add to Cart" button                       │
│ HTML Form submitted (POST)                                      │
│ POST /cart                                                      │
│ Headers: sessionID, product_id=1, quantity=2                   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ Go Frontend Server (addToCartHandler)                           │
│                                                                  │
│ 1. Parse form data: product_id="1", quantity="2"               │
│ 2. Get sessionID from cookies                                  │
│ 3. Call CartService.AddItem(sessionID, product_id, quantity)   │
│    via gRPC                                                     │
│ 4. If success:                                                  │
│    - Redirect to /cart (HTTP 303)                              │
│    - Browser automatically follows redirect                    │
│ 5. If error:                                                    │
│    - Render error template                                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ Browser: Follows redirect to /cart                              │
│ GET /cart                                                       │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ Go Frontend Server (viewCartHandler)                            │
│                                                                  │
│ 1. Get sessionID from cookies                                  │
│ 2. Call CartService.GetCart(sessionID) via gRPC                │
│ 3. For each item in cart:                                      │
│    - Call ProductCatalogService.GetProduct(productID)          │
│    - Convert prices using CurrencyService                      │
│ 4. Render cart.html template with updated cart                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ HTML Response: Cart page with updated items                    │
│ Browser displays cart with:                                     │
│ - Updated product count                                         │
│ - Prices converted to user's currency                           │
│ - Total cost calculated                                         │
│ - Checkout button ready                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Go's Role: Bridge Between Frontend & Backend Services

### Go Frontend Service = **API Orchestrator + Template Engine**

```
┌──────────────────────────────────────────────────────────────────┐
│ Go Frontend Service                                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Layer 1: HTTP Server                                            │
│ - Listen on port 8080                                           │
│ - Route incoming HTTP requests                                  │
│ - Manage sessions/cookies                                       │
│ - Authentication middleware                                     │
│                                                                   │
│ Layer 2: gRPC Clients (Service Orchestration)                  │
│ - ProductCatalogService (Get products)                          │
│ - CartService (Get/Add/Remove items)                            │
│ - CurrencyService (Convert prices)                              │
│ - CheckoutService (Process orders)                              │
│ - PaymentService (Process payments)                             │
│ - ShippingService (Calculate shipping)                          │
│ - RecommendationService (Get suggestions)                       │
│ - AdService (Fetch ads)                                         │
│                                                                   │
│ Layer 3: Data Processing                                        │
│ - Format prices                                                  │
│ - Calculate totals                                               │
│ - Validate input                                                 │
│ - Handle errors gracefully                                       │
│                                                                   │
│ Layer 4: Template Engine                                        │
│ - Parse HTML templates                                           │
│ - Inject dynamic data                                            │
│ - Render complete HTML strings                                   │
│                                                                   │
│ Layer 5: Static File Serving                                    │
│ - CSS files                                                      │
│ - JavaScript files                                               │
│ - Images                                                         │
│ - Fonts                                                          │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
     ↑                                         ↓
  Browser                              Backend Services
  (HTML/CSS/JS)                        (gRPC APIs)
```

---

## Why Go for Frontend (Not Node.js/Python)?

### Comparison

| Aspect | Go | Node.js | Python |
|--------|----|---------| -------|
| **Performance** | ⭐⭐⭐⭐⭐ Fast compiled | ⭐⭐⭐⭐ Event loop | ⭐⭐⭐ Good enough |
| **Concurrency** | ⭐⭐⭐⭐⭐ Goroutines | ⭐⭐⭐⭐ Async/await | ⭐⭐⭐ Async |
| **Memory** | ⭐⭐⭐⭐⭐ Low (~50MB) | ⭐⭐⭐⭐ Medium (~150MB) | ⭐⭐⭐ Higher (~200MB) |
| **Container Image** | ⭐⭐⭐⭐⭐ Small ~50MB | ⭐⭐⭐ Medium ~200MB | ⭐⭐⭐ Medium ~300MB |
| **gRPC Speed** | ⭐⭐⭐⭐⭐ Native | ⭐⭐⭐⭐ Good | ⭐⭐⭐ Good |
| **Startup Time** | ⭐⭐⭐⭐⭐ <100ms | ⭐⭐⭐⭐ ~300ms | ⭐⭐⭐ ~500ms |

### Go's Advantages for Frontend Service:

1. **Concurrent Service Calls**
   ```go
   // Go can handle 100 users simultaneously, each calling 5-6 backend services
   // Each request gets goroutine, no thread pool limitations
   // Example: homeHandler calls 3 services = 300 concurrent gRPC calls (100 users × 3)
   ```

2. **Low Resource Footprint (Critical in Kubernetes)**
   ```
   Go container: 50-100 MB RAM, <100ms startup
   Can run 100 frontend pods in a cluster cheaply
   ```

3. **Built-in HTTP/gRPC Libraries**
   ```go
   - No external dependencies for HTTP server
   - Native gRPC support
   - Standard library covers most needs
   ```

4. **Single Binary Deployment**
   ```
   Dockerfile:
   FROM golang:1.21
   COPY . /app
   RUN go build -o frontend .
   
   Result: One executable file, no npm/pip dependencies
   ```

5. **Template Engine**
   ```go
   // Go's html/template is safe and efficient
   // Prevents XSS attacks automatically
   // Fast rendering of 1000+ products
   templates.ExecuteTemplate(w, "home", data)
   ```

---

## Complete Request/Response Cycle

### Example: User Browses Products

```
REQUEST PHASE:
═════════════
1. Browser sends: GET /
   
2. Go Frontend Server receives request
   └─→ homeHandler() executes
   
3. Inside homeHandler:
   a) Extract sessionID from cookies
   b) Call ProductCatalogService.ListProducts() (gRPC)
      └─→ Wait for response...
      └─→ Receives [Product{id:1, name:"Shirt", price:1299}, ...]
   
   c) Call CurrencyService.GetSupportedCurrencies() (gRPC)
      └─→ Receives ["INR", "USD", "EUR", ...]
   
   d) Call CartService.GetCart(sessionID) (gRPC)
      └─→ Receives [CartItem{productId:2, quantity:1}, ...]
   
   e) Process data:
      - Convert product prices to user's currency
      - Format prices with currency symbol
      - Count total items in cart
   
   f) Execute template/home.html:
      └─→ Go engine injects processed data
      └─→ Loops through products
      └─→ Renders HTML string with all products
      └─→ Includes links to CSS and JS files


RESPONSE PHASE:
═══════════════
4. Go sends HTTP response:
   Status: 200 OK
   Content-Type: text/html; charset=utf-8
   Set-Cookie: session_id=xyz123; Max-Age=172800
   
   Body: (Complete HTML)
   <!DOCTYPE html>
   <html>
     <head>
       <link href="/static/styles/styles.css">
       <script src="/static/vendor/aos/aos.js"></script>
     </head>
     <body>
       <h1>Your Style, Your Statement</h1>
       <div id="products">
         <div class="product-card" data-aos="fade-in">
           <h3>Premium Shirt</h3>
           <p class="price">₹1,299.99</p>
           <form method="POST" action="/cart">
             <input type="hidden" name="product_id" value="1">
             <select name="quantity">...</select>
             <button type="submit">Add to Cart</button>
           </form>
         </div>
         ... (more products)
       </div>
       <div>Cart Items: 1</div>
     </body>
   </html>


BROWSER PHASE:
═════════════
5. Browser receives HTML response
   └─→ Parse HTML
   └─→ Download CSS from /static/styles/styles.css
   └─→ Download JS from /static/vendor/aos/aos.js
   └─→ Render page (HTML + CSS)
   └─→ Execute JavaScript (animation library)
   └─→ User sees complete styled page with working buttons


USER INTERACTION:
════════════════
6. User clicks "Add to Cart" button
   └─→ HTML form submits POST to /cart
   └─→ Browser sends: POST /cart, product_id=1, quantity=2
   
7. Go addToCartHandler processes:
   └─→ Call CartService.AddItem(sessionID, product_id, quantity)
   └─→ If success: Redirect to /cart (GET request)
   └─→ Browser follows redirect
   
8. Go viewCartHandler processes:
   └─→ Call CartService.GetCart(sessionID)
   └─→ Render cart.html template with items
   └─→ Send HTML response
   
9. Browser displays updated cart
   └─→ User sees their item added with quantity and price
```

---

## File Organization Summary

```
frontend/
├── main.go                    ← Go HTTP server setup + routes
├── handlers.go                ← Request handlers (home, product, cart, checkout, etc)
├── rpc.go                     ← gRPC client calls to backend services
├── middleware.go              ← Auth, logging, session management
├── validator/                 ← Input validation logic
├── money/                     ← Currency/price calculations
├── genproto/                  ← Generated gRPC client code (from .proto files)
│   ├── demo.pb.go
│   └── demo_grpc.pb.go
│
├── templates/                 ← HTML templates (Go html/template)
│   ├── home.html              ← Homepage with products
│   ├── product.html           ← Single product details
│   ├── cart.html              ← Shopping cart
│   ├── login.html             ← Login form
│   ├── register.html          ← Registration form
│   ├── checkout.html          ← Checkout form
│   ├── order.html             ← Order confirmation
│   ├── header.html            ← Shared header (navigation, logo)
│   ├── footer.html            ← Shared footer
│   ├── assistant.html         ← AI shopping assistant
│   └── error.html             ← Error messages
│
├── static/                    ← Static assets (served as-is)
│   ├── styles/                ← CSS files
│   │   ├── styles.css         ← Main stylesheet
│   │   ├── cart.css           ← Cart page styles
│   │   ├── bot.css            ← Chatbot styles
│   │   └── itk-fashion.css    ← Fashion theme
│   │
│   ├── images/                ← Product images
│   │   ├── product-1.webp
│   │   ├── product-2.webp
│   │   └── ...
│   │
│   ├── icons/                 ← SVG/PNG icons
│   │   └── ...
│   │
│   └── vendor/                ← External JS libraries
│       └── aos/
│           └── aos.js         ← Animate On Scroll library
│
├── Dockerfile                 ← Container image definition
├── Jenkinsfile                ← CI/CD pipeline
├── go.mod                     ← Go dependencies
└── go.sum                     ← Dependency lock file
```

---

## Middleware Chain (Request Processing Order)

```
HTTP Request comes in
        ↓
┌─────────────────────────────────────┐
│ logHandler (logging middleware)      │
│ - Log request path, method, params   │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ ensureSessionID (session middleware) │
│ - Check for session_id cookie        │
│ - Generate new if not exists         │
│ - Set session cookie (48 hour max)   │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ requireAuth (authentication middleware)
│ - Check JWT token validity           │
│ - Verify token not expired           │
│ - Reject if invalid                  │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ otelhttp (observability middleware)  │
│ - Generate trace ID                  │
│ - Track request latency              │
│ - Collect metrics                    │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ Route to appropriate handler         │
│ - homeHandler (GET /)                │
│ - productHandler (GET /product/:id)  │
│ - viewCartHandler (GET /cart)        │
│ - addToCartHandler (POST /cart)      │
│ - etc.                               │
└─────────────────────────────────────┘
        ↓
        ↓ (Handler processes: calls gRPC services, renders template)
        ↓
HTTP Response sent back to browser
```

---

## Key Takeaways

### 1. **Go ≠ Node.js SPA**
   - NOT building a JavaScript bundle
   - NOT sending JSON APIs
   - Frontend is **Server-Side Rendered**

### 2. **Go's Four Jobs:**
   1. HTTP server (receive requests)
   2. gRPC orchestrator (call backend services)
   3. Template engine (render HTML)
   4. Static file server (CSS, JS, images)

### 3. **HTML/CSS/JS Hierarchy:**
   ```
   Go generates HTML (dynamic) ← Contains product data, prices, user info
   ↓
   CSS styles HTML (visual) ← Makes it look nice
   ↓
   JS enhances HTML (interaction) ← Scroll animations, form validation
   ```

### 4. **Why Go for This?**
   - Single HTTP request calls 3-6 backend services
   - Must be fast, lightweight, concurrent
   - Go's goroutines perfect for this
   - Small memory footprint (Kubernetes-friendly)
   - Built-in template engine (no external framework needed)

### 5. **User Never Sees Complexity:**
   - Browser just sees HTML/CSS/JS
   - But behind scenes: Go orchestrating multiple microservices
   - All backend complexity hidden in Go server

---

## Real-World Comparison

### Traditional SPA (React/Vue)
```
Browser ←→ API Server (Node.js)
           └→ [API calls]
           └→ [API calls]
           └→ [API calls]
           
Complexity: Browser makes 5+ network requests
Page load: 2-3 seconds (bundle parsing + gzipping)
```

### SSR with Go (This Architecture)
```
Browser ←→ Frontend Service (Go)
              └→ [gRPC call] (internal)
              └→ [gRPC call] (internal)
              └→ [gRPC call] (internal)
              └→ [Render HTML]
           
Complexity: Single HTTP request, multiple backend calls handled server-side
Page load: <500ms (pre-rendered HTML)
SEO: ✅ Better (HTML sent immediately, not JavaScript)
```

---

## Conclusion

**Go + HTML/CSS/JS is a Server-Side Rendering (SSR) architecture where:**

- **Go** = Backend orchestrator (calls services, processes data)
- **HTML** = Template structure (generated dynamically by Go)
- **CSS** = Styling (served as static files)
- **JS** = Light interactivity (scroll animations, form enhancements)

This is a **production-grade, high-performance** architecture perfect for:
- E-commerce platforms
- Content-heavy websites
- Systems with complex backend orchestration
- Kubernetes deployments (low resource usage)
- SEO-critical applications
