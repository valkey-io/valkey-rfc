---
RFC: 1
Status: Informational
---

Valkey RFC
==========

This repository is a collection of feature proposals and descriptions of changes to Valkey
that require some more detail than just the text in a pull request or an issue.
It is loosely inspired by RFCs and by Python's enhancement proposals (PEP).

Each feature or larger topic is described in a markdown file that's named in
uppercase and ends in `.md`. These files are not formally numbered, but we use
the pull request number that initially added an RFC to refer to the change. For example,
this description in the README.md file was written in RFC #1.

Workflow
--------

An RFC starts off as a pull request. It's reviewed for formatting, style,
consisteny and content quality. The content shouldn't be very vague or unclear.
Then the proposal is merged. This doesn't mean that the feature is approved for
inclusion in Valkey. It's still just a proposal.

Each file has one of the following statuses:

* **Proposed**, meaning the file was added but there's no decision about it yet.
* **Approved**, meaning the core team has made a decision to accept the feature.
* **Rejected**, meaning the core team has made a decision to not accpt the feature.
* **Informational**, for information that is not a feature, like this README file.

The core team (the Technical Steering Committee) can change the status and make
changes. For larger changes, the PR making the change is mentioned too and can
be referred to by their respective pull-request numbers.

What's useful to include?
-------------------------

The Valkey RFC format is not a strict format, but should include the following
sections unless they are unnecessary for the proposal you are submitting.

* Status and RFC number (the pull-request number).
* Abstract. A few sentences describing the feature.
* Motivation. What the feature solves and why the existing functionality is not
  enough.
* Design considerations. A description of the design constraints and
  requirements for the proposal. Comparisons with similar features in other
  projects.
* Specification. A more detailed description of the feature, including why
  certain details in the design have been chosen.
* Links to related material such as issues, pull requests, papers, or other references.
