---
layout: post
title: "iOS7 blurred views"
date: 2013-08-02 16:53
tags: [ios7, development]
---

iOS 7 is (apparently) all about the blur. This article explains one way of mimicing the iOS 7 Control Center, bluring it's background as it moves into view.

{% img /images/posts/ios-7-blurred-views.png %}

## DSLGlassView

.```DSLGlassView``` is our ```UIView``` subclass that implements the blurred view. When a ```DSLGlassView``` is added to the view heirarchy, it grabs a snapshot of it's superview. It then blurs this view using the UIImage category methods from Apple's 'Running With A Snap' sample code. Once we have a blurred snapshot, we set it to be the content of the view's layer.

``` objective-c
- (void)didMoveToSuperview {
    [super didMoveToSuperview];

    if (self.superview != nil) {
        // Hide ourself so we don't appear in the snapshot
        self.hidden = YES;

        // Give the view we've been added to a chance to draw itself
        dispatch_async(dispatch_get_main_queue(), ^{
            // Grap a UIImage of the superview
            UIGraphicsBeginImageContextWithOptions(self.superview.bounds.size, YES, 0);
            [self.superview drawViewHierarchyInRect:CGRectMake(0, 0, self.superview.bounds.size.width, self.superview.bounds.size.height) afterScreenUpdates:NO];
            __block UIImage *superviewImage = UIGraphicsGetImageFromCurrentImageContext();
            UIGraphicsEndImageContext();

            // Create a blurred version on a background thread because it's slow
            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), ^{
                superviewImage = [superviewImage dsl_applyLightEffect];

                // Now we have a blurred image of the superview, use it as our layer's content back on the main thread
                dispatch_async(dispatch_get_main_queue(), ^{
                    self.layer.contents = (id)[superviewImage CGImage];
                    [self updateLayerContentsRect];

                    // Make ourself visible again
                    self.hidden = NO;
                });
            });
        });
    }
}
```

The main trick is to set the ```contentsRect``` of the layer to match the area of the superview the ```DSLGlassView``` covers. Whenever the size or position of the ```DSLGlassView``` is updated, we update it's ```contentsRect```.


``` objective-c
- (void)setFrame:(CGRect)frame {
    [super setFrame:frame];
    [self updateLayerContentsRect];
}

- (void)setCenter:(CGPoint)center {
    [super setCenter:center];
    [self updateLayerContentsRect];
}

- (void)updateLayerContentsRect {
    if (self.superview != nil) {
        // Set our layer's content rect so it shows the part of the superview we're covering
        CGRect contentsRect = self.frame;
        contentsRect.origin.x /= self.superview.bounds.size.width;
        contentsRect.origin.y /= self.superview.bounds.size.height;
        contentsRect.size.width /= self.superview.bounds.size.width;
        contentsRect.size.height /= self.superview.bounds.size.height;

        self.layer.contentsRect = contentsRect;
    }
}
```

Bear in mind, while this approach works great with a view moving over static views, it won't work if the superview's contents are changing.

## Summary

The code for this post is available on Github <ADD_GITHUB_URL> and includes a sample project.
