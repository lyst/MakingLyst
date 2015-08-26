# Lyst iOS Coding Style Guide

This is the beginning steps of the iOS style guide for the mobile team. It is, by necessity, objective-c focused but does begin to consider Swift.

### Caveat Emptor:
- It is (currently, and probably always will be) incomplete
- It is a living document and subject to change (your changes are welcomed)
- It is not (yet) fully enforced automatically

### On Naming
As Uncle Bob says:

* Classes should be nouns
* Methods should be verbs

### Attribution and header info
Your file headers should contain useful information only. The file already has a name, as does the class, and your (clean, well named, hint hint) code documents itself.

```objective
//
//  Created by YOURNAME on CREATIONDATE.
//  Copyright (c) 2012-2015 Lyst. All rights reserved.
//
```

### Pull Requests
“Pull request diffs must be 500 lines or less. No large diffs are allowed.” - Square Register

Trialling: we will consider a diff to be the difference between the number of deletions and additions. So, if you make 1,100 deletions and 600 additions that’s still okay.

### Work In Progress Pull Requests
In order to let others see what you are doing as early as possible, consider committing and pushing your branch to github immediately you start it and opening a pull request with a WIP: prefix.

e.g. ‘WIP: IOS-101 Refactoring All The Things’

### Branch / Pull Request Naming
Branches should reference their JIRA task ID and a summary of what they are about. 

‘ios-101-refactortheroom’ is a good name
‘refactor’ is a bad name

### Categories
Category files must be named in this way

	ClassName+CategoryName.h / ClassName+CategoryName.m

Categories must group like functionality. Use new categories for new functionality.

	NSString+ReverseString
	NSString+Base64

### Braces
No conditionals shall be coded without braces (in order to prepare for Swift). 

```objectivec
if (conditional) {single.line.statement;}
	
if (conditional) {
	multiple;
	line;
	statement;
}
```

### dot.notation
We favour dot notation for properties, over square brackets, in order to prepare for Swift.

```objectivec
myObject.sizeProperty;
```

### `[dot.notation withMethods]`
We favour dot notation for single property access before calling a methods on that property, over square brackets, in order to prepare for Swift.

```objectivec
[myObject.sizeProperty methodName]
```

### Chaining
We try to avoid chaining multiple methods or properties together. This makes code hard to comprehend and impossible to debug without changing the code first.

This is not good practice

```objectivec
myObject.frame.size.width; 	// we have no idea what size is now
```

This is good practice

```objectivec
CGRect frame = myObject.frame;
CGSize frameSize = frame.size.
frameSize.width;
```

### `#define` constant values
`#defines` for constant values should be replaced by the (type checker friendly) 

```objectivec
const type name = value;
```

### Macros
Where possible, macros should be avoided in favour of functions.  

They are easy to reason about, to step through in debug, and come with free type checking goodness.

Use inline functions where you have performance concerns.

### `id`
Always avoid `id` whenever possible. It bypasses all of the compiler type safety checking goodness and encourages ‘clever’ thinking. 

### `instancetype`
Always prefer `instancetype` over `id` for factory and init methods. 

This not only gives you better type safety at compile time, but can avoid some nasty gotchas.

Favour initialisers for classes expressed as class methods. For example:

```objectivec
+(instancetype)classFactory
{
	return [[self alloc] init];
}
```

Caveat: beware of calling alloc with the specific class name (.e.g [MyClass alloc] init]). Doing so means, then we won’t be able to create subclasses.

### `NSDecimalNumber` vs `float`
As Apple mention in their Apple Pay documentation, floats and doubles are not to be trusted for financial calculations

> Although they may appear to be more convenient, IEEE floating point data types such as float and Double are not suitable for financial calculations. These data types use a base-2 representation of numbers, which means that some decimal numbers can’t be represented exactly—for example, 0.42 must be approximated as 0.41999 repeating. Such approximations can cause financial calculations to return incorrect results. [developer.apple.com/library/ios/ApplePay_Guide/CreateRequest.html](https://developer.apple.com/library/ios/ApplePay_Guide/CreateRequest.html#//apple_ref/doc/uid/TP40014764-CH3-SW2)

### Avoiding Half Pixel Calculations
Round all calculations that have to do with UI layout. There are some exceptions to this (such as rendering a 2 pixel line, either side of a half pixel) but as a general rule, rounded CGFloats avoid fuzziness.

Use our round64Helper, floor64Helper or ceil64Helper on pixel calculations.

### Self Retain Cycles
To avoid self retain cycles, create a weak self outside the block: 

```objectivec
__weak typeof(self) weakSelf = self;
```

And then transform it to a strong one inside of the block:

```objectivec
__strong typeof(self) strongSelf = weakSelf;
```

then use strongSelf.whatever in your block. Not using a strong reference will cause a compiler error.

### `!` conditional statements
In order to prepare for our migration to Swift, all future conditionals using the `!` operator are to be avoided unless they act upon a true boolean ([https://developer.apple.com/library/prerelease/mac/documentation/Swift/Conceptual/Swift_Programming_Language/BasicOperators.html](developer.apple.com/library/prerelease/mac/documentation/Swift/Conceptual/Swift_Programming_Language/BasicOperators.html)).

This will not pass PR anymore:

```objectivec
NSObject *object = [some thing];
if(!object) { // }
```

This will 

```objectivec
NSObject *object = [some thing];
if(object != nil) { // }
```

As will this

```objectivec
BOOL success = [some thing];
if(!success) { // }
```

And this

```objectivec
BOOL success = [some thing];
if(success == NO) { // }
```

### Literals
We will favour the modern literal syntax (even though it’s just syntactic sugar) 

```objectivec
NSString *str = @"Hello";
NSNumber *num = @1;
NSNumber *enumNum = @(LYNum);

NSObject *object = myDictionary["MyKey"]
```

We favour this because it produces cleaner, more expressive, code.

### Signedness
We favour signed integers over unsigned integers, to harmonise our coding best practice with that of Swift and aid future interoperability between code bases (Swift and Objective-C in the same project).

Specifically, use `NSInteger` whenever you need an integer value that is not of a specific byte size (and question why you need a specific byte size).

### `nullability`
Always annotate pointers with `nonnull` or `nullable`. It is a requirement that you always annotate methods and properties that are pointers with either `nonnull` or `nullable` and you are encouraged to reject PR’s without them. 
This improves clarity of intent, and allows the compiler to perform more checks on your code which will enable better bridging when we start blending Swift and Objective-C together.

### Readonly properties by default
All properties should be readonly until they cannot be (for easier testing and less brittle code)

```objectivec
@property (nonatomic, copy, readonly) NSString *name;
```

### Nonatomic properties by default
All properties should be nonatomic until they cannot be (for speed)

```objectivec
@property (nonatomic, copy, readonly) NSString *name;
```

### Copy properties for mutable types
All `NSString`, `NSDictionary` and `NSArray` properties should be copy, not strong (to prevent being handed an NSMutable version which could then change)

```objectivec
@property (nonatomic, copy, readonly) NSString *name;
```

### When it is not good to use properties
When they cause side effects, as Clang will warn you…

	“Property access result unused - getters should not be used for side effects”

### Writing Tests
Each test method should match this pattern

```objectivec
-(void)testThat
{
	// given

	// when

	// then
}
```

Then should have at most one assert since the whole method should be for a single test. Multiple asserts make it hard to discern the true nature of the test or it’s failures. 

The exception to this rule is where the asserts are used in the given, in order to assert a good known state before the when actions.

### Pairing methods visually close
In the case where we have methods that need another method: (e.g `addObserver`/`removeObserver` for Notification Centre) always have them visually close to ensure that we are not missing the pair (the `removeObserver` for example).

### Lyst iOS Team Definition Of “Done”
When deciding whether to move a task from in progress to ready for review, consider this checklist as a way to be assured that you are, indeed, done. If you cannot answer yes to each item, you are probably not done.

1. You are “code complete”
1. You have written all of the unit tests you consider necessary (and they pass)
1. You have written all of the integration tests you consider necessary (and they pass)
1. You have written all of the UI tests you consider necessary (and they pass)
1. You have documented any new analytics events you added
1. The acceptance criteria on the ticket are up to date, and complete
1. You are confident that your code passes the acceptance criteria
1. It has been reviewed by design team, where appropriate
1. If this feature is not to be used yet, then it is detached from the project in an appropriate way (feature toggle, no integration, etc)
1. If this were to be merged immediately, into master, and released today, you would have no qualms


