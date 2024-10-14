Title: Role-Based Access Control (RBAC) with Keycloak in Node Backend  
Date: 2024-10-14  
Category: web development  
Tags: keycloak, rbac, node, keycloak-connect  
Author: Leon Ramzews  
Summary: Securing a Node.js backend with Keycloak such that only users with certain roles are allowed to make API calls.


## Introduction

In this blog post, I’ll walk you through how to:

- Set up Keycloak, from installation to configuration,  
- Secure a simple Node.js backend by protecting API endpoints using Keycloak authentication,  
- Implement role-based access control (RBAC), ensuring only users with appropriate roles can access certain API routes,  
- Create simple frontend buttons that pass the Keycloak token to the backend based on the currently logged-in user.  

A minimal working example is available here: [https://github.com/JulianRotti/keycloak-sample-project](https://github.com/JulianRotti/keycloak-sample-project). Make sure to choose the `backend-integration` branch for both the main project `keycloak-sample-project` and its submodules `keycloak-react` and `keycloak-node`.

---

## Setting Up Keycloak

### Installing Keycloak with Docker

To get Keycloak running quickly, I set it up using Docker. Below is the `docker-compose.yml` file I used to manage the Keycloak instance:

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
- **Image**: I used the `25.0.6` version of the Keycloak image from quay.io.
- **Admin Credentials**: Set to `admin/admin` for quick local setup.
- **Volumes**: Persistent data storage for Keycloak is handled using Docker volumes.

After setting this up, I ran `docker-compose up` to make Keycloak accessible at [http://localhost:8080](http://localhost:8080).

### Configuring Keycloak

1. **Creating a Realm**: I created a new realm called `my-website`—an isolated space to manage users, roles, and clients.
2. **Defining Roles**: 
    - `viewer`  
    - `editor`  
3. **Creating Users**:
    - `test_viewer` (assigned the `viewer` role)  
    - `test_editor` (assigned **both** the `editor` and `viewer` roles for easier testing and role management)  
4. **Setting Up Clients**:
    - **Frontend Client**: 
        - Client ID: `my-website-frontend`  
        - Root URL: `http://localhost:3000`  
        - Web Origins: `http://localhost:3000`  
        - Redirect URIs: `http://localhost:3000/*`  
    - **Backend Client**:
        - Client ID: `my-website-backend`  
        - Turned **Client Authentication** on for secure token verification.  

---

## Node Backend

### Node Backend Structure with Keycloak RBAC

For the backend, I structured the project using best practices by organizing the code into specific layers. This allows for cleaner and more maintainable code. Here's a breakdown of the folder structure:

```plaintext
src/
├── models/
├── services/
├── middleware/
├── controllers/
├── routes/
```

- **models/**: Contains data models for our backend. In this case, since we're using mock data, I used classes and singleton patterns to simulate database behavior.  
- **services/**: Handles business logic and mock data interaction. This layer is responsible for manipulating the data provided by the models.  
- **middleware/**: Contains middleware for request handling, including Keycloak authentication.  
- **controllers/**: Responsible for handling the logic related to specific API routes (e.g., fetching or posting data).  
- **routes/**: Defines the application's API endpoints and connects them to the appropriate controller and middleware functions.  

---

### Keycloak Middleware

The next step was setting up Keycloak middleware for authentication and RBAC. I used the `keycloak-connect` package for this. The middleware is configured in `middleware/authMiddleware.js` and is responsible for initializing Keycloak and protecting routes.

To set up Keycloak interactions, I started by importing the `keycloak-connect` package:

```js
import Keycloak from 'keycloak-connect';
```

#### Keycloak Configuration

The configuration for Keycloak is stored in an `.env` file in the root directory. This allows for easy customization of the Keycloak URL, realm, and client ID.

```js
// Config from .env file
const keycloakConfig = {
    url: process.env.NODE_APP_KEYCLOAK_URL,
    realm: process.env.NODE_APP_KEYCLOAK_REALM,
    clientId: process.env.NODE_APP_KEYCLOAK_CLIENT_ID
};

// Create a new Keycloak instance
const keycloak = new Keycloak(keycloakConfig);
```

#### Initializing Keycloak

Since the backend receives tokens from the frontend, the Keycloak instance is initialized with `bearer-only: true`. This mode ensures that users authenticate using tokens, and the backend doesn’t need to manage sessions.

```js
export const keycloak = new Keycloak({}, {
    'realm': process.env.NODE_APP_KEYCLOAK_REALM,
    'auth-server-url': `${process.env.NODE_APP_KEYCLOAK_URL}`,
    'ssl-required': 'external',
    'resource': process.env.NODE_APP_KEYCLOAK_CLIENT_ID,
    'confidential-port': 0,
    'bearer-only': true
});
```

### Protecting Routes with Roles

For role-based access control (RBAC), I created a helper function `protectWithRole` to protect routes based on user roles. This function abstracts the underlying Keycloak logic, making it easier to replace or modify the authentication provider in the future. By using `protectWithRole`, we only need to make changes in the middleware file if we decide to switch from Keycloak to another provider, rather than updating every route.

```js
// Check if user has a specific role
export const protectWithRole = (role) => {
    return keycloak.protect(`realm:${role}`);
};
```

This function ensures that only users with the appropriate roles can access certain routes, enhancing security in your Node backend.

---

### Using Keycloak in the Backend

#### Initializing Keycloak in `index.js`

To apply Keycloak globally, we need to initialize the middleware in our `index.js`. This ensures that all API routes are protected by Keycloak.

```js
import express from 'express';
import { keycloak } from './middleware/authMiddleware.js';

const app = express();

// Initialize Keycloak globally
app.use(keycloak.middleware());

// Start the server
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

### Protecting API Routes with RBAC

In the `routes/AppRoutes.js` file, we define our API routes and apply role-based protection. In this example, only users with the `viewer` role can access the `get-data` endpoint.

```js
import express from 'express';
import { protectWithRole } from '../middleware/authMiddleware.js';
// Add your controller import here (e.g., appController)

const router = express.Router();

router.get('/get-data/:id', protectWithRole('viewer'), appController.getDataById);

export default router;
```

In this setup, if a user tries to access the `/get-data/:id` route without the appropriate role (`viewer` in this case), they will receive an "Access Denied" error.

> **Note**: This is all we need to do in order to integrate Keycloak into the backend for role-based access control. To test this, I will now complete the remaining code to provide the models, services, controllers, and routes, which can be used to fetch data and test role-based protection in practice.

### Sample Node Backend

The following sections describe the layered structure of the Node backend, using role-based access control (RBAC) with Keycloak. Each layer of the application is structured to maintain a clean separation of concerns and ensure scalability, maintainability, and security.

#### Models

The **models** layer mimics a database model structure, similar to how you would use something like Sequelize for interacting with a SQL database. In this example, we mock the behavior of a database with in-memory data to simulate how models would interact with an actual database.

Here's how it works:

- `MockDataModel`: Represents a basic data model with `id` and `name` properties.  
- `DataBaseModel`: Mimics the behavior of a database with methods to create and retrieve data. It implements the **singleton pattern** to ensure only one instance of the database exists.  

```js
// src/models/MockDataModel.js

export class MockDataModel {
  constructor(id, name) {
    this.id = id;
    this.name = name;
  }
}

// mimic the behaviour of sequelize
export class DataBaseModel {
    // do not create a new instance if there is already one (singleton pattern)
    constructor(mockDataStore = []) {
        if (!DataBaseModel.instance) {
          this.mockDataStore = mockDataStore;
          DataBaseModel.instance = this;
        }
        return DataBaseModel.instance;
      }

    create(name) {
        const maxId = Math.max(...this.mockDataStore.map(item => item.id));
        const id = maxId + 1;
        const newData = new MockDataModel(id, name);
        this.mockDataStore.push(newData);
        return newData;
    }

    getById(id) {
        return this.mockDataStore.find(item => item.id === id) || null;
    }

    findAll() {
        return this.mockDataStore;
    }
}
```

#### Services

The **services** layer handles the direct interaction with the data. In a real-world scenario, this layer would communicate with a database, making queries and returning the results. In our case, we define mock data and simulate these interactions by manipulating the in-memory data.

Here's how it works:

- `getDataById`: Takes an ID, parses it, and returns the corresponding data from the mock database.  
- `getAllData`: Retrieves all the mock data.  
- `createData`: Adds new data to the mock database.  

```js
// src/services/MockDataService.js

import { MockDataModel, DataBaseModel } from "../models/MockDataModel.js";

const mockData = [
    new MockDataModel(1, 'Mario'),
    new MockDataModel(2, 'Wario'),
    new MockDataModel(3, 'Peach'),
    new MockDataModel(4, 'Luigi')
];

const dataBase = new DataBaseModel(mockData);

export const getDataById = async (id) => {
    const parsedId = parseInt(id, 10);
    return await dataBase.getById(parsedId);
}

export const getAllData = async () => {
    return await dataBase.findAll();
}

export const createData = async (name) => {
    return dataBase.create(name);
}
```

#### Controllers

The **controllers** layer defines the logic for handling API requests. Each controller function corresponds to an API endpoint and is responsible for receiving requests, interacting with services, and returning appropriate responses. Controllers act as the intermediary between the routes and services.

Here's how it works:

- `getDataById`: Handles GET requests for retrieving specific data by ID.  
- `getAllData`: Handles GET requests to return all available data.  
- `postData`: Handles POST requests for adding new data.  

```js
// src/controllers/AppController.js

import * as mockDataService from "../services/MockDataService.js";

export const getDataById = async (req, res) => {
    try {
        const { id } = req.params;
        const data = await mockDataService.getDataById(id);
        if (data) {
            res.status(200).json(data);
        } else {
            res.status(404).json({ error: 'Data not found.' });
        }
    } catch (error) {
        res.status(500).json({ error: `Failed to get data by id: ${error.message}` });
    }
};

export const getAllData = async (req, res) => {
    try {
        const allData = await mockDataService.getAllData();
        res.status(200).json(allData);
    } catch (error) {
        res.status(500).json({ error: `Failed to get all data: ${error.message}` });
    }
};

export const postData = async (req, res) => {
    try {
        const { name } = req.body;
        const newData = await mockDataService.createData(name);
        res.status(201).json(newData);
    } catch (error) {
        res.status(500).json({ error: `Failed to create data: ${error.message}` });
    }
};
```

#### Routes

The **routes** layer combines all the pieces together, defining how the application's endpoints are structured and which roles are allowed to access them. This is where role-based access control (RBAC) is enforced using Keycloak.

Here's how it works:

- The routes use `protectWithRole` to restrict access based on user roles. Only users with the appropriate roles (e.g., `viewer`, `editor`) can access specific endpoints.  

```js
// src/routes/AppRoutes.js

import express from 'express';
import * as appController from '../controllers/AppController.js';
import { protectWithRole } from '../middleware/authMiddleware.js';

const router = express.Router();

router.get('/get-data/:id', protectWithRole('viewer'), appController.getDataById);
router.get('/get-data', protectWithRole('viewer'), appController.getAllData);
router.post('/post-data', protectWithRole('editor'), appController.postData);

export default router;
```

#### Index.js

In the **index.js** file, we initialize the express server and set up middleware for Keycloak and CORS. **CORS** is important for handling cross-origin requests from the frontend, especially when the frontend and backend are hosted on different ports or domains.

Here's how it works:
- **Keycloak middleware** is initialized globally, enabling the entire app to be protected by Keycloak RBAC.
- **CORS** is enabled to allow requests from the React frontend hosted on `http://localhost:3000`.

```js
// src/index.js

import express from 'express';
import AppRoutes from './routes/AppRoutes.js';
import cors from 'cors';  // Import CORS middleware
import { keycloak } from './middleware/authMiddleware.js';

const app = express();

// initialize Keycloak globally
app.use(keycloak.middleware());

// Middleware
app.use(express.json());

// Enable CORS for requests from http://localhost:3000 (React frontend)
app.use(cors({
    origin: 'http://localhost:3000',  // Allow only your frontend
}));

// Routes
app.use('/api', AppRoutes);

// Start the server
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

### Testing with Curl

To test the role-based access control (RBAC) protected API, you first need to obtain an access token from Keycloak and use it to make requests to your API. For this, Keycloak and the Node.js backend should be up and running via `npm start`.

#### Step 1: Get Access Token

To retrieve an access token for the user `test_editor`, run the following `curl` command. This will authenticate the user and return the access token, which you can use to make authorized API requests.

```bash
curl -X POST 'http://localhost:8080/realms/my-website/protocol/openid-connect/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'client_id=my-website-frontend' \
  -d 'grant_type=password' \
  -d 'username=test_editor' \
  -d 'password=test'
```

This command returns a JSON response containing the access token (`access_token`). You will use this token to authenticate requests to the Node.js API.

#### Step 2: Make Curl Requests to the API

Now, use the obtained access token to make requests to the API.

1. **Get All Data** (requires the `viewer` role):

```bash
curl -X GET 'http://localhost:5000/api/get-data' \
  -H 'Authorization: Bearer <ACCESS_TOKEN>'
```

2. **Get Data by ID** (requires the `viewer` role):

```bash
curl -X GET 'http://localhost:5000/api/get-data/1' \
  -H 'Authorization: Bearer <ACCESS_TOKEN>'
```

3. **Post New Data** (requires the `editor` role):

```bash
curl -X POST 'http://localhost:5000/api/post-data' \
  -H 'Authorization: Bearer <ACCESS_TOKEN>' \
  -H 'Content-Type: application/json' \
  -d '{"name": "Yoshi"}'
```

Replace `<ACCESS_TOKEN>` with the token you obtained in Step 1. Each request is authenticated, and depending on the role assigned to the user, access to the endpoints will either succeed or fail.

---

### Integration with a React Frontend

In the next step, I integrated the React frontend with the backend from my previous blog post. The idea is to use Keycloak for both frontend and backend RBAC, ensuring secure data access for users based on their roles.

To achieve this, I added:

- **services/api.js**: Provides functions to call the backend API and pass the Keycloak token of the logged-in user.  
- **components/ApiButtons.js**: Buttons that call these API functions when clicked.  
- **pages/HomePage.js**: A page where these buttons are available, making RBAC testing easier.  

This setup (along with the already implemented login/logout button) offers a simple way to test RBAC functionality from frontend to backend.

#### Services

### `keycloak.js`

In this file, I added a utility function, `getAccessToken`, to retrieve the Keycloak access token for the currently authenticated user. This token is critical when making secure API calls from the frontend to the backend. It ensures that the requests are properly authenticated and authorized.

Here's how it works:

- `getAccessToken`: This function checks if the user is authenticated with Keycloak. If the user is authenticated, it retrieves and returns the access token. This token is then used in the `Api.js` service functions (e.g., `getAllData`, `postData`) to authorize API calls to the backend.  

```js
// src/services/keycloak.js

// Function to get the access token of the current authenticated user
export const getAccessToken = () => {
    if (keycloak.authenticated) {
        return keycloak.token; // Return the access token
    } else {
        console.error('User is not authenticated.');
        return null; // Or handle it according to your needs
    }
};
```

By calling `getAccessToken` inside the button components (e.g., `GetAllDataButton`), we can ensure that only authenticated users are able to make authorized requests to the backend. If the user is not authenticated, the function will return `null`, and an error message can be displayed accordingly.

##### `api.js`

The `services/api.js` file contains functions for making HTTP requests to the backend API with the user's access token.

Here's how it works:

- `getAllData`: Fetches all data from the backend.  
- `getDataById`: Fetches data by ID.  
- `postData`: Sends new data to the backend.  

```js
// src/services/api.js

import axios from 'axios';

// Base URL for the API
const BASE_URL = 'http://localhost:5000/api'; // Adjust this if necessary

// Function to get all data
export const getAllData = async (accessToken) => {
    try {
        const response = await axios.get(`${BASE_URL}/get-data`, {
            headers: {
                Authorization: `Bearer ${accessToken}`,
            },
        });
        return response.data;
    } catch (error) {
        console.error('Error fetching all data:', error);
        throw error;
    }
};

// Function to get data by ID
export const getDataById = async (id, accessToken) => {
    try {
        const response = await axios.get(`${BASE_URL}/get-data/${id}`, {
            headers: {
                Authorization: `Bearer ${accessToken}`,
            },
        });
        return response.data;
    } catch (error) {
        console.error(`Error fetching data by ID ${id}:`, error);
        throw error; 
    }
};

// Function to post new data
export const postData = async (data, accessToken) => {
    try {
        const response = await axios.post(`${BASE_URL}/post-data`, data, {
            headers: {
                Authorization: `Bearer ${accessToken}`,
            },
        });
        return response.data;
    } catch (error) {
        console.error('Error posting data:', error);
        throw error; 
    }
};
```

#### Components

In the `components/ApiButtons.js` file, buttons are defined to trigger API requests from the frontend when clicked. These buttons use the `getAccessToken` function to retrieve the current user's access token, which is passed to the backend to authenticate and authorize the request. The token is verified on the backend to ensure that the user has the appropriate permissions (based on roles) to access the data.

Here's how it works:

- `GetAllDataButton`: Retrieves the access token using `getAccessToken`, calls the `getAllData` function with the token, and displays the data returned by the backend.  
- `GetDataByIdButton`: Uses the access token to call the `getDataById` function with a specific ID and displays the corresponding data.  
- `PostDataButton`: Sends new data to the backend using the `postData` function, along with the access token to authorize the request.  

This approach ensures that only authenticated users with the proper roles can interact with these API endpoints.

```js
// src/components/ApiButtons.js

import React, { useContext } from 'react';
import { getAccessToken } from '../services/keycloak.js';
import { AuthContext } from '../contexts/AuthContext.js';
import { getAllData, getDataById, postData } from '../services/api.js';

export function GetAllDataButton() {
    const { isAuthenticated } = useContext(AuthContext);

    const handleButtonClick = async () => {  
        if (isAuthenticated) {
            const accessToken = getAccessToken();
            try {
                const data = await getAllData(accessToken);
                alert(JSON.stringify(data));  // Use JSON.stringify for better alert visibility
            } catch (error) {
                alert("Error fetching data: " + error.message);
            }
        } else {
            alert("You need to be logged in to access this data.");
        }
    }

    return (
        <div>
            <button onClick={handleButtonClick}>
                Get all data
            </button>
        </div>
    );
}

export function GetDataByIdButton() {
    const { isAuthenticated } = useContext(AuthContext);
    const id = 1;  // Example ID
    const handleButtonClick = async () => {  
        if (isAuthenticated) {
            const accessToken = getAccessToken();
            try {
                const data = await getDataById(id, accessToken);
                alert(JSON.stringify(data));
            } catch (error) {
                alert("Error fetching data: " + error.message);
            }
        } else {
            alert("You need to be logged in to access this data.");
        }
    }

    return (
        <div>
            <button onClick={handleButtonClick}>
                Get data with id {id}
            </button>
        </div>
    );
}

export function PostDataButton() {
    const { isAuthenticated } = useContext(AuthContext);
    const newData = { "name": "Heiko" }; 

    const handleButtonClick = async () => {  
        if (isAuthenticated) {
            const accessToken = getAccessToken();
            try {
                const data = await postData(newData, accessToken);
                alert("Data posted successfully!");
            } catch (error) {
                alert("Error posting data: " + error.message);
            }
        } else {
            alert("You need to be logged in to post data.");
        }
    }

    return (
        <div>
            <button onClick={handleButtonClick}>
                Post data with name {newData.name}
            </button>
        </div>
    );
}
```

#### Pages

The `pages/HomePage.js` file contains the homepage with buttons to interact with the backend and test RBAC.

Here's how it works:

- All the buttons are rendered on this page.  
- Users can log in and click the buttons to see how RBAC works based on their roles.  

```js
// src/pages/HomePage.js

import React from 'react';
import { GetAllDataButton, GetDataByIdButton, PostDataButton } from '../components/ApiButtons.js';

const HomePage = () => {
  return (
    <div>
      <h1>This page is accessible for everyone (including not authenticated users)</h1>
      <GetAllDataButton />
      <GetDataByIdButton />
      <PostDataButton />
    </div>
  );
};

export default HomePage;
```

#### Testing the Integration

To test the integration of Keycloak, backend, and frontend:

1. Ensure all services (Keycloak, backend, and frontend) are running via `npm start`.  
2. Log in as a user in the frontend (e.g., `test_editor`), and try accessing the different buttons.  
3. The buttons will perform API calls to the backend with the Keycloak access token, and the responses will depend on the user's roles.
   - Users with the `viewer` role can get data.  
   - Users with the `editor` role can post data.  
