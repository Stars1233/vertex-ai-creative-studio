# Developer's Guide

Welcome to the GenMedia Creative Studio application! This guide provides an overview of the application's architecture and a step-by-step tutorial for adding new pages. Its purpose is to help you understand the project's structure and contribute effectively.

## Application Architecture

This application is built using Python with the [Mesop](https://mesop-dev.github.io/mesop/) UI framework and a [FastAPI](https://fastapi.tiangolo.com/) backend. The project is structured to enforce a clear separation of concerns, making the codebase easier to navigate, maintain, and extend.

Here is a breakdown of the key directories and their roles:

*   **`main.py`**: This is the main entry point of the application. It is responsible for:
    *   Initializing the FastAPI server.
    *   Mounting the Mesop application as a WSGI middleware.
    *   Handling root-level routing for authentication and redirects.
    *   Applying global middleware, such as the Content Security Policy (CSP).

*   **`pages/`**: This directory contains the top-level UI code for each distinct page in the application (e.g., `/home`, `/imagen`, `/veo`). Each file in this directory typically defines a function that builds the UI for a specific page using Mesop components.

*   **`components/`**: This directory holds reusable UI components that are used across multiple pages. For example, the page header, side navigation, and custom dialogs are defined here. This promotes code reuse and a consistent look and feel.

*   **`models/`**: This is where the core business logic of the application resides. It contains the code for interacting with backend services, such as:
    *   Calling Generative AI models (e.g., Imagen, Veo).
    *   Interacting with databases (e.g., Firestore).
    *   Handling data transformations and other business logic.

*   **`state/`**: This directory defines the application's state management classes using Mesop's `@me.stateclass`. These classes hold the data that needs to be shared and preserved across different components and user interactions.

*   **`config/`**: This directory is for application configuration. It includes:
    *   `default.py`: For defining default application settings and loading environment variables.
    *   `navigation.json`: For configuring the application's side navigation.
    *   `rewriters.py`: For storing the prompt templates used by the AI models.

## Firestore Setup

This application uses Firestore to store metadata for the media library. Here's how to set it up:

1.  **Create a Firestore Database:** In your Google Cloud Project, create a Firestore database in Native Mode.

2.  **Create a Collection:** Create a collection named `genmedia`. This is the default collection name, but it can be overridden with the `GENMEDIA_COLLECTION_NAME` environment variable.

3.  **Create an Index:** Create a single-field index for the `timestamp` field with the query scope set to "Collection" and the order set to "Descending". This will allow the library to sort media by the time it was created.

4.  **Set Security Rules:** To protect your data, set the following security rules in the "Rules" tab of your Firestore database:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Match any document in the 'genmedia' collection
    match /genmedia/{docId} {
      // Allow read and write access only if the user is authenticated
      // and their email matches the 'user_email' field in the document.
      allow read, write: if request.auth != null && request.auth.token.email == resource.data.user_email;
    }
  }
}
```

## Configuration-Driven Architecture

A key architectural principle in this project is the use of centralized, type-safe configuration to drive the behavior of the UI and backend logic. This approach makes the application more robust, easier to maintain, and less prone to bugs.

A prime example of this is the handling of the VEO and Imagen models. Instead of hardcoding model names, capabilities, and constraints throughout the application, we define them in a single location:

-   **`config/veo_models.py`**
-   **`config/imagen_models.py`**

These files contain `dataclass` definitions that serve as a **single source of truth** for each model's properties, such as:

-   Supported aspect ratios
-   Valid video durations (min, max, and default)
-   Maximum number of samples
-   Supported generation modes (e.g., `t2v`, `i2v`, `interpolation`)

The UI components then read from these configuration objects to dynamically build the user interface. For example, the model selection dropdowns, sliders, and feature toggles are all populated and configured based on the capabilities of the currently selected model. This means that to add a new model or update an existing one, you only need to modify the relevant configuration file, and the entire application will adapt automatically.

This pattern is the preferred way to manage model capabilities and other complex configurations in this project.

## Testing Strategy

This project uses a combination of unit tests and integration tests to ensure code quality and reliability. The tests are located in the `test/` directory and are built using the `pytest` framework.

### Unit Tests

Unit tests are used to verify the internal logic of individual functions and components in isolation. They are fast, reliable, and do not depend on external services. We use mocking to simulate the behavior of external APIs, allowing us to test our code's logic without making real network calls.

### Integration Tests

Integration tests are used to verify the interaction between our application and live Google Cloud services. These tests make real API calls and are essential for confirming that our code can successfully communicate with the VEO and Imagen APIs.

-   **Marker:** Integration tests are marked with the `@pytest.mark.integration` decorator.
-   **Execution:** These tests are skipped by default. To run them, use the `-m` flag:
    ```bash
    pytest -m integration -v -s
    ```

### Component-Level Tests

For testing the data flow within our components (e.g., from a successful API call to the Firestore logging), we use component-level integration tests. These tests mock the external API calls but let the internal data handling and state management logic run as it normally would. This is a powerful way to catch bugs in our data mapping and event handling logic.

### Test Configuration

Tests that require access to Google Cloud Storage can be configured to use a custom GCS bucket via the `--gcs-bucket` command-line option. See the `test/README.md` file for more details.

## How to Add a New Page

Adding a new page to the application is a straightforward process. Here are the steps:

### Step 1: Create the Page File

Create a new Python file in the `pages/` directory. The name of the file should be descriptive of the page's purpose (e.g., `my_new_page.py`).

In this file, define a function that will contain the UI for your page. This function should accept the application state as an argument.

**`pages/my_new_page.py`:**
```python
import mesop as me
from state.state import AppState
from components.header import header
from components.page_scaffold import page_frame, page_scaffold

def my_new_page_content(app_state: AppState):
    with page_scaffold():
        with page_frame():
            header("My New Page", "rocket_launch")
            me.text("Welcome to my new page!")
```

### Step 2: Register the Page in `main.py`

Next, you need to register your new page in `main.py` so that the application knows how to serve it.

1.  Import your new page function at the top of `main.py`:
    ```python
    from pages.my_new_page import my_new_page_content
    ```

2.  Add a new `@me.page` decorator to define the route and other page settings:
    ```python
    @me.page(
        path="/my_new_page",
        title="My New Page - GenMedia Creative Studio",
        on_load=on_load,
    )
    def my_new_page():
        my_new_page_content(me.state(AppState))
    ```

### Step 3: Add the Page to the Navigation

To make your new page accessible to users, you need to add it to the side navigation.

1.  Open the `config/navigation.json` file.
2.  Add a new JSON object to the `pages` array for your new page. Make sure to give it a unique `id`.

**`config/navigation.json`:**
```json
{
  "pages": [
    // ... other pages ...
    {
      "id": 60, // Make sure this ID is unique
      "display": "My New Page",
      "icon": "rocket_launch",
      "route": "/my_new_page",
      "group": "workflows"
    }
  ]
}
```

### Step 4: (Optional) Control with a Feature Flag

If you want to control the visibility of your new page with an environment variable, you can add a `feature_flag` to its entry in `navigation.json`.

```json
{
  "id": 60,
  "display": "My New Page",
  "icon": "rocket_launch",
  "route": "/my_new_page",
  "group": "workflows",
  "feature_flag": "MY_NEW_PAGE_ENABLED"
}
```

Now, the "My New Page" link will only appear in the navigation if the `MY_NEW_PAGE_ENABLED` environment variable is set to `True` in your `.env` file.

That's it! When you restart the application, your new page will be available at the route you defined and will appear in the side navigation.

## Key Takeaways from the VTO Page Development

- **GCS URI Handling:** This is a critical and recurring theme. The `common.storage.store_to_gcs` function returns a **full** GCS URI (e.g., `gs://your-bucket/your-object.png`). When using this value, you must be careful not to prepend the `gs://` prefix or the bucket name again. Doing so will create an invalid path and lead to "No such object" errors.
    - **For API Calls:** Pass the GCS URI returned from `store_to_gcs` directly to the API.
    - **For Displaying in Mesop:** To create a public URL for the `me.image` component, use the `.replace("gs://", "https://storage.mtls.cloud.google.com/")` method on the full GCS URI.

- **Displaying GCS Images:** The `me.image` component requires a public HTTPS URL, not a `gs://` URI. To display images from GCS, replace `gs://` with `https://storage.mtls.cloud.google.com/`.

- **State Management:** Avoid using mutable default values (like `[]`) in your state classes. Instead, use `field(default_factory=list)` to ensure that a new list is created for each user session.

- **UI Components:** If a component doesn't support a specific parameter (like `label` on `me.slider`), you can often achieve the same result by wrapping it in a `me.box` and using other components (like `me.text`) to create the desired layout.

- **Generator Functions:** When working with generator functions (those that use `yield`), make sure to include a `yield` statement after updating the state to ensure that the UI is updated.
