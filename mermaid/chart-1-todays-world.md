# Chart 1: Today's World (On-Prem Agents Owned by Convergint)

```mermaid
---
config:
  theme: base
  layout: dagre
  look: neo
---
flowchart LR
  subgraph customerSite [Customer Site]
    securitySystem[Security System]
    convergintAgent[Convergint Agent]
    securitySystem <--> convergintAgent
  end

  subgraph convergintCloud [Convergint Cloud]
    inventoryReceiver[Inventory Receiver]
    eventReceiver[Event Receiver]
  end

  convergintAgent -->|inventory| inventoryReceiver
  convergintAgent -->|events| eventReceiver
```
