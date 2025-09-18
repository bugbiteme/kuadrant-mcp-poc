# kuadrant-mcp-poc

### Introduction:
#test
This POC used the [everything](https://github.com/modelcontextprotocol/servers/tree/main/src/everything) MCP server as it has Streamable-HTTP capabilities

## Prerequisties

- K8s cluster
- Kuadrant installed, for more info see Kuadrant [docs](https://docs.kuadrant.io/1.2.x/install-helm/)

### Set the environment variables

```sh
export QUAY_USERNAME=xxxx # Quay username
export KUADRANT_AWS_ACCESS_KEY_ID=xxxx # AWS Key ID with access to manage the DNS Zone ID below
export KUADRANT_AWS_SECRET_ACCESS_KEY=xxxx # AWS Secret Access Key with access to manage the DNS Zone ID below
```

## Installation
1. Update the `mcp-server/mcp-server-everything.yaml` file with your root domain.

```yaml

1. Install the MCP Server in your K8s cluster.

```sh
   kubectl apply -f mcp-server/mcp-server-everything.yaml
```

1. Ensure the MCP everything Server pod is running.

```sh
   kubectl get pods -n mcp-server
```

1. Create the gateway namespace:

```sh
    kubectl create ns mcp-gateway
```

1. Create the secret credentials in the same namespace as the Gateway - these will be used to configure DNS:

```sh
   kubectl -n mcp-gateway create secret generic aws-credentials \
   --type=kuadrant.io/aws \
   --from-literal=AWS_ACCESS_KEY_ID=$KUADRANT_AWS_ACCESS_KEY_ID \
   --from-literal=AWS_SECRET_ACCESS_KEY=$KUADRANT_AWS_SECRET_ACCESS_KEY
```

1. Create the secret credentials in the cert-manager namespace:

```sh

   kubectl -n cert-manager create secret generic aws-credentials \
   --type=kuadrant.io/aws \
   --from-literal=AWS_ACCESS_KEY_ID=$KUADRANT_AWS_ACCESS_KEY_ID \
   --from-literal=AWS_SECRET_ACCESS_KEY=$KUADRANT_AWS_SECRET_ACCESS_KEY
```

1. Create the Kuadrant Gateway NOTE Update the `kuadrant/gateway.yaml` file with your root domain.

```sh
    kubectl apply -f kuadrant/gateway.yaml
```

1. Create the Lets encrypt Cluster issuer and TlS Policy:

```sh
  kubectl apply -f kuadrant/tls.yaml
```

    **Note**: The cert can't be self signed. MCP clients dont accept self signed certs yet.

1. Create the DNSPolicy:

```sh
   kubectl apply -f kuadrant/dns.yaml
```

1. After a minute, Test that the MCP server is responding

```sh
curl -v -X POST "https://api.KUADRANT_ZONE_ROOT_DOMAIN_GOES_HERE/mcp" \
      -H "Content-Type: application/json" \
      -H "Accept: application/json, text/event-stream" \
      -d '{
            "jsonrpc": "2.0",
            "method": "initialize",
            "params": {
              "protocolVersion": "2024-11-05",
              "capabilities": {
                "resources": {},
                "tools": {},
                "prompts": {}
              },
              "clientInfo": {
                "name": "test-curl-client",
                "version": "1.0"
              }
            },
            "id": 1
          }'
```

1. Create the Rate limit policy:

```sh
    kubectl apply -f kuadrant/rlp.yaml
```

1. Create the Auth policy and the API Key secrets:

```sh
   kubectl apply -f kuadrant/auth.yaml
```

### MCP Client

In this example we are using Vscode Co pilot as our MCP client as it has capabilities to connect using Streamable HTTP

1. Add the following to your VSCode setting.json. For more info please see the official VSCode MCP [docs](https://code.visualstudio.com/docs/copilot/chat/mcp-servers#_add-an-mcp-server-to-your-user-settings)

```
"mcp": {
    "servers": {
      "my-mcp-server": {
        "type": "http",
        "url": "https://KUADRANT_ZONE_ROOT_DOMAIN_GOES_HERE/mcp",
        "headers": {
          "Authorization": "APIKEY IAMALICE"
        }
      }
    }
  },
```

2. The Client should be able to use the server now and you should be able to avail of the MCP server tools. Open co pilot in vscode using `⌃⌘I` and in the drop down click agent.

3. In the chat type:

```sh
    add 500 and 500
```

4. Vscode will ask you about using allowing Co pilot invoke the tool from the MCP server, ensure you allow it.

5. The chat will output the answer and also show you it ran `add` from the MCP server

6. If you leave out the Auth header in the config or ask the chat questions using the tool to many times to quickly, the auth and rate limit policies deployed will deny and limit the requests.
