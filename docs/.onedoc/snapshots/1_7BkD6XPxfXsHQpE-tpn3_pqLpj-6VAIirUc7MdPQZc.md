# “Blue doc” design doc template

```
#begin-approvals-addon-section

See go/g3a-approvals for instructions on how to add reviewers. Do not edit this section manually.
```

**Self link:** [go/bluedoc](http://goto.google.com/bluedoc)

**Visibility**:  *See [go/data-security-policy](https://goto.google.com/data-security-policy) for definitions if you want to change this.*

**Status**: 

**Authors**: , 

**Contributors**: , 

**Team**: *Provide a [team/](http://team/) link to the team owning this design*

**PRD**: *This is where the go/ link to your [PRD](https://moma.corp.google.com/glossary/prd) should be. Use [go/indigodoc](http://go/indigodoc) to write it.*

**Tracking Buganizer issue/hotlist**: b/

**Last major revision**: 

*Note: Instead of using a table of contents, use the outline view on the left of the page. It can be edited manually if you don’t want to show detailed headings.*

*Sections of this template have been curated by many experienced SWEs, so while not all sections of this template will apply to all systems, please use good judgment before you delete a section.*

*Send suggestions to* [@design-group](http://who/design-group)*. This document is [go/bluedoc-template](http://go/bluedoc-template).*

*The italicized text in sections below are instructions that should be **deleted** before sending the doc out for review.*

## Context

### Objective

*In 1-2 sentences, describe the business goal of this project. At a high level, what should be different once this design (or an alternative) is in place?*

*For example: "gShoe should keep the user's feet warm and dry even when they walk through the snow."*

*This section should not:*

1. *Describe the problem (use "Background")*

1. *Propose a solution (use "Design")*

1. *Be a detailed list of requirements (use[ go/indigodoc](https://goto.google.com/indigodoc) to create a PRD or link an existing one above).* 

### Background

*Provide context for an unfamiliar reader to understand the proposal.*

1. *What is the problem?*

1. *Why is it important to solve?*

1. *What solutions do we already have that solve similar problems?*

*This section should not:*

1. *Be a lengthy explanation with a lot of details. Instead, prefer to link to other docs where available.*

1. *Include information about your design*

1. *Talk about ideas to solve the problem.*

## Design

### Overview

*One page high-level overview; put details in the next section and background in the previous section. Should be understandable by a new Google engineer not working on the project. Diagrams can be especially useful to quickly convey the shape of the solution. This section does not need to prove that the solution meets the objective or why it is better than alternatives.*

### Infrastructure

*Describe which existing infrastructure (e.g., Spanner, Colossus, Kansas, Muppet, …) you plan on reusing, and how they will interact.*

*Describe in sufficient detail whatever new pieces you're adding. For certain pieces like Chubby, usage has to be approved so contact those teams early, i.e., send them your first draft.*

*For more information, see [go/devguides](http://go/devguides) or [go/stacks](http://go/stacks).*

### Detailed design

*If you are designing a frontend system, you can view a description of this section [here](https://engdoc.corp.google.com/eng/doc/design_doc_templates/designdoc-template-frontend.html?cl=head#detailed_design). If you are designing a backend system, you can view a description of this section [here](https://engdoc.corp.google.com/eng/doc/design_doc_templates/designdoc-template-backend.html?cl=head#detailed_design). You might want to embed code via [code blocks](https://goto.google.com/smartchips#zippy=%2Cinsert-a-code-block), like this:*

```
// Example code can go here.

```

*Weigh the pros and cons of the approach you are recommending and provide a justification for why you chose this approach over those discussed in the “Alternatives considered” section below. For smaller details discuss alternatives inline while reserving the “Alternatives considered” section for overall alternative designs.*

*Make sure readers can find your code.  The tracking issue / hotlist listed in the header is fine if CLs are linked to it; if not, list the directories where the code will go.*

### Alternatives considered

*Clearly list the other potential approaches to meeting the objective that you considered and why the current proposal was ultimately selected. In the rare cases where requirements or system constraints only allow for one possible high-level approach, that should be highlighted here and alternatives to specific details should still be discussed in-line in the detailed design section.*

*An effective strategy is to compare dimensions of the solution space and how they vary across alternatives, such as latency, data staleness, cpu cost, engineering investment, etc. A color-coded table explaining favorability in that dimension and how important it is from green through yellow, orange and red quickly conveys the rationale to a reader, though this is just one possible approach to explain the trade-offs. For example:*

|  | Proposed solution | Alternative 1 | Alternative 2 |
| --- | --- | --- | --- |
| Dimension 1 (e.g. engineering investment) | ➕ Degree of favorability (e.g. trivial implementation) | ～ Degree of moderate negativity (e.g. 1 SWE quarter) | ➖ Degree of negativity (e.g. complete rebuild of new system, 1 SWE year) |
| Dimension 2 (e.g. resource costs) | ～ Degree of slight negativity (e.g. 1K additional GCUs) | ➕ Degree of favorability (e.g. 20k GCU savings) | ➕ Degree of favorability (e.g. 20k GCU savings) |
| Dimension 3 (e.g. end-user perceived latency) | ～ Degree of moderate negativity (e.g. additional 100ms latency at 99%ile) | ➖ Degree of negativity (e.g. additional 5 second latency at 50%ile) | ➖ Degree of negativity (e.g. additional 5 second latency at 50%ile) |

### Dependencies

*Discuss your dependencies on other services. What happens if they're unavailable for a period of time? Distinguish between dependencies that are required for general operation, vs those that provide extra data quality, vs those that are useful for debugging. Which services must be running for your job to start up? Don't forget subtle dependencies like resolving names using DNS or checking the local time.*

*Are you introducing any cycles, such as blocking on a service that can't run if your jobs aren't already up? If you have doubts, discuss your use case with the team that runs the service.*

*For advice on name resolution, configuration and other dependencies, see [go/infradag-policy](https://goto.google.com/infradag-policy)*.

### Migrations

*Describe any data or system migrations.  Incomplete migrations hurt Google: they add enormous complexity, hurt reliability, and make programming unpleasant.  If existing systems must be turned down to achieve the proposed design, describe how that transition will happen.  "How do we get there from here?" is difficult, but often overlooked.  The change originator is generally [responsible for migrations](http://go/churn-policy).*

### Technical debt

*This section is required to meet [Tech Debt Maturity Assessment](http://go/TDMM-Assessment) Level 3.*

*Things to consider*

1. *Is there known technical debt incurred by implementing this design?*

1. *Is there a risk associated with technical debt (e.g., deprecation of underlying technologies) incurred by implementing this design?*

1. *How will technical debt identified during the execution of this design be tracked and followed up with?*

1. *Are there libraries or code paths that would be deprecated? If so, what is the plan to migrate the usage to the new piece.* 

1. *Are there libraries or code paths that would be obsolete? When will they be deleted?*

1. *Are there any servers that could be turned down? If so, please document the turndown process.*

### Potential patents

*Are there potentially patentable inventions in the project? Please note:*

1. *Whether Google's patent counsel has been contacted*. See [go/whichlawyer](http://go/whichlawyer) for PA-specific contact info.

1. *Whether any project aspects may be patentable.*

*The work may be patentable if the design:*

1. *Provides for something not otherwise commercially available.*

1. *Does something better/faster/cheaper than what currently exists.*

1. *Addresses an unresolved need.*

*Please consult our patent counsel to discuss what design aspects should be protected. If something may warrant patent protection, complete an invention disclosure form at [go/patents](https://goto.google.com/patents) before talking to the patent counsel.*

## Quality attributes

*A [quality attribute](http://go/quality-attribute) is a property of a system, such as latency or security. A system’s qualities are orthogonal to its functionality.*

### Security

*Google services and apps are regularly attacked by malicious actors. In this section, describe the countermeasures you have in place to prevent or mitigate each type of potential attack you've identified. Consider attacks from a variety of sources, including both external and internal risks. If your application doesn't require security considerations, explicitly state so and explain why.*

*Designs with substantial changes to Google’s security posture generally benefit from a dedicated threat model. [go/crimsondoc](http://goto.google.com/crimsondoc) can be used to author a threat model. Once your design is finalized, trigger your [security review](http://go/security-review) at [go/securitydesignreview](http://goto.google.com/securitydesignreview). For more insight into security reviews and related requirements, see [Alphabet Privacy & Security Trainings](http://goto.google.com/ise-security-review-training-alphabet).*

### Reliability

*Discuss handling of local data loss, transient errors (e.g., temporary outages) and how they affect your system. Bear in mind that reliability issues with your dependencies can often cause reliability issues for your system.*

*What do you use that inherently provides reliability and redundancy for data (for example, Colossus)? What do you use that requires data to be backed up (for example, Bigtable)? How is the data backed up? How is it restored? What happens between the time data is lost and the time it's restored? In the case of a partial loss, can you keep serving and can you restore only missing portions of your backups to your serving datastore?*

*What are the costs of replicating your data?*

### Data integrity

*Discuss how you will detect, provision for and recover from data corruption and loss.*

*How will you find out about data corruption or loss in your datastores? What sources of data loss are detected? (User error, application bug, storage platform bug, site/replica disaster.) How long will it take to notice each of these types of losses? What is your plan to recover from each of these types of losses?*

### Privacy

*All projects/products must go through a privacy review to ensure that the products we build respect our users and reflect Google's commitment to privacy (more info:[ go/basicprivacypolicy](https://goto.google.com/basicprivacypolicy)).*

*If your project collects/logs/processes/derives/stores/shares data from or about users, customers, vendors or employees, you will definitely need a privacy review.*

*If you'd like to discuss your project with a privacy expert, contact your friendly PWG ([go/pwg](http://go/pwg)).*

*They may ask you to start working on a[ Privacy Design Document (PDD)](https://eldar.corp.google.com/), which you can then include in your launch review when you're ready.*

*Follow this guide for how to create a launch and submit the required privacy documents:[ go/privacy-launch-guide](http://go/privacy-launch-guide).*

### Scalability

*How does your system scale? Consider both data size increase (if applicable) and traffic increase (if applicable).*

*Please consider the current machine situation: adding more machines might take much longer than you think or might not happen during the lifetime of your project. What initial resources will you need? Plan early and carefully. Also, general machine utilization is a concern, using more resources than you need will block expansion of your service.*

### Latency

*This section can be skipped if you are not designing a server on a human facing path. What latency do you need/expect? In particular, make sure you understand the bottlenecks of the pieces you're reusing. All services should define latency targets.*

*Before filling in this section, see these pages for content on latency:*

* *Learn the basics about latency: [go/latency-basics](https://goto.google.com/latency-basics)*

* *Learn about latency metrics: [go/latency-metrics](https://goto.google.com/latency-metrics)*

* *Deeper dives on client-side latency measurements: [go/gws-csi-timings](https://goto.google.com/gws-csi-timings)*

### Abuse

*As your service/feature gets popular, it will be subject to abuse (incl. spam). Describe the potential for abuse of your service/features. [Contact the Common Abuse Tools Team](https://mail.google.com/mail/u/0/?fs=1&tf=cm&source=mailto&to=abuse-eng@google.com) for more information on tools to fight this.*

*Some things you may want to watch for:*

1. *Spammers often create accounts in bulk to spam a service. How would you detect that? If your users sign up using gaia, GRADS spam score may help to find such bulk account creation.*

1. *Spammers often perform a large number of activities in a short time period, something that normal users do not engage in. You may want to consider rate limiting users' activity (e.g., #api calls per minute, emails sent per day, etc.). Bouncer and QuotaServer are available for this purpose.*

1. *You may consider classifying spam and abusive content by algorithmic means. The general purpose spam classifier (SpamIAm) or the repository of objectionable content (Ocelot) are available to aid in this effort.*

1. *Do you provide users a way to report abuse and have a tool for your product ops team to review such reports? You may consider integrating with the Common Abuse Review Tool (CART) for this purpose.*

1. *Does your service/feature serve ads, allow your users to serve ads, involve changes to ad formats, introduce a new type of ad or present a new type of interaction with ads? Contact [adspam-design-review@google.com](mailto:adspam-design-review@google.com). Let them know your plans, and ask for a review to ensure that you won't be disrupting existing filtering or giving spammers an opportunity to spam more effectively. For more information, see the [review checklist](https://sites.google.com/a/google.com/adspam/Home/adspam-design-review-checklist).*

### Accessibility

*There are over **1 billion people in the world with some form of disability**, and it is important to ensure that your product works for these users with accessibility needs. Not only is accessibility (“a11y”) part of Google's mission to make the world's information "universally accessible", but there are strong legal and business reasons to ensure that your product is accessible. The Google-wide standard for measuring product accessibility is the Google Accessibility Rating, and most product areas (including for internal-only products) require that a product reach a rating of GAR 4 before launch. Accessibility should be considered from the earliest design stages, and below are some resources to help you get started:*

1. *Review the accessibility tests that will be required to pass in order to reach [Google Accessibility Rating 4](http://go/gar)*

1. *Understand common [accessibility personas](http://go/a11y-personas)*

1. *Review the many resources available to you at [go/accessibility](http://go/accessibility)*,

1. *Consult early and often via [accessibility office hours](http://go/a11y-oh) and with the [“a11y” tag on YAQS](https://yaqs.googleplex.com/eng/t/a11y)*

1. *Contact the Accessibility Working Group with any additional questions*

### Testability

*Consider the external systems your system depends on (Spanner, Flume, etc.).*  

1. *How fast will your[ unit test cycle](http://go/unit-testing) run?*  

1. *What test infrastructure (e.g.[ test doubles](http://go/choose-test-double)) from external systems will you use?*

1. *Does the external system have a test instance for[ integration testing](http://go/integration-testing)?*

1. *Do you need any tests that run outside of the standard unit test cycle (e.g. presubmit, postsubmit, or release)?*

*Consider future systems that will depend on yours.*

1. *What test facilities (e.g.[ test doubles](http://go/test-doubles), staging environments, local instances) will you provide so that they can run integration tests?*

1. *How will you guarantee that changes to your system won't break those that depend on it?*

*Some systems, like search rankings or LLM-based text generators, return results that aren't "right" or "wrong", only "better" or "worse". If this describes your system, consider how you will evaluate it:*

1. *Are there standard metrics to evaluate how your system performs, or will you define new ones?*

1. *Do these metrics require you to collect and store live data?*

1. *How will evaluation metrics be integrated into your release or other processes?*

### Internationalization & localization

*Define your language scope and roadmap based on [go/google-languages](http://go/google-languages), or figure out how your product will at least eventually scale to more countries and languages. Keep in mind that even a US-only product should work in more than just English, and that some products in some countries need a fair amount of lead time, e.g. for data and business partnerships, legal and compliance.*

*Consider how your product will apply to the [Next Billion Users](http://go/nbu)*

*Partner with [your Localization Project Manager (LPM)](http://go/gloc-who) early: they can help with more than just [translating your UI](http://go/transconsole).*

*Follow the [i18n FrontEnd best practices](https://g3doc.corp.google.com/i18n/g3doc/bestpractices_swe.md#frontend)*

*Consult early: [go/i18n](http://go/i18n)*

### Compliance

*What are the compliance needs, such as [DMA](http://go/dma), [SOX](http://go/sox), [Takeout](http://go/takeout), or [workspace data location requirements](http://go/workspace-dl)? How will the system stay in compliance? Do you need to connect to any auditing, reporting and enforcement mechanisms?*

## Project management

### Work estimates

*In addition to measuring a project's quality, you need to measure its progress. Estimate how long each phase will take (please be detailed; subtask granularity should be roughly one week)*

### Documentation plan

*What documentation will be provided (i.e., add new docs or updated existing docs) to the users and developers of this product?  When will this documentation be available to them?  If contextual documentation is required within this product, please state why.*

*The documentation can be made available to the users and developers closer to or immediately after the completion of the effort.  After creating the documentation, update the plan in the design doc with the date of publication of documentation along with links to the documentation.*

### Launch plans

*What are the launch plans for your project? This includes, but is not limited to:*

1. *What visible changes* will *your project cause on the site*?

1. *What will be the impact on production and/or partners*?

1. *What new servers will be introduced*?

1. *Rough timeline for releasing your project in different languages and countries.*

*For information about the launch process and what to do if your project makes a visible change to the site, see the [Production Launch Review](http://go/plr-start).*

## Operations

### SLAs

*If your application makes any service level guarantees, what mechanisms are in place for auditing, monitoring, etc.? And how can you guarantee the stated level of reliability?*

### Monitoring & alerting

*Are new [CUJ](http://go/CUJ)s being added or an existing one being changed? Does your system monitor them using probers? If so, include the work needed to update them.*

### Logging plan

*All systems that handle user requests **must** log information about these requests for the log analysis system to collect and analyze. Log information allows us to understand system user behavior and system related business metrics. In this section, describe the information that you are going to log. You might find it helpful to refer to [Sawmill Central](https://g3doc.corp.google.com/logs/docs/g3doc/index.md?) for suggestions.*

*Google has strict rules about logging visual elements on search results pages that are not search results or ads. See [Visual Element Logging](http://go/ve-logging-howto) and [Feature Logging](http://go/feature-logging-user-guide).*

### Rollback strategy

*In order to improve incident management response, which will result in reduced time to mitigate issues, teams should document and test their ability to rollback major change surfaces. This includes, but is not limited to:*

1. *documenting the overall rollback strategy;*

1. *identifying surface areas which are not able to be rolled back and defining a fix-forward strategy.*

## Document history

*Google Docs has built-in document revision history:  File > Version history > See version history, and you can name versions e.g., v1.0. However, readers must have EDIT permission to see Docs’ built-in history, so use this section if you want VIEWERS and COMMENTERS to see what’s changed.*

