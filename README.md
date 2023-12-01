# Mangata Diagrams
Repository, where all diagrams are stored.

Set `plantuml:{filename}` as a fence information. `filename` is used as the file name of generated diagrams. In the following case, `md-sample-sequence.svg` is created.
`filename` is required.

Files will be uploaded to `https://storage.googleapis.com/mangata-diagrams/svg/*` folder that is public and URLs can be used anywhere needed.

## (ETH Rollup MVP) ETH -> Mangata -> Eigen Layer AVS Deposit flow
`https://storage.googleapis.com/mangata-diagrams/svg/mangata-eth-rollup-mvp.svg`
```plantuml:mangata-eth-rollup-mvp
@startuml

actor       "ETH User"       as user
participant "Mangata ETH Contract"   as mangatacontract

box "Bob - dude who runs Mangata Collator + Sequencer as one service" #LightBlue
participant "Sequencer (Bob)"   as sequencer
participant "Collator (Bob)" as collator
end box

participant "Mangata Updater" as updater

participant "Eigen Agregator & TM" as agregator

collections "Eigen Operators (AVS)" as operator
participant "Eigen ETH Contract"   as eigencontract
participant "Kusama Relay"   as relay

user --> mangatacontract: Trigger deposit of 1 MGA token 

collator --> collator: It is collators (Bob) order to build a block
sequencer --> mangatacontract: Sequencer (Bob) will read ETH contract updates
sequencer --> collator: Submit Extrinsic with new ETH deposits
collator --> collator: Collator (Bob) will produce block

agregator --> collator: Reads that new N block(s) was produced by some collator
agregator --> operator: Submits task: Finalize blocks
operator --> relay: Check finalisation on Relay chain
operator --> operator: (optional) execute try-runtime block validation
operator --> operator: Sign the response with operator PK
operator --> agregator: Returns finished task
agregator --> eigencontract: Submits TX on ETH Contract with the hashed information
eigencontract --> eigencontract: Stored data for each block in a key-value storage
eigencontract --> eigencontract: Removes old block data

updater --> eigencontract: Subscribed for block finalisation
updater --> updater: Stores lates finalized block by Eigen layer
updater --> collator: Subscribed for deposit event with finished dispute period
updater --> mangatacontract: Once required block is finilised and there is deposit that needs to be confirmed, executes TX on ETH


mangatacontract --> eigencontract: Confirms that required hashes match
mangatacontract --> user: Deposit is confirmed

@enduml
```

## Mangata BE team workflow and release process
`https://storage.googleapis.com/mangata-diagrams/svg/be-workflow-and-release.svg`
```plantuml:be-workflow-and-release
@startuml

actor       "Engineer"       as dev
actor       "Reviewer"       as review
actor       "QA Team"       as qa
participant "feature branch" as feature
control "Pull Request" as pr
participant "develop branch" as develop
participant "release branch" as release
participant "main branch" as main

develop --> feature: New branch matching Jira ticket ID and short name is created
dev --> feature: Engineer implements the feature with idividual commits.
feature --> pr: *Pull Request* is created
pr --> pr: e2e Tests executed
pr --> pr: fungible environment spawned
pr --> pr: custom build created
review --> pr: Code review is done by the team
dev --> pr: Code review changes are applied
qa --> pr: QA runs isolated test cases in isolated environments
pr --> develop: Squash commits and merge when all checks passed
develop --> release: We will check out to release branch
dev --> release: Runtime versions incremented and specific commit message triggers the build 
qa --> release: QA will test the RC
dev --> release: Hotfixes will be applied to RC branch
release --> main: Merging into main will automatically deploy on Rococo
release --> develop: Merging into develop
release --> release: Release branch is deleted
main --> main: Maunaul steps to automaticaly deploy images into GCP for Kusama and Polkadot

@enduml
```

![](./svg/be-workflow-and-release.svg)

## Mangata FE team workflow and release process
`https://storage.googleapis.com/mangata-diagrams/svg/fe-workflow-and-release.svg`
```plantuml:fe-workflow-and-release
@startuml

actor       "Engineer"       as dev
actor       "Reviewer"       as review
actor       "QA Team"       as qa
participant "feature branch" as feature
control "Pull Request" as pr
participant "main branch" as main

main --> feature: New branch matching Jira ticket ID and short name is created
dev --> feature: Engineer implements the feature with idividual commits.
feature --> pr: *Pull Request* is created
pr --> pr: Unit tests are executed
review --> pr: Code review is done by the team
dev --> pr: Code review changes are applied
pr --> main: Squash commits and merge when all checks passed
main --> main: Automatic deploy on develop environment
qa --> main: QA team runs requered test cases
dev --> main: Release to production will be triggered by git *TAG*
main --> main: Automation will increase the version and generate changelog
dev --> main: Hotfixes will be commited and released as a new patch version

@enduml
```

![](./svg/fe-workflow-and-release.svg)