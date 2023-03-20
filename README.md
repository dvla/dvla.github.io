# DVLA Engineering

[https://dvla.github.io/](https://dvla.github.io/)

## Content Guidelines

Post content must have a strong technical focus with other engineers being the primary audience. There are other communication channels for non-technical content and delivery or service updates such as [dvladigital.blog.gov.uk](https://dvladigital.blog.gov.uk/).

### Content Guidelines - Security

Consider if the content of your post, including and images, has any security concerns:

- would the information be revealing to an attacker?
- is the content too revealing of backend processes?
- do we need to make make specific details public?

Post content must not include:

- identifiers for production infrastructure e.g. IP addresses, network ranges, cloud provider IDs etc
- data relating to real customers or staff members

If you have any doubt please seek support from the cyber security team before publishing your content.

### Content Guidelines - Communication

You must consider if the content of your post could cause damage to the reputation of the organisation.

Your post content should not describe, or be directly related to:

- changes that impact staff experience e.g. automation of work, changes to the types of work undertaken
- changes that impact customers experience e.g. changes to customer process, customer waiting times
- specific DVLA internal or external services

If you have any doubt please seek support from the [External Communications](mailto:external.comms@dvla.gov.uk) team before publishing your content.

## Approval Process

All changes to content must be approved by at least one person from the engineering review control group. This takes the form of a github pull request and is automatically enforced.

Formal approval within the wider organisation is not required by default but may be requested based on e.g. security or reputational concerns.

---

## Installing Hugo

### MacOS

```bash
brew install hugo
```

## Instructions for viewing

This module contains a theme that is a submodule so cloning is recommended with this flag

```bash
git clone --recurse-submodules $repo-url
```

If the repo is already cloned then you will need to execute the following command to populate the themes/book dir

```bash
git submodule update --init --recursive
```

After cloning the repository to your local machine, navigate to the repository root (dvla-reference-architecture).

on command line:

```bash
git checkout master                   'ensure you are on the master branch'
git pull                              'get latest changes from master branch'
hugo server -D                        'starts site on localhost:1313'
```
