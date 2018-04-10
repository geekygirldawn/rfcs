# Summary

Introduces a new resource interface with the following goals:

* Introduce versioning to the resource interface, so that we can maintain
  backwards-compatibility.

* Support for spaces (concourse/concourse#1707).

* Introduce a more airtight "versioned artifacts" interface, tightening up
  loopholes in today's resource API to ensure that resources are pointing at an
  external source of truth and cannot be partially implemented or hacky.

* Introduce a "notifier" interface. This is to replace a whole class of resource
  types that will not be able to fit in to an "artifact" interface.

* Extend the versioned artifacts interface to support deletion of versions,
  either by an explicit `delete` call or by somehow noticing that versions have
  disappeared.


# Proposal

At this early stage of the RFC, it's easiest for me to just use Elm syntax and
pretend this is all in a type system.

In reality, each of these functions would be scripts with JSON requests passed
to them on stdin. I'm going to use the `Bits` type just to represent which calls
have state on disk to either access (like `put`) or return (like `get`). This
would normally just be the working directory of the script.

## General Types

```elm
-- arbitrary configuration
type alias Config = Dict String Json.Value

-- identifier for space, i.e. {"branch":"foo"}
type alias Space = Dict String String

-- identifier for version, i.e. {"version":"1.2"}
type alias Version = Dict String String

-- arbitrary ordered metadata (we may make this fancier in the future)
type alias Metadata = List (String, String)

-- data on disk
type alias Bits = ()
```

## Versioned Artifacts interface

```elm
spaces  : Config -> Set Space
check   : Config -> Space -> Maybe Version -> List (Version, Metadata)
get     : Config -> Space -> Version -> Bits
put     : Config -> Bits -> (Space, Set Version)
destroy : Config -> Bits -> Dict Space (Set Version)
```

## Notifications interface

```elm
notify : Config -> Notification -> ()

type Notification
  = BuildStarted
      { build : BuildMetadata
      , self : Maybe BuildInput
      }
  | BuildFinished
      { build : BuildMetadata
      , status : Status
      , self : Maybe BuildInput
      }

type alias BuildMetadata =
  { teamName : String
  , pipelineName : String
  , jobName : String
  , buildName : String
  , buildID : Int
  , status : String
  }

type alias BuildInput =
  { space : Space
  , version : Version
  }
```


# Examples

TODO:

- Pull Requests
- Feature branches
- Build matrixes
- Generating branches (and propagating them downstream)
- Semver artifacts
- Fanning out against multiple IaaSes
- Pool resource?
- BOSH deploys


# Summary of Changes

## Overarching Changes

* Add a `info` script which prints the resource's API version, e.g.
  `{"version":"2.0"}`. This will start at `2.0`. If `/info` does not exist we'll
  execute today's resource interface behavior.

* Rather than running `/opt/resource/X`, discover the supported resource APIs by
  invoking `info`. This allows us to be more flexible in what kinds of resources
  we can support (versioned artifacts, notifications, ???), and where the
  scripts may live (`/opt/resource` is very Linux specific).

* Today's resource interface (`/in`, `/out`, `/check`) becomes more specifically
  a "versioned artifacts" or just "artifacts" resource interface.

* Introduction of some sort of schema validation for resource configuration.

* Remove the distinction between `source` and `params`; resources will receive a
  single `config`. The distinction will remain in the pipeline. This makes it
  easier to implement a resource without planning ahead for interesting dynamic
  vs. static usage patterns, and will get more powerful with #684.


## Changes to Versioned Artifact resources

* Add a `spaces` actopm, which is used to discover spaces.

  It's not a verb, partly because most verbs would be ambiguous with `check`,
  and partly because the API is practically stateless; it just returns whatever
  spaces there are.

* Add a `destroy` action, which looks like `put` (can be given files and params
  to determine what to destroy) but returns the set of versions that it
  deleted.

  This is to support bulk deletes, e.g. to garbage collect intermediate
  artifacts after a final build is shipped.

* Change `check` and `get` to always run against a particular space, given by
  the request payload.

* Change `check` to include metadata for each version. Change `get` and `put` to
  no longer return it.

  This way metadata is always immediately available, and only comes from one
  place.

  The original thought was that metadata collection may be expensive, but so far
  we haven't seen that to be the case.

* Change `get` script to no longer return a version, since it's always given one
  now. As a result, `get` no longer has a response; it just succeeds or fails.

* Change `get` and `put` to run with the bits as their working directory, rather
  than taking the path as an argument. This was something people would trip up
  on when implementing a resource.

* Change `put` to emit a list of versions, rather than just one.

  TODO: it's undecided whether `put` should be given a single space, return a
  single space, or return many spaces + versions.

  Technically the `git` resource may push many commits, so this is necessary to
  track them all as outputs of a build. This could also support batch creation.

  To ensure `check` is the source of truth for ordering, the versions returned
  by `put` are not order dependent. A `check` will be performed to discover
  them in the correct order, and then each version will be saved as an output
  of the build. The latest version of the set will then be fetched.

* Change `put` to write its JSON response to a specified file, rather than
  `stdout`, so that we don't have to be attached to process its response.

  This is one of the few ways a build can error after the ATC reattaches
  (`unexpected end of JSON`). With it written to a file, we can just try to read
  the file when we re-attach after seeing that the process exited. This also
  frees up stdout/stderr for normal logging, which has been an occasional
  pitfall during resource development/debugging.


## Introduction of "notifier" resource interface

A resource may now implement a "notifier" interface. Notifications must be
explicitly enabled in the pipeline. This is so that a resource type can't
suddenly decide to include a notifier and start doing surprising things.

A notifier has a single hook, `notify`, and receives some sort of JSON payload
describing the event. The event is purely for side-effects and has no
implications for pipeline semantics.

If a notification fails to send, the build shall error. To be honest, I haven't
thought about this much yet, but I think it's better to be conservative.

A resource type may implement both "artifacts" and "notifier" interfaces. If a
resource implements both, it will be given information about its occurrence in
the build (space and version). This will be crucial for e.g. reporting the
status of a pull request or commit back to GitHub.

Add support for a "notifier" resource type. They have one hook, `notify`,
which is invoked with various types of notifications
(`build_started`, `build_finished`, as the poster children).


# Caveats

Terminology is now even more confusing. "Resource type" could mean either the
"git" vs. "s3" or "notifier" vs. "artifact".


# Open Questions

## Can we reduce the `check` overhead?

With spaces there will be more `check`s than ever. Right now, there's one
container per recurring `check`. Can we reduce the container overhead here by
requiring that resource `check`s be side-effect free and able to run in parallel?

There may be substantial security implications for this.


## Is `destroy` general enough to be a part of the interface?

It may be the case that most resources cannot easily support `destroy`. One
example is the `git` resource. It doesn't really make sense to `destroy` a
commit. Even if it did (`push -f`?), it's a kind of weird workflow to support
out of the box.

Could we instead just have `put` and ensure that we `check` in such a way that
deleted versions are automatically noticed? What would the overhead of this be?


## Should `put` be given a space or return the space?

The verb `PUT` in HTTP implies an idempotent action against a given resource. So
it's intuitive that the `put` verb here would do the same.

However, many of today's usage of `put` would be against a dynamically
determined space. For example, most semver workflows involve `put`ing with the
version determined by a file (often coming from the `semver` resource). So the
space isn't known statically at pipeline configuration time.

What's more, the resulting space for a semver push would only be `MAJOR.MINOR`,
excluding the final patch segment. This is annoying to have to explicitly
configure in your build.

If we instead have `put` return both the space and the versions, this would be a
lot simpler.


## Should a notifier resource be able to "observe" another resource?

This would allow e.g. a generic `git` resource and specific
`github-commit-status` resource type rather than having to bake them all
together.

The downside here is that it would be introducing cross-resource-type interface
contracts. Resource A would have to understand resource B's versions/spaces.
This gets more possible with schema verification but still feels risky.


# New Implications

Here are a few use cases that resources were sometimes used for inappropriately:

## Single-state resources

Resources that really only have a "current state", such as deployments. This is
still "change over time", but the difference is that old versions become invalid
as soon as there's a new one.

These can now be done by always marking the old versions as "deleted".


## Non-linearly versioned artifact storage

This can be done by representing each non-linear version in a separate space.
For example, generated code could be pushed to a generated (but deterministic)
branch name, and that space could then be passed along.