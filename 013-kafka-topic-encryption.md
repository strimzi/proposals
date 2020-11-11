<!-- This template is provided as an example with sections you may wish to comment on with respect to your proposal. Add or remove sections as required to best articulate the proposal. -->

# Proxy-Based Kafka Per-Topic Encryption

*Provide a brief summary of the feature you are proposing to add to Strimzi.*

## Current situation

*Describe the current capability Strimzi has in this area.*


Typically Apache Kafka Systems use disk or file system encryption when
encryption at rest is required. This is *NOT* sufficient for example
to meet many enterprise's compliance standards for sensitive personal information data.

## Motivation

*Explain the motivation why this should be added, and what value it brings.*


There are very  stringent requirements in regard to who  is allowed to
view certain types of enterprise data. For example, financial data may
only be viewed by  certain mandated financial officers.  Consequently,
relational databases  support _database level_  encryption.
This avoid the problem whereby third parties, such as system admins due
to their elevated privileges to the system on which the applications
run, have direct access to the data.


The Problem then can be described are follows: *to allow Apache Kafka to support security best practices and compliance standards with broker based topic encryption*.
## Proposal

Provide an introduction to the proposal. Use sub sections to call out considerations, possible delivery mechanisms etc.

## Affected/not affected projects

Call out the projects in the Strimzi organisation that are/are not affected by this proposal. 
here comes text

## Compatibility

Call out any future or backwards compatibility considerations this proposal has accounted for.

## Rejected alternatives

Call out options that were considered while creating this proposal, but then later rejected, along with reasons why.
