Title: Role-Based Access Control (RBAC) with Keycloak in React Frontend
Date: 2024-10-06  
Category: wbb development
Tags: keycloak, rbac, react, keycloak-js  
Author: Leon Ramzews  
Summary: Securing a React frontend with Keycloak such that only users with certain roles are allowed to access pages or components in the application.

## Introduction

In this blog post, I’ll walk you through how I:

- Set up Keycloak, from installation to configuration,
- Embedded Keycloak into a simple React frontend so that users can log in and out via a button,
- Implemented role-based access control (RBAC), ensuring that certain pages or components in the frontend are only accessible to users with the appropriate roles.

A minimal working example is available here: [https://github.com/JulianRotti/keycloak-react](https://github.com/JulianRotti/keycloak-react).

---

## Setting Up Keycloak

### Installing Keycloak with Docker

The first step was setting up Keycloak using Docker. I created a `docker-compose.yml` file to manage the Keycloak instance:

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:25.0.6
    container_name: keycloak
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
    ports:
      - "8080:8080"
    command: start-dev
    volumes:
      - keycloak_data:/opt/keycloak/data

volumes:
  keycloak_data:
```

#### Key Points:
- **Image**: The Keycloak image version `25.0.6` from the quay.io repository was used.
- **Admin Credentials**: Admin user was created with `admin/admin`.
- **Volumes**: Persistent Keycloak data is stored via Docker volumes.

Once the Docker Compose file was ready, I ran `docker-compose up`, making Keycloak accessible via [http://localhost:8080](http://localhost:8080).

### Configuring Keycloak

With Keycloak running, I configured it for my frontend:

1. **Creating a Realm**: I created a realm called `my-website`, an isolated space for managing users, roles, and clients.

2. **Defining Roles**:
    - `viewer`
    - `editor`

3. **Creating Users**:
    - `test_viewer` (assigned `viewer` role)
    - `test_editor` (assigned `editor` role)
    - I also needed to **create credentials (password)** for these users.

4. **Setting Up the Frontend Client**:
    - **Client ID**: `my-website-frontend`
    - **Root URL & Web Origins**: `http://localhost:3000`
    - **Valid Redirect URLs & Post-Logout Redirect URIs**: `http://localhost:3000/*`


#### Key Points:
- **Realm**: Isolates users and clients for a specific application.
- **RBAC Roles**: Keycloak's role system supports RBAC.
- **Client Settings**: Proper configuration ensures secure communication between React and Keycloak.

---

## React Frontend

### React Frontend Structure with Keycloak RBAC

When building a React application that integrates Keycloak for Role-Based Access Control (RBAC), organizing the folder structure is critical for maintainability and clarity. Here’s the folder structure I used and the purpose of each folder:

```plaintext
src/
├── components/
│   ├── Sidebar/
│   │   ├── Sidebar.js
│   │   ├── KeycloakButton.js
│   │   └── UserInfo.js
│   └── AccessControl.js
├── contexts/
│   └── AuthContext.js
├── pages/
│   ├── HomePage.js
│   ├── EditorRole.js
│   ├── AllRoles.js
│   └── NoAccess.js
├── routes/
│   ├── AppRoutes.js
│   └── RouteConfig.js
├── services/
│   └── keycloak.js
├── App.js
├── index.js
```

### `App.js` and `index.js`

The **key idea** behind the structure is to **centralize the authentication state** in `contexts/AuthContext.js`. By wrapping the entire app in the `AuthProvider` inside `App.js`, all components can access the authentication state consistently.

#### **`App.js`**
```js
import React from 'react';
import { BrowserRouter as Router } from 'react-router-dom';
import Sidebar from './components/Sidebar/Sidebar.js';
import AppRoutes from './routes/AppRoutes.js';
import { AuthProvider } from './contexts/AuthContext.js'; 

function App() {
  return (
    <AuthProvider>
      <Router>
        <Sidebar>
          <AppRoutes />
        </Sidebar>
      </Router>
    </AuthProvider>
  );
}

export default App;
```

#### **`index.js`**
```js
import React from 'react';
import { createRoot } from 'react-dom/client';
import './index.css';
import App from './App.js';  // Main App component

const rootElement = document.getElementById('root');
const root = createRoot(rootElement);

root.render(<App />);
```

---

### Components

The `components/` folder contains reusable components and higher-order components (HOCs) for layout and controlling access to pages.

#### **`Sidebar/Sidebar.js`**
Handles navigation, renders links based on the available routes, and enforces role-based access via the `AccessControl` component. It includes a login/logout button and user information.

```js
import React from 'react';
import { Link } from 'react-router-dom'; 
import routes from '../../routes/RouteConfig.js'; 
import KeycloakButton from './KeycloakButton.js'; 
import UserInfo from './UserInfo.js';
import AccessControl from '../AccessControl.js';

export default function Sidebar({ children }) {
  return (
    <div style={{ display: 'flex' }}>
        <div style={{ width: '20%', float: 'left' }}>
            <div>
                {routes
                .filter(route => !route.hidden)
                .map((route) => (
                    <AccessControl key={route.path} requiredRoles={route.roles}>
                        <p key={route.path}>
                            <Link to={route.path}>{route.name}</Link>
                        </p>
                    </AccessControl>
                ))}
            </div>

            <KeycloakButton />
            <UserInfo />
        </div>
        <div style={{ width: '80%', float: 'left' }}>
            {children} 
        </div>
    </div>
  );
}

```

#### **`KeycloakButton.js`**
Provides a login/logout button based on the user’s authentication status.

```js
import React, { useContext } from "react";
import { loginKeycloak, logoutKeycloak } from "../../services/keycloak.js";
import { AuthContext } from "../../contexts/AuthContext.js";

export default function KeycloakButton() {
    const { isAuthenticated } = useContext(AuthContext);

    const handleButtonClick = () => {  
        if (isAuthenticated) {
            logoutKeycloak();
        } else {
            loginKeycloak();
        }
    }

    return (
        <div>
            <button onClick={handleButtonClick}>
                { isAuthenticated ? "Logout" : "Login" }
            </button>
        </div>
    );
}
```

#### **`UserInfo.js`**
Displays the current username and roles if the user is authenticated.

```js
import React, { useContext } from "react";
import { getUsernameAndRoles } from "../../services/keycloak.js";
import { AuthContext } from "../../contexts/AuthContext.js";

export default function UserInfo () {
    const { isAuthenticated } = useContext(AuthContext);

    if (!isAuthenticated) {
        return null;
    }

    const {username, roles} = getUsernameAndRoles(['editor', 'viewer']);

    return (
        <div>
            <p>{ `username: ${username}` }</p>
            <p>{ `roles: ${roles.join(", ")}` }</p>
        </div>
    );
}
```

#### **`AccessControl.js`**

This HOC checks if the authenticated user has the necessary role(s) to access a page. If not, it redirects to the "No Access" page or hides certain components. 

```js
import React, { useContext } from 'react';
import { AuthContext } from '../contexts/AuthContext.js';
import { hasRole } from '../services/keycloak.js';
import { Navigate } from 'react-router-dom';

export default function AccessControl ({ requiredRoles, children, fallback = null, redirect = false}) {
    
    const isAuthenticated = useContext(AuthContext);
    
    if (requiredRoles.includes('public')) {
        return children;
    }
    
    const hasRequiredRole = requiredRoles.some(role => hasRole(role));

    if (!isAuthenticated || !hasRequiredRole) {
        return redirect ? <Navigate to="/no-access" replace/> : fallback;
    }

    return children;
}
```

Here we use this component to restrict the access to entire pages, but you can also use this component to **conditionally hide buttons or sections**. Here's an example of how to hide a button:

```js
<AccessControl requiredRoles={['editor']}>
  <button>Edit Content</button>
</AccessControl>
```

---

### Contexts

The `contexts/` folder contains the `AuthContext.js`, which manages the **global authentication state**. This context is crucial because it ensures that **authentication listeners are set up once**, avoiding multiple instances or duplicate event handling.

#### **`AuthContext.js`**
The `AuthContext.js` provides authentication state and methods, initializing Keycloak and handling session changes. For that authentification listeners from the keycloak service are set up so that every time the user logs in or out the `isAuthenticated` state is updated.

```js
import React, { createContext, useState, useEffect } from 'react';
import { initKeycloak, authentificationListener, checkKeycloakLogin } from "../services/keycloak.js";

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
    const [isAuthenticated, setIsAuthenticated] = useState(false);
    const [authLoading, setAuthLoading] = useState(true);

    useEffect(() => {
      const initializeKeycloak = async () => {
        await initKeycloak();
        // set the authentication state for the first time
        setIsAuthenticated(checkKeycloakLogin());
        // set up event listeners for updating the authentification state in the future
        authentificationListener(setIsAuthenticated);
        setAuthLoading(false); 
      };

      initializeKeycloak();
    }, []);

    if (authLoading) {
        return <div>Loading...</div>; 
    }
    // return a provider in which the authentication state can be accessed
    return (
        <AuthContext.Provider value={{ isAuthenticated }}>
            {children}
        </AuthContext.Provider>
    );
};

```

---

### Pages

The `pages/` folder contains the main views that users navigate to. These pages are protected by the `AccessControl` component based on user roles.

- **`HomePage.js`**: Accessible by all users, including unauthenticated ones.
- **`EditorRole.js`**: Restricted to users with the `editor` role.
- **`AllRoles.js`**: Accessible to both `editor` and `viewer` roles.
- **`NoAccess.js`**: Shown when a user tries to access a page without proper roles.

---

### Routes

The `routes/` folder manages the application’s routing and defines which roles have access to specific pages.

#### **`RouteConfig.js`**
Defines the routes, components to render, and roles required to access them.

```js
import { lazy } from 'react';

// Centralized route configuration
const routes = [
  {
    name: 'Home',
    path: '/',
    component: lazy(() => import('../pages/HomePage.js')),
    roles: ['public'],
  },
  {
    name: 'Editor Role',
    path: '/editor-role',
    component: lazy(() => import('../pages/EditorRole.js')),
    roles: ['editor'],
  },
  {
    name: 'All Roles',
    path: '/all-roles',
    component: lazy(() => import('../pages/AllRoles.js')),
    roles: ['editor', 'viewer'],
  },
  {
    name: 'No Access',
    path: '/no-access',
    component: lazy(() => import('../pages/NoAccess.js')),
    roles: ['public'],
    hidden: true,
  },
];

export default routes;
```

#### **`AppRoutes.js`**
Applies role-based access control to routes using the `AccessControl` component. If a user does not have the required role to access a page `redirect={true}` makes sure that the user is redirected to the no access page.

```js
import React, { Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';
import routes from './RouteConfig.js';
import AccessControl from '../components/AccessControl.js';

const AppRoutes = () => {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        {routes.map((route) => (
            <Route
            key={route.path} 
            path={route.path} 
            element={
                <AccessControl requiredRoles={route.roles} redirect={true}>
                    <route.component />
                </AccessControl>
            } />
        ))}
      </Routes>
    </Suspense>
  );
};

export default AppRoutes;

```

---

### Services

The `services/` folder contains helper functions for interacting with Keycloak. The file `keycloak.js` includes initialization, login, logout, and role-checking logic.

#### **`keycloak.js`**

```js
import Keycloak from 'keycloak-js';

// Config from .env file
const keycloakConfig = {
    url: process.env.REACT_APP_KEYCLOAK_URL,
    realm: process.env.REACT_APP_KEYCLOAK_REALM,
    clientId: process.env.REACT_APP_KEYCLOAK_CLIENT_ID
};

// Create a new Keycloak instance
const keycloak = new Keycloak(keycloakConfig);

// Initialize Keycloak with onLoad check-sso (so that the user is not automatically prompted to the keycloak login page)
export const initKeycloak = async () => {
    try {
        keycloak.init({ onLoad: 'check-sso' });
    } catch (error) {
        console.error('Failed to initialize adapter:', error);
    }
}

// Other functions include loginKeycloak, logoutKeycloak, hasRole, and getUsernameAndRoles
```