# Concourse RFCs

> A process for collaborating on substantial changes to Concourse.


## What should be proposed as an RFC?

RFCs should be opened for changes that have a substantial impact on Concourse
users and developers. The RFC process enables us to collaborate during the
architecture and feature design process, before we get to code and implementation.

An important goal of this process is to keep a record of the decision-making
process as a proposal evolves. This is done via the pull request review and
commenting system. This allows discussions to be marked as "resolved", rather
than wrangling an ever-growing comment chain on a GitHub issue.


## RFC Process

1. Fork this repository.
1. Copy the `000-example` RFC template, naming it something like
   `123-my-proposal`.
  * Don't worry too much about the number; they don't have to be sequential.
    You can try to predict your pull request number for example, and/or just
    edit it after submitting.
1. Write your RFC in `proposal.md` under your RFC directory.
    * Try to outline the motivation for the proposal first. A proposal with
      no context is more likely to fall under scrutiny.
    * Having a summary near the beginning of the proposal is also helpful.
    * Take special care to think about any risks or drawbacks to your proposal
      ahead-of-time. These have to be assessed at some point! If you're not
      sure how to resolve them, leave them under an "open questions" section,
      and we can all try to work through them together.
1. Submit a pull request. Your proposal may include any dependent assets
   (example content, screenshots) under its RFC directory. For convenience,
   link to the rendered proposal in the pull request body, like so:

   ```
   [Rendered](https://github.com/{YOUR NAME}/rfcs/blob/{YOUR BRANCH}/123-my-proposal/proposal.md)
   ```

   Try to keep the description light, since most content should be in the
   `proposal.md` already. But feel free to reference any relevant GitHub
   issues, since that helps with context-building.
1. Each RFC will be assigned to at least one reviewer. Feel free to reach out
   to them if you need help on any part of the process or with the proposal
   itself.
1. Community members are expected to submit feedback by leaving comments on
   lines in the pull request and submitting reviews. This allows conversations
   to be marked "resolved" and prevents the comment history on the pull request
   from growing larger and larger.
  * As the RFC author, feel free to leave your own comments/feedback, using the
    pull request as a "captain's log" as you think about the problem more and
    reach key decisions. The point of all this is to have a clear public record
    for the decision-making process.
1. Amendments to the RFC should be made by pushing more commits to the RFC
   branch. **Please do not rebase and force-push over your own commits.**
   Instead, try to make meaningful commits that summarize their changes.


### Resolution

Once consensus builds and things slow down, the RFC will be granted with one of
the following labels:

* `resolution/merge`: there are no outstanding objections to the RFC and
  implementation can begin as soon as the RFC is merged.
* `resolution/postpone`: there are no outstanding objections to the RFC, but we
  have decided to defer its implementation until some time in the future, and
  until then it's better to leave the proposal unmerged in case things change
  by the time we can get to implementation.
* `resolution/close`: we have decided not to accept the RFC, and have no plans
  for implementation.

These labels mark the beginning of the final phase of the RFC. During this
point, any additional feedback will be sought out by communicating it on our
[blog](https://medium.com/concourse-ci).

There will then be a two-week quiet period on the RFC. If during this time
there is a challenge to the resolution, the label will be removed and the RFC
process will continue. Ideally there are no changes to the RFC during this
period (all typos should be resolved by now, and wording should be fairly
clear).

Depending on the resolution, the following will happen to the RFC pull request:

* `resolution/merge`: the PR will be merged!
* `resolution/postpone`: the PR will be closed and stamped with a `postponed`
  label. At some point in the future the pull request may be re-opened.
* `resolution/close`: the PR will be closed with no additional label.


## Implementing an RFC

Once an RFC is accepted, an associated issue will be opened on the [`concourse`
repository](https://github.com/concourse/concourse) repository, referencing the
RFC's pull request. This issue can be created by the RFC author or assignee.

By the time an RFC is merged, we should have a pretty good idea of who's going
to implement it. This may or may not be the same person that submitted the RFC.
Large-scale proposals for example may be picked up by the core Concourse team
instead (but obviously that'd be something we agree on prior to merging).

The implementation process itself falls under the normal [Concourse development
process](https://github.com/concourse/concourse/blob/master/CONTRIBUTING.md).


## License

All RFCs, and any accompanying code and example content, will fall under the
Apache v2 license present at the root of this repository.
