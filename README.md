# Gasp Diagrams
Repository, where all diagrams are stored.

Set `plantuml:{filename}` as a fence information. `filename` is used as the file name of generated diagrams. In the following case, `md-sample-sequence.svg` is created.
`filename` is required.

Files will be uploaded to `https://storage.googleapis.com/mangata-diagrams/svg/*` folder that is public and URLs can be used anywhere needed.

## (ETH Rollup MVP) ETH -> Gasp -> Eigen Layer AVS Deposit/Withdrawal flow
`https://storage.googleapis.com/mangata-diagrams/svg/gasp-eth-rollup-mvp.svg`
```plantuml:gasp-eth-rollup-mvp

@startuml

actor       "ETH Metamask User"       as user

participant "Gasp ETH Contract"   as gaspcontract

box "Bob - dude who runs Gasp Collator + Sequencer as one service" #LightBlue
participant "Sequencer (Bob)"   as sequencer
participant "Collator (Bob)" as collator
end box

participant "Gasp Updater" as updater

participant "Eigen Agregator & TM" as agregator

collections "Gasp Finalizer (AVS)" as operator
collections "Eigen ETH Contracts"   as eigencontract

user --> gaspcontract: Trigger deposit of 1 WETH token 

gaspcontract --> gaspcontract: Checks if ERC20 token exists on the network

collator --> collator: It is collators (Bob) order to produce a block and he just produced it

sequencer --> collator: (Subscription) Checks weather my collator just built block
sequencer --> collator: Checks the sequencer_latest_processed_block and sequencer_latest_processed_transaction_id

sequencer --> collator: Fetch PENDING_REQUESTS (reads)

 loop for each PENDING_REQUESTS read
 
 sequencer --> gaspcontract: get pending reads based on lastProccessedRequestOnL1 and lastAcceptedRequestOnL1
 
 sequencer --> sequencer: compare PENDING_REQUESTS read and pending read from ETH contract. Using hashing, or byte array value.

    alt PENDING_REQUESTS read is NOT same as on ETH contract
      sequencer --> collator: Submit Cancel request extrinsic
    else if the values match
        note over sequencer
          We dont need to do any action.
        end note
    end
 
 end

sequencer --> gaspcontract: Read all ETH dep/with based on sequencer_latest_processed_block sequencer_latest_processed_transaction_id
sequencer --> collator: Submit provide_l1_read extr. with all fetched ETH dep/with

collator --> collator: store new value into pending_l1_reads with dispute period (block number)

group Separate action on Collator (maybe will move to separate UML)

  loop for each block at end_dispute_period
  
    loop for each l1_read
    
    
        collator --> collator: Validates if Sequencer have enough Stake to submit
        collator --> collator: Removes the READ right from sequencer (node storage update)
    
      alt l1_read is DEPOSIT
          collator --> collator: Deposits validations (balance,...)
          alt ETH address DOES exists
            collator --> collator: Fetch Gasp version of ETH adress
          else ETH address does NOT exists
            collator --> collator: Creates new ETH compatible Gasp address
          end
          alt ETH Asset registry DOES exists
            collator --> collator: Fetch Gasp version of ERC20 token
          else ETH address does NOT exists
            collator --> collator: Register new asset registry and use it
          end
        collator --> collator: Mint token for user
      else l1_read is WITHDRAWAL
        collator --> collator: Withdrawal validations (balance,token existence,...)
        collator --> collator: Burn token for user
      else l1_read is DELETE_PENDING_UPDATES
         note over collator
          This action is triggered from Gasp contract when dep/with is confirmed by the updater.
        end note
        collator --> collator: Deletes all processed pending updates
      else l1_read is CANCEL_RESOLUTION
        note over collator
          This action is triggered from Gasp contract when cancelation is resolved. It will be resolved by comparing the reads.
        end note
        alt CANCEL_RESOLUTION is CANCEL_APPROVED
          collator --> collator: Find malicious l1_read in history with Sequencer address
          collator --> collator: Call slash_sequencer(seq_address)
        else CANCEL_RESOLUTION is CANCEL_NOT_APPROVED
          collator --> collator: Find malicious cancel request in history with Sequencer address
          collator --> collator: Call slash_sequencer(seq_address)
        end
      
      else l1_read is ONLY_INFO_UPDATE
        note over collator
          We dont need to do any action, it serves as information event for our contracts.
        end note
        
      end
    
        collator --> collator: Prepare batches of pending updates based on update_batch_size configuration
        collator --> collator: Prepare and store the Merkle Root of all the WITHDRAWAL and DEPOSIT of pending update batch
        
        note over collator
            Pending updates should have identifier
            We need to have an API to fetch proofs based on the pending update identifier
            Our merkle leafs should be unique
        end note
        collator --> collator: Store succesfull WITHDRAWAL or DEPOSIT to pending_updates with Merkle Root and Proofs
        collator --> collator: Returns back the READ right for sequencer (node storage update)
    
    end
    
  
  end
end

agregator --> collator: Reads that new N block(s) was produced by some collator
agregator --> operator: Submits task: "Finalize GASP blocks"
operator --> collator: Reads pending_updates hashes and Merkel Root of pending_updates
note over operator
  We need an option to switch off the block validation.
end note
operator --> operator: Validates N blocks and do storage proof - N should be configurable
operator --> operator: Prepare the Merkel Root of pending_updates as task response
operator --> operator: Sign the response with operator PK
operator --> agregator: Returns finished task
agregator --> eigencontract: Submits TX with Merkel Root information
eigencontract --> eigencontract: Stores Merkel Root information with batch identifier
eigencontract --> eigencontract: Stores pending_updates hashes in a key-value storage

updater --> collator: (Subscription) Checks weather my collator just built block
updater --> eigencontract: Subscribed for block finalisation
updater --> eigencontract: Fetch the eygen layer Merkel root
updater --> collator: Read all batched penfing_updates?
updater --> gaspcontract: Executes TX on ETH with the Merkel Root coresponding with pending_update batch identifier

gaspcontract --> gaspcontract: Stores Merkel Root with pending_update batch identifier

note over gaspcontract
  Ferries TODO
end note

alt pending_update is DEPOSIT
  user --> collator: Get the pending update.
  user --> eigencontract: Get the proofs of pending_update
  user --> gaspcontract: Close deposit (bringing pending_update + signed proof from Eigen layer)
  gaspcontract --> gaspcontract: Validate the proofs
  gaspcontract --> user: Locks amount to the contract.
else l1_read is WITHDRAWAL
  user --> collator: Get the pending update.
  user --> eigencontract: Get the proofs of pending_update
  user --> gaspcontract: Close withdrawal (bringing pending_update + signed proof from Eigen layer)
  gaspcontract --> gaspcontract: Validate the proofs
  gaspcontract --> user: Sends funds to user address
end

@enduml
```

![](./svg/gasp-eth-rollup-mvp.svg)


## Operator list sharing
`https://storage.googleapis.com/mangata-diagrams/svg/operators-list-sharing.svg`
```plantuml:operators-list-sharing

@startuml

box "Eigen Layer" #LightBlue
collections       "Gasp Finelizers (AVS)"       as operator
participant       "Aggregator"       as aggregator
participant       "Eigen Layer contract"       as eigencontract
participant       "Gasp Eigen contract"       as gaspeigencontract
end box

box "Gasp Rolldown" #LightYellow
collections       "Arbitrum OR Gasp Contract"       as arbcontr
collections       "Arb OR Eth Updater"       as updater
end box

note over aggregator
  Trigger is N block or registration event
end note
aggregator --> operator: Creates a task "Share operator list" 
operator --> eigencontract: Fetch the the current aggregated BLS key and operators stake value
operator --> aggregator: Signed response of aggregated BLS key and operators stake value
note over aggregator
  Currently we will sign with older
end note
aggregator --> gaspeigencontract: Signs the task response
gaspeigencontract -> gaspeigencontract: Verifies the new operator list with quorum threshold
gaspeigencontract -> gaspeigencontract: (optional) Verifies the state, not only the quorum threshold
updater -> gaspeigencontract: On operator list change event, fetch the new list information
updater -> arbcontr: Submits new information 
arbcontr -> arbcontr: Validates the new list with older aggreagated key and stake information
alt operator list is VALID
  arbcontr -> arbcontr: Updates the storage with new aggregated key and operator stakes.
else operator list is INVALID
  arbcontr -> updater: Failed tx, no update
end

@enduml
```

![](./svg/operators-list-sharing.svg)


## Metamask EVM signing
`https://storage.googleapis.com/mangata-diagrams/svg/gasp-metamask-signing.svg`
```plantuml:gasp-metamask-signing

@startuml

actor       "Metamask User"       as user


participant "app.gasp.finance" as app

participant "SDK" as sdk

actor       "Metamask Wallet"       as met

actor       "Polkadot Wallet"       as pol

box "Gasp Node" #LightBlue
participant "Gasp RPC API"   as rpc
participant "Gasp Extrinsics API" as ext
end box

user --> app: Click wants to SWAP Token A for token B
app --> sdk: sign transaction with unified sign SDK method

alt METAMASK wallet is used
  sdk --> rpc: Specific API to encode my sign method with params
  rpc --> sdk: Returns encoded JSON representation of calling method
  sdk --> met: Request to sign tx with aritrary JSON encoded data received from Gasp RPC
  met --> user: Opening the signing popup
  user --> met: Signs the tx
  met --> sdk: Return signature
  sdk --> ext: Execute (multy)SWAP with signature from Metamask
  ext --> rpc: Decode the JSON data with same method it was encoded
else POLKADOT wallet is used
  sdk --> pol: Request to sign tx
  pol --> user: Opening sign popup
  user --> pol: Signs tx
  pol --> sdk: Returns sidned tx
  sdk --> ext: Execute (multy)SWAP with signature from Polkadot compatible wallet
end
  ext --> ext: Executes the SWAP
  ext --> sdk: successfull tx execution
  sdk --> user: successfull tx execution

@enduml
```

![](./svg/gasp-metamask-signing.svg)

## Gasp BE team workflow and release process
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

## Gasp FE team workflow and release process
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
