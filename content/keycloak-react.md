Title: Role-Based Access Control (RBAC) with Keycloak in React Frontend
Date: 2024-10-06  
Category: web development
Tags: keycloak, rbac, react, keycloak-js  
Author: Leon Ramzews  
Summary: Securing a React frontend with Keycloak such that only users with certain roles are allowed to access pages or components in the application.

## Introduction

In this blog post, I’ll walk you through how I:

- Set up Keycloak, from installation to configuration,
- Embedded Keycloak into a simple React frontend so that users can log in and out via a button,
- Implemented role-based access control (RBAC), ensuring that certain pages or components in the frontend are only accessible to users with the appropriate roles.

A minimal working example is available here: [https://github.com/JulianRotti/keycloak-sample-project](https://github.com/JulianRotti/keycloak-sample-project). Choose the branch `frontend-integration` for both the main project `keycloak-sample-project` and the submodule `keycloak-react`.

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

When building a React application that integrates Keycloak for Role-Based Access Control (RBAC), I aimed to keep the folder structure as modular and organized as possible. Here’s a breakdown:

```plaintext
src/
├── components/
├── contexts/
├── pages/
├── routes/
├── services/
```

- **components/**: Contains UI components like a `Sidebar`. More importantly, it includes a reusable `AccessControl` component, which conditionally renders content based on the current user’s roles.
- **contexts/**: Contains the `AuthContext.js` file, which centralizes the authentication state using Keycloak. This provides the app with a global authentication context that makes the authentication state and role-checking functions available throughout the app.
- **pages/**: Includes the main views (pages) of the application. Some pages are restricted based on user roles, enforced using `AccessControl`.
- **routes/**: Defines the app’s routes and applies role-based access control (RBAC) for each page.
- **services/**: Contains helper functions that handle Keycloak initialization, login, logout, and role management.

---

### Keycloak Service

The first step was to create a set of service functions to manage Keycloak in the application. These functions are located in `services/keycloak.js`. To handle Keycloak interactions, I used the `keycloak-js` package:

```js
import Keycloak from 'keycloak-js';
```

#### Keycloak Configuration

The configuration for Keycloak is stored in an `.env` file in the root directory, which includes settings such as the Keycloak URL, realm, and client ID:

```js
// Config from .env file
const keycloakConfig = {
    url: process.env.REACT_APP_KEYCLOAK_URL,
    realm: process.env.REACT_APP_KEYCLOAK_REALM,
    clientId: process.env.REACT_APP_KEYCLOAK_CLIENT_ID
};

// Create a new Keycloak instance
const keycloak = new Keycloak(keycloakConfig);
```

#### Initializing Keycloak

I chose to initialize Keycloak with `onLoad: 'check-sso'`. This avoids automatically triggering a login when the app starts and allows the user to log in via a button when they choose to.

```js
// Initialize Keycloak without forcing a login page on app load
export const initKeycloak = async () => {
    try {
        await keycloak.init({ onLoad: 'check-sso' });
    } catch (error) {
        console.error('Failed to initialize Keycloak:', error);
    }
};
```

#### Login and Logout

These functions handle logging the user in and out of the app:

```js
// Log the user in via Keycloak
export const loginKeycloak = () => {
    keycloak.login();
};

// Log the user out via Keycloak
export const logoutKeycloak = () => {
    keycloak.logout({ redirectUri: `${window.location.origin}/` });
};
```

#### Checking Authentication Status

This function checks if the user is currently authenticated:

```js
// Check if the user is authenticated
export const checkKeycloakLogin = () => keycloak.authenticated;
```

#### Authentication Listener

To keep the app in sync with the authentication status, I set up listeners that react to authentication events. These update the state whenever the user logs in or logs out:

```js
// Set up listeners for login/logout events
export const authentificationListener = (setIsAuthenticated) => {
    keycloak.onAuthSuccess = () => setIsAuthenticated(true);
    keycloak.onAuthLogout = () => setIsAuthenticated(false);
};
```

#### Role and User Information

Keycloak provides token information containing the user’s roles and username. These helper functions allow us to extract that data:

```js
// Check if the user has a specific role
export const hasRole = (role) => {
    if (!keycloak.authenticated || !keycloak.tokenParsed.realm_access.roles) {
        console.error("Role check failed: Token or roles not available.");
        return false;
    }
    return keycloak.tokenParsed.realm_access.roles.includes(role);
};

// Get username and roles
export const getUsernameAndRoles = (givenRoles) => {
    return {
        username: keycloak.tokenParsed.preferred_username,
        roles: keycloak.tokenParsed.realm_access.roles.filter(role => givenRoles.includes(role))
    };
};
```

---

### Keycloak Context and Provider

The `contexts/` folder contains the `AuthContext.js` file, which manages the global authentication state. This centralized context ensures that authentication listeners are set up only once, preventing duplicated event handling and keeping the app in sync.

In `AuthContext.js`, the state is managed using `useState`, and the effect (`useEffect`) sets up the Keycloak initialization and authentication listeners.

#### `AuthContext.js`

Here’s how the authentication state is managed and shared globally:

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
        setIsAuthenticated(checkKeycloakLogin());
        authentificationListener(setIsAuthenticated);
        setAuthLoading(false); 
      };

      initializeKeycloak();
    }, []);

    if (authLoading) return <div>Loading...</div>;

    return (
        <AuthContext.Provider value={{ isAuthenticated }}>
            {children}
        </AuthContext.Provider>
    );
};
```

### Wrapping `App.js` in the Auth Provider

By wrapping the entire application inside the `AuthProvider` component, the authentication state becomes available across the app, including in any nested components.

```js
import React from 'react';
import { AuthProvider } from './contexts/AuthContext.js'; 

function App() {
  return (
    <AuthProvider>
        {/* Other components go here */}
    </AuthProvider>
  );
}

export default App;
```

### Entry Point

Finally, the app is rendered in `index.js`, which is the entry point for the application:

```js
import React from 'react';
import { createRoot } from 'react-dom/client';
import './index.css';
import App from './App.js';

const rootElement = document.getElementById('root');
const root = createRoot(rootElement);

root.render(<App />);
```

---

### Components

The final step is to provide components for:
- Handling access control  
- Displaying the login/logout button  
- Showing user information  

#### Access Control

To manage role-based access in the app, I implemented a reusable **Access Control** component, which acts as a Higher-Order Component (HOC). This component wraps other components, and based on the logged-in user’s roles, either displays or hides the wrapped content.

The `AccessControl` component ensures that only users with the appropriate roles can access certain parts of the UI. If the user lacks the required role, the component either hides the content or redirects them to a "No Access" page.

Here’s how the `AccessControl` component is structured in `components/AccessControl.js`:

### Inputs
- **requiredRoles**: The roles necessary to view the wrapped content.
- **children**: The component or content wrapped by `AccessControl` that you want to protect.
- **fallback**: Optional. A fallback component that is shown if access is denied.
- **redirect**: Optional. If `true`, users are redirected to a "No Access" page instead of seeing the fallback.

### Functionality
1. It fetches the `isAuthenticated` state from the `AuthContext`, ensuring the user’s authentication status is up to date.
2. It checks if `public` is included in the `requiredRoles`. If so, the content is accessible to everyone, including unauthenticated users.
3. It verifies if the authenticated user has one of the required roles using the `hasRole` function. If the user is not authenticated or lacks the necessary role:
   - If `redirect` is `true`, the user is sent to the "No Access" page.
   - Otherwise, the fallback content (if provided) is displayed.
4. If the user has the required role, the component renders the wrapped content.

Here’s the implementation:

```js
import React, { useContext } from 'react';
import { AuthContext } from '../contexts/AuthContext.js';
import { hasRole } from '../services/keycloak.js';
import { Navigate } from 'react-router-dom';

export default function AccessControl ({ requiredRoles, children, fallback = null, redirect = false }) {
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

This component can be used to protect entire pages or hide individual elements. For example, to hide a button:

```js
<AccessControl requiredRoles={['editor']}>
  <button>Edit Content</button>
</AccessControl>
```

With this approach, we ensure that sensitive content is hidden from unauthorized users while providing a fallback or redirection option.

#### Login/Logout Button

Since both the Login/Logout button and user information will be placed in the Sidebar, these components are stored in the `components/Sidebar/` folder.

To implement the `KeycloakButton` for logging in and out, we utilize the `login` and `logout` service functions from the Keycloak service, along with the authentication state from the context provider.

Here's how it works:

- The button fetches the `isAuthenticated` state from the `AuthContext`, which determines if the user is currently logged in.  
- A `handleButtonClick` function is defined, which checks the user's authentication state:
    - If the user is logged in, it calls the `logoutKeycloak` function to log them out.  
    - If the user is not logged in, it calls the `loginKeycloak` function to initiate the login process.  
- The button dynamically renders "Login" or "Logout" based on the user's authentication status and executes the appropriate action when clicked.  

```js
import React, { useContext } from "react";
import { loginKeycloak, logoutKeycloak } from "../../services/keycloak.js";
import { AuthContext } from "../../contexts/AuthContext.js";

export default function KeycloakButton() {
    const { isAuthenticated } = useContext(AuthContext);

    const handleButtonClick = () => {
        isAuthenticated ? logoutKeycloak() : loginKeycloak();
    };

    return (
        <button onClick={handleButtonClick}>
            { isAuthenticated ? "Logout" : "Login" }
        </button>
    );
}
```

#### Displaying User Information

In `components/Sidebar/UserInfo.js`, we define a component to display the current user's username and roles. This component leverages the `AuthContext` to access the authentication state and the `getUsernameAndRoles` helper function from the Keycloak service.

Here's how the component works:

- It first retrieves the `isAuthenticated` state from `AuthContext` to check if the user is logged in:
    - If the user is not authenticated, the component returns `null` and does not display anything.  
    - If the user is authenticated, it calls the `getUsernameAndRoles` function, which fetches the username and filters the user’s roles based on the provided role list (e.g., `editor`, `viewer`).  
- The component then displays the username and the user's roles, formatted as a comma-separated list.  

```js
import React, { useContext } from "react";
import { getUsernameAndRoles } from "../../services/keycloak.js";
import { AuthContext } from "../../contexts/AuthContext.js";

export default function UserInfo () {
    const { isAuthenticated } = useContext(AuthContext);

    if (!isAuthenticated) {
        return null;
    }

    const { username, roles } = getUsernameAndRoles(['editor', 'viewer']);

    return (
        <div>
            <p>{ `username: ${username}` }</p>
            <p>{ `roles: ${roles.join(", ")}` }</p>
        </div>
    );
}
```

Before we move on to integrating these components into a sidebar, let’s finalize the remaining pieces of the project.

---

### Pages

The `pages/` folder contains the main views that users

 navigate to. These pages are protected by the `AccessControl` component based on user roles:

- **`HomePage.js`**: Accessible to all users, including unauthenticated ones.
- **`EditorRole.js`**: Restricted to users with the `editor` role.
- **`AllRoles.js`**: Accessible to both `editor` and `viewer` roles.
- **`NoAccess.js`**: Shown when a user tries to access a page without the required roles.

---

### Routes

The `routes/` folder manages the app’s routing and defines the roles required to access each page.

#### Configuring Routes

In `routes/RouteConfig.js`, I defined the routes, their corresponding components, and the roles required for access. These roles are passed to the `AccessControl` component.

```js
import { lazy } from 'react';

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

### Using Keycloak Components

There are two main areas where RBAC is applied:

1. Routing: Protecting entire pages based on the user’s roles.
2. Sidebar: Displaying accessible links and login/logout buttons.

#### Routing

In `routes/AppRoutes.js`, role-based access control is applied to each route using the `AccessControl` component. This ensures that only users with the required roles can access specific pages. 

Here's how it works:

- The component maps through the routes defined in `RouteConfig.js`.  
- For each route, the `AccessControl` component is wrapped around the page's corresponding component, ensuring that access is restricted based on the roles specified for that route.  
- The `redirect={true}` prop is passed to `AccessControl`. If the user lacks the necessary roles, they are automatically redirected to a "No Access" page, preventing unauthorized access.  
- The component uses `Suspense` to handle dynamic imports, displaying a fallback loading state while the route components are being fetched.  

```js
import React, { Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';
import routes from './RouteConfig.js';
import AccessControl from '../components/AccessControl.js';

const AppRoutes = () => (
  <Suspense fallback={<div>Loading...</div>}>
    <Routes>
      {routes.map(route => (
        <Route
          key={route.path}
          path={route.path}
          element={
            <AccessControl requiredRoles={route.roles} redirect={true}>
              <route.component />
            </AccessControl>
          }
        />
      ))}
    </Routes>
  </Suspense>
);

export default AppRoutes;
```

#### Displaying Pages in a Sidebar

In `components/Sidebar/Sidebar.js`, we implement a simple sidebar that integrates the `UserInfo` and `KeycloakButton` components, alongside navigation links for available routes. The sidebar ensures role-based access control by wrapping each route link in the `AccessControl` component.

Here's how it works:

- The sidebar iterates over the routes defined in `RouteConfig.js`. For each route, it checks if the route should be visible (i.e., not hidden).  
- Each visible route is rendered as a link using `Link` from `react-router-dom`. However, before the link is displayed, it’s wrapped in the `AccessControl` component, which checks whether the user has the required roles to access the route.  
- The `KeycloakButton` is included for login/logout functionality, dynamically switching between "Login" and "Logout" depending on the user’s authentication state.  
- The `UserInfo` component displays the current user's username and roles, provided they are authenticated.  
- The sidebar is split into two sections: the navigation links, login/logout button, and user info are displayed on the left, while the main content (passed as `children`) is displayed on the right.  

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
      <div style={{ width: '20%' }}>
        {routes
          .filter(route => !route.hidden)
          .map(route => (
            <AccessControl key={route.path} requiredRoles={route.roles}>
              <p>
                <Link to={route.path}>{route.name}</Link>
              </p>
            </AccessControl>
          ))}
        <KeycloakButton />
        <UserInfo />
      </div>
      <div style={{ width: '80%' }}>
        {children}
      </div>
    </div>
  );
}
```

---

### Bringing the Pieces Together

In `App.js`, we bring everything together. The app is wrapped in the `AuthProvider`, and the routes and sidebar are loaded inside the `Router` component:

```js
import React from 'react';
import { BrowserRouter as Router } from 'react-router-dom';
import SimpleSidebar from './components/Sidebar/Sidebar.js';
import AppRoutes from './routes/AppRoutes.js';
import { AuthProvider } from './contexts/AuthContext.js';

function App() {
  return (
    <AuthProvider>
      <Router>
        <SimpleSidebar>
          <AppRoutes />
        </SimpleSidebar>
      </Router>
    </AuthProvider>
  );
}

export default App;
```## React Frontend

### React Frontend Structure with Keycloak RBAC

When building a React application that integrates Keycloak for Role-Based Access Control (RBAC), I aimed to keep the folder structure as modular and organized as possible. Here’s a breakdown:

```plaintext
src/
├── components/
├── contexts/
├── pages/
├── routes/
├── services/
```

- **components/**: Contains UI components like a `Sidebar`. More importantly, it includes a reusable `AccessControl` component, which conditionally renders content based on the current user’s roles.
- **contexts/**: Contains the `AuthContext.js` file, which centralizes the authentication state using Keycloak. This provides the app with a global authentication context that makes the authentication state and role-checking functions available throughout the app.
- **pages/**: Includes the main views (pages) of the application. Some pages are restricted based on user roles, enforced using `AccessControl`.
- **routes/**: Defines the app’s routes and applies role-based access control (RBAC) for each page.
- **services/**: Contains helper functions that handle Keycloak initialization, login, logout, and role management.

---

### Keycloak Service

The first step was to create a set of service functions to manage Keycloak in the application. These functions are located in `services/keycloak.js`. To handle Keycloak interactions, I used the `keycloak-js` package:

```js
import Keycloak from 'keycloak-js';
```

#### Keycloak Configuration

The configuration for Keycloak is stored in an `.env` file in the root directory, which includes settings such as the Keycloak URL, realm, and client ID:

```js
// Config from .env file
const keycloakConfig = {
    url: process.env.REACT_APP_KEYCLOAK_URL,
    realm: process.env.REACT_APP_KEYCLOAK_REALM,
    clientId: process.env.REACT_APP_KEYCLOAK_CLIENT_ID
};

// Create a new Keycloak instance
const keycloak = new Keycloak(keycloakConfig);
```

#### Initializing Keycloak

I chose to initialize Keycloak with `onLoad: 'check-sso'`. This avoids automatically triggering a login when the app starts and allows the user to log in via a button when they choose to.

```js
// Initialize Keycloak without forcing a login page on app load
export const initKeycloak = async () => {
    try {
        await keycloak.init({ onLoad: 'check-sso' });
    } catch (error) {
        console.error('Failed to initialize Keycloak:', error);
    }
};
```

#### Login and Logout

These functions handle logging the user in and out of the app:

```js
// Log the user in via Keycloak
export const loginKeycloak = () => {
    keycloak.login();
};

// Log the user out via Keycloak
export const logoutKeycloak = () => {
    keycloak.logout({ redirectUri: `${window.location.origin}/` });
};
```

#### Checking Authentication Status

This function checks if the user is currently authenticated:

```js
// Check if the user is authenticated
export const checkKeycloakLogin = () => keycloak.authenticated;
```

#### Authentication Listener

To keep the app in sync with the authentication status, I set up listeners that react to authentication events. These update the state whenever the user logs in or logs out:

```js
// Set up listeners for login/logout events
export const authentificationListener = (setIsAuthenticated) => {
    keycloak.onAuthSuccess = () => setIsAuthenticated(true);
    keycloak.onAuthLogout = () => setIsAuthenticated(false);
};
```

#### Role and User Information

Keycloak provides token information containing the user’s roles and username. These helper functions allow us to extract that data:

```js
// Check if the user has a specific role
export const hasRole = (role) => {
    if (!keycloak.authenticated || !keycloak.tokenParsed.realm_access.roles) {
        console.error("Role check failed: Token or roles not available.");
        return false;
    }
    return keycloak.tokenParsed.realm_access.roles.includes(role);
};

// Get username and roles
export const getUsernameAndRoles = (givenRoles) => {
    return {
        username: keycloak.tokenParsed.preferred_username,
        roles: keycloak.tokenParsed.realm_access.roles.filter(role => givenRoles.includes(role))
    };
};
```

---

### Keycloak Context and Provider

The `contexts/` folder contains the `AuthContext.js` file, which manages the global authentication state. This centralized context ensures that authentication listeners are set up only once, preventing duplicated event handling and keeping the app in sync.

In `AuthContext.js`, the state is managed using `useState`, and the effect (`useEffect`) sets up the Keycloak initialization and authentication listeners.

#### `AuthContext.js`

Here’s how the authentication state is managed and shared globally:

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
        setIsAuthenticated(checkKeycloakLogin());
        authentificationListener(setIsAuthenticated);
        setAuthLoading(false); 
      };

      initializeKeycloak();
    }, []);

    if (authLoading) return <div>Loading...</div>;

    return (
        <AuthContext.Provider value={{ isAuthenticated }}>
            {children}
        </AuthContext.Provider>
    );
};
```

### Wrapping `App.js` in the Auth Provider

By wrapping the entire application inside the `AuthProvider` component, the authentication state becomes available across the app, including in any nested components.

```js
import React from 'react';
import { AuthProvider } from './contexts/AuthContext.js'; 

function App() {
  return (
    <AuthProvider>
        {/* Other components go here */}
    </AuthProvider>
  );
}

export default App;
```

### Entry Point

Finally, the app is rendered in `index.js`, which is the entry point for the application:

```js
import React from 'react';
import { createRoot } from 'react-dom/client';
import './index.css';
import App from './App.js';

const rootElement = document.getElementById('root');
const root = createRoot(rootElement);

root.render(<App />);
```

---

### Components

The final step is to provide components for:

- Handling access control  
- Displaying the login/logout button  
- Showing user information  


#### Access Control

To manage role-based access in the app, I implemented a reusable **Access Control** component, which acts as a Higher-Order Component (HOC). This component wraps other components, and based on the logged-in user’s roles, either displays or hides the wrapped content.

The `AccessControl` component ensures that only users with the appropriate roles can access certain parts of the UI. If the user lacks the required role, the component either hides the content or redirects them to a "No Access" page.

Here’s how the `AccessControl` component is structured in `components/AccessControl.js`:

##### Inputs
- **requiredRoles**: The roles necessary to view the wrapped content.
- **children**: The component or content wrapped by `AccessControl` that you want to protect.
- **fallback**: Optional. A fallback component that is shown if access is denied.
- **redirect**: Optional. If `true`, users are redirected to a "No Access" page instead of seeing the fallback.

##### Functionality
1. It fetches the `isAuthenticated` state from the `AuthContext`, ensuring the user’s authentication status is up to date.
2. It checks if `public` is included in the `requiredRoles`. If so, the content is accessible to everyone, including unauthenticated users.
3. It verifies if the authenticated user has one of the required roles using the `hasRole` function. If the user is not authenticated or lacks the necessary role:
   - If `redirect` is `true`, the user is sent to the "No Access" page.
   - Otherwise, the fallback content (if provided) is displayed.
4. If the user has the required role, the component renders the wrapped content.

Here’s the implementation:

```js
import React, { useContext } from 'react';
import { AuthContext } from '../contexts/AuthContext.js';
import { hasRole } from '../services/keycloak.js';
import { Navigate } from 'react-router-dom';

export default function AccessControl ({ requiredRoles, children, fallback = null, redirect = false }) {
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

This component can be used to protect entire pages or hide individual elements. For example, to hide a button:

```js
<AccessControl requiredRoles={['editor']}>
  <button>Edit Content</button>
</AccessControl>
```

With this approach, we ensure that sensitive content is hidden from unauthorized users while providing a fallback or redirection option.

#### Login/Logout Button

Since both the Login/Logout button and user information will be placed in the Sidebar, these components are stored in the `components/Sidebar/` folder.

To implement the `KeycloakButton` for logging in and out, we utilize the `login` and `logout` service functions from the Keycloak service, along with the authentication state from the context provider.

Here's how it works:

- The button fetches the `isAuthenticated` state from the `AuthContext`, which determines if the user is currently logged in.  
- A `handleButtonClick` function is defined, which checks the user's authentication state:
    - If the user is logged in, it calls the `logoutKeycloak` function to log them out.  
    - If the user is not logged in, it calls the `loginKeycloak` function to initiate the login process.  
- The button dynamically renders "Login" or "Logout" based on the user's authentication status and executes the appropriate action when clicked.  

```js
import React, { useContext } from "react";
import { loginKeycloak, logoutKeycloak } from "../../services/keycloak.js";
import { AuthContext } from "../../contexts/AuthContext.js";

export default function KeycloakButton() {
    const { isAuthenticated } = useContext(AuthContext);

    const handleButtonClick = () => {
        isAuthenticated ? logoutKeycloak() : loginKeycloak();
    };

    return (
        <button onClick={handleButtonClick}>
            { isAuthenticated ? "Logout" : "Login" }
        </button>
    );
}
```

#### Displaying User Information

In `components/Sidebar/UserInfo.js`, we define a component to display the current user's username and roles. This component leverages the `AuthContext` to access the authentication state and the `getUsernameAndRoles` helper function from the Keycloak service.

Here's how the component works:

- It first retrieves the `isAuthenticated` state from `AuthContext` to check if the user is logged in:
    - If the user is not authenticated, the component returns `null` and does not display anything.  
    - If the user is authenticated, it calls the `getUsernameAndRoles` function, which fetches the username and filters the user’s roles based on the provided role list (e.g., `editor`, `viewer`).  
- The component then displays the username and the user's roles, formatted as a comma-separated list.  

```js
import React, { useContext } from "react";
import { getUsernameAndRoles } from "../../services/keycloak.js";
import { AuthContext } from "../../contexts/AuthContext.js";

export default function UserInfo () {
    const { isAuthenticated } = useContext(AuthContext);

    if (!isAuthenticated) {
        return null;
    }

    const { username, roles } = getUsernameAndRoles(['editor', 'viewer']);

    return (
        <div>
            <p>{ `username: ${username}` }</p>
            <p>{ `roles: ${roles.join(", ")}` }</p>
        </div>
    );
}
```

Before we move on to integrating these components into a sidebar, let’s finalize the remaining pieces of the project.

---

### Pages

The `pages/` folder contains the main views that users navigate to. These pages are protected by the `AccessControl` component based on user roles:

- **`HomePage.js`**: Accessible to all users, including unauthenticated ones.
- **`EditorRole.js`**: Restricted to users with the `editor` role.
- **`AllRoles.js`**: Accessible to both `editor` and `viewer` roles.
- **`NoAccess.js`**: Shown when a user tries to access a page without the required roles.

---

### Routes

The `routes/` folder manages the app’s routing and defines the roles required to access each page.

#### Configuring Routes

In `routes/RouteConfig.js`, I defined the routes, their corresponding components, and the roles required for access. These roles are passed to the `AccessControl` component.

```js
import { lazy } from 'react';

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

### Using Keycloak Components

There are two main areas where RBAC is applied:

1. Routing: Protecting entire pages based on the user’s roles  
2. Sidebar: Displaying accessible links and login/logout buttons  

#### Routing

In `routes/AppRoutes.js`, role-based access control is applied to each route using the `AccessControl` component. This ensures that only users with the required roles can access specific pages. 

Here's how it works:

- The component maps through the routes defined in `RouteConfig.js`.  
- For each route, the `AccessControl` component is wrapped around the page's corresponding component, ensuring that access is restricted based on the roles specified for that route.  
- The `redirect={true}` prop is passed to `AccessControl`. If the user lacks the necessary roles, they are automatically redirected to a "No Access" page, preventing unauthorized access.  
- The component uses `Suspense` to handle dynamic imports, displaying a fallback loading state while the route components are being fetched.  

```js
import React, { Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';
import routes from './RouteConfig.js';
import AccessControl from '../components/AccessControl.js';

const AppRoutes = () => (
  <Suspense fallback={<div>Loading...</div>}>
    <Routes>
      {routes.map(route => (
        <Route
          key={route.path}
          path={route.path}
          element={
            <AccessControl requiredRoles={route.roles} redirect={true}>
              <route.component />
            </AccessControl>
          }
        />
      ))}
    </Routes>
  </Suspense>
);

export default AppRoutes;
```

#### Displaying Pages in a Sidebar

In `components/Sidebar/Sidebar.js`, we implement a simple sidebar that integrates the `UserInfo` and `KeycloakButton` components, alongside navigation links for available routes. The sidebar ensures role-based access control by wrapping each route link in the `AccessControl` component.

Here's how it works:

- The sidebar iterates over the routes defined in `RouteConfig.js`. For each route, it checks if the route should be visible (i.e., not hidden).  
- Each visible route is rendered as a link using `Link` from `react-router-dom`. However, before the link is displayed, it’s wrapped in the `AccessControl` component, which checks whether the user has the required roles to access the route.  
- The `KeycloakButton` is included for login/logout functionality, dynamically switching between "Login" and "Logout" depending on the user’s authentication state.  
- The `UserInfo` component displays the current user's username and roles, provided they are authenticated.  
- The sidebar is split into two sections: the navigation links, login/logout button, and user info are displayed on the left, while the main content (passed as `children`) is displayed on the right.  

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
      <div style={{ width: '20%' }}>
        {routes
          .filter(route => !route.hidden)
          .map(route => (
            <AccessControl key={route.path} requiredRoles={route.roles}>
              <p>
                <Link to={route.path}>{route.name}</Link>
              </p>
            </AccessControl>
          ))}
        <KeycloakButton />
        <UserInfo />
      </div>
      <div style={{ width: '80%' }}>
        {children}
      </div>
    </div>
  );
}
```

---

### Bringing the Pieces Together

In `App.js`, we bring everything together. The app is wrapped in the `AuthProvider`, and the routes and sidebar are loaded inside the `Router` component:

```js
import React from 'react';
import { BrowserRouter as Router } from 'react-router-dom';
import SimpleSidebar from './components/Sidebar/Sidebar.js';
import AppRoutes from './routes/AppRoutes.js';
import { AuthProvider } from './contexts/AuthContext.js';

function App() {
  return (
    <AuthProvider>
      <Router>
        <SimpleSidebar>
          <AppRoutes />
        </SimpleSidebar>
      </Router>
    </AuthProvider>
  );
}

export default App;
```






