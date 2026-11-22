# Selected code

Short excerpts from the private CARA-OT source code: 182 of about 3,400 lines of Python, copied without changes. `# ...` marks lines left out. The full source is available on request.

| Excerpt | File |
|---|---|
| [Parser plugin interface](#1-parser-plugin-interface) | cara_ot/parsers/base.py |
| [Semantic diff](#2-semantic-diff) | cara_ot/semantic.py |
| [Cross-artifact safety enrichment](#3-cross-artifact-safety-enrichment) | cara_ot/semantic.py |
| [Impact propagation](#4-impact-propagation) | cara_ot/impact.py |
| [Change correlation](#5-change-correlation) | cara_ot/correlate.py |
| [Recovery assurance](#6-recovery-assurance) | cara_ot/recovery.py |
| [LLM grounding checker](#7-llm-grounding-checker) | cara_ot/explain.py |

## 1. Parser plugin interface

Every file format is a plugin that turns an exported artifact into normalised entities and typed relations. Adding a vendor format means adding one class; the engines never change.

`cara_ot/parsers/base.py` · lines 1–6, 13–44

```python
"""Plugin base class for artifact parsers.

To support a new vendor or file format, subclass ``ArtifactParser``, set
``kind`` and register it with ``@register``. Nothing else in CARA-OT needs to
change: every later engine only sees the normalised entities and relations.
"""
# ...
REGISTRY: dict[str, "ArtifactParser"] = {}


def register(cls):
    REGISTRY[cls.kind] = cls()
    return cls


@dataclass
class ParseResult:
    entities: dict[str, Entity] = field(default_factory=dict)
    relations: list[Relation] = field(default_factory=list)
    errors: list[str] = field(default_factory=list)

    def add(self, ent: Entity) -> Entity:
        self.entities[ent.id] = ent
        return ent

    def link(self, src: str, rel: str, dst: str, note: str = "") -> None:
        self.relations.append(Relation(src, rel, dst, note))


class ArtifactParser:
    kind: str = ""
    description: str = ""

    def parse(self, text: str, asset: str, inventory: Inventory, path: str) -> ParseResult:
        raise NotImplementedError

    # Each parser can describe what a change to one of its entities means.
    def classify(self, change) -> None:  # pragma: no cover - default no-op
        return None
```

## 2. Semantic diff

Files are compared by meaning, not by text: two artifact versions are parsed and their entities compared attribute by attribute, so comments and formatting produce no change.

`cara_ot/semantic.py` · lines 74–88

```python
        ea, eb = a.entities, b.entities
        for eid in sorted(set(ea) | set(eb)):
            old, new = ea.get(eid), eb.get(eid)
            if old and new:
                sa, sb = old.semantic_attrs(), new.semantic_attrs()
                for attr in sorted(set(sa) | set(sb)):
                    if sa.get(attr) != sb.get(attr):
                        _emit(changes, a, b, b.kind, "modified", old, new, attr, sa.get(attr), sb.get(attr),
                              cur.inventory)
            elif new:
                _emit(changes, a, b, b.kind, "added", None, new, None, None, None, cur.inventory)
            else:
                _emit(changes, a, b, b.kind, "removed", old, None, None, None, None, cur.inventory)
        if len(changes) == before:
            stats["files_changed_without_semantic_change"].append(path)
```

## 3. Cross-artifact safety enrichment

An HMI file cannot know that an alarm watches a protective PLC value; the graph can. A change is marked safety-related when it reaches a protective PLC variable, unless the alarm watches the opposite trip direction.

`cara_ot/semantic.py` · lines 95–122

```python
    # Cross-artifact enrichment: an HMI alarm or tag pointing at a protective
    # PLC variable is safety-related even though the HMI file cannot know that.
    ents = {**base.all_entities(), **cur.all_entities()}
    for ch in changes:
        if ch.safety_related:
            continue
        refs, alarm = list(ch.refs), None
        for r in ch.refs:
            e = ents.get(r)
            if e is None:
                continue
            if e.type == "hmi_alarm":
                alarm = e
            if e.type == "hmi_tag" and e.attrs.get("device") and e.attrs.get("plc_variable"):
                refs.append(f"{e.attrs['device']}:var:{e.attrs['plc_variable']}")
        for r in refs:
            e = ents.get(r)
            if e is None or e.type not in ("plc_var", "plc_fb") or not e.attrs.get("protective"):
                continue
            # A low-level alarm on a value that only trips on HIGH (or vice versa)
            # is an operational alarm, not part of the protection.
            trip = set(e.attrs.get("_trip_dir") or [])
            cond = str(alarm.attrs.get("condition") or "") if alarm else ""
            if alarm and trip and cond:
                if (">" in cond and "high" not in trip) or ("<" in cond and "low" not in trip):
                    continue
            ch.safety_related = True
            break
```

## 4. Impact propagation

Breadth-first traversal over relations that carry operational effect. Every affected asset keeps the path that reached it, which becomes evidence for the analyst and the AI layer.

`cara_ot/impact.py` · lines 17–35

```python
def propagate(G, change: SemanticChange, max_depth: int = 12) -> None:
    seeds = [r for r in change.refs if r in G]
    parent: dict[str, str | None] = {s: None for s in seeds}
    q = deque((s, 0) for s in seeds)
    while q:
        n, d = q.popleft()
        if d >= max_depth:
            continue
        for _, v, data in G.out_edges(n, data=True):
            if data["rel"] in PROPAGATE and v not in parent:
                parent[v] = n
                q.append((v, d + 1))

    def path_to(n):
        p = []
        while n is not None:
            p.append(n)
            n = parent[n]
        return list(reversed(p))
```

## 5. Change correlation

Two changes are linked when they reach a common controller or field device within the time window; network and HMI boxes on their own do not count. Chains are the connected components of these links (union-find).

`cara_ot/correlate.py` · lines 3–11, 27–33, 52–70

```python
Two changes belong to the same change chain when

1. they reach a common controller or field device in the dependency graph
   (shared *operational* context - network/HMI/firewall boxes on their own
   do not count, otherwise everything would correlate), and
2. they happened within the correlation time window.

Chains are the connected components of that relation. Each link keeps the
shared asset and the graph paths as evidence.
# ...
def operational_key(ch: SemanticChange, inv: Inventory) -> set[str]:
    key = set()
    for a in [ch.asset, *ch.affected_assets]:
        asset = inv.assets.get(a)
        if asset is not None and asset.type not in INFRA_TYPES:
            key.add(a)
    return key
# ...
    for i, a in enumerate(changes):
        for b in changes[i + 1:]:
            shared = keys[a.id] & keys[b.id]
            if not shared:
                continue
            ta, tb = _dt(a.detected_at), _dt(b.detected_at)
            if ta is None or tb is None:
                continue
            gap = abs((tb - ta).total_seconds()) / 60
            if gap > window_minutes:
                continue
            via = sorted(shared, key=lambda x: _rank(x, inv))[0]
            links.append({
                "a": a.id, "b": b.id, "shared_assets": sorted(shared, key=lambda x: _rank(x, inv)),
                "via": via, "minutes_apart": round(gap, 1),
                "path_a": a.impact_paths.get(via, via if via == a.asset else ""),
                "path_b": b.impact_paths.get(via, via if via == b.asset else ""),
            })
            parent[find(a.id)] = find(b.id)
```

## 6. Recovery assurance

Each component needed to rebuild an asset is checked against the backup catalogue: present, intact (SHA-256) and restoring the right configuration.

`cara_ot/recovery.py` · lines 7–17, 98–108, 134–135

```python
* MISSING          - no backup recorded
* CORRUPT          - backup file missing or its SHA-256 does not match the manifest
* OUTDATED         - backup exists but does not semantically match the configuration
                     we would need to restore (see below)
* VERSION_MISMATCH - firmware image version differs from what the device needs
* VERIFIED         - present, intact and restores the right configuration

"The right configuration" depends on authorisation: after an *unauthorised*
change we need the known-good (baseline) configuration to roll back; after
an *authorised* change the backup must already contain the new approved
configuration, otherwise it is stale.
# ...
        for comp in profile:
            cands = sorted((b for b in items if b.attrs.get("asset") == a.id and b.attrs.get("component") == comp),
                           key=lambda b: b.attrs.get("_created") or "")
            row = {"component": comp, "label": COMPONENT_LABEL.get(comp, comp), "backup": None,
                   "version": None, "status": "MISSING", "detail": "No backup recorded in the repository"}
            if cands:
                item = cands[-1]
                row.update(backup=item.id.split(":", 1)[1], version=item.attrs.get("version"))
                problem = _integrity(cur, item)
                if problem:
                    row.update(status="CORRUPT", detail=problem)
# ...
        ok = sum(1 for c in comps if c["status"] == "VERIFIED")
        status = "COMPLETE" if ok == len(comps) else ("NONE" if ok == 0 else "PARTIAL")
```

## 7. LLM grounding checker

After the local LLM writes its explanation, every asset identifier and every number with a unit is checked against the evidence bundle. Unsupported claims are flagged in the dashboard.

`cara_ot/explain.py` · lines 26–27, 290–310

```python
ID_RX = re.compile(r"\b[A-Za-z][A-Za-z0-9]*(?:-[A-Za-z0-9]+)+\b")
NUM_RX = re.compile(r"(?<![\w.-])(\d+(?:\.\d+)?)\s*(°C|degC|kPa|bar|ms|s\b|sec(?:ond)?s?\b|%|Hz|A\b|rpm)")
# ...
def check_grounding(text: str, evidence: dict) -> dict:
    ev_text = json.dumps(evidence, default=str)
    ev_ids = set(ID_RX.findall(ev_text))
    ev_nums = {_num(n) for n in re.findall(r"\d+(?:\.\d+)?", ev_text)}
    claims, unsupported = [], []
    for tok in sorted(set(ID_RX.findall(text))):
        if not any(ch.isdigit() for ch in tok):
            continue
        ok = tok in ev_ids or tok in ev_text
        claims.append({"claim": tok, "type": "identifier", "supported": ok})
        if not ok:
            unsupported.append(tok)
    for num, unit in NUM_RX.findall(text):
        ok = _num(num) in ev_nums
        claims.append({"claim": f"{num} {unit}", "type": "value", "supported": ok})
        if not ok:
            unsupported.append(f"{num} {unit}")
    total = len(claims)
    sup = sum(1 for c in claims if c["supported"])
    return {"claims_checked": total, "supported": sup, "unsupported": unsupported,
            "grounding_score": round(sup / total, 3) if total else 1.0}
```

---

© 2026 Snehansu Nag. All rights reserved. These excerpts are shown for review only; see [LICENSE](LICENSE).
