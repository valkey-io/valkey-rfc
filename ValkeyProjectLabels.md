---
RFC: 3
Status: Proposed
---

# Valkey Project Labeling RFC

## Description

As this project is still rather new it might be good time to order some of our labeling in order to improve issues/PRs tracking and filtering.
While it is always possible to use flat labeling hierarchy, Keeping labels in a logical hierarchy will help maintain a clearer state and help better manage open issues and PRs development and improve filtering and searches. 
 

## Label classes

### Type Label

Indicate the type of the issue/change. In most cases do not expect a single PR/Issue to hold multiple type labels.

#### Sub-Classes

type::new-feature
    This indicates new major feature and/or capability is introduced.
    For example: RDMA support, Valkey Functions, JSON module support etc...
type::enhancement
    This indicates a change and/or enhancement to an existing feature/ability.
    For example performance improvements, changes to commands arguments etc...
type::rebranding
    Valkey branding related changes
type::refactoring
    Internal code changes
type::bug-fix
    Introduce a fix for identified software defect 
type::polish
    Fix typos, style, etc
type::documentation
    Improvements or additions to documentation
type::bug
    Reporting of a new software defect
type::test-failure
    Reporting of a new test failure
type::question
    Issue meant to ask a question
type::de-crapify
    Correct crap decisions made in the past.

### Status Label

Indication of the CURRENT status of the PR/Issue. In any given time, any PR/Issue should have only a single state label (since it cannot exists in 2 different states) 
After a PR was merged it is expected to have no status label (or we will can consider adding a "status::completed" label)

#### Sub-Classes

status::wontfix
    Indicates this will not be worked on. (Mostly for terminating issues)
status::help wanted
    Indicates the issue/PR is nor being worked on and is pending for available assignee 
status::major-decision-pending
    Major decision pending by TSC team
status::pending-refinement
    This issue/request is still a high level idea that needs to be further refined
status::pending-test-coverage
    This PR is pending introduction of complete test coverage
status::pending-missing-dco
    PRs which are ready to be merged but are blocked because they are missing a dco
status::to-be-merged
    Indicates the PR is reviewed and ready to be merged.

### Require Label

Indication of an external implementation is required following this issue/PR.
Unlike the status and type labels, we can expect to have multiple impact labels for a single issue/PR.

#### Sub-Classes

requires::doc-pr
    This change needs to update a documentation page. Remove label once doc PR is open.
requires::release-notes
    This issue/PR should get a line item in the release notes.

### Impact Label

Indicates different aspects which might be impacted by this change/issue. 
Unlike the status and type labels, we can expect to have multiple impact labels for a single issue/PR.

#### Sub-Classes

impact::breaking-change
    Indicates a possible backwards incompatible change.
impact::performance
    Indicates a possible performance impact.
impact::new-commands
    Indicates new commands/subcommands will be added as part of this PR/Issue
impact::connectivity
    Indicates changes to the Valkey connection layer (eg RDMA support, TCP/TLS related changes etc...) 
impact::cluster
    Indicates changes impact ONLY when valkey is run in enabled cluster mode   
impact::non-cluster
    Indicates changes impact ONLY when valkey is run in disabled cluster mode
