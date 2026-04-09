# ChargeLK Admin Portal - Quick Start Guide

## Prerequisites
- Node.js 20+ and npm
- ChargeLK.UM backend running (default: http://localhost:5002)
- ChargeLK.OCPP backend running (default: http://localhost:5001)

## Installation

```bash
cd ChargeLK.AdminUI
npm install
```

## Configuration

### Environment Variables

The Admin UI uses environment variables for different deployment scenarios.

**Development (`.env.development`)** - Already configured:
```env
VITE_API_BASE_URL=/api
VITE_APP_NAME=ChargeLK Admin
VITE_APP_VERSION=1.0.0
VITE_ENABLE_DEBUG=true
```

**Production (`.env.production`)** - Create for deployment:
```env
VITE_API_UM_URL=http://localhost:5002/api
VITE_API_OCPP_URL=http://localhost:5001/api
VITE_WS_OCPP_URL=ws://localhost:5000
VITE_APP_NAME=ChargeLK Admin
VITE_APP_VERSION=1.0.0
VITE_ENABLE_DEBUG=false
```

See [ENV-CONFIGURATION.md](../../ChargeLK.AdminUI/ENV-CONFIGURATION.md) for complete guide.

### API Proxy (Development)

The `vite.config.ts` is already configured with Vite proxy:

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api/um': {
        target: 'http://localhost:5002',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api\/um/, '/api')
      },
      '/api/ocpp': {
        target: 'http://localhost:5001',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api\/ocpp/, '/api')
      }
    }
  }
})
```

This proxies API requests to avoid CORS issues in development.

## Running the Application

### Development
```bash
npm run dev
```
Access at: http://localhost:3000

**Note:** Make sure both backend services are running:
- UM Service: http://localhost:5002
- OCPP Service: http://localhost:5001

### Production Build
```bash
# Create .env.production if needed
cp .env.production.template .env.production
nano .env.production  # Update with your server URLs

# Build
npm run build

# Preview build locally
npm run preview
```

## Default Login
Use the credentials from your ChargeLK.UM service. If you haven't created a user yet:

```bash
# Use the ChargeLK.UM API to create an admin user
curl -X POST http://localhost:5001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@chargelk.com",
    "userName": "admin",
    "password": "Admin@123"
  }'
```

## Features

### 1. Dashboard
- Real-time charge point statistics
- Active transaction monitoring
- OCPP tag overview
- Weekly transaction trends

### 2. Charge Points Management
- View all charge points with online/offline status
- Register new charge points
- View detailed charge point information
- Remote operations:
  - Start/stop transactions
  - Unlock connectors
  - Reset charge points (soft/hard)
  - Change configurations

### 3. OCPP Tags (RFID)
- Create and manage RFID tags
- Block/unblock tags with reasons
- Link tags to users
- Set expiry dates
- Search and filter

### 4. Transactions
- View all charging transactions
- Filter by status (active/completed)
- Search by charge point or tag
- View energy consumption
- Pagination support

### 5. User Management
- Create new users
- Lock/unlock user accounts
- Manage roles and permissions
- View user activity

## Redux DevTools

Install Redux DevTools browser extension:
- [Chrome](https://chrome.google.com/webstore/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd)
- [Firefox](https://addons.mozilla.org/en-US/firefox/addon/reduxdevtools/)

Features:
- Inspect Redux state
- View API calls and responses
- Time-travel debugging
- Export/import state

## Project Structure

```
src/
├── main.tsx                 # App entry point
├── App.tsx                  # Main app component
├── theme.ts                 # MUI theme configuration
├── store/
│   ├── index.ts            # Redux store setup
│   ├── hooks.ts            # Typed hooks
│   ├── api/
│   │   ├── baseApi.ts      # RTK Query base
│   │   ├── authApi.ts      # Auth endpoints
│   │   ├── usersApi.ts     # User endpoints
│   │   ├── chargePointsApi.ts
│   │   ├── ocppTagsApi.ts
│   │   └── transactionsApi.ts
│   └── slices/
│       └── authSlice.ts    # Auth state
├── pages/
│   ├── Login.tsx
│   ├── Dashboard.tsx
│   ├── ChargePoints.tsx
│   ├── ChargePointDetail.tsx
│   ├── OcppTags.tsx
│   ├── Transactions.tsx
│   ├── Users.tsx
│   └── Settings.tsx
└── components/
    └── Layout/
        └── Layout.tsx
```

## API Endpoints Reference

### User Management API (`/api/um`)
- `POST /auth/login` - User login
- `POST /auth/register` - User registration
- `POST /auth/refresh` - Refresh token
- `POST /auth/logout` - Logout
- `GET /auth` - Get all users
- `GET /auth/{id}` - Get user by ID
- `POST /auth/role` - Create role
- `POST /auth/user-role` - Assign role to user
- `POST /auth/{id}/lock` - Lock user
- `POST /auth/{id}/unlock` - Unlock user

### OCPP API (`/api/ocpp`)

#### Charge Points
- `GET /chargepoints` - Get all charge points
- `GET /chargepoints/{chargeBoxId}` - Get charge point details
- `POST /chargepoints` - Register charge point
- `DELETE /chargepoints/{id}` - Delete charge point
- `POST /chargepoints/{chargeBoxId}/remote-start` - Start transaction
- `POST /chargepoints/{chargeBoxId}/remote-stop` - Stop transaction
- `POST /chargepoints/{chargeBoxId}/unlock-connector` - Unlock connector
- `POST /chargepoints/{chargeBoxId}/reset` - Reset charge point

#### OCPP Tags
- `GET /ocpptags` - Get all tags
- `GET /ocpptags/{id}` - Get tag by ID
- `GET /ocpptags/by-idtag/{idTag}` - Get tag by tag string
- `POST /ocpptags` - Create tag
- `PUT /ocpptags/{id}` - Update tag
- `POST /ocpptags/{id}/block` - Block tag
- `POST /ocpptags/{id}/unblock` - Unblock tag
- `DELETE /ocpptags/{id}` - Delete tag

#### Transactions
- `GET /transactions` - Get paginated transactions
- `GET /transactions/{id}` - Get transaction details
- `GET /transactions/active` - Get active transactions
- `GET /transactions/by-chargepoint/{chargePointId}` - Get transactions by charge point

## Common Tasks

### Adding a New API Endpoint

1. **Define Types** (`src/store/api/yourApi.ts`):
```typescript
export interface YourEntity {
  id: number
  name: string
}
```

2. **Create Endpoint**:
```typescript
export const yourApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    getEntities: builder.query<YourEntity[], void>({
      query: () => '/api/path/entities',
      providesTags: ['YourEntity'],
    }),
    createEntity: builder.mutation<YourEntity, Partial<YourEntity>>({
      query: (data) => ({
        url: '/api/path/entities',
        method: 'POST',
        body: data,
      }),
      invalidatesTags: ['YourEntity'],
    }),
  }),
})

export const { useGetEntitiesQuery, useCreateEntityMutation } = yourApi
```

3. **Use in Component**:
```typescript
import { useGetEntitiesQuery, useCreateEntityMutation } from '../store/api/yourApi'

function YourComponent() {
  const { data, isLoading, error } = useGetEntitiesQuery()
  const [create] = useCreateEntityMutation()
  
  const handleCreate = async (newEntity) => {
    await create(newEntity).unwrap()
  }
  
  // ...
}
```

### Debugging API Calls

1. **Redux DevTools**: View all API calls in the Redux DevTools
2. **Network Tab**: Check browser network tab for requests
3. **Console**: RTK Query logs cache hits/misses in development

### Cache Management

```typescript
// Manual cache invalidation
dispatch(yourApi.util.invalidateTags(['YourEntity']))

// Refetch query
refetch()

// Reset API state
dispatch(yourApi.util.resetApiState())
```

## Troubleshooting

### Issue: API calls fail with 404
**Solution**: Check Vite proxy configuration and ensure backend services are running

### Issue: Authentication redirects to login
**Solution**: Check if token is stored in Redux state and localStorage

### Issue: Data not updating after mutation
**Solution**: Ensure `invalidatesTags` is set correctly in mutation

### Issue: Type errors in IDE
**Solution**: Run `npm run build` to check for TypeScript errors

### Issue: CORS errors
**Solution**: Configure CORS in backend or use Vite proxy

## Performance Tips

1. **Use RTK Query hooks**: They automatically handle caching
2. **Set `skip` parameter**: Skip queries when data isn't needed
3. **Use `refetchOnMount`**: Control when queries refetch
4. **Pagination**: Implemented in transactions, use for large datasets

## Security Considerations

1. **Token Storage**: Tokens are stored in Redux with persistence
2. **Automatic Logout**: 401 responses trigger automatic logout
3. **Token Refresh**: Automatic token refresh prevents session expiry
4. **HTTPS**: Use HTTPS in production

## Deployment

### Build for Production
```bash
npm run build
```

### Deploy to Static Hosting
The `dist/` folder contains static files. Deploy to:
- Nginx
- Apache
- Vercel
- Netlify
- AWS S3 + CloudFront

### Environment Variables
Create `.env.production`:
```
VITE_API_UM_URL=https://api.chargelk.com/um
VITE_API_OCPP_URL=https://api.chargelk.com/ocpp
```

Update `baseApi.ts` to use environment variables:
```typescript
baseUrl: import.meta.env.VITE_API_BASE_URL || '/api'
```

## Support & Resources

- **Documentation**: See `/ChargeLK.WIKI/RTK-IMPLEMENTATION-PLAN.md`
- **Backend APIs**: ChargeLK.UM and ChargeLK.OCPP services
- **RTK Query Docs**: https://redux-toolkit.js.org/rtk-query/overview
- **MUI Components**: https://mui.com/material-ui/

## Next Steps

1. Configure API proxy for your environment
2. Create admin user in backend
3. Login and explore features
4. Customize theme in `src/theme.ts`
5. Add role-based UI rendering
6. Implement WebSocket for real-time updates
7. Add unit and integration tests

---

**Version**: 1.0.0  
**Last Updated**: December 16, 2025

