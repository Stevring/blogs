# Key-Value Observation

KVO: a mechanism that allows objects to be notified of changes to properties of other objects.

## Basic Usage

### Approach 1 (OC and Swift)

1. register: call `addObserver:forKeyPath:options:context` on the **observed** object.
2. receive change notifications: implement `observeValueForKeypath:ofObject:change:context` on the **observing** object.
3. unregister: call `removeObserver:forKeyPath:` on the **observed** object.

**Context**

The `context` parameter can be used to make sure that this change notification is destined for your observer.

For example, when a superclass also observes the same object on the same key path, you may pass different contexts in super- and sub-class observations.

### Approach 2 (Swift only)

1. register: call `observe(_:options:changeHandler:)` on the **observing** object (the observed property should be annotated with `@objc dynamic`, returns an `NSKeyValueObservation` observation object).
2. receive change notifications in the **changeHandler** closure.
3. unregister: call `invalidate` on the observation object, or when the observation object is deinited it auto-removes observation.

## To-many Relationship Observing

**to-many relationship**

If A has a property that holds multiple Bs, then A has a to-many relationship with B. If the property is an array, then it is an **ordered to-many** relationship. If the property is a set, then it is an **unordered to-many** relationship.

For an ordered to-many relationship:

| how                                                                                    | operation          | receive notification | change.kind                   | change.new                       | change.old                     |
| :------------------------------------------------------------------------------------- | :----------------- | :------------------- | :---------------------------- | :------------------------------- | :----------------------------- |
| call methods on object returned by `mutableArrayValueForKey:`                          | add/remove/replace | YES                  | insertion/removal/replacement | present on insertion/replacement | present on removal/replacement |
| directly call methods on array                                                         | add/remove/replace | NO                   | -                             | -                                | -                              |
| call `willChange:valuesAtIndexes:forKey:` and `didChange:valuesAtIndexes:forKey:` pair | add/remove/replace | YES                  | insertion/removal/replacement | present on insertion/replacement | present on removal/replacement |

Note:

1. Only Approach 1 receives new value / old value in notification. Although Approach 2 has the new/old key present, the value is empty (maybe Apple's bug).
2. Note 2: For an unordered to-many relationship: similar with the above.
3. `willChangeXXX` and `didChangeXXX` should be called in pair, otherwise the notification won't be sent.

## Control Notifications

### Automatic and Manual Notifications

**automatic notifications** will be sent when mmutate a property with

1. KVC methods (`setValue:forKey:` `set<Key>:`)
2. [collection proxy object](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/AccessingCollectionProperties.html#//apple_ref/doc/uid/10000107i-CH4-SW1)
3. Accessor methods (`setKey:`)

automatic notification can be disabled by overriding `automaticallyNotifiesObserversForKey:`.

**manual notifications** will be sent when calling `willChangeXXX` and `didChangeXXX` pair.

### Dependent Keys

override the class method `keyPathsForValuesAffectingValueForKey:` to tell the system that changes to some properties will also affect one specific property, and observations should be sent if these properties change.
