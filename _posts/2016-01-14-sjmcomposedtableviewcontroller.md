---
layout: post
title: "SJMComposedTableViewController"
date: 2016-01-14 21:22:23 +0100
tags: cocoa coding
---

Interface Builder makes it very easy to create static table views for things like settings screens. With SJMComposedTableViewController I try to simplify the creation of tables programmatically, without the drawbacks of using storyboards. 

# Problem

The first big iOS app I was working on required around 20 different screens, which is quite a lot for a phone app. The app was developed in a small team and I wanted to concentrate mostly on the internals and the syncing with the backend. But when delegating UI work to other coders, a few things became apparent quickly: 

* Working on storyboards in teams leads to merge conflicts. 
* Large storyboards are often messy and slow. 
* Things get inconsistent (e.g. the way a row gets deselected). 

I also suspect that the traditional approach hinders experimentation. Should a text input, a picker view or a segmented control be replaced with a separate screen? If that new screen can be implemented with a few lines of code, developers are more likely to test several options and choose the best one. But if it requires to create a new class, implement delegates or even connect storyboards, the effort might not seem worth it. Or even worse, someone who did all that work might not want to throw it away, even if the simpler solution turns out to better. 

# Solution

My solution is a generic tableview controller where cells can be added in code. 

```objective_c
@interface SJMComposedTableViewController : UITableViewController
- (void)addCell:(UITableViewCell *)cell;
- (void)addSection;
- (void)setSectionHeader:(NSString *)header;
- (void)setSectionFooter:(NSString *)footer;
- (void)reset;
@end
```

The methods should be more or less self explanatory. Cells and section separators can be appended like it was a list. A header or footer can be defined for the current section. `reset` removes all sections and cells. 

```objective_c
SJMComposedTableViewController *vc;
vc = [[SJMComposedTableViewController alloc] init];
vc.title = @"Test";
SJMActiveCell *cell = [SJMActiveCell cell];
cell.key = @"Hi";
[vc addCell:cell];
[self.navigationController pushViewController:vc animated:YES];
```

The example creates a new table view controller in place, adds a single cell and pushes it onto the navigation stack. Since the cells are added in code, they can be created dynamically (e.g. from an array of options). I usually save a reference to cells I need to change later, so I don't have to reconstruct the table if a value changes. (`[tableview reloadData]` can lead to a weird user experience, e.g. by interrupting animations.)

For more complex tables I subclass `SJMComposedTableViewController` and add the cells in a function called `compose`. I then call compose from `initWithStyle:` and/or `initWithCoder:` to provide loading from a storyboard. 

You might have noticed that `SJMComposedTableViewController` doesn't reuse cells. All cells stay allocated as long as the view controller exists. While this is fine for small tables, it can be a memory hog. If your table contains a lot of cells, I recommend not using my class, or creating a subclass and handling `tableView:cellForRowAtIndexPath:` separately for the large sections. 

# SJMActiveCell

Any `UITableViewCell` can be added to `SJMComposedTableViewController`, but its true power becomes apparent when using `SJMActiveCell`. The active cell provides a block that is called when the cell is selected within a composed table. 

```objective_c
SJMActiveCell *cell = [SJMActiveCell cell];
cell.key = @"Is this useful?";
cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;
cell.actionBlock = ^(SJMActiveCell *cell) {
    SJMComposedTableViewController *vc;
    vc = [[SJMComposedTableViewController alloc] init];
    vc.title = @"Useful?";
    SJMActiveCell *ans = [SJMActiveCell cell];
    ans.key = @"Yes";
    ans.actionBlock = ^(SJMActiveCell *cell) {
        cell.value = @"Yes";
        [self userLikesMe];
        [self.navigationController popViewControllerAnimated:YES];
    };
    [vc addCell:ans];
    [self.navigationController pushViewController:vc animated:YES];
};
[self addCell:cell];
```

The example shows a simple cell asking a question. When tapped, an answer can be chosen in a new table that is dismissed automatically upon selection. The view controller with the answers is only created when needed, and deallocated automatically afterwards. No additional classes or files are required. 

```objective_c
@interface SJMActiveCell : UITableViewCell
+ (SJMActiveCell *)cell;
@property (strong, nonatomic) NSString *key;
@property (strong, nonatomic) NSString *value;
@property (nonatomic) CGFloat height;
@property (nonatomic) BOOL enabled;
@property (strong, nonatomic) void (^actionBlock)(SJMActiveCell *cell);
- (void)deselectAnimated:(BOOL)animated;
@end
```

The `key` and `value` properties are just accessors for the text of their textLabel and detailTextLabel. `height` will tell the composed table to use a custom height (unless its 0). `enabled` can be used to disable a cell, which will grey out the cell and make the action block unavailable. If `actionBlock` is set, the cell will be selectable and the block will be called upon selection. 

`deselectAnimated:` will tell the table view controller to deselect the cell. It should be called in each action block that doesn't push a view controller, so the cell is deselected after the tap ends. 

`SJMActiveCell` can be subclassed to provide web views (automatic height calculation), text input or other things. But for many applications, the class is capable enough. Since it inherits from `UITableViewCell`, properties of the text labels can be changed. I tend to use centered labels for actions and multiline labels to show information. 

# Source Code

The source code is available on [GitHub](https://github.com/stepmuel/ComposedTableViewController). The repository includes a demo project with additional usage examples. I've been using the code in several projects and am only making small improvements from time to time. So the code can be considered stable and well tested.  

If my code saves you a significant amount of time, consider a small donation to my [PayPal account](https://www.paypal.me/heap).


