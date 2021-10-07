* Name: Inclusive_Bus_Roles
* Date: 2020-10-07
* Pull Request: [#26](https://github.com/opencomputeproject/Security/pull/26)

# Objective

This RFC proposes new language for referring to devices' connection roles on a
bus, replacing the outdated "master" and "slave" nomenclature.

# Proposal

We propose replacing all occurrences of "master" and "slave" with "host" and
"device". This terminology is already used in new proposals and language for
Cerberus, so this RFC simply makes existing practice normative.

# Specification Changelist

See above. `s/master/host/g` and `s/slave/device/g`, taking care to not replace
"master" where it does not refer to a bus role.

# Alternatives Considered

Many other choices exist for this nomenclature: "primary/secondary",
"host/target", and "client/server". Our choice is arbitrary and merely reflects
language we were already using.
