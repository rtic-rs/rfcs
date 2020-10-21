# Hibernating

A member that will be absent or busy for an extended period of time and not able
to participate in the Team during that time should put themselves in "hibernation"
state. During this hibernation period:

- The member will be removed from their GitHub teams'. `cc @rtic-rs/$team`
  will *not* cc them.

- The member name will be removed from the list of teams in the rtic-rs/rfcs
  README, but they will be listed under the 'Hibernating' section of that
  README.

- The member will not be counted when computing the number of votes required to
  reach majority on matters that concern the teams they are a member of. This
  includes PRs labeled `T-all` that need a decision.

To enter or leave the hibernation state the member will:

- Notify their teams (e.g. `cc @rtic-rs/$team`) and send a PR to
  rtic-rs/rfcs updating the README to reflect their hibernation state.
