## coze.cn Cloud YAML Format

YAML type: `code` | Status: YAML shape unverified; runtime constraints verified (2026-10-26)

> The cloud YAML shape below is based on source code analysis.
> For a verified YAML shape, export a real workflow from coze.cn containing this node.
> Runtime and sandbox constraints are verified against the official docs; see
> "Runtime & Sandbox Constraints" below.

```yaml
    - id: "200001"
      type: code
      title: 代码节点
      position:
        x: 0
        y: 0
      parameters:
        node_inputs:
            - name: input
              input:
                type: string
                value:
                    path: output
                    ref_node: "100001"
        node_outputs:
            output:
                type: string
                value: null
```

---

# Code Node (type: "5")

## Purpose
Executes user-written code (Python or JavaScript/TypeScript) with defined input parameters and structured output variables.

## Runtime & Sandbox Constraints (coze.cn)

> Verified against the official Coze docs on 2026-10-26:
> https://docs.coze.cn/guides_code_node
>
> This section documents the runtime; the cloud YAML shape above is still
> unverified. Do not treat the YAML example as a confirmed export.

| Item | Python | JavaScript / TypeScript |
|---|---|---|
| Runtime | CPython **3.11.3** standard library | V8 **11.3.244.8** (Node.js **20.3.1**), ECMAScript 2022 |
| Allowed third-party deps | `requests_async`, `numpy` only | `dayjs` **1.8.36**, `lodash` **4.17.20** only |
| Network | HTTP/HTTPS only; `http.client` unsupported; use `requests_async` | HTTP/HTTPS only |
| Node timeout | 60s default, configurable 0.1s-60s | same |
| Retry | 0 or 1 | same |
| Code nodes per workflow | max **50** | same |
| Function limit | **one function only**; must return an object | same |
| Sleep | prefer `asyncio.sleep()`; `time.sleep()` blocks and may trigger throttling; max delay 1 minute | use `setTimeout` / async APIs |
| Error outputs | `isSuccess`, `errorBody` when using fallback / exception branch | same |

### JavaScript runtime

- V8 11.3.244.8 / Node.js 20.3.1, ECMAScript 2022.
- Only `dayjs` and `lodash` are available as third-party modules.
- WinterCG-style Web APIs include `atob`, `btoa`, `console`, `setTimeout`,
  `clearTimeout`, `structuredClone`, `URL`, `URLSearchParams`,
  `AbortController`, `AbortSignal`, `TextEncoder`, `TextDecoder`, `WebStreams`,
  and `WebCrypto` (AES/HMAC/SHA only).

### Behavior notes

- The platform explicitly says **do not write multiple functions**; even a
  single output must be returned as an object.
- The code-node service was upgraded on 2025-09-16. Run the node before
  publishing; do not publish untested code.
- Official docs do not state memory, CPU, filesystem, environment-variable, or
  output-size limits. Treat those as unknown and probe them if needed.

### Sources

- https://docs.coze.cn/guides_code_node
- https://docs.coze.cn/api/open/docs/guides/code_node

## Core Fields
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `inputs.code` | `string` | Yes | The source code to execute. |
| `inputs.language` | `int64` | Yes | Programming language: `3` = Python, `5` = JavaScript/TypeScript. |
| `inputs.inputParameters` | `InputValueDTO[]` | No | Input variables passed to the code as function arguments. |
| `outputs` | `VariableMetaDTO[]` | Yes | Output variable definitions. The code must return an object matching these fields. |
| `nodeMeta` | `NodeMetaFE` | No | Frontend display metadata. |

## Full Schema

### Frontend FormData (`code/types.ts`)
```typescript
interface FormData {
  inputParameters: InputValueVO[];
  nodeMeta: NodeDataDTO['nodeMeta'];
  outputs: OutputValueVO[];
  codeParams: CodeEditorValue;       // { code: string; language: LanguageEnum }
}
```

### Frontend CodeEditorValue
```typescript
interface CodeEditorValue {
  code: string;
  language: LanguageEnum;  // 3 = Python, 5 = TypeScript
}
```

### Language Enum (`code/constants.ts`)
| Value | Language | Display Name |
|-------|----------|-------------|
| `3` | Python | Python |
| `5` | TypeScript | JavaScript |

Note: The open-source version only supports Python (backend limitation).

### Backend DTO (submitted format from `transformOnSubmit`)
```typescript
{
  nodeMeta: NodeMetaFE;
  inputs: {
    inputParameters: InputValueDTO[];
    code: string;
    language: LanguageEnum;
  };
  outputs: VariableMetaDTO[];
}
```

### Backend Go Config (`code/code.go`)
```go
type Config struct {
    Code     string
    Language coderunner.Language   // "python" or "javascript"
}
```

### Backend Language Mapping
| Frontend Value | Backend Value | coderunner.Language |
|----------------|--------------|---------------------|
| `3` | Python | `"python"` |
| `5` | JavaScript | `"javascript"` |

### Backend Go Runner (`code/code.go`)
```go
type Runner struct {
    outputConfig map[string]*vo.TypeInfo
    code         string
    language     coderunner.Language
    runner       coderunner.Runner
    importError  error
}
```

### Canvas VO (`vo/canvas.go`)
```go
type CodeRunner struct {
    Code     string `json:"code"`
    Language int64  `json:"language"`
}
```

### Frontend Constants (`code/constants.ts`)
| Constant | Value | Description |
|----------|-------|-------------|
| `INPUT_PATH` | `"inputParameters"` | Form path for input parameters |
| `CODE_PATH` | `"codeParams"` | Form path for code editor value |
| `OUTPUT_PATH` | `"outputs"` | Form path for output variables |

### Default Values
**Default Inputs:**
```typescript
[{ name: 'input' }]
```

**Default Outputs:**
```typescript
[
  { name: 'key0', type: ViewVariableType.String },        // 1
  { name: 'key1', type: ViewVariableType.ArrayString },    // 99
  {
    name: 'key2', type: ViewVariableType.Object,           // 6
    children: [
      { name: 'key21', type: ViewVariableType.String },    // 1
    ],
  },
]
```

**Default Python Code Template:**
```python
async def main(args: Args) -> Output:
    params = args.params
    # your logic here
    ret: Output = {
        "key0": "value0",
        "key1": ["value1"],
        "key2": {
            "key21": "value21"
        }
    }
    return ret
```

**Default TypeScript Code Template:**
```typescript
async function main({ params }: Args): Promise<Output> {
    const ret = {
        "key0": "value0",
        "key1": ["value1"],
        "key2": {
            "key21": "value21"
        }
    };
    return ret;
}
```

### Python Import Validation
The backend validates Python imports at build time:
- **Allowed**: Python 3.11.3 standard library modules, except the blacklist below.
- **Blacklisted standard library modules**: `curses`, `dbm`, `ensurepip`, `fcntl`, `grp`, `idlelib`, `lib2to3`, `msvcrt`, `pwd`, `resource`, `syslog`, `termios`, `tkinter`, `turtle`, `turtledemo`, `venv`, `winreg`, `winsound`, `multiprocessing`, `threading`, `socket`, `pty`, `tty`.
- **Third-party modules**: only `requests_async` and `numpy` are available. Other third-party imports are not supported.
- **HTTP client**: `http.client` is explicitly unsupported; use `requests_async`. Because `urllib.request` depends on `http.client`, treat it as unavailable unless verified in the current runtime.

## Node Registry Metadata
| Property | Value |
|----------|-------|
| `type` | `StandardNodeType.Code` = `"5"` |
| `nodeDTOType` | `"5"` |
| `size` | default node size, styled with `width: 484` |
| `inputParametersPath` | `"inputParameters"` |
| `outputsPath` | default outputs path |
| Backend NodeType | `"CodeRunner"` (ID: 5) |

## Variable Reference Rules
- Input variables are defined in `inputs.inputParameters` and passed to the code function as `params` (Python: `args.params`, TypeScript: `{ params }`).
- Each input maps to an upstream node output via a reference expression.
- Output variables are defined in `outputs` and must match the return value structure of the code.
- Downstream nodes reference outputs as `{{node_id.variable_name}}`.
- For Object types, nested fields can be referenced as `{{node_id.variable_name.child_name}}`.

## Example Snippet
```json
{
  "id": "node_xyz",
  "type": "5",
  "data": {
    "nodeMeta": {
      "title": "Code"
    },
    "inputs": {
      "inputParameters": [
        {
          "name": "input",
          "input": {
            "type": "ref",
            "content": {
              "sourceNodeId": "0",
              "outputKey": "user_input"
            }
          }
        }
      ],
      "code": "async def main(args: Args) -> Output:\n    params = args.params\n    ret: Output = {\n        \"result\": params[\"input\"].upper()\n    }\n    return ret",
      "language": 3
    },
    "outputs": [
      {
        "name": "result",
        "type": 1
      }
    ]
  }
}
```
