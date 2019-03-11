# ember-data-change-tracker

[![Build Status](https://secure.travis-ci.org/danielspaniel/ember-data-change-tracker.png?branch=master)](http://travis-ci.org/danielspaniel/ember-data-change-tracker) [![Ember Observer Score](http://emberobserver.com/badges/ember-data-change-tracker.svg)](http://emberobserver.com/addons/ember-data-change-tracker) [![npm version](https://badge.fury.io/js/ember-data-change-tracker.svg)](http://badge.fury.io/js/ember-data-change-tracker)

**New**  
  - Experimental feature
    - isDirty, hasDirtyRelations computed properties  
      - Set up in [configuration](https://github.com/danielspaniel/ember-data-change-tracker#configuration) as { enableIsDirty: true }    
      - It is experimental and a has one crippling defect, it can not track object type
        attributes. But if you don't have object types it works fine.  

This addon aims to fill in the gaps in the change tracking / rollback that ember data does now.
 
 - Currently ember-data 
    - tracks changes for numbers/strings/date/boolean attributes
    - has a ```changedAttributes()``` method to see what changed => [ last, current ]
    - has a ```rollbackAttributes()``` method to rollback attributes
    - has a ```hasDirtyAttributes``` computed property 
  
 - This addon:
    - tracks modifications in attributes that are object/json/custom type
    - tracks replacement of belongsTo associations
    - tracks replacement/changes in hasMany associations
    - adds a ```modelChanges()``` method to DS.Model
    - adds a ```rollback()``` method to DS.Model
    - adds a ```isDirty``` computed property to DS.Model ( only if enabled in configuration )
    - adds a ```hasDirtyRelations``` computed property to DS.Model ( only if enabled in configuration )
    - Only works with 
      - ember-data versions 2.7+ ( if you have polymphic relationships )
      - ember-data versions 2.5+ ( if you don't )
    - Can be used in two modes 
      - auto track mode
      - manual track mode ( the default )
       
## Installation

* `ember install ember-data-change-tracker`

## Why?

  Say there is a user model like this:

```javascript
  export default Model.extend({
       name: attr('string'),  // ember-data tracks this already
       info: attr('object'),  // ember-data does not track modifications
       json: attr(),          // ember-data does not track modifications if this is object
       company: belongsTo('company', { async: false, polymorphic: true }),  // ember-data does not track replacement
       profile: belongsTo('profile', { async: true }), // ember-data does not track replacement
       projects: hasMany('project', { async: false }), // ember-data does not track additions/deletions
       pets: hasMany('pet', { async: true, polymorphic: true }) // ember-data does not track additions/deletions
   });
```

  You can not currently rollback the info, json if they are modified 
    or company, profile, projects and pets if they change.
  
  
### model changes
  
  - The method ```modelChanges()``` is added to model
  - Shows you any changes in an object attribute type
    - whether modified or replacing the value
    - attr() will default to 'object' type
    - works with any custom type you have created
  - Shows when you replace a belongsTo association
  - Shows when you add to a hasMany association
  - Shows when you delete from a hasMany association
  - Merges ember-data `changeAttribute()` information into one unified change object
  - Unlike ember-data no last and current value is shown, just the boolean => true
    - Though you will see [last value, current value] for the attributes that ember-data tracks 

Example: ( remove from a hasMany )
```javascript
  user.get('projects').removeObject(firstProject); // remove project1
  user.modelChanges() //=> {projects: true }
```


### Rollback

  - The method ```rollback()``` is added to model
  - If you're not using auto track you have to call ```startTrack()``` before editing 
  - Performace wise, it's way faster than you think it should be. 
    - Tested on model with hundreds of items in a hasMany association.
    - Though you might want to think twice when tracking one with thousands 
    
Usage: 
  
- make and makeList are from [ember-data-factory-guy](https://github.com/danielspaniel/ember-data-factory-guy). 
  - they create and push models ( based on factories ) into the ember-data store
 
```javascript 
    let info = {foo: 1};
    let projects = makeList('project', 2);
    let [project1] = projects;
    let pets = makeList('cat', 4);
    let [cat, cat2] = pets;
    let bigCompany = make('big-company');
    let smallCompany = make('small-company');

    let user = make('user', { profile: profile1, company: bigCompany, pets, projects });

    // manual tracking model means you have to explicitly call => startTrack
    // to save the current state of things before you edit
    user.startTrack();   

    // edit things  
    user.setProperties({
      'info.foo': 3,
      company: smallCompany,
      profile: profile2,
      projects: [project1],
      pets: [cat1, cat2]
    });

    user.rollback();

    // it's all back to the way it was
    user.get('info') //=> {foo: 1}
    user.get('profile') //=> profile1
    user.get('company') //=> bigCompany
    user.get('projects') //=> first 2 projects
    user.get('pets') //=> back to the same 4 pets

```

### isDirty, hasDirtyRelations
 - Computed properties to check if the model has changed
 - Not enabled by default 
  - Need to set enableIsDirty ( true ) on model or global [configuration](https://github.com/danielspaniel/ember-data-change-tracker#configuration)
 - The only attributes that can NOT be tracked with isDirty are object/array
   attributes
 
Usage:
  
```javascript 

    let info = {foo: 1};
    let pets = makeList('cat', 4);
    let [cat, cat2] = pets;
    let bigCompany = make('big-company');
    let smallCompany = make('small-company');

    let user = make('user', { company: bigCompany, pets });

    user.startTrack();   

    // edit things  
    user.set('name', "new name");
    user.get('isDirty'); //=> true
    
    user.rollback();
    user.get('isDirty'); //=> false
    
    user.set('company', smallCompany);
    user.get('hasDirtyRelations'); //=> true
    user.get('isDirty'); //=> true

    user.rollback();
    user.get('isDirty'); //=> false

    user.set('pets', [cat, cat2]);
    user.get('hasDirtyRelations'); //=> true
    user.get('isDirty'); //=> true
    
    user.rollback();
    user.get('isDirty'); //=> false

    // things that don't work      
    user.set('info.foo', 3); 
    user.get('isDirty'); //=> false ( object/array attributes don't work for computed isDirty )

```

### Configuration
  
  - Global configuration 
    - By default the global settings are: 
      - { **trackHasMany**: *true*, **auto**: *false*, **enableIsDirty**: *false* }
        - Essentially this says, track everything in the model but only when I tell you
        - Since this is manual mode you probably want to track everything 
          since you are focused on one edit at a time, hence trackHasMany is on
    - The options available are: 
      - **trackHasMany** : should hasMany associations be tracked? ( _true_ is default )
        - this is just a shortcut to exclude all the hasMany relations
      - **auto** : should tracking be turned on by default? ( _false_ is default )
        - auto tracking means when any model is saved/updated/reloaded the tracker will save
          the current state, allowing you to rollback anytime 
      - **enableIsDirty** : sets up computed properties on a model
        - ```hasDirtyRelations```  for checking on changed relationships  
        - ```isDirty```            for checking on any changes
          - NOTE: not working for object type attributes, since those are too 
            difficult to observe for the purpose of computed properties
            
  - Model configuration
    - Takes precedence over global
      - So, globally auto track could be off, but on one model you can turn it on
    - The options available are: 
       - **trackHasMany** : same as global trackHasMany  
       - **auto** : same as global auto  
       - **only** : limit the attributes/associations tracked on this model to just these
       - **except** : don't include these attributes/associations
       - You can use 'only' and 'except' at the same time, but you could also clean your nose with a pipe cleaner         

```javascript
  // file config/environment.js
  var ENV = {
    modulePrefix: 'dummy',
    environment: environment,
    rootURL: '/',
    locationType: 'auto',
    changeTracker: { trackHasMany: true, auto: true }, 
    EmberENV: {
    ... rest of config

```
  - Set options on the model

```javascript
  // file app/models/user.js
  export default Model.extend({
    changeTracker: {only: ['info', 'company', 'pets']}, // settings for user models

    name: attr('string'),
    info: attr('object'),
    json: attr(),
    company: belongsTo('company', { async: false, polymorphic: true }),
    profile: belongsTo('profile', { async: true }),
    projects: hasMany('project', { async: false }),
    pets: hasMany('pet', { async: true, polymorphic: true })
  });
```

### Serializer extras
  - Mixin is provided that will allow you to remove any attributes/associations
    that did not change from the serialized json
  - Useful when you want to reduce the size of a json payload
  - removing unchanged values can be big reduction at times

Example:

  Let's say you set up the user model's serializer with keep-only-changed mixin

```javascript
// file: app/serializers/user.js
import DS from 'ember-data';
import keepOnlyChanged from 'ember-data-change-tracker/mixins/keep-only-changed';

export default DS.RESTSerializer.extend(keepOnlyChanged);
```

Then when you are updating the user model

```javascript
user.set('info.foo', 1);
user.serialize(); //=> '{ info: {"foo:1"} }'
```

Without this mixin enabled the json would look like:
```javascript
  { name: "dude", info: {"foo:1"}, company: "1" companyType: "company", profile: "1" }
```
where all the attributes and association are included whether they changed or not


## Extra's 
  - Adds a few more helpful methods to ember data model
    - ```didChange(key) ```
      - did the value on this key change?
    - ```savedTrackerValue(key)``` 
      - this is the value that the key had after it was created/saved and 
      before any modifications

Usage:
```javascript
  user.startTrack(); // saves all keys that are being tracked
  user.savedTrackerValue('info') //=> {foo: 1}  original value of info
  user.set('info.foo', 8)      
  user.didChange('info') //=> true
  user.savedTrackerValue('info') //=> {foo: 1}  original value of info    
```           
 
## Known Issues
 - When pushing data to the store directly to create a model ( usually done when using 
   websockets .. but same issue if using factory guy) you need to call ```model.saveTrackerChanges()``` 
   manually after creating that new model   
 - Testing 
   - In unit / integration tests you have to manually initialize change-tracker 
     if you are testing anything that requires the addon to be enabled

For example:
 
```javascript

import {moduleForModel, test} from 'ember-qunit';
import {make, manualSetup} from 'ember-data-factory-guy';
import {initializer as changeInitializer} from 'ember-data-change-tracker';

moduleForModel('project', 'Unit | Model | project', {

  beforeEach() {
    manualSetup(this.container);
    changeInitializer();
  }
});

```                                   
