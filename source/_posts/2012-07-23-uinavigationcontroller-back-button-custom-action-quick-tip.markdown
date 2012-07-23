---
layout: post
title: "UINavigationController Back Button Custom Action (Quick Tip)"
date: 2012-07-23 19:12
comments: true
categories: [Objective-C, UINavigationController, iOS, UITableView, ComboBox]
---

We all know the good, old `UINavigationController` that is used in almost every iOS application. `UINavigationController` has a default action when back button is pressed on one of its stack's view controllers i.e. - go back one level in the navigation stack. However, there are cases when we need to do something extra when back button is pressed. <!-- more --> For example - *and this is just an example usage*, if we want to mimic combo box - like control in iOS (which is usually implemented via single, right detail `UITableViewCell` in the parent = table view cell with style `UITableViewCellStyleValue1`) and we do the selection by pushing a new, child view controller with a checkbox enabled `UITableView`.

{% img left /images/posts_images/tableviewselection.png 618 569 Settings.app %} Such an implementation can be found in the stock Settings.app if we choose Sounds and if we take a look at the Ringtone right detail UITableViewCell. That is a combo box - like structure and selecting it brings a new, child view controller with selection logic.

<br />    
<br />    
<br />     

In such a scenario, what I like to do is set the parent's right detail `UITableViewCell` with the selected value from the child `UITableView` when the back button on the child view controller is pressed. That way I don't have to set the selected value on every selection user does in the child UITableView (at `tableView: didSelectRowAtIndexPath:` with a call to the delegate) so I only do the setting once, at the end. I was having trouble finding the *right & correct* place to put the delegate call that sets the selected value on the parent and finally after some thinking around, I ended up with this awesome spot to put your back button action call.

```objc Where to put your back button action method call
- (void)viewWillDisappear:(BOOL)animated
{
    if ([self.navigationController.viewControllers indexOfObject:self] == NSNotFound) {
        [self.delegate setParentSelectedCity:self.selectedCity];
    }
    [super viewWillDisappear:animated];
}
```

This code should be put in the child view controller and it **ensures** us that back button has been pressed because `self` is no longer in the `UINavigationController` stack. What I also have here is my `setParentSelectedCity:` delegate method that sets the parent's right detail `UITableViewCell`. `self.selectedCity` is a property that holds the selected value by the user.

---

Of course, you can put whatever method call you want on that place.

Thanks for reading my post, I hope it may be of any help.