# gha-trickery

Experiments with namespace confusion in GitHub Actions.

## Hypothesis

GitHub Actions has `uses:` references, like so:

```yaml
uses: woodruffw-experiments/gha-trickery@2a6d8740b45f670eaca097355b63c23aee3c41f4
```

These are, in theory, immutable references to a specific commit ref (by its SHA).
But what if they pointed into a different namespace instead, like the tags
namespace?

## Outcomes

Turns out this doesn't really work.

1. GitHub doesn't allow full-length SHA refs as tag names:

    ```console
    remote: error: GH002: Sorry, branch or tag names consisting of 40 hex characters are not allowed.
    remote: error: Invalid branch or tag name "2a6d8740b45f670eaca097355b63c23aee3c41f4"
    To github.com:woodruffw-experiments/gha-trickery.git
    ! [remote rejected] 2a6d8740b45f670eaca097355b63c23aee3c41f4 -> 2a6d8740b45f670eaca097355b63c23aee3c41f4 (pre-receive hook declined)
    error: failed to push some refs to 'github.com:woodruffw-experiments/gha-trickery.git'
    ```

    ...so `git tag <SHA-ref>` from an evil branch won't work.

2. What about truncated refs, like this?

    ```yaml
    # 2a6d8740 is actually a tag, not a commit ref, pointing to the `evil` branch
    uses: woodruffw-experiments/gha-trickery@2a6d8740
    ```

    This actually works, in the sense that the `evil` branch is used:

    ```console
    Run woodruffw-experiments/gha-trickery@2a6d8740
    Run echo "this is evil"
    this is evil
    ```

    ...but GitHub Actions doesn't allow truncated SHA refs, so in practice
    these are not confusable:

    ```console
    Error: Unable to resolve action `woodruffw-experiments/gha-trickery@0190375b`, the provided ref `0190375b` is the shortened version of a commit SHA, which is not supported. Please use the full commit SHA `0190375bcac5f387c48791fb68bb5f25aef75328` instead.
    ```

TL;DR: there's a limited amount of namespace confusion, but seemingly not enough
to trick a pre-existing workflow into switching from a commit ref to a tag ref.

## Other avenues

`uses:` supports other namespaces that are confusable, like tags and branches.

GitHub seems to prefer a branch over a tag when they conflict:

```console
# good-branch is safe in branches namespace, evil in tags namespace
Run woodruffw-experiments/gha-trickery@good-branch
Run echo "nothing to see here"
nothing to see here
```

So, there's a limited confusion vector here:

1. A workflow has `uses: foo/bar@good`, which is normally a reference to a "safe" tag (in the tags namespace)
1. A disgruntled maintainer pushes a malicious branch named `good`, **without** deleting the original `good` tag
1. The original workflow now uses the malicious branch
