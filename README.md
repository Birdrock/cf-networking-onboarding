# CF Networking Program Onboarding
An onboarding program heavily inspired by [cf-onboarding](https://github.com/pivotal/cf-onboarding)

## Networking Program Onboarding 
Networking Program Onboarding is intended to be a _facilitated_ exploration of the networking components of Cloud Foundry, embarked upon with other Networking Program members. It provides:

1. A self-paced learning environment paired with others who are learning too.
1. A coherent, if cursory, overview of a complicated product.
1. Empathy for the customer who uses that product.
1. The opportunity to struggle through these problems and really learn the material.
1. A little knowledge of the breadth of work that the networking program does
1. A little knowledge of how to debug networking components when things don't work as planned

To run an Onboarding Week in your office, **read the [facilitation](FACILITATING.md) docs** 

## Usage
### Import stories to Tracker (from source)
The stories in this repo are divided by epic (e.g. HTTP Routes, Route Integrity, etc) They are provided in .prolific format. To grab the most recent versions of stories from master or another branch:

1. Clone this repo
1. Run `./build` (requires Docker)
1. Import your newly created csv file (`networking-program-onboarding-tracker.csv`) to a new Tracker project
1. WARNING: concatenating CSVs is a risky/inadvisable business, so stories generated by this script are slightly buggy in inconsequential ways (e.g. the first letter or two of a single story title goes missing).

## Contributing
Yes please! Check out the [contributing guide](./CONTRIBUTING.md) for more details.

## Feedback

Here is the link for feedback after Pivots have gone through the onboarding: https://forms.gle/23751ieopY6hbHMM9
