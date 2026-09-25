# ADR-0002: Feature to variants model

Status: Accepted.

Decision: one semantic FeatureDefinition references zero or more FeatureVariants. Vehicle/platform implementations are separate variants.

Consequences: no single hard-coded implementation per feature; applicability selects one exact variant or blocks.