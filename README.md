# org-omg-sysmlv2

[![CI](https://github.com/kotoba-lang/org-omg-sysmlv2/actions/workflows/ci.yml/badge.svg)](https://github.com/kotoba-lang/org-omg-sysmlv2/actions/workflows/ci.yml)

**A scoped subset of [OMG SysML v2](https://www.omg.org/sysml/sysmlv2/)
(Systems Modeling Language) as portable EDN/Clojure data, built via
threading-friendly Clojure builder functions -- not a parser for SysML's
textual concrete syntax.** SysML v2's `.sysml` textual grammar is large and,
as of this writing, still evolving alongside the formal specifications
themselves; parsing it is explicitly out of scope for this v1 (see
Follow-ups). Instead, exactly like this org's `org-oasis-open-xmile`, you
construct a model by calling Clojure functions -- you get the same EDN a
concrete-syntax parser would eventually produce, without depending on that
still-moving grammar.

SysML v2 layers on top of **KerML** (Kernel Modeling Language), an
application-independent kernel with formal semantics. `sysml.kerml` models
KerML's `Type`/`Classifier`/`Feature`/`Specialization`/`Featuring`/
`Multiplicity`; `sysml.model` builds SysML's signature **Definition/Usage
duality** on top of it (a `Definition` is a reusable type, a `Usage` is an
occurrence of one in context -- e.g. a `Vehicle` PartDefinition nests an
`engine` PartUsage typed by an `Engine` PartDefinition, which nests a
`fuelIn` PortUsage typed by a `FuelPort` PortDefinition).

**Sources.** The formal specifications were adopted 2025-07-21 and
editorially updated 2026-03 for ISO submission:
[KerML 1.0](https://www.omg.org/spec/KerML/1.0/PDF) (OMG document
formal/26-03-01), [SysML 2.0 Language](https://www.omg.org/spec/SysML/2.0/Language/PDF)
(formal/26-03-02), and its companion
[Transformation spec](https://www.omg.org/spec/SysML/2.0/Transformation/PDF)
(formal/26-03-03) -- see the [OMG SysML v2 page](https://www.omg.org/sysml/sysmlv2/)
and [About-KerML](https://www.omg.org/spec/KerML/1.0/About-KerML)/
[About-SysML](https://www.omg.org/spec/SysML/2.0/About-SysML). Precise
structural shape (which attributes/references each class actually has) is
ground-truthed against the open-source
[SysML-v2-Pilot-Implementation](https://github.com/Systems-Modeling/SysML-v2-Pilot-Implementation)'s
Ecore metamodels -- `org.omg.sysml/model/kerml.ecore` and
`org.omg.sysml.model/src/main/resources/model/SysML.ecore` -- whose embedded
`eAnnotations` documentation reproduces spec-clause-level prose per class.
Source code docstrings in this library cite the specific ecore file and
approximate line number for each concept's structural shape.

## Maturity

| | |
|---|---|
| Role | data model (KerML kernel + SysML Definition/Usage elements + structural validation) |
| Concrete syntax | **not implemented** -- EDN builder functions only, no `.sysml` text parser (see Follow-ups) |
| KerML coverage | `Type`/`Classifier`/`Feature`/`Specialization`/`Featuring`/`Multiplicity`, plus `Conjugation` |
| SysML coverage | `Package`, `PartDefinition`/`Usage`, `AttributeDefinition`/`Usage`, `PortDefinition`/`Usage`, `ConnectionDefinition`/`Usage`, `InterfaceDefinition`/`Usage`, `ItemDefinition`/`Usage`, `ActionDefinition`/`Usage` (structural only), `RequirementDefinition`/`Usage` (subject/actor/required-constraint, satisfy/verify structural traceability) |
| Behavior/requirements semantics | structural only -- no execution, no constraint solving, no requirements-verification reasoning (see Follow-ups) |
| Tests | builder/query coverage for every namespace, incl. specialization-closure and validation error/warning classification |
| Runtime deps | `kotoba-lang/dsl-core` (validation-problem convention) only |

## Namespaces

- `sysml.kerml` -- the KerML 1.0 kernel EDN schema (`:kerml/*` keys):
  `Type`/`Classifier`/`Feature` element builders, `Specialization`/
  `Featuring`/`Conjugation` relationship builders, and queries including
  `all-supertypes` (a DFS/worklist closure over Specialization, mirroring
  KerML's own `Type::allSupertypes()` operation).
- `sysml.model` -- the SysML-level EDN schema (`:sysml/*` keys) built on
  top of `sysml.kerml` types: every Definition is a `kerml/classifier`,
  every Usage a `kerml/feature`, tagged with `:sysml/element-kind`.
  Threading-friendly builders for all covered Definition/Usage pairs plus
  `Package`, and structural queries (`definition-of`, `nested-usages`/
  `all-nested-usages`, `all-supertypes` re-exported for Definition-to-
  Definition generalization chains).
- `sysml.validate` -- structural checks returning `kotoba.dsl.problem`-
  shaped problems: dangling/wrong-kind Definition references, illegal
  Specialization cycles (DFS white/gray/black), out-of-range Multiplicity
  bounds, dangling Connection/Interface ends, and `:warn`-level scope-out
  notices for Action elements and satisfy/verify relationships.

## Contract

```clojure
(require '[sysml.model :as sm]
         '[sysml.validate :as validate])

(def vehicle-model
  (-> (sm/model "vehicle-model")
      (sm/add-element (sm/port-definition "FuelPort"))
      (sm/add-element (sm/part-definition "Engine"))
      (sm/add-element (sm/port-usage "fuelIn" "FuelPort"))
      (sm/add-element (-> (sm/part-usage "engine" "Engine")
                           (sm/nest "fuelIn")))               ; engine owns a nested Port
      (sm/add-element (-> (sm/part-definition "Vehicle")
                           (sm/nest "engine")))))              ; Vehicle owns a nested Part

(validate/valid? (validate/validate vehicle-model))  ;=> true

(sm/definition-of vehicle-model "engine")   ;=> the Engine PartDefinition map
(sm/all-nested-usages vehicle-model "Vehicle")  ;=> #{"engine" "fuelIn"}
```

## Follow-ups (v2, out of scope for this landing)

- **Textual concrete syntax** -- no `.sysml` parser. SysML v2's grammar
  (Xtext, `org.omg.sysml.xtext/src/org/omg/sysml/xtext/SysML.xtext` in the
  Pilot Implementation) is large and, per the OMG's own release cadence
  (beta1 through beta4 before formal adoption), still evolving; this
  library gives you the EDN a parser would eventually produce.
- **Action/behavior execution semantics** -- `ActionDefinition`/
  `ActionUsage` are modeled as data only (structurally, like any other
  Definition/Usage pair); no control-flow/execution engine.
- **Requirements verification/satisfaction reasoning** -- `satisfy`/
  `verify` relationships are modeled as named structural elements only
  (`sysml.model/satisfy-requirement-usage`/`verify-requirement-usage`); no
  requirements-traceability engine evaluates them. A real `VerificationCase`
  concept (KerML `RequirementVerificationMembership`) is not modeled.
- **Constraint solving** -- `RequirementDefinition`/`Usage`'s required
  constraints are plain expression strings (`:sysml/required-constraints`),
  and Analysis/Calculation/Constraint Definitions generally are not
  modeled as their own element kinds; no constraint solver.
- **Views/Viewpoints** and any **graphical/diagram notation** -- not
  modeled, matching this library's data-only scope.
- **Multiplicity bound expressions** -- KerML's real `Multiplicity` bounds
  are `Expression` trees (so a bound can itself be computed); this library
  simplifies bounds to plain integers.
- **Port conjugation semantics** -- `sysml.kerml/conjugate` records a
  `Conjugation` relationship structurally, but does not automatically
  flip Feature `direction` the way real KerML conjugation does.

## Test

```bash
clojure -M:test
```

## License

MIT.
