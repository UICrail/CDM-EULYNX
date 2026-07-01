# Contents

This repository contains a translation, into OWL, of the EULYNX DataPrep model, version 1.2 (short: EDP).

The original version can be found on [the EULYNX original website](https://eulynx.eu/resource-hub-dataprep-model/).

The transformation was performed using the UMLtoOWL conversion tool hosted on [GitHub](https://github.com/Airy59/UMLtoOWL). That repository is not public; please contact the owner if interested.

The original EULYNX DP documentation still applies.

No attempts were made to make EDP (as OWL) more "semantic" than EDP (as UML). In particular, no attempt was made to conflate properties bearing the same name and having similar meanings into one property. Any further "semantization" would require competencies in signalling engineering, beyond CDM engineering.

# Transformation rules

The transformation was performed in two steps:

* Rule-based UML to OWL translation. The translation is generic.
* Post-processing. The post-processing is specific to EULYNX. Its main goals are
  * to replace the original references of EDP to RSM 1.2 (UML version) by references to Semantic RSM (OWL version).
  * to remove artefacts that are specific to the original UML tool (Enterprise Architect)
These two steps are detailed below.

The output is displayed here as a single (merged) ontology, which is unreasonably big for practical usage and maintenance, but easier to inspect.

The original transformation however delivers linkes ontologies reflecting the original UML packaging.

In both cases, the namespaces reflect the original UML packaging.

# UML to OWL — common transformation rules

This section describes the **canonical** conversion performed by the `umltoowl` package: Enterprise Architect–oriented UML 2 XMI is read, normalized into an internal model, then emitted as per-package Turtle graphs (OWL 2 DL–oriented T-Box). The implementation lives mainly in `umltoowl/parse_ea.py`, `umltoowl/model.py`, `umltoowl/emit.py`, `umltoowl/naming.py`, and `umltoowl/api.py`.

The transformation is **lossy** and carries **no provenance** from UML to OWL (each ontology document is annotated accordingly in Turtle).

---

## Overall process

1. **Parse XMI** (`parse_ea.parse_ea`): build a `ParsedModel` (packages, classes, enumerations, generalizations, owned attributes, binary associations, UML notes keyed by element id). Class **owned attributes** record `aggregation` when present (used for composition → `owl:InverseFunctionalProperty` in T9). Association **member ends** record `aggregation` when present (T10). Diagram-centric noise is reduced where possible (e.g. unnamed classifiers are pruned from structural use).
2. **Emit RDF** (`emit.build_graphs`): for each UML package that needs a graph, allocate ontology IRI + entity namespace (`#`), create an `owl:Ontology` shell, then emit classes, enumerations, properties, and axioms into the correct package graph.
3. **Augment module graph** (`api.convert_file`): add `owl:imports` where the RDF uses IRIs from another package namespace, and optionally add parent→child package imports when hierarchical shells are enabled.
4. **Serialize**: one Turtle file per package (optional), plus an optional single merged Turtle file for tooling/tests.

CLI entry points call `api.convert_file`; options such as `reverse_roles`, `keep_package_structure`, and `emit_package_shells` only affect steps 2–4.

---

## Packages, IRIs, and modularity

| Concern | Rule |
|--------|------|
| Package → OWL module | Every UML package that contains model elements (and optionally every package when `emit_package_shells=True`) gets its **own** ontology document IRI and a **distinct** entity namespace (`ontologyIri#localName`). Namespaces are **not** merged across packages. |
| Default IRI layout | Ontology IRI is `base/<sanitizedPackageName>` with numeric suffixes if names collide after sanitization. |
| Hierarchical IRIs | With `keep_package_structure=True`, IRI paths mirror the UML package tree: `base/<parent>/<child>/...` with the same collision handling. `emit_package_shells=True` requires this mode and ensures empty “folder” packages still get ontology shells. |
| Turtle file layout | Mirrors package naming: flat `PackageName.ttl` or nested directories when hierarchical IRIs are used. |
| Cross-package references | Whenever a triple in package *A* uses an entity IRI whose namespace belongs to package *B*, `owl:imports` from *A*’s ontology to *B*’s ontology is added automatically. Similar imports are added for SKOS when needed, and for enumeration / class ranges in other packages. |
| Merged file | All package graphs can be concatenated into one Turtle file; **logical** namespaces and import triples remain as in the modular case. |

Standard vocabularies (`owl`, `rdfs`, `rdf`, `xsd`, `skos`) are bound in each graph; SKOS is explicitly imported on graphs that declare SKOS constructs.

---

## Ingest and pre-emit normalization

| Step | Description |
|------|-------------|
| Unnamed classifiers | Elements that are `uml:Class` / `uml:Interface` without a usable `name` are treated as EA stubs; generalizations, attributes, and associations that depend only on omitted classes are pruned (see `parse_ea.prune_unresolved_class_refs`). |
| Association class shell | If an association is marked as an association class but no `uml:Class` exists for its connector id, a **synthetic** `UmlClass` shell is inserted so OWL can declare the association-class URI (`emit.ensure_association_class_nodes`). |
| EA associations vs attributes | Some exports list composition only on a **class** `ownedAttribute` (with `association` and `type` = opposite classifier) while `uml:Association` is absent or not ingested as `model.associations`. Those links are emitted **only** via T9. Other exports populate association member ends as well (T10). |

---

## Canonical transformations (ordered by typical dependencies)

The list below is the logical dependency order inside `build_graphs`: later steps assume IRIs and graphs from earlier steps.

### T1 — Package IRI allocation

**Input:** `ParsedModel`, base IRI, layout flags.  
**Output:** `PackageIRIs` per package: `ontology_iri`, `entity_ns` (usually `ontology_iri + "#"`).  
**Depends on:** package tree, class/enum/association package membership (and all packages if shells enabled).

### T2 — Ontology document shell

**Output:** For each allocated package, an RDF graph with one `owl:Ontology` resource, standard prefix bindings, and a fixed `rdfs:comment` noting lossy XMI→OWL conversion.  
**Depends on:** T1.

### T3 — Enumerations as SKOS

**Output:** Per enumeration: a `skos:ConceptScheme`, `owl:imports` of the SKOS vocabulary document, and one `skos:Concept` per literal (`skos:inScheme`, `skos:prefLabel`). Literal-level UML notes become `rdfs:comment` on concepts when ids are available.  
**Depends on:** T1–T2.  
**Used by:** T5 (datatype vs object typing), T7 (enum-typed attributes need SKOS import on consumer graphs).

### T4 — Classes as `owl:Class`

**Output:** Each `UmlClass` → `owl:Class` with `rdfs:label` and optional `rdfs:comment` from UML notes (`ownedComment` / `Comment` bodies, deduplicated). Local names are sanitized; fragment collisions within a package namespace append the class XMI id.  
**Depends on:** T1–T2.  
**Used by:** all property domains/ranges, generalizations, restrictions.

### T5 — Reserved local names for properties

**Output:** Per package, the set of local name fragments already used by **classes** and **SKOS** IRIs is reserved so OWL property IRIs never **pun** the same IRI as a class (OWL 2 DL illegal: class and property same IRI).  
**Depends on:** T3–T4.

### T6 — Property occurrences and fragment resolution

**Collects:** owned attributes; navigable association ends (as object properties); association-class “link” pseudo-properties from the association class to each participant.  
**Role direction (`reverse_roles`):**

- `False` (default, TRANSMODEL-style): the property **domain** is the class at the **opposite** end from the named role; **range** is the role end’s type.
- `True` (common UML style): **domain** is the role end’s type; **range** is the opposite end’s type.

**Disambiguation** when assigning OWL local names (`resolve_property_fragments`):

1. Homonyms or multiple owning classes → suffix `_<SourceClassName>` (sanitized).
2. Still clashing across packages → suffix `_<packageSlug>`.
3. Fallback → physical key (XMI ids) and numeric suffixes.

**Depends on:** T4–T5.  
**Used by:** T9–T11.

### T7 — Generalizations → `rdfs:subClassOf`

**Output:** Each `Generalization` → `rdfs:subClassOf` from specific to general, in the **child class’s** package graph. Unresolved ends are skipped with a log entry.  
**Model-specific suppression:** the edge `RemainingShuntingRouteRelease` ⊑ `ConfiguredBaseObject` is **omitted** by policy (`_suppress_generalization_for_owl`).  
**Depends on:** T4.

### T8 — Partition axioms under a generalization parent

Derived from the same generalization set as T7 (respecting the same suppression).

| Parent | Condition | OWL pattern |
|--------|-----------|-------------|
| Abstract | All children in the **same** package as the parent | `owl:disjointUnionOf` listing child classes on the parent. |
| Abstract | Children span multiple packages | No `disjointUnionOf` on parent; instead **pairwise** `owl:disjointWith` between siblings **within each child package** graph. |
| Concrete | Single inheritance model (`has_multiple_inheritance` false) and ≥2 children | Pairwise `owl:disjointWith` between children under the **parent’s** graph. |

If the model allows multiple inheritance, disjointness is **not** inferred for concrete parents.

**Depends on:** T4, T7.

### T9 — Attributes → datatype or object properties

**Typing:**

- Type refers to a **class** → `owl:ObjectProperty`, `rdfs:range` = class URI; cross-package range adds `owl:imports`.
- Type refers to an **enumeration** → `owl:ObjectProperty` with `rdfs:range` = `owl:Thing` (DL-friendly alongside SKOS individuals); `owl:imports` to the enum’s package and SKOS when the enum lives elsewhere.
- Otherwise primitive / unknown → `owl:DatatypeProperty` with XSD range from a fixed string→XSD map; unknown primitives default to `xsd:string`. Unresolved class-like refs use `owl:Thing` as range and are logged.

**Multiplicity (only `0..1` and `1` are interpreted) for datatype properties:**

- `0..1` or `1` → also `owl:FunctionalProperty`.
- `1` → additionally an existential `owl:Restriction` (`owl:someValuesFrom` = XSD / datatype filler) on the domain class.

**Object-typed attributes (UML `ownedAttribute` on a class):**

Enterprise Architect often encodes compositions as a property on the **domain** class with `aggregation="composite"` and `uml:Property` `type` = **range** classifier (the same logical construct as T10, but **not** always duplicated as an ingestible `uml:Association` in XMI). The parser stores `aggregation` on `OwnedAttribute`.

| Situation | `owl:InverseFunctionalProperty` | `owl:FunctionalProperty` | Existential `owl:someValuesFrom` |
|-----------|--------------------------------|--------------------------|-------------------------------------|
| `aggregation=composite` on the attribute | Yes | Yes, when effective multiplicity is `0..1` or `1` (multiplicity absent under composition is treated as `1`) | Yes when effective multiplicity is `1` |
| Not composite | No | Yes, when multiplicity is `0..1` or `1` | Yes only when multiplicity is `1` |

**Effective multiplicity** for composite attributes: if UML multiplicity is absent or not interpreted (`Mult.IGNORED` in code), it is treated as exactly `1` for existential and functional characterisation.

UML notes on the attribute become `rdfs:comment` on the property.

**Depends on:** T3–T4, T6.

### T10 — Binary associations → object properties

**Output:** For each **navigable** end with resolved class types at both ends: `owl:ObjectProperty` with domain/range per T6 role convention. Non-navigable ends do not yield a property.

**Domain-side member (multiplicity + composition):** For `domain → range`, the relevant UML member is usually:

- the association end whose ingested `type` is the **range** class (Enterprise Architect: navigable property on the domain class lists `type` = range and carries `aggregation` / multiplicities), or
- if that cannot be identified uniquely, the end whose `type` is the **domain** class (association-owned opposite end pattern).

When `domain` and `range` are the **same** class (self-association), the implementation falls back to the end whose `type` is the domain class.

**OWL cardinality (after declaration):**

| Situation | `owl:InverseFunctionalProperty` | `owl:FunctionalProperty` | Existential `owl:someValuesFrom` on the domain class |
|-----------|--------------------------------|--------------------------|------------------------------------------------------|
| Composition on the domain-side member (`aggregation=composite`) | Always | Yes, when effective multiplicity is `0..1` or `1` | Yes when effective multiplicity is `1` (missing multiplicity under composition is treated as `1`) |
| Not composition | Yes, when domain-side multiplicity is `0..1` or `1` | Yes, under the same condition | Yes **only** when domain-side multiplicity is `1` |
| Other domain-side multiplicities | No | No | No |

**Effective multiplicity** for composition on associations: if domain-side multiplicity is absent or not interpreted (`Mult.IGNORED`), it is treated as exactly `1` for existential and functional characterisation (same idea as T9).

**Semantics:** **composition** implies each range instance is linked from at most one domain instance (`InverseFunctionalProperty`). **Functional** and **existential** restrictions use **effective** domain-side multiplicity as in the table. Resolution of which association member is “domain-side” is implemented as `_association_domain_semantic_end` (prefer member whose `type` is the **range** class), then `_association_source_end` if needed.

**Association-class link properties** (T11) do not use this table unless the XMI supplies aggregation on those ends (unusual).

If **both** ends are navigable and both properties exist, `owl:inverseOf` links the two property IRIs (declared from the perspective of one end’s domain package). Cross-package domain/range adds imports.

**Depends on:** T4, T6. **XMI:** `uml:Property` `aggregation` on association ends is ingested in `parse_ea` (`AssociationEnd.aggregation`). **Note:** Many EA exports expose the same composition only on class `ownedAttribute` (see T9); those do not appear in `model.associations` if no `uml:Association` is ingested.

### T11 — Association classes

**Output:** The association class id maps to an `owl:Class` (T4). Two **link** object properties are emitted from the association class to each participant (`<AssocName>_link_<ParticipantName>` style, subject to T6 naming). Cross-package participants add imports.

**Depends on:** T4, T6, T10 structure (same association object).

### T12 — Notes → `rdfs:comment`

UML comment bodies attached to classes, enumerations, enumeration literals, attributes, association ends, and association-class link properties are copied to `rdfs:comment` on the corresponding OWL resource (deduplicated per resource).

---

## Interdependency summary

The diagram below is plain text so it renders without a Mermaid extension.

```
T1 Package IRIs
  └─► T2 Ontology shells
        ├─► T3 SKOS enumerations ──► T5 Reserve class/SKOS fragments
        └─► T4 owl:Class ───────────► T5
                                      └─► T6 Property names
        T4 ──► T7 subClassOf ──► T8 Disjointness
        T4,T3,T6 ──► T9 Attributes
        T4,T6 ─────► T10 Associations
        T4,T6 ─────► T11 Assoc-class links
        T3,T4,T6 ──► T12 rdfs:comment
```

Read “A ──► B” as: B depends on A (or on a chain through A).

---

## API-level steps after `build_graphs`

| Step | Function | Role |
|------|----------|------|
| Cross-namespace imports | `augment_cross_package_imports` | Ensures every graph imports every other ontology whose entity namespace appears as an object IRI in that graph (excluding W3C vocabularies). |
| Package-tree imports | `augment_package_tree_imports` | When `emit_package_shells=True`, parent ontology imports each **direct** child package ontology. |
| Merge | `merge_payload_ttl` | Union of all triples into one graph for a single deliverable file. |

---

## Implementation reference (code)

| Topic | Location |
|-------|----------|
| Attribute `aggregation` ingest | `parse_ea.walk_property_as_attribute` → `OwnedAttribute.aggregation` |
| Association end `aggregation` ingest | `parse_ea.collect_association_ends` → `AssociationEnd.aggregation` |
| Object attribute cardinality (IFP / functional / existential) | `emit.build_graphs` (attribute loop); composite uses `_aggregation_is_composite` |
| Association cardinality | `_association_domain_semantic_end`, `_association_source_end`, `_emit_association_property_cardinality` in `emit.py` |
| Multiplicity enum | `model.Mult` (`0..1`, `1`, `ignored` for anything else) |

---

## EDP deliverable chain

The canonical TTL from this document may be fed to the **EDP post-processor** ([`uml-to-owl-transformation-edp-post-process.md`](uml-to-owl-transformation-edp-post-process.md)) for Geo/RSM cleanup, CSV property substitution, merged ontology identity (`edp.ttl`), and reference ontology imports.

# EDP post-processing rules

This section describes rules implemented in `umltoowl/edp_post_process.py`. They apply to **Turtle files already produced** by the canonical converter (see [`uml-to-owl-transformation-common.md`](uml-to-owl-transformation-common.md)), typically under `out/EDP`. Outputs are written to a separate directory (default `out/EDP/PP`); source TTL files are **not** modified.

Input Turtle may already declare `owl:FunctionalProperty`, `owl:InverseFunctionalProperty`, `owl:ObjectProperty`, and `owl:Restriction` axioms from T9/T10. Post-processing **preserves** those types when it rewrites triples (for example Rule 2 keeps `owl:FunctionalProperty` when promoting a datatype property to an object property where the substitution row requires it).

The script loads **all** `*.ttl` files in the source directory into one union graph for **detection** (e.g. which diagram-only classes to remove, which CSV substitution rows match). **Mutations** are applied **per file** when writing outputs.

Default inputs:

- Property substitutions: `umltoowl/substitutions/edp_properties.csv`
- Ontology header literals: `umltoowl/EDP_annotations.csv`

---

## Execution order (per deliverable run)

Order matters: restriction cleanup assumes mirror properties and deprecated classes are removed first; filler relaxation assumes Rule 2 has possibly promoted datatype properties to object properties.

| Order | Name | Scope |
|------:|------|--------|
| 1 | Rule 1 — annotation-only classes | Union for detection; then each file |
| 2 | Rule 1b (phase A) — deprecated Geo classes | Each file |
| 3 | Rule 1b — restrictions without fillers | Each file (after deprec class removal) |
| 4 | Rule 1b — `LinearElement` bridge | Each file |
| 5 | Rule 1b — `AnchorPoint` bridge | Each file |
| 6 | Rule 1b — `RegisteredIntrinsicCoordinate` / `atPort` | Each file |
| 7 | Rule 1c — RSM mirror datatype properties | Each file |
| 8 | Orphan restrictions (no `owl:onProperty`) | Each file |
| 9 | Rule 2 — property substitution (CSV) | Each file |
| 10 | Relax XSD fillers in object-property restrictions | Each file |
| 11 | Merged-file only: canonical ontology + import strip/add | Only the merged source file |
| 12 | Ontology header annotations (CSV) | Each file (merged file restricted to canonical IRI) |
| 13 | Prefix bindings + serialize | Each file |

The merged source basename defaults to `merged.ttl`; output is renamed to `edp.ttl`.

---

## Rule 1 — Remove annotation-only `owl:Class` entities

**Purpose:** Drop diagram placeholders or shells that never participate in structural axioms.

**Criterion:** An IRI is a candidate if it has `rdf:type owl:Class`. It is **removed** if every triple where it appears as subject or object uses only:

- `rdf:type owl:Class`, and/or
- predicates treated as **annotations** (RDFS label/comment/seeAlso/isDefinedBy; OWL deprecation/version metadata; selected SKOS documentation predicates — see `_ANNOTATION_PREDICATES` in code).

**Effect:** All triples whose subject **or object** is in the removed set are deleted from each output graph.

---

## Rule 1b — Geo_information alignment and RSM linking

### Phase A — Deprecated coordinate / alignment classes

**Removed class IRIs (fixed set):** `GeometricCoordinate`, `HorizontalAlignmentSegment`, `Cant`, `AlignmentCantSegment`, `VerticalAlignmentSegment`, `ElevationAndInclination` in the `Geo_information` namespace.

**Closure:** Any OWL property typed with an OWL property metaclass whose `rdfs:domain` or `rdfs:range` points at a class already marked for removal is added to the removal set; repeat until fixed point. Then **every** triple involving any IRI in that set (as subject, predicate, or object) is removed.

### Restrictions without value fillers

Immediately after deprecated-class removal: any `owl:Restriction` that still has `owl:onProperty` but **no** filler predicate from the configured set (`owl:someValuesFrom`, `owl:allValuesFrom`, cardinalities, `owl:onClass`, etc.) is removed entirely (all triples involving that restriction node).

### `LinearElement`

- Remove the datatype property `refersToRsmLinearElement` and **every** triple that mentions it (subject, predicate, or object).
- If `Geo_information#LinearElement` is declared `owl:Class`, ensure `rdfs:subClassOf` `https://cdm.ovh/rsm/topology/topo#LinearElement` once.

### `AnchorPoint`

- Same pattern for `refersToRsmAnchorPoint` → remove all axioms using it.
- If `Geo_information#AnchorPoint` is declared `owl:Class`, ensure `rdfs:subClassOf` `https://cdm.ovh/rsm/linearReferencing/rlrs#AnchorPoint` once.

### `RegisteredIntrinsicCoordinate` — `atPort`

- Remove `refersToRsmIntrinsicCrd` and all triples using it.
- If `RegisteredIntrinsicCoordinate` is an `owl:Class` and `atPort` is not yet declared:
  - Declare `atPort` as `owl:ObjectProperty` and `owl:InverseFunctionalProperty`, domain = `RegisteredIntrinsicCoordinate`, range = `topo:Port`, with a fixed `rdfs:comment`.
  - Add two `rdfs:subClassOf` restrictions on the class: existential `someValuesFrom` `topo:Port`, and qualified cardinality **1..4** on `topo:Port` (encoding former 1..4 multiplicity).

---

## Rule 1c — Legacy RSM mirror datatype properties

**Removed property IRIs (fixed list):** mirror `refersToRsm*` datatype properties holding external IRIs as `xsd:string` for:

- Signal, Route body, Vehicle stop, Vehicle passage detector, TPS device

Each listed IRI is removed with **all** triples that mention it (declarations, annotations, restrictions, etc.).

---

## Orphan `owl:Restriction` cleanup

After Rules 1b/1c strip properties from restrictions:

- Any `owl:Restriction` with **no** `owl:onProperty` is removed recursively (including as object of `rdfs:subClassOf`) until no such nodes remain.

This avoids invalid OWL structures in tools such as Protégé.

---

## Rule 2 — CSV property substitution

**Source:** Rows in the substitutions CSV with columns:

`domain_iri`, `generated_property_iri`, `range_iri`, `replacement_property_iri`, `super_property_iri`, `replacement_range_iri`

**Match:** A row applies when the graph contains the **generated** property with:

- `rdfs:domain` including `domain_iri`, and
- `rdfs:range`: either equals `range_iri` when provided, or when `range_iri` is blank: property is `owl:DatatypeProperty` with `rdfs:range xsd:string`.

**Apply:**

- If replacement IRI equals generated (`same` in CSV or identical IRI): keep IRI; replace domain/range with CSV values; add `rdfs:subPropertyOf` super property if missing.
- Else: delete all triples mentioning the old property; reattach subject/predicate/object occurrences to the new IRI; add `rdfs:subPropertyOf`; do not copy `rdfs:label` from old to new when replacing IRI.

**Datatype → object promotion:** If `range_iri` is blank (string datatype row) and `replacement_range_iri` is **not** an XML Schema datatype IRI, the property is retyped from `owl:DatatypeProperty` to `owl:ObjectProperty`. Other `rdf:type` triples on the new property (such as `owl:FunctionalProperty`) are carried over when the substitution logic rewrites them; `owl:InverseFunctionalProperty` is preserved the same way when present on the generated IRI.

**Union pre-check:** The pipeline builds a scratch graph after Rules 1, 1b, 1c to record which substitution rows match at least once (logged as applied vs not matched).

---

## Relax XSD restriction fillers (`relax_xsd_datatype_restriction_fillers_to_thing`)

**After Rule 2:** For every `owl:Restriction` whose `owl:onProperty` resolves to an **object property** (`rdf:type owl:ObjectProperty`), any `owl:someValuesFrom` or `owl:allValuesFrom` filler that is an **XSD datatype IRI** is replaced with `owl:Thing`.

**Rationale:** Promoted datatype properties often leave restrictions whose fillers are still XSD IRIs; DL tools expect class expressions there.

---

## Merged deliverable (`merged.ttl` → `edp.ttl`)

Only the file whose basename matches the configured merged source (default `merged.ttl`):

1. **`normalize_edp_merged_ontology_document`** — Ensure `https://cdm.ovh/edp` is typed `owl:Ontology` if absent.
2. **`strip_redundant_imports_from_merged_edp_graph`** — Remove `owl:imports` where the object is another ontology **already embedded** in the same graph; remove imports of `http://www.w3.org/2004/02/skos/core` (SKOS terms are inlined).
3. **`add_rsm_reference_ontology_imports_to_merged_edp`** — On the canonical ontology `https://cdm.ovh/edp`, add missing `owl:imports` for:
   - `https://cdm.ovh/rsm/topology/topo`
   - `https://cdm.ovh/rsm/localisation/loca`
   - `https://cdm.ovh/rsm/track/trfn`
   - `https://cdm.ovh/rsm/linearReferencing/rlrs`

Step 2 is intended to run **before** step 3 so embedded module imports are stripped first; external RSM document IRIs are not embedded as ontologies in EDP.

---

## Ontology header annotations (CSV)

**Source:** `property_name`, `value` columns. Short names map to DC, DCTerms, RDFS, OWL predicates; full IRIs may be used as property names.

**Apply:**

- **Merged `edp.ttl`:** annotations attach **only** to `https://cdm.ovh/edp`.
- **Other TTL files:** every `owl:Ontology` subject in the file receives each annotation triple.

`dc` and `dcterms` prefixes are bound when annotations are added.

---

## Serialization helpers

**`bind_rsm_namespace_prefixes`:** Declares Turtle prefixes `trfn`, `topo`, `loca`, `rlrs` for the fixed RSM namespace IRIs used in axioms and imports.

---

## Logging

`run()` writes a human-readable report to `_post_process_edp.log` (default under the output directory), including rule headers, class IRIs removed under Rule 1, fixed IRIs for Rules 1b/1c, per-row substitution status, and aggregate triple counts for each transformation class.

---

## Relationship to the canonical converter

Post-processing **does not** re-read XMI. It assumes Turtle already reflects the rules in [`uml-to-owl-transformation-common.md`](uml-to-owl-transformation-common.md) (classes, SKOS enumerations, attributes, associations, imports, composition / functional / inverse-functional object properties where applicable).

**Added axioms outside the XMI pipeline:** Rule 1b declares `atPort` on `RegisteredIntrinsicCoordinate` with `owl:InverseFunctionalProperty` and OWL 2 qualified cardinality restrictions; those IRIs are fixed in `edp_post_process.py`, not emitted by `emit.build_graphs`.

**CLI / wrapper:** `python -m umltoowl.edp_post_process` (see also `edp_pp.sh` in the repository if present).


