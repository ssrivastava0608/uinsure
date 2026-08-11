# uinsure

Whats good about the pipeline:

- blue/green deployment
- DR using slot swap

What has been improved:

- Splitted for build and deployment to make sure they are separate from each other. The artifact can now be build once, and redeployed again and again without the need to build again
- CI enabled on main/release/feature/hotfix branch
- Templates with variables added to make sure code is not re written for each env and same block of codes can now be used for differenet env by assing different variables
