- Feature Name: rename-rtfm
- Start Date: 2020-04-20
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes the renaming of the RTFM framework/ecosystem to another, less overloaded name.

# Motivation
[motivation]: #motivation

As the RTFM framework has gained popularity, there has been [notable](https://twitter.com/deisum/status/1247636837415505922) [surprise](https://twitter.com/r1ckp/status/1242449324283506689) at the name. Although the name is an abbreviation of the "Real Time For the Masses" title, it unfortunately shares an initialism with the expression "Read The Fucking Manual", which has existed [since at least 1979](https://en.wikipedia.org/wiki/RTFM), and likely earlier. This initialism is widespread in the software development industry. Due to this, searchability of "RTFM" is also relatively challenging, unless combined with other keywords, like "RTFM Rust" or "RTFM Embedded".

From the anecdotal standpoint of someone who has attempted to propose the use of RTFM to commercial entities based on technical merit, the name has caused a signficant negative response upon the first impression (laughter, surprise, "wait, really?") nearly every time I have mentioned it, at both an engineering as well as a management level. At the moment, I actively avoid referring to the project as "RTFM", and instead use the full name "Real Time for the Masses" just to avoid this as the first impression, and hope that they are less initially shocked when first introduced to the features of the framework.

In order to improve marketability, as well as remove potential perceptions of hostility due to the name, this RFC proposes a change of name for the project.

Tentatively this RFC proposes "CompOS: Compile Time OS" as a placeholder, however suggestions from the primary RTFM contributors are likely more appropriate.

# Detailed design
[design]: #detailed-design

It will be necessary to rebrand all components of the project, including:

* The GitHub organization
* The codebase
* The documentation
* The Matrix Room
* The main website (https://rtfm.rs)

Existing resources should retained, and should redirect to the new name.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

As this is a rebranding, it would be necessary to communicate to folks that the name is changing, and why. This can be done effectively by sharing through the general embedded working group communication mediums, such as:

* The Rust-Embedded Matrix Room
* The Rust-Embedded Twitter
* The Rust-Embedded Newsletter

# Drawbacks
[drawbacks]: #drawbacks

This rename will require signficant, if not complicated, work to execute.

It is possible that the transition may confuse some users, if the transition is not well communicated.

# Alternatives
[alternatives]: #alternatives

The alternative is to retain the current name, and to continuing colliding with the "Read The Fucking Manual" initialism. This is likely to continue being problematic for adoption from commercial/professional entities.

# Unresolved questions
[unresolved]: #unresolved-questions

* Specific name bikeshedding
