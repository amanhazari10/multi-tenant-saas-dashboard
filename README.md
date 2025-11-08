# Multi-Tenant SaaS Dashboard

A production-ready, full-stack SaaS dashboard application featuring strict tenant isolation, dynamic per-tenant branding, and comprehensive validation. Built with React, Node.js/Express, MongoDB, and Docker.

## Features

### ✅ Core Architecture
- **Tenant Resolution**: Support for subdomain, URL path prefix, or signed header-based tenant identification
- **Data Isolation**: MongoDB partitioning with tenantId discriminator and compound indexes
- **Access Control**: JWT-based authentication with tenantId claims and middleware enforcement
- **Runtime Theming**: Per-tenant branding without code redeployment (logos, colors, typography)
- **Secure Assets**: Tenant-scoped asset storage with ETags and fallback defaults

### ✅ Management & Observability
- **Admin Console**: Tenant admins can update theme, name, and feature toggles
- **Structured Logging**: Correlation IDs and structured logs with tenantId
- **Metrics**: Per-tenant performance tracking and dashboards
- **Observability**: Request tracing and audit logs

### ✅ Performance & Testing
- **Compound Indexes**: Optimized (tenantId, createdAt) queries with pagination
- **Rate Limiting**: Per-tenant request throttling
- **Validation Tests**: Smoke tests confirming isolation and theming correctness
- **Seed Tenants**: Pre-configured tenants (acme, globex) for testing

### ✅ Deployment
- **Docker Compose**: Complete dev environment with frontend, backend, and MongoDB
- **Environment-Based Config**: Local, staging, and production configurations
- **Containerized**: Separate Dockerfiles for backend and frontend

## Project Structure

```
multi-tenant-saas-dashboard/
├── backend/
│   ├── src/
│   │   ├── middleware/          # Tenant resolution, JWT auth, logging
│   │   │   ├── tenant.js        # Tenant context middleware
│   │   │   ├── auth.js          # JWT verification
│   │   │   └── logger.js        # Structured logging
│   │   ├── models/              # MongoDB schemas
│   │   │   ├── Tenant.js        # Tenant model
│   │   │   ├── User.js          # User model
│   │   │   ├── Resource.js      # Sample resource model
│   │   │   └── Theme.js         # Theme configuration model
│   │   ├── routes/              # API endpoints
│   │   │   ├── auth.js          # Login, signup
│   │   │   ├── tenant.js        # Tenant admin APIs (theme, settings)
│   │   │   ├── resource.js      # CRUD operations for resources
│   │   │   └── asset.js         # Asset upload/retrieval
│   │   ├── utils/
│   │   │   ├── themeManager.js  # Theme fetching and caching
│   │   │   ├── assetManager.js  # Asset storage and serving
│   │   │   └── validators.js    # Tenant isolation validators
│   │   ├── seed/                # Database seeders
│   │   │   └── seed.js          # Populate test tenants
│   │   ├── tests/               # Isolation and smoke tests
│   │   │   ├── isolation.test.js
│   │   │   └── theme.test.js
│   │   └── app.js               # Express app entry
│   ├── Dockerfile               # Backend container
│   ├── package.json
│   └── .dockerignore
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── ThemeProvider.jsx    # React Context for theme
│   │   │   ├── Login.jsx            # Authentication form
│   │   │   ├── Dashboard.jsx        # Main dashboard UI
│   │   │   ├── AdminConsole.jsx     # Tenant settings panel
│   │   │   ├── ResourceList.jsx     # Display tenant resources
│   │   │   └── Navigation.jsx       # App header/nav
│   │   ├── utils/
│   │   │   ├── api.js              # API client with tenant context
│   │   │   ├── auth.js             # Auth helpers (JWT storage)
│   │   │   └── validators.js       # Form and data validators
│   │   ├── themes/                  # Theme configs (fallback)
│   │   │   ├── default.css
│   │   │   └── variables.css
│   │   ├── App.jsx                 # App entry point
│   │   └── index.jsx               # React DOM render
│   ├── Dockerfile                  # Frontend container
│   ├── package.json
│   └── .dockerignore
├── docker-compose.yml              # Local dev environment
├── .env.example                    # Environment template
├── README.md                       # This file
└── ARCHITECTURE.md                 # Detailed architecture docs
```

## Quick Start

### Prerequisites
- Docker & Docker Compose
- Node.js 16+ (for local development)
- MongoDB 5+ (or use Docker)

### Local Development

```bash
# Clone the repository
git clone https://github.com/yourusername/multi-tenant-saas-dashboard.git
cd multi-tenant-saas-dashboard

# Create .env from template
cp .env.example .env

# Start services (backend, frontend, MongoDB)
docker-compose up --build

# In another terminal, seed test tenants
docker-compose exec backend npm run seed

# Run tests
docker-compose exec backend npm test
```

Once running:
- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:5000
- **MongoDB**: mongodb://localhost:27017

### Default Test Tenants
- **Tenant ID**: `acme` (primary color: #003366)
- **Tenant ID**: `globex` (primary color: #006600)

## API Endpoints

### Authentication
```
POST /api/auth/login
Body: { "email": "user@acme.com", "password": "pass", "tenantId": "acme" }
Response: { "token": "jwt_token", "tenant": {...}, "theme": {...} }
```

### Tenant Admin
```
GET  /api/tenant/theme
GET  /api/tenant/settings
POST /api/tenant/theme
Body: { "colors": { "primary": "#...", "accent": "#..." }, "logo": "url" }
```

### Resources (Example CRUD)
```
GET    /api/resource              # List tenant's resources (paginated)
POST   /api/resource              # Create resource
GET    /api/resource/:id          # Get single resource
PUT    /api/resource/:id          # Update resource
DELETE /api/resource/:id          # Delete resource
```

**Note**: All requests require `Authorization: Bearer <jwt_token>` header. Tenant is resolved from JWT claims.

## Tenant Resolution

The backend supports three tenant resolution methods (checked in order):

1. **Header-based** (Highest Priority):
   ```
   X-Tenant-Id: acme
   ```

2. **URL Path Prefix**:
   ```
   /t/acme/api/resource
   ```

3. **Subdomain** (Not enabled in scaffold, requires reverse proxy config):
   ```
   acme.example.com/api/resource
   ```

## Data Isolation Mechanism

All database queries filter by `tenantId`:

```javascript
// Example: Only fetch resources for the authenticated tenant
const resources = await Resource.find({ tenantId: req.tenantId })
  .limit(20)
  .sort({ createdAt: -1 });
```

Database indexes ensure fast queries:
```javascript
resourceSchema.index({ tenantId: 1, createdAt: -1 });
```

## Runtime Theming

On login, the client receives the tenant's theme:

```javascript
// Backend response
{
  "token": "...",
  "theme": {
    "tenantName": "ACME Corp",
    "logo": "https://assets.example.com/acme/logo.png",
    "colors": { "primary": "#003366", "accent": "#FF6600" },
    "typography": { "fontFamily": "Arial, sans-serif", "fontSize": 14 }
  }
}
```

Frontend React component applies CSS variables:

```jsx
// ThemeProvider.jsx
Re.useEffect(() => {
  document.documentElement.style.setProperty('--primary', theme.colors?.primary);
  document.documentElement.style.setProperty('--accent', theme.colors?.accent);
}, [theme]);
```

## Testing

### Run Isolation Tests
```bash
npm test -- isolation.test.js
```

Tests verify:
- Cross-tenant reads are blocked
- Cross-tenant writes are blocked
- JWT tokens reject mismatched tenantId

### Run Smoke Tests
```bash
npm test -- smoke.test.js
```

Tests verify:
- Tenant login flow
- Theme loading on dashboard
- Admin console theme updates
- Resource CRUD operations

### Manual Testing

1. Open http://localhost:3000
2. Login as acme user (username: `acme@acme.com`, password: `password123`)
3. Verify acme theme appears (blue primary color)
4. Open Admin Console, change primary color, refresh page
5. Logout and login as globex user
6. Verify globex theme loads (green primary color)

## Environment Variables

See `.env.example` for all variables:

```bash
# MongoDB
MONGO_URI=mongodb://mongo:27017/saas_db

# JWT
JWT_SECRET=your-super-secret-key-change-in-prod
JWT_EXPIRES_IN=24h

# Server
PORT=5000
NODE_ENV=development

# Frontend
REACT_APP_API_URL=http://localhost:5000

# Logging
LOG_LEVEL=info
```

## Production Deployment

1. Update `.env` with production values
2. Build images: `docker-compose build`
3. Push to registry (e.g., Docker Hub, ECR)
4. Deploy on Kubernetes, Docker Swarm, or managed container service
5. Set up SSL/TLS reverse proxy (e.g., Nginx, CloudFlare)
6. Configure MongoDB Atlas or managed database
7. Set up observability (e.g., DataDog, New Relic)

## Architecture Decisions

### Why Separate DBs vs. Shared DB with TenantId?

This scaffold uses **shared database with tenantId discriminator**:
- **Pros**: Easier operational overhead, single MongoDB instance, cost-effective
- **Cons**: Requires strict filtering; indexes essential

For strict isolation, use separate databases per tenant:
```javascript
const dbUri = `mongodb://mongo:27017/saas_${tenantId}`;
```

### Why JWT with TenantId Claim?

- **Stateless auth**: No session storage needed
- **Tenant context in token**: Middleware can verify tenant match
- **Scalable**: Works across multiple backend instances

### Why CSS Variables for Theming?

- **Dynamic**: Applied at runtime without recompile
- **Performance**: Native browser support, minimal overhead
- **Fallback**: Graceful degradation on older browsers

## Troubleshooting

### "Cross-tenant isolation failed" test error
- Ensure `tenantId` is checked in ALL database queries
- Verify JWT middleware enforces tenant match
- Check compound indexes are created

### Theme not updating in admin console
- Clear browser cache (Ctrl+Shift+Delete)
- Check backend returns updated theme in response
- Verify CSS variables are applied (DevTools > Computed styles)

### MongoDB connection refused
- Check MongoDB is running: `docker ps | grep mongo`
- Verify `MONGO_URI` in `.env` matches docker-compose service

### React app not connecting to backend
- Verify `REACT_APP_API_URL` in `.env`
- Check CORS is enabled on backend
- Inspect Network tab in DevTools for failed requests

## Next Steps

1. **Enhanced Admin Console**: Add feature toggles, usage analytics
2. **Multi-Region Replication**: MongoDB geographically distributed
3. **GraphQL API**: Alternative to REST with better querying
4. **Mobile App**: React Native mobile client
5. **Advanced Reporting**: Business intelligence dashboards per tenant
6. **Webhook Integration**: Tenant event notifications
7. **SSO Integration**: SAML, OAuth for enterprise tenants

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit changes: `git commit -am 'Add feature'`
4. Push to branch: `git push origin feature/your-feature`
5. Submit a pull request

## License

MIT License - see LICENSE file for details

## Support

For issues or questions:
- Open a GitHub Issue
- Check existing documentation in `/docs`
- Review test files for usage examples
