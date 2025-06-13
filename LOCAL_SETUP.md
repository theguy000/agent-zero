# Local Development Setup

This guide provides instructions for setting up and running the application components (SearXNG, UI, CLI, Cloudflare Tunnel) directly on your local machine without using Docker.

**Disclaimer:** This setup is intended for development and testing purposes only. It may not offer the same level of isolation, reproducibility, and ease of management as the Docker-based deployment. Some features or configurations might behave differently. For production or a more stable environment, please refer to the Docker setup instructions in the main `README.md`.

## Prerequisites

Before you begin, ensure you have the following installed on your system:

*   **Python 3.8+:** Required for SearXNG and the UI/CLI components.
*   **pip:** Python package installer.
*   **git:** For cloning SearXNG.
*   **bash:** For running the helper scripts.
*   **Cloudflared:** (Optional) If you plan to use the Cloudflare Tunnel to expose your local instance. You can download it from [here](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation/).
*   **Poetry:** For SearXNG dependency management. Installation instructions: `pip install poetry`.
*   **Node.js and npm:** (Optional, if you want to modify and rebuild the UI's frontend assets).

## 1. Running SearXNG Locally

This script automates the setup and execution of a local SearXNG instance.

**Script: `run_searxng_local.sh`**

```bash
#!/bin/bash

# Variables
SEARXNG_DIR="searxng-src" # Directory to clone SearXNG into
SEARXNG_SETTINGS_FILE="settings.yml"
SEARXNG_PORT="8080" # Default SearXNG port

# Check if SearXNG directory exists
if [ ! -d "$SEARXNG_DIR" ]; then
  echo "SearXNG directory not found. Cloning SearXNG..."
  git clone https://github.com/searxng/searxng "$SEARXNG_DIR"
  if [ $? -ne 0 ]; then
    echo "Error cloning SearXNG. Please check your internet connection and git installation."
    exit 1
  fi
else
  echo "SearXNG directory found. Skipping clone."
fi

cd "$SEARXNG_DIR" || exit

# Check if settings.yml exists, create a basic one if not
if [ ! -f "$SEARXNG_SETTINGS_FILE" ]; then
  echo "Creating a basic settings.yml..."
  cat > "$SEARXNG_SETTINGS_FILE" <<EOF
use_default_settings: true

server:
  port: $SEARXNG_PORT
  bind_address: "127.0.0.1" # Bind to localhost for local development

# Add any other specific settings you need for local development
# For example, to enable specific engines:
#
# general:
#   instance_name: "My Local SearXNG"
#
# search:
#   # ... your other search settings
#
# engines:
#   - name: 'google'
#     engine: 'google'
#     shortcut: 'g'
#     # You might need to configure cookies or other parameters for some engines
#   - name: 'duckduckgo'
#     engine: 'duckduckgo'
#     shortcut: 'ddg'
EOF
fi

echo "Installing/Updating SearXNG dependencies using Poetry..."
poetry install
if [ $? -ne 0 ]; then
  echo "Error installing dependencies with Poetry. Please check your Poetry installation and Python environment."
  exit 1
fi

echo "Starting SearXNG on http://127.0.0.1:$SEARXNG_PORT..."
echo "Note: This script will run SearXNG in the foreground. Press Ctrl+C to stop."

# Activate virtual environment if poetry created one and run SearXNG
# The command to run searxng might vary slightly based on poetry's setup
# Trying common ways to run it via poetry
if poetry run python searx/webapp.py -s "$SEARXNG_SETTINGS_FILE"; then
  echo "SearXNG stopped."
else
  echo "Failed to start SearXNG directly with 'poetry run python searx/webapp.py'."
  echo "Attempting to activate venv and run..."
  # Try to find the venv path if `poetry shell` isn't suitable for scripting
  VENV_PATH=$(poetry env info --path)
  if [ -n "$VENV_PATH" ] && [ -f "$VENV_PATH/bin/activate" ]; then
    # shellcheck disable=SC1091
    source "$VENV_PATH/bin/activate"
    python searx/webapp.py -s "$SEARXNG_SETTINGS_FILE"
    deactivate
  else
    echo "Could not determine Poetry virtual environment. Please start SearXNG manually within the '$SEARXNG_DIR' directory."
    echo "You might need to run 'poetry shell' then 'python searx/webapp.py -s $SEARXNG_SETTINGS_FILE'"
  fi
fi

cd ..
```

**To use it:**

1.  Save the script above as `run_searxng_local.sh` in your project's root directory.
2.  Make it executable: `chmod +x run_searxng_local.sh`
3.  Run it: `./run_searxng_local.sh`

This will clone SearXNG (if not already present), install its dependencies using Poetry, and start the SearXNG web server on `http://127.0.0.1:8080`.

**Note:**
*   The script creates a basic `settings.yml` if one doesn't exist. You can customize this file with your preferred SearXNG settings.
*   SearXNG will run in the foreground. Press `Ctrl+C` to stop it.
*   Ensure Poetry is installed and configured correctly.

## 2. Running the UI Locally

This script runs the Gradio UI application, pointing to the local SearXNG instance.

**Script: `run_local_ui.sh`**

```bash
#!/bin/bash

# Default SearXNG URL for local instance
SEARXNG_INSTANCE_URL="http://127.0.0.1:8080"
UI_APP_FILE="src/app.py" # Path to your Gradio UI application
UI_PORT="7860" # Port for the Gradio UI

# Check if the UI application file exists
if [ ! -f "$UI_APP_FILE" ]; then
    echo "UI application file not found at $UI_APP_FILE"
    echo "Please ensure the path is correct."
    exit 1
fi

echo "Starting the Gradio UI..."
echo "It will connect to SearXNG at: $SEARXNG_INSTANCE_URL"
echo "UI will be available at: http://127.0.0.1:$UI_PORT (or the port Gradio chooses)"

# Set environment variables for the UI application
export SEARXNG_INSTANCE_URL="$SEARXNG_INSTANCE_URL"
export GRADIO_SERVER_NAME="127.0.0.1" # Bind Gradio to localhost
export GRADIO_SERVER_PORT="$UI_PORT"

# Assuming your Gradio app is in src/app.py and can be run directly
# You might need to adjust this command based on your project structure
# e.g., if you use a virtual environment or a different entry point.

# Create a virtual environment if it doesn't exist and install requirements
VENV_DIR=".venv-ui"
if [ ! -d "$VENV_DIR" ]; then
    echo "Creating Python virtual environment for UI at $VENV_DIR..."
    python3 -m venv "$VENV_DIR"
    if [ $? -ne 0 ]; then
        echo "Failed to create virtual environment."
        exit 1
    fi
fi

# shellcheck disable=SC1091
source "$VENV_DIR/bin/activate"

echo "Installing/Updating UI dependencies from requirements.txt..."
pip install -r requirements.txt
if [ $? -ne 0 ]; then
    echo "Failed to install UI requirements."
    deactivate
    exit 1
fi


python3 "$UI_APP_FILE"

echo "Gradio UI stopped."
deactivate
```

**To use it:**

1.  Ensure your SearXNG instance is running (from Step 1).
2.  Save the script above as `run_local_ui.sh` in your project's root directory.
3.  Make it executable: `chmod +x run_local_ui.sh`
4.  Run it: `./run_local_ui.sh`

This will start the Gradio UI, which should be accessible at `http://127.0.0.1:7860` (or another port if 7860 is busy). The UI will be configured to use the local SearXNG instance.

## 3. Running the CLI Locally

This script runs the CLI application, interacting with the local SearXNG instance.

**Script: `run_local_cli.sh`**

```bash
#!/bin/bash

# Default SearXNG URL for local instance
SEARXNG_INSTANCE_URL="http://127.0.0.1:8080"
CLI_APP_FILE="src/cli.py" # Path to your CLI application

# Check if the CLI application file exists
if [ ! -f "$CLI_APP_FILE" ]; then
    echo "CLI application file not found at $CLI_APP_FILE"
    echo "Please ensure the path is correct."
    exit 1
fi

echo "Starting the CLI application..."
echo "It will connect to SearXNG at: $SEARXNG_INSTANCE_URL"

# Set environment variables for the CLI application
export SEARXNG_INSTANCE_URL="$SEARXNG_INSTANCE_URL"

# Create a virtual environment if it doesn't exist and install requirements
# This uses the same venv as the UI for simplicity, you can separate them if needed.
VENV_DIR=".venv-ui" # Using the same venv as UI
if [ ! -d "$VENV_DIR" ]; then
    echo "Creating Python virtual environment for CLI at $VENV_DIR..."
    python3 -m venv "$VENV_DIR"
    if [ $? -ne 0 ]; then
        echo "Failed to create virtual environment."
        exit 1
    fi
fi

# shellcheck disable=SC1091
source "$VENV_DIR/bin/activate"

echo "Installing/Updating CLI dependencies from requirements.txt (if not already done by UI setup)..."
pip install -r requirements.txt
if [ $? -ne 0 ]; then
    echo "Failed to install CLI requirements."
    deactivate
    exit 1
fi

# Run the CLI, passing any arguments provided to this script
python3 "$CLI_APP_FILE" "$@"

echo "CLI application finished."
deactivate
```

**To use it:**

1.  Ensure your SearXNG instance is running.
2.  Save the script above as `run_local_cli.sh` in your project's root directory.
3.  Make it executable: `chmod +x run_local_cli.sh`
4.  Run it with your desired query and options: `./run_local_cli.sh "your search query" --params "language=en,format=json"`

The CLI will output the search results to your terminal.

## 4. Exposing Your Local Instance with Cloudflare Tunnel (Optional)

This script helps you expose your locally running UI (and by extension, SearXNG) to the internet using a Cloudflare Tunnel.

**Prerequisites for this script:**
*   `cloudflared` CLI installed and authenticated.
*   A domain/subdomain configured in your Cloudflare account.

**Script: `run_local_tunnel.sh`**

```bash
#!/bin/bash

# Configuration
LOCAL_UI_URL="http://127.0.0.1:7860" # The local URL of your Gradio UI
# Set this environment variable or replace the placeholder directly
CLOUDFLARE_HOSTNAME="${YOUR_CLOUDFLARE_TUNNEL_HOSTNAME}"

# Check if cloudflared is installed
if ! command -v cloudflared &> /dev/null; then
    echo "cloudflared could not be found. Please install it first."
    echo "Visit: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation/"
    exit 1
fi

# Check if hostname is set
if [ -z "$CLOUDFLARE_HOSTNAME" ] || [ "$CLOUDFLARE_HOSTNAME" == "your-tunnel-hostname.example.com" ]; then
    echo "Error: CLOUDFLARE_HOSTNAME is not set or is set to the placeholder."
    echo "Please set the YOUR_CLOUDFLARE_TUNNEL_HOSTNAME environment variable or edit the script."
    exit 1
fi

echo "Starting Cloudflare Tunnel..."
echo "Exposing: $LOCAL_UI_URL"
echo "To Hostname: $CLOUDFLARE_HOSTNAME"
echo "Press Ctrl+C to stop the tunnel."

# Run cloudflared tunnel
# The command creates a new named tunnel or uses an existing one if the name/config matches
# For a persistent named tunnel, you'd typically create it first with `cloudflared tunnel create <name>`
# and then `cloudflared tunnel route dns <name> <hostname>`.
# This script uses a simpler approach for quick, temporary tunnels.
cloudflared tunnel --url "$LOCAL_UI_URL" --hostname "$CLOUDFLARE_HOSTNAME"

# If you have a pre-configured tunnel ID, you might use:
# cloudflared tunnel --no-autoupdate run --token <YOUR_TUNNEL_TOKEN>
# Or if you have a tunnel name linked to your DNS:
# cloudflared tunnel run <YOUR_TUNNEL_NAME>

echo "Cloudflare Tunnel stopped."
```

**To use it:**

1.  Ensure your UI is running locally (via `run_local_ui.sh`).
2.  Install and authenticate `cloudflared`.
3.  Set the `YOUR_CLOUDFLARE_TUNNEL_HOSTNAME` environment variable to your desired Cloudflare hostname (e.g., `export YOUR_CLOUDFLARE_TUNNEL_HOSTNAME="mysearx.mydomain.com"`) or directly edit the script.
4.  Save the script above as `run_local_tunnel.sh` in your project's root directory.
5.  Make it executable: `chmod +x run_local_tunnel.sh`
6.  Run it: `./run_local_tunnel.sh`

This will create a tunnel, and your local UI will be accessible via the specified Cloudflare hostname.

## Important Notes for Local Setup

*   **Dependency Management:** These scripts assume basic Python/pip/Poetry usage. If you have complex dependency needs or conflicts, consider using more robust virtual environment management.
*   **SearXNG Configuration:** The local SearXNG instance uses a minimal `settings.yml`. For advanced configurations (e.g., specific engines, API keys, theming), you'll need to modify `searxng-src/settings.yml` accordingly. Refer to the official SearXNG documentation.
*   **Security:** Exposing local services with Cloudflare Tunnel is generally secure, but always be mindful of what you expose. The local SearXNG and Gradio instances are bound to `127.0.0.1` by default in these scripts, which is good practice.
*   **Resource Usage:** Running multiple services locally can consume significant system resources.
*   **Updates:** You are responsible for manually updating SearXNG (e.g., by pulling changes in the `searxng-src` directory and re-running dependency installation) and other components.

This local setup provides a flexible way to work on the project components. Remember to consult the official documentation for each tool (SearXNG, Gradio, Cloudflared) for more in-depth information.
```
