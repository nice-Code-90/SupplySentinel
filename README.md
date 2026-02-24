# SupplySentinel

SupplySentinel is a modular procurement and inventory orchestrator designed to synchronize unstructured external supplier data (PDF, XLSX) with Microsoft Dynamics 365 Business Central.

The system is built on .NET 10 and powered by the Microsoft Agent Framework. By leveraging an agentic "brain," the system autonomously decides how to parse documents, resolve data conflicts, and synchronize with the ERP through a robust Tool-calling architecture.

## 🏗️ Architecture & Project Structure

SupplySentinel follows Clean Architecture principles, decoupled into five specialized projects to ensure provider independence and strict separation of concerns:

- **SupplySentinel.Domain**
  The core layer containing entities (Item, Vendor, PriceConflict), value objects, and Tool abstractions. It defines what the Agent can do without knowing how it does it.

- **SupplySentinel.Application**
  The agent's "brain." It hosts the AIAgent (via Microsoft Agent Framework) and orchestrates multi-turn conversations and workflows. It uses MediatR for in-process messaging.

- **SupplySentinel.Infrastructure**
  The AI implementation layer. It provides model clients for the Agent. While it currently uses Cerebras for sub-second inference, the framework allows for hot-swapping to Azure OpenAI, Anthropic, or Ollama.

- **SupplySentinel.Infrastructure.REST**
  The dedicated integration layer for D365 Business Central. It implements the RESTful Tools that the Agent calls to read/write ERP data.

- **SupplySentinel.Api**
  The host project (ASP.NET Core 10). It runs the Background Worker for proactive inventory polling and hosts SignalR Hubs for real-time feedback to the UI.

## 🧠 AI Strategy: Microsoft Agent Framework

SupplySentinel uses the Microsoft Agent Framework to manage the lifecycle of the AI assistant. This ensures the system is model-agnostic—the logic remains the same whether you use a local Llama model or a cloud-based GPT.

### Agentic Tool-Calling

The Agent is equipped with a specific set of tools defined in the Application and Infrastructure layers:

| Tool Name               | Responsibility                                               | Tech Provider       |
| :---------------------- | :----------------------------------------------------------- | :------------------ |
| **DocumentReaderTool**  | Ingests raw PDF/XLSX and extracts text streams.              | iText / OpenXML     |
| **InterpretationTool**  | Performs semantic analysis to map text to structured JSON.   | MS Agent (Cerebras) |
| **LogicalChunkingTool** | Intelligent document splitting for high-token price lists.   | .NET Logic          |
| **ERPComparisonTool**   | Fetches live master data from Business Central for auditing. | Infrastructure.REST |
| **BCDataSyncTool**      | Updates BC Price Lists and generates Purchase Orders.        | Infrastructure.REST |

## 🔄 Agentic Workflow

1. **Ingestion & Trigger**: The system detects a new file or an inventory dip (< threshold) via the Background Worker.
2. **Analysis**: The AIAgent initializes and calls the `DocumentReaderTool` and `InterpretationTool` to understand the supplier's offering.
3. **Cross-Reference**: The Agent autonomously calls the `ERPComparisonTool` to pull the "Source of Truth" from Business Central.
4. **Conflict Resolution**: If the Agent identifies a lower price or a stock emergency, it pushes a notification via SignalR.
5. **Execution**: Upon user approval, the Agent finalizes the workflow by calling the `BCDataSyncTool` to update the ERP.

## ⚡ Tech Stack

- **Framework**: .NET 10 (C# 14)
- **Agent Engine**: Microsoft Agent Framework (Public Preview 2026)
- **Inference**: Model-agnostic (Current: Cerebras gpt-oss-120b)
- **ERP Integration**: D365 Business Central API v2.0
- **Real-time UI**: SignalR
- **Frontend**: Angular 21+ 

## ⚙️ Configuration & Model Swapping

Because we use the Microsoft Agent Framework, switching the LLM provider is a simple configuration change in `Appsettings.json`.

```json
{
  "AgentConfiguration": {
    "Provider": "Cerebras",
    "Model": "llama-3.3-70b",
    "Instructions": "You are a procurement specialist..."
  },
  "BusinessCentral": {
    "TenantId": "...",
    "ClientId": "...",
    "ClientSecret": "..."
  }
}
```
