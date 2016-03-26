---
layout: post
title: "Convenient NSLayoutConstraint Creation"
date: 2015-12-21 01:01:01 +0100
tags: cocoa coding
---

Setting up auto layout programmatically tends to be a rather unpleasant chore. I tried to change that by creating a UIView category for convenient NSLayoutConstraint creation using a simple format language.

# Problem

I'm not a huge fan of the Xcode Interface Builder. It feels slow, unpolished and is cumbersome to use. I often get Interface Builder related merge conflicts and as the number of views grows, the tool intended to improve my productivity starts to feel more and more like a dungeon crawler. For that reason, I end up creating most of my layouts in code. 

When creating layouts programmatically, auto layout can still be a big help as it can replace a lot of annoying math done in the `layoutSubviews` method. The [visual format language](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/AutolayoutPG/VisualFormatLanguage.html) makes laying out views as easy as creating ASCII art. But more often than not, I know exactly which individual constraints I want to use and would like to add them directly. In code, this might look like this:

```objective_c
NSLayoutConstraint *c1, *c2, *c3, *c4;
c1 = [NSLayoutConstraint constraintWithItem:v1
                                  attribute:NSLayoutAttributeTop
                                  relatedBy:NSLayoutRelationEqual
                                     toItem:v2
                                  attribute:NSLayoutAttributeTop
                                 multiplier:1.0
                                   constant:0.0];
c2 = [NSLayoutConstraint constraintWithItem:v1
                                  attribute:NSLayoutAttributeBottom
                                  relatedBy:NSLayoutRelationEqual
                                     toItem:v2
                                  attribute:NSLayoutAttributeBottom
                                 multiplier:1.0
                                   constant:0.0];
c3 = [NSLayoutConstraint constraintWithItem:v1
                                  attribute:NSLayoutAttributeLeft
                                  relatedBy:NSLayoutRelationEqual
                                     toItem:v2
                                  attribute:NSLayoutAttributeLeft
                                 multiplier:1.0
                                   constant:0.0];
c4 = [NSLayoutConstraint constraintWithItem:v1
                                  attribute:NSLayoutAttributeRight
                                  relatedBy:NSLayoutRelationEqual
                                     toItem:v2
                                  attribute:NSLayoutAttributeRight
                                 multiplier:1.0
                                   constant:0.0];
[self addConstraints:@[c1, c2, c3, c4]];
```

This code will make the frame of view v1 match v2. A lot of typing for such a simple task! I could probably save a couple of lines by using the visual format language, but I don't even know which format strings I had to use to achieve the same result. 

# Non-Visual Format Language

To simplify the creation of single constraints, I came up with a very simple format language. It only allows the creation of one constraint at a time, and is implemented as a UIView category (UIView+ConstraintFormat). It will significantly simplify the example above. 

```objective_c
// syntax: view1.attribute1 = multiplier * view2.attribute2 + constant
NSDictionary *views = NSDictionaryOfVariableBindings(v1, v2);
[self addConstraintWithFormat:@"v1.top = v2.top" views:views];
[self addConstraintWithFormat:@"v1.bottom = v2.bottom" views:views];
[self addConstraintWithFormat:@"v1.left = v2.left" views:views];
[self addConstraintWithFormat:@"v1.right = v2.right" views:views];
```

My category provides two more functions I frequently use. `addConstraintsForView:toFillView:` will implement the initial example, and `addConstraintsForView:toFillViewController:` will make a view cover the view of a view controller, excluding the top- and bottom bar (if visible). 

# Implementation

The heart of the category is a relatively simple regular expression. (Spaces added for readability.)

	(\w+) \. (\w+) = ([+-]?[\d\.]+\*)? (([^\W\d]\w*) \. (\w+))? ([+-]?[\d\.]+)?

Before applying the expression, all white spaces are removed from the input string. This make the expression shorter and more readable readable since no `\s*` is needed between the tokens. Floating point numbers are matched with `([+-]?[\d\.]+)` which would theoretically allow to have multiple decimal points. The second view name matcher `([^\W\d]\w*)` enforces the first character to be non-numeric. This is required to not mistake a number with decimal point for a view.attribute token. 

Auto layout provides error messages for illegal or conflicting constraints. Since a developer already has to test constraints and deal with those errors, I didn't think it was necessary to thoroughly validate the format string. Invalid format strings will just create unwanted constraints that need to be debugged the same way an unwanted constrain created by a valid format string would. In case someone wants to write a less forgiving matching pattern or even write a formal syntax definition, please let me know!

The source code is available on [GitHub](https://github.com/stepmuel/ConstraintFormat). It contains the UIView+ConstraintFormat category and a small example iOS app to demonstrate its usage. I hope it will help clean up your code and make you less dependent on interface builder. 


