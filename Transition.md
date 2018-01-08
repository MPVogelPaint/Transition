- [Transition](#transition)
- [Production Tracking Beta](#production-tracking-beta)
  - [Batch Module](#batch-module)
    - [Overview](#overview)
    - [Local Storage Batch Design](#local-storage-batch-design)
    - [Loading Design](#loading-design)
    - [Saving Design](#saving-design)
    - [Form components](#form-components)
    - [Guide for new Batch Part and Batch tab](#guide-for-new-batch-part-and-batch-tab)
      - [Component Logic vs Batch Logic](#component-logic-vs-batch-logic)
      - [Required Files for Batch Parts](#required-files-for-batch-parts)
      - [Scaffolding out the Batch Part files](#scaffolding-out-the-batch-part-files)
      - [Scaffolding out the Batch Part component specific logic](#scaffolding-out-the-batch-part-component-specific-logic)
      - [Scaffolding out the Batch Part save logic](#scaffolding-out-the-batch-part-save-logic)
- [Eparts SDS Services](#eparts-sds-services)
  - [Eparts SDS Uploader](#eparts-sds-uploader)
  - [SDS Email Log (Download Distribution Log)](#sds-email-log-download-distribution-log)
  - [SDS Audit](#sds-audit)
- [ITInfo](#itinfo)
  - [Service Desk Export](#service-desk-export)
- [Misc](#misc)
  - [CheckFoxproFileSyncStatus](#checkfoxprofilesyncstatus)
  - [Visual Studio Code Extensions](#visual-studio-code-extensions)
  - [Sproc Object Mapper](#sproc-object-mapper)

---
# Transition
---

# Production Tracking Beta

## Batch Module

### Overview

The idea that we were trying to go for with the batch module is a way to split of all of the parts to a batch to make loading easier.
We also wanted to store the loaded batch information locally to make reloading a batch fast and give a chance for limited offline capabilities.
Storing the batch locally allows changing different batch parts and then committing a single save action to save all the parts.

### Local Storage Batch Design

When a batch is loaded we want to only load the necessary parts of the batch, not the whole thing. For example when we load the `Order` tab we want to only load the data related to that tab.
However, we also want to keep all of the batch parts together in local storage and attached to a single `Batch` object there. This lead to having a `Batch.Order` batch part.
When the batch is stored the first time it will just be a `Batch` object with the `Batch.Order` property loaded with data.
Once we move to the `Complete` tab we will load that data and attach it to the `Batch` object on the `Batch.Complete` property. The single `Batch` object will now have two batch parts,
and this can continue for any number of parts we decide to add to the `Batch`. This gives an amount of modularity to the batch.

We store the batch using a single key `PT-Batch-123456`.
1. The `PT-` signifies that this local storage object should go with `Production Tracking`, this is needed because local storage is shared at a per `Origin` level.
So basically per sub-domain, and since our sites are one the same origin we need this to keep the apps local storage items together.
2. The `Batch-` signifies that the object in local storage is a `Batch` type object. We current store other object there too.
3. The final number string is the identifier for the `MFG.Production_Batch`. The unique key gives a way to keep track of the stored batch. Can't use something that might be temporary like a Lot Number.

When the `Batch` object is loaded and is being consumed by the `Component` we want to also keep track of any changes that occur to the object.
What we do is any time that a form detects a `(ngModelChange)` we bind that to a function that eventually call the `Batch Service`'s Update Local Storage function. That function will take the Batch or Part
and store it Local storage. This can happen immediately as it doesn't appear to be a intensive operation even when local storage is almost full.

The second part to keeping track of any changes is that we actually keep two copies of each `Batch Part` that is loaded.
1. The first copy `originalData` is just that. The original copy of the data that was loaded from the server. This copy of the data shouldn't be updated except when reloading from the server.
2. The second copy of the data is `dirtyData` is the data that should be bound to the templates forms controls, and is editable. This copy of the data will be what is accessible by the user to edit.
Keeping two copies of the data is necessary as we can't whole trust Angular's dirty/pristine state as we may be loading the data from local storage, and Angular's form states will be pristine even if the local storage data is dirty.
With the two copies of the data we can compare them against each other to tell which fields truly are different.
This can be used for a visual indicator for the user, as a way to check if the data has changed before saving, or to see how the data should be reloaded from the server.

In keeping track of local storage we also have a property on the whole `Batch` object called `lastTouched`. This indicates when the last time this Batch Object was loaded by a component.
When local storage is full this is used to determine what items can be purged to make room for new `Batch`es.

Each `Batch Part` also has a `dateLoaded` property and similar to the `lastTouched` it indicates when the last time the `Batch Part` was loaded.
This is used however to determine if the `Batch Part` should be loaded from Local Storage or reloaded from the API.

### Loading Design

A `Batch` and `Batch Part` can be loaded in a couple of different ways.

(```batch.service.ts -> getBatch(...)```)
The first attempt will be to check local storage for the `Batch/Batch Part`. This will setup the `Batch Part` to a initial state if it didn't exist. It will also setup the observable property on the `Batch Part`.

Once the batch is returned to the component to be consumed we can `subscribe` to `obs` observable. We need to subscribe at this level or the subscription will return after the object has returned to the component.
For the `Batch Order` the obs will look like this:
(```batch-order.service.ts -> getBatchDataOrder(...)```)
1. If the local storage `Batch` object exists with that `Batch Part` then we need to check if it is *stale* or not.
2. If it is not *stale* then we can load from Local Storage.
3. If it is *stale* then we need to check if it is *pristine* or not.
4. If it is *pristine* then we can load the whole `Batch Part` (`Batch.Order`) again.
5. If it is not *pristine* then we need to do a partial load. This will leave `dirtyData` fields alone if they are actually *dirty*.
 *Pristine* field will have both copies of the data updated. This allows the user to keep things that they've edited there, but also updated the data to show what is currently known on the server.
6. If the local storage `Batch` object does not exist then it can be loaded from the server normally.

### Saving Design

The saving for a `Batch` is intended to be a single action that can cover multiple `Batch Part`.
The `batch.service.ts -> save(...)` is the main driver of the operation.
How it works is that each `Batch Part` will have it's own `save(...)` method that returns a `ResponseDetail` about the save.
Then when the global save is called that will check with each `Batch Part` that has been loaded and check to see if it is *dirty*. If so it will add that `Batch Part`'s `save` operation to the main `observable`.
When the main observable is ready all the sub parts are subscribed to and triggers those save operations on the server.
When they all complete it will determine if there any of the parts had issues and report that to the user, otherwise it will reload the data from the server to make sure everything is fresh after the save.
We need to be able to call all the `Batch Part`s save operations from the `batch.service.ts` from any component because a `Batch Part` could be changed, then the user might reload the page and come back to another `Batch Part` and save both.

### Form components

Angular allows us an easy way to make mini components good at one thing. Because the bindings for Batch form controls can be tricky with the two sets of data (`originalData` and `dirtyData`) having a mini reusable component allows us to get it right once and then not worry about it again.

### Guide for new Batch Part and Batch tab

This section will hopefully be useful when setting up new Batch Parts and different tabs in the Batch Module.

#### Component Logic vs Batch Logic

Components will interact with their data in different ways to meet the requirements that the component needs. All Batch Parts will load and save their data in a similar way however. These sections will refer to logic in loading or saving a batch part with the global Batch Service as *Batch Part Logic* and logic specific to the current component / implementation details for the component as *Component Logic*.

#### Required Files for Batch Parts

For this example I'll use Batch Complete as existing code to follow.

Files and folder that need to be added / updated for adding the initial *Batch Part Logic*.

1. Component Directory (`batch-complete`)
    1. `batch-complete.component.css` - Should be included during generation, but no Batch Part data is required in this file.
    2. `batch-complete.component.html` - Should be included during generation, but no Batch Part data is required in this file. This will eventually have *Component Logic* that interacts with the *Batch Part*, but not required at the outset.
    3. `batch-complete.component.ts` - Should be included during generation, and we will be adding *Batch Part Logic* to this file for loading *Batch Parts*.
    4. `batch-complete.model.ts` - This will need to be added manually. This will define the shape of the object we get back from the API and that will be part of the `BatchData` object.
    5. `batch-complete.service.ts` - This will house logic for retrieving or storing the *Batch Part's* data from the API and some related logic. Like all the other services this will need to be manually added the module so it can be provided.
2. Batch Directory (`batch`)
    1. `batch.model.ts` - We will add to this file to support the new *Batch Part*.
    2. `batch.service.ts` - We will add to this file to support the new *Batch Part*.
3. Batch Navigation Directory (`batch-navigation`)
    1. `batch-navigation.mock.ts` - We will add or update a record in this file when adding a new tab to the Batch Module.

#### Scaffolding out the Batch Part files

Once these files have been added we can start fleshing them out.

1. Fill out the Batch Part's Model classes - `batch-complete/batch-complete.model.ts`

    First will be the actual model class for the Batch Part (`batch-complete.model.ts`). We should have this model start by matching what fields are being returned from the API.

    ```ts
    export class BatchComplete {
      BatchID: number;
      ...
      FillDetails: BatchFillDetail[];
    }

    export class BatchFillDetail {
      BatchFillDetailID: number;
      ...
    }
    ```

    Because the API will return a sub-collection of `BatchFillDetails` with our `BatchComplete` object, we also define that model in this file too. This may not always be the case as the API could return a sub-collection of a type already defined by another file. Since we consider Fill Details a part of the Batch Completion and not really possible to separable the entities is why `BatchFillDetail` is defined here.

2. Expand the Batch Model class and enum - `batch/batch.model.ts`

    The entire Batch is part of the `batch` object. For that object to hold our new *Batch Part* we need to expand that model and related enum.

    ```ts
    import { BatchComplete } from '../batch-complete/batch-complete.model';
    ...
    export class Batch {
      ...
      Complete: BatchData<BatchComplete>;
      ...
      constructor(id?: number) {
        ...
        this.Complete = null;
      }
    }
    ```

    So in this update we add the Batch Part class as an import so we can use that type later.

    Then we add a property `Complete`, this is how we will access the `Batch Complete` data.
    The added `Complete` property is wrapped in another class `BatchData`. We will get more detail on `BatchData` class later, but it is a wrapper class that gives extra ways to interact with the data. Having our `BatchComplete` class be wrapped in this give s consistent way to do those extra features. (See [Local Storage Batch Design](#local-storage-batch-design) for more detail.)

    Finally we add `this.Complete = null;` to the constructor to make the initial Batch object have a property for the the `Complete` during `batch` object creation.

    In the same file we will add to the enum `BatchPart`:

    ```ts
    export enum BatchPart {
      ...
      Complete = 'Complete'
    }
    ```

    The string value `'Complete'` should match the property name `Complete` and the property name should match the field that we added to the `batch` object: `Complete: BatchData<BatchComplete>`. Adding to this enum makes interacting with the Batch object more consistent and easier to do later.  Using an enum rather than plain strings allows Typescript to enforce data integrity.

3. Update the Batch Part's Service (`batch-complete/batch-complete.service.ts`)

    We want to update the *Batch Part's* service class so that we retrieve data with it.

    ```ts
    // Import Statements
    ...
    export class BatchCompleteService {
      constructor(
        private http: Http
      ) { }

      getBatchDataComplete(batchIdentifier: BatchIdentifier, existingBatchComplete: BatchData<BatchComplete>): Observable<BatchData<BatchComplete>> {
        let batchDataOrderObservable: Observable<BatchData<BatchComplete>>;
        let staleDate: Date = new Date();
        staleDate = subMinutes(staleDate, environment.batchPartStaleLimit);

        if (existingBatchComplete != null && existingBatchComplete.originalData != null) {
          // Batch Order Object exists in local storage
          if (isBefore(existingBatchComplete.dateLoaded, staleDate)) {
            // Batch Order Object exceeds the stale limit.
            if (this.isPristine(existingBatchComplete)) {
              // Stale and pristine we can just reload. No potential data lost.
              batchDataOrderObservable = this.loadBatchComplete(batchIdentifier, existingBatchComplete);
            } else {
              // Stale and dirty we need to reload the originalData copy and warn user.
              batchDataOrderObservable = this.partialLoadBatchComplete(batchIdentifier, existingBatchComplete);
            }
          } else {
            // Batch Order Object is within the stale limit.
            batchDataOrderObservable = Observable.of(existingBatchComplete);
          }
        } else {
          // Batch Order Object doesn't exist in local storage and needs to be loaded fresh.
          batchDataOrderObservable = this.loadBatchComplete(batchIdentifier, existingBatchComplete);
        }

        return batchDataOrderObservable;
      }
    }
    ```

    So in this block we setup the class's initial imports and make the `http` class available to use.

    Then in the `getBatchDataComplete(...)` function we are making a way to return an observable of the `BatchComplete` data from either the API or from the one that is already known in local storage (more on that part later). The logic in the method is checking different properties on the `Batch Data` object of type `Batch Complete`. If the object doesn't exist it will go straight to the API or if the object is *pristine* and *stale*. Otherwise if it is *dirty* or *fresh* then it will load it in a different manner. See the Loading Design section.

    The reason this loading mechanic is more complex is that we want to keep track of two copies of the data with the `BatchData<T>` design. One being editable `dirtyData` and the other only what is returned by the server `originalData`. See the Local Storage Batch Design section for more detail on these ideas.

    This function we've setup calls other functions on the `batch-complete.service.ts` so we will need those too.

    ```ts
    private loadBatchComplete(batchIdentifier: BatchIdentifier, existingBatchComplete: BatchData<BatchComplete>): Observable<BatchData<BatchComplete>> {
      return this.http
        .get(`${environment.webapiRoot}BatchCompletion/${batchIdentifier.BatchID}`)
        .map((r: Response) => {
          let batchComplete = r.json() as BatchComplete;
          return new BatchData<BatchComplete>(batchComplete);
        });
    }
    ```

    This function is the default way to load data from the API. It will setup both copies (`dirtyData` and `originalData`) to what is returned by the server by using the `BatchData<T>` constructor.

    ```ts
    private partialLoadBatchComplete(batchIdentifier, existingBatchComplete: BatchData<BatchComplete>): Observable<BatchData<BatchComplete>> {
      return this.http
        .get(`${environment.webapiRoot}BatchCompletion/${batchIdentifier.BatchID}`)
        .map((r: Response) => {
          let freshBatchComplete = r.json() as BatchComplete;
          return this.refreshBatchComplete(existingBatchComplete, freshBatchComplete);
      });
    }

    private refreshBatchComplete(existingBatchComplete: BatchData<BatchComplete>, freshBatchComplete: BatchComplete): BatchData<BatchComplete> {
      Object.keys(existingBatchComplete.dirtyData).forEach(batchCompleteFieldKey => {
        if (batchCompleteFieldKey === 'FillDetails') {
          this.refreshCollectionField(existingBatchComplete.dirtyData.FillDetails, existingBatchComplete.originalData.FillDetails, freshBatchComplete.FillDetails, 'BatchFillDetailID');
        } else if (batchCompleteFieldKey === 'StaffTimeDetails') {
          this.refreshCollectionField(existingBatchComplete.dirtyData.StaffTimeDetails, existingBatchComplete.originalData.StaffTimeDetails, freshBatchComplete.StaffTimeDetails, 'BatchStaffID');
        } else if (batchCompleteFieldKey === 'SerialFillDetails') {
          this.refreshCollectionField(existingBatchComplete.dirtyData.SerialFillDetails, existingBatchComplete.originalData.SerialFillDetails, freshBatchComplete.SerialFillDetails, 'BatchSerialFillDetailID');
        } else {
          if (existingBatchComplete.dirtyData[batchCompleteFieldKey] === existingBatchComplete.originalData[batchCompleteFieldKey]) {
            existingBatchComplete.dirtyData[batchCompleteFieldKey] = freshBatchComplete[batchCompleteFieldKey];
          }
          existingBatchComplete.originalData[batchCompleteFieldKey] = freshBatchComplete[batchCompleteFieldKey];
        }
      });
      return existingBatchComplete;
    }

    private refreshCollectionField(dirtyDataArray: any[], originalDataArray: any[], freshDataArray: any[], identifierField: string): void {
      dirtyDataArray.forEach(dirtyItem => {
        let originalItem = originalDataArray.find(x => x[identifierField] === dirtyItem[identifierField]);
        let freshItem = freshDataArray.find(x => x[identifierField] === dirtyItem[identifierField]);

        if (typeof freshItem === 'undefined' || freshItem == null) {
          // No fresh item
          if (dirtyItem[identifierField] < 0) {
            // Negative id indicates new item that we can leave alone on the client side.
          } else {
            // Removed from server. TODO: Should remove at client?
            dirtyDataArray = dirtyDataArray.filter(x => x[identifierField] !== dirtyItem[identifierField]);
          }
        } else {
          // Fresh Item found so still exists
          // TODO: This assumes that the collection fields won't themselves have collection fields.
          Object.keys(dirtyItem).forEach(itemFieldKey => {
            if (dirtyItem[itemFieldKey] === originalItem[itemFieldKey]) {
              dirtyItem[itemFieldKey] = freshItem[itemFieldKey];
            }
            originalItem[itemFieldKey] = freshItem[itemFieldKey];
          });
        }
      });
    }
    ```

    These three functions are a way to load the `Batch Part` in a partial sense. This will be called if we only want to leave any modifications that the user has made in place, but still refresh the rest of the data. For a *pristine* field that means both the `dirtyData` and `originalData` copies of that field will be updated. For a *dirty* field that means that only the `originalData` copy of the data will be updated. This prevents the user from losing their work while still being able to see the latest known values on the server.

    The final block we need right now is a `isPristine(...)` function. This function will need to compare each value between the `dirtyData` and `originalData` copies to tell if the value has changed. We cannot rely on the Angular *pristine* value because we are storing it in Local Storage and lose those attributes on storage and retrieval. One hurdle with the pristine check is that it cannot directly compare sub-collections based on something like the item's index in the array. Instead we need to know the identifier or primary key field so that if there are additions or removals from the array we can compare the correct items against each other. There may be an option to simplify this by looping through `Object.keys(...)` result, but you need to be able to traverse into sub-collections correctly as well.

    ```ts
    isPristine(batchComplete: BatchData<BatchComplete>): boolean {
      let result = true;

      // Here for each field in the Model we will compare the two copies of data. If different we will set result to false;

      return result;
    }
    ```

    At this point our service class should be all set to be consumed by our component for reading data. We will return to this file later when we are ready to implement saving functionality, but for now we can continue with other files.
    
    We won't consume this loading logic directly by the component; instead, we will need to go through the `Batch Service` (`batch.service.ts`).

4. Updating the Batch Service (`batch/batch.service.ts`)

    The Batch Service is going to facilitate access to each of the `Batch Part`'s service classes and Local Storage.
    
    We will begin by updating the `import` references and local properties to support the new *Batch Part*.

    ```ts
    ...
    import { BatchCompleteService } from '../batch-complete/batch-complete.service';
    import { BatchComplete } from '../batch-complete/batch-complete.model';
    ...

    export class BatchService {
      ...
      constructor(
        ...
        batchCompleteService: BatchCompleteService
      ) { }
      ...
    }
    ```

    With the property and types available we can expand the `getBatch(...)` function. This function will be called by our Component later to get the batch from local storage and setup the observables for the required batch parts (indicated by the second parameter: `batchParts`).

    ```ts
    getBatch(...): Observable<Batch> {
      ...
      if (typeof batchParts !== 'undefined' && batchParts != null) {
        batchParts.forEach(batchPart => {
          switch(batchPart) {
            ...
            case BatchPart.Complete: {
              if (batch.Complete == null) {
                batch.Complete = new BatchData<BatchComplete>();
                batch.Complete.isStale = true;
                batch.Complete.isPristine = true;
                batch.Complete.dateLoaded = new Date();
              } else {
                batch.Complete.isStale = isBefore(batch.Complete.dateLoaded, staleDate);
                if (batch.Complete.isStale) {
                  batch.Complete.isPristine = this.batchCompleteService.isPristine(batch.Complete);
                }
              }
              break;
            ...
            }
          }
        });
      }
      ...
      batch = this.setupObs(batch, batchIdentifier);
      return Observable.of(batch);
    }
    ```

    With this change the new `Batch Part` will be setup correctly when the batch is requested and asks for that part. What's going on is that we need to setup the object to an initial state if it hasn't been loaded before. And if it has been loaded before we want to check on its status (pristineness and freshness).

    This `getBatch(...)` function we updated calls out to another function `setupObs(...)` that we will also need to update to work with our new `Batch Part`.

    ```ts
    setupObs(batch: Batch, batchIdentifier: BatchIdentifier): Batch {
      ...
      if (batch.Complete != null) {
        batch.Complete.obs = this.batchCompleteService.getBatchDataComplete(batchIdentifier, batch.Complete);
      }
      ...
      return batch;
    }
    ```

    With this we can see it is calling out to the function we just created in the `Batch Part`'s Service that is in charge of returning the `Batch Part`'s data. This function will setup the cold observable to be subscribed to by the Component and store it in the `obs` field of our `BatchData<BatchPart>` property. We don't subscribe to the `obs` at this level because it is an asynchronous action and will instead wait to subscribe to it until we are in the Component of our new `Batch Part`.

    All the other methods (except the save functions) should be able to work on their own without needing an update. The other functions in this file are intended to provide a consistent way to work with the `Batch Part`s. Some are utility functions and some are functions to ensure storing the `batch` in Local Storage is consistent and safe.

5. Adding Batch Part Logic to the Component - (`batch-complete/batch-complete.component.ts`)

    Finally we've gotten to the Batch Part's Component file. This file will house both *Batch Part Logic* and *Component Logic*.

    We will beign with scaffolding out the *Batch Part Logic*. Unlike many other components in the application, these *Batch Part* components won't directly access their respective services. Instead they will interact with the `Batch Service` which will inturn interact with those services. We do this so that the `Batch Service` facilitates consistent access to the service and to local storage.

    So getting started on the component we will add some boilerplate items.

    ```ts
    // We need some imports from Angular and RxJs
    import { Component, OnInit, OnDestroy } from '@angular/core';
    import { ActivatedRoute, Params } from '@angular/router';
    import { Observable } from 'rxjs/Observable';
    import { Subscription } from 'rxjs/Subscription';

    // We need to import the model for this specific Batch Part
    import { BatchComplete, BatchFillDetail } from './batch-complete.model';
    // We need to import the model for the entire batch, the Batch Data type, and the enum for BatchPart
    import { Batch, BatchData, BatchPart } from '../batch/batch.model';
    // We need to import the Batch Identifier model and service to access our batch part correctly.
    import { BatchIdentifier } from '../batch-identifier.model';
    import { BatchIdentifierService } from '../batch-identifier.service';

    // We will add more imports later for Component Logic and for saving the batch part
    ...
    export class BatchCompleteComponent implements OnInit, OnDestroy {
      // We need a local property to keep track of the Batch Identifier. The Batch Identifier is a collection of keys and identifiers to access a specific Batch.
      batchIdentifier: BatchIdentifier;
      // We need a local property of the Batch Part's data that we get back from the Batch Service. We initialize it here to it is able to be bound to on the first iteration of the Angular life cycle. It is a BatchData type rather than a simple BatchPart type.
      batchComplete: BatchData<BatchComplete> = new BatchData<BatchComplete>(new BatchComplete());
      // We will add more *Component Logic* properties. This will include things related to the View Mode, lookups, and external buttons like in the Application Navbar Menu.
      ...

      // For the Batch Part Logic these are the only services that need to be setup. We will add more services / classes / subscriptions when adding the Component Logic.
      constructor(
        private batchService: BatchService,
        private batchIdentifierService: BatchIdentifierService,
        private route: ActivatedRoute,
        ...
      )

      ngOnInit() {
        // We compartmentalize the data loading logic to it's own function. There might be other non-data related loading logic that might need to be included here and don't want those things together.
        this.loadData();
      }
    }
    ```

    This is the initial boilerplate that we need before we start getting into the details of loading the new Batch Part's data.

    ```ts
    ...
    loadData(): void {
      this.setupBatchComplete(); // This is logic specific to loading the desired Batch Part.

      // Here we can add logic / function calls to load other lookup data that only needs to be accessed once. Metadata / lookup data that is dependent on the specific batch should happen after that batch has been loaded.

      /* Example: This set of metadata doesn't care what the batch part's details are so it can be loaded at this time.
      this.unitOfMeasureService.getCommonUnitsOfMeasure()
        .subscribe((unitsOfMeasure: UnitOfMeasure[]) => {
          this.unitsOfMeasure = unitsOfMeasure;
        });
      */
    }

    // This function is going to be the main function for loading the Batch Part's data.
    setupBatchComplete(): void {
      this.route.parent.params
        .switchMap((params: Params) => {
          return this.batchIdentifierService.getBatchID(params['id']);
        })
        .switchMap((batchIdentifier: BatchIdentifier[]) => {
          this.batchIdentifier = batchIdentifier[0];
          return this.batchService.getBatch(this.batchIdentifier, [BatchPart.Complete]);
        })
        .subscribe((batch: Batch) => {
          batch.Complete.obs
            .subscribe((batchComplete: BatchData<BatchComplete>) => {
              this.batchComplete = batchComplete;
              this.batchCompleteChange();
            });
        });
    }
    ```

    While `loadData(...)` is pretty straightforward, `setupBatchComplete(...)` has a lot going on inside of it so let's break it down.

    The first section:

    ```ts
    this.route.parent.params
      .switchMap((params: Params) => {
        return this.batchIdentifierService.getBatchID(params['id']);
      })
    ```

    What's happening is we are looking at the Route Parameters (these are different than Query Parameters). The Route Parameter of `id` is setup by the `batch-routing.module.ts` when we make the path to all the Batch and Batch Parts. The format of `batch\<id>\<batch tab>`. That route param of `id` is then added to the `BatchIdentifierService` function `getBatchID(...)` as the parameter. The `id` parameter should be a `Lot Number` to access the `Batch` by. Because we are using a `switchMap(...)` this function (and therefor the following chained functions) will be recalled anytime the `id` parameter changes.

    The second `.switchMap(...)`:

    ```ts
    .switchMap((batchIdentifier: BatchIdentifier[]) => {
      this.batchIdentifier = batchIdentifier[0];
      return this.batchService.getBatch(this.batchIdentifier, [BatchPart.Complete]);
    })
    ```

    This `switchMap(...)` is receiving a result from the previous `switchMap(...)` of the `BatchIdentifier` object for the given `id`. The returning `BatchIdentifier` object will have a key value on it of `BatchID` which is the comes from the `MFG.Production_Batch.Production_Batch_PK` field. This and possibly other fields in the `BatchIdentifier` will be used by the API/DB to access the `Batch Part`'s data.

    So with that Identifier now available we will call our `Batch Service`'s `getBatch(...)` function. This function is one of the ones we expanded earlier to work with the new `Batch Part`. When we call this function we pass the BatchIdentifier (for accessing the API correctly), and we pass an array of `Batch Part`s. We pass an array because there are times when a component will need to load more than one `Batch Part`. For our example we will leave it with a single `Batch Part`. By adding the other Batch Part loading logic to the `Batch Part's Service` we made it easier if this new `Batch Part` needs to be used elsewhere later on too.

    The `getBatch(...)` function will return the whole `Batch` object, and make sure it is setup to load the desired `Batch Part`s we passed in the array (although there might not be any data in our `Batch Part` just yet).

    Our final `.subscribe(...)` is where we really get the `Batch` object and related `Batch Parts` available to work with.

    ```ts
    .subscribe((batch: Batch) => {
      batch.Complete.obs
        .subscribe((batchComplete: BatchData<BatchComplete>) => {
          this.batchComplete = batchComplete;
          this.batchCompleteChange();
        });
    });
    ```

    In this example our `Batch Part` (`Batch Complete`) is the only batch part that we care about on the `Batch` object we only need to work with it in this area. We take the `Batch` object that is setup by the previous function, and since in an earlier step we altered the `setupObs(...)` function in the `batch.service.ts` section; we have an `obs` property that will represent the `Batch Part` Service's function `getBatchDataComplete(...)` function. So when we subscribe to that function now it will determine if we have a *stale* or *fresh*, *pristine* or *dirty* `Batch Part` and will take the appropriate action to return the data either what was existing in the Local Storage `Batch` object, or will call out to the API to get something fresher. We take what ever the result of the `obs` subscription and store it in our local property `this.batchComplete`. We now have an object available to be bind our template's data to.

    The final line `this.batchCompleteChange()` is a method we will call in the component to store our `Batch Part` again in Local Storage (we want to do this if we got new data from the API that needs to be remembered by the application).

    We can add that method now. Note if there are more is more than one `Batch Part` that is being worked with in the component then each time that `Batch Part` is changed it should notify it's own version of that function or allow for switching what `Batch Part` type is being updated.

    ```ts
    ...
    batchCompleteChange(): void {
      this.batchService.storeBatch(this.batchIdentifier, this.batchComplete, BatchPart.Complete);
    }
    ...
    ```

    So this simple function is calling a function that didn't need to be edited on the `Batch Service` that will take the enum of the the `Batch Part` (`BatchPart.Complete`) to know what part is being passed/updated and then the actual local property of `this.batchComplete` will get added to the `Batch` object and stored in Local Storage. As mentioned in Local Storage Batch Design section this should be chained into by any changes that occur in the `Batch Part`'s Component.

    At this point the `Batch Part`'s data should be available and we can now get into the implementation details for the component. Once we have some Component Logic we will return to the Batch Part Logic for saving.

#### Scaffolding out the Batch Part component specific logic

Now that `Batch Part`'s data is available to be consumed in the client we can get into the implementation details for our new component. Some of the first things we will do is build an example of how to interact with both set of the data contained in the `BatchData<BatchPart>` that we just created and made available to the component. We will also look at setting up the form to have different states (view/edit). Finally look at some details about keeping our Local Storage `Batch` object up to date with changes that occur in the component.

1. We will start out simple with allowing the component to recognize different view states or `ViewMode`s. The Application Menu (`core/application-menu`) should already have a set of buttons that are available when viewing a `batch` (url = `batch/.../batchPart), but we will have to wire up our component to listen to those events and handle them for our context.

    We will continue to expand the component file (`batch-complete.component.ts`)

    ```ts
    ...
    import { ViewMode } from '../../core/view-mode/view-mode.model'; // There is a View Mode enum already available to use.
    import { ApplicationMenuService } from '../../core/application-menu/application-menu.service';
    ...

    export class BatchCompleteComponent implements OnInit, OnDestroy {
      ...
      // This is a local property of our imported enum. This enum contains all possible states the page could be in.
      // We need to set a local property to a copy of it because our template can't directly access an imported enum.
      ViewMode = ViewMode;
      // We will also have a local property to keep track of the current state of the page, and will start the page off in `View` mode.
      mode: ViewMode = ViewMode.View;
      // We need a subscription to be able to listen to our Application Menu actions
      menuSubscription: Subscription
      ...

      constructor(
        ...
        applicationMenuService: ApplicationMenuService // We will add a property of the service to be used by the component.
      ) {
        // Right away we will have our constructor being to listen to the Application Menu
        this.menuSubscription = applicationMenuService.menuEvent$.subscribe(event => {
          // When the Application Menu emits an event we will process that event with this function.
          this.processMenuEvent(event);
        })
      }

      // We won't do anything new to this function.
      ngOnInit() {
        this.loadData();
      }

      // But like the ngOnInit() we now need to do an action during the destroy phase
      ngOnDestroy() {
        // Unsubscribing from the Application Menu prevents a memory leak and prevents the subscription from staying open and listening after it should be gone.
        this.menuSubscription.unsubscribe();
      }

      ...

      processMenuEvent(event): void {
        switch (event) {
          case 'EDIT':
            this.changeViewMode(ViewMode.Edit);
            break;
          case 'CANCEL':
            this.changeViewMode(ViewMode.View);
            break;
          case 'SAVE':
            this.save();
            break;
        }
      }

      changeViewMode(mode: ViewMode): void {
        this.mode = mode;
      }

      ...
    }
    ```

    At this point we have a functionality to listen to the Application Menu and react to the View Mode changes from buttons available.

2. With the ability to switch between page states we can add some form items to see how the data should be rendered on the page. A simple field in the example `Batch Part`: `Batch Complete` is `CompleteAmount` which is a simple numeric value. Since the value is available in our `this.batchComplete` local property we can start work in the template file: `batch-complete.component.html`.

    While a simple binding like this would suffice to display the value it doesn't really take advantage of the extra work that we've put in to setting up our `Batch Part`'s `Batch Data` typed object. A simple binding like this also doesn't take advantage of our `ViewMode` property to display an editable field.

    ```html
    <span>{{batchComplete.dirtyData.CompleteAmount}}</span>
    ```

    To fully take advantage of our two copies of the data and the view state we should have the form control react in different ways and give an option to edit the value:

    ```html
    <div [ngClass]="{'form-group': true, 'has-warning': batchComplete.dirtyData.CompleteAmount !== batchComplete.originalData.CompleteAmount && viewMode !== ViewMode.New}">
      <label class="control-label" for="completeAmountInput">Complete Amount
        <i *ngIf="batchComplete.dirtyData.CompleteAmount !== batchComplete.originalData.CompleteAmount && viewMode !== ViewMode.New" class="glyphicon glyphicon-info-sign" tooltip="Original Value: {{batchComplete.originalData.CompleteAmount}}"></i>
      </label>
      <input *ngIf="viewMode === ViewMode.Edit || viewMode === ViewMode.New" type="number" id="completeAmountInput" class="form-control" [(ngModel)]="batchComplete.dirtyData.CompleteAmount" (ngModelChange)="batchCompleteChange()" />
      <p *ngIf="viewMode === ViewMode.View" class="form-control-static">{{batchComplete.dirtyData.CompleteAmount}}</p>
    </div>
    ```

    This will render to something like this:

    ![alt text](BatchComplete_CompleteAmount.gif "Logo Title Text 1")

    Fair amount of stuff going on in that block, but what it gives us is a couple of nice things:

    1. Paragraph or an Input element based on our View Mode. Plain text paragraph elements are easier to read than something like a disabled input during a View Mode.
    2. The Input element is a `type="number"` which will give the browser a spinner option and keep the data value the correct number type.
    3. The element has consistent Bootstrap styling with the rest of the application's forms.
    4. The previous known value for the field is also known.
        1. This gives us an option to show a styling of `has-warning` for the control if the value has changed. This gives the user a quick visual indication what parts they've touched.
        2. We can display the previous value incase they pause while working and need to know the what it was before they started modifying it.
    5. With the `(ngModelChange)=...` we are able to notify the batch service with the function set up in the previous section for storing an updated `Batch Part`.

    We can break down each section of the template

    1. The wrapping `<div>` is styled to get a class of `has-warning` if the copies of data are different (*dirty*), and if view mode is not *New* (View Mode of *New* would only have ever have a value in the `dirtyData` copy.)
    2. The next block `<label>` has a `for` property to give a click on the label focus to the Input element, and then the text value of `Complete Amount`.
        1. Inside that Label we have an optional Icon with a tooltip. The icon shows with the same rule as the `has-warning` class for the form control, and the tooltip is being bound to the `originalData` copy of data. It's ok to use this field in the template such that the user never has an option to edit the underlying value.
    3. Next we have an `<input>` tag that only shows during the view modes for *Edit* or *New*. It binds to the `dirtyData` copy of the field and has a change event trigger to store updated values to our Local Storage object.
    4. The final element is the plain text paragraph display of the `dirtyData` this gives a easier way to read the values of the form when the page is in *View* mode.
    - Side Note: We can see we are using the `enum`: `ViewMode` in the template here. This is why we needed to import it and make a local property copy of the `enum`.

3. In order to get these nice to have those features we have the tradeoff of a relatively complex template markup. Since it is likely that our forms will have several input controls form number values it'd be nice to be able to write this markup quickly and consistently. We can make a mini-component just for this one bunch of form control elements and house any related logic in that component. Making a generic version gives us a way to easily update all of them in the future if we have a necessary fix or feature improvement.

    In the `batch/batch-form` directory we could add a new component `batch-form-input-number` directory and component. This input template style is unique to the batch module where we have this dual data copies (`BatchData`). Other simpler forms like in the `metadata` module don't require this added functionality. If they need components for simpler form controls those would be separate and likely live in the `core` module.

    For our `batch-form-input-number.component.html` template we can make it generic by doing something like this:

    ```html
    <div [ngClass]="{'form-group': true, 'has-warning': value !== originalValue && viewMode !== ViewMode.New}">
      <label class="control-label" [for]="id">{{label}}
        <i *ngIf="value !== originalValue && viewMode !== ViewMode.New" class="glyphicon glyphicon-info-sign" tooltip="Original Value: {{originalValue | commaNumbers}}"></i>
      </label>
      <input *ngIf="viewMode === ViewMode.Edit || viewMode === ViewMode.New" type="number" [id]="id" class="form-control" [(ngModel)]="value" />
      <p *ngIf="viewMode === ViewMode.View" class="form-control-static">{{value | commaNumbers}}</p>
    </div>
    ```

    Data pieces that change are turned into property fields; things like the label's value, the value for the originalData, or the id of the element. To get these fields populated we need to accept them as `Input` parameters to our component. To add these we will edit out `batch-form-input-number.component.ts` file.

    ```ts
    import { Component, forwardRef, Input } from '@angular/core';
    import { NG_VALUE_ACCESSOR } from '@angular/forms';
    import { BatchFormWrapper, BatchFormValueAccessor } from '../batch-form-wrapper';
    import { ViewMode } from '../../../core/view-mode/view-mode.model';

    export const BATCH_FORM_VALUE_ACCESSOR: BatchFormValueAccessor = {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => BatchFormInputNumberComponent),
      multi: true
    };

    @Component({
      selector: 'app-batch-form-input-number',
      templateUrl: './batch-form-input-number.component.html',
      styleUrls: [
        './batch-form-input-number.component.css',
        '../batch-form.css'
      ],
      providers: [BATCH_FORM_VALUE_ACCESSOR]
    })
    export class BatchFormInputNumberComponent extends BatchFormWrapper<number> {
      @Input() id: string;
      @Input() label: string;
      @Input() originalValue: number;
      @Input() viewMode: ViewMode;
    }
    ```

    So in this component file we've got some special things going on.

    `@Input()` parameters. The input parameters give a way to make the form component generic.

    1. The `id` parameter is a simple string of the of what the `input` element's `id` attribute should be. Letting the user specify this allow the focus to work correctly when clicking on the label text.
    2. The `label` parameter is a simple string of what to stuff into the `label` element to let the user know what data the `input` is for.
    3. The `originalValue` parameter gives the component knowledge of the `batchComplete.originalValue.XXX` field.
    4. The `viewMode` parameter gives a way to know what state the page is in (View, Edit, New).

    You might notice that this component doesn't have anything defined for the `ViewMode` enum, and doesn't have a property for the `value` parameter that we are accessing in the template file. These have been already been refactored in this component to a wrapper class: `batch-form/batch-form-wrapper.ts`. The `value` parameter here is related to the `NG_VALUE_ACCESSOR` property that we are importing from `angular/forms`.

    With this wrapper class we are saying that anything that extends it (like our new form component does with: `export class BatchFormInputNumberComponent extends BatchFormWrapper<number>` line) will also gain the properties and logic contained in it.

    Looking at that file we're extending we see this:

    ```ts
    import { ControlValueAccessor } from '@angular/forms';
    import { InjectionToken, Type } from '@angular/core';
    import { ViewMode } from '../../core/view-mode/view-mode.model';

    const noop = () => { };

    /**
    * This class gives easy access to the ControlValueAccessor which allows components/directives that accept two way data binding with ngModel.
    */
    export abstract class BatchFormWrapper<T> implements ControlValueAccessor {
      innerValue: T;
      onTouchedCallback: () => void = noop;
      onChangeCallback: (_: T) => void = noop;
      ViewMode = ViewMode;

      get value(): T {
        return this.innerValue;
      }

      set value(value: T) {
        if (value !== this.innerValue) {
          this.innerValue = value;
          this.onChangeCallback(value);
        }
      }

      // Set touched on blur
      onBlur() {
        this.onTouchedCallback();
      }

      // Form ControlValueAccessor interface
      writeValue(value: T) {
        if (value !== this.innerValue) {
          this.innerValue = value;
        }
      }

      // Form ControlValueAccessor interface
      registerOnChange(fn: any) {
        this.onChangeCallback = fn;
      }

      // Form ControlValueAccessor interface
      registerOnTouched(fn: any) {
        this.onTouchedCallback = fn;
      }
    }

    export class BatchFormValueAccessor {
      provide: InjectionToken<ControlValueAccessor>;
      useExisting: Type<any>;
      multi: boolean;
    }
    ```

    Basically all this is doing is giving us a way to extend the normal `[(ngModel)]` syntax of interacting with the component. It also gives us a place to store the `ViewMode` enum once rather than having to do it in each `batch form` component. The NG_VALUE_ACCESSOR can be trickier to work with in some contexts so the other option is to use an `@Input()` for the value and during the `(ngModelChange)` in the form component have that hit an event emitter as an `@Output` parameter and handle it at the parent component level.

    So with this new `batch-form` component using the existing `wrapper` class we have a component that abstracts some of the details of working with that type of form control with our `Batch Data` / `Batch Part` data. We can return to our component template and implement a usage of this mini-component as a replacement to our earlier markup.

    (`batch-complete.component.html`)

    ```html
    <app-batch-form-input-number [(ngModel)]="batchComplete.dirtyData.CompleteAmount" [id]="'completeAmountInput'" [label]="'Complete Amount'" [originalValue]="batchComplete.originalData.CompleteAmount" [viewMode]="mode" (ngModelChange)="batchCompleteChange()"></app-batch-form-input-number>
    ```

    Here we are interacting with `[(ngModel)]` in the same way as before thanks to the `wrapper` class and the `NG_Value_Accessor`. We also pass in the values we need to our `@Input()` parameters. A `@Input()` parameter like `[originalValue]` is passing a property from the component; A parameter like `[label]` is passing a string value defined here in the template file. We are also still able to chain into our Local Storage save function with our `(ngModelChange)` trigger. (If we the `batch-form` component doesn't use `NG_Value_Accessor` we will use a `@Output()` event emitter to record that change in the components properties and chain into Local Storage save function).

    Now anytime we want to do this style of form element we can just reuse the same mini-component from our `batch-form` directory. Other similar components should be stored in the same directory such that they're able to be generic and reused in other `batch` components. There will be times when a form control type is specific enough to a single entity that it doesn't need a generic `batch-form` component and it is fine to leave those as part of the main component.

    At this point we can continue to build out more elements and form fields to meet the requirements for the component in the same fashion as we've done in this past couple of steps. Once we have a some edit functionality we will want to be able to save the form to the API.

#### Scaffolding out the Batch Part save logic

As mentioned in previous sections we will now return to the our `Batch Part` service and the `Batch`'s service to implement save functionality. The `Batch` module has a global save function. Triggering the `Batch > Save` button from the Application Menu will trigger each `Batch Part`'s save function that has been loaded by the user and is dirty. This allows a user to make changes across multiple tabs, review them, and then save all their work as a single transaction with the server. To facilitate working with saving a `Batch Part`'s data even if we aren't on that particular tab we need to rely on our `Batch Service` (`batch/batch.service.ts`) and how it will interact with our `Batch Part`'s service.

1. To start off we will work with our new `Batch Part`'s (`batch-complete.service.ts`)

    It will already have the loading functions and a pristine checking function, but we need to add a function to talk to our API and send updates that way.

    ```ts
    export class BatchCompleteService {
    ...

    updateBatchComplete(batchComplete: BatchComplete): Observable<ResponseDetail> {
      return this.http
        .put(`${environment.webapiRoot}BatchCompletion`, batchComplete)
        .map((r: Response) => {
          return r.json() as ResponseDetail;
        });
    }

    ...
    }
    ```

    Our save (`put`) function will pass our `batchComplete` object to the API. Notice that at this point we aren't dealing with the `BatchData<T>`, but just a `Batch Part` that we've made; the API doesn't care about the `BatchData<T>` level of detail. We are also expecting our API to return a `ResponseDetail` this is a class that can tell us about any errors that happen on the server and let us know any key value that we might need. We need to be able to know when our save function has completed so we can take an appropriate action after it has completed and inform the user about how it went.

    Like the load function we won't call this save function directly from our component, but rather let the `Batch Service` facilitate it.

2. With our specific save available we can expand our `Batch Service` to support it. (`batch/batch.service.ts`)

    We won't need to add any import statements or local properties since we did that when we updated it for loading our `Batch Part`, but we can expand the `save(...)` function for our `Batch Part`.

    ```ts
    export class BatchService {
      ...

      save(batch: Batch): Observable<any> {
        let saveObservableArray = [];

        ...
        if(batch.Complete != null && !this.batchCompleteService.isPristine(batch.Complete)) {
          saveObservableArray.push(this.batchCompleteService.updateBatchComplete(batch.Complete.dirtyData));
        }
        ...

        return Observable.forkJoin(saveObservableArray);
      }

      ...
    }
    ```

    We are passed the whole `Batch` object to this function so that each `Batch Part` can be processed together.

    So by expanding this array we've informed the `Batch Service` save function how our `Batch Part` should be saved if it is dirty. What is happening is that we start the function with an empty array `saveObservableArray` and we look at each `Batch Part` to see if it is loaded and dirty. If so then we will add our save function (passing to the function only the `dirtyData` of our `Batch Part`) to the array. Finally the function is returning an `forkJoin Observable`.

    ForkJoins work by calling all the Observables in the Array and then waiting until all have been returned before continuing their after execution logic. We will want to wait for all the `Batch Part`s to have had a change to save before we decide what actions we want to take.

    This function will be called from our new `Batch Part`'s component file.

3. Now that the `Batch Service` save function is ready we can make it accessible from our `Batch Part`'s component. (`batch-complete.component.ts`)

    ```ts
    export class BatchCompleteComponent implements OnInit, OnDestroy {
      ...
        processMenuEvent(event): void {
          switch (event) {
            case 'EDIT':
              this.changeViewMode(ViewMode.Edit);
              break;
            case 'CANCEL':
              this.changeViewMode(ViewMode.View);
              break;
            case 'SAVE':
              // Before we ignored this available action from our Application Menu Events, but now we will implement a function for it.
              this.save();
              break;
          }
        }

        save(): void {
          this.batchService.getBatch(this.batchIdentifier, [BatchPart.Complete])
            .subscribe((batch: Batch) => {
              this.batchService.save(batch)
                .subscribe((batchPartResponseDetails: ResponseDetail[]) => {
                  let summaryResponseDetail = this.batchService.processResponseDetails(batchPartResponseDetails, this.batchIdentifier);
                  // Do something with summary.
                  if (summaryResponseDetail.Success) {
                    // Reload the batch data.
                    this.setupBatchComplete();
                    this.changeViewMode(ViewMode.View);
                  } else {
                    // Warn user about the issue
                    // TODO: BS Modal Popup?
                    alert(summaryResponseDetail.Message);
                  }
                });
            });
        }
    ...
    }
    ```

    So our new save function in the component will be triggered by the Application Menu's `Save` button and then we will store the logic in our `save()` function.

    1. The first part of the the function is asking the `Batch Service` (and Local Storage) for a copy of the `Batch` as it currently exists.

    ```ts
    this.batchService.getBatch(this.batchIdentifier, [BatchPart.Complete])
      .subscribe((batch: Batch) => {
        ...
      });
    ```

    2. Once that object is returned we can pass it to our save function.

    ```ts
    this.batchService.save(batch)
      .subscribe((batchPartResponseDetails: ResponseDetail[]) => {
        ...
      });
    ```

    Since we've set up the new `Batch Part`'s save function to return a `ResponseDetail` type and the other `Batch Part`s have done the same we will get back an Array of their responses from the `ForkJoin` observable. Hopefully all the saves have been successful.

    3. To check the `ResponseDetails` a function has already been setup on the `Batch Service`. It will aggregate their results (basically just a Yes/No if anything went wrong). A side note with the `processResponseDetails` is it will set the `Batch Parts` (`BatchData<T>`) to a state so that the next time they're loaded it is done from the server rather than the Local Storage.

        1. If the summary was successful, great, we can call our `setupBatchComplete()` function to reload the data from the server. This will make it so the user can see what it looks like on the server now. We will also pop the user back to `View` Mode.

        2. If the summary wasn't successful, we will want to warn the user that something did not go as expected so that they have a chance to either make changes to the `Batch` or notify a programmer about a bug.

At this point we should have been able to save our `Batch Part` that was loaded from the server or local storage, and edited by the user!  The rest of the work should just be implementation details.

---

# Eparts SDS Services

- Programmer contact at Autologue is usually Kyle Bloom (kbloom@autologue.com).
- Michelle Honse is the other IT stakeholder in the application and often mediates between programmers and Autologue / status of the project.

[Kiln Repository](https://vpidev.kilnhg.com/Code/Repositories/netservice/EpartsSDSSync)

## Eparts SDS Uploader

See Case 10072.

This is mostly working. There are a few SDS records that don't get all the data points they need to do a successful upload so it isn't actually attempted (just logged in SaM.Eparts_SDS_Sync).

There are three modes for this:
1. Full Upload: This will grab all SDSs that are current and attempt an upload.
2. New Upload: This will compare the current list of SDSs with the SDS upload log (SaM.Eparts_SDS_Sync) entries that are marked as completed. This way once a SDS is uploaded successfully we won't attempt it again. (This might need some work because during testing they cleared their side and we needed to truncate our logs for this upload to work again).
3. Single Upload: This will upload a single SDS based on the GUID given to the app as a parameter.

This likely could have it's query improved as right now it loads each part of it with different stored procedure calls, but getting some of the columns together can be tricky as they come from different servers / databases or would require some special query work with either pivots or xml->csv for the items field.

We had been working with Michelle Honse recently to get the uploader pointed to Production, so it may still need some tweaking. As of 1/4/2018 we were still testing.

Application is deployed to `\\TaskServer\c$\DVApps\EpartsSDSSync` and is run via the Windows Task Scheduler on that server.

## SDS Email Log (Download Distribution Log)

See Case 9748.

This is to pull down the log of emails sent by the Eparts site. The Audit works off of a optionally supplied date range. Without the date range it should default to emails within the last week.

## SDS Audit

See Case 9747.

This is to compare versions of SDSs on the Eparts site to the ones we think are current. If there are SDSs that are out of date we should upload the new version of it. This is currently a WIP in the solution, just commented out.

---

# ITInfo

Primary Contact would be Mike DJ since he is the main stakeholder in the app. He decides what features should be added, but other IT staff use it too and might have ideas. Ashton has been building some fixes/improvements/feature requests with ITInfo recently.

Latest requests that Ashton hasn't heard about would be the Service Request List update from Mike on 12/21: Case 10416.
Otherwise the open cases assigned to him or me are still open.

Main application case list:
10416 - New Request from Mike about Service Vendor List query.
10196 - I know Ashton was in the middle of this before he left for holiday.
10229 - I think Ashton finished the multi column sorting, not sure if he still has any local changes left.
10226 - Parent Case with the ITInfo batch 2 changes from Mike.

## Service Desk Export

Primary contact is currently Dean Walhof.

Related to ITInfo is the Service Desk Export. It is pulling from the same data even though it is actually a different [Kiln Repository](https://vpidev.kilnhg.com/Code/Repositories/sqlssis/ServiceDesk). The latest request from Dean Walholf on 12/20 was with Case 10410 to update how it exported some rows. This requested change would likely just require an altered stored procedure.

Made a change on 1/8/2018, but hasn't been tested by Dean yet.

---

# Misc

## CheckFoxproFileSyncStatus

Task Scheduler:
1. Actions > Program/script: `Powershell.exe`
2. Action > Add arguments: `-noexit -command "& 'c:\Scripts\PS\CheckFoxproFileSyncStatus.ps1' 1"`
3. Triggers > Begin the task: `On a schedule`
4. Triggers > Settings: `Daily` and Recur every: `1` days
5. Triggers > advanced settings > Enabled: `checked`.

## Visual Studio Code Extensions

1. Angular Language Service - Auto Imports, Autocomplete in Templates from Components, Go to definition - [Link](https://marketplace.visualstudio.com/items?itemName=Angular.ng-template)
2. TSLint - Underlines issues with Typescript code - [Link](https://marketplace.visualstudio.com/items?itemName=eg2.tslint)
3. Code Spell Checker - Better spellchecking - [Link](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker)
4. Beautify - Formats Code (`Alt` + `Shift` + `F`) - [Link](https://marketplace.visualstudio.com/items?itemName=HookyQR.beautify)

## Sproc Object Mapper

This is a file that I made to take a stored procedure and scaffold out some commonly used things with the results it gets.

This requires can only be run against an instance of SQL Server 2012 or newer.

All you need to do it plug in the stored procedure name that you're interested in getting the results from.

The script has got serious room for improvement, but just been adding to it for a long time.

Results:

Column|Usage
:--|:--
DB Model Class | This will be the initial scaffolding for the database model class in the .NET Business Project.
C# Model Class | This will be the initial scaffolding for the api model class in the .NET Business Project. (This might need to be expanded if the class should have sub-collection items or something determined at the server.)
Mapper Bindings | This is the initial Tiny Mapper binding between the DB Model Class and the C# Model Class bindings.
Angular Model class (wip) | This is a class that should match the C# Model class, but have it changed to use the types available to Typescript instead of C#.
CTOR Params | This is for an Angular model class if it has a constructor. This is the optional parameter list.
CTOR Body | This is for an Angular model class if it has parameters then these would be the default bindings.
isPristine | This is a default `isPristine(...)` comparison for models in the `Batch` module.

Not guaranteed to be perfect, but just a starting point.

```sql
USE ENTERPRISE
GO

DECLARE @SprocName NVARCHAR(128) = 'MFG.Batch_Type_List'

SELECT 'public '+( CASE
				   --TODO: Filestream and XML types not supported
					   WHEN system_type_name = 'bit'
					   THEN 'bool'
					   WHEN system_type_name = 'tinyint'
					   THEN 'byte'
					   WHEN system_type_name = 'binary'
							OR system_type_name = 'image'
							OR system_type_name = 'rowversion'
							OR system_type_name = 'timestamp'
							OR system_type_name LIKE 'varbinary%'
					   THEN 'byte[]'
					   WHEN system_type_name = 'date'
							OR system_type_name = 'datetime'
							OR system_type_name LIKE 'datetime2%'
							OR system_type_name = 'smalldatetime'
					   THEN 'DateTime'
					   WHEN system_type_name = 'datetimeoffset'
					   THEN 'DateTimeOffset'
					   WHEN system_type_name LIKE 'decimal%'
							OR system_type_name = 'money'
							OR system_type_name LIKE 'numeric%'
							OR system_type_name = 'smallmoney'
					   THEN 'decimal'
					   WHEN system_type_name LIKE 'float%'
					   THEN 'double'
					   WHEN system_type_name = 'uniqueidentifier'
					   THEN 'Guid'
					   WHEN system_type_name = 'smallint'
					   THEN 'Int16'
					   WHEN system_type_name = 'int'
					   THEN 'int'
					   WHEN system_type_name = 'bigint'
					   THEN 'Int64'
					   WHEN system_type_name = 'sql_variant'
					   THEN 'Object'
					   WHEN system_type_name = 'real'
					   THEN 'Single'
					   WHEN system_type_name LIKE 'char%'
							OR system_type_name LIKE 'nchar%'
							OR system_type_name = 'ntext'
							OR system_type_name LIKE 'nvarchar%'
							OR system_type_name = 'text'
							OR system_type_name LIKE 'varchar%'
					   THEN 'string'
					   WHEN system_type_name = 'time'
					   THEN 'TimeSpan'
					   WHEN system_type_name = 'xml'
					   THEN 'Xml'
				   END )+( CASE
							   WHEN is_nullable = 1
									AND ( system_type_name = 'bit'
										  OR system_type_name = 'tinyint'
										  OR system_type_name = 'date'
										  OR system_type_name = 'datetime'
										  OR system_type_name LIKE 'datetime2%'
										  OR system_type_name = 'smalldatetime'
										  OR system_type_name = 'datetimeoffset'
										  OR system_type_name LIKE 'decimal%'
										  OR system_type_name = 'money'
										  OR system_type_name LIKE 'numeric%'
										  OR system_type_name = 'smallmoney'
										  OR system_type_name LIKE 'float%'
										  OR system_type_name = 'uniqueidentifier'
										  OR system_type_name = 'smallint'
										  OR system_type_name = 'int'
										  OR system_type_name = 'bigint'
										  OR system_type_name = 'real'
										  OR system_type_name = 'time' )
							   THEN '?'
							   ELSE ''
						   END )+' '+name+' { get; set; }' AS 'DB Model Class',
	   'public '+( CASE
				   --TODO: Filestream and XML types not supported
					   WHEN system_type_name = 'bit'
					   THEN 'bool'
					   WHEN system_type_name = 'tinyint'
					   THEN 'byte'
					   WHEN system_type_name = 'binary'
							OR system_type_name = 'image'
							OR system_type_name = 'rowversion'
							OR system_type_name = 'timestamp'
							OR system_type_name LIKE 'varbinary%'
					   THEN 'byte[]'
					   WHEN system_type_name = 'date'
							OR system_type_name = 'datetime'
							OR system_type_name LIKE 'datetime2%'
							OR system_type_name = 'smalldatetime'
					   THEN 'DateTime'
					   WHEN system_type_name = 'datetimeoffset'
					   THEN 'DateTimeOffset'
					   WHEN system_type_name LIKE 'decimal%'
							OR system_type_name = 'money'
							OR system_type_name LIKE 'numeric%'
							OR system_type_name = 'smallmoney'
					   THEN 'decimal'
					   WHEN system_type_name LIKE 'float%'
					   THEN 'double'
					   WHEN system_type_name = 'uniqueidentifier'
					   THEN 'Guid'
					   WHEN system_type_name = 'smallint'
					   THEN 'Int16'
					   WHEN system_type_name = 'int'
					   THEN 'int'
					   WHEN system_type_name = 'bigint'
					   THEN 'Int64'
					   WHEN system_type_name = 'sql_variant'
					   THEN 'Object'
					   WHEN system_type_name = 'real'
					   THEN 'Single'
					   WHEN system_type_name LIKE 'char%'
							OR system_type_name LIKE 'nchar%'
							OR system_type_name = 'ntext'
							OR system_type_name LIKE 'nvarchar%'
							OR system_type_name = 'text'
							OR system_type_name LIKE 'varchar%'
					   THEN 'string'
					   WHEN system_type_name = 'time'
					   THEN 'TimeSpan'
					   WHEN system_type_name = 'xml'
					   THEN 'Xml'
				   END )+( CASE
							   WHEN is_nullable = 1
									AND ( system_type_name = 'bit'
										  OR system_type_name = 'tinyint'
										  OR system_type_name = 'date'
										  OR system_type_name = 'datetime'
										  OR system_type_name LIKE 'datetime2%'
										  OR system_type_name = 'smalldatetime'
										  OR system_type_name = 'datetimeoffset'
										  OR system_type_name LIKE 'decimal%'
										  OR system_type_name = 'money'
										  OR system_type_name LIKE 'numeric%'
										  OR system_type_name = 'smallmoney'
										  OR system_type_name LIKE 'float%'
										  OR system_type_name = 'uniqueidentifier'
										  OR system_type_name = 'smallint'
										  OR system_type_name = 'int'
										  OR system_type_name = 'bigint'
										  OR system_type_name = 'real'
										  OR system_type_name = 'time' )
							   THEN '?'
							   ELSE ''
						   END )+' '+REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '')+' { get; set; }' AS 'C# Model Class',
	   'config.Bind(source => source.'+name+', target => target.'+REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '')+');' AS 'Mapper Bindings',
	   REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '')+': '+( CASE
																									   --TODO: Filestream and XML types not supported
																										   WHEN system_type_name = 'bit'
																										   THEN 'boolean'
																										   WHEN system_type_name = 'tinyint'
																										   THEN 'any'
																										   WHEN system_type_name = 'binary'
																												OR system_type_name = 'image'
																												OR system_type_name = 'rowversion'
																												OR system_type_name = 'timestamp'
																												OR system_type_name LIKE 'varbinary%'
																										   THEN 'any'
																										   WHEN system_type_name = 'date'
																												OR system_type_name = 'datetime'
																												OR system_type_name LIKE 'datetime2%'
																												OR system_type_name = 'smalldatetime'
																										   THEN 'Date'
																										   WHEN system_type_name = 'datetimeoffset'
																										   THEN 'any'
																										   WHEN system_type_name LIKE 'decimal%'
																												OR system_type_name = 'money'
																												OR system_type_name LIKE 'numeric%'
																												OR system_type_name = 'smallmoney'
																										   THEN 'number'
																										   WHEN system_type_name LIKE 'float%'
																										   THEN 'number'
																										   WHEN system_type_name = 'uniqueidentifier'
																										   THEN 'any'
																										   WHEN system_type_name = 'smallint'
																										   THEN 'number'
																										   WHEN system_type_name = 'int'
																										   THEN 'number'
																										   WHEN system_type_name = 'bigint'
																										   THEN 'number'
																										   WHEN system_type_name = 'sql_variant'
																										   THEN 'any'
																										   WHEN system_type_name = 'real'
																										   THEN 'any'
																										   WHEN system_type_name LIKE 'char%'
																												OR system_type_name LIKE 'nchar%'
																												OR system_type_name = 'ntext'
																												OR system_type_name LIKE 'nvarchar%'
																												OR system_type_name = 'text'
																												OR system_type_name LIKE 'varchar%'
																										   THEN 'string'
																										   WHEN system_type_name = 'time'
																										   THEN 'any'
																										   WHEN system_type_name = 'xml'
																										   THEN 'any'
																									   END )+';' AS 'Angular Model class (wip)',
																									   REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '')+'?: '+( CASE
																									   --TODO: Filestream and XML types not supported
																										   WHEN system_type_name = 'bit'
																										   THEN 'boolean'
																										   WHEN system_type_name = 'tinyint'
																										   THEN 'any'
																										   WHEN system_type_name = 'binary'
																												OR system_type_name = 'image'
																												OR system_type_name = 'rowversion'
																												OR system_type_name = 'timestamp'
																												OR system_type_name LIKE 'varbinary%'
																										   THEN 'any'
																										   WHEN system_type_name = 'date'
																												OR system_type_name = 'datetime'
																												OR system_type_name LIKE 'datetime2%'
																												OR system_type_name = 'smalldatetime'
																										   THEN 'Date'
																										   WHEN system_type_name = 'datetimeoffset'
																										   THEN 'any'
																										   WHEN system_type_name LIKE 'decimal%'
																												OR system_type_name = 'money'
																												OR system_type_name LIKE 'numeric%'
																												OR system_type_name = 'smallmoney'
																										   THEN 'number'
																										   WHEN system_type_name LIKE 'float%'
																										   THEN 'number'
																										   WHEN system_type_name = 'uniqueidentifier'
																										   THEN 'any'
																										   WHEN system_type_name = 'smallint'
																										   THEN 'number'
																										   WHEN system_type_name = 'int'
																										   THEN 'number'
																										   WHEN system_type_name = 'bigint'
																										   THEN 'number'
																										   WHEN system_type_name = 'sql_variant'
																										   THEN 'any'
																										   WHEN system_type_name = 'real'
																										   THEN 'any'
																										   WHEN system_type_name LIKE 'char%'
																												OR system_type_name LIKE 'nchar%'
																												OR system_type_name = 'ntext'
																												OR system_type_name LIKE 'nvarchar%'
																												OR system_type_name = 'text'
																												OR system_type_name LIKE 'varchar%'
																										   THEN 'string'
																										   WHEN system_type_name = 'time'
																										   THEN 'any'
																										   WHEN system_type_name = 'xml'
																										   THEN 'any'
																									   END )+', ' AS 'CTOR Params',
																									   'this.'+REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '')+' = ' + REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '') + ';' AS 'CTOR Body',
																									   CASE WHEN system_type_name in ('date', 'datetime', 'smalldatetime') or system_type_name like 'datetime2%' then
																									   '!isEqual(XXX.originalData.'+REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '')+', XXX.dirtyData.' + REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '') + ') ||' else 
																									   'XXX.originalData.'+REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '')+' !== XXX.dirtyData.' + REPLACE(REPLACE(REPLACE(REPLACE(name, '_FK', 'ID'), '_PK', 'ID'), '_XK', 'ID'), '_', '') + ' ||' end  AS 'isPristine'

FROM sys.dm_exec_describe_first_result_set_for_object
( OBJECT_ID(@SprocName), NULL );
```