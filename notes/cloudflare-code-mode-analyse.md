# Analyse: Cloudflare Code Mode MCP – Lohnt sich ein ähnliches Muster?

> Erstellt: 2026-03-08
> Anlass: mcp-stack und mcp-proxmox wachsen stetig – Skalierbarkeit evaluieren

---

## Was ist Cloudflare Code Mode?

Cloudflare hat für ihren MCP-Server (der die gesamte Cloudflare API mit ~2.500 Endpoints abdeckt) ein neues Muster eingeführt: **Code Mode**.

Anstatt jeden Endpoint als einzelnes Tool zu exponieren, gibt es nur **zwei Tools**:

- `search(query)` – durchsucht die OpenAPI-Spec, gibt relevante Teile zurück (ohne alles in den Kontext zu laden)
- `execute(code)` – führt JavaScript in einer Sandbox aus, das gegen die typisierte API schreibt

**Ergebnis:** Token-Verbrauch sinkt von ~1,17 Mio. auf ~1.000 Token pro Session – eine Reduktion von **99,9%**.

### Kernidee

```
Traditionell:   Agent → list_dns_records() → create_dns_record() → ...
Code Mode:      Agent → execute("const records = await cf.dns.list(...); ...")
```

Das *Wissen über die API* ist selbst das Tool – der Agent lernt zur Laufzeit durch `search()`, was möglich ist.

---

## Lohnt sich das für unseren lokalen MCP-Stack?

**Kurzantwort: Vollständiges Code Mode nicht – aber die modularen Prinzipien schon.**

Code Mode ist primär für den Fall optimiert:
- Einzelne API mit 1000+ Endpoints in einem MCP-Server
- Token-Kosten sind kritisch
- Code-Ausführung in einer Sandbox ist sicher

Unser Stack hat andere Anforderungen: viele kleine, spezialisierte Server statt einem Monolithen.

---

## Was sich ab sofort lohnt

### 1. FastMCP 3.0 Server-Komposition (geringer Aufwand)

Statt alle Tools in einer Datei: Sub-Server pro Domäne, zusammenmontiert zu einem Haupt-Server.

```python
from fastmcp import FastMCP

vm_server = FastMCP("vms")
storage_server = FastMCP("storage")
network_server = FastMCP("network")

main = FastMCP("proxmox")
main.mount("vm", vm_server)       # → vm_list, vm_start, vm_stop
main.mount("storage", storage_server)
main.mount("network", network_server)
```

Vorteil: Klare Namespaces, unabhängig erweiterbar, kein Naming-Konflikt.

### 2. Tag-basiertes Enable/Disable (geringer Aufwand)

Destruktive Tools standardmäßig deaktivieren:

```python
@mcp.tool(tags={"destructive"})
def delete_vm(vmid: int): ...

mcp.disable(tags={"destructive"})  # Standardmäßig aus
mcp.enable(tags={"destructive"})   # Explizit freischalten
```

Besonders sinnvoll für `mcp-proxmox` mit 88 Tools – viele davon sind destruktiv.

### 3. Lightweight Search-Tool (wenn >200 Tools)

Wenn der Stack auf 200+ Tools anwächst: Ein Meta-Tool `search_tools(query)` das Tool-Metadaten durchsucht, bevor der Agent einen Tool-Call macht. Kein Sandbox-Overhead, aber ähnliche Kontext-Effizienz.

---

## Konkrete Empfehlung für unsere Projekte

| Projekt | Status | Empfehlung |
|---|---|---|
| `mcp-proxmox` (88 Tools) | Wächst | Server-Komposition: VMs / Storage / Network / Cluster als Module |
| `mcp-stack` (Gateway) | Stabil | Kein Handlungsbedarf – bereits gut strukturiert |
| Zukünftige Server | – | Von Anfang an mit FastMCP 3.0 und Komposition starten |

**Priorisierung:**
1. Bei `mcp-proxmox`: Server-Komposition umsetzen (mittlerer Aufwand, hoher Nutzen)
2. Destruktive Tools mit Tags markieren und standardmäßig deaktivieren
3. Kein vollständiges Code Mode – zu komplex für den Nutzen

---

## Quellen

- [Cloudflare Blog: Code Mode – give agents an entire API in 1,000 tokens](https://blog.cloudflare.com/code-mode-mcp/)
- [Cloudflare Blog: Code Mode – the better way to use MCP](https://blog.cloudflare.com/code-mode/)
- [FastMCP Server Composition Docs](https://gofastmcp.com/servers/composition)
- [FastMCP 3.0 – What's New](https://www.jlowin.dev/blog/fastmcp-3-whats-new)
- [Dynamic Tool Discovery (Speakeasy)](https://www.speakeasy.com/mcp/tool-design/dynamic-tool-discovery)
