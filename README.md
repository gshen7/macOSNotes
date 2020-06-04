# Documenting various things for building macOS apps
* short quick form documentation in a way that makes sense to me - trying to be as heavy with code samples as _reasonable_ for me
* feel free to contribute in a way that is _reasonable_ to you

# Swift
## Accessibility API
* some [official documentation](https://developer.apple.com/documentation/applicationservices) exists but I've had difficulty finding more, so most of how I learned was through searches on GitHub of the things listed in the official documentation
* note that this API exists in objective c as well but the notes below provide syntax in swift

### Accessibility Inspector Tool
* hierarchy of UI elements can be explored using the [accessibility inspector tool in xcode](https://developer.apple.com/library/archive/documentation/Accessibility/Conceptual/AccessibilityMacOSX/OSXAXTestingApps.html)
* for inspecting menus, open the menu using the accessibility inspector, then refresh using command R, or open the menu, then use option space to toggle the inspector selector

#### Objects
##### AXUIElement
* everything is an `AXUIElement`

###### Useful Attributes
**kAXWindowsAttribute**
* if a `AXUIElement` is an application, this is all the elements that are windows for that application

**kAXFocusedUIElementAttribute**
* the corresponding `AXUIElement` that is in focus for the systemwide `AXUIElement`

**kAXChildrenAttribute**
* the `AXUIElement` children of an `AXUIElement` 

###### Useful Actions
**kAXPressAction**
* presses the element

**kAXShowMenuAction**
* shows the menu if there is one

#### Functions
##### AXIsProcessTrustedWithOptions(options)
* returns `Bool`
* options is in the form of a dictionary
* checkOptPrompt as true prompts the user to allow control 

##### AXUIElementCreateApplication(pid)
* returns `AXUIElement`
* gets the UI element of the application with pid

##### AXUIElementCreateSystemWide()
* returns `AXUIElement`
* gets UI element representing entire system

##### AXUIElementCopyAttributeNames(element, pointer)/AXUIElementCopyActionNames(element, pointer)
* puts all the attribute/action names available for an `AXUIElement` in the variable at pointer
```
var names:CFArray? = nil
AXUIElementCopyAttributeNames(element, &names)
```

##### AXUIElementCopyAttributeValue(element, attribute, pointer)
* puts all the value for a specified attribute for an `AXUIElement` in the variable at pointer (ex. `kAXFocusedUIElementAttribute`)
```
var focused: CFTypeRef?
AXUIElementCopyAttributeValue(systemWide, kAXFocusedUIElementAttribute as CFString, &focused)
```
* for values that are certain types (ex. `kAXPositionAttribute` is a `CGPoint`), need to use `AXValueGetValue`
```
var positionRef: AnyObject?
AXUIElementCopyAttributeValue(focusedWindow!, kAXPositionAttribute as CFString, &positionRef)

var position = CGPoint.zero
AXValueGetValue(positionRef as! AXValue, .cgPoint, &position)
```

##### AXUIElementGetPid(element, pointer)
* puts the pid that is managing an `AXUIElement` in the variable at pointer
```
var pid: pid_t = 0
AXUIElementGetPid(element as! AXUIElement, &pid)
```

##### AXUIElementCopyAttributeValues(element, attribute, index, limit, pointer)
* for attributes that are arrays, puts the attribute values in the variable at pointer
* I've always used index as 0 and limit as 99999
* most used for me for the children attribute
```
var childrenArray: CFArray?
AXUIElementCopyAttributeValues(element, kAXChildrenAttribute as CFString, 0, 99999, &childrenArray)
var children = (childrenArray as! Array<AXUIElement>)
```

##### AXUIElementPerformAction(element, actionType)
* performs an action (ex. press)
```
AXUIElementPerformAction(element, kAXPressAction as CFString)
```

##### AXUIElementIsAttributeSettable(element, attribute, pointer)
* checks if an attribute is settable (ex. focused)
```
var att: DarwinBoolean = false
AXUIElementIsAttributeSettable(font, kAXFocusedAttribute as CFString, &(att))
```

##### AXUIElementSetAttributeValue(element, attribute, value)
* sets an attribute, although the value type is sometimes annoying (ex. focused)
```
AXUIElementSetAttributeValue(element, kAXFocusedAttribute as CFString, kCFBooleanTrue as CFTypeRef)
```

##### AXUIElementSetMessagingTimeout(element, secondsAsFloat)
* used this so main thread isn't blocked whenever trying to do something with accessibility api
* i think no matter what the function will eventually occur, this just limits how long you wait for it to return
* setting a timeout of 0.0 didn't seem to keep it from being blocked but 0.01 was sufficient
* i didnt use this every time i used the accessibility api; i only used it for performing certain actions that were known to be problematic

## Global event monitoring
### NSEvent approach
#### Functions
##### NSEvent.addGlobalMonitorForEvents(matching, handler)
* can't stop events from propagating - because handler has to return void

### CGEvent approach
#### Objects
##### CGEventMask
* based on bit flags 
* for each event you want to listen for (ex. keyDown), take the bitwise OR with `(1 << CGEventType.keyDown.rawValue)`
```
var eventMask = (1 << CGEventType.keyDown.rawValue) | (1 << CGEventType.flagsChanged.rawValue)
...
eventMask = eventMask | (1 << CGEventType.leftMouseDown.rawValue) | (1 << CGEventType.rightMouseDown.rawValue) | (1 << CGEventType.otherMouseDown.rawValue)
```

#### Functions
##### CGEvent.tapCreate(location, placement, options, eventMask, callbackFunction, userInfo)
* returns `CGEventTap`
* placement allows for setting if the callback occurs before or after existing responses
* callback has to be outside of any class and return an optional `Unmanaged<CGEvent>` type
* I just set userInfo to nil
* then follow up code including enabling the tap 
```
guard let eventTap = CGEvent.tapCreate(tap: .cgSessionEventTap, place: .headInsertEventTap, options: .defaultTap, eventsOfInterest: CGEventMask(eventMask), callback: respondToEvent, userInfo: nil) else {
    print("event tap failed to create")
    exit(1)
}
let loopSource = CFMachPortCreateRunLoopSource(kCFAllocatorDefault, eventTap, 0)
CFRunLoopAddSource(CFRunLoopGetCurrent(), loopSource, .commonModes)
CGEvent.tapEnable(tap: eventTap, enable: true)
CFRunLoopRun()
```
* in the callback function, `return nil` stops propagation, while `return Unmanaged.passRetained(event)` propagates it

### CGEvent to NSEvent
```
let nsEvent = NSEvent.init(cgEvent: event)!
```

## Swift Data Types
### Tuples
* access using .#
* create with ()

## Visual Elements
### Getting rid of the dock icon
* add a row in info.plist
* key: Applicaiton is agent (UIElement)
* type: Boolean
* Value: YES

## Delegates
### Accessing app delegate elsewhere
* with `NSApplication.shared.delegate`

# objective c
## Syntax for functions
### Declaring and calling functions with multiple arguments
* argVariableName is what is used within the function
* argName is what is used when calling the function 
* function declaration
```
- (void)funcName:(argType)argVariableName argName:(argType)argVariableName
{
```
* function call
```
[self.funcName:argValue argName:argValue];
```

### Calling a function from a parent view controller
```
if (self.delegate && [self.delegate respondsToSelector:@selector(funcInParent:)]) {
    [self.delegate funcAtParent:argValueIfNeeded];
}
```

## NSDate
### Getting and using components
```
NSDateComponents *components = [[NSCalendar currentCalendar] components:NSCalendarUnitDay | NSCalendarUnitMonth | NSCalendarUnitYear fromDate:date];
// usage: components.year, components.month, components.day
```

# where things are in xcode
## Sandbox
* some things require you to not be sandboxing
* remove by clicking the project in project navigator, then the signing and capabilities tab, then clicking the x in the sandbox section

---

If you found this helpful and want to support me, consider sponsoring me via (Venmo)[https://venmo.com/garyshen]/(Paypal)[paypal.me/GaryShen]
