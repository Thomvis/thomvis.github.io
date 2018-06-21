---
layout: post
title: "Customizing the source location of failures reported by XCTest"
---

If you're writing unit tests and use helper methods to factor out common assertions, you might have encountered situations where test failures would not show up in the correct location in your Xcode editor. The failure would be reported in the helper method, instead of in the actual test. Especially if you call the helper method multiple times, perhaps even from multiple tests, it would become hard to figure out the context of the failures.

![Two failures in a helper method](/media/xctest-failure1.jpg)

In this situation, I can see that the assert in `assertSymmetry` failed twice, but I can't see the context of the failures. I can also see that `testSerialization` failed, but it doesn't show which part of the test failed.

## Solution
To have the failures show up inside the `testSerialization` test, we need to capture the file and line number for each call to `assertSymmetry` and pass them to `XCTAssertEqual`. `file` and `line` are optional parameters of `XCTAssertEquals`, which default to the file path and line number of the `XCTAssertEquals` call site. You can pass them to override this behavior and explicitly specify the file and line to display the failure at. To capture the values for the call sites of `assertSymmetry`, we add `file` and `line` as optional parameters to it and use the `#file` and `#line` compiler directives as their default values. The Swift compiler fills them in, so we don't have to.

This results in every call to `assertSymmetry` from `testSerialization` to be expanded to something like `assertSymmetry(of: field, file: "/path/to/Test.swift", line: 15)`. The result: failures are now reported on the line where we call `assertSymmetry`, instead of where we call `XCTAssertEqual`.

![Failures in the actual test method](/media/xctest-failure2.jpg)

This works not only for `XCTAssertEquals`, but also for `XCTAssert`, `XCTAssertNil`, `XCTFail` and any other XCTest assertion. On top of that, many third party testing libraries also support it, e.g. [Nimble](https://github.com/Quick/Nimble/blob/d0bea4ff70428c0fb30d26c8c1fa2500cd4570ff/Sources/Nimble/DSL.swift#L4) and [iOSSnapshotTestCase](https://github.com/uber/ios-snapshot-test-case/blob/e8418651f1cfae782163881e86c9457eef7c9525/FBSnapshotTestCase/SwiftSupport.swift#L11).

All the right failures, in all the right places.
