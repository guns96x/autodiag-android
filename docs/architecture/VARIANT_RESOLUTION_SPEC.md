# Variant Resolution Specification

**Status:** NORMATIVE.

## 1. Output

`VariantSelectionResult` is exactly one of:

```kotlin
public sealed interface VariantSelectionResult {
    public data class ExactMatch(val variant: FeatureVariant, val ecu: EcuIdentity) : VariantSelectionResult
    public data object NoMatch : VariantSelectionResult
    public data class Ambiguous(val candidates: List<String>) : VariantSelectionResult
    public data class PartialOnly(val candidates: List<String>) : VariantSelectionResult
    public data class Protected(val candidates: List<String>) : VariantSelectionResult
    public data class Conflict(val candidates: List<String>) : VariantSelectionResult
}
```

## 2. Candidate construction

For every `feature.variantIds`:
1. resolve the referenced variant;
2. resolve ECU by `ecuLogicalAddress`;
3. reject missing ECU as non-match;
4. require `variant.protocolProfileId` to exist;
5. run `PROTOCOL_CONSISTENCY.md`;
6. evaluate `platformSelector`;
7. evaluate `applicability`.

A rule returning UNKNOWN does not become MATCH.

## 3. Applicability rule semantics

- `All`: MATCH only when all children MATCH; CONFLICT dominates; UNKNOWN dominates NO_MATCH only if no child is NO_MATCH.
- `AnyOf`: MATCH when any child MATCH; otherwise CONFLICT if any CONFLICT; otherwise UNKNOWN if any UNKNOWN; otherwise NO_MATCH.
- `Not`: MATCH ↔ NO_MATCH; UNKNOWN/CONFLICT remain unchanged.
- `FieldEquals`, `FieldIn`, `FieldPrefix`, `VersionRange`: missing identity field = UNKNOWN.
- `InWhitelist`: missing/non-EXACT whitelist = UNKNOWN.
- `FeatureValueIs`: absent current feature value = UNKNOWN.
- `Always`: MATCH.

## 4. Evidence status partition

After applicability:
- EXACT + MATCH -> executable candidate.
- PROTECTED + MATCH -> protected candidate.
- CONFLICT + MATCH or protocol-consistency conflict -> conflict candidate.
- PARTIAL/UNRESOLVED + MATCH/UNKNOWN -> partial candidate.

## 5. Specificity

Specificity is a tuple, compared lexicographically:

```
(
  exactIdentityPredicates,
  whitelistPredicates,
  versionPredicates,
  platformPredicates,
  protocolPredicates,
  ecuPredicates,
  featureValuePredicates,
  totalPositivePredicates
)
```

Counts are the number of positively matched leaf rules of each class. `Always` contributes zero. Negations do not increase specificity.

## 6. Selection

1. If one or more conflict candidates exist at the highest matched specificity -> `Conflict`.
2. If executable EXACT candidates exist:
   - choose the unique highest-specificity candidate;
   - if two or more tie -> `Ambiguous`.
3. Else if protected candidates exist -> `Protected`.
4. Else if partial candidates exist -> `PartialOnly`.
5. Else -> `NoMatch`.

Array order, source order, source address and variant ID are never tie-breakers.

## 7. Write gate

Only `ExactMatch` may enter `WriteTransactionEngine`. All other results are hard blocks.
