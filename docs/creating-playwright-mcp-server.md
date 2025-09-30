# How to Create a Playwright MCP Server for Automation

This guide walks you through creating a Model Context Protocol (MCP) server that uses Playwright for browser automation. By the end, you'll have a working MCP server that can automate web interactions.

## Overview

A Playwright MCP server enables AI assistants to:
- Navigate web pages
- Click elements and fill forms
- Take screenshots
- Extract content from pages
- Run automated tests
- Perform complex web scraping tasks

## Prerequisites

- Node.js 18 or higher
- TypeScript knowledge
- Basic understanding of Playwright
- Familiarity with the Model Context Protocol

## Project Setup

### 1. Initialize Your Project

```bash
mkdir playwright-mcp-server
cd playwright-mcp-server
npm init -y
```

### 2. Install Dependencies

```bash
# MCP SDK
npm install @modelcontextprotocol/sdk

# Playwright for browser automation
npm install playwright

# Development dependencies
npm install -D typescript @types/node
npx playwright install
```

### 3. Configure TypeScript

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### 4. Update package.json

Add the following to your `package.json`:

```json
{
  "name": "playwright-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "bin": {
    "playwright-mcp-server": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "watch": "tsc --watch",
    "prepare": "npm run build"
  }
}
```

## Implementation

### Create the Server

Create `src/index.ts`:

```typescript
#!/usr/bin/env node

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
} from "@modelcontextprotocol/sdk/types.js";
import { chromium, Browser, Page } from "playwright";

// Browser management
let browser: Browser | null = null;
let page: Page | null = null;

async function ensureBrowser() {
  if (!browser) {
    browser = await chromium.launch({ headless: true });
  }
  if (!page) {
    page = await browser.newPage();
  }
  return { browser, page };
}

async function cleanup() {
  if (page) {
    await page.close();
    page = null;
  }
  if (browser) {
    await browser.close();
    browser = null;
  }
}

// Define available tools
const TOOLS: Tool[] = [
  {
    name: "playwright_navigate",
    description: "Navigate to a URL",
    inputSchema: {
      type: "object",
      properties: {
        url: {
          type: "string",
          description: "The URL to navigate to",
        },
      },
      required: ["url"],
    },
  },
  {
    name: "playwright_screenshot",
    description: "Take a screenshot of the current page",
    inputSchema: {
      type: "object",
      properties: {
        name: {
          type: "string",
          description: "Name for the screenshot",
        },
        fullPage: {
          type: "boolean",
          description: "Capture full page (default: true)",
          default: true,
        },
      },
      required: ["name"],
    },
  },
  {
    name: "playwright_click",
    description: "Click an element on the page",
    inputSchema: {
      type: "object",
      properties: {
        selector: {
          type: "string",
          description: "CSS selector for the element to click",
        },
      },
      required: ["selector"],
    },
  },
  {
    name: "playwright_fill",
    description: "Fill out an input field",
    inputSchema: {
      type: "object",
      properties: {
        selector: {
          type: "string",
          description: "CSS selector for the input field",
        },
        value: {
          type: "string",
          description: "Value to fill",
        },
      },
      required: ["selector", "value"],
    },
  },
  {
    name: "playwright_evaluate",
    description: "Execute JavaScript in the page context",
    inputSchema: {
      type: "object",
      properties: {
        script: {
          type: "string",
          description: "JavaScript code to execute",
        },
      },
      required: ["script"],
    },
  },
  {
    name: "playwright_get_content",
    description: "Get the HTML content of the current page",
    inputSchema: {
      type: "object",
      properties: {},
    },
  },
];

// Create the MCP server
const server = new Server(
  {
    name: "playwright-mcp-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// Handle tool listing
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: TOOLS,
}));

// Handle tool execution
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    const { page } = await ensureBrowser();

    switch (name) {
      case "playwright_navigate": {
        const { url } = args as { url: string };
        await page.goto(url, { waitUntil: "networkidle" });
        return {
          content: [
            {
              type: "text",
              text: `Successfully navigated to ${url}`,
            },
          ],
        };
      }

      case "playwright_screenshot": {
        const { name: screenshotName, fullPage = true } = args as {
          name: string;
          fullPage?: boolean;
        };
        const screenshot = await page.screenshot({
          fullPage,
          type: "png",
        });
        return {
          content: [
            {
              type: "image",
              data: screenshot.toString("base64"),
              mimeType: "image/png",
            },
            {
              type: "text",
              text: `Screenshot taken: ${screenshotName}`,
            },
          ],
        };
      }

      case "playwright_click": {
        const { selector } = args as { selector: string };
        await page.click(selector);
        return {
          content: [
            {
              type: "text",
              text: `Clicked element: ${selector}`,
            },
          ],
        };
      }

      case "playwright_fill": {
        const { selector, value } = args as {
          selector: string;
          value: string;
        };
        await page.fill(selector, value);
        return {
          content: [
            {
              type: "text",
              text: `Filled ${selector} with: ${value}`,
            },
          ],
        };
      }

      case "playwright_evaluate": {
        const { script } = args as { script: string };
        const result = await page.evaluate(script);
        return {
          content: [
            {
              type: "text",
              text: `Evaluation result: ${JSON.stringify(result)}`,
            },
          ],
        };
      }

      case "playwright_get_content": {
        const content = await page.content();
        return {
          content: [
            {
              type: "text",
              text: content,
            },
          ],
        };
      }

      default:
        return {
          content: [
            {
              type: "text",
              text: `Unknown tool: ${name}`,
            },
          ],
          isError: true,
        };
    }
  } catch (error) {
    return {
      content: [
        {
          type: "text",
          text: `Error: ${error instanceof Error ? error.message : String(error)}`,
        },
      ],
      isError: true,
    };
  }
});

// Run the server
async function runServer() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  
  console.error("Playwright MCP Server running on stdio");

  // Cleanup on exit
  process.on("SIGINT", async () => {
    await cleanup();
    process.exit(0);
  });
}

runServer().catch((error) => {
  console.error("Fatal error running server:", error);
  process.exit(1);
});
```

## Building and Testing

### 1. Build the Server

```bash
npm run build
```

### 2. Test Locally

Create a test script `test.sh`:

```bash
#!/bin/bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node dist/index.js
```

Run it:

```bash
chmod +x test.sh
./test.sh
```

## Configuration

### Claude Desktop

Add to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "node",
      "args": ["/path/to/your/playwright-mcp-server/dist/index.js"]
    }
  }
}
```

### VS Code

Add to `.vscode/mcp.json` in your workspace:

```json
{
  "servers": {
    "playwright": {
      "command": "node",
      "args": ["/path/to/your/playwright-mcp-server/dist/index.js"]
    }
  }
}
```

## Advanced Features

### Adding More Tools

You can extend the server with additional Playwright functionality:

```typescript
{
  name: "playwright_wait_for_selector",
  description: "Wait for an element to appear",
  inputSchema: {
    type: "object",
    properties: {
      selector: {
        type: "string",
        description: "CSS selector to wait for",
      },
      timeout: {
        type: "number",
        description: "Timeout in milliseconds",
        default: 30000,
      },
    },
    required: ["selector"],
  },
}
```

### Error Handling Best Practices

1. **Timeout Handling**: Set reasonable timeouts for all operations
2. **Selector Validation**: Validate selectors before use
3. **Resource Cleanup**: Always clean up browser resources
4. **Detailed Error Messages**: Provide context in error messages

### Performance Optimization

1. **Browser Reuse**: Keep browser instance alive between requests
2. **Page Pooling**: Maintain a pool of pages for concurrent operations
3. **Selective Screenshots**: Only capture full page when necessary
4. **Network Optimization**: Use appropriate wait conditions

## Publishing

### 1. Prepare for Publishing

Update your `package.json`:

```json
{
  "name": "@yourorg/playwright-mcp-server",
  "version": "1.0.0",
  "description": "Playwright MCP server for browser automation",
  "keywords": ["mcp", "playwright", "automation"],
  "repository": {
    "type": "git",
    "url": "https://github.com/yourorg/playwright-mcp-server"
  }
}
```

### 2. Publish to npm

```bash
npm publish --access public
```

### 3. Add to MCP Servers List

Create a pull request to add your server to the [MCP Servers README](https://github.com/modelcontextprotocol/servers).

## Example Use Cases

### Web Scraping

```typescript
// Navigate to a page
await playwright_navigate({ url: "https://example.com" })

// Extract data
await playwright_evaluate({
  script: "Array.from(document.querySelectorAll('h2')).map(h => h.textContent)"
})
```

### Form Automation

```typescript
// Fill out a form
await playwright_fill({ selector: "#email", value: "user@example.com" })
await playwright_fill({ selector: "#password", value: "secret" })
await playwright_click({ selector: "button[type='submit']" })
```

### Visual Testing

```typescript
// Take before screenshot
await playwright_screenshot({ name: "before" })

// Make changes
await playwright_click({ selector: "#theme-toggle" })

// Take after screenshot
await playwright_screenshot({ name: "after" })
```

## Resources

- [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
- [Playwright Documentation](https://playwright.dev/)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Official MCP Servers Repository](https://github.com/modelcontextprotocol/servers)
- [Microsoft Playwright MCP Server](https://github.com/microsoft/playwright-mcp) - Official implementation

## Troubleshooting

### Browser Won't Launch

```bash
# Install browser binaries
npx playwright install chromium
```

### Permission Errors

Make sure your script is executable:

```bash
chmod +x dist/index.js
```

### Module Import Errors

Ensure `"type": "module"` is set in package.json and TypeScript is configured for ESM.

## Next Steps

1. Add more Playwright tools (hover, drag-and-drop, file uploads)
2. Implement resource providers for saved screenshots
3. Add prompts for common automation workflows
4. Create configuration options for browser preferences
5. Add support for multiple browsers (Firefox, Safari)
6. Implement session management for persistent automation

## License

Most MCP servers use the MIT License. Choose an appropriate license for your project.
