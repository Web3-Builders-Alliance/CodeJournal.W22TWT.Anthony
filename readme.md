"Practice makes perfect" they say, so let's just begin

# Code journal:

By reading, dissecting and analyzing others code, i should hopefully be able to grasp more of rust /cosmwasm SC concepts.

## Starting point: Cw-plus

Cw-plus repo is the go to repo for any beginner and seemed like a good place to start.

Just noticed that there were two different implementations of cw3 `multisig spec` ie fixed or flexible, which kinda caught my eyes.

### Methodology

To get a better understanding on how each modules articulate with one another, we'll try to analyze contracts file by file, and as follows:

- start with state file to have a general idea of what will be stored
- look at messages and queries enum to see what kind of interactions are possible
- look at possible errors
- look at the implementation itself to get a more comprehensive vision of how this works
