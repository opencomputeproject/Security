# Cerberus RFCs

Proposals for improving the Cerberus specifications are handled through the RFC
process. An RFC describes a problem with the protocol as it exists today and
proposes a solution.

Unlike IETF RFCs, Cerberus RFCs are non-normative, and primarily exist to record
discussions leading to particular design decisions in the protocol, and to
provide a starting point for deciding whether to adopt a proposal.

## Creating an RFC

Creating an RFC is done as follows:
1. Pick a name for the RFC, say, "My Cool Proposal".
2. Find the next RFC number in the sequence, say, 0284, and copy the template
   file into `0284-My_Cool_Proposal.md`.
3. Fill out the template as described, except for the PR link.
4. Open a PR to add your RFC to the repo, and modify the text to include the PR
   link (this is so that there is an easy way to find the discussion of the
   proposal for future readers of the proposal itself).
5. Discuss!

## Accepting and implementing an RFC

An RFC is accepted when its PR is approved and merged by a specification owner.
When and whether to accept an RFC is at the owners' discretion, but will usually
occur after some discussion of the RFC has occured on the PR.

After the RFC is merged, the author may *implement* it by submitting a PR making
text modifications to the spec(s) relevant to the RFC; in principle, the only
thing up for discussion at this point should be the precise wording of the
change.

The current RFC owners are:
* bryankel
* chweimer
