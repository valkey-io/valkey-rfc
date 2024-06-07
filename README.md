Valkey RFC
==========

This is a collection of feature proposals and descriptions of changes to Valkey
that require some more details than just the text in a pull request or an issue.
It is loosely inspired by RFCs and by Python's enhancement proposals (PEP).

Each feature or larger topic is described in a markdown file that's named in
uppercase and ends in `.md`. These files are not formally numbered, but we use
the pull request number that added an RFC to refer to the change. For example,
this description in the README.md file was written in RFC #1.

Workflow
--------

An RFC starts off as a pull request. It's reviewed for formatting, style and
consistency. Then the proposal is merged. This doesn't mean that the feature is
approved for inclusion in Valkey. It's still just a proposal.

Each file has a status like "draft", "suggested", "approved" or "rejected". The
possible statuses are yet to be decided.

The core team can later change the status and make changes. For larger changes,
the PR making the change is mentioned too and can be referred to by their
respective pull-request numbers.

What's useful to include?
-------------------------

This is not a strict requirement of what to include in each file. What's useful
to include may vary from case to case.

* Status and the numbers of the PRs adding or making major changes to the file.
* Abstract. A few sentences describing the feature.
* Motivation. What the feature solves and why the existing functionality is not
  enough.
* Rationale. Why certain design decisions have been made. Comparisons with
  similar features in other projects.
* Specification. A more detailed description of the feature.
* Links to related issues and PRs on this and other repos.
