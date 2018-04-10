# AsyncChain - type-safe, multi-step asynchronous tasks

AsyncChain is a Haskell library aiming to provide safe and convenient way for building and handling multi-step asynchronous tasks.

*The library and this README file are not ready yet. The code contains only an initial idea implementation and is completely unusable in current form*

## Basic idea

I will present the basic goal of library in a use-case example.  
The library when finished will allow library authors to create interfaces returning "handles" to multi-step Async-like object with logic of handling steps encoded into its type.    
Let's say we have a function returning an action of inclusion of a transaction into some kind of ledger. 

```haskell
includeTransaction :: Untrusted Tx -> IO (AsyncChain '[ AsyncStepManual Tx, AsyncStepAutomatic] 
                                                     -- ^ ----------------- ^ ---------------- > types of async steps
                                                     '[ ValidationStatus,   InclusionStatus ]
                                                     -- ^ ----------------- ^ ---------------- > feedback for asyncchain user
                                                  
```
The type  of the function tells us that given not yet validated transaction, there is some action starting two-step task, which when being handled first tries to validate the transaction, when this is finished, a feedback will be returned in the form of object of type ValidationStatus. Should the validation succeed and return a valid transaction (object of Tx), the next step is to include it in the ledger which may take some time, after it's done, handler is provided with an object of InclusionStatus.
The type may look uneasy but it is automatically derived by the library when AsyncChain is built. 

### AsyncChain building

How can the ``` includeTransaction ``` be implemented?  
Let's say we have some functions:
```
data ValidationStatus = ...
data InclusionStatus = ...

validateTx :: Untrusted Tx -> IO (ValidationStatus, Maybe Tx)
validateTx = ... -- returns Nothing when validation fails

includeTx :: Tx -> IO InclusionStatus
includeTx = ... -- computationally expensive process of transaction inclusion
```

AsyncChain library uses a bit of type-level computation to build AsyncChain object from simple (a bit annotated) IO actions:

```haskell
includeTransaction :: Untrusted Tx -> IO (AsyncChain '[ AsyncStepManual Tx, AsyncStepAutomatic] 
                                                     '[ ValidationStatus,   InclusionStatus ]
includeTransaction untTx = buildAsyncChain $ start (validateTx unTx)
                                               &:> (nonFallSimple includeTx)
```

And here's short explanation of used library functions:  
```buildAsyncChain``` - transforms an intermediate chain builder expresion into a AsyncChain and starts first step immediately.  
```start``` - demarks start of chain builder expression  
```(&:>)``` - reverse "cons"  for async actions appending   
```nonFallSimple``` - one of async actions type annotations

### AsyncChain handling / running - TODO

*Notice: This section is not finished and doesn't yet present examples showing usefulness of the library*  

Here's how ``includeTransaction`` AsyncChain can be handled/run:

```

exampleHandler = (handleValidationStatus, mockValidTransaction) >:$ handleInclusionStatus >:$ HFin)

example1 = do
  asc <- includeTransaction aTx
  waitHandleSteps exampleHandler asc -- turns asyncchain into monolithic synchronous operation
  
exampleHandler2 = -- ... handle some of the steps


example2 = do
  asc <- includeTransaction aTx
  (results, ascRest) <- waitHandleSome exampleHandler2 -- synchronously handle some of steps and collect results
  ... -- do something with results
  as <- toAsync ascRest -- given that all of the remaining steps are automatic we can turn ascRest into a normal monolithic Async
  
  
```
