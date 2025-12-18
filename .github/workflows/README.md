# GitHub Actions Workflows

This repository contains two types of workflows:

1. **Azure Container Registry Deployment** - Builds and deploys the application container to ACR
2. **Agent Deployment Workflows** - Deploys/updates individual agents to Microsoft Foundry

---

## Azure Container Registry Deployment

This workflow automatically builds and deploys the application from the `src/` folder to Azure Container Registry when changes are pushed to the main branch.

## Prerequisites

### 1. Azure Container Registry
You need an Azure Container Registry already created. If you don't have one, create it using:

```bash
az acr create --resource-group <resource-group> --name <registry-name> --sku Basic
```

### 2. Enable Admin Access (for username/password authentication)

```bash
az acr update --name <registry-name> --admin-enabled true
```

### 3. Get ACR Credentials

```bash
# Get the login server
az acr show --name <registry-name> --query loginServer --output tsv

# Get credentials
az acr credential show --name <registry-name>
```

## Required GitHub Secrets

Configure the following secrets in your GitHub repository (Settings → Secrets and variables → Actions → New repository secret):

### Required Secrets:

1. **AZURE_CONTAINER_REGISTRY**
   - Your Azure Container Registry login server (e.g., `myregistry.azurecr.io`)
   - Find it: `az acr show --name <registry-name> --query loginServer -o tsv`

2. **AZURE_CONTAINER_REGISTRY_USERNAME**
   - Your ACR admin username
   - Find it: `az acr credential show --name <registry-name> --query username -o tsv`

3. **AZURE_CONTAINER_REGISTRY_PASSWORD**
   - Your ACR admin password
   - Find it: `az acr credential show --name <registry-name> --query "passwords[0].value" -o tsv`

4. **ENV**
   - The contents of your `.env` file (all environment variables)
   - This should be a multi-line secret containing all your environment variables
   - Example format:
     ```
     AZURE_OPENAI_ENDPOINT=https://your-endpoint.openai.azure.com/
     AZURE_OPENAI_API_KEY=your-key-here
     COSMOS_DB_ENDPOINT=https://your-cosmos.documents.azure.com:443/
     # Add all other environment variables
     ```

## Workflow Details

The workflow ([.github/workflows/deploy-to-acr.yml](.github/workflows/deploy-to-acr.yml)):

1. **Triggers**: On push to `main` branch when files in `src/` change
2. **Authentication**: Uses ACR username/password authentication
3. **Build Context**: Uses only the `src/` folder
4. **Tagging Strategy**:
   - `latest` tag for main branch
   - Branch-specific tags with git SHA
5. **Security**:
   - Creates `.env` file from GitHub secret during build
   - Cleans up `.env` file after build (even if build fails)
   - `.env` file is never committed to the repository

## Image Tags

Images are tagged with:
- `latest` - Always points to the latest build from main
- `main-<git-sha>` - Specific commit identifier
- `main` - Branch identifier

## Testing the Workflow

1. Set up all required secrets in GitHub
2. Make a change to any file in the `src/` folder
3. Commit and push to the `main` branch:
   ```bash
   git add .
   git commit -m "Test ACR deployment"
   git push origin main
   ```
4. Monitor the workflow in the "Actions" tab of your GitHub repository

## Pulling the Image

After successful deployment, pull and run your image:

```bash
# Login to ACR
az acr login --name <registry-name>

# Pull the image
docker pull <registry-name>.azurecr.io/shopping-assistant:latest

# Run the container
docker run -p 8000:8000 <registry-name>.azurecr.io/shopping-assistant:latest
```

## Alternative: Using Azure Service Principal (More Secure)

For production environments, consider using an Azure Service Principal instead of admin credentials:

```bash
# Create a service principal
az ad sp create-for-rbac --name "github-actions-acr" --role acrpush \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerRegistry/registries/<registry-name> \
  --sdk-auth

# Use the output to create GitHub secrets:
# - AZURE_CREDENTIALS (entire JSON output)
```

Then update the workflow to use the `azure/login@v1` action instead of docker-login.

## Troubleshooting

### Build Fails
- Check the Actions tab for detailed error logs
- Verify all secrets are correctly set
- Ensure the Dockerfile is valid

### Authentication Errors
- Verify ACR credentials are correct
- Check that admin access is enabled on ACR
- Ensure secrets don't have extra spaces or newlines

### .env Issues
- Verify the ENV secret contains valid environment variables
- Check that the format matches your application's requirements
- Look for special characters that might need escaping

## Security Best Practices

✅ **DO:**
- Store all secrets in GitHub Secrets
- Use the ENV secret for all environment variables
- Regularly rotate ACR credentials
- Use Azure Service Principal for production

❌ **DON'T:**
- Commit `.env` files to the repository
- Hardcode secrets in workflow files
- Share ACR credentials publicly
- Use admin credentials in production (use service principal instead)

---

## Agent Deployment Workflows

These workflows automatically deploy or update individual agents to Microsoft Foundry when their configuration files change. Each agent has its own workflow file to allow independent deployment.

### Workflow Configuration

The workflow files have been configured with the correct paths and references for each agent:

| Workflow File | Agent Initializer | Prompt File | Tool Files |
|--------------|-------------------|-------------|------------|
| `deploy-loyalty-agent.yml` | `customerLoyaltyAgent_initializer.py` | `CustomerLoyaltyAgentPrompt.txt` | `discountLogic.py` |
| `deploy-cart-manager-agent.yml` | `cartManagerAgent_initializer.py` | `CartManagerPrompt.txt` | None |
| `deploy-interior-design-agent.yml` | `interiorDesignAgent_initializer.py` | `InteriorDesignAgentPrompt.txt` | `imageCreationTool.py`, `aiSearchTools.py` |
| `deploy-inventory-agent.yml` | `inventoryAgent_initializer.py` | `InventoryAgentPrompt.txt` | `inventoryCheck.py` |
| `deploy-shopper-agent.yml` | `shopperAgent_initializer.py` | `ShopperAgentPrompt.txt` | `aiSearchTools.py` |

**Note**: The `.github/` directory has been removed from `.gitignore` to allow these workflow files to be tracked in version control.

### Available Agent Workflows

1. **deploy-loyalty-agent.yml** - Customer Loyalty Agent
2. **deploy-cart-manager-agent.yml** - Cart Manager Agent
3. **deploy-interior-design-agent.yml** - Interior Design Agent
4. **deploy-inventory-agent.yml** - Inventory Agent
5. **deploy-shopper-agent.yml** - Shopper Agent (Cora)

### Prerequisites

#### 1. Create a Service Principal

Create a service principal with contributor access to your resource group:

```bash
az ad sp create-for-rbac --name "TechWorkshopL300AzureAI" --json-auth --role contributor --scopes /subscriptions/<subscription-id>/resourceGroups/<resource-group>
```

Save the entire JSON output for the next step.

#### 2. Grant Service Principal Access to Microsoft Foundry

The service principal needs the **Azure AI User** role on your Microsoft Foundry resource:

1. Navigate to your Microsoft Foundry resource in the Azure Portal
2. Go to **Access control (IAM)** → **Add** → **Add role assignment**
3. Select the **Azure AI User** role
4. Under **Members**, select **User, group, or service principal**
5. Search for and select `TechWorkshopL300AzureAI`
6. Click **Review + assign**

### Required GitHub Secrets

Configure these secrets in your GitHub repository (Settings → Secrets and variables → Actions):

1. **AZURE_CREDENTIALS**
   - The complete JSON output from the `az ad sp create-for-rbac` command
   - This allows the workflow to authenticate with Azure

2. **ENV**
   - The contents of your `.env` file (all environment variables)
   - Must include all agent IDs and Azure connection strings
   - Example format:
     ```
     AZURE_AI_AGENT_ENDPOINT=https://your-foundry.cognitiveservices.azure.com/
     AZURE_AI_AGENT_MODEL_DEPLOYMENT_NAME=gpt-4o
     cora=<agent-id>
     interior_designer=<agent-id>
     customer_loyalty=<agent-id>
     inventory_agent=<agent-id>
     cart_manager=<agent-id>
     # Add all other environment variables
     ```

### Workflow Triggers

Each agent workflow is triggered by changes to:
- The agent's initializer file (e.g., `src/app/agents/customerLoyaltyAgent_initializer.py`)
- The agent's prompt file (e.g., `src/prompts/CustomerLoyaltyAgentPrompt.txt`)
- Any tool files the agent uses (e.g., `src/app/tools/discountLogic.py`)
- The workflow file itself

All workflows also support manual triggering via `workflow_dispatch`.

### Workflow Details

Each agent workflow:
1. Checks out the repository
2. Creates `.env` file from the `ENV` secret
3. Sets up Python 3.12
4. Installs dependencies from `src/requirements.txt`
5. Authenticates with Azure using the service principal
6. Runs the agent initializer script

The initializer checks if the agent already exists (by reading the agent ID from the `.env` file):
- **If agent exists**: Updates the agent with new configuration
- **If agent doesn't exist**: Creates a new agent

### Agent Tools Reference

| Agent | Tools Used | Tool Files |
|-------|-----------|------------|
| Customer Loyalty | `calculate_discount` | `discountLogic.py` |
| Cart Manager | None (uses conversation context) | None |
| Interior Design | `create_image`, `product_recommendations` | `imageCreationTool.py`, `aiSearchTools.py` |
| Inventory | `inventory_check` | `inventoryCheck.py` |
| Shopper (Cora) | `product_recommendations` | `aiSearchTools.py` |

### Testing Agent Deployment

1. Make a change to an agent's prompt file, initializer, or tool
2. Commit and push to the main branch:
   ```bash
   git add .
   git commit -m "Update customer loyalty agent prompt"
   git push origin main
   ```
3. Monitor the workflow in the **Actions** tab
4. Verify the agent was updated in Microsoft Foundry

### Manual Workflow Trigger

You can also manually trigger any agent workflow:

1. Go to the **Actions** tab in your GitHub repository
2. Select the desired agent workflow from the left sidebar
3. Click **Run workflow** → **Run workflow**

### Troubleshooting Agent Deployments

#### Authentication Errors
- Verify `AZURE_CREDENTIALS` secret is correctly set with the full JSON output
- Confirm the service principal has the **Azure AI User** role on Microsoft Foundry
- Check that the service principal has not expired

#### Agent Not Updating
- Ensure the agent ID in the `ENV` secret matches the actual agent ID in Microsoft Foundry
- Verify the `.env` file contains all required environment variables
- Check the workflow logs for specific error messages

#### Python Errors
- Verify `requirements.txt` includes all necessary packages
- Check that the Python version matches (3.12)
- Ensure the initializer file path is correct in the workflow

### Best Practices

✅ **DO:**
- Keep agent IDs in the ENV secret up to date
- Test changes locally before pushing
- Use descriptive commit messages
- Monitor workflow runs in the Actions tab

❌ **DON'T:**
- Commit agent IDs or credentials to the repository
- Manually create duplicate agents
- Skip testing prompt changes before deployment
- Delete agents without updating the ENV secret
