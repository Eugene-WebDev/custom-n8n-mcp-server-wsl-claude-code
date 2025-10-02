# custom-n8n-mcp-server-wsl-claude-code

## # Complete n8n MCP Server Setup Guide

**Battle-tested setup guide with lessons learned and troubleshooting tips**

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 0: Install Claude Code](#step-0-install-claude-code)
4. [Step 1: Prepare WSL Environment](#step-1-prepare-wsl-environment)
5. [Step 2: Create MCP Server Project](#step-2-create-mcp-server-project)
6. [Step 3: Write the MCP Server Code](#step-3-write-the-mcp-server-code)
7. [Step 4: Build and Configure](#step-4-build-and-configure)
8. [Step 5: Configure n8n Credentials](#step-5-configure-n8n-credentials)
9. [Step 6: Connect Claude Code to MCP Server](#step-6-connect-claude-code-to-mcp-server)
10. [Step 7: Test Your Setup](#step-7-test-your-setup)
11. [Troubleshooting](#troubleshooting)
12. [Lessons Learned](#lessons-learned)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Windows 10 Machine                        │
│                                                              │
│  ┌─────────────────┐           ┌──────────────────────┐    │
│  │   Claude Code   │ ◄────────►│  WSL Ubuntu 24.04    │    │
│  │    (Windows)    │           │                      │    │
│  └─────────────────┘           │  ┌────────────────┐  │    │
│                                 │  │ n8n MCP Server │  │    │
│                                 │  │  (Node.js)     │  │    │
│                                 │  └────────┬───────┘  │    │
│                                 └───────────┼──────────┘    │
└─────────────────────────────────────────────┼───────────────┘
                                              │
                                              │ HTTPS/API
                                              ▼
                                  ┌─────────────────────┐
                                  │   n8n Instance      │
                                  │   (VPS/Cloud)       │
                                  │ n8n.yoursite.com    │
                                  └─────────────────────┘
```

**Components:**
- **Claude Code**: CLI tool running on Windows that you interact with
- **MCP Server**: Runs in WSL Ubuntu, acts as a bridge between Claude and n8n
- **n8n**: Your workflow automation tool hosted on a VPS

**Why this architecture?**
- MCP server runs locally for low latency with Claude Code
- Communicates with remote n8n via HTTP API (no VPN needed)
- Easy to develop and debug locally

---

## Prerequisites

### Required Software
- ✅ Windows 10/11 with WSL2 enabled
- ✅ WSL Ubuntu 24.04 distribution
- ✅ Node.js (will be installed in WSL)
- ✅ n8n instance with API access enabled

### Required Access
- 🔑 n8n instance URL (e.g., `https://n8n.yoursite.com`)
- 🔑 n8n API key (generate from n8n Settings → API)

---

## Step 0: Install Claude Code

### On Windows (PowerShell or Command Prompt)

```powershell
# Install Claude Code globally via npm
npm install -g @anthropic-ai/claude-code

# Verify installation
claude-code --version
```

**💡 Tip:** If you don't have Node.js on Windows yet, download it from [nodejs.org](https://nodejs.org/) (LTS version recommended).

---

## Step 1: Prepare WSL Environment

### 1.1 Install Node.js in WSL

Open WSL Ubuntu terminal:

```bash
# Navigate to home directory
cd ~

# Install Node.js 20 LTS (recommended)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo bash -
sudo apt-get install -y nodejs

# Verify installation
node --version   # Should show v20.x.x
npm --version    # Should show 10.x.x
```

**⚠️ Common Issue:** Docker Desktop may interfere with WSL environments. Make sure you're installing Node.js in the correct WSL distribution (Ubuntu-24.04).

**💡 Tip:** Check your WSL distributions with:
```bash
wsl -l -v
```

---

## Step 2: Create MCP Server Project

### 2.1 Initialize Project

In your WSL Ubuntu terminal:

```bash
# Create project directory
cd ~
mkdir n8n-mcp-server
cd n8n-mcp-server

# Initialize Node.js project
npm init -y
```

### 2.2 Install Dependencies

```bash
# Install MCP SDK and required packages
npm install @modelcontextprotocol/sdk axios dotenv

# Install TypeScript and type definitions
npm install -D @types/node typescript
```

**Expected output:**
- Should install ~100+ packages
- No vulnerabilities (or only low-severity ones)

### 2.3 Create TypeScript Configuration

Create `tsconfig.json`:

```bash
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}
EOF
```

### 2.4 Update package.json

```bash
# Set module type and scripts
npm pkg set type="module"
npm pkg set scripts.build="tsc"
npm pkg set scripts.start="node build/index.js"
```

---

## Step 3: Write the MCP Server Code

### 3.1 Create Source Directory

```bash
mkdir src
```

### 3.2 Create the Server File

Create `src/index.ts` with the following content:

```typescript
#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import axios from "axios";
import * as dotenv from "dotenv";

dotenv.config();

const N8N_URL = process.env.N8N_URL || "http://your-vps-ip:5678";
const N8N_API_KEY = process.env.N8N_API_KEY || "";

const api = axios.create({
  baseURL: `${N8N_URL}/api/v1`,
  headers: {
    "X-N8N-API-KEY": N8N_API_KEY,
  },
});

const server = new Server(
  {
    name: "n8n-mcp-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// List available tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "list_workflows",
        description: "List all workflows in n8n",
        inputSchema: {
          type: "object",
          properties: {},
        },
      },
      {
        name: "get_workflow",
        description: "Get details of a specific workflow by ID",
        inputSchema: {
          type: "object",
          properties: {
            id: {
              type: "string",
              description: "Workflow ID",
            },
          },
          required: ["id"],
        },
      },
      {
        name: "create_workflow",
        description: "Create a new workflow",
        inputSchema: {
          type: "object",
          properties: {
            name: {
              type: "string",
              description: "Workflow name",
            },
            nodes: {
              type: "array",
              description: "Array of workflow nodes",
            },
            connections: {
              type: "object",
              description: "Node connections",
            },
            settings: {
              type: "object",
              description: "Workflow settings",
            },
          },
          required: ["name"],
        },
      },
      {
        name: "update_workflow",
        description: "Update an existing workflow",
        inputSchema: {
          type: "object",
          properties: {
            id: {
              type: "string",
              description: "Workflow ID",
            },
            name: {
              type: "string",
              description: "Workflow name",
            },
            nodes: {
              type: "array",
              description: "Array of workflow nodes",
            },
            connections: {
              type: "object",
              description: "Node connections",
            },
          },
          required: ["id"],
        },
      },
      {
        name: "delete_workflow",
        description: "Delete a workflow by ID",
        inputSchema: {
          type: "object",
          properties: {
            id: {
              type: "string",
              description: "Workflow ID",
            },
          },
          required: ["id"],
        },
      },
      {
        name: "activate_workflow",
        description: "Activate a workflow",
        inputSchema: {
          type: "object",
          properties: {
            id: {
              type: "string",
              description: "Workflow ID",
            },
          },
          required: ["id"],
        },
      },
      {
        name: "deactivate_workflow",
        description: "Deactivate a workflow",
        inputSchema: {
          type: "object",
          properties: {
            id: {
              type: "string",
              description: "Workflow ID",
            },
          },
          required: ["id"],
        },
      },
    ],
  };
});

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    switch (request.params.name) {
      case "list_workflows": {
        const response = await api.get("/workflows");
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify(response.data, null, 2),
            },
          ],
        };
      }

      case "get_workflow": {
        const { id } = request.params.arguments as { id: string };
        const response = await api.get(`/workflows/${id}`);
        return {
          content: [
            {
              type: "text",
              text: JSON.stringify(response.data, null, 2),
            },
          ],
        };
      }

      case "create_workflow": {
        const workflowData = request.params.arguments;
        const response = await api.post("/workflows", workflowData);
        return {
          content: [
            {
              type: "text",
              text: `Workflow created successfully: ${JSON.stringify(response.data, null, 2)}`,
            },
          ],
        };
      }

      case "update_workflow": {
        const { id, ...updateData } = request.params.arguments as any;
        const response = await api.patch(`/workflows/${id}`, updateData);
        return {
          content: [
            {
              type: "text",
              text: `Workflow updated successfully: ${JSON.stringify(response.data, null, 2)}`,
            },
          ],
        };
      }

      case "delete_workflow": {
        const { id } = request.params.arguments as { id: string };
        await api.delete(`/workflows/${id}`);
        return {
          content: [
            {
              type: "text",
              text: `Workflow ${id} deleted successfully`,
            },
          ],
        };
      }

      case "activate_workflow": {
        const { id } = request.params.arguments as { id: string };
        await api.patch(`/workflows/${id}`, { active: true });
        return {
          content: [
            {
              type: "text",
              text: `Workflow ${id} activated`,
            },
          ],
        };
      }

      case "deactivate_workflow": {
        const { id } = request.params.arguments as { id: string };
        await api.patch(`/workflows/${id}`, { active: false });
        return {
          content: [
            {
              type: "text",
              text: `Workflow ${id} deactivated`,
            },
          ],
        };
      }

      default:
        throw new Error(`Unknown tool: ${request.params.name}`);
    }
  } catch (error: any) {
    return {
      content: [
        {
          type: "text",
          text: `Error: ${error.message}\n${error.response?.data ? JSON.stringify(error.response.data, null, 2) : ""}`,
        },
      ],
      isError: true,
    };
  }
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("n8n MCP Server running on stdio");
}

main();
```

**⚠️ Important:** If copying via bash heredoc, you may encounter issues with template literals (`${}`). Instead:
- Use a text editor (nano, vim, or VS Code with WSL extension)
- Or have Claude Code create it via the Write tool

---

## Step 4: Build and Configure

### 4.1 Compile TypeScript

```bash
cd ~/n8n-mcp-server

# Compile TypeScript to JavaScript
node ./node_modules/typescript/lib/tsc.js

# Verify build output
ls -la build/
```

**Expected output:**
- `build/index.js` file created (~8-9KB)
- No compilation errors

**⚠️ Common Issue:** If you see "node: command not found", your shell environment isn't loading the PATH correctly. Try:
```bash
# Use login shell
bash -l
cd ~/n8n-mcp-server
node ./node_modules/typescript/lib/tsc.js
```

**💡 Alternative:** You can also use:
```bash
npx tsc
# Or
npm run build
```

---

## Step 5: Configure n8n Credentials

### 5.1 Get Your n8n API Key

1. Open your n8n instance in a browser (e.g., `https://n8n.yoursite.com`)
2. Go to **Settings** → **API**
3. Click **Generate API Key**
4. Copy the key (you won't see it again!)

### 5.2 Create .env File

In WSL terminal:

```bash
cd ~/n8n-mcp-server

# Create .env file (use nano or your preferred editor)
nano .env
```

Add your credentials:

```env
N8N_URL=https://n8n.yoursite.com
N8N_API_KEY=your-actual-api-key-here
```

**💡 Tips:**
- Use `https://` if your n8n is behind a reverse proxy (SSL)
- Use your subdomain if you have one (better than IP address)
- If using IP: `http://123.45.67.89:5678` (note the port!)

**⚠️ Security:** Never commit `.env` to git! Add it to `.gitignore`:
```bash
echo ".env" >> .gitignore
```

---

## Step 6: Connect Claude Code to MCP Server

### 6.1 Create Claude Code Configuration

The config file should be at: `C:\Users\YourUsername\.claude\claude_desktop_config.json`

Create it with this content:

```json
{
  "mcpServers": {
    "n8n": {
      "command": "wsl",
      "args": [
        "-d",
        "Ubuntu-24.04",
        "--",
        "bash",
        "-c",
        "cd ~/n8n-mcp-server && node build/index.js"
      ]
    }
  }
}
```

**💡 How to create this file:**

**Option 1: Via Claude Code (recommended)**
- Ask Claude: "Create the MCP config file for me"
- Claude can use the Write tool to create it directly

**Option 2: Manually**
```powershell
# In PowerShell
mkdir $env:USERPROFILE\.claude -Force
notepad $env:USERPROFILE\.claude\claude_desktop_config.json
# Paste the JSON content and save
```

**⚠️ Important:** Make sure the WSL distribution name matches yours. Check with:
```powershell
wsl -l -v
```
If your distribution has a different name (e.g., "Ubuntu"), update the `-d` parameter.

---

## Step 7: Test Your Setup

### 7.1 Restart Claude Code

If Claude Code is already running, exit and restart it to load the new configuration.

```powershell
# Exit current session (Ctrl+C or type 'exit')
# Start fresh
claude-code
```

### 7.2 Test MCP Tools

Try these commands with Claude:

```
"List all my n8n workflows"

"Show me the available n8n tools"

"Get details for workflow ID <your-workflow-id>"
```

**Expected behavior:**
- Claude should list your workflows
- You should see workflow data returned as JSON
- No connection errors

---

## Troubleshooting

### Issue: "node: command not found" in WSL

**Cause:** Node.js isn't installed or not in PATH

**Solution:**
```bash
# Check if Node.js is installed
which node

# If not found, install it:
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo bash -
sudo apt-get install -y nodejs
```

---

### Issue: "Cannot connect to n8n" or API errors

**Possible causes:**
1. Wrong URL in `.env`
2. Invalid API key
3. n8n API not enabled
4. Firewall blocking your IP

**Solutions:**

**Check 1: Test n8n API manually**
```bash
# From WSL terminal
curl -H "X-N8N-API-KEY: your-api-key" https://n8n.yoursite.com/api/v1/workflows
```

Expected: JSON list of workflows
If error: Check URL, API key, or firewall

**Check 2: Verify .env file**
```bash
cd ~/n8n-mcp-server
cat .env
```

**Check 3: Enable n8n API**
- In n8n, go to Settings → API
- Make sure "API" is enabled
- Generate a fresh API key

**Check 4: Firewall/VPS settings**
- Ensure your VPS allows incoming connections on port 5678 (or 443 if using reverse proxy)
- Your home IP might need to be whitelisted

---

### Issue: TypeScript compilation errors

**Cause:** Template literals not properly escaped when creating via bash heredoc

**Solution:**
Use Claude Code's Write tool or a text editor to create `src/index.ts` instead of bash heredoc. The template literals (`${}`) can get corrupted with certain bash escaping methods.

---

### Issue: Claude Code doesn't see MCP tools

**Possible causes:**
1. Config file in wrong location
2. Syntax error in JSON
3. WSL distribution name mismatch

**Solutions:**

**Check 1: Verify config location**
```powershell
Get-Content $env:USERPROFILE\.claude\claude_desktop_config.json
```

**Check 2: Validate JSON syntax**
Use a JSON validator or:
```powershell
Get-Content $env:USERPROFILE\.claude\claude_desktop_config.json | ConvertFrom-Json
```
No error = valid JSON

**Check 3: Verify WSL distribution name**
```powershell
wsl -l -v
```
Make sure the name in `-d` parameter matches exactly (case-sensitive)

---

### Issue: CMD.EXE or UNC path errors

**Cause:** Running npm/node commands from Windows CMD instead of pure WSL environment

**Solution:**
Always run build commands from within WSL terminal:
```bash
# In WSL terminal (not Windows CMD/PowerShell)
cd ~/n8n-mcp-server
node ./node_modules/typescript/lib/tsc.js
```

---

## Lessons Learned

### ✅ Do's

1. **Use subdomains over IP addresses**
   - `https://n8n.yoursite.com` is better than `http://123.45.67.89:5678`
   - More stable, easier to remember, better for SSL

2. **Install Node.js in the correct WSL distribution**
   - Docker Desktop creates additional WSL distributions
   - Always check with `wsl -l -v` which one you're using

3. **Use login shells when PATH issues occur**
   - `bash -l` ensures environment variables are loaded
   - Helps with node/npm not found errors

4. **Test API connectivity manually first**
   - Use `curl` to verify n8n API before debugging MCP server
   - Saves time identifying where the problem is

5. **Let Claude Code create files when possible**
   - Avoids bash escaping issues with template literals
   - Use the Write tool for complex files

### ❌ Don'ts

1. **Don't use bash heredoc for files with template literals**
   - `${}` syntax gets mangled by bash variable substitution
   - Use text editors or Claude's Write tool instead

2. **Don't mix Windows and WSL node/npm**
   - Can cause PATH confusion
   - Keep builds entirely in WSL

3. **Don't forget to rebuild after code changes**
   - TypeScript needs compilation: `npm run build`
   - Changes in `src/` don't apply until built to `build/`

4. **Don't commit .env to version control**
   - Contains sensitive API keys
   - Always add to `.gitignore`

---

## Advanced: Extending the MCP Server

### Adding More n8n Operations

To add support for executions, credentials, or other n8n API endpoints:

1. Add new tool definitions in `ListToolsRequestSchema` handler
2. Add corresponding case in `CallToolRequestSchema` handler
3. Rebuild: `npm run build`
4. Restart Claude Code to pick up changes

### Example: List Executions

```typescript
// In ListToolsRequestSchema handler, add to tools array:
{
  name: "list_executions",
  description: "List workflow executions",
  inputSchema: {
    type: "object",
    properties: {
      workflowId: {
        type: "string",
        description: "Filter by workflow ID (optional)"
      }
    }
  }
}

// In CallToolRequestSchema handler, add new case:
case "list_executions": {
  const { workflowId } = request.params.arguments as { workflowId?: string };
  const url = workflowId
    ? `/executions?workflowId=${workflowId}`
    : '/executions';
  const response = await api.get(url);
  return {
    content: [
      {
        type: "text",
        text: JSON.stringify(response.data, null, 2),
      },
    ],
  };
}
```

---

## Useful Commands Reference

### WSL Commands

```bash
# Enter WSL from Windows
wsl -d Ubuntu-24.04

# List WSL distributions
wsl -l -v

# Navigate to project
cd ~/n8n-mcp-server

# Check Node.js version
node --version

# Rebuild project
npm run build

# View .env file
cat .env

# Edit .env file
nano .env
```

### Windows Commands

```powershell
# Check Claude Code config
Get-Content $env:USERPROFILE\.claude\claude_desktop_config.json

# Access WSL files from Windows
explorer.exe \\wsl.localhost\Ubuntu-24.04\root\n8n-mcp-server

# Start Claude Code
claude-code
```

---

## Security Best Practices

1. **API Key Management**
   - Store in `.env` file only
   - Never hardcode in source files
   - Rotate keys periodically

2. **Network Security**
   - Use HTTPS for n8n connection when possible
   - Consider VPN or IP whitelisting for production
   - Don't expose n8n API publicly without authentication

3. **Access Control**
   - Use read-only API keys if only listing workflows
   - Separate keys for dev/prod environments

---

## Additional Resources

- [n8n API Documentation](https://docs.n8n.io/api/)
- [Model Context Protocol Docs](https://modelcontextprotocol.io/)
- [Claude Code Documentation](https://docs.claude.com/claude-code)
- [WSL Documentation](https://learn.microsoft.com/en-us/windows/wsl/)

---

## Summary

You now have:
- ✅ Claude Code installed on Windows
- ✅ MCP server running in WSL Ubuntu
- ✅ Connection to your n8n instance
- ✅ Ability to manage n8n workflows via Claude

**Next steps:**
- Ask Claude to list your workflows
- Create new workflows via natural language
- Integrate n8n automation into your development workflow

---

**Created:** October 2025
**Tested on:** Windows 10, WSL Ubuntu 24.04, n8n self-hosted
**Claude Code Version:** Latest

---

*This guide is based on real-world implementation with actual troubleshooting scenarios encountered during setup.*
