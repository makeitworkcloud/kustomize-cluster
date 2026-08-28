# Operator guidance

This directory owns cluster operators. Preserve each operator's upstream/Kustomize composition, CRD dependencies, sync behavior, and SOPS secret generators.

On this single-node cluster, do not add resource requests or limits by default; they can prevent scheduling or cause throttling. Follow the existing operator-specific pattern when an exception is required. Diagnose operator failures through the owning Argo CD Application, managed resources, events, pods, logs, and metrics before proposing a change.
