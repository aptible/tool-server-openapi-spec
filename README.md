# tool-server-openapi-spec
OpenAPI 3.0.0 spec for an Aptible tool server. A tool server exposes custom tools to Aptible AI.

The specs in this repository can generate conformant scaffolding for a tool server.

## Getting Started

Install [`openapi-generator`](https://github.com/OpenAPITools/openapi-generator?tab=readme-ov-file#1---installation)
and choose a language/framework from the list of
[supported OpenAPI server stubs](https://github.com/OpenAPITools/openapi-generator?tab=readme-ov-file#overview).

For the rest of this example, we'll assume Python with the Flask framework for concreteness and
walk through the steps to implement a "Hello, World!" tool server. Server generation in other frameworks
will involve similar steps.

First, in a directory where you want to implement the server, generate the scaffolding:

```bash
openapi-generator generate \
-i https://raw.githubusercontent.com/aptible/tool-server-openapi-spec/main/1.0.0/tool-server.yaml \
-g python-flask
```

You now need to fill in the implementation of the controllers. First, edit `openapi_server/controllers/security_controller.py`
to make your server validate the bearer authentication token that Aptible will pass for authentication. A basic
implementation that assumes you're passing the token into the server as an environment variable requires filling in a few
lines to complete the existing `info_from_bearerAuth` stub:

```python
import hmac
import os


def info_from_bearerAuth(token):
    if hmac.compare_digest(token, os.environ['TOOL_SERVER_TOKEN']):
        return {}  # Success
    return None  # Failure
```

Next, edit `openapi_server/controllers/security_controller.py` to add an implementation of the `get_tools` function:

```python
from openapi_server.models.tool import Tool
from openapi_server.models.tools_list import ToolsList

def get_tools():
    return ToolsList(
        [
            Tool(
                tool_id="hello_world",
                description="Returns the string \"Hello, World!\" when you pass JSON input { \"name\": \"World\" }."
                            "Pass a different name for a different greeting.",
                human_short_title="Hello World Generator",
                human_description="Generate a personalized greeting by passing a JSON object with a \"name\" key."
            )
        ]
    )
```

and the `call_tool` function:

```python
from openapi_server.models.tool_call_result import ToolCallResult

def call_tool(tool_id, body):
    if tool_id == "hello_world":
        name = body['name']
        return ToolCallResult({"summary": f"Hello, {name}!"})
    else:
        raise NotFound(f"No tool registered with ID {tool_id}")
```

Follow instructions in the generated `README.md` to install dependencies, set a `TOOL_SERVER_TOKEN` for testing in your shell environment, and start your server.
If you're testing locally, you should now be able to hit the tool server endpoints and see everything working:

```bash
$ curl http://localhost:8080/tools -H "Authorization: Bearer ${TOOL_SERVER_TOKEN}"
{
  "tools": [
    {
      "description": "Returns the string \"Hello, World!\" when you pass JSON input { \"name\": \"World\" }. Pass a different name for a different greeting.",
      "human_description": "Generate a personalized greeting by passing a JSON object with a \"name\" key.",
      "human_short_title": "Hello World Generator",
      "tool_id": "hello_world"
    }
  ]
}
```

```bash
$ curl -X POST http://localhost:8080/tool/hello_world -H "Authorization: Bearer ${TOOL_SERVER_TOKEN}" -H "Content-Type: application/json" -d '{"name": "World"}'
{
  "result": {
    "summary": "Hello, World!"
  }
}
```

Now add some real tools and run the server behind a HTTPS endpoint to serve your tools in production.
