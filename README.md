# Mistral Document AI with Microsoft Foundry

This project demonstrates how to send local documents to a Mistral Document AI deployment in Microsoft Foundry using a Python notebook.

Microsoft Entra ID is the recommended authentication method. An optional API-key example is also included for environments where key authentication is enabled.

## Prerequisites

- [uv](https://docs.astral.sh/uv/getting-started/installation/)
- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli)
- Access to the Microsoft Foundry resource and Mistral OCR deployment
- The **Cognitive Services User** role on the Foundry resource

The project uses Python 3.12. `uv` automatically selects the installed Python version defined in `.python-version`.

## 1. Install dependencies

From the project directory, run:

```powershell
uv sync
```

This creates and manages a project-local `.venv`. You do not need to activate it.

Dependencies are restored through Microsoft's Python package feed proxy, configured in `pyproject.toml`.

## 2. Configure the deployment

Create your local environment file:

```powershell
Copy-Item .env.example .env
```

Open `.env` and configure:

```dotenv
AZURE_FOUNDRY_ENDPOINT=https://deploy-resource.services.ai.azure.com
AZURE_MISTRAL_OCR_DEPLOYMENT=mistral-document-ai-2512
```

The notebook derives the OCR request URL by appending `/providers/mistral/azure/ocr` to the Foundry resource endpoint. Confirm that the deployment name matches the name shown in Microsoft Foundry.

Do not commit `.env`; it is excluded by `.gitignore`.

## 3. Sign in to Azure

The notebook uses `DefaultAzureCredential` for passwordless local authentication:

```powershell
az login
```

If you have access to multiple subscriptions, select the appropriate one:

```powershell
az account set --subscription "<subscription name or ID>"
```

Your signed-in identity must have the **Cognitive Services User** role on the Foundry resource. New role assignments may take several minutes to propagate.

## 4. Add a document

Place the document to process in the `documents` directory. Customer documents are excluded from source control by default.

Update `DOCUMENT_PATH` in the notebook:

```python
DOCUMENT_PATH = Path("documents/your-document.pdf")
```

Supported formats depend on the deployment and include common document and image formats such as PDF, DOCX, PPTX, PNG, and JPEG.

## 5. Start the notebook

Launch JupyterLab through the managed environment:

```powershell
uv run jupyter lab
```

Open `mistral_ocr_demo.ipynb` and run the cells in order. Uncomment the OCR request and result-display lines after the endpoint and document are configured.

## Run through Azure AI Gateway

Use `mistral_ocr_ai_gateway_demo.ipynb` when the Foundry deployment is exposed through Azure API Management (APIM).

You need:

1. An APIM API whose base URL includes an API path suffix and whose operation forwards `/providers/mistral/azure/ocr` to the Foundry resource.
2. An APIM subscription with access to that API or its product.
3. One of these APIM-to-Foundry authentication configurations:
   - **Bearer-token passthrough:** APIM preserves the client's `Authorization` header. The local user needs the **Cognitive Services User** role on the Foundry resource.
   - **Gateway managed identity:** APIM applies an `authentication-managed-identity` policy for `https://ai.azure.com`. The APIM identity needs the **Cognitive Services User** role on the Foundry resource.

Add these settings to `.env`:

```dotenv
AZURE_AI_GATEWAY_ENDPOINT=https://<gateway>.azure-api.net/<api-path>
AZURE_AI_GATEWAY_SUBSCRIPTION_KEY=<APIM subscription key>
AZURE_AI_GATEWAY_KEY_HEADER=Ocp-Apim-Subscription-Key
AZURE_AI_GATEWAY_AUTH_MODE=gateway_managed_identity
```

`AZURE_AI_GATEWAY_ENDPOINT` must include the APIM API path suffix, but not `/providers/mistral/azure/ocr`; the notebook appends that operation path.

For Mistral OCR, point `AZURE_AI_GATEWAY_ENDPOINT` at the custom APIM API created for the provider-specific OCR route, such as `https://<gateway>.azure-api.net/mistral-ocr`. Do not use the Foundry-generated project API route, because it targets the standard Foundry model API surface rather than `/providers/mistral/azure/ocr`.

The notebook sends the APIM key in the header configured by `AZURE_AI_GATEWAY_KEY_HEADER`. Custom APIM APIs normally use `Ocp-Apim-Subscription-Key`; use the `name` shown in a `401` response's `WWW-Authenticate` header if your API differs. A gateway-generated `401 Access Denied` usually means that the key is missing, sent under the wrong header, invalid, or not authorized for the API. Do not place the key directly in the notebook or commit `.env`.

For staged troubleshooting, open `mistral_ocr_ai_gateway_diagnostics.ipynb` and run one checkpoint at a time. It isolates local configuration, DNS/TLS, APIM subscription access, backend authentication, direct Foundry access, and the real OCR payload without displaying credentials.

## Optional API-key authentication

API-key authentication is disabled on the included Foundry resource. To demonstrate it against a resource where keys are enabled, add the key to `.env`:

```dotenv
AZURE_MISTRAL_OCR_API_KEY=<API key>
```

Then replace `entra_headers()` with `api_key_headers()` in the request cell. Never put an API key directly in the notebook or commit it to source control.
