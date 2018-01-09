So for the `Process Step` we want to want to grab the Batch's Processes. Brian has been working on developing that for the Setup tab since it will be used there so we shouldn't have to mess around with any of the Service stuff since he's already implemented it.  What we need to do to gain access to the "Batch Process" "Batch Part" is make sure we load it like we do the "Batch Complete" data.

So in our Component file we are going to inform the `Batch Service` that we need the `Processes` Batch Part.

Right now when we make that call we are only making sure the `Complete` Batch Part is available.

```ts
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

When we make the `this.batchService.getBatch(...) call we will expand it to be:

```ts
...
return this.batchService.getBatch(this.batchIdentifier, [BatchPart.Complete, BatchPart.Processes]);
...
```

Now we can assume that we will be able to work with the `Processes` property of the `Batch` object correctly. The `Processes` property is a little bit different since it is an array and our `Complete` is an object, but shouldn't matter.

To actually work with the object we will expand what we do after the `Batch` object is returned so in the `.subscribe(...)` section we will work with the `Processes` property too.

```ts
...
.subscribe((batch: Batch) => {
  batch.Complete.obs
    .subscribe((batchComplete: BatchData<BatchComplete>) => {
      this.batchComplete = batchComplete;
      this.batchCompleteChange();
    });

  batch.Processes.obs
    .subscribe((batchProcesses: BatchData<BatchProcesses[]>) => {
      this.batchProcesses = batchProcesses;
      this.batchProcessesChange();
    });
});
```

(This might require a forkJoin or be better with one, but I think we'll be ok without it so for now we'll do it this way.)

So we will need to also add a property of `batchProcesses: BatchData<BatchProcess[]> = new BatchData<BatchProcess[]>(new Array<BatchProcess>());` to our property list in the component.  We also need the `this.batchProcessesChange()` method, which is simply:

```ts
...
batchCompleteChange(): void {
  ...
}

batchProcessesChange(): void {
  this.batchService.storeBatch(this.batchIdentifier, this.batchProcesses, BatchPart.Processes);
}
...
```

Should be pretty much exactly the same as the Batch Complete change method.

So at this point we should be loading the processes in the same way that Batch Setup is doing, and can now use it in our component.

We want a list of Processes available in our Staff Hours panel so we can update that section of our template.

```html
...
<td>
  <app-batch-table-select [(ngModel)]="staffTimeDetail.StaffRoleID" [viewMode]="mode" [originalValue]="getOriginal_BatchCompleteStaffTimeDetail(staffTimeDetail.BatchStaffID, 'StaffRoleID')" [originalDisplayValue]="getOriginal_BatchCompleteStaffTimeDetail(staffTimeDetail.BatchStaffID, 'StaffRoleName')" [dataSource]="staffRoles" [dataSourceValueField]="'ProductionStaffRoleId'" [dataSourceDisplayField]="'Name'" [dirtyDisplayValue]="staffTimeDetail.StaffRoleName" (ngModelChange)="staffTimeDetailsStaffRoleChange(staffTimeDetail)"></app-batch-table-select>
</td>
<td>
  <div [ngClass]="{'form-group': true, 'has-warning': isChanged_StaffTimeDetail(staffTimeDetail.BatchStaffID, 'ProcessStep')}">
    <input *ngIf="mode === ViewMode.Edit" type="text" class="form-control" [(ngModel)]="staffTimeDetail.ProcessStep" (ngModelChange)="batchCompleteChange()">
    <p *ngIf="mode === ViewMode.View" class="form-control-static" (ngModelChange)="batchCompleteChange()">{{staffTimeDetail.ProcessStep}}</p>
  </div>
</td>
<td>
  <app-batch-table-typeahead-staff-lookup [viewMode]="mode" [value]="staffTimeDetail.PersonID" [displayValue]="staffTimeDetail.StaffName" [originalValue]="getOriginal_BatchCompleteStaffTimeDetail(staffTimeDetail.BatchStaffID, 'PersonID')" [originalDisplayValue]="getOriginal_BatchCompleteStaffTimeDetail(staffTimeDetail.BatchStaffID, 'StaffName')" [dropup]="false" [placement]="'left'" (onChange)="batchStaffTimeDetailChange($event, staffTimeDetail)"></app-batch-table-typeahead-staff-lookup>
</td>
...
```

We will change this section to implement another `<app-batch-table-select>` component using the Processes.

```html
...
  <app-batch-table-select [(ngModel)]="staffTimeDetail.ProcessStep" [viewMode]="mode" [originalValue]="getOriginal_BatchCompleteStaffTimeDetail(staffTimeDetail.BatchStaffID, 'BatchProcessID')" [originalDisplayValue]="getOriginal_BatchCompleteStaffTimeDetail(staffTimeDetail.BatchStaffID, 'BatchProcess')" [dataSource]="batchProcesses.dirtyData" [dataSourceValueField]="'BatchProcessID'" [dataSourceDisplayField]="'ProcessName'" [dirtyDisplayValue]="staffTimeDetail.BatchProcess" (ngModelChange)="staffTimeDetailsBatchProcessChange(staffTimeDetail)"></app-batch-table-select>
...
```

So this should make a drop down that will allow us to chose the Batch Process for the given Staff Hours.

Like Roles we will need to implement a custom `(ngModelChange)` function in a similar way.

We also do not yet have the BatchProcess or BatchProcessID as fields on the Staff Hours row because they are relatively new fields so we will need to update the stored procedure to return those columns, then update the db, api, and angular models to support it and the db->api mapper call.

We will then need to make sure our update stored procedure is able to support the new field. We need to be careful when altering our update sprocs because it might have been initially for a legacy application and we won't want to break that with a change. We can do one of two things make the new parameter that we will pass an optional parameter (initalize it to null in the sproc definition), or make a new stored procedure for our new application. I'd want to do the first option, but worry if a Batch is edited in PTB and then in the legacy application that it might wipe out that column due to the null parameter.