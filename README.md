# Socky API specification

This repository will handle current specification of Socky API and all of it's components. Specification will be handled in file SPECIFICATION.md.

## Development of API

All development should be handled in "unstable" branch. After all issues are fixed and proposed changes of API implemented release should be frozen, current state of repository tagged and merged to "master" branch.

## Proposing changes to API

Changes could be proposed using 2 methods: creating issue or making pull request. Both will be discussed and pulled after accepting.

## API version and base implementation version

Base implementations of Socky will be written in Ruby and Javascript. Any part of this implementations should be compatible to equal or nearest lesser version of API. If some bigger changes will force increasing version of any component of implementation the in order to preserve previous point it is possible to skip API version numbers(i.e. jumping 2 numbers up)