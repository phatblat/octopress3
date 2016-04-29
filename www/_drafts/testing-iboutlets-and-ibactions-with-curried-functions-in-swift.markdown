---
layout: post
title: "Testing IBOutlets and IBActions With Curried Functions in Swift"
tags: swift, interface-builder, testing, function-currying, quick, nimble
---

> This post is a eulogy to syntactic-sugar-filled curried functions.

## IBOutlet Testing

A couple years ago I built a universal iPhone / iPad app with three storyboards:

- iPhone
- iPad
- shared scenes

Many of the view controllers between the device-specific storyboards were shared and thus the outlets and actions all had to be the same. Each outlet property was bound to two different storyboards and required launching the app in different simulators to manually validate they were hooked up correctly. It was a constant challenge to keep them in sync whenever renaming a property or method. Most of the time I would forget to update at least one outlet and I’d have another lovely crasher from the device I forgot to test on.

I came up with a scheme to use unit tests to assert that these outlets and actions were bound correctly so that I could validate them almost instantly. I ended up with a handful of ugly functions that I never shared.

## Swift Curried Functions

When I learned that Swift had super-clean function currying syntax I refactored these ugly helper functions into something much more beautiful, learning about a new language feature in the process.

## Outlet Assertion

Calling one of these curried outlet assertion functions in Swift is very simple.

```swift
it("has a leftDoneButton outlet") {
	hasButtonOutlet("leftDoneButton")
}
```

> The tests in this post use the [Quick ] testing framework

So, what is that `hasButtonOutlet` magic? It’s a partially-applied function (I think).

```swift
var hasButtonOutlet: String -> UIButton?
hasButtonOutlet = outlet(viewController)
```

Calling the fully-applied function would look like this:

```swift
outlet(viewController)("leftDoneButton”)
```

The point of partially applying the function is so that `viewController` doesn’t have to be passed in on each call. It reduces noise and makes the tests more readable. This is handy when you have dozens of outlets and are chasing down which one you mistyped.

```swift
private func outlet(viewController: UIViewController) -> (String) -> AnyObject? {
	return { (outlet: String) -> AnyObject? in
		guard let object = viewController.valueForKey(outlet)
			else { fail("\(outlet) outlet was nil"); return nil }

		return object
	}
}
```

## Action Assertion

The action assertion functions are similarly simple.

```swift
it("receives a didTapDone action from leftDoneButton") {
	receivesAction("didTapDone", from: "leftDoneButton")
}
```

One caveat is that they require an outlet on the thing sending the action. A lot of the time an outlet isn’t necessary for an action-sending UI element, but I haven’t found a way to get the actions from the view controller (yet).

Here’s the setup for the partially-applied `receivesAction`:

```swift
var receivesAction: (String, from: String) -> Void
receivesAction = action(viewController)
```

The implementation of the `action` function is more complex as getting to the action differs depending on whether the UI element is a `UIBarButtonItem` or a type of `UIControl`.

```swift
func action(viewController: UIViewController) -> (String, from: String) -> Void {
	return { (expectedAction: String, expectedOutlet: String) in
		let optionalControl = outlet(viewController)(expectedOutlet)

		var target: AnyObject?
		var action: String?

		if let control = optionalControl {
			switch control {
			case let button as UIBarButtonItem:
				target = button.target
				action = button.action.description
			case let control as UIControl:
				target = control.allTargets().first!
				var allActions: [String] = []
				for event: UIControlEvents in [.TouchUpInside, .ValueChanged] {
					allActions += control.actionsForTarget(target!, forControlEvent: event) ?? []
				}

				// Filter down to the expected action
				action = allActions.filter({$0 == expectedAction}).first
			default:
				fail("Unhandled control type: \(control.dynamicType)")
			}
		}

		expect(target) === viewController
		expect(action).toNot(beNil())
		if let action = action {
			expect(action) == expectedAction
		}
	}
}
```

Credit goes to @jonreid for his [post on stack overflow] for how to test IBActions.

## Code

A full project demonstrating these helper functions is at:
https://github.com/phatblat/CurriedOutletFunctions

## Deprecated 😭

Shortly after @allonsykraken posted [Hipster Swift](http://krakendev.io/blog/hipster-swift), I learned that the that the super-clean syntactic sugar version of curried functions is [going away in Swift 3] and it made me sad. While this is a more esoteric language feature, I really like how curried functions can be used to simplify an API. Also, the way Swift implemented curried functions made them so easy to use.

Isn’t this:

```swift
func fourChainedFunctions(a: Int)(b: Int)(c: Int)(d: Int) -> Int {
	return a + b + c + d
}
```

…so much cleaner than this? [^curried-function-example]

```swift
func fourChainedFunctions(a: Int) -> (Int -> (Int -> (Int -> Int))) {
	return { b in
		return { c in
			return { d in
				return a + b + c + d
			}
		}
	}
}

fourChainedFunctions(1)(2)(3)(4)
```

[^curried-function-example]: Borrowed with :heart: from http://krakendev.io/blog/hipster-swift#currying

Apple, you can take my sweet curry, but you'll never take my Sriracha.
