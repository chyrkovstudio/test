---
permalink: /pluggable-triggers/
layout: page
---
# Pluggable Triggers

## Overview

The Pluggable Trigger framework (Foundations Winter ‘23 onwards) enables multiple triggers to be added to a single SObject with a defined execution order. Salesforce supports multiple triggers to be added to a single SObject natively but the order is non-deterministic and the Salesforce documentation recommends having a single trigger per SObject. This has been overcome by adding a single trigger to a SObject which invokes one or more trigger plugins.

The fferpcore.PluggableTrigger class is invoked by the single apex trigger and searches for corresponding fferpcore__Plugin__mdt records. The plugin record defines: a trigger plugin to instantiate, the SObject it should be applied to, and an order number.

The information and code snippets provided below apply to the most recent Foundations package.

## Adding a Pluggable Trigger

1. Create the trigger implementation.

The fferpcore.PluggableTriggerApi.Plugin interface is used to implement a trigger plugin. This has a method for each value of Salesforce’s System.TriggerOperation enum, which will be invoked accordingly. You can also add a constructor to allow the value of Trigger.new or Trigger.old to be passed into your trigger.  See the example implementation below, the type of the records list can be changed from ‘SObject’ to the actual type of your SObject.

````
public class TriggerName implements fferpcore.PluggableTriggerApi.Plugin {
   // Value of Trigger.new (or Trigger.Old for delete operations)
   private List<SObject> records = new List<SObject>();


   public TriggerName(List<SObject> records){
       this.records = records;
   }
  
   public void onBeforeInsert() {
      
   }


   public void onBeforeUpdate(Map<Id, SObject> existingRecords) {
      
   }


   public void onBeforeDelete() {
      
   }


   public void onAfterInsert() {
      
   }


   public void onAfterUpdate(Map<Id, SObject> existingRecords) {
      
   }


   public void onAfterDelete() {
      
   }


   public void onAfterUndelete() {
      
   }
}
````

2. Create the trigger constructor.

The trigger constructor is used to create instances of your trigger plugin from step one. See the API documentation for fferpcore.PluggableTriggerApi.PluginConstructor for more details. An example implementation is provided below - the reference to fferpcore__BillingDocument__c should be changed to your target SObject. Note that this class and the methods within it must be declared as global so that the Foundations package can invoke them.

````
global class TriggerNameConstructor
   implements fferpcore.PluggableTriggerApi.PluginConstructor {


   global SObjectType sObjectType() {
       // The SObjectType for the SObject you want to add the trigger to,
       // e.g. Billing Document.
       return fferpcore__BillingDocument__c.SObjectType;
   }


   global fferpcore.PluggableTriggerApi.Plugin construct(
       List<SObject> records,
       fferpcore.PluggableTriggerApi.Context context // Currently unused
   ) {
       return new TriggerName(records);
   }
}
````

3. Add an Apex Trigger for the target SObject (can be skipped if the SObject already uses the pluggable trigger framework).

This trigger should invoke fferpcore.PluggableTrigger.runTriggerHandler(), which will find all the corresponding trigger plugins and execute them in order. You do not need to add an Apex Trigger if one already exists on this SObject, either in a Certinia managed package or unmanaged on your org. See the list of existing Certinia pluggable triggers. 

````
trigger BillingDocumentTrigger on fferpcore__BillingDocument__c(
   before insert,
   before update,
   before delete,
   after insert,
   after update,
   after delete,
   after undelete
) {
   try {
       fferpcore.PluggableTrigger.runTriggerHandler();
   } catch (Exception e) {
       for (SObject record : Trigger.new != null ? Trigger.new : Trigger.old) {
           record.addError(e);
       }
   }
}
````

4. Create a fferpcore__Plugin__mdt record.

A custom metadata record (on the FDN Plugin type) effectively registers your trigger with the Pluggable Trigger Framework. The required fields should be populated according to the table below.

| Field           | Value                                                                                                                                  |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Label           | Any suitable label for your trigger.                                                                                                   |
| FDN Plugin Name | Any suitable name for your trigger.                                                                                                    |
| Class Name      | The API name of the trigger constructor class (e.g.TriggerNameConstructor from step 2).                                                |
| Plugin Number   | Any integer value (e.g., 100). See the Ordering section for more guidance.                                                             |
| Extension Point | fferpcore.PluggableTriggerApi.PluginConstructor                                                                                        |
| Additional Data |The API name of the SObject you are adding the trigger to (e.g. fferpcore__BillingDocument__c).                                         |
| Disabled        |If selected, the plugin is disabled in Foundations.                                                                                     |
| Deprecated      |If selected, the plugin is deprecated. (This field is a Developer controlled field that allows it to be upgradable in package updates.) |

## Ordering

The Plugin Number field (fferpcore__PluginNumber__c) on the FDN Plugin custom metadata type (fferpcore__Plugin__mdt) controls the order that the triggers are executed. Any integer value can be used (including negative values); they can also be non-continuous (e.g., 1, 5, 15). If the same value is used multiple times then the triggers with that value will be executed in any order (e.g., for 1, 2, 2, 3 the two triggers with ‘2’ will have a non-deterministic order but they execute after ‘1’ and before ‘3’).

## Existing Pluggable Triggers

Multiple Certinia packages use the Pluggable Trigger Framework. If you are adding a trigger to a Certinia SObject or a standard object, we recommend that you choose a plugin number which is 100 or higher; this avoids conflicts or invalid data. New Certinia releases may add pluggable triggers to other SObjects, which may still result in conflict. 

We recommend against using pluggable and non-pluggable triggers on the same SObject where possible. Non-Certinia managed packages may add triggers to standard objects; in this case we can not determine whether the pluggable or non-pluggable triggers will execute first. As of Winter ‘24 the existing Certinia pluggable triggers are listed in the table below.

| SObject                               | Plugin Number | Pluggable Trigger                                      |
| ------------------------------------- | ------------- | ------------------------------------------------------ |
| fferpcore__BillingDocument__c         | -10           | ffbc.BillingDocumentsConstructor                       |
|                                       | 0             | fferpcore.BillingDocuments.Constructor                 |
|                                       | 10            | c2g.BillingDocumentsConstructor                        |
| fferpcore__BillingDocumentLineItem__c | -10           | ffbc.BillingDocumentLineItemsConstructor               |
|                                       | 0             | fferpcore.BillingDocumentLineItems.Constructor         |
|                                       | 20            | c2g.BillingDocumentLinesConstructor                    |
| pse__Proj__c                          | 1             | fpsai.ProjectsConstructor                              |
| ffscpq__Estimate__c                   | 1             | ffscpqint.EstimateTriggerPluginConstructor             |
| pse__Region__c                        | 2             | ffpsai.RegionsConstructor                              |
| pse__Practice__c                      | 2             | ffpsai.PracticesConstructor                            |
| pse__Grp__c                           | 2             | ffpsai.GroupsConstructor                               |